# Admin Functions

Only admin can call the following functions. The main purpose includes:
* Change the governance parameters
* Upgrade contract
* Global liquidation, including:
  * Switch into "Emergency" status which (1) stops tradings and withdraws, (2) sets the global settlement price
  * Correct hacked (ex: Oracle price hack) data in "Emergency" status

:warning: **Due to the importance of global liquidation, MCDEX will establish a community-led governance committee as soon as possible, and the committee will develop detailed global liquidation trigger mechanisms and processing procedures.**

## Perpetual

* Perpetual.addWhitelistAdmin: Add another admin
* Perpetual.addWhitelisted: Add a new Exchange/AMM contract that can buy/sell from this Perpetual
* Perpetual.setGovernanceParameter: Modify the Perpetual's parameters including:
  * initialMarginRate: Minimum margin balance rate when opening positions
  * maintenanceMarginRate: Minimum margin balance rate to prevent liquidation
  * liquidationPenaltyRate: The penalty rate that gives to the keeper when liquidation happens
  * penaltyFundRate: The penalty rate that gives to the insurance fund when liquidation happens
  * takerDevFeeRate: Taker fee rate that gives to the developer when using Exchange
  * makerDevFeeRate: Maker fee rate that gives to the developer when using Exchange
  * lotSize: Minimum position size
  * tradingLotSize: Minimum position size when trading
  * longSocialLossPerContracts: Social loss per each long position. Can only be called in the "Emergency" status
  * shortSocialLossPerContracts: Social loss per each short position. Can only be called in the "Emergency" status
* Perpetual.setGovernanceAddress: Modify the Perpetual's parameters including:
  * amm: Set the AMM contract address
  * globalConfig: Set the GlobalConfig contract address
  * dev: Set the developer address that gathering fee from tradings
* Perpetual.beginGlobalSettlement: Enter the "Emergency" status with a "settlement price". In this status, all trades and withdrawals will be disabled until "endGlobalSettlement"
* Perpetual.setCashBalance: Modify account.cashBalance. Can only be called in the "global settlement" status
* Perpetual.endGlobalSettlement: Enter the "global settlement" status. In this status, all traders can withdraw their MarginBalance
* Perpetual.withdrawFromInsuranceFund: Withdraw insurance fund. Typically happen in the "global settlement" status

## AMM

* AMM.addWhitelistAdmin: Add another admin
* AMM.addWhitelisted: Add a new Exchange contract that can buy/sell from this AMM
* AMM.setGovernanceParameter: Modify the AMM's parameters including:
  * poolFeeRate: The fee rate that gives to the shares holder when using AMM
  * poolDevFeeRate: The fee rate that gives to the developer when using AMM
  * emaAlpha: The exponential moving average (EMA) parameter that is used to calculate the MarkPrice
  * updatePremiumPrize: Anyone can call AMM.updateIndex to keep the IndexPrice up-to-date with the oracle and get this prize
  * markPremiumLimit: The MarkPrice is always around the IndexPrice · (1 ± markPremiumLimit). Note that maximum FundingRate is also controlled by this parameter
  * fundingDampener: A reasonable trading price range. If the FairPrice is inner IndexPrice · (1 ± fundingDampener), FundingRate will be 0

## Global Config

* GlobalConfig.setGlobalParameter: Modify the global parameters including:
  * withdrawalLockBlockCount: A trader can withdraw his/her MarginBalance after applying and wait for withdrawalLockBlockCount blocks
  * brokerLockBlockCount: A trader can change his/her Broker after applying and wait for brokerLockBlockCount blocks
