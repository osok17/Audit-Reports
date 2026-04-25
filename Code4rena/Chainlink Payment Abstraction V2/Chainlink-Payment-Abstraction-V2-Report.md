# [Chainlink Payment Abstraction V2 Report](https://code4rena.com/audits/2026-03-chainlink-payment-abstraction-v2)

| ID | Title |
|:--:|:---|
| [M-1](#m-1-strict-balance-check-in-isvalidsignature-prevents-cowswap-partial-fills) | Strict balance check in isValidSignature prevents CowSwap partial fills |
| [L-1](#l-1-missing-active-auction-validation-in-feed-id-reassignment-breaks-ongoing-auctions) | Missing Active Auction Validation in Feed ID Reassignment Breaks Ongoing Auctions |

# [M-1] Strict balance check in `isValidSignature` prevents CowSwap partial fills
https://github.com/code-423n4/2026-03-chainlink/blob/5317782e3855d9547412ab5a490257d7c5e51fac/src/GPV2CompatibleAuction.sol#L144-L147
## Finding description
The `GPV2CompatibleAuction` contract integrates CowSwap's EIP-1271 signature verification logic. The code enforces that orders sent to CowSwap must support partial fills (`if (!order.partiallyFillable) revert;`), which indicates that the protocol intends to allow multiple solvers to concurrently and incrementally fill large auction orders.

However, within the `isValidSignature()` function, there exists a global balance check that requires the current balance to be greater than or equal to `order.sellAmount`:
```solidity
uint256 assetInBalance = order.sellToken.balanceOf(address(this));
if (order.sellAmount > assetInBalance) {
  revert InsufficientAssetInBalance(address(order.sellToken), order.sellAmount, assetInBalance);
}
```
In CowSwap, `order.sellAmount` represents the "maximum total capacity" of the order, rather than the actual execution amount requested by a solver in a single settlement. When the CowSwap settlement contract (`GPv2Settlement`) calls `isValidSignature()`, it always passes the original unmodified order. This creates a severe logical inconsistency:

1. The protocol initiates an auction of 100,000 USDC via an off-chain workflow and posts an order with sellAmount = 100,000 to the CowSwap API (marked as partially fillable).
2. A retail user finds the price attractive and purchases a very small portion (e.g., 1 USDC) via the native `bid()` function. The contract's actual balance becomes 99,999 USDC.
3. A CowSwap solver observes the orderbook and identifies a profitable opportunity. The solver attempts a partial fill, for example filling 3,000 USDC.
4. During settlement, CowSwap internally passes the original order (with sellAmount = 100,000) to `isValidSignature()` for validation.
5. The code executes: `if (100,000 > 99,999) revert;` Even though the solver only intends to partially fill the order, and the contract still holds sufficient balance (99,999), this valid partial fill transaction is reverted.

The project team mentioned in their known issues:
- when a non CowSwap solver bids on the auction, the limit order quantity becomes stale on the CowSwap orderbook until the next workflow run. This may lead to failed orders attempted by solvers trying to fill 100% of the order during that time window. 

However, in reality, during the "gap window" between a balance change and the workflow re-signing and publishing a new order, not only do 100% fills fail, but even valid partial fills from solvers will also fail. Even after the workflow updates the order, this issue can repeatedly occur under normal operation.

### Impact
- The protocol explicitly enforces `partiallyFillable = true`, making concurrent partial fills a core intended mechanism. The global balance check completely paralyzes this core mechanism. Any minor bid transitions the order into a permanently frozen state for partial fillers until the off-chain system intervenes, reducing a decentralized
- Frequent reverts will cause Solvers to waste Gas fees. Consequently, the protocol's orders are highly likely to be flagged as "Toxic Orderflow" by Solver risk engines, leading to a complete refusal of service.

## Recommended mitigation steps
When a Solver attempts to execute a settlement, the underlying `GPv2Settlement` contract will naturally call `ERC20.transferFrom` during the asset transfer phase. If the amount the Solver attempts to transfer exceeds the contract's actual balance at that time, the underlying token contract will naturally execute a safe mathematical revert.
Consider removing this redundant and restrictive balance validation logic from `isValidSignature()`.
```diff
function isValidSignature(...) ... {
    // ...
    if (order.sellAmount == 0) {
      revert Errors.InvalidZeroAmount();
    }
-   uint256 assetInBalance = order.sellToken.balanceOf(address(this));
-   if (order.sellAmount > assetInBalance) {
-     revert InsufficientAssetInBalance(address(order.sellToken), order.sellAmount, assetInBalance);
-   }
    // ... 
```

## POC
The PoC demonstrates how a legitimate partial fill fails after a minor balance reduction occurs.
- Copy the test below into `/test/integration/gpv2-compatible-auction/is-valid-signature/isValidSignature.t.sol`, then run `forge test --mt test_isValidSignature_RevertWhen_PartiallyFilled`
```solidity
function test_isValidSignature_RevertWhen_PartiallyFilled() external {
// Verify that the original 100,000 USDC order is fully valid before any state changes.
bytes4 magicValue = s_auction.isValidSignature(s_orderId, abi.encode(s_order));
assertEq(magicValue, IERC1271.isValidSignature.selector);

// Simulate a small bid or a prior partial fill that reduces the contract’s actual asset balance by 100 USDC (1e6)
uint256 currentBalance = s_mockUSDC.balanceOf(address(s_auction));
uint256 newBalance = currentBalance - 100e6;

deal(address(s_mockUSDC), address(s_auction), newBalance);

// Ensure the balance has indeed decreased
assertEq(s_mockUSDC.balanceOf(address(s_auction)), newBalance);

// CowSwap solver attempts to partially fill the order
vm.expectRevert(
    abi.encodeWithSelector(
    GPV2CompatibleAuction.InsufficientAssetInBalance.selector,
    address(s_mockUSDC),
    s_order.sellAmount, // Original order capacity: 100,000 USDC
    newBalance          // Current physical balance: 99,900 USDC
    )
);
s_auction.isValidSignature(s_orderId, abi.encode(s_order));
}
```

# [L-1] Missing Active Auction Validation in Feed ID Reassignment Breaks Ongoing Auctions
https://github.com/code-423n4/2026-03-chainlink/blob/5317782e3855d9547412ab5a490257d7c5e51fac/src/PriceManager.sol#L265-L277
## Finding description and impact
In the `_applyFeedInfoUpdates` function, if an admin assigns a `dataStreamsFeedId` that is already in use by another asset, the system executes a Feed ID rotation logic.
The code identifies the old asset that originally owned this Feed ID (`previousAssetForFeedId`) and silently clears its Data Streams configuration:
```solidity
address previousAssetForFeedId = s_dataStreamsFeedIdToAsset[feedInfo.dataStreamsFeedId];
if (previousAssetForFeedId != address(0) && previousAssetForFeedId != asset) {
  FeedInfo storage previousAssetFeedInfo = s_feedInfo[previousAssetForFeedId];
  // ...
  previousAssetFeedInfo.dataStreamsFeedId = bytes32(0);
  previousAssetFeedInfo.dataStreamsFeedDecimals = 0;
  delete s_dataStreamsPrice[previousAssetForFeedId];
}
```
Within this execution flow, the defense hook `_onFeedInfoUpdate(asset, false)` only validates the active auction status of the new incoming asset. The code completely fails to check whether the stripped old asset (`previousAssetForFeedId`) is currently undergoing an active auction.

Silently removing the `dataStreamsFeedId` from an active `previousAssetForFeedId` forces the ongoing auction to downgrade and rely on the fallback push oracle. This leads to one of two severe outcomes:
1. If `stalenessThreshold` is strict (e.g., optimized for Data Streams): The fallback push oracle's heartbeat interval (e.g., 1 to 24 hours) will heavily exceed this tight threshold. This triggers continuous `StaleFeedData` reverts, completely bricking the auction until it reaches its natural timeout.

2. If `stalenessThreshold` is loose (e.g., adapted for the push oracle): The auction survives, but its pricing lags significantly behind Data Streams. This causes abrupt jumps in the auction's floor price calculation, breaking the smooth decay curve of the Dutch auction and directly exposing the auctioned assets to MEV arbitrage.

If the admin attempts to fix the issue by restoring the `dataStreamsFeedId` or modifying the `stalenessThreshold` for the affected asset, the transaction will revert inside the `_onFeedInfoUpdate()` hook because the asset is still flagged as being in an ongoing auction.

## Recommended mitigation steps
Trigger the state validation hook for the old asset (`previousAssetForFeedId`) before clearing its data to ensure it is not locked in an active auction.
```diff
address previousAssetForFeedId = s_dataStreamsFeedIdToAsset[feedInfo.dataStreamsFeedId];
if (previousAssetForFeedId != address(0) && previousAssetForFeedId != asset) {
+  // Ensure the asset being stripped of its oracle config is not in a live auction
+  _onFeedInfoUpdate(previousAssetForFeedId, true);

   FeedInfo storage previousAssetFeedInfo = s_feedInfo[previousAssetForFeedId];
   //...
}
```

## POC
- Copy the test below into `test/poc/C4PoC.t.sol`, then run `forge test --mt test_PoC_FeedIdReassignmentSilentlyBreaksActiveAuction -vv`
```solidity
function test_PoC_FeedIdReassignmentSilentlyBreaksActiveAuction() public {
  // 1. Start an auction for WETH
  _startAuction(address(mockWETH), 10e18);

  // 2. Advance time so that the fallback Data Feed (push oracle) becomes stale
  // stalenessThreshold is 1 hour, so we move forward by 1 hour + 1 second
  skip(1 hours + 1 seconds);

  // 3. Simulate fresh price updates from Data Streams (high-frequency oracle)
  // Note: the underlying Data Feed timestamp remains stale (1 hour old)
  _transmitPrices(4_000e18, 1e18, 20e18); 

  // Check: auction is still functional since Data Streams is prioritized
  uint256 linkNeededBefore = _getAssetOutAmount(address(mockWETH), 1e18);
  assertTrue(linkNeededBefore > 0);

  PriceManager.FeedInfo memory wethInfoBefore = auction.getFeedInfo(address(mockWETH));
  console2.log("WETH DataStreams ID (Before):");
  console2.logBytes32(wethInfoBefore.dataStreamsFeedId);
  console2.log("WETH DataStreams Decimals (Before):", wethInfoBefore.dataStreamsFeedDecimals);

  // 4. Admin reassigns WETH's DataStreams feed ID to WBTC
  PriceManager.ApplyFeedInfoUpdateParams[] memory updates = new PriceManager.ApplyFeedInfoUpdateParams[](1);
  updates[0] = PriceManager.ApplyFeedInfoUpdateParams({
      asset: address(mockWBTC), 
      feedInfo: PriceManager.FeedInfo({
          dataStreamsFeedId: i_mockWETHFeedId,
          usdDataFeed: AggregatorV3Interface(address(mockWethUsdFeed)), 
          dataStreamsFeedDecimals: 18,
          stalenessThreshold: 1 hours
      })
  });

  _changePrank(assetAdmin);
  auction.applyFeedInfoUpdates(updates, new address[](0));

  PriceManager.FeedInfo memory wethInfoAfter = auction.getFeedInfo(address(mockWETH));

  // Verify: WETH's DataStreams config is silently cleared
  assertEq(wethInfoAfter.dataStreamsFeedId, bytes32(0), "WETH DataStreams ID was not silently cleared");
  assertEq(wethInfoAfter.dataStreamsFeedDecimals, 0, "WETH DataStreams decimals were not silently cleared");
  console2.log("WETH DataStreams ID (After):");
  console2.logBytes32(wethInfoAfter.dataStreamsFeedId);
  console2.log("WETH DataStreams Decimals (After):", wethInfoAfter.dataStreamsFeedDecimals);

  // 5. With Data Streams removed, the system falls back to the stale Data Feed
  // Any bid attempt now reverts due to stale oracle data
  _changePrank(bidder);
  vm.expectRevert(Errors.StaleFeedData.selector);
  auctionBidder.bid(address(mockWETH), 1e18, new Caller.Call[](0));

  // 6. Admin cannot fix the issue because WETH is still in a live auction
  updates[0].asset = address(mockWETH);
  _changePrank(assetAdmin);
  vm.expectRevert(BaseAuction.LiveAuction.selector);
  auction.applyFeedInfoUpdates(updates, new address[](0));
}
```
