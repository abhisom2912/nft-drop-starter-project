# Adding Liquidity

Adding And Removing Liquidity is handled by ConcentratedLiquidityPoolMananger.sol  by its mint and burn functions respectively. 
ConcentratedLiquidityPoolMananger is Dfyn's Concentrated Liquidity Pool periphery contract that combines non-fungible position management and staking.
Lets look at mint function first

### ConcentratedLiquidityPoolMananger:

While adding liquidity Mint function in ConcentratedLiquidityPoolManager is called which mints a new position as a NFT returned as positionId.

The  **`mint`** function is used to mint a new non-fungible token (NFT) or add liquidity to an existing one. The NFT is associated with a Concentrated Liquidity Pool, and its value is determined by the amount of liquidity it holds and the range of prices it covers.

```solidity
/// @notice Mints Position
    /// @param pool address of pool
    /// @param lowerOld tick below lower tick
    /// @param lower lower tick
    /// @param upperOld tick below upper
    /// @param upper upper tick
    /// @param amount0Desired token0 input amount
    /// @param amount1Desired token1 input amount
    /// @param native user wallet/vault
    /// @param minLiquidity min liquidity to mint
    /// @param positionId position nft id

    function mint(
        IConcentratedLiquidityPool pool,
        int24 lowerOld,
        int24 lower,
        int24 upperOld,
        int24 upper,
        uint128 amount0Desired,
        uint128 amount1Desired,
        bool native,
        uint256 minLiquidity,
        uint256 positionId
    ) external payable returns (uint256 _positionId) {
```

The function takes several parameters:

- **`pool`** is the address of the Concentrated Liquidity Pool that the NFT is associated with.
- **`lowerOld`**, **`lower`**, **`upperOld`**, and **`upper`** are ticks that define the range of prices covered by the NFT.
- **`amount0Desired`** and **`amount1Desired`** are the amounts of the two tokens (token0 and token1) that the user wants to add to the NFT.
- **`native`** is a boolean value that indicates whether the user's wallet/vault is a native token or not.
- **`minLiquidity`** is the minimum amount of liquidity that must be added to the NFT in order to mint it.
- **`positionId`** is the ID of the NFT, if it already exists. If it is set to 0, a new NFT will be minted.

```solidity
require(masterDeployer.pools(address(pool)), "INVALID_POOL");
cachedMsgSender = msg.sender;
cachedPool = address(pool);
```

The code first checks that the address of the provided liquidity pool is in the **`pools`** mapping of the **`masterDeployer`** contract. If it is not, the function execution is reverted with the error message "INVALID_POOL". The code then saves the caller's address in the **`cachedMsgSender`** variable and the pool's address in the **`cachedPool`** variable.

```solidity
uint128 liquidityMinted = uint128(
	pool.mint(
	    IConcentratedLiquidityPoolStruct.MintParams({
	        lowerOld: lowerOld,
	        lower: lower,
	        upperOld: upperOld,
	        upper: upper,
	        amount0Desired: amount0Desired,
	        amount1Desired: amount1Desired,
	        native: native
	    })
		)
);

require(liquidityMinted >= minLiquidity, "TOO_LITTLE_RECEIVED");
```

The code first calls the **`mint`** function on the provided liquidity pool, passing in a **`MintParams`** struct with the input values of the **`mint`** function as its fields. This **`mint`** function on the liquidity pool is responsible for creating the new liquidity tokens and returning the amount of tokens minted.

The code then checks that the amount of tokens minted is greater than or equal to the **`minLiquidity`** value provided as input to the **`mint`** function. If it is not, the function execution is reverted with the error message "TOO_LITTLE_RECEIVED".

```solidity
(uint256 feeGrowthInside0, uint256 feeGrowthInside1) = pool.rangeFeeGrowth(lower, upper);
```

The function then calls the**`rangeFeeGrowth`** function on the provided liquidity pool, passing in the **`lower`** and **`upper`** values as input. This **`rangeFeeGrowth`** function returns a tuple of two **`uint256`** values representing the fee growth inside the range defined by the **`lower`** and **`upper`** values. The returned tuple is then assigned to the **`feeGrowthInside0`** and **`feeGrowthInside1`** variables.

These variables are later used to update the **`feeGrowthInside0`** and **`feeGrowthInside1`** fields of a **`Position`** struct, which represents a position in the liquidity pool. The **`feeGrowthInside0`** and **`feeGrowthInside1`** fields of the **`Position`** struct are used to track the fees that have been accrued on the position.

```solidity
if (positionId == 0) {
    // We mint a new NFT.
    _positionId = nftCount.minted;
    positions[_positionId] = Position({
        pool: pool,
        liquidity: liquidityMinted,
        lower: lower,
        upper: upper,
        latestAddition: uint32(block.timestamp),
        feeGrowthInside0: feeGrowthInside0,
        feeGrowthInside1: feeGrowthInside1
    });
    mint(msg.sender);
}
```

The `**positionId**` parameter is checked to see if it is equal to 0. If it is, a new NFT is minted. The **`_positionId`**variable is set to the current value of **`nftCount.minted`**, which is likely a counter of the number of NFTs that have been minted. Then, a new **`Position`**struct is created and added to the **`positions`**mapping using **`_positionId`** as the key. The **`Position`**struct stores information about the NFT, including the address of the liquidity pool where the NFT is minted, the amount of liquidity that was minted, the range of the liquidity pool, the time at which the NFT was last added to, and the `**feeGrowthInside0**` and `**feeGrowthInside1**` values. Finally, the **`mint`**
function is called and passed the caller's address.

```solidity
else {
    // We increase liquidity for an existing NFT.
    _positionId = positionId;
    Position storage position = positions[_positionId];
    require(address(position.pool) == address(pool), "POOL_MIS_MATCH");
    require(position.lower == lower && position.upper == upper, "RANGE_MIS_MATCH");
    require(ownerOf(positionId) == msg.sender, "NOT_ID_OWNER");
    // Fees should be claimed first.
    position.feeGrowthInside0 = feeGrowthInside0;
    position.feeGrowthInside1 = feeGrowthInside1;
    position.liquidity += liquidityMinted;
    // Incentives should be claimed first.
    position.latestAddition = uint32(block.timestamp);
}
```

If the **`positionId`** is not 0, indicating that this is an existing NFT position being added to, the function first checks that the specified **`pool`**address matches the pool address stored in the **`Position`**struct for the given NFT, and that the **`lower`**and **`upper`**bounds match the ones stored in the struct. It then checks that the caller of the function is the owner of the NFT. If all these checks pass, the function updates the **`Position`**struct with the new liquidity added to the pool, the new fee growth rates, and the timestamp of the latest addition.

```solidity
emit IncreaseLiquidity(address(pool), msg.sender, _positionId, liquidityMinted);

cachedMsgSender = address(1);
cachedPool = address(1);
```

After minting or increasing liquidity, the function emits an event called **`IncreaseLiquidity`**
with information about the transaction, then resets local variables for security reasons.

### ConcentratedLiquidityPool

Lets take a look at the mint function of CLP which returns the liquidity minted

We calculate the liquidity minted by DyDxMath library
Mint function changes the state of the ticks  by inserting new ticks in the linked list

```solidity
/// @dev Mints LP tokens - should be called via the CL pool manager contract.
function mint(MintParams memory mintParams) public lock returns (uint256 liquidityMinted) {
```

This code defines a **`mint`** function in a Concentrated Liquidity Pool contract. The function is used to mint new liquidity pool (LP) tokens, which represent a user's share of the liquidity in the pool.

The function takes a **`MintParams`** struct as input, which contains the following fields:

- **`lowerOld`** and **`upperOld`**: the ticks immediately below the range of prices covered by the LP token.
- **`lower`** and **`upper`**: the ticks immediately above the range of prices covered by the LP token.
- **`amount1Desired`**: the amount of token1 that the user wants to add to the pool.
- **`amount0Desired`**: the amount of token0 that the user wants to add to the pool.
- **`native`**: a boolean value that indicates whether the user's wallet/vault is a native token or not.

```solidity
    Validators._ensureTickSpacing(mintParams.lower, mintParams.upper, tickSpacing);
    uint256 priceLower = uint256(TickMath.getSqrtRatioAtTick(mintParams.lower));
    uint256 priceUpper = uint256(TickMath.getSqrtRatioAtTick(mintParams.upper));
    uint256 currentPrice = uint256(price);
```

The **`Validators._ensureTickSpacing()`**function is  being used to ensure that the **`mintParams.lower`**and **`mintParams.upper`**values are valid and meet the criteria for tick spacing

We then fetch the **`priceLower`** and **`priceUpper`** using **`getSqrtRatioAtTick`**

```solidity
liquidityMinted = DyDxMath.getLiquidityForAmounts(
        priceLower,
        priceUpper,
        currentPrice,
        uint256(mintParams.amount1Desired),
        uint256(mintParams.amount0Desired)
    );
```

here we calculate the liquidity that will be minted based on a range of prices (**`priceLower`**
 and **`priceUpper`**), the current price (**`currentPrice`**), and the desired amounts of two assets (**`mintParams.amount1Desired`** and **`mintParams.amount0Desired`**) using **`getLiquidityForAmounts`**  function of **`DyDxMath`** library.

```solidity
    // Ensure no overflow happens when we cast from uint256 to int128.
    if (liquidityMinted > uint128(type(int128).max)) revert Overflow();

    _updateSecondsPerLiquidity(uint256(liquidity));
```

Here we check if the amount of liquidity minted (**`liquidityMinted`**) is greater than the maximum value that can be stored in an **`int128`**data type. If it is, the code will revert with an **`Overflow`** error.

After this check, the code calls a function **`_updateSecondsPerLiquidity`** and passes in the **`liquidity`**  which updates the **`secondsGrowthGlobal`**

```solidity
unchecked {
        (uint256 amount0Fees, uint256 amount1Fees) = _updatePosition(
            msg.sender,
            mintParams.lower,
            mintParams.upper,
            int128(uint128(liquidityMinted))
        );
        if (amount0Fees > 0) {
            _transfer(token0, amount0Fees, msg.sender, false);
            reserve0 -= uint128(amount0Fees);
        }
        if (amount1Fees > 0) {
            _transfer(token1, amount1Fees, msg.sender, false);
            reserve1 -= uint128(amount1Fees);
        }

        if (priceLower <= currentPrice && currentPrice < priceUpper) liquidity += uint128(liquidityMinted);
    }
```

Here we update the positions by calling the **`_updatePosition`** function and passing in the **`mintParams.lower`**, **`mintParams.upper`**, and  **`liquidityMinted`**. This function returns a tuple with the amounts of fees for token0 and token1, which are stored in the **`amount0Fees`** and **`amount1Fees`** variables.

If either of these amounts are greater than 0, the code transfers the appropriate amount of tokens to the sender's account, and then subtracts the transferred amount from the corresponding reserve.

Finally, if the **`currentPrice`** is within the specified range (**`priceLower`** and **`priceUpper`**), the **`liquidity`** variable is incremented by  **`liquidityMinted`**.

```solidity
InsertTickParams memory data = InsertTickParams({
        nearestTick: nearestTick,
        currentPrice: uint160(currentPrice),
        tickCount: tickCount,
        amount: uint128(liquidityMinted)
    });

    (nearestTick, tickCount) = Ticks.insert(
        ticks,
        limitOrderTicks,
        feeGrowthGlobal0,
        feeGrowthGlobal1,
        secondsGrowthGlobal,
        mintParams.lowerOld,
        mintParams.lower,
        mintParams.upperOld,
        mintParams.upper,
        data
    );
```

Then we create a **`InsertTickParams`** struct with the **`nearestTick`**, **`currentPrice`**, **`tickCount`**, and **`amount`**  values and then pass this struct as **`data`** it to  **`Ticks.insert`**along with the other params of the insert function.

```solidity
(uint128 amount0Actual, uint128 amount1Actual) = DyDxMath.getAmountsForLiquidity(
    priceLower,
    priceUpper,
    currentPrice,
    liquidityMinted,
    true
);
```

Then we  calls the **`DyDxMath.getAmountsForLiquidity`**function and passes in the **`priceLower`**, **`priceUpper`**, **`currentPrice`**, **`liquidityMinted`**, and **`true`**values. This function calculates the amounts of tokens that should be minted for a given liquidity and price range.

The function returns a tuple containing the calculated amounts of tokens for each token, which are stored in the **`amount0Actual`** and **`amount1Actual`**  variables.

```solidity
IPositionManager(msg.sender).mintCallback(token0, token1, amount0Actual, amount1Actual, mintParams.native);
```

It then calls the **`mintCallback`** function of the position manager contract to update the users position. 

```solidity

    if (amount0Actual != 0) {
        reserve0 += amount0Actual;
        if (reserve0 + limitOrderReserve0 > _balance(token0)) revert Token0Missing();
    }

    if (amount1Actual != 0) {
        reserve1 += amount1Actual;
        if (reserve1 + limitOrderReserve1 > _balance(token1)) revert Token1Missing();
    }

    emit Mint(msg.sender, amount0Actual, amount1Actual, mintParams.lower, mintParams.upper);
}
```

Then we check if the **`amount0Actual`** and **`amount1Actual`** variables are not equal to 0, and if so, increments the corresponding **`reserve`**  by the calculated amount.

After this, we check  if the sum of the **`reserve`**  and the corresponding **`limitOrderReserve`**  is greater than the balance of the corresponding token. If this is the case, the code will revert with a **`Token0Missing`** or **`Token1Missing`** error, depending on which token is missing.

Finally, it emits a **`Mint`** event to indicate that new LP tokens have been minted.

### Ticks.sol

Next in line is the insert function from Ticks library

The require statements make sure that the ticks are placed in the correct range, where,
**`MIN_TICK = -887272`**   and   **`MAX_TICK = 887272`** 

```solidity
function insert(
        mapping(int24 => IConcentratedLiquidityPoolStruct.Tick) storage ticks,
        mapping(int24 => ILimitOrderStruct.LimitOrderTickData) storage limitOrderTicks,
        uint256 feeGrowthGlobal0,
        uint256 feeGrowthGlobal1,
        uint160 secondsGrowthGlobal,
        int24 lowerOld,
        int24 lower,
        int24 upperOld,
        int24 upper,
        IConcentratedLiquidityPoolStruct.InsertTickParams memory data
    ) public returns (int24, uint256) {
        
```

This function appears to be used for inserting data into two mappings: 

**`ticks`**and **`limitOrderTicks`**. The **`lowerOld`**, **`lower`**, **`upperOld`**, and **`upper`**
 parameters are used to specify the range of ticks to insert the data into. The **`data`**
 parameter is passed with **`InsertTickParams`** Struct.

```solidity
require(lower < upper, "WRONG_ORDER");
require(TickMath.MIN_TICK <= lower, "LOWER_RANGE");
require(upper <= TickMath.MAX_TICK, "UPPER_RANGE");
```

These **`require`** statements are used to ensure that the input is valid. The first **`require`** statement checks that **`lower`** is less than **`upper`**. This is necessary because the input data must be inserted into a range of ticks, and this range must be well-defined (i.e., it must have a lower and an upper bound). If **`lower`** is not less than **`upper`**, it means that the range is not well-defined, and an error message is thrown.

The second and third **`require`** statements check that **`lower`** and **`upper`** are within the minimum and maximum tick range, respectively. This is necessary because the input data can only be inserted into ticks within this range. If **`lower`** or **`upper`** are outside of this range, an error message is thrown.

```solidity
uint128 currentLowerLiquidity = ticks[lower].liquidity;
```

Then we update the **`liquidity`** field of the **`ticks[lower]`**mapping. The **`currentLowerLiquidity`** variable is used to store the current liquidity at the **`lower`** tick.

```solidity
if (currentLowerLiquidity != 0 || lower == TickMath.MIN_TICK || limitOrderTicks[lower].isActive) {
    // We are adding liquidity to an existing tick.
    ticks[lower].liquidity = currentLowerLiquidity + data.amount;
    if (currentLowerLiquidity == 0 || lower != TickMath.MIN_TICK) {
        if (lower <= data.nearestTick) {
            ticks[lower].feeGrowthOutside0 = feeGrowthGlobal0;
            ticks[lower].feeGrowthOutside1 = feeGrowthGlobal1;
            ticks[lower].secondsGrowthOutside = secondsGrowthGlobal;
        }
    }
}
```

Here we are adding liquidity to an existing tick. It does this by checking whether the current liquidity at the **`lower`** tick is not zero or **`lower`** is equal to the minimum tick value, or the **`isActive`** field of the **`limitOrderTicks[lower]`** mapping is **`true`**. If any of these conditions are true, we add the **`amount`** field of the **`data`** struct to the current liquidity at the **`lower`** tick.

Additionally, if the current liquidity at the **`lower`** tick is zero and **`lower`** is not equal to the minimum tick value, we set the **`feeGrowthOutside0`**, **`feeGrowthOutside1`**, and **`secondsGrowthOutside`** fields of the **`ticks[lower]`** mapping to the **`feeGrowthGlobal0`**, **`feeGrowthGlobal1`**, and **`secondsGrowthGlobal`** values, respectively. This is only done if **`lower`** is less than or equal to the **`nearestTick`** field of the **`data`** struct. 

```solidity
else {
    // We are inserting a new tick.
    IConcentratedLiquidityPoolStruct.Tick storage old = ticks[lowerOld];
    int24 oldNextTick = old.nextTick;

    require(
        (old.liquidity != 0 || lowerOld == TickMath.MIN_TICK || limitOrderTicks[lowerOld].isActive) &&
            lowerOld < lower &&
            lower < oldNextTick,
        "LOWER_ORDER"
    );

    if (lower <= data.nearestTick) {
        ticks[lower] = IConcentratedLiquidityPoolStruct.Tick(
            lowerOld,
            oldNextTick,
            data.amount,
            feeGrowthGlobal0,
            feeGrowthGlobal1,
            secondsGrowthGlobal
        );
    } else {
        ticks[lower] = IConcentratedLiquidityPoolStruct.Tick(lowerOld, oldNextTick, data.amount, 0, 0, 0);
    }

    old.nextTick = lower;
    ticks[oldNextTick].previousTick = lower;
    unchecked {
        data.tickCount++;
    }
}
```

Then we are  inserting a new tick into the **`ticks`** mapping. It is done by first checking that the current liquidity at the **`lowerOld`** tick is not zero or **`lowerOld`** is equal to the minimum tick value, or the **`isActive`** field of the **`limitOrderTicks[lowerOld]`** mapping is **`true`**, and that **`lowerOld`** is less than **`lower`** and **`lower`** is less than the next tick after **`lowerOld`**. If any of these conditions are not met, an error message is thrown.

If **`lower`** is less than or equal to the **`nearestTick`** field of the **`data`**  struct, then we create a new **`Tick`** struct  with the **`lowerOld`**, next tick after **`lowerOld`**, **`amount`**, **`feeGrowthGlobal0`**, **`feeGrowthGlobal1`**, and **`secondsGrowthGlobal`** fields, and inserts it into the **`ticks[lower]`** mapping. Otherwise, we create a new **`Tick`** struct  with the **`lowerOld`**, next tick after **`lowerOld`**, and **`amount`** fields, and sets the **`feeGrowthOutside0`**, **`feeGrowthOutside1`**, and **`secondsGrowthOutside`** fields to zero.

Finally, we update the **`nextTick`** field of the **`old`** tick and the **`previousTick`** field of the next tick after **`old`**, and increments the **`tickCount`** field of the **`data`** struct. This  is used for maintaining the linked list of ticks.

```solidity
	
	uint128 currentUpperLiquidity = ticks[upper].liquidity;
  if (currentUpperLiquidity != 0 || upper == TickMath.MAX_TICK || limitOrderTicks[upper].isActive) {
      // We are adding liquidity to an existing tick.
      ticks[upper].liquidity = currentUpperLiquidity + data.amount;
      if (currentUpperLiquidity == 0 || upper != TickMath.MAX_TICK) {
          if (upper <= data.nearestTick) {
              ticks[upper].feeGrowthOutside0 = feeGrowthGlobal0;
              ticks[upper].feeGrowthOutside1 = feeGrowthGlobal1;
              ticks[upper].secondsGrowthOutside = secondsGrowthGlobal;
          }
      }
  } else {
      // Inserting a new tick.
      IConcentratedLiquidityPoolStruct.Tick storage old = ticks[upperOld];
      int24 oldNextTick = old.nextTick;

      require(
          (old.liquidity != 0 || limitOrderTicks[upperOld].isActive || upperOld == TickMath.MAX_TICK) &&
              oldNextTick > upper &&
              upperOld < upper,
          "UPPER_ORDER"
      );

      if (upper <= data.nearestTick) {
          ticks[upper] = IConcentratedLiquidityPoolStruct.Tick(
              upperOld,
              oldNextTick,
              data.amount,
              feeGrowthGlobal0,
              feeGrowthGlobal1,
              secondsGrowthGlobal
          );
      } else {
          ticks[upper] = IConcentratedLiquidityPoolStruct.Tick(upperOld, oldNextTick, data.amount, 0, 0, 0);
      }
      old.nextTick = upper;
      ticks[oldNextTick].previousTick = upper;
      unchecked {
          data.tickCount++;
      }
  }
```

Same procedure is carried out for Upper tick.

```solidity
int24 tickAtPrice = TickMath.getTickAtSqrtRatio(data.currentPrice);

        if (data.nearestTick < upper && upper <= tickAtPrice) {
            unchecked {
                data.nearestTick = upper;
            }
        } else if (data.nearestTick < lower && lower <= tickAtPrice) {
            unchecked {
                data.nearestTick = lower;
            }
        }

return (data.nearestTick, data.tickCount);
```

The **`tickAtPrice`**variable is calculated using the **`getTickAtSqrtRatio`**method from the **`TickMath`**
We check if the **`nearestTick`** value is less than the **`upper`**or **`lower`**value and, if it is, sets the **`nearestTick`**value to the **`upper`**or **`lower`**value, respectively. Finally, we return a tuple containing the updated **`nearestTick`**and **`tickCount`** values.

### mintCallBack

Callback function that is triggered when a mint operation is performed on a token.

```solidity
function mintCallback(
        address token0,
        address token1,
        uint256 amount0,
        uint256 amount1,
        bool native
    ) external override {
       
```

The function takes in several parameters, including the addresses of two tokens (**`token0`** and **`token1`**), the amount of each token being minted (**`amount0`** and **`amount1`**), and a boolean value indicating  native  token of the current chain (**`native`**).

```solidity
require(msg.sender == cachedPool, "UNAUTHORIZED_CALLBACK");
if (native) {
    _depositFromUserToVault(token0, cachedMsgSender, msg.sender, amount0);
    _depositFromUserToVault(token1, cachedMsgSender, msg.sender, amount1);
} else {
    vault.transfer(token0, cachedMsgSender, msg.sender, amount0);
    vault.transfer(token1, cachedMsgSender, msg.sender, amount1);
}
cachedMsgSender = address(1);
cachedPool = address(1);
}
```

The function first checks that the caller of the function is the expected **`cachedPool`** address, and throws an error if it is not. If the **`native`** parameter is **`true`**, the function calls the **`_depositFromUserToVault`**method for both the **`token0`** and **`token1`** tokens, passing in the **`amount0`** and **`amount1`** amounts, respectively. Otherwise, the function calls the **`transfer`**
 method of the **`vault`** object to transfer the **`token0`** and **`token1`** tokens from the **`cachedMsgSender`** address to the **`cachedPool`** address. Finally, the function resets the **`cachedMsgSender`** and **`cachedPool`** values to the address **`1`.**