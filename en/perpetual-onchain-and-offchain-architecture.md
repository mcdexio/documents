# The On-Chain Contract and Off-Chain Order Book Architecture

With the on-chain AMM and off-chain order book running at the same time, MCDEX users have two trading choices:

| Trading method        | Advantages | Disadvantages |
|-----------------------|------------|---------------|
| Trade with AMM        | Fully decentralized trading. Can be called by another contract<br>Simple & intuitive UX | Potential risk of worse slippage |
| Trade with Order book | Better liquidity<br>Similar experience with Perpetual on centralized exchanges | Off-chain order book |

This hybrid architecture is shown in the figure below:

![mai2-arch](asset/mai2-arch.png)

The Mai Protocol V2 is mainly composed of three parts: Perpetual, AMM and Exchange.
* Perpetual: Stores data of margin accounts including collaterals and positions
* AMM: Implements a Uniswap-like interface for user to directly interact with contract.
* Exchange: Implements "match" interface for order book trading, similar to what Mai protocol V1 achieved

As a decentralized exchange, we profoundly understand the importance of an on-chain trading interface, which means that anyone can call the interface function of the smart contract to trade without relying on any off-chain facilities. AMM provides the on-chain interface we are talking about.

The off-chain order book matching interface is a supplement to improve the user experience. Due to the current inefficiency of blockchain, the Hybrid model of off-chain matching and the on-chain transactions is one of the solutions to achieve efficiency.

Another key component is the decentralized oracle for obtaining the index price (spot price). After extensive research into decentralized oracle solutions, MCDEX team unanimously concluded that Chainlink's Price Reference Contracts are the best option in the market for sourcing and securing data. Chainlink already has an ETH/USD Price Reference Contract live on the Ethereum mainnet that is used and supported by several other top DeFi protocols. We are currently leveraging Chainlink's ETH/USD price feed as the index price for the ETH-PERP contract.
