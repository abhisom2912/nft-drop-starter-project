# Dfyn V2 Ticks

## What is a Tick?

In financial markets, a tick refers to the smallest possible change in the price of a financial instrument. For instance, if a particular asset has a tick size of $0.01, this means that its price can only be altered in increments of $0.01 and cannot fluctuate by a smaller amount. The tick size for different securities may vary depending on various factors, such as the liquidity of the security and the minimum price movement that the market can accommodate.

## Tick Representation in Dfyn V2:

Dfyn V2 allows for more precise trading by dividing the price range $[0,∞]$ into granular ticks, similar to an order book exchange:

- The system defines the price range for each tick rather than relying on user input.
- Trades within a tick still follow the pricing function of the AMM, but the equation must be updated when the price crosses a tick.
- Orders can be executed at any price within the tick's range. In the case of limit orders, we have "limit order ticks" (discussed in the next section).
- For every new position, we insert two elements into the linked list based on the range's start and end prices. To keep the list manageable, we limit the range's prices to be a power of $1.0001$. For example, our range could start at $tick(0)$ with a price of $1.0000 = 1.0001^0$ and end at  $tick(23028)$, which corresponds to a price of approximately $10.0010 = 1.0001^{23028}$. Using this approach, we can cover the entire $(0, inf)$ price range using only 24-bit integer values.

The linked list is represented by a mapping of 24-bit integers to "Tick" structures, where each "Tick" holds pointers to the previous and next tick, as well as liquidity and other variables tracking fees and time spent within a range.

```jsx
struct Tick {
        int24 nextTickToCross; // based on zeroForOne
        uint160 secondsGrowthGlobal; // used to identify yield
        uint256 currentLiquidity; // concentrated liquidity is marked as store in the upper tick
        uint256 feeGrowthGlobalA; // fee counter
        uint256 feeGrowthGlobalB;
        bool zeroForOne; // token 0 to token 1 or vice versa
        uint24 tickSpacing;
    }
```

##