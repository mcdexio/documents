# Mai: A Protocol for Trading Decentralized Derivatives

## Motivation
With the development of the DeFi ecosystem, more and more synthetic assets and derivatives, such as [Market Protocol](https://marketprotocol.io), [UMA Protocol](https://umaproject.org), and [Yield Protocol](http://research.paradigm.xyz/Yield.pdf), are born in the world of blockchain. These protocols are good solutions to build derivatives contracts but often lack a simple, efficient, and easy-to-use trading mechanism. 

Existing trading protocols, such as 0x, Hydro, and Uniswap, have well solved ERC20 tokens exchange requirements. However, traders often have difficulty in trading ERC20 position tokens of those derivatives directly. There are basically two reasons.First of all, the trade of decentralized derivatives is usually accompanied by minting, redeeming, and exchange of position tokens. The existing ERC20 trading protocols can only complete the exchange, lacking the functions of minting and redeeming. Furthermore, the pricing of derivatives is often different from that of position tokens. Traders need complex price conversion to trade position tokens directly.

To solve the above difficulties, we designed a new trading protocol called "Mai Protocol" on Ethereum. 

**Mai Protocol's goal is to make trading decentralized derivatives easy and efficient.**

The name Mai comes from two Chinese characters "买," which means buy and "卖," which means sell. Using pinyin (the modern system for transliterating Chinese characters to Latin letters) "买" is spelled Mǎi and "卖" is spelled Mài.

We design and build Mai Protocol in stages. In the first release, **Mai protocol first supports trading Market Protocol contracts**. We will gradually support other derivatives in future versions.

## Challenges

Market Protocol provides a secure framework that supports decentralized issuance of index futures contracts. Market Protocol succeeds in fully tokenizing the positions of contracts. The position tokens of Market Protocol are standard ERC20 tokens, and traders can trade the tokens within any exchange. However, the user experience of directly trading position tokens is abysmal. The chanllenges are as follows:

First, a long trader and a short trader can not make a deal directly. Someone of them needs to mint a pair of position tokens (consisting of long position token and short position token) and sell the token of the opposite side to the counterparty to make the deal. For example, the long trader mint a pair of position tokens and sell the short position token to the short trader. Otherwise, to conclude the transaction, a market maker is needed to mint the position tokens. The market maker sells the long position token to the long trader and sells the short position token to the short trader.

Second, the pricing of the position token is not intuitive. The price of the position token does not equal to the price of the underlying asset. The position token's value equals to the position's margin. As a result, traders need to convert the asset price to the corresponding position token price with the price range (PRICE_FLOOR and PRICE_CAP):

```
For vanilla contract:
Long Position Token Price = ASSET_PRICE - PRICE_FLOOR
Short Position Token Price = PRICE_CAP - ASSET_RICE

For inverse contract:
Long Position Token Price = 1/PRICE_FLOOR - 1/(ASSET_PRICE)
Short Position Token Price = 1/(ASSET_PRICE) - 1/(PRICE_CAP)
```

Third, traders need to deal with two kinds of position tokens (Long Position Token and Short Position Token), each of which has both buy and sell operations. It means that two order books are required to trade a Market Protocol's contract. It is very different from traditional derivatives trading, which makes traders confused.

## Solution

Mai Protocol builds a trading platform that is very similar to traditional derivatives trading platforms. Mai protocol encapsulates the minting, exchange and redeeming operations for position tokens and provides only two operations, Buy and Sell, for echo Market Protocol contract. 

Traders do all trading processes through Buy and Sell operations within Mai Protocol. If the trader expects the underlying asset price to rise, he can enter the long position at the asset price by Buy operation. If the trader expects the underlying asset price to fall, then he can enter the short position by Sell operation. Besides, traders with long positions can decrease or close positions by Sell operation, and traders with short positions can decrease or close positions by Buy operation. The trader only needs to place an order on the price of the underlying asset. Mai Protocol automatically calculates the position token price corresponding to the price of the order.

The Mai Protocol relayer maintains a unified order book for each Market Protocol contract. The order book sorts all the orders by the bidding/asking prices of the underlying asset.

The match engine can match the orders on the different side of the order book. According to the positions of the two counterparties, there are four different matching types. Mai Protocol smart contract performs different processes (exchange, minting, or redeeming) for different match types:

| Trader A's Side | Trader A's Position | Trader B's Side  | Trader B's Position  | Mai Protocol Smart Contract Process             |
|-----------------|---------------------|------------------|----------------------|-------------------------------------------------|
| Buy             | Positive or Zero    |  Sell            |  Positive            | Transfer the long position token from B to A and transfer the collateral token from A to B |
| Buy             | Positive or Zero    |  Sell            |  Negative or Zero    | Mint a pair of position tokens from Market Protocol, send the long position token to A and the short position token to B |
| Buy             | Negative            |  Sell            |  Positive            | Redeem the pair of position tokens through Market Protocol and send the returned collateral tokens to A and B |
| Buy             | Negative            |  Sell            |  Negative or Zero    | Transfer the short position token from A to B and transfer the collateral token from B to A |

*"Positive" means the trader has long position tokens. "Negative" means the trader has short position tokens. "Zero" means the trader has no position token.*

When trading frequently, the position tokens may be redeemed immediately after be minted. To reduce redundant minting and redeeming operations, Mai Protocol builds a minting pool. The pool reserves some position tokens in advance. Mai protocol tries to mint or redeem from the pool first. Only when the pool is insufficient, Mai protocol calls Market Protocol's mint or redeem interfaces.

## Architecture
![mai-arch](asset/mai-arch.png)

### MAI Exchange

The MAI Exchange contract is at the core of the entire protocol. It handles the matching and cancellation of orders, and is designed to support the exchange of various assets.

Specifically, the bulk of the exchange logic resides in the public method matchOrders, which:

- Checks signatures and verifies the integrity of orders
- Computes the resulting state that would occur as a result of matching the passed in orders via a set of limit exchange matching engine rules (Buy, Sell, Mint, Redeem)

### Minting Pool

The minting pool is expected to serve as a buffer between proxy and MARKET Protocol. The proxy can be set to work with a deployed minting pool, or to interact with MARKET Protocol directly. Owner of the pool must deposit tokens before trading to enable the features.

The minting pool could help the proxy to save fees on:

- Using MKT (Market Token) for minting instead of using collaterals which is usually more expensive than former
- Recycling tokens for further exchanges, saving both gas and minting fees


## Acknowledgments

Mai is inspired by the [0x project](https://github.com/0xProject) and [Hydro](https://github.com/HydroProtocol).

## Source Code

View the Mai Protocol source code at [GitHub](https://github.com/mcdexio/mai-protocol).

