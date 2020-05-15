# Contract interfaces

## Perpetual

**Interact with collaterals (cash balance):**

```solidity
getCashBalance(address guy)
```

Return cash balance storage variables of guy, which is defined as:

```solidity
struct CollateralAccount {
    // currernt deposited erc20 token amount, representing in decimals 18
    int256 balance;
    // the amount of withdrawal applied by user
    // which allowed to withdraw in the future but not available in trading
    int256 appliedBalance;
    // applied balance will be appled only when the block height below is reached
    uint256 appliedHeight;
}
```

See "Broker & Withdraw Time Lock" section in the [references page](https://mcdex.io/references/Perpetual#trade-with-the-order-book) for the design of time lock.

```solidity
deposit(uint256 amount) NORMAL
```
Deposit transfer collaterals from caller's address to contract. It accept an amount parameter which indicates the total amount of collaterals that user wants to transfer to contract.
Approval is required;

**amount should be a fixed float with token's decimals which will be convert to a decimals-18 representation. E.G. Jim deposits 1e6 USDT, later he will found 1e18 collaterals in his account of Mai protocol v2. This only affects internal calculation*

```solidity
depositEther() NORMAL
```

Ether version of deposit, using msg.value instead. See description above for details. When using ether, the decimals will be automatically set to 18.

```solidity
applyForWithdrawal(uint256 amount) NORMAL
```

Request for further withdrawal of caller's account. This method functions like approve of erc20. Trader could apply for the amount that far greater than actual balance he owned, but applied part is no longer available for position trading.

```solidity
withdraw(uint256 amount) NORMAL
```

Withdraw given amount collateral back to caller's ethereum address. Note that withdraw doesn't distinguish between erc20 and ether.

```solidity
settle() SETTLED
```

Settle is a special version of withdraw which is only available when global settlement is done. It will close all opened position of caller then drain the remaining collaterals to caller.

Settle can be call multiple times but only the first successful call will actually do the job.


```solidity
depositToInsuranceFund(uint256 amount)
depositEtherToInsuranceFund()
```

Deposit collaterals from caller's address to contract. The funds will not go to any account but recorded by an variable in perpetual.

-----

**Interact with positions:**

There do exists interface to manipulate position and collateral for trading purpose, but basically the perpetual contract does not directly handle trading request from user. It only provider interfaces for whitelisted callers like **exchange** and **amm** contract.

Calling trade method of perpetual may break constraints that long position should always equals to short positions.

The only interface available to a typical trader is:

```solidity
getPosition(address guy)
```

Return position storage variables of guy, which is defined as:

```solidity
struct PositionAccount {
    LibTypes.Side side;
    uint256 size;
    uint256 entryValue;
    int256 entrySocialLoss;
    int256 entryFundingLoss;
}
```


```solidity
totalSize(Side side)
```

Return total size of positions. Side has 3 available value: FLAT(0), SHORT(1) and LONG(2).

*The constraint is that totalSize(SHORT) should always be equal to totalSize(LONG) and totalSize(FLAT) should always be zero.*

```solidity
socialLossPerContract(Side side)
```

Return social loss per contract of given side. Normally, the value of each side should never decrease.

```solidity
positionMargin(address guy)
```

Methods of calculating position value requires mark price from amm contract. This price will be replaced by settlement price set by administrator in settlement status (SETTLING, SETTLED).

```solidity
maintenanceMargin(address guy)
```

Return maintenance margin value of account base on current mark price. The value should always be lower than initial margin value.

If the current value of positions falls below maintenance margin, the account will go into a status called 'unsafe'. An unsafe account can be liquidated by any trader within the same perpetual to prevent possible loss.

When the value falls below zero, the account is 'bankrupt'.

```solidity
pnl(address guy)
```

Return profit and loss of account base on current mark price. This value would be added to balance on remargin, which is usually happens on withdrawal and settlement.

```solidity
availableMargin(address guy)
```

Return available margin of account base on current mark price.

The pnl is already included.

```solidity
drawableBalance(address guy)
```

Return drawable balance of account base on current mark price. Trader could get balance through apply-withdraw progress.

The pnl is already included.

```solidity
isIMSafe(address guy)       // is initial margin safe.
isSafe(address guy)         // is maintainance margin safe;
isBankrupt(address guy)
```

These three test method is used to test current status of an account.

```solidity
insuranceFundBalance()
```

Insurance fund is claimed from liquidation penalty and will be used to recover social loss.


```soldiity
liquidate(address guy, uint256 maxAmount) public returns (uint256, uint256) NORMAL SETTLING
```

Liquidate bites a part of position from an unsafe account (See isSafe for definition of unsafe) to make it safe again. The call should have enough margin to hold the bitten positions or the call will fail.

All losses generated by liquidate will be removed from insurance fund first if possible, or the loss will become social loss.

-----

**Interact with brokers:**

Broker is a actor who helps trader to accomplish orderbook trading. Broker is set by trader, with a few blocks delay.

To receive trading fee, a broker must assign positive maker/taker fee rate in order data structure. And if given a negative trading fee, the broker will pay trader for making / taking order.

There are two typically value of broker:

- broker is set to a contract call who calls match method of exchange methods (orderbook trading);
- broker is set to address of amm (perpetual proxy actually) to enable amm trading.

Trader cannot apply both the trading modes above since the broker variable can hold exactly one address at the same time.

There is a delay mechanism on setting broker address, see "Broker & Withdraw Time Lock" section in the [references page](https://mcdex.io/references/Perpetual#trade-with-the-order-book) for the design of time lock.

Perpetual contract provider interface of setting brokers, but the applying delay is determined by global config.

```solidity
currentBroker(address trader)
```

Return current broker's address of given trader address. If last setting is not applied, this function will return last applied broker address.


```solidity
setBroker(address broker)
```

Set caller's broker to given account.

```solidity
getBroker(address trader)
```

Return broker storage of given account. For normal case, trader should call currentBroker instead.

-----

## Exchange

Exchange protocol dose exchange between one taker and some makers. Traders sign for their order content and another actor named broker call match method for them, claiming trading fee from both side.

```solidity
    struct OrderParam {
        address trader;
        uint256 amount;
        uint256 price;
        bytes32 data;
        OrderSignature signature;
    }

    struct Order {
        address trader;
        address broker;
        address perpetual;
        uint256 amount;
        uint256 price;
        bytes32 data;
    }

    matchOrders(
        OrderParam memory takerOrderParam,
        OrderParam[] memory makerOrderParams,
        address perpetual,
        uint256[] memory amounts
    )
```

Length of parameter 'amounts' should equal to the length of 'makerOrderParams'.

The matchOrders methods will try to match order from taker with each order from makers. If matched, taker will receive positions from maker, and collateral is paid to maker at the price maker asking for. Matched amount is exactly the same  as the amount in parameter 'amounts'.

Some properties is encoded into data field:

| Name           | Length (bytes) | Description                                                  |
| -------------- | -------------- | ------------------------------------------------------------ |
| Version        | 1              | supported protocol version. 1 = Mai Protocol V1, 2 = Mai Protocol V2 |
| Side           | 1              | Side of order. 1 = short, other = long                       |
| isMarketOrder  | 1              | 0 = limit order, 1 = market order                            |
| expiredAt      | 5              | order expiration time in seconds                             |
| asMakerFeeRate | 2              | (int16) maker fee rate (base 100,000)                        |
| asTakerFeeRate | 2              | (int16) taker fee rate (base 100,000)                        |
| deprecated     | 2              |                                                              |
| salt           | 8              | salt                                                         |
| isMakerOnly    | 1              | is maker only order                                          |
| isInversed     | 1              | is for inversed contract. if true, price and side will be inversed in matching |
| reserved       | 8              |                                                              |


```solidity
    matchOrderWithAMM(LibOrder.OrderParam memory takerOrderParam, address _perpetual, uint256 amount)
```

Match taker orders with amm. The main difference between this method and amm trading methods is that they require different broker setting.

**Currently, broker CAN NOT profit from calling matchOrderWithAMM for trader like matchOrders. This method is designed for other purpose**

## AMM

AMM has some Uniswap-like interfaces which allows trader to trade with internal assets pool.

```solidity
    createPool(uint256 amount)
```

Open asset pool by deposit to amm. Only available when pool is empty.

```solidity
    buy(uint256 amount, uint256 limitPrice, uint256 deadline)
```

Buy position from amm. It could be open or close or both based on which side of position a trader owns.

LimitPrice is the upperbound of bid price. Deadline is a unix-timestamp in seconds. Any unsatisfied condition will fail trading transaction.

```solidity
    sell(uint256 amount, uint256 limitPrice, uint256 deadline)
```

Similar to buy, but limitPrice is the lowerbond of bid price.

```solidity
    addLiquidity(uint256 amount)
```

Add liquidity to amm asset pool. See design of amm for details.


```solidity
    removeLiquidity(uint256 shareAmount)
```

Remove liquidity to amm asset pool. See design of amm for details.


```solidity
    settleShare()
```

A special method to remove liquidity only works in settled status. Use a different equation to calculate how much collateral should be returned for a share.

```solidity
    updateIndex()
```

Update index variable in amm. Caller will get some reward determined by governance parameter.

```solidity
    shareTokenAddress()
```

Return address of share token for current amm. One deployed instance of share token is only available to one amm contract.

```solidity
    indexPrice()
```

Return index read from oracle, updated through updateIndex call.

```solidity
    currentFairPrice()
    positionSize()
    currentAvailableMargin()
```

Return position properties of amm contract.

```solidity
    lastFundingState()
    lastFundingRate()
    currentFundingState()
    currentFundingRate()
    currentAccumulatedFundingPerContract()
```

Return funding related variables.
