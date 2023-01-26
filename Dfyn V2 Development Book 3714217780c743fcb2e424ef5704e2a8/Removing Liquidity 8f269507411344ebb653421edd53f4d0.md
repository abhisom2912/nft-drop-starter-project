# Removing Liquidity

For removing liquidity we call the burn() function in ConcentratedLiquidityPoolManager.sol where in the msg.sender should be the owner of the tokenId.

```jsx
/// @notice Burns the position
    /// @param tokenId position id
    /// @param amount amount to burn
    /// @param recipient receipient address
    /// @param unwrapVault to userwallet/vault
    /// @param minimumOut0 minimum amount0 to claim
    /// @param minimumOut1 minimum amount1 to claim
    /// @return token0Amount actual token0 returned
    /// @return token1Amount actual token1 returned
    function burn(
        uint256 tokenId,
        uint128 amount,
        address recipient,
        bool unwrapVault,
        uint256 minimumOut0,
        uint256 minimumOut1
    ) external returns (uint256 token0Amount, uint256 token1Amount) {
        require(msg.sender == ownerOf(tokenId), "NOT_ID_OWNER");
```

We get the data about the existing position from the tokenId provided.
Next we compare the amount of liquidity to be burnt  with the amount of liquidity in the position.

If amount to be burnt is less than entire liquidity position we burn the amount and update the feeGrowthInside and reduce the burn amount liquidity from the position.
If amount to be burnt is same as the entire liquidity of the position we burn the entire position range along with the tokenId and delete the tokenId from the positions mapping

```jsx
Position memory position = positions[tokenId];

        (uint256 token0Fees, uint256 token1Fees, uint256 feeGrowthInside0, uint256 feeGrowthInside1) = positionFees(tokenId);

        if (amount < position.liquidity) {
            (token0Amount, token1Amount, , ) = position.pool.burn(position.lower, position.upper, amount);

            positions[tokenId].feeGrowthInside0 = feeGrowthInside0;
            positions[tokenId].feeGrowthInside1 = feeGrowthInside1;
            positions[tokenId].liquidity -= amount;
        } else {
            amount = position.liquidity;
            (token0Amount, token1Amount, , ) = position.pool.burn(position.lower, position.upper, amount);
            burn(tokenId);
            delete positions[tokenId];
        }
```

Check if the received token amounts are greater than or equal to min amounts.

if yes then transfer the assets to the recipient 

```jsx
require(token0Amount >= minimumOut0 && token1Amount >= minimumOut1, "TOO_LITTLE_RECEIVED");

        unchecked {
            token0Amount += token0Fees;
            token1Amount += token1Fees;
        }

        _transferBoth(position.pool, recipient, token0Amount, token1Amount, unwrapVault);

        emit DecreaseLiquidity(address(position.pool), msg.sender, tokenId, amount);
```

The burn() function of ConcentratedLiquidityPoolManager.sol calls the burn() of ConentratedLiquidityPool.sol Lets take a look at that.

```jsx
/// @notice Burns a position
    /// @param lower lower tick
    /// @param upper upper tick
    /// @param amount amount to burn
    function burn(
        int24 lower,
        int24 upper,
        uint128 amount
    )
        public
        lock
        returns (
            uint256 token0Amount,
            uint256 token1Amount,
            uint256 token0Fees,
            uint256 token1Fees
        )
```

We find the sqrtPrice at the lower and upper ticks and get the current price of the pool from price.

```jsx
uint160 priceLower = TickMath.getSqrtRatioAtTick(lower);
        uint160 priceUpper = TickMath.getSqrtRatioAtTick(upper);
        uint160 currentPrice = price;
```

We update the secondsGrowthGlobal of the liquidity

```jsx
_updateSecondsPerLiquidity(uint256(liquidity));
```

Check if the current Price lies in our range

```jsx
unchecked {
            if (priceLower <= currentPrice && currentPrice < priceUpper) liquidity -= amount;
        }
```

If yes then the token0 and token1 amount

```jsx
(token0Amount, token1Amount) = DyDxMath.getAmountsForLiquidity(
            uint256(priceLower),
            uint256(priceUpper),
            uint256(currentPrice),
            uint256(amount),
            false
        );
```

After that we fetch the  fees accumulated in token0 and token1  and add them to the token amounts which are to be returned to the user. Also deduct these total amounts from the reserve and transfer it to the user.

```jsx
// Ensure no overflow happens when we cast from uint128 to int128.
        if (amount > uint128(type(int128).max)) revert Overflow();

        (token0Fees, token1Fees) = _updatePosition(msg.sender, lower, upper, -int128(amount));

        uint256 amount0;
        uint256 amount1;

        unchecked {
            amount0 = token0Amount + token0Fees;
            amount1 = token1Amount + token1Fees;
            reserve0 -= uint128(amount0);
            reserve1 -= uint128(amount1);
        }

        _transferBothTokens(msg.sender, amount0, amount1);
```

Now since we have burnt the liquidity we remove the ticks belonging to that position and emit the Burn event.

```jsx
(nearestTick, tickCount) = Ticks.remove(ticks, limitOrderTicks, lower, upper, amount, nearestTick, tickCount);

        emit Burn(msg.sender, amount0, amount1, lower, upper);
```