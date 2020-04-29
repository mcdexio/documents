# Internal Design of AMM

<blockquote>:warning: NOTE: This is a technical document of the smart contract. If you don't care how smart contracts are written, please skip this document.</blockquote>

- [Internal Design of AMM](#internal-design-of-amm)
  - [Functions and Motivation](#functions-and-motivation)
  - [Variables](#variables)
    - [Governance](#governance)
    - [Pool storage](#pool-storage)
    - [Pool computed](#pool-computed)
  - [Operations](#operations)
    - [CreatePool(Amount)](#createpoolamount)
    - [BuyFromPool(Amount, LimitPrice, Deadline)](#buyfrompoolamount-limitprice-deadline)
    - [SellToPool(Amount, LimitPrice, Deadline)](#selltopoolamount-limitprice-deadline)
    - [AddLiquidity(Amount)](#addliquidityamount)
    - [RemoveLiquidity(ShareAmount)](#removeliquidityshareamount)
    - [UpdateIndex()](#updateindex)
    - [Funding()](#funding)

## Functions and Motivation

Every Perpetual contract has an AMM (Automated Market Maker). Every AMM has a Margin Account of the Perpetual. This margin account called as "Liquidity Pool" ("PerpetualProxy" in the source code). Liquidity Pool MUST holds long positions (in vanilla contract) and fully-collateralized margin.

Any ETH account can provide liquidity into the Liquidity Pool by calling AddLiquidity() function. After AddLiquidity():
* Liquidity Pool adds some long positions
* The provider adds some short positions
* The provider gets some ShareToken(s) in order to record his liquidity ratio of the whole Liquidity Pool.
* The provider now holds x short positions, and the share tokens imply he/she holds x long positions in the Liquidity Pool. So the net position = 0 which means currently he/she does not have any risk exposure.

## Variables

### Governance
- GovPoolFeeRate: Fee rate of trading volume to share holders
- GovPoolDevFeeRate: Fee rate of trading volume to the Governance Committee
- GovEMAAlpha: The weight of EMA (exponential moving average). EMA_t = GovEMAAlpha * newVal + (1 - GovEMAAlpha) * EMA_(t-1). ex: in order to average samples = 30，alpha = 2/(30+1)=6.45%. We also cache this following values
  - GovEMAAlpha2 = 1 - GovEMAAlpha
  - GovEMAAlpha2Ln = Ln(1 - GovEMAAlpha)
- GovUpdateIndexPrize: The prize (the unit is collateral) of who calling updateIndex()
- GovMarkPremiumLimit: Limit of the PremiumRate when calculating the MarkPrice, ± 0.5%
- GovFundingDampener: Dampener when calculating FundingRate, ± 0.05%

### Pool storage
- IndexPrice: Price from the Oracle
- TotalSupply: Total supply of ShareToken
- BalanceOf: The LP's share balance
- LastEMAPremium: Last EMAPremium when calling funding()
- LastPremium: Last Premium when calling funding()
- LastIndexPrice: Last IndexPrice when calculating the Premium
- LastFundingTime：The timestamp when calculating the AccumulatedFundingPerContract
- AccumulatedFundingPerContract: The accumulated FundingPayment of 1 long position since this Perpetual deployed. The unit is the collateral. AccumulatedFundingPerContract > 0 means that long positions pay. AccumulatedFundingPerContract < 0 means that short positions pay.


### Pool computed
- PoolAvailableMargin:= CashBalance - EntryValue - SocialLoss - FundingLoss
    - NOTE: This formula is different to Perpetual.AvailableMargin
- PositionSize: Long positions count in the Pool. Equal to Perpetual.PositionSize
- FairPrice: PoolAvailableMargin / PositionSize
- LastPremium:= FairPrice - LastIndexPrice
- EMAPremium:
  - n = Now() - LastFundingTime (NOTE: Don't depend on web3's block.time, if you need a realtime EMAPremium, MarkPrice, PremiumRate and FundingRate. Please re-calculate them according to this document）
  - EMAPremium = (LastEMAPremium - LastPremium) * Pow(GovEMAAlpha2, n) + LastPremium
- MarkPrice: LastIndexPrice + EMAPremium, Limited by (LastIndexPrice * (1 ± GovMarkPremiumLimit))
- PremiumRate: (MarkPrice - LastIndexPrice) / LastIndexPrice
- FundingRate: Maximum(GovFundingDampener, PremiumRate) + Minimum(-GovFundingDampener, PremiumRate). The FundingRate is defined as a 8-hour interest rate
- FundingPayment: The collateral paid/received = FundingRate * IndexPrice * PositionSize * HoldPositionTimeSpan / 8hours

## Operations

### CreatePool(Amount)
Create the pool. Can called by anyone.

Require:
- PositionSize == 0 && PoolAvailableMargin == 0

Steps:
- Transfer IndexPrice * Amount * 2 collateral from the sender into LiquidityPool
- Sender trades with LiquidityPool: 
  - Sender is short position
  - LiquidityPool is long position
  - price = IndexPrice
  - amount = Amount
- Mint Amount ShareToken into Sender (implies TotalSupply:+= Amount)
- Funding()

### BuyFromPool(Amount, LimitPrice, Deadline)
The trader buy/long. Can called by anyone.

Steps:
- Calculate the trading price: Price = PoolAvailableMargin / (PositionSize - Amount)
- Sender trades with LiquidityPool:
  - Sender is long position
  - LiquidityPool is short position
  - price = Price
  - amount = Amount
- Fees from sender: PoolFee = GovPoolFeeRate * Price * Amount, DevFee = GovPoolDevFeeRate * Price * Amount
- Funding()

Require:
- broker == LiquidityPool
- BlockTime < DeadLine
- Amount < PositionSize
- Trading Price <= LimitPrice
- After trading, LiquidityPool.IsSafe == TRUE && PoolAvailableMargin > 0
- Sender.IsSafe = True

### SellToPool(Amount, LimitPrice, Deadline)
The trader sell/short. Can called by anyone.

Steps:
- Calculate the trading price: Price = PoolAvailableMargin / (PositionSize + Amount)
- Sender trades with LiquidityPool:
  - Sender is short position
  - LiquidityPool is long position
  - price = Price
  - amount = Amount
- Fees from sender: PoolFee = GovPoolFeeRate * Price * Amount, DevFee = GovPoolDevFeeRate * Price * Amount
- Funding()

Require:
- broker == LiquidityPool
- BlockTime < DeadLine
- Trading Price >= LimitPrice
- After trading, LiquidityPool.IsSafe == TRUE && PoolAvailableMargin > 0
- Trader.IsSafe = True

### AddLiquidity(Amount)
Add liquidity to the LiquidityPool. Can called by anyone.

The unit of "Amount" is contract.

Steps:
- Let Price = PoolAvailableMargin / PositionSize, 
- Mint TotalSupply*Amount/PositionSize ShareToken into the Sender (implies TotalSupply:+= corresponding amount) (this formula satisfies that the increasing ratio to TotalSupply, PoolAvailableMargin, and PositionSize are all the same)
- CollateralAmount = Amount * Price * 2
- Transfer CollateralAmount collateral from the Sender into LiquidityPool 
   - Sender.CashBalance -= CollateralAmount
   - LiquidityPool.CashBalance += CollateralAmount
- Sender trades with LiquidityPool:
  - Sender is short position
  - LiquidityPool is long position
  - price = Price
  - amount = Amount
- Funding()

Require:
- broker == LiquidityPool
- After AddLiquidity, Sender.IsSafe == TRUE
- After AddLiquidity, LiquidityPool.CashBalance > 0
- After AddLiquidity, LiquidityPool.IsSafe == TRUE && PoolAvailableMargin > 0

### RemoveLiquidity(ShareAmount)

Remove liquidity from LiquidityPool. Can called by anyone.

Steps:
- Let Price = PoolAvailableMargin / PositionSize, 
- Let Amount = ShareAmount * PositionSize / TotalSupply
- Send 2 * Price * Amount collateral from LiquidityPool into Sender
- Sender trades with LiquidityPool:
  - Sender is long position
  - LiquidityPool is short position
  - price = Price
  - amount = Amount
- Burn ShareAmount ShareToken from Sender (implies TotalSupply:-= ShareAmount)
- Funding()

Require:
- After RemoveLiquidity, Sender.IsSafe == TRUE
- After RemoveLiquidity, LiquidityPool.IsSafe == TRUE 
- If Pool.PositionSize > 0 THEN PoolAvailableMargin > 0

### UpdateIndex()

Save the IndexPrice from the Oracle in order to let the AMM update-to-date. Can called by anyone.

Steps:
1. If IndexPrice != LastIndexPrice, transfer GovUpdateIndexPrize collateral from Dev into Sender
2. Funding()


### Funding()

Update the FundingRate. Can called by anyone.

Steps:
1. If LastFundingTime == 0 THEN
   - LastFundingTime = BlockTime
   - LastPremium = fairPrice - indexPrice
   - LastEMAPremium = the same as LastPremium
   - LastIndexPrice = IndexPrice
   - end (break)
2. Read the IndexPrice from the Oracle. If the IndexPrice modified, funding() will run twice (equivalent to the calculation of 2 segments of the piecewise function)
   - Let funding variables up-to-date to IndexTimestamp (step 3-5). After that, LastFundingTime:= IndexTimestamp
   - Run step 3-5 again, so that LastFundingTime:= Now()
3. If LastFundingTime != BlockTime THEN 
   1. Calculate AccumulatedFundingRate: The following formula limit and dampener the Funding curve. The 4 points (-GovMarkPremiumLimit, -GovFundingDampener, +GovFundingDampener, +GovMarkPremiumLimit) segment the curve into 5 part, so that the entry price and exit price of the EMA can be arranged into 5 * 5 = 25 cases. In order to reduce the amount of calculation, the code is expanded into 25 branches. Check [Implementation of Funding Rate](internal-amm-funding-rate.md) for details.
     - n:= BlockTime - LastFundingTime (the unit is second)
     - v0 = LastEMAPremium; vt = (LastEMAPremium - LastPremium) * Pow(GovEMAAlpha2, n) + LastPremium
     - vLimit = GovMarkPremiumLimit * LastIndexPrice
     - vDampener = GovFundingDampener * LastIndexPrice
     - T(y) = Log(GovEMAAlpha2, (y-LastPremium)/(v0-LastPremium)), ceiling to integer. This function find the time of the given y on curve
     - R(x, y) = (LastEMAPremium - LastPremium)*(Pow(GovEMAAlpha2, x) - Pow(GovEMAAlpha2, y))/(GovEMAAlpha) + LastPremium * (y-x). This function sums the curve between [x, y) time periods
     - If v0 <= -vLimit:
       - If vt <= -vLimit:
         - Acc = (-vLimit + vDampener) * n
       - ELSE If vt <= -vDampener:
         - t1:= T(-vLimit);
         - Acc = (-vLimit) * t1 + R(t1, n) + vDampener * n
       - ELSE If vt <= +vDampener:
         - t1:= T(-vLimit); t2:= T(-vDampener); 
         - Acc = (-vLimit) * t1 + R(t1, t2) + vDampener * t2
       - ELSE If vt <= +vLimit:    
         - t1:= T(-vLimit); t2:= T(-vDampener); t3:= T(+vDampener);
         - Acc = (-vLimit) * t1 + R(t1, t2) + R(t3, n) + vDampener * (t2 - (n - t3))
       - ELSE:    
         - t1:= T(-vLimit); t2:= T(-vDampener); t3:= T(+vDampener); t4:= T(+vLimit);
         - Acc = (-vLimit) * t1 + R(t1, t2) + R(t3, t4) + vLimit * (n - t4) + vDampener * (t2 - (n - t3))
     - If v0 <= -vDampener:
       - If vt <= -vLimit:
         - t4:= T(-vLimit);
         - Acc = R(0, t4) + (-vLimit) * (n - t4) + vDampener * n
       - ELSE If vt <= -vDampener: 
         - Acc = R(0, n) + vDampener * n
       - ELSE If vt <= +vDampener:
         - t2:= T(-vDampener); 
         - Acc = R(0, t2) + vDampener * t2
       - ELSE If vt <= +vLimit:    
         - t2:= T(-vDampener); t3:= T(+vDampener);
         - Acc = R(0, t2) + R(t3, n) + vDampener * (t2 - (n - t3))
       - ELSE:    
         - t2:= T(-vDampener); t3:= T(+vDampener); t4:= T(+vLimit);
         - Acc = R(0, t2) + R(t3, t4) + vLimit * (n - t4) + vDampener * (t2 - (n - t3))
     - If v0 <= +vDampener:
       - If vt <= -vLimit:
         - t3:= T(-vDampener); t4:= T(-vLimit);
         - Acc = R(t3, t4) + (-vLimit) * (n - t4) + vDampener * (n - t3)
       - ELSE If vt <= -vDampener:      
         - t3:= T(-vDampener);
         - Acc = R(t3, n) + vDampener * (n - t3)
       - ELSE If vt <= +vDampener:
         - Acc = 0
       - ELSE If vt <= +vLimit:
         - t3:= T(+vDampener);
         - Acc = R(t3, n) - vDampener * (n - t3)
       - ELSE:
         - t3:= T(+vDampener); t4:= T(+vLimit);
         - Acc = R(t3, t4) + vLimit * (n - t4) - vDampener * (n - t3)
     - If v0 <= +vLimit:
       - If vt <= -vLimit:
         - t2:= T(+vDampener); t3:= T(-vDampener); t4:= T(-vLimit);
         - Acc = R(0, t2) + R(t3, t4) + (-vLimit) * (n - t4) + vDampener * (n - t3 - t2)
       - ELSE If vt <= -vDampener:
         - t2:= T(+vDampener); t3:= T(-vDampener);
         - Acc = R(0, t2) + R(t3, n) + vDampener * (n - t3 - t2)
       - ELSE If vt <= +vDampener:
         - t2:= T(+vDampener);
         - Acc = R(0, t2) - vDampener * t2
       - ELSE If vt <= +vLimit:
         - Acc = R(0, n) - vDampener * n
       - ELSE:
         - t4:= T(+vLimit);
         - Acc = R(0, t4) + vLimit * (n - t4) - vDampener * n
     - Else:
       - If vt <= -vLimit:                      
         - t1:= T(+vLimit); t2:= T(+vDampener); t3:= T(-vDampener); t4:= T(-vLimit);
         - Acc = vLimit * t1 + R(t1, t2) + R(t3, t4) + (-vLimit) * (n - t4) + vDampener * (n - t3 - t2)
       - ELSE If vt <= -vDampener:
         - t1:= T(+vLimit); t2:= T(+vDampener); t3:= T(-vDampener);
         - Acc = vLimit * t1 + R(t1, t2) + R(t3, n) + vDampener * (n - t3 - t2)
       - ELSE If vt <= +vDampener:
         - t1:= T(+vLimit); t2:= T(+vDampener);
         - Acc = vLimit * t1 + R(t1, t2) - vDampener * t2
       - ELSE If vt <= +vLimit:
         - t1:= T(+vLimit);
         - Acc = vLimit * t1 + R(t1, n) - vDampener * n
       - ELSE:
         - Acc = (vLimit - vDampener) * n
   2. AccumulatedFundingPerContract:+= Acc / (8*3600)
   3. LastFundingTime = BlockTime
   4. LastEMAPremium = vt
4. LastPremium = FairPrice - IndexPrice
5. LastIndexPrice = IndexPrice

Require:
  - IsEmergency == FALSE
