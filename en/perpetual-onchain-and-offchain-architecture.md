# Roles and the Hybrid Architecture

There're several roles in the Perpetual system including:
- [Roles and the Hybrid Architecture](#roles-and-the-hybrid-architecture)
  - [Traders](#traders)
  - [AMM](#amm)
  - [Off-chain Order Book](#off-chain-order-book)
  - [Broker](#broker)
  - [Oracle](#oracle)
  - [Admin](#admin)

## Traders

Traders trade long or short positions in the system. With the on-chain AMM and off-chain order book running at the same time, traders have two trading choices:

| Trading method        | Advantages | Disadvantages |
|-----------------------|------------|---------------|
| Trade with AMM        | Fully decentralized trading. Can be called by another contract<br>Simple & intuitive UX | Potential risk of worse slippage |
| Trade with Order book | Better liquidity<br>Similar experience with Perpetual on centralized exchanges | Off-chain order book |

This hybrid architecture is shown in the figure below:

![mai2-arch](asset/mai2-arch.png)

The trader first deposits collaterals into the `Perpetual`, then trades with `Exchange` or `AMM` to get his/her positions.

## AMM

AMM is also a special Trader who is always the counter-party to the caller.

As a decentralized exchange, we profoundly understand the importance of an on-chain trading interface, which means that anyone can call the interface function of the smart contract to trade without relying on any off-chain facilities. AMM provides the on-chain interface we are talking about.

## Off-chain Order Book

The off-chain order book matching interface is a supplement to improve the user experience.

Due to the current inefficiency of blockchain, the Hybrid model of off-chain matching and the on-chain transactions is one of the solutions to achieve efficiency.

## Broker

The broker is the entry point for a trader to interact with the system. The broker can either be:
* AMM
* An order book relayer. The relayer is a ordinary ETH account who matches the orders in the order book, and sends transactions into the chain.

For security reasons, MCDEX only allows traders to trade through one broker at the same time. Traders have to call "change broker" command to switch the broker.

## Oracle

Another key component is the decentralized oracle for obtaining the index price (spot price). After extensive research into decentralized oracle solutions, MCDEX team unanimously concluded that Chainlink's Price Reference Contracts are the best option in the market for sourcing and securing data. Chainlink already has an ETH/USD Price Reference Contract live on the Ethereum mainnet that is used and supported by several other top DeFi protocols. We are currently leveraging Chainlink's ETH/USD price feed as the index price for the ETH-PERP contract.

## Admin

Admin is a special account who has power to:
* Change the governance parameters
* Upgrade contract
* Global liquidation

Check [this doc](/en/perpetual-admin-functions.md) for all admin functions.

:warning: **Due to the importance of global liquidation, MCDEX will establish a community-led governance committee as soon as possible, and the committee will develop detailed global liquidation trigger mechanisms and processing procedures.**
