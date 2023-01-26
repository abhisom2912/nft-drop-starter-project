# Creating a Pool

The pool deployment process start at the  MasterDeployer  Contract

### IMasterDeployer.sol  :

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later

pragma solidity >=0.8.0;

/// @notice Dfyn pool deployer interface.
interface IMasterDeployer {
    function dfynFee() external view returns (uint256);

    function limitOrderFee() external view returns (uint256);

    function dfynFeeTo() external view returns (address);

    function vault() external view returns (address);

    function migrator() external view returns (address);

    function pools(address pool) external view returns (bool);

    function deployPool(address factory, bytes calldata deployData) external returns (address);

    function protocolFee() external view returns (uint256, uint256);
}
```

Since we can have multiple type of pools, where each type has its own unique factory MasterDeployer Contract is used to deploy pools.
For now we’ll be deploying Concentrated Liquidity Pool.

```solidity
function deployPool(address _factory, bytes calldata _deployData) external returns (address pool) {
      }
```

The **`deployPool`** function is used to deploy a new pool using a factory contract. The function takes two arguments: an **`address`** for the factory contract that will be used to deploy the pool, and **`bytes calldata`** for the deployment data that will be passed to the factory contract.

```solidity
 if (!whitelistedFactories[_factory]) revert NotWhitelisted();
```

Before deploying the pool, the function checks if the factory contract is on the whitelist by checking the **`whitelistedFactories`** mapping. If the factory contract is not on the whitelist, the function will revert with the **`NotWhitelisted`** error.

```solidity
pool = IPoolFactory(_factory).deployPool(_deployData);
pools[pool] = true;
emit DeployPool(_factory, pool, _deployData);
```

If the factory contract is on the whitelist, the function calls the **`deployPool`** function on the factory contract, passing in the deployment data as an argument. The address of the newly deployed pool is then stored in the **`pools`** mapping and an event is emitted to indicate that a new pool has been deployed. The function returns the address of the newly deployed pool.

---

### ConcentratedLiquidityPoolFactory.sol :

Lets take a look at the  **`deployPool`** function on the factory contract.

```solidity
function deployPool(bytes memory _deployData) external returns (address pool) {
        address[] memory tokens;

        (address tokenA, address tokenB, uint160 price, uint24 tickSpacing) = abi.decode(_deployData, (address, address, uint160, uint24));

        // Revert instead of switching tokens and inverting price.
        if (tokenA > tokenB) revert WrongTokenOrder();

        // Strips any extra data.
        // Don't include price in _deployData to enable predictable address calculation.
        _deployData = abi.encode(tokenA, tokenB, tickSpacing);
        bytes32 salt = keccak256(abi.encode(tokenA, tokenB));
        tokens = new address[](2);
        tokens[0] = tokenA;
        tokens[1] = tokenB;
        (pool) = ConcentratedPoolDeployer(deployer).deployConcentratedPool(salt);
        _registerPool(pool, tokens, keccak256(_deployData));
        IConcentratedLiquidityPool(pool).initialize(_deployData, masterDeployer, limitOrderManager);
        IConcentratedLiquidityPool(pool).setPrice(price);
        IConcentratedLiquidityPool(pool).updateSwapFee(MIN_SWAP_FEE);
        IConcentratedLiquidityPool(pool).updateProtocolFee();
    }

```

The **`deployPool`** function is used to deploy a new concentrated liquidity pool using the **`ConcentratedPoolDeployer`** contract. The function takes one argument: **`bytes memory`** for the deployment data that will be passed to the **`ConcentratedPoolDeployer`** contract.

The function begins by decoding the deployment data using the **`abi.decode`** function. The deployment data should contain the addresses of two tokens, a price, and a tick spacing. The function then checks if the tokens are in the correct order, and if not, it reverts with the **`WrongTokenOrder`** error.

Next, the function strips any extra data from the deployment data, and calculates a **`salt`** value by calling the **`keccak256`** function on the two token addresses. The function then deploys a new concentrated liquidity pool using the **`ConcentratedPoolDeployer`** contract, passing in the **`salt`** value as an argument. The address of the newly deployed pool is stored in the **`pool`** variable.

The function then registers the new pool by calling the **`_registerPool`** function, passing in the pool address, an array of the two token addresses, and the **`keccak256`** hash of the deployment data. The function then initializes the pool by calling the **`initialize`** function on the pool contract, passing in the deployment data, the address of the **`masterDeployer`**, and the address of the **`limitOrderManager`**. The function then sets the pool's price by calling the **`setPrice`** function, and sets the swap fee by calling the **`updateSwapFee`** function. Finally, the function updates the protocol fee by calling the **`updateProtocolFee`** function. The function returns the address of the newly deployed pool.

### ConcentratedPoolDeployer.sol

Lets take at the deployConcentratedPool function called by CLPFactory.

```solidity

    function deployConcentratedPool(bytes32 salt) external returns (address pool) {
        factory = msg.sender;
        pool = address(new ConcentratedLiquidityPool{salt: salt}());
    }
```

The **`deployConcentratedPool`** function is used to deploy a new concentrated liquidity pool contract. The function takes one argument: **`bytes32`** for the **`salt`** value that will be used to create a unique address for the pool contract.

The function begins by storing the caller's address in the **`factory`** variable. This will be the address of the factory contract that is calling this function. The function then creates a new instance of the **`ConcentratedLiquidityPool`** contract, passing in the **`salt`** value as an argument to the contract's constructor. The address of the newly deployed contract is stored in the **`pool`** variable, and the function returns this value