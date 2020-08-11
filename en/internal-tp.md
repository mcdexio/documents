# Internal Design of Tokenized Position

:warning: NOTE: This is a technical document of the smart contract. If you don't care how smart contracts are written, please skip this document.

## Functions and Motivation

The "TokenizedPosition"(TP) contract is used to convert a long position of Perpetual into ERC20 token. A long position [can be kept fully collateralized automatically](https://github.com/mcdexio/documents/blob/master/en/how-to-add-liquidity-to-amm.md), so it is possible to make a long position leave the Perpetual and enter the public Ethereum ecosystem.

For example: An ETH-PERP inversed perpetual is collteralized with ETH. So the short position (The short position is from a human perspective. In the contract it is still a long position) is a synthetic USD. A BTC-USDT-PERP vanilla perpetual is collteralized with USDT. So the long position is a synthetic BTC. etc.

Keep in mind that minting and redeeming will maintain the fineness of the coin, where the fineness is defined as MarginBalance / PositionSize of the TP contract.

The TP has 3 status, it also affected by the underlying Perpetual's status.

- Underlying Perpetual is Normal
  - TP.Normal: Can mint, redeem, ERC20 functions are available
  - TP.Paused: Almost all functions are not available (even ERC20.transfer is paused) except unpause or stop
  - TP.Stopped: Can not mint. Can redeem
- Underlying Perpetual is Emergency
  - Can not mint, redeem. ERC20.transfer is available
- Underlying Perpetual is Settled
  - Can not mint, redeem. Can settle. ERC20.transfer is available
  
| Perpetual status | TP status | mint | redeem | settle | ERC20.transfer |
|------------------|-----------|------|--------|--------|----------------|
| Normal           | Normal    | ✔    | ✔     |        | ✔              |
| Normal           | Paused    |      |        |        |                |
| Normal           | Stopped   |      | ✔      |        | ✔              |
| Emergency        | *         |      |         |        | ✔ if not TP.paused |
| GlobalSettled    | *         |      |         | ✔     | ✔ if not TP.paused |

### mint(Amount)

Transfer the collateral into the TP contract, and mint ERC20 tokens.

- Calculate how much the collateral is, while keep the MarginBalance / PositionSize unchanged
  - If the positionSize == 0, the TP must be an empty contract
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
- Transfer DeltaCash from the sender to the TP
- The sender sells Amount positions to the TP at Price. So TP is always openning the long position

Require:

- Not TP.paused
- Not TP.stopped
- Not Perpetual.IsEmergency
- IsSafe == TRUE after calling this function

### redeem(Amount)

Burn ERC20 tokens and transfer the collateral back to the sender.

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
- The sender buys Amount positions to the TP at Price. So TP is always closing the long position
- Transfer DeltaCash from the TP to the sender

Require:

- Not TP.paused
- Not Perpetual.IsEmergency
- IsSafe == TRUE after calling this function

Note: This function works in tp.stopped status

### settle()
