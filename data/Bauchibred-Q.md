# QA Report for **Loopfi**

## Table of Contents

| Issue ID | Description |
| -------- | ----------- |
| [QA-01](#qa-01-configuration-changes-that-would-drastically-affect-users-and-should-be-behind-a-timelock) | Configuration changes that would drastically affect users and should be behind a timelock |
| [QA-02](#qa-02-do-not-have-incorrect-storage-gap-sizes) | Do not have incorrect storage gap sizes |
| [QA-03](#qa-03-remove-stale-documentation-from-when-adding-new-quoted-tokens) | Remove stale documentation from when adding new quoted tokens |
| [QA-04](#qa-04-consider-not-having-chainlinks-oracle-address-as-an-immutable-var) | Consider not having Chainlink's oracle address as an immutable var |
| [QA-05](#qa-05-update-documentation-for-when-transferring-out-rewards) | Update documentation for when transferring out rewards |
| [QA-06](#qa-06-setters-should-always-have-equality-checkers) | Setters should always have equality checkers |
| [QA-07](#qa-07-query-chainlink-via-proxy-instead-of-price-aggregator-directly) | Query Chainlink via proxy instead of price aggregator directly |

## QA-01 Configuration changes that would drastically affect users and should be behind a timelock

### Proof of Concept

Several configuration update actions within the protocol can negatively impact users' transactions if changes occur just prior to transaction execution.

For example, take a look at the new in-scope `Locking.sol` contract: https://github.com/code-423n4/2024-10-loopfi/blob/d219f0132005b00a68f505edc22b34f9a8b49766/src/Locking.sol#L40-L44

```solidity
    function setCooldownPeriod(uint256 _cooldownPeriod) external onlyOwner {
        cooldownPeriod = _cooldownPeriod;
        emit CooldownPeriodSet(_cooldownPeriod);
    }

```

This function updates the value for the cooldown period can be changed which would then have users committing to transactions under different terms than expected due to these configuration changes.

### Impact

Users may face unexpected terms in their transactions.

### Recommended Mitigation Steps

Allow users to specify expected configuration values as parameters. Transactions should revert if the actual configuration does not match the user-specified expectations or these changes should be applied via a timelock.

## QA-02 Do not have incorrect storage gap sizes

### Proof of Concept

Multiple instances of this, for example see https://github.com/code-423n4/2024-10-loopfi/blob/d219f0132005b00a68f505edc22b34f9a8b49766/src/oracle/PendleLPOracle.sol#L17-L41

```solidity
contract PendleLPOracle is IOracle, AccessControlUpgradeable, UUPSUpgradeable {
    using PendleLpOracleLib for IPMarket;
    /*//////////////////////////////////////////////////////////////
                               CONSTANTS
    //////////////////////////////////////////////////////////////*/

    /// @notice Chainlink aggregator address
    AggregatorV3Interface public immutable aggregator;
    /// @notice Stable period in seconds
    uint256 public immutable stalePeriod;
    /// @notice Aggregator decimal to WAD conversion scale
    uint256 public immutable aggregatorScale;
    /// @notice Pendle Market
    IPMarket public immutable market;
    /// @notice TWAP window in seconds
    uint32 public immutable twapWindow;
    /// @notice Pendle Pt Oracle
    IPPtOracle public immutable ptOracle;

    /*//////////////////////////////////////////////////////////////
                              STORAGE GAP
    //////////////////////////////////////////////////////////////*/

    uint256[50] private __gap;
    ..sniip
}

```

Or even [the ChainlinkOracle:](https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/oracle/ChainlinkOracle.sol#L28-L34)

```solidity
    /*//////////////////////////////////////////////////////////////
                              STORAGE GAP
    //////////////////////////////////////////////////////////////*/

    uint256[50] private __gap;
```

Evidently, more than one storage slot has been used in these contracts and as such the gap `var` should reflect this

### Impact

Incorrect storage gap size could lead to potential issues during future upgrades. If new variables are added in upgraded versions, they might overwrite the storage gap, potentially causing storage collisions and unexpected behavior.

### Recommended Mitigation Steps

Adjust the storage gap size to account for the storage slot already used in all instances were upgrades could occur.

This ensures that the total number of storage slots reserved for the contract (including both used slots and the gap) remains at 50, maintaining the intended/safe storage layout for future upgrades.

## QA-03 Remove stale documentation from when adding new quoted tokens

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-10-loopfi/blob/d219f0132005b00a68f505edc22b34f9a8b49766/src/quotas/PoolQuotaKeeperV3.sol#L161-L179

```solidity
    function addQuotaToken(
        address token,
        uint16 rate
    )
        external
        override
        gaugeOnly // U:[PQK-3]
    {
        if (quotaTokensSet.contains(token)) {
            revert TokenAlreadyAddedException(); // U:[PQK-6]
        }

        // The rate will be set during a general epoch update in the gauge
        quotaTokensSet.add(token); // U:[PQK-5]
        totalQuotaParams[token].cumulativeIndexLU = 1; // U:[PQK-5]
        totalQuotaParams[token].rate = rate;
        emit AddQuotaToken(token); // U:[PQK-5]
    }

```

When adding the qouted tokens we now atomically update the rates to ensure [this bug case](https://github.com/code-423n4/2024-07-loopfi-findings/issues?q=is%3Aissue+is%3Aopen+Zero+rates+on+new+quoted+tokens+allow+an+attacker+to+take+an+interest+free+quota) has been sufficiently covered, issue however is that there is still a stale comment that the rate are being updated in the next epoch, whereas they are being updated now

### Recommended Mitigation Steps

Apply this fix:

```diff
    function addQuotaToken(
        address token,
        uint16 rate
    )
        external
        override
        gaugeOnly // U:[PQK-3]
    {
        if (quotaTokensSet.contains(token)) {
            revert TokenAlreadyAddedException(); // U:[PQK-6]
        }

-        // The rate will be set during a general epoch update in the gauge
        quotaTokensSet.add(token); // U:[PQK-5]
        totalQuotaParams[token].cumulativeIndexLU = 1; // U:[PQK-5]
        totalQuotaParams[token].rate = rate;
        emit AddQuotaToken(token); // U:[PQK-5]
    }

```

## QA-04 Consider not having Chainlink's oracle address as an immutable var

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-10-loopfi/blob/d219f0132005b00a68f505edc22b34f9a8b49766/src/oracle/BalancerOracle.sol#L23-L24

```solidity
    IOracle public immutable chainlinkOracle;

```

### Impact

Using an immutable reference for the Chainlink oracle address reduces flexibility and could lead to issues if the Chainlink oracle address needs to be updated or if it becomes compromised.

### Recommended Mitigation Steps

Replace the immutable reference with a mutable state variable and implement a function to update the Chainlink oracle address:

```diff
- IOracle public immutable chainlinkOracle;
+ IOracle public chainlinkOracle;
..snip
+function updateChainlinkOracle(address newOracle) external onlyRole(MANAGER_ROLE) {
+    require(newOracle != address(0), "Invalid oracle address");
+    chainlinkOracle = IOracle(newOracle);
+}
```

## QA-05 Update documentation for when transferring out rewards

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-10-loopfi/blob/d219f0132005b00a68f505edc22b34f9a8b49766/src/pendle-rewards/RewardManager.sol#L89-L113

```solidity

    /// @dev this function doesn't need redeemExternal since redeemExternal is bundled in updateRewardIndex
    /// @dev this function also has to update rewardState.lastBalance
    function _doTransferOutRewards(
        address user
    ) internal virtual override returns (address[] memory tokens, uint256[] memory rewardAmounts, address to) {
        tokens = market.getRewardTokens();
        rewardAmounts = new uint256[](tokens.length);
        for (uint256 i = 0; i < tokens.length; i++) {
            rewardAmounts[i] = userReward[tokens[i]][user].accrued;
            if (rewardAmounts[i] != 0) {
                userReward[tokens[i]][user].accrued = 0;
                rewardState[tokens[i]].lastBalance -= rewardAmounts[i].Uint128();
                //_transferOut(tokens[i], receiver, rewardAmounts[i]);
            }
        }

        if (proxyRegistry.isProxy(user)) {
            to = IPRBProxy(user).owner();
        } else {
            to = user;
        }
        return (tokens, rewardAmounts, to);
    }

```

The comments hint an implementation of a `redeemExternal()` functionality as is present in the [forked Pendle code](https://github.com/pendle-finance/pendle-core-v2-public/blob/7e451be619353f57a8ab722234b8f9ebe0632836/contracts/core/RewardManager/RewardManager.sol#L33), issue however is that on LoopFi, there is no implementation of `redeemExternal`, rather what exists is directly querying the market via `redeem()`, see the @audit tag from below https://github.com/code-423n4/2024-10-loopfi/blob/d219f0132005b00a68f505edc22b34f9a8b49766/src/pendle-rewards/RewardManager.sol#L44-L88

```solidity
    function _updateRewardIndex()
        internal
        virtual
        override
        returns (address[] memory tokens, uint256[] memory indexes)
    {
        tokens = market.getRewardTokens();
        indexes = new uint256[](tokens.length);

        if (tokens.length == 0) return (tokens, indexes);

        if (lastRewardBlock != block.number) {
            // if we have not yet update the index for this block
            lastRewardBlock = block.number;

            uint256 totalShares = _rewardSharesTotal();
            // Claim external rewards on Market
            market.redeemRewards(address(vault));//@audit

            for (uint256 i = 0; i < tokens.length; ++i) {
                address token = tokens[i];

                // the entire token balance of the contract must be the rewards of the contract

                RewardState memory _state = rewardState[token];
                (uint256 lastBalance, uint256 index) = (_state.lastBalance, _state.index);

                uint256 accrued = IERC20(tokens[i]).balanceOf(vault) - lastBalance;

                if (index == 0) index = INITIAL_REWARD_INDEX;
                if (totalShares != 0) index += accrued.divDown(totalShares);

                rewardState[token] = RewardState({
                    index: index.Uint128(),
                    lastBalance: (lastBalance + accrued).Uint128()
                });

                indexes[i] = index;
            }
        } else {
            for (uint256 i = 0; i < tokens.length; i++) {
                indexes[i] = rewardState[tokens[i]].index;
            }
        }
    }
```

### Recommended Mitigation Steps

Fix the stale comments.

## QA-06 Setters should always have equality checkers

### Proof of Concept

Multiple instances across scope, for example, take a look at the new in-scope `Locking.sol` contract: https://github.com/code-423n4/2024-10-loopfi/blob/d219f0132005b00a68f505edc22b34f9a8b49766/src/Locking.sol#L40-L44

```solidity
    function setCooldownPeriod(uint256 _cooldownPeriod) external onlyOwner {
        cooldownPeriod = _cooldownPeriod;
        emit CooldownPeriodSet(_cooldownPeriod);
    }

```

This function updates the value for the cooldown period can be changed which would then have users committing to transactions under different terms than expected due to these configuration changes.

Evidently, the function above and similar setters are used to make updates to already set values but there are no checks to see if the states are not the already passed in values.

### Impact

QA

### Recommended Mitigation Steps

Consider checking if the value being set is what's already stored and just skip this attempt instead.

## QA-07 Query Chainlink via proxy instead of price aggregator directly

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-10-loopfi/blob/d219f0132005b00a68f505edc22b34f9a8b49766/src/oracle/ChainlinkOracle.sol#L96-L111

```solidity
    function _fetchAndValidate(address token) internal view returns (bool isValid, uint256 price) {
        Oracle memory oracle = oracles[token];
        try AggregatorV3Interface(oracle.aggregator).latestRoundData() returns (
            uint80 /*roundId*/,
            int256 answer,
            uint256 /*startedAt*/,
            uint256 updatedAt,
            uint80 /*answeredInRound*/
        ) {
            isValid = (answer > 0 && block.timestamp - updatedAt <= oracle.stalePeriod);
            return (isValid, wdiv(uint256(answer), oracle.aggregatorScale));
        } catch {
            // return the default values (false, 0) on failure
        }
    }

```

This function fetches and validates the latest price from Chainlink, problem however is that Chainlink recommends using the proxy and not the priceAggregator directly as a best practice.

### Impact

QA - best practice

### Recommended Mitigation Steps

Follow the mentioned [best practices from Chainlink](https://docs.chain.link/data-feeds):
You can call the `latestRoundData()` function directly on the aggregator, but it is a best practice to use the proxy instead so that changes to the aggregator do not affect your application. Similar to the proxy contract, the aggregator contract has a latestAnswer variable, owner address, latestTimestamp variable, and several others.
