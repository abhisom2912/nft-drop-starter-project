# Concise of Dfyn V1

### Constant Product AMM:

Dfyn V1 is a general exchange that implements a constant product AMM algorithm $(xy=k)$.

A constant product automated market maker (AMM) maintains a constant product of the pool of assets it holds. This means that the total value of the assets in the pool remains constant, regardless of changes in the value of individual assets. Because they maintain a constant product, these AMMs can provide predictable returns for liquidity providers.

### Inefficiencies of Constant Product AMM:

One potential issue with constant product AMMs is that they can become inefficient under certain conditions. This can happen when the prices of the assets in the pool change significantly. If the prices of the assets in the pool diverge significantly, the pool may become unbalanced, with some assets becoming overvalued and others becoming undervalued. This can lead to inefficiencies in the market and can reduce the attractiveness of the AMM as a liquidity provider. To mitigate this issue, some constant product AMMs may employ rebalancing mechanisms to ensure that the pool remains balanced and efficient.

Before discussing the innovations and new features of Dfyn V2, let's first examine the shortcomings of Dfyn V1. Because not all trading pairs are equal in terms of price volatility, they can be grouped accordingly:

1. Tokens with medium to high price volatility make up the majority of this group. Most tokens do not have their prices pegged to a specific value and are therefore subject to market fluctuations. As a result, this group includes the majority of tokens.
2. Tokens with low volatility belong to this group. This group primarily consists of pegged tokens, such as stablecoins like USDC, USDT, and DAI. It also includes variants of wrapped ETH, such as ETH/stETH and ETH/rETH.

These groups have different liquidity requirements for their pool configurations. For pegged tokens, high liquidity is necessary to prevent large trades from impacting the token's price. The value of USDC and USDT must remain stable, regardless of the volume of tokens being traded. Dfyn V1's general AMM algorithm is not well-suited for stablecoin trading, so alternative AMMs like Curve have become more popular for this type of trading. The issue with Dfyn V1 is that its pool liquidity is distributed without bounds, allowing trades at any price from 0 to infinity.

![Untitled](Concise%20of%20Dfyn%20V1%20d2d4b79417c54026a302f342ff94d2b6/Untitled.png)

However, this approach can lead to inefficiencies in the use of capital. Historical prices of an asset tend to fluctuate within a certain range, whether that range is narrow or wide. Therefore, providing liquidity in a price range that is far from the current price or is unlikely to be reached does not make good use of capital.