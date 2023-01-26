# Cancelling Limit Order

The process of cancelling the limit order starts from LimitOrderManager by calling the cancelLimitOrder 

First we check whether msg.sender is the owner of the tokenId if yes then we proceed further to fetch the data about the limit order. After that we check if the order is still active or not.

```solidity
/// @notice Function to cancel limitorder
/// @param tokenId limitorder id
/// @param unwrapVault to user wallet/vault
function cancelLimitOrder(uint256 tokenId, bool unwrapVault) public {
	require(msg.sender == limitOrderToken.ownerOf(tokenId), "NOT_ID_OWNER");
	
	LimitOrder memory limitOrder = limitOrders[tokenId];
	
	require(limitOrder.status != LimitOrderStatus.closed, "Limit Order:Inactive");
```

First we fetch the totalClaimableAmount  and then calculate the refundAmount based on zeroForOne. In this function we call another function named cancelLimitOrder of CLP if that returns as a success  we transfer  the refund amount back.

```solidity
(, , , , , , address _token0, address _token1) = limitOrder.pool.getImmutables();

uint256 totalClaimableAmount = limitOrder.amountOut - limitOrder.claimedAmount;
uint256 refundAmount;
if (limitOrder.zeroForOne) {
// refundAmount = (totalClaimableAmount * 10**12) / limitOrder.price;
refundAmount = FullMath.mulDivRoundingUp(totalClaimableAmount, 2**96, limitOrder.sqrtpriceX96);
refundAmount = FullMath.mulDivRoundingUp(refundAmount, 2**96, limitOrder.sqrtpriceX96);
limitOrder.pool.cancelLimitOrder(refundAmount, limitOrder.tick, limitOrder.zeroForOne);
_transferOut(_token0, msg.sender, refundAmount, unwrapVault);
} else {
// refundAmount = (limitOrder.price * totalClaimableAmount) / (10**12);
refundAmount = FullMath.mulDivRoundingUp(limitOrder.sqrtpriceX96, totalClaimableAmount, 2**96);
refundAmount = FullMath.mulDivRoundingUp(limitOrder.sqrtpriceX96, refundAmount, 2**96);
limitOrder.pool.cancelLimitOrder(refundAmount, limitOrder.tick, limitOrder.zeroForOne);
_transferOut(_token1, msg.sender, refundAmount, unwrapVault);
}
```

Then we burn the tokenId and update the status of limit order as closed.

```solidity
limitOrderToken.burn(tokenId);
  limitOrders[tokenId].status = LimitOrderStatus.closed;

  emit CancelLimitOrder(address(limitOrder.pool), tokenId, refundAmount);
}
```

Lets take a look at the cancelLimitOrder function of CLP

Based on zeroForOne first we check whether available liquidity on that tick is greater than the amount

```solidity
function cancelLimitOrder(
        uint256 amount,
        int24 tick,
        bool zeroForOne
    ) external onlyLimitOrderManager lock {
    require(zeroForOne ? limitOrderTicks[tick].token0Liquidity >= amount : limitOrderTicks[tick].token1Liquidity >= amount, "E3");
```

 

Then we call the cancelLimitOrderTick which returns the neartestTick and tickCount. Then we deduct the amount from the reserve accordingly 

```solidity
(nearestTick, tickCount) = Ticks.cancelLimitOrderTick(ticks, limitOrderTicks, tick, zeroForOne, amount, nearestTick, tickCount);

        if (zeroForOne) {
            limitOrderReserve0 -= amount;
            _transferToken(token0, msg.sender, amount);
        } else {
            limitOrderReserve1 -= amount;
            _transferToken(token1, msg.sender, amount);
        }
        emit CancelLimitOrder(tick, amount, zeroForOne);
    }
```