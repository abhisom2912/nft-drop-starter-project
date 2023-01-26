# Concentrated Liquidity Pool

Concentrated liquidity pools are a generalization of the traditional $xy = k$ pool. In the traditional model, all users provide liquidity on a $(0, inf)$ price range; however, in concentrated liquidity pools, each user can pick the range in which they want to provide liquidity. This allows users to narrow down the liquidity provision range, which amplifies their liquidity - meaning traders experience lesser price impact, and liquidity providers accrue more fees. This improves capital efficiency by allowing liquidity providers to put more liquidity into a narrow price range, which makes Dfyn more diverse: it can now have pools configured for pairs with different volatility. This is how DfynV2 improves DfynV1.

In a nutshell, a DfynV2 pair is many small DfynV1 pairs. The main difference between V1 and V2 is that, in V2, there are **many price ranges** in one pair. And each of these shorter price ranges has **finite reserves**. The entire price range from 0 to infinite is split into shorter price ranges, with each of them having its own amount of liquidity. But, what’s crucial is that within that shorter price range, **it works exactly as DfynV1**.

Now, let’s try to visualize it. What we’re saying is that we don’t want the curve to be infinite. We cut it at points a*a* and b*b* and say that these are the boundaries of the curve. Moreover, we shift the curve so the boundaries lay on the axes. This is what we get:

![Untitled](Concentrated%20Liquidity%20Pool%2035dcb3d066fb4ed9b084a738a04a9a2b/Untitled.png)

Buying or selling tokens moves the price along the curve. A price range limits the movement of the price. When the price moves to either of the points, the pool becomes **depleted**: one of the token reserves will be 0, and buying this token won’t be possible.

On the chart above, let’s assume that the start price is at the middle of the curve. To get to point a, we need to buy all available y and maximize x in the range; to get to point b, we need to buy all available x and maximize y in the range. At these points, there’s only one token in the range.

What happens when the current price range gets depleted during a trade? The price slips into the next price range. If the next price range doesn’t exist, the trade ends up fulfilled partially -we’ll see how this works later in the book.

This is how liquidity is spread in [the USDC/ETH pool in production](https://info.uniswap.org/#/pools/0x8ad599c3a0ff1de082011efddc58f1908eb6e6d8):

![https://uniswapv3book.com/images/milestone_0/usdceth_liquidity.png](https://uniswapv3book.com/images/milestone_0/usdceth_liquidity.png)

You can see that there’s a lot of liquidity around the current price, but the further we go from there, the less liquidity there is – this is because liquidity providers strive to have higher efficiency of their capital. Also, the whole range is not infinite, its upper boundary is shown in the image.