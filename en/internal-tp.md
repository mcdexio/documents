# Internal Design of Tokenized Position

:warning: NOTE: This is a technical document of the smart contract. If you don't care how smart contracts are written, please skip this document.

## Functions and Motivation

The "TokenizedPosition" contract is used to convert a long position of Perpetual into ERC20 token. A long position [can be kept fully collateralized automatically](https://github.com/mcdexio/documents/blob/master/en/how-to-add-liquidity-to-amm.md), so it is possible to make a long position leave the Perpetual and enter the public Ethereum ecosystem.

For example: An ETH-PERP inversed perpetual is collteralized with ETH. So the short position (The short position is from a human perspective. In the contract it is still a long position) is a synthetic USD. A BTC-USDT-PERP vanilla perpetual is collteralized with USDT. So the long position is a synthetic BTC. etc.

Keep in mind that minting and redeeming will maintain the fineness of the coin, where the fineness is defined as MarginBalance / PositionSize of the TokenizedPosition contract.

### mint(tpAmount)
 

### redeem(tpAmount)

### settle()
