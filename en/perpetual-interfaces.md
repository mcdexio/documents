# Contract interfaces

# Perpetual

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

Check __ for details of how application mechanism protect orderbook trading.

```solidity
deposit(uint256 amount) NORMAL
```
Deposit transfer collaterals from caller's address to contract. It accept an amount parameter which indicates the total amount of collaterals that user wants to transfer to contract.
Approval is required;

**amount should be a fixed float with token's decimals which will be convert to a decimals-18 representation. E.g. Jim depoist 1e6 usdt, later he will found 1e18 collaterals in his account of mai protocol v2. This only affects internal calculation*

```solidity
depositEther() NORMAL
```

Ether version of deposit, using msg.value instead. See description above for details. When using ether, the decimals will be automaticly set to 18.

```solidity
applyForWithdrawal(uint256 amount) NORMAL
```

Request for further withdrawal of caller's account. This method functions like approve of erc20. Trader could apply for the amount that far greater than actual balance he owned, but applied part is no longer available for position trading.

```solidity
withdraw(uint256 amount) NORMAL
```

Withdraw given amount collateral back to caller's ethereum address. Note that withdraw doesn't distinguish bewteen erc20 and ether.

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

There do exists interface to manipulate position and collateral for trading purpose, but basicly the perpetual contract does not directly handle trading request from user. It only provider interfaces for whitelisted callers like **exchange** and **amm** contract.

Calling trade method of perpetual may break contraints that long position should always equals to short positions.

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

*The contraint is that totalSize(SHORT) should always be equal to totalSize(LONG) and totalSize(FLAT) should always be zero.*

```solidity
socialLossPerContract(Side side)
```

Return social loss per contract of given side. Normally, the value of each side should never decrease.

```solidity
positionMargin(address guy)
```

> Methods of calcuating position value requires mark price from amm contract. This price will be replaced by settlement price set by administrator in settlement status (SETTLING, SETTLED).

```solidity
maintenanceMargin(address guy)
```

Return maintainance margin value of account base on current mark price. The value should always be lower than initial margin value.

If the current value of positions falls below maintainance margin, the account will go into a status called 'unsafe'. An unsafe account can be liquidated by any trader within the same perpetual to prevent possible loss.

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

Insurance fund is chaimed from liquidation penaty and will be used to recover social loss.


```soldiity
liquidate(address guy, uint256 maxAmount) public returns (uint256, uint256) NORMAL SETTLING
```

Liquidate bites a part of position from an unsafe account (See ___ for definition of unsafe) to make it safe again. The call should have enough margin to hold the biten positions or the call will fail.

All losses generated by liquidate will be removed from insurance fund first if possible, or the loss will become social loss.

-----

**Interact with brokers:**

Broker is a actor who helps trader to accomplish orderbook trading. Broker is set by trader, with a few blocks delay.

To receive trading fee, a broker must assign positive maker/taker fee rate in order data structure. And if given a negative trading fee, the broker will pay trader for making / taking order.

There are two typically value of broker:

- broker is set to a contract call who calls match method of exchange methods (orderbook trading);
- broker is set to address of amm (perpetual proxy acturally) to enable amm trading.

Trader cannot apply both the trading modes above since the broker variable can hold exactly one address at the same time.

There is a delay mechanism on setting broker address, See ____.

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

# Exchange





# AMM
