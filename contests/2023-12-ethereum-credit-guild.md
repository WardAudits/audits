# Ethereum Credit Guild

The code under review can be found in [`ethereumcreditguild`](https://github.com/code-423n4/2023-12-ethereumcreditguild).

## Findings Summary

| ID | Description | Severity |
| :-: | - | :-: |
| [M-01](#M-01) | Absence of Sequencer Uptime check risks Dutch Auctions executing at poor prices | Medium |

## [M-01] Absence of Sequencer Uptime check risks Dutch Auctions executing at poor prices 

**Context:**
- [AuctionHouse.sol#L118-L161](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/AuctionHouse.sol#L118-L161)

**Description:** 

Since the anticipated deployment of Ethereum Credit Guild on the Ethereum mainnet and L2s like Arbitrum, we must consider the existence of the Sequencer and the risks it may pose. While using Dutch Auctions on L2s, we notice that the sequencer uptime isn't taken into account, but similar to what occurred on the [Arbitrum network on December 15th](https://dedaub.com/blog/arbitrum-sequencer-outage), the sequencer outage is a possibility for the future.

In a case where there are ongoing auctions while the sequencer is offline, the auction prices will continue to decrease during the sequencer is offline. When the sequencer comes back online, users will have the opportunity to bid these auctions at prices significantly lower than the market rate. This may actually even be the favorable outcome, as missing out entirely on the auction is also a possibility. Referring to the example mentioned in [AuctionHouse.sol#L33-L37](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/AuctionHouse.sol#L33-L37), considering a 30-minute auction with a midPoint of 10 minutes and 50 seconds, if the sequencer remains inactive for just 11 minutes (passing the `midPoint`) it could lead to bad prices and if it stays offline for the entire 30 minutes, it would lead to forced debt forgiveness through `forgive()`. The size of bad debts can be anything and can have a dramatic impact considering the importance of their liquidation to the health of the protocol.

You can also take a look at [this report](https://github.com/sherlock-audit/2023-06-Index-judging/issues/40) where a similar case occurred.

**Recommendation:** 

Determine the maximum tolerable delay for the sequencer (11 minutes may be a good choice) and invalidate the auction if the sequencer was down for maximum tolerable delay or more during the auction period.