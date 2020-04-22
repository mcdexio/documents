# How to Add Liquidity to AMM

This article details how to provide/remove liquidity to AMM.

Because MCDEX Perpetual is [vanilla contract](https://mcdex.io/references/Perpetual#vanilla--inverse-contract) on the blockchain, the discussion here takes vanilla contract as an example. Note that ETH-PERP appears as a corresponding inverse contract on the UI.

## Trade with AMM

As a market participant, AMM's behavior is similar to traditional market makers: AMM gives bid and ask prices, and traders buy/long or sell/short with AMM. MCDEX Perpetual AMM currently uses a constant product pricing model. This is a pricing model that has been fully validated in Uniswap.

Unlike traditional market makers, anyone can provide AMM with liquidity and increase AMM's market making depth by adding inventory to AMM. We call the person who provides liquidity to AMM a liquidity provider. Liquidity providers are exposed to risks when imbalanced between long and short, and obtain trade fee income.

## AMM's Margin Account

Like ordinary traders, AMM has a margin account. There are only margin balances and long position in this margin account, and the algorithm of AMM makes the effective leverage of its long position is always less than 1, which also means that AMM margin account is always fully collateralized and will not be liquidation. The margin balance and long position in this margin account are also called AMM's inventory.

We use `y` to represent the number of long position in AMM, then the **AMM's Available Margin** is expressed as `x`

```
x = Margin Balance - y * Entry Price
```

`Entry Price` is the average price of AMM entering the long position. The above formula guarantees that AMM position will always be fully collateralized according to the Entry Price. The margin balance minus the position occupancy is used as the available margin for AMM.


It is worth noting that the available margin calculated by the margin account at the initial margin rate is

```
Available Margin = Margin Balance - y * Mark Price * Initial Margin  > x
```

This means that the **AMM's Available Margin** is alway less than the **Account's Available Margin**.

x and y are also AMM inventory quantities. The pricing model requires that `xy = k` remains unchanged before and after the transaction, and it can be concluded that the price of the transaction through AMM is:

```
The price to buy Δy contracts from AMM: P( Δy ) = x / ( y - Δy )
The price to sell Δy contracts to AMM: P( Δy ) = x / ( y + Δy )
```

When a trader goes long through AMM, the long position size of AMM (`y`) drops and the `AMM's Available Margin (x)` of will rise. This process consumes the long position inventory.

When a trader goes short through AMM, the long position size of AMM (`y`) rises and the `AMM's Available Margin (x)`will fall. This process consumes the available margin inventory.

For more mathematical derivation of AMM's pricing formula, please refer to [here](https://mcdex.io/references/Perpetual#automated-market-maker)。

From the above pricing formula, it can be concluded that the pricing of AMM is only related to the inventory quantity x and y of AMM. When the product k of xy is larger, the lower the slippage given by the pricing formula, resulting in a better liquidity. So adding liquidity to AMM means increasing the values ​​of x and y.



## Add liquidity to AMM

The liquidity provider increases AMM's liquidity by adding inventory to AMM, increasing the value of AMM inventory quantity x and y. The following process is completed in single contract call:

- Increase x: The Liquidity provider increases the number of AMM's long position (`x`) by Δx. To achieve this, the provider sells Δx contracts to AMM, at AMM's Mid Price (`x / y`).

- Increase y: The liquidity provider transfers the collateral tokens directly from their margin account to AMM's margin account, the amount transferred is `2 ∙ Δx ∙ x / y`

The collateral transferred to AMM is divided into two equal parts, one part is used for the margin occupied by the new position of AMM, and the other part is used to increase the `AMM's Avaiable Margin`. It can be proved that after increasing the liquidity, AMM's `Mid Price = x / y` remains unchanged.

After adding liquidity, the provider will get AMM share tokens according to the amount of the inventory provided. Share tokens represent ownership of the remaining inventory in the liquidity pool. When remove liquidity of AMM, the liquidity provider can obtain the inventory in the liquidity pool (including the long position and margin balance) in proportion to the share tokens by redeeming the share tokens.

It should be noted that since the provider sell contracts to AMM when adding liquidity, the position size of the provider will decrease. If the liquidity provider has no position before the operation, the position size in of the provider will become negative after the operation. On the other hand, due to the need to transfer collateral from the liquidity provider’s margin account to AMM's, if the provider originally has a position, the effective leverage of the position will normally increase due to the decrease of margin balance.


`Example 1` Alice ’s margin account is as follows

|Position Size| Margin Balance| Share of AMM|
|:--:|:---------:|:-----:|
|  0 |  50       |  0    |

And AMM's margin account:

|Position Size(`y`)| AMM's Available Margin(`x`) | Margin Balance | Mid Price(`x/y`) | 
|:---:|:-----------:|:---------:|:---------:|
| 10  |  100        |    200    | 10        |

Alice adds 1 contract liquidity to AMM:

1. Alice sells one contract to AMM at mid-price 10, then Alice's position size becomes -10 and AMM's position size becomes 11.
2. Alice transfers 10 * 1 * 2 = 20 collateral tokens to AMM

After the operation is completed, Alice's margin account

|Position Size| Margin Balance| Share of AMM|
|:--:|:---------:|:-----:|
|  -1|  30       |  1/11 |

And AMM's margin account:

|Position Size(`y`)| AMM's Available Margin(`x`) | Margin Balance | Mide Price(`x/y`) | 
|:--:|:---------:|:---------:|:------:|
| 11 |  110       |   220    | 10     |


`Example 2` Alice ’s margin account is as follows

|Position Size| Margin Balance| Share of AMM|
|:--:|:---------:|:-----:|
|  1 |  50       |  0    |

And AMM's margin account:

|Position Size(`y`)| AMM's Available Margin(`x`) | Margin Balance | Mide Price(`x/y`) | 
|:--:|:---------:|:---------:|:------:|
| 10 |  100       |   200    | 10     |

Alice adds 1 contract liquidity to AMM:

1. Alice sells one contract to AMM at mid-price 10, then Alice's position size becomes 0 and AMM's position size becomes 11.
2. Alice transfers 10 * 1 * 2 = 20 collateral tokens to AMM

After the operation is completed, Alice's margin account

|Position Size| Margin Balance| Share of AMM|
|:--:|:---------:|:-----:|
|  0 |  30       |  1/11 |

And AMM's margin account:

|Position Size(`y`)| AMM's Available Margin(`x`) | Margin Balance | Mide Price(`x/y`) | 
|:--:|:---------:|:---------:|:------:|
| 11 |  110       |   220    | 10     |


In this example, Alice had a long position at the beginning. After the adding operation, Alice had no position. This means Alice transfered her position to AMM.


`Example 1` Alice ’s margin account is as follows

|Position Size| Margin Balance| Share of AMM|
|:--:|:---------:|:-----:|
|  -1 |  50       |  0 |

And AMM's margin account:

|Position Size(`y`)| AMM's Available Margin(`x`) | Margin Balance | Mide Price(`x/y`) | 
|:--:|:---------:|:---------:|:------:|
| 10 |  100       |   200    | 10     |

Alice adds 1 contract liquidity to AMM::

1. Alice sells one contract to AMM at mid-price 10, then Alice's position size becomes -2 and AMM's position size becomes 11.
2. Alice transfers 10 * 1 * 2 = 20 collateral tokens to AMM

After the operation is completed, Alice's margin account


|Position Size| Margin Balance| Share of AMM|
|:--:|:---------:|:-----:|
|  -2 |  30       |  1/11 |

And AMM's margin account:


|Position Size(`y`)| AMM's Available Margin(`x`) | Margin Balance | Mide Price(`x/y`) | 
|:--:|:---------:|:---------:|:------:|
| 11 |  110       |   220    | 10     |

In this example, Alice had a short position at the beginning. After the addition operation, Alice's short position further increased.

## Risk exposure and Revenues

Simply providing liquidity to AMM does not increase the risk exposure of providers. This is because the short position caused by the adding operation is exactly equal to the AMM's long position attributable to the provider. For example, the above `Example 1`, when the adding operation is completed, AMM situation is as follows:


|Position Size(`y`)| AMM's Available Margin(`x`) | Margin Balance | Mide Price(`x/y`) |  Alice's shares | Position attributable to Alice | Margin Balance attributable to Alice |
|:--:|:------------:|:---------:|:---------:|:-----------:|:---------------:|:----------------:|
| 11 |  110         |    220    | 10        | 1/11        |       1         |       20         |

At this time, Alice's margin account is as follows:

|Position Size| Margin Balance| Share of AMM|
|:--:|:---------:|:-----:|
|  -1|  30       |  1/11 |

Alice's overall balance:

|Overall Position| Overall Margin Balance|
|:------:|:---------:|
|  1-1=0|  20+30=50  |

The overall balance is consistent with Alice’s origin margin account. It shows that providing liquidity to AMM only means transfering the provider's inventory to AMM's margin account.

When other traders trade with AMM, it will change AMM's long position size `y`, and change the amount of long position attributable to Alice in AMM. At this time, Alice's overall position size is no longer 0, and Alice has risk exposure.

For example: Bob buy 1 contract from AMM at the price `p = 110 / (11-1) = 11`, (ignore the trade fee)

The situation of AMM after the trade is as follows:

|Position Size(`y`)| AMM's Available Margin(`x`) | Margin Balance | Mide Price(`x/y`) |  Alice's shares | Position attributable to Alice | Margin Balance attributable to Alice |
|:--:|:------------:|:---------:|:---------:|:-----------:|:---------------:|:---------------:|
| 10 |  120         |    220    | 12        | 1/11        |       10/11     |  20             |


At this time, Alice's overall position size is `10/11-1 = -0.0909`, which is 0. As a result, Alice has the risk exposure of -0.0909 contract. This risk exposure disappears after another trader sell 1 contract to AMM.

The upper limit of the provider's risk exposure is the quantity x and y of the inventory he/her adds to the AMM. In practice, the provider can monitor the status of AMM. When risk exposure occurs, the provider can hedge the risk exposure on other exchanges to maintain risk neutrality.

What's more, when a trader trades with AMM, he needs to pay an additional 0.075% transaction fee, of which 0.06% will enter the AMM's margin account, increasing the AMM margin balance. As the provider has a share of the margin balance, the margin balance attributable to the provider rises consequently. When the provider removes liquidity from AMM, the provider can obtain the fee in proportion to the share. The fee is an incentive for liquidity providers.

 If the liquidity provider fully hedges its risk exposure, it can greatly reduce the risk and obtain a relatively stable fee income. On the other hand, there is also a premium/discount between the AMM price and other exchanges' prices, and the provider can also arbitrage meanwhile hedging. 
 
 Finally, in addition to the profit and loss caused by price fluctuations, the position of perpetual contracts also has the profit and loss caused by funding. Liquidity providers also need to consider the problem of funding according to specific strategies. In short, the strategies of liquidity providers to AMM can be very rich.


## Remove Liquidity from AMM

The liquidity provider can withdraw its share in the pool at any time. When remvoe liquidity, the following operations occur simultaneously:

1. AMM transfers the margin balance attributable to the provider.
2. The provider buys contracts from AMM at the middle price x / y. The trade amount is equal to the long position size attributable to the provider in AMM.

`Example 1`, followed by Example 1 of "Add Liquidity to AMM", if Alice remove liquidity at this time, after the operation is completed,

AMM's margin account:

|Position Size(`y`)| AMM's Available Margin(`x`) | Margin Balance | Mide Price(`x/y`) |  Alice's shares |
|:--:|:------------:|:---------:|:---------:|:-----------:|
| 10 |  100         |    200    | 10        | 0           |

And Alice's margin account:

|Position Size| Margin Balance| Share of AMM|
|:--:|:---------:|:-----:|
|  0 |  50       |  0    |

This is equivalent to reverting to Alice's original state

`Example 2`, follow the example in "Risk Exposure and Revenue", if Alice removes liquidity after Bob's transaction is completed, then the AMM's margin account:

|Position Size(`y`)| AMM's Available Margin(`x`) | Margin Balance | Mide Price(`x/y`) |  Alice's shares | Position attributable to Alice | Margin Balance attributable to Alice |
|:--:|:------------:|:---------:|:---------:|:-----------:|:---------------:|:--------------:|
| 10 |  110         |    200    | 12        | 0           |       0         |  0             |

And Alice's margin account:


|Position Size| Margin Balance| Share of AMM|
|:--:|:---------:|:-----:|
| -0.0909  |  50       |  0    |

It is showed that due to Bob's transaction, Alice has a risk exposure of -0.0909 contracts. This risk exposure still exists after removing liquidity, which means that removing liquidity does not change the provider's risk exposure. The provider can eliminate the risk exposure by closing the position later.









