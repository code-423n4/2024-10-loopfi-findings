---
sponsor: "LoopFi"
slug: "2024-10-loopfi"
date: "2025-02-17"
title: "LoopFi"
findings: "https://github.com/code-423n4/2024-10-loopfi-findings/issues"
contest: 456
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the LoopFi smart contract system written in Solidity. The audit took place between October 11 — October 18, 2024.

## Wardens

7 Wardens contributed reports to LoopFi:

  1. [pkqs90](https://code4rena.com/@pkqs90)
  2. [0xAlix2](https://code4rena.com/@0xAlix2) ([a\_kalout](https://code4rena.com/@a_kalout) and [ali\_shehab](https://code4rena.com/@ali_shehab))
  3. [chaduke](https://code4rena.com/@chaduke)
  4. [Evo](https://code4rena.com/@Evo)
  5. [ZanyBonzy](https://code4rena.com/@ZanyBonzy)
  6. [Bauchibred](https://code4rena.com/@Bauchibred)

This audit was judged by [Koolex](https://code4rena.com/@koolex).

Final report assembled by [thebrittfactor](https://twitter.com/brittfactorC4).

# Summary

The C4 analysis yielded an aggregated total of 7 unique vulnerabilities. Of these vulnerabilities, 2 received a risk rating in the category of HIGH severity and 5 received a risk rating in the category of MEDIUM severity.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the [C4 LoopFi repository](https://github.com/code-423n4/2024-10-loopfi), and is composed of 31 smart contracts written in the Solidity programming language and includes 3943 lines of Solidity code.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# High Risk Findings (2)
## [[H-01] Rewards might be lost due to the error that `_updateRewardIndex()` might advance `lastBalance` without advancing index for a token](https://github.com/code-423n4/2024-10-loopfi-findings/issues/25)
*Submitted by [chaduke](https://github.com/code-423n4/2024-10-loopfi-findings/issues/25), also found by [Evo](https://github.com/code-423n4/2024-10-loopfi-findings/issues/28)*

The function `_updateRewardIndex()` is used to update the `lastBalance` and `index` of each reward token. This function will be called when a user deposits, withdraws collateral or claims rewards.

However, the function might not advance `index` when `accrued.divDown(totalShares) = 0`. This might happen when `totalShares` is too big and `accrued` is too small. One case is that the number of decimals for the reward token is too small.

<https://github.com/code-423n4/2024-10-loopfi/blob/d219f0132005b00a68f505edc22b34f9a8b49766/src/pendle-rewards/RewardManager.sol#L74>

For example, the USDC token only has 6 decimals.

Suppose `accrued` = `$100 = 100*10**6`, and `totalShares` `= 200M = 200 * 10** 6 * 10**18`; then we have  `accrued.divDown(totalShares) = 0`.

Furthermore, if function `_updateRewardIndex()` is called more frequently, either because a malicious user keeps calling `getRewards()` (the gas fee is low on Arbitrum) or simply because the community is large so there is a high chance that for each block (per 12 seconds on Ethereum), there is someone who calls a `withdraw`/`deposit`/`getRewards` function. As a result, `accrued` could be small, leading to `accrued.divDown(totalShares) = 0`. Meanwhile, `_updateRewardIndex()` always advances `lastBalance` when `accrued !=0`:

<https://github.com/code-423n4/2024-10-loopfi/blob/d219f0132005b00a68f505edc22b34f9a8b49766/src/pendle-rewards/RewardManager.sol#L78>

This means the accrued rewards are lost! Nobody will receive the rewards since index has not changed.

More importantly, due to the rounding down error for `accrued.divDown(totalShares)`, there is always a slight loss for the rewards, which is accumulative over time.

### Recommended Mitigation Steps

The fix is simple. Calculate `deltaIndex = accrued.divDown(totalShares)` and advance `lastBalance` by `deltaIndex.mulDown(totalShares)`. In this way, `index` and `lastBalance` will always advance in the same pace; in particular if index does not advance, then `lastBalance` will not advance either. The rounding down error is eliminated too since the `lastBalance` will not be `accrued` but by `deltaIndex.mulDown(totalShares)`.

### Assessed type

Math

**[0xtj24 (LoopFi) confirmed](https://github.com/code-423n4/2024-10-loopfi-findings/issues/25#event-14924296502)**

**[0xAlix2 (warden) commented](https://github.com/code-423n4/2024-10-loopfi-findings/issues/25#issuecomment-2471431354):**
 > @Koolex - I agree that this is an issue; however, the audit [docs](https://github.com/code-423n4/2024-10-loopfi?tab=readme-ov-file#general-questions) states that the ERC20s that are used by the protocol are WETH and PendleLPs which are both 18 decimals.
> 
>> `ERC20 used by the protocol | WETH, PendleLPs`
> 
> But I'm not sure if that should be considered valid in this context.

**[Koolex (judge) commented](https://github.com/code-423n4/2024-10-loopfi-findings/issues/25#issuecomment-2483641241):**
 > There is another issue here. 
> 
> > malicious user keeps calling `getRewards()`

***

## [[H-02] `CDPVault.sol#liquidatePositionBadDebt()` doesn't correctly handle profit and loss](https://github.com/code-423n4/2024-10-loopfi-findings/issues/2)
*Submitted by [pkqs90](https://github.com/code-423n4/2024-10-loopfi-findings/issues/2), also found by [0xAlix2](https://github.com/code-423n4/2024-10-loopfi-findings/issues/29)*

<https://github.com/code-423n4/2024-10-loopfi/blob/main/src/CDPVault.sol#L702> 

<https://github.com/code-423n4/2024-10-loopfi/blob/main/src/PoolV3.sol#L593>

### Impact

When liquidating bad debt, the profit and loss is not correctly handled. This will cause incorrect accounting to lpETH stakers.

### Bug Description

*Note: This is based on the 2024-07 Loopfi audit [H-12](https://github.com/code-423n4/2024-07-loopfi-findings/issues/57) issue. This protocol team applied a fix, but the fix is incomplete.*

There are two issues that needs to be fixed in the new codebase:

1. The `profit` that is passed in `pool.repayCreditAccount(debtData.debt, profit, loss);` should actually use `debtData.accruedInterest`. This is because we should first "assume" full debt and interest is paid off, and calculate the loss part independently.

2. The `loss` is correctly calculated in `PoolV3#repayCreditAccount`, but the if-else branch is incorrectly implemented. Currently, it can't handle the case where both profit and loss is non-zero. This would cause a issue that the loss will not be accounted, and will ultimately cause loss to lpETH holders (loss will be implicitly added to the users who hold lpETH) instead of lpETH stakers.

The second fix was also suggested in the original issue, but it isn't applied.

CDPVault.sol:

```solidity
        takeCollateral = position.collateral;
        repayAmount = wmul(takeCollateral, discountedPrice);
        uint256 loss = calcTotalDebt(debtData) - repayAmount;
        uint256 profit;
        if (repayAmount > debtData.debt) {
@>          profit = repayAmount - debtData.debt;
        }
        ...
@>      pool.repayCreditAccount(debtData.debt, profit, loss); // U:[CM-11]
        // transfer the collateral amount from the vault to the liquidator
        token.safeTransfer(msg.sender, takeCollateral);
```

PoolV3.sol:

```solidity
    function repayCreditAccount(
        uint256 repaidAmount,
        uint256 profit,
        uint256 loss
    )
        external
        override
        creditManagerOnly // U:[LP-2C]
        whenNotPaused // U:[LP-2A]
        nonReentrant // U:[LP-2B]
    {
        ...
        if (profit > 0) {
            _mint(treasury, _convertToShares(profit)); // U:[LP-14B]
@>      } else if (loss > 0) {
            address treasury_ = treasury;
            uint256 sharesInTreasury = balanceOf(treasury_);
            uint256 sharesToBurn = _convertToShares(loss);
            if (sharesToBurn > sharesInTreasury) {
                unchecked {
                    emit IncurUncoveredLoss({
                        creditManager: msg.sender,
                        loss: _convertToAssets(sharesToBurn - sharesInTreasury)
                    }); // U:[LP-14D]
                }
                sharesToBurn = sharesInTreasury;
            }
            _burn(treasury_, sharesToBurn); // U:[LP-14C,14D]
        }
        ...
    }
```

### Recommended Mitigation Steps

In CDPVault, change to `pool.repayCreditAccount(debtData.debt, debtData.accruedInterest, loss)`.

In PoolV3:

```solidity
        if (profit > 0) {
            _mint(treasury, convertToShares(profit)); // U:[LP-14B]
+       }
+       if (loss > 0)
-       } else if (loss > 0) {
            ...
        }
```

**[0xtj24 (LoopFi) confirmed](https://github.com/code-423n4/2024-10-loopfi-findings/issues/2#event-14924455631)**

**[Koolex (judge) commented](https://github.com/code-423n4/2024-10-loopfi-findings/issues/2#issuecomment-2468583902):**
 > Why Profit should be `debtData.accruedInterest`?
> 
> For the second part, could you please provide a case where profit and loss are non-zero in PJQA?

**[pkqs90 (warden) commented](https://github.com/code-423n4/2024-10-loopfi-findings/issues/2#issuecomment-2475321438):**
 > @Koolex - Here's an example scenario:
> 1. User originally taken out a debt of 100, and interest grows to 50, so `debtData.debt = 100, debtData.accruedInterest = 50, calcTotalDebt(debtData) = 150)`.
> 2. User collateral is only 100, and after multiplying `discountPrice`, the `repayAmount` is only 90. Bad debt occurs.
> 3. `loss = calcTotalDebt(debtData) - repayAmount` is equal to `150 - 90 = 60`.
> 4. Since `repayAmount < debtData.debt`, we would have `profit = 0`.
> 
> This means for `PoolV3#repayCreditAccount`, 60 shares would be burned from the treasury, while instead it should be 10 (because original debt was 100, repaid is 90, `100 - 90 = 10`).
> 
> You can also see that if `repayAmount` was 101, we would calculate `profit = 1`, and in `PoolV3#repayCreditAccount` we would mint 1 share instead. This means there is a 61 (`1 - (-60) = 61`) gap in treasury shares when the repaid amount diff is only 11 (`101 - 90 = 11`), which does not make any sense.
> 
> ```solidity
>     function calcTotalDebt(DebtData memory debtData) internal pure returns (uint256) {
>         return debtData.debt + debtData.accruedInterest; //+ debtData.accruedFees;
>     }
> 
>     function liquidatePositionBadDebt(address owner, uint256 repayAmount) external whenNotPaused {
>         ...
>         takeCollateral = position.collateral;
>         repayAmount = wmul(takeCollateral, discountedPrice);
> @>      uint256 loss = calcTotalDebt(debtData) - repayAmount;
>         uint256 profit;
>         if (repayAmount > debtData.debt) {
> @>          profit = repayAmount - debtData.debt;
>         }
>         ...
> @>      pool.repayCreditAccount(debtData.debt, profit, loss); // U:[CM-11]
>         // transfer the collateral amount from the vault to the liquidator
>         token.safeTransfer(msg.sender, takeCollateral);
>     }
> ```
> 
> PoolV3.sol:
>
> ```solidity
>     function repayCreditAccount(
>         uint256 repaidAmount,
>         uint256 profit,
>         uint256 loss
>     )
>         external
>         override
>         creditManagerOnly // U:[LP-2C]
>         whenNotPaused // U:[LP-2A]
>         nonReentrant // U:[LP-2B]
>     {
>         uint128 repaidAmountU128 = repaidAmount.toUint128();
> 
>         DebtParams storage cmDebt = _creditManagerDebt[msg.sender];
>         uint128 cmBorrowed = cmDebt.borrowed;
>         if (cmBorrowed == 0) {
>             revert CallerNotCreditManagerException(); // U:[LP-2C,14A]
>         }
> 
>         if (profit > 0) {
>             _mint(treasury, _convertToShares(profit)); // U:[LP-14B]
>         } else if (loss > 0) {
>             address treasury_ = treasury;
>             uint256 sharesInTreasury = balanceOf(treasury_);
>             uint256 sharesToBurn = _convertToShares(loss);
>             if (sharesToBurn > sharesInTreasury) {
>                 unchecked {
>                     emit IncurUncoveredLoss({
>                         creditManager: msg.sender,
>                         loss: _convertToAssets(sharesToBurn - sharesInTreasury)
>                     }); // U:[LP-14D]
>                 }
>                 sharesToBurn = sharesInTreasury;
>             }
>             _burn(treasury_, sharesToBurn); // U:[LP-14C,14D]
>         }...
>     }
> ```

**[Koolex (judge) commented](https://github.com/code-423n4/2024-10-loopfi-findings/issues/2#issuecomment-2483674091):**
 > @pkqs90 - Could you please point out the incomplete fix? This is important, since if there is no indication that the sponsor intended to fix it, it would be out of scope (according to this [announcement](https://discord.com/channels/810916927919620096/1293582092533497946/1296431424819298365)).

**[pkqs90 (warden) commented](https://github.com/code-423n4/2024-10-loopfi-findings/issues/2#issuecomment-2484527115):**
 > @Koolex - The 2024-07 code had `pool.repayCreditAccount(debtData.debt, 0, loss);` https://github.com/code-423n4/2024-07-loopfi/blob/main/src/CDPVault.sol#L624, and was later fixed to `pool.repayCreditAccount(debtData.debt, profit, loss);` https://github.com/code-423n4/2024-10-loopfi/blob/main/src/CDPVault.sol#L702.
> 
> The suggested fix was also mentioned the original report for [H-12](https://github.com/code-423n4/2024-07-loopfi-findings/issues/57).

***
 
# Medium Risk Findings (5)
## [[M-01] Invalid handling of flash loan fees in `PositionAction::onCreditFlashLoan`, forcing it to always revert](https://github.com/code-423n4/2024-10-loopfi-findings/issues/27)
*Submitted by [0xAlix2](https://github.com/code-423n4/2024-10-loopfi-findings/issues/27), also found by pkqs90 ([1](https://github.com/code-423n4/2024-10-loopfi-findings/issues/8), [2](https://github.com/code-423n4/2024-10-loopfi-findings/issues/7))*

When users take a flash loan, an amount is sent to the receiver, and then some action takes place, after that action that sent amount is expected to be paid and some fee. Users can also call `PositionAction::decreaseLever` through a proxy, to "Decrease the leverage of a position by taking out a credit flash loan to withdraw and sell collateral", after it is called, the flash loan lender sends the credit and calls `onCreditFlashLoan`, which handles all that logic.

This was reported in [Issue 524](https://github.com/code-423n4/2024-07-loopfi-findings/issues/524), and a fix has been implemented. However, the fix is incomplete, and the `onCreditFlashLoan` will always revert when the fees are `>0`.

The fix added includes adding fees to the approval amounts:

```solidity
underlyingToken.forceApprove(address(leverParams.vault), subDebt + fee);
underlyingToken.forceApprove(address(flashlender), subDebt + fee);
```

That fix still misses a point; the amount coming from the flash lender is constant, and that amount will be used to repay a position's debt. The issue here is that all the amount is being used to repay without accounting for the extra fees that the flash lender will be requesting.

This causes `PositionAction::onCreditFlashLoan` to always revert.

### Proof of Concept

Add the following test in `src/test/integration/PositionAction20.lever.t.sol`:

```solidity
function test_leverageDownNotAccountingFees() public {
    // Re-initialize the system to have fees > 0
    flashlender = new Flashlender(IPoolV3(address(liquidityPool)), 0.01 ether);
    liquidityPool.setCreditManagerDebtLimit(address(flashlender), type(uint256).max);
    positionAction = new PositionAction20(
        address(flashlender),
        address(swapAction),
        address(poolAction),
        address(vaultRegistry)
    );

    uint256 depositAmount = 1_000 ether;
    uint256 borrowAmount = 200 ether;

    deal(address(token), user, depositAmount);

    address[] memory assets = new address[](2);
    assets[0] = address(underlyingToken);
    assets[1] = address(token);

    vm.startPrank(user);

    // User deposits 1k ETH collateral
    token.approve(address(vault), type(uint256).max);
    vault.deposit(address(userProxy), depositAmount);

    // User borrows 200 ETH
    userProxy.execute(
        address(positionAction),
        abi.encodeWithSelector(
            positionAction.borrow.selector,
            address(userProxy),
            address(vault),
            CreditParams({amount: borrowAmount, creditor: user, auxSwap: emptySwap})
        )
    );

    vm.expectRevert(bytes("ERC20: transfer amount exceeds balance"));
    userProxy.execute(
        address(positionAction),
        abi.encodeWithSelector(
            positionAction.decreaseLever.selector,
            LeverParams({
                position: address(userProxy),
                vault: address(vault),
                collateralToken: address(token),
                primarySwap: SwapParams({
                    swapProtocol: SwapProtocol.BALANCER,
                    swapType: SwapType.EXACT_OUT,
                    assetIn: address(token),
                    amount: 1 ether,
                    limit: 2 ether,
                    recipient: address(positionAction),
                    residualRecipient: address(positionAction),
                    deadline: block.timestamp,
                    args: abi.encode(weightedPoolIdArray, assets)
                }),
                auxSwap: emptySwap,
                auxAction: emptyPoolActionParams
            }),
            2 ether,
            address(userProxy)
        )
    );

    vm.stopPrank();
}
```

### Recommended Mitigation Steps

```diff
function onCreditFlashLoan(
    address /*initiator*/,
    uint256 /*amount*/,
   uint256 fee,
    bytes calldata data
) external returns (bytes32) {
    ...

    // sub collateral and debt
    ICDPVault(leverParams.vault).modifyCollateralAndDebt(
        leverParams.position,
        address(this),
        address(this),
        0,
-       -toInt256(subDebt)
+       -toInt256(subDebt - fee)
    );

    ...

    return CALLBACK_SUCCESS_CREDIT;
}
```

### Assessed type

DoS

**[amarcu (LoopFi) confirmed](https://github.com/code-423n4/2024-10-loopfi-findings/issues/27#event-14816614121)**

***

## [[M-02] Invalid handling of risdual amount in `PositionAction::onCreditFlashLoan`, forcing it to revert](https://github.com/code-423n4/2024-10-loopfi-findings/issues/26)
*Submitted by [0xAlix2](https://github.com/code-423n4/2024-10-loopfi-findings/issues/26)*

Users can call `PositionAction::decreaseLever` through a proxy, to "Decrease the leverage of a position by taking out a credit flash loan to withdraw and sell collateral". After it is called, the flash loan lender sends the credit and calls `onCreditFlashLoan`, which handles all that logic. When doing so, users are supposed to swap their collateral withdrawn into debt tokens so that the flash loan can be repaid.

The protocol tries to handle the residual amount from the swap (`swapped - paid debt`), by trying to repay extra debt for the designated position, using:

```solidity
if (residualAmount > 0) {
    underlyingToken.forceApprove(address(leverParams.vault), residualAmount);
    ICDPVault(leverParams.vault).modifyCollateralAndDebt(
        leverParams.position,
        address(this),
        address(this),
        0,
        -toInt256(residualAmount)
    );
}
```

However, this is invalid for 2 main reasons:

1. This is trying to repay extra debt than what the user is trying to, which is passed in the `primarySwap.amount`.
2. If the user tries to repay his whole debt using `decreaseLever` the TX will revert, as it'll try to repay some nonexistent debt.

### Proof of Concept

Add the following test in `src/test/integration/PositionAction20.lever.t.sol`:

```solidity
function test_leverageDownWrongResidualHandling() public {
    uint256 depositAmount = 1_000 ether;
    uint256 borrowAmount = 200 ether;

    deal(address(token), user, depositAmount);

    address[] memory assets = new address[](2);
    assets[0] = address(token);
    assets[1] = address(underlyingToken);

    vm.startPrank(user);

    // User deposits 1k ETH collateral
    token.approve(address(vault), type(uint256).max);
    vault.deposit(address(userProxy), depositAmount);

    // User borrows 200 ETH
    userProxy.execute(
        address(positionAction),
        abi.encodeWithSelector(
            positionAction.borrow.selector,
            address(userProxy),
            address(vault),
            CreditParams({amount: borrowAmount, creditor: user, auxSwap: emptySwap})
        )
    );

    userProxy.execute(
        address(positionAction),
        abi.encodeWithSelector(
            positionAction.decreaseLever.selector,
            LeverParams({
                position: address(userProxy),
                vault: address(vault),
                collateralToken: address(token),
                primarySwap: SwapParams({
                    swapProtocol: SwapProtocol.BALANCER,
                    swapType: SwapType.EXACT_IN,
                    assetIn: address(token),
                    amount: vault.virtualDebt(address(userProxy)),
                    limit: 0,
                    recipient: address(positionAction),
                    residualRecipient: address(positionAction),
                    deadline: block.timestamp,
                    args: abi.encode(weightedPoolIdArray, assets)
                }),
                auxSwap: emptySwap,
                auxAction: emptyPoolActionParams
            }),
            201 ether,
            address(userProxy)
        )
    );

    vm.stopPrank();
}
```

Logs:

    Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 2.65s (4.18ms CPU time)

    Ran 1 test suite in 2.66s (2.65s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

    Failing tests:
    Encountered 1 failing test in src/test/integration/PositionAction20.lever.t.sol:PositionAction20_Lever_Test
    [FAIL. Reason: CallerNotCreditManagerException()] test_leverageDownWrongResidualHandling() (gas: 1088990)

### Recommended Mitigation Steps

Rather than using the residual amount to repay excess debt (that might not even exist), transfer it to the designated residual recipient. Alternatively, check if the user still has any remaining debt. If they do, use the residual to repay it; otherwise, transfer the residual amount to the recipient.

### Assessed type

DoS

**[amarcu (LoopFi) confirmed and commented](https://github.com/code-423n4/2024-10-loopfi-findings/issues/26#issuecomment-2456987192):**
 > The error in the failing test is because of a faulty setup. This will happen if the flashlender contract is not added to the poolv3 as a credit manager.

**[0xAlix2 (warden) commented](https://github.com/code-423n4/2024-10-loopfi-findings/issues/26#issuecomment-2471399743):**
 > @Koolex - I believe there's confusion here, the error here isn't because "the flashlender contract is not added to the poolv3 as a credit manager", please let me explain:
> 
> In [PoolV3.sol](https://github.com/code-423n4/2024-10-loopfi/blob/main/src/PoolV3.sol), if you search for `CallerNotCreditManagerException` you can find 2 occurrences, the first is [here](https://github.com/code-423n4/2024-10-loopfi/blob/main/src/PoolV3.sol#L141-L145):
>
> ```solidity
> function _revertIfCallerNotCreditManager() internal view {
>     if (!_creditManagerSet.contains(msg.sender)) {
>         revert CallerNotCreditManagerException(); // U:[PQK-4]
>     }
> }
> ```
>
> This is indeed the case that the sponsor is referencing; however, there's another occurrence [here](https://github.com/code-423n4/2024-10-loopfi/blob/main/src/PoolV3.sol#L587-L589):
>
> ```solidity
> if (cmBorrowed == 0) {
>     revert CallerNotCreditManagerException(); // U:[LP-2C,14A]
> }
> ```
>
> This is the case that the report is discussing, where this error is thrown when the borrowed of a position equals 0.
> 
> You can easily confirm this by changing the amount passed to `amount` in the above PoC to a lower value and see that the test doesn't fail, example below:
>
> ```diff
> function test_leverageDownWrongResidualHandling() public {
>     uint256 depositAmount = 1_000 ether;
>     uint256 borrowAmount = 200 ether;
> 
>     deal(address(token), user, depositAmount);
> 
>     address[] memory assets = new address[](2);
>     assets[0] = address(token);
>     assets[1] = address(underlyingToken);
> 
>     vm.startPrank(user);
> 
>     // User deposits 1k ETH collateral
>     token.approve(address(vault), type(uint256).max);
>     vault.deposit(address(userProxy), depositAmount);
> 
>     // User borrows 200 ETH
>     userProxy.execute(
>         address(positionAction),
>         abi.encodeWithSelector(
>             positionAction.borrow.selector,
>             address(userProxy),
>             address(vault),
>             CreditParams({amount: borrowAmount, creditor: user, auxSwap: emptySwap})
>         )
>     );
> 
>     userProxy.execute(
>         address(positionAction),
>         abi.encodeWithSelector(
>             positionAction.decreaseLever.selector,
>             LeverParams({
>                 position: address(userProxy),
>                 vault: address(vault),
>                 collateralToken: address(token),
>                 primarySwap: SwapParams({
>                     swapProtocol: SwapProtocol.BALANCER,
>                     swapType: SwapType.EXACT_IN,
>                     assetIn: address(token),
> -                   amount: vault.virtualDebt(address(userProxy)),
> +                   amount: vault.virtualDebt(address(userProxy)) / 2,
>                     limit: 0,
>                     recipient: address(positionAction),
>                     residualRecipient: address(positionAction),
>                     deadline: block.timestamp,
>                     args: abi.encode(weightedPoolIdArray, assets)
>                 }),
>                 auxSwap: emptySwap,
>                 auxAction: emptyPoolActionParams
>             }),
>             201 ether,
>             address(userProxy)
>         )
>     );
> 
>     vm.stopPrank();
> }
> ```
> 
> Hence, I believe there's some confusion here and this is a valid medium and would appreciate if you could take another look.

**[Koolex (judge) commented](https://github.com/code-423n4/2024-10-loopfi-findings/issues/26#issuecomment-2483604397):**
 > @amarcu - I have run the PoC and changed the error:
> 
> ```
> // src/PoolV3.sol : L588
>         if (cmBorrowed == 0) {
>             revert CallerNotCreditManagerException(); // U:[LP-2C,14A]
>         }
> ```
> 
> to other a different one. When running the PoC, it throws this error.  This confirms the error is caused when there is no debt left `cmBorrowed == 0`.

***

## [[M-03] `PositionAction4626.sol#_onWithdraw` should withdraw from position CDPVault position instead of `address(this)`](https://github.com/code-423n4/2024-10-loopfi-findings/issues/13)
*Submitted by [pkqs90](https://github.com/code-423n4/2024-10-loopfi-findings/issues/13)*

*Note: This is based on the 2024-07 Loopfi audit [M-35](https://github.com/code-423n4/2024-07-loopfi-findings/issues/81) issue. This protocol team applied a fix, but the fix is incomplete.*

Only the bug in the `_onDeposit()` was fixed, but not the one in `_onWithdraw()`. `PositionAction4626.sol#_onWithdraw` does not withdraw from the correct position, it should withdraw from `position` instead of `address(this)`.

```solidity
    function _onDeposit(address vault, address position, address src, uint256 amount) internal override returns (uint256) {
        address collateral = address(ICDPVault(vault).token());

        // if the src is not the collateralToken, we need to deposit the underlying into the ERC4626 vault
        if (src != collateral) {
            address underlying = IERC4626(collateral).asset();
            IERC20(underlying).forceApprove(collateral, amount);
            amount = IERC4626(collateral).deposit(amount, address(this));
        }

        IERC20(collateral).forceApprove(vault, amount);

        // @audit-note: This was fixed.
        return ICDPVault(vault).deposit(position, amount);
    }


    function _onWithdraw(
        address vault,
        address /*position*/,
        address dst,
        uint256 amount
    ) internal override returns (uint256) {
        // @audit-note: This is still a bug.
@>      uint256 collateralWithdrawn = ICDPVault(vault).withdraw(address(this), amount);

        // if collateral is not the dst token, we need to withdraw the underlying from the ERC4626 vault
        address collateral = address(ICDPVault(vault).token());
        if (dst != collateral) {
            collateralWithdrawn = IERC4626(collateral).redeem(collateralWithdrawn, address(this), address(this));
        }

        return collateralWithdrawn;
    }
```

### Recommended Mitigation Steps

```diff
-       uint256 collateralWithdrawn = ICDPVault(vault).withdraw(address(this), amount);
+       uint256 collateralWithdrawn = ICDPVault(vault).withdraw(position, amount);
```

**[amarcu (LoopFi) confirmed](https://github.com/code-423n4/2024-10-loopfi-findings/issues/13#event-14870931535)**

***

## [[M-04] `PositionActionPendle.sol#_onWithdraw` does not have slippage parameter `minOut` set](https://github.com/code-423n4/2024-10-loopfi-findings/issues/10)
*Submitted by [pkqs90](https://github.com/code-423n4/2024-10-loopfi-findings/issues/10), also found by [ZanyBonzy](https://github.com/code-423n4/2024-10-loopfi-findings/issues/32) and [Bauchibred](https://github.com/code-423n4/2024-10-loopfi-findings/issues/31)*

When performing withdraws on `PositionActionPendle` and exiting Pendle pools, users may lose funds due to not setting slippage.

### Bug Description

*Note: This is a new issue that was introduced by the latest code diff.*

The dataflow for withdrawing on `PositionActionPendle` is:
1. User withdraws collateral (which is a Pendle token) from CDPVault.
2. User performs pendle pool exit.

The issue is in step 2; since `minOut` is set to 0, users may receive less output tokens than expected.

```solidity
    function _onWithdraw(
        address vault,
        address position,
        address dst,
        uint256 amount
    ) internal override returns (uint256) {
        uint256 collateralWithdrawn = ICDPVault(vault).withdraw(address(position), amount);
        address collateralToken = address(ICDPVault(vault).token());

        if (dst != collateralToken && dst != address(0)) {
            PoolActionParams memory poolActionParams = PoolActionParams({
                protocol: Protocol.PENDLE,        
@>              minOut: 0, // @audit-bug: No slippage.
                recipient: address(this),     
                args: abi.encode(
                    collateralToken,          
                    collateralWithdrawn,       
                    dst                        
                )
            });

            bytes memory exitData = _delegateCall(
                address(poolAction),
                abi.encodeWithSelector(poolAction.exit.selector, poolActionParams)
            );

            collateralWithdrawn = abi.decode(exitData, (uint256));
        }
        return collateralWithdrawn;
    }
```

Also note that this is similar to the 2024-07 Loopfi audit finding [M-39](https://github.com/code-423n4/2024-07-loopfi-findings/issues/38), which also talks about slippage in ERC4626. However, this Pendle withdraw exit pool code is new, and not existant in the last audit. Thus this should be considered a new bug.

### Recommended Mitigation Steps

Allow user to set a `minOut` parameter for withdraw functions, especially for Pendle position and ERC4626 position.

**[amarcu (LoopFi) confirmed](https://github.com/code-423n4/2024-10-loopfi-findings/issues/13#event-14870931535)**

***

## [[M-05] `PositionAction.sol#onCreditFlashLoan` may end up with stuck funds for `EXACT_IN` primary swaps](https://github.com/code-423n4/2024-10-loopfi-findings/issues/3)
*Submitted by [pkqs90](https://github.com/code-423n4/2024-10-loopfi-findings/issues/3)*

*Note: This is a new issue that was introduced by the latest code diff.*

When conducting a `decreaseLever` action, the final swap inside `onCreditFlashLoan()` is from collateral token to debt token. If the swap is `EXACT_IN` type, this means all collateral token is used for the swap. The output tokens are then split to two parts:

1. Repay the flashloan (and fees).
2. Send back to CDPVault to repay debt.

However, the second part have some issues. Mainly because if the repaid amount is larger than debt amount, the repayed amount will be capped to the debt amount (See CDPVault.sol code below). This means there may be some debt tokens ending up dangling in the PositionAction.sol contract, which the user does not have access to.

To explain a bit more, this is a reasonable scenario, because the amount of collateral tokens used for swap comes from `uint256 withdrawnCollateral = _onDecreaseLever(leverParams, subCollateral);`, and for PositionAction4626, `_onDecreaseLever()` supports gathering collateral tokens by exiting from pools (e.g., Balancer). This means it is totally possible that the amount of collateral tokens used for swap values more than the user's debt in CDPVault.

PositionAction.sol:

```solidity
    function onCreditFlashLoan(
        address /*initiator*/,
        uint256 /*amount*/,
        uint256 fee,
        bytes calldata data
    ) external returns (bytes32) {
        if (msg.sender != address(flashlender)) revert PositionAction__onCreditFlashLoan__invalidSender();
        (LeverParams memory leverParams, uint256 subCollateral, address residualRecipient) = abi.decode(
            data,
            (LeverParams, uint256, address)
        );

        uint256 subDebt = leverParams.primarySwap.amount;
        underlyingToken.forceApprove(address(leverParams.vault), subDebt + fee);
        // sub collateral and debt
        ICDPVault(leverParams.vault).modifyCollateralAndDebt(
            leverParams.position,
            address(this),
            address(this),
            0,
            -toInt256(subDebt)
        );
        
        // withdraw collateral and handle any CDP specific actions
@>      uint256 withdrawnCollateral = _onDecreaseLever(leverParams, subCollateral);

        if (leverParams.primarySwap.swapType == SwapType.EXACT_IN) {
            leverParams.primarySwap.amount = withdrawnCollateral;

            bytes memory swapData = _delegateCall(
                address(swapAction),
                abi.encodeWithSelector(swapAction.swap.selector, leverParams.primarySwap)
            );

            uint256 swapAmountOut = abi.decode(swapData, (uint256));
            uint256 residualAmount = swapAmountOut - subDebt;

            // sub collateral and debt
@>          if (residualAmount > 0) {
                underlyingToken.forceApprove(address(leverParams.vault), residualAmount);
                ICDPVault(leverParams.vault).modifyCollateralAndDebt(
                    leverParams.position,
                    address(this),
                    address(this),
                    0,
                    -toInt256(residualAmount)
                );
            }
        }
        ...
    }
```

PositionAction4626.sol:

```solidity
    function _onDecreaseLever(
        LeverParams memory leverParams,
        uint256 subCollateral
    ) internal override returns (uint256 tokenOut) {
        // withdraw collateral from vault
        uint256 withdrawnCollateral = ICDPVault(leverParams.vault).withdraw(leverParams.position, subCollateral);

        // withdraw collateral from the ERC4626 vault and return underlying assets
        tokenOut = IERC4626(leverParams.collateralToken).redeem(withdrawnCollateral, address(this), address(this));

        if (leverParams.auxAction.args.length != 0) {
            _delegateCall(
                address(poolAction),
                abi.encodeWithSelector(poolAction.exit.selector, leverParams.auxAction)
            );

            tokenOut = IERC20(IERC4626(leverParams.collateralToken).asset()).balanceOf(address(this));
        }
    }
```

CDPVault.sol:

```solidity
    function modifyCollateralAndDebt(
        address owner,
        address collateralizer,
        address creditor,
        int256 deltaCollateral,
        int256 deltaDebt
    ) public {
        ...
        if (deltaDebt > 0) {
            ...
        } else if (deltaDebt < 0) {
            uint256 debtToDecrease = abs(deltaDebt);

            uint256 maxRepayment = calcTotalDebt(debtData);
@>          if (debtToDecrease >= maxRepayment) {
                debtToDecrease = maxRepayment;
                deltaDebt = -toInt256(debtToDecrease);
            }

            uint256 scaledDebtDecrease = wmul(debtToDecrease, poolUnderlyingScale);
            poolUnderlying.safeTransferFrom(creditor, address(pool), scaledDebtDecrease);
        }
    }
```

### Recommended Mitigation Steps

Send the residual tokens back to `residualRecipient` instead of trying to repay debt.

### Assessed type

Token-Transfer

**[amarcu (LoopFi) confirmed and commented](https://github.com/code-423n4/2024-10-loopfi-findings/issues/3#issuecomment-2437541861):**
 > The scenario is valid if the case where the debt repayment is capped, but not sure about the severity. 

**[Koolex (judge) decreased severity to Medium and commented](https://github.com/code-423n4/2024-10-loopfi-findings/issues/3#issuecomment-2468540319):**
 > @pkqs90 - Please clarify the likelihood and the following in PJQA:
> 
> > To explain a bit more, this is a reasonable scenario, because the amount of collateral tokens used for swap comes from uint256 `withdrawnCollateral = _onDecreaseLever(leverParams, subCollateral);`, and for PositionAction4626, `_onDecreaseLever()` supports gathering collateral tokens by exiting from pools (e.g., Balancer). This means it is totally possible that the amount of collateral tokens used for swap values more than the user's debt in CDPVault.

**[pkqs90 (warden) commented](https://github.com/code-423n4/2024-10-loopfi-findings/issues/3#issuecomment-2475345661):**
 > @Koolex For PositionAction4626 actions, this issue is more likely to occur because:
> 1. Uses a `redeem()` call for ERC4626 vaults to get collateral tokens. Users cannot know exactly how much tokens are withdrawn beforehand, this is a dynamic value.
> 2. If `auxAction.args` exists, it will perform a pool exit to get collateral tokens. Both balancer and pendle pool exits use a dynamic output value (e.g., Balancer uses `EXACT_BPT_IN_FOR_ONE_TOKEN_OUT`)
> 
> Also considering the general use case:
> 1. When conducting collateral token `->` debt token swap, the output of swapped out debt token is a dynamic value.
> 2. If the position was liquidate-able, and is partially liquidated by other users by accidentally frontrunning, the required repay amount would be smaller than expected.
> 
> All above makes it more likely the user over repays his position.
> 
> ```solidity
>     function _onDecreaseLever(
>         LeverParams memory leverParams,
>         uint256 subCollateral
>     ) internal override returns (uint256 tokenOut) {
>         // withdraw collateral from vault
>         uint256 withdrawnCollateral = ICDPVault(leverParams.vault).withdraw(leverParams.position, subCollateral);
> 
>         // withdraw collateral from the ERC4626 vault and return underlying assets
> @>      tokenOut = IERC4626(leverParams.collateralToken).redeem(withdrawnCollateral, address(this), address(this));
> 
>         if (leverParams.auxAction.args.length != 0) {
> @>          _delegateCall(
>                 address(poolAction),
>                 abi.encodeWithSelector(poolAction.exit.selector, leverParams.auxAction)
>             );
> 
>             tokenOut = IERC20(IERC4626(leverParams.collateralToken).asset()).balanceOf(address(this));
>         }
>     }
> ```
> 
> ```solidity
>     function _balancerExit(PoolActionParams memory poolActionParams) internal returns (uint256 retAmount) {
>         ...
>         balancerVault.exitPool(
>             poolId,
>             address(this),
>             payable(poolActionParams.recipient),
>             ExitPoolRequest({
>                 assets: assets,
>                 minAmountsOut: minAmountsOut,
> @>              userData: abi.encode(ExitKind.EXACT_BPT_IN_FOR_ONE_TOKEN_OUT, bptAmount, outIndex),
>                 toInternalBalance: false
>             })
>         );
>     }
> ```

***

# Disclosures

C4 is an open organization governed by participants in the community.

C4 audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.
