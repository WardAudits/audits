# Revolution Protocol

The code under review can be found in [`revolutionprotocol`](https://github.com/code-423n4/2023-12-revolutionprotocol).

## Findings Summary

| ID | Description | Severity |
| :-: | - | :-: |
| [H-01](#H-01) | Delegation to address(0) causes permanent loss of voting power | High |
| [M-01](#M-01) | Unrestricted continuous bidding can cause Denial of service | Medium |
| [L-01](#L-01) | Possible to set the `timeBuffer` too big that it freezes the bidder's funds | Low |

## [H-01] Delegation to address(0) causes permanent loss of voting power

**Context:**
- [VotesUpgradeable.sol#L206-L213](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/main/packages/revolution/src/base/VotesUpgradeable.sol#L206-L213)
- [NontransferableERC20Votes.sol#L29](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/main/packages/revolution/src/NontransferableERC20Votes.sol#L29)
- [ERC20VotesUpgradeable.sol#L24](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/main/packages/revolution/src/base/erc20/ERC20VotesUpgradeable.sol#L24)

**Description:** 

As stated in the comment on line 12 of NontransferableERC20Votes.sol, delegation of vote power can be done through the `delegate` function or by providing a signature to be used with `delegateBySig`. However, these functions do not prevent users from delegating to address(0), leading users to permanently lose their voting power, intentionally or accidentally.

A test demonstrating that delegation to address(0) is possible and indeed results in the loss of voting power. You can paste this into `NontransferableERC20.t.sol`. Also see: Issue in Nouns Builder repo for the same bug https://github.com/code-423n4/2022-09-nouns-builder-findings/issues/203

```solidity
function testVotingAndDelegationToAddress0() public {
        address delegate = address(0);
        uint256 mintAmount = 1000 * 1e18;

        // Mint tokens to the owner
        vm.startPrank(address(erc20TokenEmitter));
        erc20Token.mint(address(this), mintAmount);
        vm.stopPrank();

        // Delegate voting power
        erc20Token.delegate(delegate);

        // @bug Voting power is lost.
        assertEq(erc20Token.getVotes(delegate), 0);

        // Ensure that no tokens were transferred in the process of delegation
        assertEq(erc20Token.balanceOf(delegate), 0, "Delegation should not transfer tokens");
}
```

**Recommendation:** 

Don't allow delegation to address(0) by adding the relevant check to the internal `delegate()` function as the following:

```diff
    function _delegate(address account, address delegatee) internal virtual {
+       require(delegatee != address(0), "Votes: delegate to the zero address");
        VotesStorage storage $ = _getVotesStorage();
        address oldDelegate = delegates(account);
        $._delegatee[account] = delegatee;

        emit DelegateChanged(account, oldDelegate, delegatee);
        _moveDelegateVotes(oldDelegate, delegatee, _getVotingUnits(account));
    }
```

## [M-01] Unrestricted continuous bidding can cause Denial of service

This issue has been invalidated because it was claimed to be a "design choice" by the sponsor, but we think it's important to point it out.

**Context:**

- [AuctionHouse.sol#L191-L192](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L191-L192)

**Description:**

In `AuctionHouse::createBid` function, if bids are received within `timeBuffer` of the auction end time, the auctions end time gets extended by `block.timestamp + timeBuffer`. However, due to the lack of a limit on the repetition of adding `timeBuffer`, a contract holding sufficient ether could continuously create bids increasing by `minBidIncrementPercentage` within the `timeBuffer`, potentially leading to a denial of service. If this attack is repeated enough, it can lead to the auction not settling for much longer than expected and prevent the opening of a new auction. This could pose a big issue especially in the early stages of the DAO when bids are not that high, as the amount of ether required for the DoS would also be low.

```solidity
function testBid_Dos() public {
    createDefaultArtPiece();

    auction.unpause();
    vm.deal(address(1), 100 ether);

    vm.startPrank(address(1));
    (uint256 verbId, uint256 amount, uint256 startTime, uint256 endTime, address payable bidder, bool settled) =
        auction.auction();
    emit log_named_uint("endTime", endTime);

    auction.createBid{value: 1 ether}(0, address(1)); // Assuming the first auction's verbId is 0
    (verbId, amount, startTime, endTime, bidder, settled) = auction.auction();
    emit log_named_uint("startTime", startTime);
    emit log_named_decimal_uint("nextMinBid", getMinBidAmount(), 18);
    assertEq(amount, 1 ether, "Bid amount should be set correctly");
    assertEq(bidder, address(1), "Bidder address should be set correctly");

    skip(endTime - 10 minutes);

    for (uint256 i; i < 50; i++) {
        auction.createBid{value: getMinBidAmount()}(0, address(1)); // Assuming the first auction's verbId is 0
        skip(10 minutes);
        emit log_named_decimal_uint("nextMinBid", getMinBidAmount(), 18);
        (verbId, amount, startTime, endTime, bidder, settled) = auction.auction();
        emit log_named_uint("endTime", endTime);
    }

    vm.stopPrank();
    vm.startPrank(0xcDBD6B7A61B66eC7aF24B57b942225EF0A440AF7); // DAO address
    auction.pause();
    auction.settleAuction();
    vm.stopPrank();
}

function getMinBidAmount() public view returns (uint256) {
    (, uint256 amount,,,,) = auction.auction();
    return amount + (amount * auction.minBidIncrementPercentage()) / 100;
}
```

**Recommendation:**

Add the following code to prevent multiple extensions;

```solidity
bool extended = _auction.endTime - block.timestamp < timeBuffer;
if (extended && !_auction.isExtended) {
auction.endTime = _auction.endTime = block.timestamp + timeBuffer;
_auction.isExtended = true;
}
```

## [L-01] Possible to set the `timeBuffer` too big that it freezes the bidder's funds

**Context:**

- [AuctionHouse.sol#L277-L281](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/main/packages/revolution/src/AuctionHouse.sol#L277-L281)

**Description:**

It's possible to set timeBuffer to a big value with [setTimeBuffer()](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/main/packages/revolution/src/AuctionHouse.sol#L277-L281) function, since there is no control.

```solidity
File: AuctionHouse.sol
function setTimeBuffer(uint256 _timeBuffer) external override onlyOwner {
    timeBuffer = _timeBuffer;

    emit AuctionTimeBufferUpdated(_timeBuffer);
}
```

**Recommendation:**

Consider an upper limit for timeBuffer.
