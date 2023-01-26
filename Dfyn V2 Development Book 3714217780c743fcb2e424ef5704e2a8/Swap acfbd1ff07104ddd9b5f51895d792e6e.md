# Swap

Swap function is in the CLP contract

### ConcentratedLiquidityPool.sol

This function takes data and path as args and returns amountOut and amountIn

```solidity
/// @dev Swaps one token for another. The router must prefund this contract and ensure there isn't too much slippage.
    function swap(bytes memory data, bytes memory path) public lock returns (uint256 amountOut, uint256 amountIn) {
```

First we decode the data

```solidity
(bool zeroForOne, address recipient, bool unwrapVault, int256 quantity) = abi.decode(data, (bool, address, bool, int256));
```

Then we initialise the SwapCache struct and updateSecondsPerLiquidity

```solidity
SwapCache memory cache = SwapCache({
            feeAmount: 0,
            protocolFee: 0,
            feeGrowthGlobalA: zeroForOne ? feeGrowthGlobal1 : feeGrowthGlobal0,
            feeGrowthGlobalB: zeroForOne ? feeGrowthGlobal0 : feeGrowthGlobal1,
            currentPrice: uint256(price),
            currentLiquidity: uint256(liquidity),
            amountIn: quantity > 0 ? uint256(quantity) : 0,
            amountOut: quantity > 0 ? 0 : uint256(-quantity),
            totalAmount: 0,
            exactIn: quantity > 0,
            nextTickToCross: zeroForOne ? nearestTick : ticks[nearestTick].nextTick,
            limitOrderAmountOut: 0, //amount of output given by limitorder
            limitOrderAmountIn: 0, //amount of input limitorder consumed
            limitOrderReserve: zeroForOne ? limitOrderReserve0 : limitOrderReserve1
        });
        _updateSecondsPerLiquidity(cache.currentLiquidity);
```

In this while loop we traverse from the current tick to the next tick in the direction of the swap till the amount to be swapped is exhausted or we reach the extreme end ticks.
The _excecuteConcentrateLiquidity() function executes the normal swap and _exceuteLimitOrder() function executes the limit orders.

```solidity
while ((cache.exactIn ? cache.amountIn != 0 : cache.amountOut != 0)) {
            SwapCacheLocal memory swapLocal = SwapCacheLocal({
                nextTickPrice: uint256(TickMath.getSqrtRatioAtTick(cache.nextTickToCross)),
                cross: false,
                fee: 0,
                amountIn: 0,
                amountOut: 0
            });
            // console.logUint(cache.amountIn);
            // console.logUint(cache.amountOut);
            // console.logInt(cache.nextTickToCross);

            require(cache.nextTickToCross != TickMath.MIN_TICK && cache.nextTickToCross != TickMath.MAX_TICK, "E4");

            // TODO:[amountOut] this should return amount out and amount in
            (cache.amountOut, cache.currentPrice, swapLocal.cross, cache.amountIn, swapLocal.fee) = SwapExcecuter
                ._executeConcentrateLiquidity(cache, zeroForOne, swapLocal.nextTickPrice, swapLocal.cross, swapFee);
            // console.logUint(cache.amountIn);
            // console.logUint(cache.amountOut);
            // reset amountIn and amountOut and update totalAmount
            cache.totalAmount += cache.exactIn ? cache.amountOut : cache.amountIn;
            (cache.amountOut, cache.amountIn) = cache.exactIn ? (uint256(0), cache.amountIn) : (cache.amountOut, uint256(0));

            // TODO:[amountOut] update remaining amount depending on exactIn or exactOut
            // cache.feeGrowthGlobalA is the feeGrowthGlobal counter for the output token.
            // It increases each swap step.
            // TODO:[amountOut] see how the fee growth will be impacted by amount out
            // console.logUint(swapLocal.fee);
            // console.logUint(dfynFee);
            // console.logUint(cache.feeGrowthGlobalB);
            // console.logUint(cache.feeGrowthGlobalA);
            // console.logUint(cache.currentLiquidity);

            if (swapLocal.fee != 0)
                (cache.protocolFee, cache.feeGrowthGlobalB) = FeeHandler.handleFees(
                    FeeHandler.FeeHandlerRequest({
                        feeAmount: swapLocal.fee,
                        dfynFee: dfynFee,
                        protocolFee: cache.protocolFee,
                        currentLiquidity: cache.currentLiquidity,
                        feeGrowthGlobal: cache.feeGrowthGlobalB
                    })
                );

            // console.log(swapLocal.cross);
            // console.logUint(cache.feeGrowthGlobalB);
            // console.logUint(cache.feeGrowthGlobalA);
            //checks limit order liquidity exist in next tick & excecutes if liquidity exist
            if (swapLocal.cross) {
                {
                    SwapExcecuter.ExecuteLimitResponse memory response = SwapExcecuter._executeLimitOrder(
                        ExecuteLimitOrderParams({
                            sqrtpriceX96: swapLocal.nextTickPrice,
                            tick: cache.nextTickToCross,
                            zeroForOne: zeroForOne,
                            amountIn: cache.amountIn,
                            amountOut: cache.amountOut,
                            limitOrderAmountOut: cache.limitOrderAmountOut,
                            limitOrderAmountIn: cache.limitOrderAmountIn,
                            cross: swapLocal.cross,
                            token0LimitOrderFee: token0LimitOrderFee,
                            token1LimitOrderFee: token1LimitOrderFee,
                            exactIn: cache.exactIn,
                            limitOrderFee: limitOrderFee
                        }),
                        limitOrderTicks
                    );
                    // Set the state of cache and other variables in scope
                    cache.amountIn = response.amountIn;
                    cache.amountOut = response.amountOut;
                    swapLocal.cross = response.cross;
                    token0LimitOrderFee = response.token0LimitOrderFee;
                    token1LimitOrderFee = response.token1LimitOrderFee;
                    cache.limitOrderAmountOut = response.limitOrderAmountOut;
                    cache.limitOrderAmountIn = response.limitOrderAmountIn;

                    // reset amountIn and amountOut and update totalAmount
                    cache.totalAmount += cache.exactIn ? cache.amountOut : cache.amountIn;
                    (cache.amountOut, cache.amountIn) = cache.exactIn ? (uint256(0), cache.amountIn) : (cache.amountOut, uint256(0));
                }
                if (swapLocal.cross)
                    (cache.currentLiquidity, cache.nextTickToCross) = Ticks.cross(
                        ticks,
                        Ticks.TickCrossRequest({
                            nextTickToCross: cache.nextTickToCross,
                            secondsGrowthGlobal: secondsGrowthGlobal,
                            currentLiquidity: cache.currentLiquidity,
                            feeGrowthGlobalA: cache.feeGrowthGlobalA,
                            feeGrowthGlobalB: cache.feeGrowthGlobalB,
                            zeroForOne: zeroForOne,
                            tickSpacing: tickSpacing
                        })
                    );
            }
        }

```

we then fetch the price from cache and set the newNearestTick based on zeroForOne direction

```solidity
price = uint160(cache.currentPrice);

int24 newNearestTick = zeroForOne ? cache.nextTickToCross : ticks[cache.nextTickToCross].previousTick;

if (nearestTick != newNearestTick) {
    nearestTick = newNearestTick;
    liquidity = uint128(cache.currentLiquidity);
}
```

We check for exactIn or exactOut and g=fetch the amountIn and amountOut and then based on zeroForOne we do the transfers.

```solidity
(amountOut, amountIn) = cache.exactIn ? (cache.totalAmount, uint256(quantity)) : (uint256(-quantity), cache.totalAmount);

        if (zeroForOne) {
            _transfer(token1, amountOut, recipient, unwrapVault);
            emit Swap(recipient, amountIn, amountOut, zeroForOne, cache.nextTickToCross);
        } else {
            _transfer(token0, amountOut, recipient, unwrapVault);
            emit Swap(recipient, amountIn, amountOut, zeroForOne, cache.nextTickToCross);
        }
```

We send a callback to the quoter, this swapCallBack is for the quoter to fetch quotes.

```solidity

IDfynCallBack(msg.sender).swapCallBack(cache.exactIn, unwrapVault, amountIn, amountOut, path);
```

Here we check whether the amountIn in is same as difference of reserves then we update the fees and reserves.

```solidity
require(
            amountIn ==
                _balance(zeroForOne ? token0 : token1) -
                    ((zeroForOne ? reserve0 : reserve1) + (zeroForOne ? limitOrderReserve0 : limitOrderReserve1)),
            "TM"
        );
        _updateFees(zeroForOne, cache.feeGrowthGlobalB, uint128(cache.protocolFee));
        _updateReserves(zeroForOne, uint128(amountIn), amountOut, cache.limitOrderAmountIn, cache.limitOrderAmountOut);
        // adding limitOrder liquidity with normal liquidity
}
```