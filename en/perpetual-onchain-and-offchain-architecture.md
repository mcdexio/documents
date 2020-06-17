# Roles and the Hybrid Architecture

## The Hybrid Architecture

The Mai Protocol V2 is mainly composed of three parts: Perpetual, AMM and Exchange.
* Perpetual: Stores data of margin accounts including collaterals and positions
* AMM: Implements a Uniswap-like interface for user to directly interact with contract.
* Exchange: Implements "match" interface for order book trading, similar to what Mai protocol V1 achieved

This hybrid architecture is shown in the figure below:

![mai2-arch](asset/mai2-arch.png)

We will explain this architecture from the perspective of different roles:

- [Roles and the Hybrid Architecture](#roles-and-the-hybrid-architecture)
  - [The Hybrid Architecture](#the-hybrid-architecture)
  - [Traders](#traders)
  - [AMM](#amm)
  - [Off-chain Order Book](#off-chain-order-book)
  - [Broker](#broker)
  - [Liquidity Provider](#liquidity-provider)
  - [Oracle](#oracle)
  - [Admin](#admin)

## Traders

The trader first deposits collaterals into the `Perpetual`, then trades with `Exchange` or `AMM` to get his/her positions. With the on-chain AMM and off-chain order book running at the same time, traders have two trading choices:

| Trading method        | Advantages | Disadvantages |
|-----------------------|------------|---------------|
| Trade with AMM        | Fully decentralized trading. Can be called by another contract<br>Simple & intuitive UX | Potential risk of worse slippage |
| Trade with Order book | Better liquidity<br>Similar experience with Perpetual on centralized exchanges | Off-chain order book |

## AMM

AMM is also a special Trader who is always the counter-party to the caller.

As a decentralized exchange, we profoundly understand the importance of an on-chain trading interface, which means that anyone can call the interface function of the smart contract to trade without relying on any off-chain facilities. AMM provides the on-chain interface we are talking about.

## Off-chain Order Book

The off-chain order book matching interface is a supplement to improve the user experience.

Due to the current inefficiency of blockchain, the Hybrid model of off-chain matching and the on-chain transactions is one of the solutions to achieve efficiency.

## Broker

A Broker is an actor who helps trader to accomplish trading. It is a normal ETH address of the order book. The broker is set in the signature of every orders. To receive trading fee, a broker must assign positive maker/taker fee rate in order data structure. And if given a negative trading fee, the broker will pay trader for making / taking order.

## Liquidity Provider

We call the person who provides liquidity to AMM a Liquidity Provider. Anyone can provide AMM with liquidity and increase AMM's market making depth by adding inventory to AMM. Liquidity Providers are exposed to risks when imbalanced between long and short, and obtain trade fee income.

Check [How to Add Liquidity to AMM](en/how-to-add-liquidity-to-amm.md) for details.

## Oracle

Another key component is the decentralized oracle for obtaining the index price (spot price). After extensive research into decentralized oracle solutions, MCDEX team unanimously concluded that Chainlink's Price Reference Contracts are the best option in the market for sourcing and securing data. Chainlink already has an ETH/USD Price Reference Contract live on the Ethereum mainnet that is used and supported by several other top DeFi protocols. We are currently leveraging Chainlink's ETH/USD price feed as the index price for the ETH-PERP contract.

## Admin

Admin is a special account who has power to:
* Change the governance parameters
* Upgrade contract
* Global settlement

Check [Admin Functions](/en/perpetual-admin-functions.md) for details.

:warning: **Due to the importance of global settlement, MCDEX will establish a community-led governance committee as soon as possible, and the committee will develop detailed global settlement trigger mechanisms and processing procedures.**
