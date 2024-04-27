
![](https://i.imgur.com/XUFwLJW.jpeg)

# Need more salt! — Credit Guild $2K Bounty

_You can also read this writeup on [Medium](https://wardaudits.medium.com/need-more-salt-credit-guild-2k-bounty-1404f07f1394)._

A week ago we've found a medium severity vulnerability in [Ethereum Credit Guild](https://twitter.com/CreditGuild) (fka Volt Protocol). It was disclosed confidentially prior to project's launch via private bug bounty since the project has not yet launched and it will take some time before they open a BBP on Immunefi. We noticed the team's security-first approach when they offered a bonus reward of GUILD tokens for the bug we found during the Code4rena contest, which is why we did not hesitate to make a private disclosure without an intermediary platform. The team managed the process very quickly and professionally.

Credit Guild actually [did a contest on Code4rena](https://code4rena.com/audits/2023-12-ethereum-credit-guild#top) in December 2023, but the reason this bug wasn’t reported during the contest is likely because `LendingTermFactory.sol` was not within the scope. We wanna make this a call for the Wardens reading this: If you feel good about the team, consider looking into contracts that are not in the scope (excluding unfinished contracts), note down any sussy parts and you can examine them more deeply after the contest. Now let's take a deep dive into this finding, demonstrating how deterministic contract creations can create an address collision despite how wide the parameters may be.

## Intro
The Credit Guild is a protocol for trust minimized pooled lending on the Ethereum network. Its goal is to scale governance to support a larger set of assets and loan terms, while minimizing third-party risk. They use what they call "terms" for lending and borrowing, and their `LendingTerm.sol` implementation contains many great nuances. Unlike well-known lending protocols such as Aave, Compound or MakerDAO, it doesn't rely on trusted oracles. LendingTerms can be created permissionlessly via a factory contract just like Uniswap pools and have their own distinct features based on the parameters given for creation. However, in order for the terms we create to be activated, these terms need to be onboarded by governance. The onboarding contract used for this purpose also acts as a factory contract.

## Bug Report

_For readers: Please note that this bug is not significant for Credit Guild's Arbitrum deployment, where transaction ordering is FCFS (first-come, first-serve), and it is specific to the Ethereum Mainnet deployment._

**Title:** Denial of service in `LendingTermFactory::createTerm()` due to address collision

**Context:** [LendingTermFactory.sol#L115-L118](https://github.com/volt-protocol/ethereum-credit-guild/blob/main/src/governance/LendingTermFactory.sol#L115-L118)

**Severity:** Medium

**Description:**
In `LendingTermFactory::createTerm()`, we create terms deterministically using the Clones library. The problem is that since the `gaugeType` parameter is not used in salt creation, terms with different `gaugeType` will have the same address, as the salt will be the same if the rest are the same. This may have been omitted due to the assumption that if the `gaugeType` is different, the params will be given differently but this allows an attacker to DoS the term creation process, griefing the protocol and the users. Attacker can create terms with `gaugeType` different from the desired `gaugeType` by monitoring the mempool (on mainnet) to make it impossible to create the terms that want to be created (and even wanted to be proposed for onboarding in the future) or by looking around the discussions of the terms that are wanted to be created in the future, or they can simply guess and create with the reasonable `lendingTermParams` but unintended `gaugeType`.

**Proof of Concept:**
The following test demonstrates that creating a term with the same parameters and different `gaugeType` using createTerm will result in a revert;

First, add this part to `LendingTermOnboardingUnitTest::setUp()` to set reference for the new `gaugeType`:
```solidity
        factory.setMarketReferences(
            2,
            LendingTermFactory.MarketReferences({
                profitManager: address(profitManager),
                creditMinter: address(rlcm),
                creditToken: address(credit)
            })
        );
```

In the test below, we create a term with `gaugeType = 1` before it's created with (the intended) `gaugeType = 2`; Making it impossible to create a term given `gaugeType = 2` with the same implementation, auction house and parameters. Attacker can do this repeatedly for each term that is wanted to be created and especially those wanted to be onboarded so changing params will not work.

The test is expected to be successful because `expectRevert()` is used. It reverts with the message `"ERC1167: create2 failed".`
```solidity
    function test_addressCollisionInCreateTerm() public {

	/*
	The term that is wanted to be created (and perhaps proposed for onboarding
	in the future) is actually wanted to be given gaugeType of 2.
	Before this, Attacker creates the term with gaugeType 1.
	*/
		
        LendingTerm term = LendingTerm(
            factory.createTerm(
                1, // gaugeType
                address(termImplementation),
                address(auctionHouse),
                abi.encode(
                    LendingTerm.LendingTermParams({
                        collateralToken: address(collateral),
                        maxDebtPerCollateralToken: _CREDIT_PER_COLLATERAL_TOKEN,
                        interestRate: _INTEREST_RATE,
                        maxDelayBetweenPartialRepay: 0,
                        minPartialRepayPercent: 0,
                        openingFee: 0,
                        hardCap: _HARDCAP
                    })
                )
            )
        );
        vm.expectRevert();
        LendingTerm term2 = LendingTerm(
            factory.createTerm(
                2, // gaugeType
                address(termImplementation),
                address(auctionHouse),
                abi.encode(
                    LendingTerm.LendingTermParams({
                        collateralToken: address(collateral),
                        maxDebtPerCollateralToken: _CREDIT_PER_COLLATERAL_TOKEN,
                        interestRate: _INTEREST_RATE,
                        maxDelayBetweenPartialRepay: 0,
                        minPartialRepayPercent: 0,
                        openingFee: 0,
                        hardCap: _HARDCAP
                    })
                )
            )
        );
    }
```


**Mitigation:**

Simply adding the `gaugeType` variable to `salt` in `LendingTermFactory::createTerm()` will solve the problem.

```diff
	bytes32 salt = keccak256(
-           abi.encodePacked(implementation, auctionHouse, lendingTermParams)
+           abi.encodePacked(implementation, auctionHouse, lendingTermParams, gaugeType)
        );
```

## Thanks

We thank the Credit Guild team for their fair rewarding of $2K, quick responses, and for allowing us to make the report public. It was truly a 5/5 bug bounty process. We hope you have enjoyed the writeup!
