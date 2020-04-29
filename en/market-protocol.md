# Market Protocol

Please refer to the Market Protocol [white paper](https://marketprotocol.io/assets/MARKET_Protocol-Whitepaper.pdf) and [GitHub](https://github.com/MARKETProtocol/MARKETProtocol) for more complete information.

## Overview

MARKET Protocol provides users with a trustless and secure framework for creating decentralized derivative position tokens, including the necessary collateral pool and position clearing infrastructure. Creating (or minting) positions results in ERC-20 tokens that function like derivatives by providing price exposure to a reference asset, either digital or traditional. Reference digital assets are not limited to ERC-20 tokens, allowing price exposure to cryptocurrencies like Bitcoin, Ripple, and Monero.

A new set of position tokens can be minted by first defining their rules and reference asset and then by depositing collateral. A single set of tokens is comprised of one long and one short position token, and the pair can be used to reclaim the collateral that backs these positions. At any point in time, the collateral in the pool fully covers the maximum gain and loss of both long and short positions, removing counterparty risk and replacing one of the core functions of traditional exchange platforms.

Once minted, MARKET Protocol positions are tradeable on any exchange, which means subsequent traders of a position token do not have to participate in the token creation process. Position tokens offer users continuous price exposure and automated future settlement. They can be stored off-exchange in users wallets and are tradable on both centralized and decentralized exchanges, allowing traders to easily enter long or short positions in any token.

A trader can close their position by selling it on an exchange, for example, by selling a long token in return for ether (ETH) or a stablecoin. Alternatively, a trader can close a long position by buying a corresponding short token and redeeming the set of tokens for a return of capital directly from the collateral pool. Finally, a trader can hold their position until expiration. In this case, oracle provides a settlement value, which is used to determine the trader’s profit or loss. Once the position token enters a settled state, the trader can use their token to call a function for a return of collateral.


## Contract Definition

Each pair of long and short MARKET Protocol position tokens are administered according to a set of predefined terms outlined within their contract specification. MARKET Protocol uses these terms to automatically settle all MARKET Protocol position tokens. Each contract specification will include the following terms:

- Reference Asset: What asset is used for pricing? This can be digital or traditional.
- Price Floor and Cap: This defines the maximum gain or loss for participants.
- Expiration Date: This is the date at which the position token is settled.
- Settlement Mechanism: An oracle will provide a settlement price to be used in the position token’s final profit and loss calculations.
- Collateral Token: What will the base asset be for profit and loss settlement? This must be an ERC-20 token and is used to collateralize the position token.

## Position Ranges

MARKET Protocol position tokens offer continuous profit and loss exposure derived from a reference asset up to the limits of the PRICE_CAP and PRICE_FLOOR defined in the contract specification. All prices between the PRICE_CAP and PRICE_FLOOR are tradeable, outcomes are not binary. Traders can replicate uncapped payoff structures by utilizing a series of tokens.

**Unlike the earlier version of Market Protocol, if the high or low of the range is breached, the token is NOT settled. The token is settled only when the contract has reached its expiration date.** 

## Leverage

Position tokens provide implicit leverage to their holders. For a trader acquiring a long token, the amount he trades for the token is the position’s maximum downside. This is always less than the notional value of the position (the current price of its reference asset) For example, a trader could post only $50 to gain price exposure to a $200 share of Tesla stock (TSLA), by buying a long position token with a PRICE_FLOOR of $150.

The amount of leverage depends on the width of the position range (the difference between the PRICE_CAP and PRICE_FLOOR) relative to the price of a reference asset. All else equal, a narrower range provides more leverage than a wider range. Furthermore, the specific amount of leverage afforded to an open position depends on the price of the position relative to its price range. For example, a trader who buys a long token near the bottom of its price range (PRICE_FLOOR) will have more leverage compared to buying near the top of its range (PRICE_CAP).

Leverage offered through MARKET Protocol differs from traditional leverage, which runs the risk of forced liquidations and unfunded positions. Instead, MARKET Protocol position tokens are fully backed by the collateral contributed during the token minting process. At any point in time, this collateral fully covers the maximum gain and loss of all tokens. A set of long and short tokens can be used to claim collateral from the pool, or a single token can be redeemed for collateral after it has expired and settled.

## Expiration & Settlement

Price oracles will be used to determine if the contract has reached its expiration date. If such criteria is met, the contracts’ position tokens enter an expired state and the settlement process automatically begins, allocating profits and losses to position token holders.

Tokens can be settled to the price of any actively traded ERC-20 token, cryptocurrency, or other listed asset by calling an oracle. For example, the defined token settlement terms could specify the last traded price of a reference asset on Kraken at a predetermined point in time. Oracle frameworks such as Thomson Reuters’ BlockOne IQ or Chainlink can be used to bring external data on to the blockchain, or an oracle can be written specifically for a contract.

**The settle price is the index price provided by the Oracle when the contract expires. If the index price is higher than the price cap, the settle price is set to the price cap. On the other hand, if the index price is lower than the price floor, the settle price is set to the price floor.**

Initially, a small group of project supports and team members will be responsible for reviewing the values used for settlement. As crowd-based consensus mechanisms evolve, we will implement additional resolution mechanisms into MARKET Protocol.

Initially, MARKET Protocol has chosen to use an internally developed oracle service which allows for the team to get to launch the initial implementation.


