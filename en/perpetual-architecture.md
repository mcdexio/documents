# Perpetual Architecture

The Mai Protocol V2 is mainly composed of three parts: Perpetual, AMM and Exchange. The Perpetual contract stores data of margin accounts including collaterals and positions. The Exchange contract implements "match" interface for order book trading, similar to what Mai protocol V1 achieved. The AMM contract implements a Uniswap-like interface for user to directly interact with contract.

![mai2-arch](asset/mai2-arch.png)

## Perpetual

Perpetual is the core of Mai Protocol V2 protocol. As mentioned above, it holds all assets owned by user, providing interfaces to manipulate balance and position. One Perpetual contract is exactly serving for one trading pair.

All assets of a trader are stored in his margin account, which contains a collateral account, a position account and its current broker, identified by user's ethereum address. Margin accounts are isolated between different perpetuals.

Every trader has to deposit before interacting with Exchange or AMM contract. Trades within system doesn't trigger real eth/erc20 transfer, but update the balance in their margin accounts.

### Collateral.sol

This contract provides low-layer functions of dealing with collaterals. The balance of an account may be affected both by depositing and PNL (Profit-and-Loss) made from position trading.

Note that deposited collateral is converted to a internal decimals presentation (default: fix float number with decimals == 18) then added to balance field.

*In some of our documents, we call balance in collateral account as cash balance to distinguish from erc20 token balance.*

### Position.sol

Similar to collateral account, this contract maintains position account for each trader and contains all position calculators required from the upper-layer contract.

### Brokerage.sol

Broker contract implements a delayed broker setter to protect order book trading from front running.

### PerpetualGovernance.sol

The governance contract holds all parameters required for running a perpetual contract, including risk parameters, addresses, and status.

Check design of perpetual for their details.

### Perpetual.sol

This contract is the core of Mai Protocal V2. It combines all components above, serving as the foundation of Exchange and AMM.

Calculation taking price as parameter reads price from AMM contract.

## Exchange

Exchange focuses on matching off-chain order for traders. It matches orders between trader (known as maker and taker), or trader and AMM.

A taker cannot match with makers and AMM at the same time.

### Exchange.sol

Exchange contract implements methods exchanging between traders or between trader and AMM.

Calling match method currently requires a caller as broker.

## AMM

AMM provides methods to add / remove liquidity and trading without order book. It can be easily called from other contract to build new DAPP.

AMM is designed to be an upgradable contract to adapt algorithm update. Since all assets are actually stored in perpetual contract, the upgrade progress could be smooth and lossless.

### AMM.sol

This contract serves three purpose: Uniswap-like trading, liquidation providing and funding rate calculation.

Traders could trade through buy/sell methods, with limited price and deadline or provide liquidation to contract to earn trading fee.

See design of AMM for more details of providing liquidation.

### AMMGovernance.sol

All parameters required by AMM goes here.

## Others

### GlobalConfig.sol

Global config is a simple contract only used to set block delay of withdrawal and broker update.

See "Broker & Withdraw Time Lock" section in the [references page](https://mcdex.io/references/#/en/perpetual?id=trade-with-the-order-book) for the design of time lock.

### PerpetualProxy.sol

To implement an upgradable AMM, the address of new AMM should alway stay the same to inherit assets from former one.

This contract plays as the role of address holder for AMM contract.