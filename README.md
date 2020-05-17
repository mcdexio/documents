# MCDEX Documents

Monte Carlo Decentralized Exchange (MCDEX) is a decentralized derivatives exchange - a trading platform enabling efficient and transparent trading with leverages.

At present, there are two types of contracts live on MCDEX:
- [MCDEX Documents](#mcdex-documents)
  - [Mai Protocol V2 - Perpetual Docs](#mai-protocol-v2---perpetual-docs)
    - [User's Guid](#users-guid)
    - [News and Reports](#news-and-reports)
    - [Development and Contract Implementation](#development-and-contract-implementation)
  - [Mai Protocol V1 - MP Futures Docs](#mai-protocol-v1---mp-futures-docs)

## Mai Protocol V2 - Perpetual Docs

[![github.com/mcdexio/mai-protocol-v2](https://img.shields.io/badge/github-mcdexio%2Fmai--protocol--v2-success)](https://github.com/mcdexio/mai-protocol-v2)

Mai Protocol V2 builds the decentralized Perpetual contracts on Ethereum. The market price is soft-pegged to the spot price of the underlying asset via a funding mechanism.

### User's Guid
* [MCDEX references](https://mcdex.io/references/Perpetual)
* [The On-Chain Contract and Off-Chain Order Book Architecture](en/perpetual-onchain-and-offchain-architecture.md)
* [The Margin-Account Model](en/margin-account-model.md)
* [How to Add Liquidity to AMM (english)](en/how-to-add-liquidity-to-amm.md)
* [How to Add Liquidity to AMM (chinese)](cn/how-to-add-liquidity-to-amm.md)

### News and Reports
* [Introducing MCDEX V2- Perpetual](https://medium.com/@montecarlodex/introduce-mcdex-v2-perpetual-c97b18ff4e23)
* [MCDEX's Perpetual Contracts to Leverage Chainlink Oracles for Index Prices](https://medium.com/@montecarlodex/mcdexs-perpetual-contracts-to-leverage-chainlink-oracles-for-index-prices-7af84eb319d9)
* [MCDEX 2020 Prospect](https://medium.com/@montecarlodex/mcdex-2020-prospect-b47a74cd94d3)
* [Why AMM is Crucial to Decentralized Perpetual Contracts](https://medium.com/@montecarlodex/why-amm-is-crucial-to-decentralized-perpetual-contracts-70e3159d270d)

### Development and Contract Implementation
* [MCDex Order Book API](https://mcdex.io/doc/api)
* [Contract Architecture](en/perpetual-architecture.md)
* [Trading Functions: What Can Traders Do](en/perpetual-interfaces.md)
* [Admin Functions: What Can the Admin Do](en/perpetual-admin-functions.md)
* [Internal Design of Perpetual](en/internal-perpetual.md)
* [Internal Design of AMM](en/internal-amm.md)
* [Implementation of Funding Rate](en/internal-amm-funding-rate.md)
* [How To Build The Contract](en/perpetual-deployment.md)
* [Contract Source Code](https://github.com/mcdexio/mai-protocol-v2)




## Mai Protocol V1 - MP Futures Docs

[![github.com/mcdexio/mai-protocol](https://img.shields.io/badge/github-mcdexio%2Fmai--protocol-success)](https://github.com/mcdexio/mai-protocol)

MP Futures is an innovative financial tool with an expiration date. The leverage is set by the Cap and Floor price. Traders are able to gain exposure to crypto asset without taking custody of the underlying asset, by receiving cash settlement rather than the physical delivery.

* [MCDEX references](https://mcdex.io/references/MPFutures)
* [Mai: A Protocol for Trading Decentralized Derivatives](en/mai.md)
* [An Introduction to Market Protocol](en/market-protocol.md)
* [Contract Source Code](https://github.com/mcdexio/mai-protocol)

