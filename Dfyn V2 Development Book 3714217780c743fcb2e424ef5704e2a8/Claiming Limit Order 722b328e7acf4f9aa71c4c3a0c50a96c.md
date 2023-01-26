# Claiming Limit Order

After successful fill of our limit order we need to claim it to complete the process. Claiming of limit orders start from LimitOrderManager contract where we call the claimLimitOrder() function

### LimitOrderManager.sol

```solidity
/// @notice To Claim LimitOrder
    /// @param tokenId limit order id
    /// @param amount amount to claim
    /// @param unwrapVault to user wallet/vault
    function claimLimitOrder(
        uint256 tokenId,
        uint256 amount,
        bool unwrapVault
    ) public {
```

First we check that the msg.sender is owner of this limit order position and the amount of the order is not zero.

```solidity
require(msg.sender == limitOrderToken.ownerOf(tokenId), "NOT_ID_OWNER");
require(amount != 0, "Claim:Amount Zero");
```

Then we fetch the data of the limit order from the tokenID. We then check the status of the order and the claimable amount.

```solidity
LimitOrder memory limitOrder = limitOrders[tokenId];

cachedPool = address(limitOrder.pool);

cachedMsgSender = msg.sender;

require(limitOrder.status != LimitOrderStatus.closed, "Limit Order: Inactive");

uint256 totalClaimableAmount = limitOrder.amountOut - limitOrder.claimedAmount;

require(totalClaimableAmount >= amount, "Amount Exceeds Claimable Amount");
```

Then this function calls claimLimitOrder() function of CLP. If successfully executed the CLP function we then update the claimedAmount of the limitOrder data. If the entire amountOut is claimed we close the limit order and burn the tokenId. Then depending on zeroForOne we do the transfers accordingly.

```solidity
limitOrder.pool.claimLimitOrder(amount, limitOrder.tick, limitOrder.zeroForOne);

limitOrders[tokenId].claimedAmount += amount;

if (limitOrders[tokenId].claimedAmount == limitOrders[tokenId].amountOut) {
    limitOrder.status = LimitOrderStatus.closed;
    limitOrderToken.burn(tokenId);
}

(, , , , , , address _token0, address _token1) = limitOrder.pool.getImmutables();

if (limitOrder.zeroForOne) {
    _transferOut(_token1, msg.sender, amount, unwrapVault);
} else {
    _transferOut(_token0, msg.sender, amount, unwrapVault);
}

emit ClaimLimitOrder(address(limitOrder.pool), tokenId, amount);
}
```

Now lets take a look at claimLimitOrder of CLP.
It can only be called by LimitOrderMananger contract.|
We check the zeroForOne depending on amountToClaim

```solidity
function claimLimitOrder(
        uint256 amountToClaim,
        int24 tick,
        bool zeroForOne
    ) external onlyLimitOrderManager lock {
        require(
            zeroForOne ? limitOrderTicks[tick].token1Claimable >= amountToClaim : limitOrderTicks[tick].token0Claimable >= amountToClaim,
            "E2"
        );
```

 in this function we call claimLimitOrderTick of Ticks library which returns nearestTick and tickCount.

```solidity
(nearestTick, tickCount) = Ticks.claimLimitOrderTick(
            ticks,
            limitOrderTicks,
            tick,
            zeroForOne,
            amountToClaim,
            nearestTick,
            tickCount
        );
```

We deduct the amountToClaim from the limitOrderReserve and then transfer it to the LimitOrderManager

```solidity
if (zeroForOne) {
            limitOrderReserve1 -= amountToClaim;
            _transferToken(token1, msg.sender, amountToClaim);
        } else {
            limitOrderReserve0 -= amountToClaim;
            _transferToken(token0, msg.sender, amountToClaim);
        }

        emit ClaimLimitOrder(tick, amountToClaim, zeroForOne);
    }
```

Next up is the claimLimitOrderTick() function of the Ticks Library

```solidity
function claimLimitOrderTick(
        mapping(int24 => IConcentratedLiquidityPoolStruct.Tick) storage ticks,
        mapping(int24 => IConcentratedLiquidityPoolStruct.LimitOrderTickData) storage limitOrderTicks,
        int24 tick,
        bool zeroForOne,
        uint256 amount,
        int24 nearestTick,
        uint256 tickCount
    ) public returns (int24, uint256) {
```

We deduct the amount from the claimableAmount 

```solidity
IConcentratedLiquidityPoolStruct.Tick storage current = ticks[tick];
if (zeroForOne) {
    limitOrderTicks[tick].token1Claimable -= amount;
} else {
    limitOrderTicks[tick].token0Claimable -= amount;
}
```

We do certain check based on ticks and if they pass we delete the tick

```solidity
if (
            current.nextTick != current.previousTick &&
            current.liquidity == 0 &&
            tick != TickMath.MIN_TICK &&
            tick != TickMath.MAX_TICK &&
            limitOrderTicks[tick].token0Liquidity == 0 &&
            limitOrderTicks[tick].token1Liquidity == 0
        ) {
            // Delete lower tick.
            IConcentratedLiquidityPoolStruct.Tick storage previous = ticks[current.previousTick];
            IConcentratedLiquidityPoolStruct.Tick storage next = ticks[current.nextTick];

            previous.nextTick = current.nextTick;
            next.previousTick = current.previousTick;

            if (nearestTick == tick) nearestTick = current.previousTick;

            delete ticks[tick];
            unchecked {
                tickCount--;
            }
            limitOrderTicks[tick].isActive = false;
        }
        return (nearestTick, tickCount);
    }     
```