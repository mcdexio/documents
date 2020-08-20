# Internal Design of Tokenized Position

:warning: NOTE: This is a technical document of the smart contract. If you don't care how smart contracts are written, please skip this document.

## Motivation

The "TokenizedPosition"(TP) contract is used to convert a long position of Perpetual into ERC20 token. A long position [can be kept fully collateralized automatically](https://github.com/mcdexio/documents/blob/master/en/how-to-add-liquidity-to-amm.md), so it is possible to make a long position leave the Perpetual and enter the public Ethereum ecosystem.

For example: An ETH-PERP inversed perpetual is collteralized with ETH. So the short position (The short position is from a human perspective. In the contract it is still a long position) is a synthetic USD. A BTC-USDT-PERP vanilla perpetual is collteralized with USDT. So the long position is a synthetic BTC. etc.

Keep in mind that minting and redeeming will maintain the fineness of the synthetic coin, where the fineness is defined as MarginBalance / PositionSize of the TP contract.

## Contract status

The TP has 3 status, it also affected by the underlying Perpetual's status.

| Perpetual status | TP status | mint | redeem | settle | ERC20.transfer |
|------------------|-----------|------|--------|--------|----------------|
| Normal           | Normal    | ✔    | ✔     |        | ✔             |
| Normal           | Paused    |      |        |        |                |
| Normal           | Stopped   |      | ✔      |        | ✔             |
| Emergency        | Normal    |      |        |        | ✔              |
| Emergency        | Paused    |      |        |        |                |
| Emergency        | Stopped   |      |        |        | ✔              |
| GlobalSettled    | Normal    |      |        | ✔      | ✔             |
| GlobalSettled    | Paused    |      |        |        |                |
| GlobalSettled    | Stopped   |      |        | ✔      | ✔             |

### mint(tpAmount)

Transfer the collateral into the TP contract, and mint ERC20 tokens.

- Amount:= 
  - If the totalSupply == 0: tpAmount
  - Otherwise: PositionSize * tpAmount / totalSupply
- Calculate how much the collateral is, while keep the MarginBalance / PositionSize unchanged
  - If the totalSupply == 0, the TP must be an empty contract
    - Price:= MarkPrice
    - DeltaCash:= MarkPrice*Amount
  - Otherwise, the TP already has some value, keep the MarginBalance / PositionSize unchanged
    - The current MarginBalance
      - OldPNL1:= MarkPrice*PositionSize - EntryValue
      - OldSocialLoss:= LongSocialLossPerContract*PositionSize - EntrySocialLoss
      - OldFundingLoss:= (AccumulatedFundingPerContract*PositionSize - EntryFundingLoss)
      - OldPNL2:= OldPNL1 - OldSocialLoss - OldFundingLoss
      - OldMarginBalance:= CashBalance + OldPNL2
    - The trading price
      - Price:= OldMarginBalance/PositionSize
    - Open position
      - NewEntryValue:= EntryValue + Price*Amount
      - NewPositionSize:= PositionSize + Amount
      - NewEntrySocialLoss:= EntrySocialLoss + LongSocialLossPerContract*Amount
      - NewEntryFundingLoss:= EntryFundingLoss + AccumulatedFundingPerContract*Amount
      - NewCashBalance:= CashBalance + DeltaCash
    - The new marginBalance after redeeming
      - NewSocialLoss:= LongSocialLossPerContract*NewPositionSize - NewEntrySocialLoss
      - NewFundingLoss:= (AccumulatedFundingPerContract*NewPositionSize - NewEntryFundingLoss)
      - NewUPNL1:= MarkPrice*NewPositionSize - NewEntryValue
      - NewUPNL2:= NewUPNL1 - NewSocialLoss - NewFundingLoss
      - NewMarginBalance:= NewCashBalance + NewUPNL2
    - Solve this equation: NewMarginBalance/NewPositionSize == OldMarginBalance/PositionSize, we will get
      - DeltaCash:= 2*OldMarginBalance*Amount/PositionSize - MarkPrice*Amount
      - Require DeltaCash >= 0
- Transfer DeltaCash from the sender to the TP
- The sender sells Amount positions to the TP at Price. So TP is always openning the long position
- Mint tpAmount ERC20

Require:

- Not TP.Paused
- Not TP.Stopped
- Not Perpetual.IsEmergency
- IsSafe == TRUE after calling this function
- TP.totalSupply == PositionSize

### redeem(tpAmount)

Burn ERC20 tokens and transfer the collateral back to the sender.

- Amount:= PositionSize * tpAmount / totalSupply
- Calculate how much the collateral is, while keep the MarginBalance / PositionSize unchanged
  - The current MarginBalance
    - OldPNL1:= MarkPrice*PositionSize - EntryValue
    - OldSocialLoss:= LongSocialLossPerContract*PositionSize - EntrySocialLoss
    - OldFundingLoss:= (AccumulatedFundingPerContract*PositionSize - EntryFundingLoss)
    - OldPNL2:= OldPNL1 - OldSocialLoss - OldFundingLoss
    - OldMarginBalance:= CashBalance + OldPNL2
  - The trading price
    - Price:= OldMarginBalance/PositionSize
  - Close position
    - RPNL1:= Price*Amount - EntryValue*Amount/PositionSize
    - SocialLoss:= (LongSocialLossPerContract - EntrySocialLoss/PositionSize)*Amount
    - FundingLoss:= (AccumulatedFundingPerContract - EntryFundingLoss/PositionSize)*Amount
    - RPNL2:= RPNL1 - SocialLoss - FundingLoss
    - NewEntrySocialLoss:= EntrySocialLoss/PositionSize*(PositionSize - Amount)
    - NewEntryFundingLoss:= EntryFundingLoss/PositionSize*(PositionSize - Amount)
    - NewCashBalance:= CashBalance + RPNL2 - DeltaCash
    - NewEntryValue:= EntryValue/PositionSize*(PositionSize - Amount)
    - NewPositionSize:= PositionSize - Amount
  - The new marginBalance after redeeming
    - NewSocialLoss:= LongSocialLossPerContract*NewPositionSize - NewEntrySocialLoss
    - NewFundingLoss:= (AccumulatedFundingPerContract*NewPositionSize - NewEntryFundingLoss)
    - NewUPNL1:= MarkPrice*NewPositionSize - NewEntryValue
    - NewUPNL2:= NewUPNL1 - NewSocialLoss - NewFundingLoss
    - NewMarginBalance:= NewCashBalance + NewUPNL2
  - Solve this equation: NewMarginBalance/NewPositionSize == OldMarginBalance/PositionSize, we will get
    - DeltaCash:= 2*OldMarginBalance*Amount/PositionSize - MarkPrice*Amount
    - Require DeltaCash >= 0
- The sender buys Amount positions to the TP at Price. So TP is always closing the long position
- Transfer DeltaCash from the TP to the sender
- Burn tpAmount ERC20

Require:

- Not TP.Paused
- Not Perpetual.IsEmergency
- IsSafe == TRUE after calling this function
- totalSupply >= tpAmount

### settle()

This function is only available when the whole Perpetual is global settled.

- balance:= balanceOf the collateral in the TP
- tpAmount:= balanceOf the sender's ERC20
- Transfer the balance * tpAmount / totalSupply from the TP to the sender
- Burn tpAmount ERC20

Require:
- Perpetual.IsGlobalSettled


### pause()

Just set TP.Paused

### stop()

Just set TP.Stopped
