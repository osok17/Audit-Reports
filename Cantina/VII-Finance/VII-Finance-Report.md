# [VII-Finance Report](https://cantina.xyz/code/eb93d215-e328-4d19-99ab-6c510acbb5aa/overview)

| ID | Title |
|:--:|:---|
| [H-1](#h-1-missing-decimal-normalization-in-getsqrtratiox96-causes-incorrect-collateral-valuation) | Missing decimal normalization in `getSqrtRatioX96` causes incorrect collateral valuation |

# [H-1] Missing decimal normalization in `getSqrtRatioX96` causes incorrect collateral valuation
## Summary
The `ERC721WrapperBase.getSqrtRatioX96()` function fails to normalize token decimals when converting oracle prices to Uniswap V3/V4 raw prices. In pools with mismatched decimals (e.g., USDC/WETH), this error causes a valuation discrepancy of up to $10^{12}$ times. 
https://github.com/VII-Finance/core-contracts/blob/103ed3b52b123beb6f1664e25e3c72e92147b396/src/ERC721WrapperBase.sol#L146-L155

## Finding Description
The `UniswapV3Wrapper` and `UniswapV4Wrapper` contracts determine the underlying asset composition of an LP token by calculating the current square root price. The implementation directly divides the oracle's unit prices:

```solidity
uint256 token0UnitValue = oracle.getQuote(unit0, ...); // e.g., 1 USDC (1e6) = $1
uint256 token1UnitValue = oracle.getQuote(unit1, ...); // e.g., 1 WETH (1e18) = $3000

// Calculates ratio based on Economic Value: 1 / 3000
sqrtRatioX96 = SafeCast.toUint160(Math.sqrt(token0UnitValue * (1 << 96) / token1UnitValue) << 48);
```

However, Uniswap mathematics rely on the **Raw Reserves Ratio** ($\frac{\text{wei}_1}{\text{wei}_0}$). The logic fails to account for the decimal difference between the tokens.
For a USDC (6 decimals) / WETH (18 decimals) pool where 1 WETH = 3000 USDC:
*   **Correct Raw Price:** $\frac{1 \text{ unit WETH}}{3000 \text{ unit USDC}} = \frac{10^{18}}{3000 \times 10^6} \approx 3.33 \times 10^8$ (Assuming Token0=USDC, Token1=WETH).
*   **Wrapper Calculated Price:** $\frac{\text{Value of 1 USDC unit}}{\text{Value of 1 WETH unit}} = \frac{1}{3000} \approx 0.00033$.

The wrapper feeds this inflated/deflated price into `UniswapPositionValueHelper`. Because the calculated price is orders of magnitude off from the tick range, the helper function incorrectly assumes the LP is composed entirely of one asset and calculates a "phantom" balance based on the erroneous price.

## Impact Explanation
1.  Attacker Gain: An attacker can mint a minimal LP position in a mismatched-decimal pool (e.g., USDC/WETH). The wrapper will value this dust position at astronomical levels (e.g., $10^{12}$ times its actual worth). The attacker can then use this inflated collateral to borrow all available assets from the lending vaults.
2.  Honest User Loss: Conversely, legitimate users depositing high-value assets (like WETH in certain pairs) may have their collateral valued near zero, triggering unfair liquidations immediately upon deposit.

## Likelihood Explanation
The vulnerability exists in the core math logic and affects all pools where `decimals(token0) != decimals(token1)`. This includes the most liquid and important pools in DeFi (USDC/ETH, WBTC/USDC, DAI/USDC). No special market conditions are required to trigger it.

## Proof of Concept
The following test compares the Wrapper's calculated SqrtPrice against the real Uniswap V3 Mainnet Pool SqrtPrice for USDC/WETH. It confirms a deviation factor of $10^{12}$.
- Copy the code below into `test/`, then run `forge test --mt test_Direct_Price_Mismatch -vv`
```solidity
import {UniswapBaseTest} from "test/uniswap/UniswapBase.t.sol";
import {UniswapV3Wrapper} from "src/uniswap/UniswapV3Wrapper.sol";
import {ERC721WrapperBase} from "src/ERC721WrapperBase.sol";
import {Addresses} from "test/helpers/Addresses.sol";
import {INonfungiblePositionManager} from "lib/v3-periphery/contracts/interfaces/INonfungiblePositionManager.sol";
import {IUniswapV3Pool} from "lib/v3-core/contracts/interfaces/IUniswapV3Pool.sol";
import {IUniswapV3Factory} from "lib/v3-core/contracts/interfaces/IUniswapV3Factory.sol";
import {IERC20Metadata} from "lib/openzeppelin-contracts/contracts/interfaces/IERC20Metadata.sol";
import {IPriceOracle} from "lib/euler-price-oracle/src/interfaces/IPriceOracle.sol";
import {IEVC} from "ethereum-vault-connector/interfaces/IEthereumVaultConnector.sol";
import {console} from "forge-std/console.sol";

// Mock Oracle: 1 WETH = $3000, 1 USDC = $1
contract SimpleMockOracle {
    address public weth; 
    constructor(address _weth) { weth = _weth; }

    function getQuote(uint256 inAmount, address base, address) external view returns (uint256) {
        if (base == weth) return (inAmount * 3000 * 1e6) / 1e18;
        return (base == weth) ? inAmount * 3000 : inAmount; 
    }
}

contract PoC_Critical_Math_Error is UniswapBaseTest {
    IUniswapV3Pool pool;

    function setUp() public override {
        // Fork Mainnet to use real USDC (6 decimals) and WETH (18 decimals)
        vm.createSelectFork(vm.envOr("MAINNET_RPC_URL", string("https://eth.llamarpc.com")), 22473612);
        evc = IEVC(Addresses.EVC);
        
        address WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
        address USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
        
        (token0, token1) = (USDC, WETH); // USDC is token0
        unitOfAccount = USDC;
        oracle = IPriceOracle(address(new SimpleMockOracle(WETH)));

        // Setup Wrapper linked to the real Mainnet USDC/WETH 0.05% Pool
        INonfungiblePositionManager npm = INonfungiblePositionManager(Addresses.NON_FUNGIBLE_POSITION_MANAGER);
        IUniswapV3Factory factory = IUniswapV3Factory(npm.factory());
        address poolAddress = factory.getPool(token0, token1, 500); 
        pool = IUniswapV3Pool(poolAddress);

        wrapper = new UniswapV3Wrapper(
            address(evc), address(npm), address(oracle), unitOfAccount, address(pool)
        );
        unit0 = 10 ** 6;
        unit1 = 10 ** 18; 
    }

    function test_Direct_Price_Mismatch() public {
        uint160 wrapperSqrtPrice = wrapper.getSqrtRatioX96(token0, token1, unit0, unit1);
        (uint160 realPoolSqrtPrice,,,,,,) = pool.slot0();
        
        // Calculate deviation
        uint256 ratio;
        if (wrapperSqrtPrice > realPoolSqrtPrice) {
            ratio = uint256(wrapperSqrtPrice) / uint256(realPoolSqrtPrice);
        } else {
            ratio = uint256(realPoolSqrtPrice) / uint256(wrapperSqrtPrice);
        }

        // Verify
        uint256 priceDeviation = ratio * ratio;
        bool isCriticalBugPresent = priceDeviation > 1000000;
        
        assertTrue(isCriticalBugPresent, "Wrapper valuation logic is critically flawed.");
        console.log("Real Pool SqrtPrice:  %s", realPoolSqrtPrice);
        console.log("Wrapper SqrtPrice:    %s", wrapperSqrtPrice);
        console.log("Deviation Factor:     %s (approx 10^12)", priceDeviation);
    }
}
```

## Recommendation
Adjust the price ratio calculation to account for the difference in token decimals. Modify `getSqrtRatioX96()` in `ERC721WrapperBase.sol`:

```solidity
function getSqrtRatioX96(address token0, address token1, uint256 unit0, uint256 unit1) ... {
    uint256 token0UnitValue = oracle.getQuote(unit0, token0, unitOfAccount);
    uint256 token1UnitValue = oracle.getQuote(unit1, token1, unitOfAccount);

    // FIX: Normalize values to raw precision (per wei)
    // Formula: Ratio = (Value0 / unit0) / (Value1 / unit1)
    // Implemented as: (Value0 * unit1) / (Value1 * unit0)
    
    uint256 numerator = token0UnitValue * unit1;
    uint256 denominator = token1UnitValue * unit0;

    // Use FullMath or mulDiv to prevent overflow and ensure precision
    uint256 ratioX128 = Math.mulDiv(numerator, 1 << 128, denominator);
    
    // Convert to X96: sqrt(ratioX128) * 2^32
    sqrtRatioX96 = SafeCast.toUint160(Math.sqrt(ratioX128) << 32);
}
```

