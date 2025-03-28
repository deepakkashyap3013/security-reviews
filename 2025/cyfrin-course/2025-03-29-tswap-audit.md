## High

### [H-1] `TSwapPool::deposit()` is missing deadline check, resulting txns to complete even after the given deadline 

**<ins>Description:</ins>** 
- The `TSwapPool::deposit()` accepts a `deadline` parameter, which according to documentation `The deadline for the transaction to be completed by`. However, this parameter is never been used.
- As a consequence, operations that add liquidity to the pool might be executed at unexpected times, in the market conditions where the deposit rate is unfavourable.


**<ins>Impact:</ins>** 
- Transactions could be sent when the market conditions are unfavourable to deposit, even when adding a deadline parameter

**<ins>Proof of Concept:</ins>**
- The `deadline` parameter is unused.

**<ins>Recommended Mitigation:</ins>** 
- Consider making the following change to the function.

```diff
    function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
    )
        external
+       revertIfDeadlinePassed(deadline)
        revertIfZero(wethToDeposit)
        returns (uint256 liquidityTokensToMint)
    {
```

### [H-2] Incorrect fee calculation in `TSwapPool::getInputAmountBasedOnOutput()` causes protocol to take too many tokens from users, resulting in lost fees

**<ins>Description:</ins>** 
- The `getInputAmountBasedOnOutput` function is intended to calculate the amount of tokens a user should deposit given an amount of tokens of output tokens. However, the function currently miscalculates the resulting amount. When calculating the fee, it scales the amount by `10_000` instead of `1_000`.

**<ins>Impact:</ins>** 
- Protocol takes more fees than expected from users.

**<ins>Proof of Concept:</ins>**
- Users swapping tokens via the `swapExactOutput` function will pay far more tokens than expected for their trades. This becomes particularly risky for users that provide infinite allowance to the `TSwapPool` contract. Moreover, note that the issue is worsened by the fact that the `swapExactOutput` function does not allow users to specify a maximum of input tokens, as is described in another issue in this report.

- It's worth noting that the tokens paid by users are not lost, but rather can be swiftly taken by liquidity providers. Therefore, this contract could be used to trick users, have them swap their funds at unfavorable rates and finally rug pull all liquidity from the pool.

- To test this, include the following code in the TSwapPool.t.sol file:

<details>

<summary>POC</summary>

```js
    function testFlawedSwapExactOutput() public {
        uint256 initialLiquidity = 100e18;
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), initialLiquidity);
        poolToken.approve(address(pool), initialLiquidity);

        pool.deposit({
            wethToDeposit: initialLiquidity,
            minimumLiquidityTokensToMint: 0,
            maximumPoolTokensToDeposit: initialLiquidity,
            deadline: uint64(block.timestamp)
        });
        vm.stopPrank();

        // User has 11 pool tokens
        address someUser = makeAddr("someUser");
        uint256 userInitialPoolTokenBalance = 11e18;
        poolToken.mint(someUser, userInitialPoolTokenBalance);
        vm.startPrank(someUser);
        
        // Users buys 1 WETH from the pool, paying with pool tokens
        poolToken.approve(address(pool), type(uint256).max);
        pool.swapExactOutput(
            poolToken,
            weth,
            1 ether,
            uint64(block.timestamp)
        );

        // Initial liquidity was 1:1, so user should have paid ~1 pool token
        // However, it spent much more than that. The user started with 11 tokens, and now only has less than 1.
        assertLt(poolToken.balanceOf(someUser), 1 ether);
        vm.stopPrank();

        // The liquidity provider can rug all funds from the pool now,
        // including those deposited by user.
        vm.startPrank(liquidityProvider);
        pool.withdraw(
            pool.balanceOf(liquidityProvider),
            1, // minWethToWithdraw
            1, // minPoolTokensToWithdraw
            uint64(block.timestamp)
        );

        assertEq(weth.balanceOf(address(pool)), 0);
        assertEq(poolToken.balanceOf(address(pool)), 0);
    }
```


</details>

**<ins>Recommended Mitigation:</ins>** 

```diff
    function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
-        return ((inputReserves * outputAmount) * 10_000) / ((outputReserves - outputAmount) * 997);
+        return ((inputReserves * outputAmount) * 1_000) / ((outputReserves - outputAmount) * 997);
    }
```


### [H-3] Lack of slippage protection in `TSwapPool::swapExactOutput()` causes users to potentially receive way fewer tokens

**<ins>Description:</ins>** 
- The `swapExactOutput` function does not include any sort of slippage protection. This function is similar to what is done in `TSwapPool::swapExactInput`, where the function specifies a `minOutputAmount`, the `swapExactOutput` function should specify a `maxInputAmount`.

**<ins>Impact:</ins>** 
- If market conditions change before the transaciton processes, the user could get a much worse swap.

**<ins>Proof of Concept:</ins>**
1. The price of 1 WETH right now is 1,000 USDC
2. User inputs a swapExactOutput looking for 1 WETH
    1. inputToken = USDC
    2. outputToken = WETH
    3. outputAmount = 1
    4. deadline = whatever
3. The function does not offer a maxInput amount
4. As the transaction is pending in the mempool, the market changes! And the price moves HUGE -> 1 WETH is now 10,000 USDC. 10x more than the user expected
5. The transaction completes, but the user sent the protocol 10,000 USDC instead of the expected 1,000 USDC


**<ins>Recommended Mitigation:</ins>** 

```diff
function swapExactOutput(
    IERC20 inputToken,
+   uint256 maxInputAmount    
    IERC20 outputToken,
    uint256 outputAmount,
    uint64 deadline
)
    public
    revertIfZero(outputAmount)
    revertIfDeadlinePassed(deadline)
    returns (uint256 inputAmount)
{
    uint256 inputReserves = inputToken.balanceOf(address(this));
    uint256 outputReserves = outputToken.balanceOf(address(this));

    inputAmount = getInputAmountBasedOnOutput(outputAmount, inputReserves, outputReserves);

+   if (inputAmount > maxInputAmount) {
+       revert TSwapPool__OutputTooHigh(inputAmount, maxInputAmount);
+   }

    _swap(
        inputToken,
        inputAmount,
        outputToken,
        outputAmount
    );
}
```


### [H-4] `TSwapPool::sellPoolTokens()` mismatches input and output tokens causing users to receive the incorrect amount of tokens

**<ins>Description:</ins>** 
- The `sellPoolTokens` function is intended to allow users to easily sell pool tokens and receive `WETH` in exchange. Users indicate how many pool tokens they're willing to sell in the `poolTokenAmount` parameter. However, the function currently miscalculaes the swapped amount.
- This is due to the fact that the `swapExactOutput` function is called, whereas the `swapExactInput` function is the one that should be called. Because users specify the exact amount of input tokens, not output.


**<ins>Impact:</ins>** 
- Users will swap the wrong amount of tokens, which is a severe disruption of protcol functionality.

**<ins>Proof of Concept:</ins>**

**<ins>Recommended Mitigation:</ins>** 
Consider changing the implementation to use swapExactInput instead of swapExactOutput. Note that this would also require changing the sellPoolTokens function to accept a new parameter (ie minWethToReceive to be passed to swapExactInput)

```diff
    function sellPoolTokens(
        uint256 poolTokenAmount,
+       uint256 minWethToReceive,    
        ) external returns (uint256 wethAmount) {
-       return swapExactOutput(i_poolToken, i_wethToken, poolTokenAmount, uint64(block.timestamp));
+       return swapExactInput(i_poolToken, poolTokenAmount, i_wethToken, minWethToReceive, uint64(block.timestamp));
    }
```


### [H-5] In `TSwapPool::_swap()` the extra tokens given to users after every swapCount breaks the protocol invariant of `x * y = k`

**<ins>Description:</ins>** 
The protocol follows a strict invariant of `x * y = k`. Where:

- `x`: The balance of the pool token
- `y`: The balance of WETH
- `k`: The constant product of the two balances

This means, that whenever the balances change in the protocol, the ratio between the two amounts should remain constant, hence the k. However, this is broken due to the extra incentive in the `_swap` function. Meaning that over time the protocol funds will be drained.

The follow block of code is responsible for the issue.

```js
@>  swap_count++;
    if (swap_count >= SWAP_COUNT_MAX) {
        swap_count = 0;
@>      outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
    }
```

**<ins>Impact:</ins>** 
- A user could maliciously drain the protocol of funds by doing a lot of swaps and collecting the extra incentive given out by the protocol.

**<ins>Proof of Concept:</ins>**

1. A user swaps 10 times, and collects the extra incentive of 1_000_000_000_000_000_000 tokens
2. That user continues to swap untill all the protocol funds are drained

<details>

<summary> POC </summary>

```js
    function testInvariantBroken() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        uint256 outputWeth = 1e17;

        vm.startPrank(user);
        poolToken.approve(address(pool), type(uint256).max);
        poolToken.mint(user, 100e18);
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));

        int256 startingY = int256(weth.balanceOf(address(pool)));
        int256 expectedDeltaY = int256(-1) * int256(outputWeth);

        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        vm.stopPrank();

        uint256 endingY = weth.balanceOf(address(pool));
        int256 actualDeltaY = int256(endingY) - int256(startingY);
        assertEq(actualDeltaY, expectedDeltaY);
    }
```

</details>

**<ins>Recommended Mitigation:</ins>** 
Remove the extra incentive mechanism. If you want to keep this in, we should account for the change in the x * y = k protocol invariant. Or, we should set aside tokens in the same way we do with fees.

```diff
-    swap_count++;
-    if (swap_count >= SWAP_COUNT_MAX) {
-        swap_count = 0;
-        outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
-    }
```



## Low

### [L-1] `TSwapPool::_addLiquidityMintAndTransfer()` event has parameters out of order

**<ins>Description:</ins>** 
- When the LiquidityAdded event is emitted in the `TSwapPool::_addLiquidityMintAndTransfer()` function, it logs values in an incorrect order. The `poolTokensToDeposit` value should go in the third parameter position, whereas the `wethToDeposit` value should go second.

**<ins>Impact:</ins>** 
- Event emission is incorrect, leading to off-chain functions potentially malfunctioning.

**<ins>Proof of Concept:</ins>**

**<ins>Recommended Mitigation:</ins>** 

```diff
- emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);
+ emit LiquidityAdded(msg.sender, wethToDeposit, poolTokensToDeposit);
```

### [L-2] Default value returned by `TSwapPool::swapExactInput()` results in incorrect return value given

**<ins>Description:</ins>** 
- The `swapExactInput` function is expected to return the actual amount of tokens bought by the caller. However, while it declares the named return value `ouput` it is never assigned a value, nor uses an explict return statement.

**<ins>Impact:</ins>** 
- The return value will always be 0, giving incorrect information to the caller.

**<ins>Proof of Concept:</ins>**

**<ins>Recommended Mitigation:</ins>** 
```diff
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

-       uint256 outputAmount = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves);
+       output = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves);

-       if (outputAmount < minOutputAmount) {
-           revert TSwapPool__OutputTooLow(outputAmount, minOutputAmount);
+       if (output < minOutputAmount) {
+           revert TSwapPool__OutputTooLow(output, minOutputAmount);
        }

-       _swap(inputToken, inputAmount, outputToken, outputAmount);
+       _swap(inputToken, inputAmount, outputToken, output);
    }
```



## Gas

### [G-1] In `TSwapPool::deposit()` unused variable should be removed for considering gas efficiency

```diff
-   uint256 poolTokenReserves = i_poolToken.balanceOf(address(this));
```

### [G-2] In `TSwapPool::swapExactInput()` the function visibility should be changed to `external`, which would be gas efficient than public

```diff
    function swapExactInput(
        IERC20 inputToken,
        uint256 inputAmount,
        IERC20 outputToken,
        uint256 minOutputAmount,
        uint64 deadline
    )
+       external
-       public
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
        returns (
            uint256 output
        )
    {
```

## Informational/Non-Critical

### [I-1] import IERC20 interface should be maintain consistency

**<ins>Description:</ins>** 

Since this project uses openzeppelin package, it is good to maintain consistency in importing third party interfaces/contracts from the same package, however it is found that in `PoolFactory` contract the `IERC20` interface is imported from `forge-std` lib, but in `TSwapPool` contract the same interface is imported from openzepplin.

```js
import { IERC20 } from "forge-std/interfaces/IERC20.sol";
```

**<ins>Recommended Mitigation:</ins>** 

The interface `IERC20` should be imported from openzepplin's package

```diff
- import { IERC20 } from "forge-std/interfaces/IERC20.sol";
+ import { IERC20 } from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
```

### [I-2] Error `PoolFactory::PoolFactory__PoolDoesNotExist` is not used and should be removed

```diff
- error PoolFactory__PoolDoesNotExist(address tokenAddress);
```

### [I-3] Lack a zero address check while initializing `PoolFactory` and `TSwapPool` contract

In [PoolFactory.sol](../src/PoolFactory.sol)
```diff
    constructor(address wethToken) {
+       if(wethToken == address(0)){
+          revert();
+       }
        i_wethToken = wethToken;
    }
```

In [TSwapPool.sol](../src/TSwapPool.sol)

```diff
    constructor(
        address poolToken,
        address wethToken,
        string memory liquidityTokenName,
        string memory liquidityTokenSymbol
    )
        ERC20(liquidityTokenName, liquidityTokenSymbol)
    {
+       if(poolToken == address(0) || wethToken == address(0)){
+          revert();
+       }
        i_wethToken = IERC20(wethToken);
        i_poolToken = IERC20(poolToken);
    }
```

### [I-4] `PoolFactory::createPool()` should use `.symbol()` instead of `.name()`

```diff
- string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).name());
+ string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).symbol());
```

### [I-5] Constant variable `TSwapPool::MINIMUM_WETH_LIQUIDITY` should not be emitted in the event

```diff
-   error TSwapPool__WethDepositAmountTooLow(uint256 minimumWethDeposit, uint256 wethToDeposit);
+   error TSwapPool__WethDepositAmountTooLow(uint256 wethToDeposit);
.
.
.
    if (wethToDeposit < MINIMUM_WETH_LIQUIDITY) {
-       revert TSwapPool__WethDepositAmountTooLow(MINIMUM_WETH_LIQUIDITY, wethToDeposit);
+       revert TSwapPool__WethDepositAmountTooLow(wethToDeposit);
    }
```

### [I-6] Constant variable `TSwapPool::MINIMUM_WETH_LIQUIDITY` should not be emitted in the event

```diff
-   error TSwapPool__WethDepositAmountTooLow(uint256 minimumWethDeposit, uint256 wethToDeposit);
+   error TSwapPool__WethDepositAmountTooLow(uint256 wethToDeposit);
.
.
.
    if (wethToDeposit < MINIMUM_WETH_LIQUIDITY) {
-       revert TSwapPool__WethDepositAmountTooLow(MINIMUM_WETH_LIQUIDITY, wethToDeposit);
+       revert TSwapPool__WethDepositAmountTooLow(wethToDeposit);
    }
```

### [I-7] In `TSwapPool::deposit()` Variable update should be made prior to external interaction to follow CEI

In [TSwapPool.sol](../src/TSwapPool.sol)
```diff
    } else {
        // This will be the "initial" funding of the protocol. We are starting from blank here!
        // We just have them send the tokens in, and we mint liquidity tokens based on the weth
+       liquidityTokensToMint = wethToDeposit;
        _addLiquidityMintAndTransfer(wethToDeposit, maximumPoolTokensToDeposit, wethToDeposit);
-       liquidityTokensToMint = wethToDeposit;
    }
```


### [I-8] Literal Instead of Constant

Define and use `constant` variables instead of using literals. If the same constant literal value is used multiple times, create a constant state variable and reference it throughout the contract.

<details><summary>4 Found Instances</summary>


Found in [TSwapPool.sol](../src/TSwapPool.sol)

```js
    uint256 inputAmountMinusFee = inputAmount * 997;
```

```js
    return ((inputReserves * outputAmount) * 10000) / ((outputReserves - outputAmount) * 997);
```

```js
    1e18, i_wethToken.balanceOf(address(this)), i_poolToken.balanceOf(address(this))
```

```js
    1e18, i_poolToken.balanceOf(address(this)), i_wethToken.balanceOf(address(this))
```

</details>

# Signature
*Deepak kashyap aka (Cynefin)* 
*Date: 2025/03/29*