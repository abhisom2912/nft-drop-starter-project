# Limit Orders

## Creating a limit Order

The process of placing a limit order starts from the LimitOrderManager contract where createLimitOrder() function is called.
 

### LimitOrderManager.sol

```solidity
/// @notice Function to create limitorder
    /// @param pool pool address
    /// @param tick limitorder tick
    /// @param lowerOld tick below limitorder tick
    /// @param upperOld tick above limitorder tick
    /// @param native user wallet/vault
    /// @param amountIn amount in
    /// @param zeroForOne direction of limitorder
    /// @return limitOrder Id
    function createLimitOrder(
        IConcentratedLiquidityPool pool,
        int24 tick,
        int24 lowerOld,
        int24 upperOld,
        bool native,
        uint256 amountIn,
        bool zeroForOne
    ) external payable returns (uint256) {
```

First we check if the pool in which limit order is being created is valid.
Zero amount limit orders cannot be placed

```solidity
require(masterDeployer.pools(address(pool)), "INVALID_POOL");
require(amountIn != 0, "Amount:Zero");
```

We store the pool address and msg.sender address

```solidity
cachedPool = address(pool);
cachedMsgSender = msg.sender;
```

We then call the createLimitOrder() function of CLP

```solidity
pool.createLimitOrder(tick, lowerOld, upperOld, amountIn, native, zeroForOne);
```

We then fetch the sqrtPrice from the tick and calculate the amountOut based on the zeroForOne.
Here zeroForOne depicts which token is the limit order being set in.

```solidity
uint160 sqrtpriceX96 = TickMath.getSqrtRatioAtTick(tick);
        uint256 amountOut;
        if (zeroForOne) {
            // amountOut = (amountIn * price) / 10**12;
            // amountOut = FullMath.mulDiv(amountIn, price, 10**12);
            amountOut = FullMath.mulDiv(amountIn, sqrtpriceX96, 2**96);
            amountOut = FullMath.mulDiv(amountOut, sqrtpriceX96, 2**96);
        } else {
            // amountOut = (amountIn * 10**12) / price;
            amountOut = FullMath.mulDiv(amountIn, 2**96, sqrtpriceX96);
            amountOut = FullMath.mulDiv(amountOut, 2**96, sqrtpriceX96);
        }
        // amountOut = amountOut / 10**6;
```

We increase the limitOrderId and mint a new limitOrderToken which is similar to a NFT position
limitOrderId is added to the array of limitOrders, CreateLimitOrder  event is emitted and limitOrderId is returned.

```solidity
limitOrderId++;

        limitOrderToken.mint(msg.sender, limitOrderId);

        limitOrders[limitOrderId] = LimitOrder({
            pool: pool,
            id: limitOrderId,
            tick: tick,
            sqrtpriceX96: uint256(sqrtpriceX96),
            zeroForOne: zeroForOne,
            amountIn: amountIn,
            amountOut: amountOut,
            claimedAmount: 0,
            status: LimitOrderStatus.active
        });

        emit CreateLImitOrder(address(pool), tick, limitOrderId);
        return limitOrderId;
}
```

Now we’ll look at the createLimitOrder() function of CLP

### ConcentratedLiquidityPool.sol

```solidity
function createLimitOrder(
        int24 tick,
        int24 lowerOld,
        int24 upperOld,
        uint256 amountIn,
        bool native,
        bool zeroForOne
    ) external lock {
```

First we check whether the ticks are valid are not if yes then we create limit order params

```solidity
Validators._ensureValidLimitOrderTick(ticks, limitOrderTicks, tickSpacing, tick, lowerOld, upperOld);
        CreateLImitOrderParams memory params = CreateLImitOrderParams({
            tick: tick,
            lowerOld: lowerOld,
            upperOld: upperOld,
            zeroForOne: zeroForOne,
            amountIn: amountIn,
            price: price,
            nearestTick: nearestTick,
            tickCount: tickCount
        });
```

Now we insert the limit Order tick and get the tickCount and nearestTick.
We check which token we are placing limit order in by zeroForOne

```solidity
(tickCount, nearestTick) = Ticks.insertLimitOrderTick(ticks, limitOrderTicks, params);
ILimitOrderManager(msg.sender).limitOrderCallback(zeroForOne ? token0 : token1, amountIn, native);

        if (zeroForOne) {
            limitOrderReserve0 += amountIn;
            if (reserve0 + limitOrderReserve0 > _balance(token0)) revert Token0Missing();
        } else {
            limitOrderReserve1 += amountIn;
            if (reserve1 + limitOrderReserve1 > _balance(token1)) revert Token1Missing();
        }
        emit CreateLimitOrder(tick, amountIn, zeroForOne);
```

As seen above we saw that we had called insertLimitOrderTick() function of the Tick library. Lets take a loot at that

### Tick.sol

This function returns the new tick count and the new nearest tick

```solidity
function insertLimitOrderTick(
        mapping(int24 => IConcentratedLiquidityPoolStruct.Tick) storage ticks,
        mapping(int24 => ILimitOrderStruct.LimitOrderTickData) storage limitOrderTicks,
        ILimitOrderStruct.CreateLImitOrderParams memory params
    ) public returns (uint256, int24) {
```

First we state all the local variables required

```solidity
int24 tick = params.tick;
int24 lowerOld = params.lowerOld;
int24 upperOld = params.upperOld;
bool zeroForOne = params.zeroForOne;
uint256 amountIn = params.amountIn;
uint128 currentNormalLiquidity = ticks[tick].liquidity;
uint256 token0limitOrderLiquidity = limitOrderTicks[tick].token0Liquidity;
uint256 token1limitOrderLiquidity = limitOrderTicks[tick].token1Liquidity;
```

here we check if we are inserting a new tick 

```solidity
if (currentNormalLiquidity != 0 || limitOrderTicks[tick].isActive || tick == TickMath.MAX_TICK || tick == TickMath.MIN_TICK) {
            if (zeroForOne) {
                token0limitOrderLiquidity += amountIn;
            } else {
                token1limitOrderLiquidity += amountIn;
            }
        }
```

if not we are adding liquidity to an existing tick

```solidity
else {
      // We are inserting a new tick.
      IConcentratedLiquidityPoolStruct.Tick storage oldLower = ticks[lowerOld];

      IConcentratedLiquidityPoolStruct.Tick storage oldUpper = ticks[upperOld];

      require(
          (oldLower.liquidity != 0 || lowerOld == TickMath.MIN_TICK || limitOrderTicks[lowerOld].isActive) &&
              (oldUpper.liquidity != 0 || upperOld == TickMath.MAX_TICK || limitOrderTicks[upperOld].isActive) &&
              (lowerOld < tick) &&
              (tick < oldLower.nextTick) &&
              (oldUpper.previousTick < tick) &&
              (upperOld > tick),
          "LUO"
      );

      if (zeroForOne) {
          token0limitOrderLiquidity = amountIn;
      } else {
          token1limitOrderLiquidity = amountIn;
      }

      ticks[tick] = IConcentratedLiquidityPoolStruct.Tick(lowerOld, upperOld, 0, 0, 0, 0);
      oldLower.nextTick = tick;
      ticks[upperOld].previousTick = tick;

      int24 tickAtPrice = TickMath.getTickAtSqrtRatio(params.price);

      if (params.nearestTick < tick && tick <= tickAtPrice) {
          params.nearestTick = tick;
      }
      unchecked {
          params.tickCount++;
      }
  }
```

Based on zeroForOne we add limit order liquidity to that particular limit order tick

```solidity
if (zeroForOne) {
            limitOrderTicks[tick].token0Liquidity = token0limitOrderLiquidity;
        } else {
            limitOrderTicks[tick].token1Liquidity = token1limitOrderLiquidity;
        }

        limitOrderTicks[tick].isActive = true;

        return (params.tickCount, params.nearestTick);
    }
```

Coming back to the createLimitOrder(0 function in CLP we had called limitOrderCallback function here from LimitOrderManager. 

### LimitOrderMananger.sol

Here we transfer the amount of the limit order token from user to the vault.
this function can only be called by the cachedPool address which should be the CLP address.
We check whether the limit order token is native or not and accordingly we do the transfers from user to the pool.

```solidity
function limitOrderCallback(
        address token,
        uint256 amount,
        bool native
    ) external {
        require(msg.sender == cachedPool, "UNAUTHORIZED_CALLBACK");

        if (native) {
            _depositFromUserToVault(token, cachedMsgSender, msg.sender, amount);
        } else {
            vault.transfer(token, cachedMsgSender, msg.sender, amount);
        }
        cachedMsgSender = address(1);
        cachedPool = address(1);
    }
```