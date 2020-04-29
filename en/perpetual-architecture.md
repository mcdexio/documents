# MCDEX v2 Perpetual Architecture

With the on-chain AMM and off-chain order book running at the same time, MCDEX users have two trading choices:
a. Trade with AMM, which is simple and intuitive.
b. Trade with order book, which is similar to trading Perpetual contract on a centralized exchange.

![mai2-arch](asset/mai2-arch.png)

The Mai Protocol V2 protocol is mainly composed of three parts: Perpetual, AMM and Exchange.
* Perpetual: Stores data of margin accounts including collaterals and positions
* Exchange: Implements "match" interface for orderbook trading, similar to what Mai protocol V1 achieved
* AMM: Implements a Uniswap-like interface for user to directly interact with contract.
