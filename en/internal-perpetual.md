# Internal Design of Perpetual

:warning: NOTE: This is a technical document of the smart contract. If you don't care how smart contracts are written, please skip this document.

- [Internal Design of Perpetual](#internal-design-of-perpetual)
  - [Functions and Motivation](#functions-and-motivation)
  - [Variables](#variables)
    - [Global Governance](#global-governance)
    - [Perpetual Governance](#perpetual-governance)
    - [Perpetual constant](#perpetual-constant)
    - [Perpetual storage](#perpetual-storage)
    - [Perpetual computed](#perpetual-computed)
    - [MarginAccount storage](#marginaccount-storage)
    - [MarginAccount computed](#marginaccount-computed)
  - [Operations](#operations)
    - [Deposit(CollateralAmount)](#depositcollateralamount)
    - [ApplyForWithdrawal(CollateralAmount)](#applyforwithdrawalcollateralamount)
    - [Withdraw(CollateralAmount)](#withdrawcollateralamount)
    - [Remargin(guy)](#remarginguy)
    - [Buy(Trader, Price, Amount) / Sell(Trader, Price, Amount)](#buytrader-price-amount--selltrader-price-amount)
    - [Liquidate(Account, MaxAmount)](#liquidateaccount-maxamount)
    - [BeginGlobalSettle(price)](#beginglobalsettleprice)
    - [GlobalSettled()](#globalsettled)
    - [Settle()](#settle)
    - [SetBroker(newBroker)](#setbrokernewbroker)


## Functions and Motivation

The "Perpetual" contract is used to manage margin accounts, positions, and PnL calculations.

There're 3 status: Normal, Emergency and GlobalSettled.
* Normal: In this status, long positions MUST always equal to short positions. ie: Unpaired position never exists.
* Emergency: If there are too many bad assets, the admin can switch Perpetual into this status.
  * Trading is NOT allowed
  * Withdraw is NOT allowed
  * Liquidate is allowed
  * Deposit is allowed in order to prevent himself from liquidation
  * Admin can modify the cashBalance and socialLoss if hacks detected (ex: oracle hack)
* GlobalSettled: Traders can now withdraw

## Variables

### Global Governance
 - GovWithdrawalLockBlockCount: Lock delay (block number) before withdraw
 - GovBrokerLockBlockCount: Lock delay (block number) before broker/relayer takes effect. This number includes the block that calls SetBroker(). ex: If GovBrokerLockBlockCount = 0, the SetBroker() takes effect immediately

### Perpetual Governance
 - GovInitialMarginRate: Initial margin rate
 - GovMaintenanceMarginRate: Maintenance margin rate
 - GovLiquidationPenaltyRate: Liquidation penalty rate of the liquidated assets to Liquidator/Keeper
 - GovPenaltyFundRate: Liquidation penalty rate of the liquidated assets to the InsuranceFund
 - GovTakerDevFeeRate: Taker fee rate of the trading volume when using the Exchange. In order to make ClosePosition easier, when closing a position, if the trader's balance is not enough, the trading fee will be limited to the balance. NOTE: trading with AMM does not use GovTakerDevFeeRate
 - GovMakerDevFeeRate: Maker fee rate of the trading volume when using the Exchange. In order to make ClosePosition easier, when closing a position, if the trader's balance is not enough, the trading fee will be limited to the balance.

NOTE: require GovMaintenanceMarginRate < GovInitialMarginRate
NOTE: require GovLiquidationPenaltyRate < GovMaintenanceMarginRate
NOTE: require GovPenaltyFundRate < GovMaintenanceMarginRate

### Perpetual constant
 - GovTradingLotSize: Amount unit when Buy/Sell. ex: If GovTradingLotSize = 10, you can only buy/sell 10, 20, ...
 - GovLotSize: PositionSize always limit to this unit. NOTE: Buy, Sell, Liquidate, RemoveLiquidity, everywhere in Perpetual should obey this constraint

NOTE: require GovTradingLotSize = GovLotSize * PositiveIntegers

### Perpetual storage

- TotalSize: Open interest
- LongSocialLossPerContract: Socialized losses per contract of long positions
- ShortSocialLossPerContract: Socialized losses per contract of short positions
- IsEmergency: Whether the current status is Emergency
- IsGlobalSettled: Whether the current status is GlobalSettled
- GlobalSettlePrice: In Emergency and GlobalSettled status, take this price as MarkPrice

### Perpetual computed
- MarkPrice: Come from AMM if it's Normal status, GlobalSettlePrice otherwise

### MarginAccount storage
- AccountOwner: MarginAccount's owner
- CashBalance: Deposited collateral
- WithdrawalApplicationAmount: Amount of collateral application for withdrawal
- WithdrawalApplicationHeight: Block height that the owner can withdraw
- PositionSide: Position side. Flat = 0, ShortPosition = 1, LongPosition = 2
- PositionSize: Position size. Always positive
- EntryValue: SUM(EntryPrice when opening position * position size)
- EntrySocialLoss: SUM(SocialLossPerContract when opening position * position size)
- EntryFundingLoss: SUM(AccumulatedFundingPerContract when opening position * position size)

### MarginAccount computed
- InitialMarginRate (IMRate):= GovInitialMarginRate
- MaintenanceMarginRate:= GovMaintenanceMarginRate
- AvgEntryPrice:=
  - PositionSize > 0: EntryValue / PositionSize. Note that this value is inexhaustible, which introduces errors. So we need delay the division until the final calculation
  - PositionSize == 0: Meaningless (The formulas that reference to AvgEntryPrice only read AvgEntryPrice when PositionSize > 0), return 0 here
- PositionMargin:= MarkPrice * PositionSize * IMRate
- MaintenanceMargin:= MarkPrice * PositionSize * MMRate
- SocialLoss
  - Long:  LongSocialLossPerContract * PositionSize - EntrySocialLoss
  - Short: ShortSocialLossPerContract * PositionSize - EntrySocialLoss
- FundingLoss
  - Long:  (AccumulatedFundingPerContract * PositionSize - EntryFundingLoss)
  - Short: -(AccumulatedFundingPerContract * PositionSize - EntryFundingLoss)
- PNL1
  - Long: (MarkPrice - AvgEntryPrice) * PositionSize = MarkPrice * PositionSize - EntryValue
  - Short: (AvgEntryPrice - MarkPrice) * PositionSize = EntryValue - MarkPrice * PositionSize
- PNL2
  - PNL1 - SocialLoss - FundingLoss
- EstimatedLiquidationPrice: The estimated price that an account will be Liquidate(d). This value is only useful on the website
  - In order to keep IsSafe = True, let MarginBalance == MaintenanceMargin, so we can get the boundary conditions
  - Note: EstimatedLiquidationPrice is meaningless when PositionSize == 0
  - Long:  (CashBalance - EntryValue - SocialLoss - FundingLoss) / (PositionSize * MMRate - PositionSize)
  - Short: (CashBalance + EntryValue - SocialLoss - FundingLoss) / (PositionSize * MMRate + PositionSize)
- MarginBalance:= CashBalance + PNL2
- AvailableMargin (the balance that can open new position) := MarginBalance - PositionMargin - WithdrawalApplicationAmount
- WithdrawableBalance:= MIN(MarginBalance - PositionMargin, WithdrawalApplicationAmount)
- IsSafe:= MarginBalance >= MaintenanceMargin
- IsBankrupt:= MarginBalance < 0

## Operations

### Deposit(CollateralAmount)
Transfer collateral from the wallet into the contract. Can only called by AccountOwner.

NOTE: CollateralAmount.decimals = collateral token's decimals. ex: ETH.decimals = 18, but some token.decimals = 6 or 8.

 - AccountOwner:= sender
 - CashBalance:+= CollateralAmount

### ApplyForWithdrawal(CollateralAmount)
Apply for withdrawal. Calling this function overwrites the previous calling. Can only called by AccountOwner.

NOTE: CollateralAmount.decimals = collateral token's decimals. ex: ETH.decimals = 18, but some token.decimals = 6 or 8.

  - WithdrawalApplicationAmount:= CollateralAmount
  - WithdrawalApplicationHeight:= CurrentBlockHeight()

### Withdraw(CollateralAmount)
Withdraw from MarginAccount into the wallet. Can only called by AccountOwner.

NOTE: CollateralAmount.decimals = collateral token's decimals. ex: ETH.decimals = 18, but some token.decimals = 6 or 8.

Require:
  - IsEmergency == FALSE && IsSafe == TRUE
  - CollateralAmount <= WithdrawableBalance 
  - CurrentBlockHeight() - WithdrawalApplicationHeight >= GovWithdrawalLockBlockCount
  - IsSafe == TRUE after calling this function

Steps:
  - Funding()
  - Remargin(sender) in order to realize PNL
  - Transfer from the contract into user
  - CashBalance:-= CollateralAmount

### Remargin(guy)
Re-calculate the margin requirement of the current MarginAccount, realize PNL into CashBalance. This function is  equivalent to close and re-open the position at current MarkPrice. 

IMPORTANT NOTE: AMM's account (LiquidityPool) MUST never call Remargin until Emergency status

  - CashBalance:+= PNL2
  - EntryValue:= MarkPrice * PositionSize, literally means AvgEntryPrice:= MarkPrice
  - EntrySocialLoss:=
    - Long:  LongSocialLossPerContract * PositionSize
    - Short: ShortSocialLossPerContract * PositionSize
  - EntryFundingLoss:= AccumulatedFundingPerContract * PositionSize

### Buy(Trader, Price, Amount) / Sell(Trader, Price, Amount)
Trade in positions. Only called by whitelist (ie: Exchange, AMM).

According to the positions of the two parties of the trade, handle open / close position operations. If any party is operating the reverse position (ex: current PositionSize = 10 long, now short 20), we have to first close their positions, and open reverse positions.

Steps to OpenPositions:
- Funding()
- PositionSide may change only if PositionSize == 0, PositionSide == "flat"
- EntryValue:+= Price * Amount
- PositionSize:+= Amount
- EntrySocialLoss:+=
  - Long: LongSocialLossPerContract * Amount
  - Short: ShortSocialLossPerContract * Amount
- EntryFundingLoss:+= AccumulatedFundingPerContract * Amount

Steps to ClosePositions:
- Funding()
- RPNL1:
  - Long: (Price - AvgEntryPrice) * Amount = Price * Amount - EntryValue * Amount / PositionSize
  - Short: (AvgEntryPrice - Price) * Amount = EntryValue * Amount / PositionSize - Price * Amount 
- SocialLoss:=
  - Long: (LongSocialLossPerContract - EntrySocialLoss / PositionSize) * Amount
  - Short: (ShortSocialLossPerContract - EntrySocialLoss / PositionSize) * Amount
- FundingLoss
  - Long:  (AccumulatedFundingPerContract - EntryFundingLoss / PositionSize) * Amount
  - Short: -(AccumulatedFundingPerContract - EntryFundingLoss / PositionSize) * Amount
- RPNL2:= RPNL1 - SocialLoss - FundingLoss
- EntrySocialLoss:= EntrySocialLoss / PositionSize * (PositionSize - Amount)
- EntryFundingLoss:= EntryFundingLoss / PositionSize * (PositionSize - Amount)
- CashBalance:+= RPNL2
- EntryValue:= EntryValue / PositionSize * (PositionSize - Amount) 
- PositionSize:-= Amount，please require PositionSize >= 0
- if PositionSize == 0, PositionSide == "flat"

Finally:
- TotalSize
  - If both parties are opening positions: TotalSize:+= Amount
  - If both parties are closing positions: TotalSize:-= Amount
  - Moves positions from A to B: TotalSize is not changed
  - Moves positions from B to A: TotalSize is not changed

Require:
  - IsEmergency == FALSE
  - After trading, PositionSize of both Buyer and Seller >= 0
  - After trading, Buyer and Seller are both IsSafe == TRUE
  - If opening position, AvailableMargin >= 0 (the old positions should also keep the IM)


### Liquidate(Account, MaxAmount)

Anyone can call Liquidate() to another account that IsSafe == FALSE to implement the liquidation. The caller/Keeper will get the penalty collateral, and trade to the liquidated account in order to move his/her positions into the Keeper's account. NOTE: the liquidated party can only close position.

Require:
- Account.IsSafe == False 

- LiquidationPrice:
  - If IsEmergency: GlobalSettlementPrice
  - ELSE: MarkPrice

1. Funding()

2. Calculate LiquidationAmount

In order to prevent from removing all positions from the liquidated account, calculate the minimum amount that lead to MarginBalance >= PositionMargin. Let the minimum amount to liquidate = X:

- Penalty:= (GovLiquidationPenaltyRate + GovPenaltyFundRate) * LiquidationPrice * X
- RPNL1:
  - Long: (LiquidationPrice - AvgEntryPrice) * X = LiquidationPrice * X - EntryValue * X / PositionSize
  - Short: (AvgEntryPrice - LiquidationPrice) * X = EntryValue * X / PositionSize - LiquidationPrice * X
- SocialLoss
  - Long: (LongSocialLossPerContract - EntrySocialLoss / PositionSize) * X
  - Short: (ShortSocialLossPerContract - EntrySocialLoss / PositionSize) * X
- FundingLoss
  - Long:  (AccumulatedFundingPerContract - EntryFundingLoss / PositionSize) * X
  - Short: -(AccumulatedFundingPerContract - EntryFundingLoss / PositionSize) * X
- RPNL2:= RPNL1 - SocialLoss - FundingLoss

- NewEntrySocialLoss:= EntrySocialLoss / PositionSize * (PositionSize - X)
- NewEntryFundingLoss:= EntryFundingLoss / PositionSize * (PositionSize - X)
- NewCashBalance:= CashBalance + RPNL2 - Penalty
- NewEntryValue:= EntryValue / PositionSize * (PositionSize - X) 
- NewPositionSize:= PositionSize - X，please require NewPositionSize >= 0
- NewPositionMargin:= LiquidationPrice * NewPositionSize * IMRate

- NewSocialLoss
  - Long:  LongSocialLossPerContract * NewPositionSize - NewEntrySocialLoss
  - Short: ShortSocialLossPerContract * NewPositionSize - NewEntrySocialLoss
- NewFundingLoss
  - Long:  (AccumulatedFundingPerContract * NewPositionSize - NewEntryFundingLoss)
  - Short: -(AccumulatedFundingPerContract * NewPositionSize - NewEntryFundingLoss)
- NewPNL1:= CalPNL(AvgEntryPrice, LiquidationPrice, NewPositionSize)
  - Long: LiquidationPrice * NewPositionSize - NewEntryValue 
  - Short: NewEntryValue - LiquidationPrice * NewPositionSize
- NewPNL2:= NewPNL1 - NewSocialLoss - NewFundingLoss
- NewMarginBalance:= NewCashBalance + NewPNL2 = 
  - Long:  CashBalance - EntryValue 
         + EntrySocialLoss + EntryFundingLoss
         - AccumulatedFundingPerContract * PositionSize
         + LiquidationPrice * PositionSize
         - LongSocialLossPerContract * PositionSize 
         - (GovLiquidationPenaltyRate + GovPenaltyFundRate) * LiquidationPrice * X
  - Short: CashBalance + EntryValue
         + EntrySocialLoss - EntryFundingLoss
         + AccumulatedFundingPerContract * PositionSize
         - LiquidationPrice * PositionSize
         - PositionSize * ShortSocialLossPerContract
         - (GovLiquidationPenaltyRate + GovPenaltyFundRate) * LiquidationPrice * X
- NewIsSafe:= NewMarginBalance >= NewMaintenanceMargin 

  Solve the NewIsSafe == True equation, we can get X

  - Long:  X >= (CashBalance - EntryValue
                + EntrySocialLoss + EntryFundingLoss
                - AccumulatedFundingPerContract PositionSize
                + LiquidationPrice * PositionSize
                - LiquidationPrice * IMRate * PositionSize
                - LongSocialLossPerContract * PositionSize)
                / (LiquidationPrice * (GovLiquidationPenaltyRate + GovPenaltyFundRate - IMRate))

  - Short: X >= (CashBalance + EntryValue
                + EntrySocialLoss - EntryFundingLoss
                + AccumulatedFundingPerContract * PositionSize
                - LiquidationPrice * PositionSize
                - LiquidationPrice * IMRate * PositionSize
                - PositionSize * ShortSocialLossPerContract)
                / (LiquidationPrice * (GovLiquidationPenaltyRate + GovPenaltyFundRate - IMRate))

- LiquidationAmount:= Clip X to [0, PositionSize]

1. Liquidate
- Run ClosePosition steps with amount = LiquidationAmount
  - RPNL1: CalcPNL(AvgEntryPrice, LiquidationPrice, LiquidationAmount)
    - Long: (LiquidationPrice - AvgEntryPrice) * LiquidationAmount = LiquidationPrice * LiquidationAmount - EntryValue * LiquidationAmount / PositionSize
    - Short: (AvgEntryPrice - LiquidationPrice) * LiquidationAmount = EntryValue * LiquidationAmount / PositionSize - LiquidationPrice * LiquidationAmount 
  - SocialLoss:=
    - Long: (LongSocialLossPerContract - EntrySocialLoss / PositionSize) * LiquidationAmount
    - Short: (ShortSocialLossPerContract - EntrySocialLoss / PositionSize) * LiquidationAmount
  - FundingLoss
    - Long:  (AccumulatedFundingPerContract - EntryFundingLoss / PositionSize) * LiquidationAmount
    - Short: -(AccumulatedFundingPerContract - EntryFundingLoss / PositionSize) * LiquidationAmount
  - RPNL2:= RPNL1 - SocialLoss - FundingLoss
  - Penalty:= (GovLiquidationPenaltyRate + GovPenaltyFundRate) * LiquidationPrice * LiquidationAmount
  - EntrySocialLoss:= EntrySocialLoss / PositionSize * (PositionSize - LiquidationAmount)
  - EntryFundingLoss:= EntryFundingLoss / PositionSize * (PositionSize - LiquidationAmount)
  - CashBalance + RPNL2 - Penalty >= 0
    - Yes: 
      - CashBalance +:= RPNL2 - Penalty
      - LiquidationLoss:= 0
    - No:
      - CashBalance:= 0
      - LiquidationLoss:= -(CashBalance + RPNL2 - Penalty)
  - EntryValue:= EntryValue * (PositionSize - LiquidationAmount) / PositionSize
  - PositionSize:-= LiquidationAmount，please require PositionSize >= 0

- Run the counter-party steps to the Liquidator:
  - Long: Buy(Liquidator, LiquidationPrice, LiquidationAmount)
  - Short: Sell(Liquidator, LiquidationPrice, LiquidationAmount)

2. Send the penalty
  - InsuranceFund:+= LiquidationPrice * LiquidationAmount * GovPenaltyFundRate
  - Liquidator.CashBalance:+= LiquidationPrice * LiquidationAmount * GovLiquidationPenaltyRate

3. TotalSize: Liquidated account can only ClosePosition, so TotalSize
  - Liquidator is opening positions: TotalSize not changed
  - Liquidator is closing positions：TotalSize:-= LiquidationAmount

4. Handle the loss
   - If InsuranceFund.CashBalance >= LiquidationLoss
     - LiquidationLoss:-=InsurraceFund
   - Else:
     - CollateralInInsuranceFund:= InsuranceFund
     - InsurraceFund:= 0
     - SocialLoss:= LiquidationLoss - CollateralInInsuranceFund
     - SocialLossPerContract:= SocialLoss / TotalSize
     - If Long: LongSocialLossPerContract += SocialLossPerContract
     - If Short: ShortSocialLossPerContract += SocialLossPerContract
   
### BeginGlobalSettle(price)
Can only called by Admin. Stops all trades and withdraws. The admin can call this function more than once if the price is incorrect.

 - IsEmergency:= TRUE
 - GlobalSettlementPrice:= price
 - Remargin(LiquidityPool). So that LiquidityPool.PositionSize = 0

### GlobalSettled()
Can only called by Admin. Stops all trades, but can withdraw.

- IsGlobalSettled:= TRUE

Require:
- IsEmergency:= TRUE

### Settle()
Call this function to withdraw CashBalances after IsGlobalSettlement == True.

Require:
  - IsGlobalSettled:= TRUE

Steps：
  - CashBalance:=
    - IF CashBalance + PNL2 > 0: CashBalance + PNL2
    - ELSE: 0
  - PositionSize:= 0
  - Transfer all CashBalance into user's wallet

### SetBroker(newBroker)
Set the broker. The broker takes effect after GovBrokerLockBlockCount blocks (including the block that calls SetBroker()). 

The possible broker can be:
* Liquidity Pool: Indicate that the user wants to trade with AMM
* Exchange Relayer: A relayer is used by an off-chain order book to send the Match information to on-chain domain. This kind of broker is a common ETH address.
