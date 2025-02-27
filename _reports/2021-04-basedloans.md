---
sponsor: "Based Loans"
slug: "2021-04-basedloans"
date: "2021-05-27"
title: "Based Loans"
findings: "https://github.com/code-423n4/2021-04-basedloans-findings/issues"
contest: 7
---

# Overview

## About C4

Code 432n4 (C4) is an open organization that consists of security researchers, auditors, developers, and individuals with domain expertise in the area of smart contracts.

A C4 code contest is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the code contest outlined in this document, C4 conducted an analysis of Based Loans’ smart contract system written in Solidity. The code contest took place between April 2 and April 7, 2021.

## Wardens

6 Wardens contributed reports to the Based Loans code contest:

- [cmichel](https://twitter.com/cmichelio)
- [gpersoon](https://twitter.com/gpersoon)
- [OxRajeev](https://twitter.com/0xRajeev)
- [Thunder](https://twitter.com/SolidityDev)
- [shw](https://twitter.com/shw9453)
- [toastedsteaksandwich](https://twitter.com/AshiqAmien)

This contest was judged by [Cem](https://twitter.com/cemozer_).

Final report assembled by [ninek](https://twitter.com/_ninek_) and [sockdrawermoney](https://twitter.com/sockdrawermoney).

# Summary

The C4 analysis yielded an aggregated total of 31 unique vulnerabilities. All of the issues presented here are linked back to their original finding.

Of these vulnerabilities, 2 received a risk rating in the category of HIGH severity, 1 received a risk rating in the category of MEDIUM severity, and 13 received a risk rating in the category of LOW severity.

C4 analysis also identified 15 non-critical recommendations.

# Scope

The code under review can be found within the [C4 code contest repository](https://github.com/code-423n4/2021-04-basedloans) and comprises 31 smart contracts written in the Solidity programming language.

# Severity Criteria

C4 assesses severity of disclosed vulnerabilities according to a methodology based on [OWASP standards](https://owasp.org/www-community/OWASP_Risk_Rating_Methodology).

Vulnerabilities are divided into 3 primary risk categories: high, medium, and low.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

Further information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code423n4.com).

<br /><br />

# High Risk Findings

## [[H-01] UniswapConfig getters return wrong token config if token config does not exist](https://github.com/code-423n4/2021-04-basedloans-findings/issues/37)

The `UniswapConfig.getTokenConfigBySymbolHash` function does not work as `getSymbolHashIndex` returns `0` if there is no config token for that symbol (uninitialized map value), but the outer function implements the non-existence check with `-1`.

The same issue occurs also for:

- `getTokenConfigByCToken`
- `getTokenConfigByUnderlying`

When encountering a non-existent token config, it will always return the token config of the **first index** (index 0) which is a valid token config for a completely different token.

This leads to wrong oracle prices for the actual token which could in the worst case be used to borrow more tokens at a lower price or borrow more tokens by having a higher collateral value, essentially allowing undercollateralized loans that cannot be liquidated.

Recommend fixing the non-existence check.

**[ghoul-sol (Based Loans) confirmed](https://github.com/code-423n4/2021-04-basedloans-findings/issues/37#issuecomment-835476066):**

> Addressed in **[this PR](https://github.com/code-423n4/2021-04-basedloans-findings/issues/37#issuecomment-835514226)**

 <br />

## [[H-02] uint(-1) index for not found](https://github.com/code-423n4/2021-04-basedloans-findings/issues/24)

Functions `getTokenConfigBySymbolHash`, `getTokenConfigByCToken` and `getTokenConfigByUnderlying` check returned index against max uint:
index != uint(-1)

-1 should indicate that the index is not found, however, a default value for an uninitialized uint is 0, so it is impossible to get -1. What is even weirder is that 0 will be returned for non-existing configs but 0 is a valid index for the 1st config.

One of the solutions would be to reserve 0 for a not found index and use it when searching in mappings. Then normal indexes should start from 1. Another solution would be to introduce a new mapping with a boolean value that indicates if this index is initialized or not but this may be a more gas costly way.

**[ghoul-sol (Based Loans) confirmed](https://github.com/code-423n4/2021-04-basedloans-findings/issues/24#issuecomment-835514161):**

> `UniswapConfig` has been refactored. Index 0 is considered a non-existent config and all comparison are against that value.

<br /><br />

# Medium Risk Findings

## [[M-01] Reward rates can be changed through flash borrows](https://github.com/code-423n4/2021-04-basedloans-findings/issues/33)

The rewards per market are proportional to their `totalBorrows` which can be changed by a large holder who deposits lots of collateral, takes out a huge borrow in the market, updates the rewards, and then unwinds the position.
They'll only pay gas fees as the borrow / repay can happen in the same block.

The `Comptroller.refreshCompSpeeds` function only checks that the single transaction is called from an EOA, but miners (or anyone if a miner offers services like flash bundles for flashbots) can still run flash-loan-like attacks by first sending a borrow tx increasing the totalBorrows, then the `refreshCompSpeeds` transaction, and then the repay of the borrow, as miners have full control over the transaction order of the block.

The new rate will then persist until the next call to `refreshCompSpeeds`.

Attackers have an incentive to drive up the rewards in markets they are a large supplier/borrower in.

The increased rewards that the attacker receives are essentially stolen from other legitimate users.

Recommend making it an admin-only function or use a time-weighted total borrow system similar to Uniswap's price oracles.

**[ghoul-sol (Based Loans) confirmed](https://github.com/code-423n4/2021-04-basedloans-findings/issues/33#issuecomment-835539656):**

> Restricting `Comptroller.refreshCompSpeeds` function to admin only would centralize an ability to update speeds. A better solution may be a bot that keeps track of markets utilizations and updates speeds when needed. That will also give a way to community to participate.
>
> Also, higher rewards would mean that all participants are getting them and that would bring even more liquidity to the given market and decrease attackers earnings. Attacker could keep moving the liquidity from market to market but everyone would follow quite quickly. If that actually happens, admin has a way to stop the rewards and make `refreshCompSpeeds` admin-only function as last resolution because comptroller is using proxy pattern.

<br /><br />

# Low Risk Findings

## [[G-01] requireNoError can be optimized](https://github.com/code-423n4/2021-04-basedloans-findings/issues/4)

The function `requireNoError` of Cether.sol contains 2 checks on _errCode == uint(Error.NO_ERROR)_.

After the first check it returns. After this errCode == uint(Error.NO_ERROR) will never be true, so doesn't have to be checked.

Recommend replacing _require(errCode == uint(Error.NO_ERROR), string(fullMessage));_
with _require(false, string(fullMessage));_

Note: Solidity 8.4 has new error handling functionality which could replace the logic of `requireNoError`

**[ghoul-sol (Based Loans) confirmed](https://github.com/code-423n4/2021-04-basedloans-findings/issues/4#issuecomment-832765987):**

> It's added to our backlog. Thanks!

<br />

## [[L-01] No account existence check for low-level call in CEther.sol](https://github.com/code-423n4/2021-04-basedloans-findings/issues/16)

Low-level calls call/delegatecall/staticcall return true even if the account called is non-existent (per EVM design). Account existence must be checked prior to calling.

The `doTransferOut()` function was changed from using a `transfer()` function (which reverts) to a `call()` function (which returns a boolean), however there is no account existence check for the destination address to. If it doesn’t exist, for some reason, `call` will still return true (not throw an exception) and successfully pass the return value check on the next line.

The checked call paths don’t seem vulnerable because they use `msg.sender/admin` and not a user-controlled address, but this may be a risk if used later in other contexts. Hence rating as low-risk.

For reference, see this related [high-risk severity finding](https://github.com/trailofbits/publications/blob/master/reviews/hermez.pdf) from Trail of Bit’s audit of Hermez Network.

Recommend checking for account-existence before the `call()` to make this safely extendable to user-controlled address contexts in future.

**[ghoul-sol (Based Loans) confirmed](https://github.com/code-423n4/2021-04-basedloans-findings/issues/16#issuecomment-832762511):**

> Recommended fix has been implemented.

<br />

## [[L-02] sweepToken() function removed in CErc20.sol from original Compound code](https://github.com/code-423n4/2021-04-basedloans-findings/issues/17)

The `sweepToken()` function in the original Compound code whose specified purpose was to recover accidentally sent ERC20 tokens to contract has been removed.

The original code comment says: “A public function to sweep accidental ERC-20 transfers to this contract. Tokens are sent to admin (timelock).” This safety measure is helpful given the number/value of accidentally stuck tokens that are sent to contracts by mistake.

Tokens accidentally sent to this contract will be stuck leading to fund loss for sender.

Recommend retaining this function unless there is a specific reason to remove it here.

### Comments:

**[ghoul-sol (Based Loans) confirmed](https://github.com/code-423n4/2021-04-basedloans-findings/issues/17#issuecomment-835528244):**

> Fixed as recommended, thanks!

<br />

## [[L-03] All except one Comptroller verify functions do not verify anything in Comptroller.sol/CToken.sol](https://github.com/code-423n4/2021-04-basedloans-findings/issues/18)

Six of the seven Comptroller verify functions do nothing. Not sure why their calls in CToken.sol have been uncommented from the original Compound version.

Except `redeemVerify()`, six other verify functions `transferVerify()`, `mintVerify()`, `borrowVerify()`, `repayBorrowVerify()`, `liquidateBorrowVerify()` and `seizeVerify()` have no logic except accessing state variables to not be marked pure. Calls to these functions were commented out in the original Compound code’s CToken.sol but have been uncommented here.

Given that they do not implement any logic, the protocol should not be making any assumptions about any defence provided from their unimplemented verification logic.

There are a number of dummy functions whose comments say “// Shh - currently unused”.

Recommend adding logic to implement verification if that is indeed assumed to be implemented but is actually not. Otherwise, comment call sites.

**[ghoul-sol (Based Loans) confirmed](https://github.com/code-423n4/2021-04-basedloans-findings/issues/18#issuecomment-834649529):**

> Fixed by commenting unused functions.

<br />

## [[L-04] Floating pragma used in Uniswap\*.sol](https://github.com/code-423n4/2021-04-basedloans-findings/issues/19)

Contracts should be deployed using the same compiler version/flags with which they have been tested. Locking the floating pragma, i.e. by not using ^ in pragma solidity ^0.6.10, ensures that contracts do not accidentally get deployed using an older compiler version with unfixed bugs.

For reference, see https://swcregistry.io/docs/SWC-103

Recommend removing ^ in “pragma solidity ^0.6.10” and change it to “pragma solidity 0.6.12” to be consistent with the rest of the contracts.

**[ghoul-sol (Based Loans) confirmed](https://github.com/code-423n4/2021-04-basedloans-findings/issues/19#issuecomment-834911660):**

> Fixed as recommended.

<br />

## [[L-05] Missing input validation may set COMP token to zero-address in Comptroller.sol](https://github.com/code-423n4/2021-04-basedloans-findings/issues/20)

Function `_setCompAddress()` is used by admin to change the COMP token address. However, there is no zero-address validation on the parameter. This may accidentally set COMP token address to zero-address but it can be reset by the admin. Any interim transactions might hit exceptional behavior.

Recommend adding zero-address check to `_comp` parameter of `_setCompAddress()`.

**[ghoul-sol (Based Loans) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/14#issuecomment-832832059):**

> Added to our backlog for future refactoring, thanks!

<br />

## [[L-06] Missing zero/threshold check for maxAssets](https://github.com/code-423n4/2021-04-basedloans-findings/issues/21)

A zero or some minimum threshold check is missing for `newMaxAssets` parameter of `_setMaxAssets()` function which is used by admin to set the maximum number of assets that controls how many markets can be entered.

If accidentally set to 0 then all users cannot enter any market which will significantly affect protocol operations.

Recommend adding zero/threshold check to `newMaxAssets` parameter.

**[ghoul-sol (Based Loans) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/21#issuecomment-835430888):**

> Added to backlog, however, it's a non-critical issue.

**[cemozerr (Judge) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/21#issuecomment-839390697):**

> Rating this as low risk as it could pose a situation where users can not enter any markets.

<br />

## [[L-07] Usage of `address.transfer`](https://github.com/code-423n4/2021-04-basedloans-findings/issues/31)

The `transfer` function is used in `Maximillion.sol` to send ETH to an account.

It is performed with a fixed amount of GAS and might fail if GAS costs change in the future or if a smart contract's fallback function handler is complex.

Consider using the lower-level `.call{value: value}` instead and checking its success return value.

**[ghoul-sol (Based Loans) confirmed](https://github.com/code-423n4/2021-04-basedloans-findings/issues/31#issuecomment-833664536):**

> `Maximillion.sol` is not being used and will be deleted.

<br />

## [[L-08] Unbounded iteration on `refreshCompSpeedsInternal`](https://github.com/code-423n4/2021-04-basedloans-findings/issues/32)

The `Comptroller.refreshCompSpeedsInternal` function iterates over all markets and does expensive computations like updating all borrower / supply indices.

When the total number of markets is high, this iteration could exceed the total block gas amount breaking the functionality and making it impossible to update the reward distribution speed.

Keep the number of markets low and/or adjust the function to be processable in several transactions.

**[ghoul-sol (Based Loans) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/32#issuecomment-835439739):**

> While true, estimated gas to update speed for 50 markets is `3377184` gas. Current block gas limit is `14,999,986`, that means we could in theory, get away with updating as many as 222 markets. This is definitely something to keep in mind along the way, however, in my opinion it's a non-critical issue, low at most.

**[cemozerr (Judge) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/32#issuecomment-839399355):**

> I will rate this as low risk, as it won't be an issue until there are many markets, and does not pose a major risk to user funds.

<br />

## [[L-09] uint[] memory parameter is tricky](https://github.com/code-423n4/2021-04-basedloans-findings/issues/12)

Using memory array parameters (e.g. uint[] memory) as function parameters can be tricky in Solidity, because an attack is possible with a very large array which will overlap with other parts of the memory. See proof of concept below.

The function `propose` of GovernorAlpha.sol seems most vulnerable because this function does not check the validity of the array lengths.

Most other functions do a loop over the array, which will fail with a large array (due to out of gas).

The following functions use a [] memory parameter:

- .\Comptroller.sol: `enterMarkets`, `claimComp`, `claimComp`, `_addCompMarkets`
- .\Governance\GovernorAlpha.sol: `propose`
- .\UniswapOracle\UniswapAnchoredView.sol: `addTokens`
- .\UniswapOracle\UniswapConfig.sol: `_addTokensInternal`

This an example to show the exploit:

```
// based on https://github.com/paradigm-operations/paradigm-ctf-2021/blob/master/swap/private/Exploit.sol

pragma solidity ^0.4.24; // only works with low solidity version

contract test{
    struct Overlap {
        uint field0;
    }
    event log(uint);

  function mint(uint[] memory amounts) public  returns (uint) {   // this can be in any solidity version
       Overlap memory v;
       v.field0 = 1234;
       emit log(amounts[0]); // would expect to be 0 however is 1234
       return 1;
     }

  function go() public { // this part requires the low solidity version
      uint x=0x800000000000000000000000000000000000000000000000000000000000000; // 2^251
      bytes memory payload = abi.encodeWithSelector(this.mint.selector, 0x20, x);
      bool success=address(this).call(payload);
  }
}
```

Recommend adding checks on the size of the array parameters to make sure they are not absurdly long.

**[ghoul-sol (Based Loans) confirmed](https://github.com/code-423n4/2021-04-basedloans-findings/issues/12#issuecomment-834696134):**

> As mentioned, this applies to either admin functions or ones that are using a loop. The `propose` function is used by governance so its outcome will be tested on forked network before voting. I don't see an immediate threat from this solidity bug but we'll keep it in mind.

<br />

## [[L-10] CarefulMath / safe math not allways used](https://github.com/code-423n4/2021-04-basedloans-findings/issues/6)

CarefulMath is used in most calculations, however it isn't always used.

Recommend double checking to see if safe math functions really are not necessary and otherwise add a comment.

**[ghoul-sol (Based Loans) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/6#issuecomment-834908151):**

> Agreed, however, in my opinion it's a non-critical issue.

**[cemozerr (Judge) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/6#issuecomment-839431704):**

> I will rate this as low risk instead of non-critical because although the warden here might not have spotted any real issues, unless CarefulMath / SafeMath is not always used, there might be hidden underflow/overflow bugs.

<br />

## [[L-11] Use 'receive' when expecting eth and empty call data](https://github.com/code-423n4/2021-04-basedloans-findings/issues/25)

Contract CEther fallback function was refactored to be compatible with the Solidity 0.6 version:

```
/**
* @notice Send Ether to CEther to mint
*/
fallback () external payable {
    (uint err,) = mintInternal(msg.value);
    requireNoError(err, "mint failed");
}
```

From Solidity 0.6 documentation:

> "The unnamed function commonly referred to as “fallback function” was split up into a new fallback function that is defined using the fallback keyword and a receive ether function defined using the receive keyword. If present, the receive ether function is called whenever the call data is empty (whether or not ether is received). This function is implicitly payable. The new fallback function is called when no other function matches (if the receive ether function does not exist then this includes calls with empty call data). You can make this function payable or not. If it is not payable then transactions not matching any other function which send value will revert. You should only need to implement the new fallback function if you are following an upgrade or proxy pattern."

In this case, "receive" may be more suitable as the function is expecting to receive ether and empty call data.

Recommend replacing "fallback" with "receive".

**[ghoul-sol (Based Loans) confirmed](https://github.com/code-423n4/2021-04-basedloans-findings/issues/25#issuecomment-835426150):**

> Fixed as recommended

<br />

## [[L-12] Allow borrowCap to be filled fully](https://github.com/code-423n4/2021-04-basedloans-findings/issues/28)

Here the condition should be '<=', not '<' to allow filling the cap fully:

```
require(nextTotalBorrows < borrowCap, "market borrow cap reached");

require(nextTotalBorrows <= borrowCap, "market borrow cap reached");
```

**[ghoul-sol (Based Loans) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/28#issuecomment-833665556):**

> Added to backlog.

<br /><br />

# Non-Critical Findings

## [[N-01] Outdated Compiler](https://github.com/code-423n4/2021-04-basedloans-findings/issues/15)

The project is using Solidity compiler version 0.6.12 which was released in July 2020, while the latest compiler version is 0.8.4. Using such an older version makes the project susceptible to any compiler bugs fixed or dangerous features deprecated since then, and also prevents it from leveraging the newly introduced features.

It may be recognized that this is harder for this project because it is making modifications to an existing older project (Compound) which uses compiler version 0.5.x.

Given Solidity’s fast release cycle, consider using a more recent version of the compiler, such as version 0.7.6.

Given that the project is already going from original Compound’s 0.5.x to 0.6.x, it may as well go to 0.7.x version. This may involve a few more breaking changes for changing from 0.6.x to 0.7.x, but there don’t seem to be that many language-level breaking features (see https://github.com/ethereum/solidity/releases/tag/v0.7.0)

### Comments:

**[ghoul-sol (Based Loans) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/15#issuecomment-833652816):**

> Using latest solidity version is best practice. However, upgrading to 0.7.x or 0.8.x requires significant refactoring and any braking changes in solidity could potentially introduce bugs. Also, upgrading at this stage of the project would delay launch further and may require another audit.

**[cemozerr (Judge) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/15#issuecomment-838997641):**

> I'm changing the severity of the issue to non-significant as Based Loans is a fork of Compound codebase, and there are no compiler-related bugs in Compound codebase AFAIK.

<br />

## [[N-02] Missed NatSpec @param for newly introduced parameter distributeAll](https://github.com/code-423n4/2021-04-basedloans-findings/issues/22)

The `distributeSupplierComp()` function has been modified to take in a third parameter which is a boolean `distributeAll`. But the corresponding NatSpec comments for the function have not been updated to add this new parameter. This could lead to minor confusion where NatSpec is consulted.

Recommend adding @param for `distributeAll` parameter.

**[ghoul-sol (Based Loans) confirmed](https://github.com/code-423n4/2021-04-basedloans-findings/issues/22#issuecomment-832829878):**

> Fixed as recommended.

<br />

## [[N-02] Privileged roles](https://github.com/code-423n4/2021-04-basedloans-findings/issues/35)

Admins can change the `comp=blo` address using `_setCompAddress` and stop pending payouts using `_dropCompMarket`.

The allotted rewards of the users may not be paid out anymore due to admins changing the reward token (`comp`) address.

Privileged admin roles make the protocol less predictable for users leading to hesitance and lost opportunity costs.

Recommend only setting the `comp/blo` address if it has not been set already.

Recommend distributing the rewards up to now before cancelling rewards using `_dropCompMarket`.

**[ghoul-sol (Based Loans) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/35#issuecomment-832725953):**

> This is technically correct, however, having a context that admin role is only temporary and will be moved to governance in the near future, I don't consider this as an issue. Especially that `Comptroller` is using a proxy pattern so admin can always change the implementation at will. I consider this a non-critical issue.

**[cemozerr (Judge) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/35#issuecomment-839425430):**

> I'm rating this a non-critical issue, as the `Comptroller` using a proxy pattern would make this change redundant.

<br />

## [[N-03] `UniswapAnchoredView`'s `PriceUpdated` event is never fired](https://github.com/code-423n4/2021-04-basedloans-findings/issues/38)

`UniswapAnchoredView`'s `PriceUpdated` event is never fired.

Unused code can hint at programming or architectural errors.

Recommend using it or removing it.

**[cemozerr (Judge) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/38#issuecomment-839428972):**

> I'm rating this as non-critical as an unused event has no drawbacks.

<br />

## [[N-04] Multiple error enums with overlapping values](https://github.com/code-423n4/2021-04-basedloans-findings/issues/1)

There are 3 error enums, which have overlapping values. This allows for mistakes with error codes and might make troubleshooting of deployed code more difficult.

There did not appear to be any such mistakes in the current code, but changes in the future might introduce mistakes.

ErrorReporter.sol:

```
contract ComptrollerErrorReporter {
    enum Error {
        NO_ERROR,
        UNAUTHORIZED,
        COMPTROLLER_MISMATCH,

contract TokenErrorReporter {
    enum Error {
        NO_ERROR,
        UNAUTHORIZED,
        BAD_INPUT,
```

CarefulMath.sol

```
contract CarefulMath {
    enum MathError {
        NO_ERROR,
        DIVISION_BY_ZERO,
```

Recommend inserting dummy values in the enums to make sure all equivalent numeric values are different.

Take care that the same enum values still have the same underlying value to prevent new mistakes.

You could for example do the following:

```
ComptrollerErrorReporter  NO_ERROR = 0
ComptrollerErrorReporter  UNAUTHORIZED = 1
ComptrollerErrorReporter  COMPTROLLER_MISMATCH = 2
TokenErrorReporter        NO_ERROR = 0
TokenErrorReporter        UNAUTHORIZED = 1
TokenErrorReporter        BAD_INPUT = 102
CarefulMath.sol           NO_ERROR = 0
CarefulMath.sol           DIVISION_BY_ZERO = 201
```

Note: In a few occasions there is a reliance on the fact that NO_ERROR = 0; see seperate issue.

Note this might break compatibility with Compound and/or other deployed code.

**[ghoul-sol (Based Loans) acknowledged](https://github.com/code-423n4/2021-04-basedloans-findings/issues/1#issuecomment-833645714):**

> This is for sure confusing and should be refactored. However, it has very low priority so I'm going to add it to the backlog.

<br />

## [[N-05] now is still used](https://github.com/code-423n4/2021-04-basedloans-findings/issues/10)

Most of the time `block.timestamp` is used, however in 1 location `now` is still used.

The global variable now is deprecated in solidity 7:
https://docs.soliditylang.org/en/v0.7.0/070-breaking-changes.html#changes-to-the-syntax

Recommend replacing `now` with `block.timestamp`.

**[ghoul-sol (Based Loans) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/10#issuecomment-832829309):**

> Added to backlog for later refactoring, thanks!

<br />

## [[N-06] Reliance on the fact that NO_ERROR = 0](https://github.com/code-423n4/2021-04-basedloans-findings/issues/2)

In several occasions it's relied upon that the error value NO_ERROR is equivalent to 0.

No problems detected based on this yet, but however in most locations there is an explicit check for NO_ERROR and comparing with 0 allows for possible future mistakes (especially if the enums would change).

Recommend replacing 0 with the appropriate version of ...NO_ERROR

**[ghoul-sol (Based Loans) acknowledged](https://github.com/code-423n4/2021-04-basedloans-findings/issues/2#issuecomment-833644248):**

> Added to backlog.

<br />

## [[N-07] Alphabetical order not complied with (contrary to the comments)](https://github.com/code-423n4/2021-04-basedloans-findings/issues/3)

The enum `FailureInfo` in ErrorReporter.sol has a comment that the values are sorted in alphabetical order.

However they are not in alphabetical order.

Recommend sorting the enum values in alphabetical order or remove the comment.

**[ghoul-sol (Based Loans) acknowledged](https://github.com/code-423n4/2021-04-basedloans-findings/issues/3#issuecomment-833643409):**

> Added to backlog, thanks!

<br />

## [[N-08] requireNoError not used in a consistent way](https://github.com/code-423n4/2021-04-basedloans-findings/issues/5)

Cether.sol has a function `requireNoError` to check for errors. This is used most of the time, however in one occasion it isn't used.

```
function getCashPrior() internal view returns (uint) {
    (MathError err, uint startingBalance) = subUInt(address(this).balance, msg.value);
    require(err == MathError.NO_ERROR);
    return startingBalance;
}
```

Recommend replacing _require(err == MathError.NO_ERROR);_

with:

_requireNoError(err, "getCashPrior failed");_

**[ghoul-sol (Based Loans) acknowledged](https://github.com/code-423n4/2021-04-basedloans-findings/issues/5#issuecomment-833646776):**

> Technically, the code works but I agree that consistency should be kept. Added to backlog.

<br />

## [[N-09] uint(-1)](https://github.com/code-423n4/2021-04-basedloans-findings/issues/7)

In several occasions constructions like `uint(-1)` and `uint96(-1)` are used the reference the maximum values of uint and uint96.

This relies on the peculiarities of numbers.

Solidity also allows the following constructions:

- type(uint).max;
- type(uint96).max;

.\CToken.sol: startingAllowance = uint(-1);
.\CToken.sol: if (startingAllowance != uint(-1)) {
.\CToken.sol: if (repayAmount == uint(-1)) {
.\CToken.sol: if (repayAmount == uint(-1)) {
.\Governance\Blo.sol: if (rawAmount == uint(-1)) {
.\Governance\Blo.sol: amount = uint96(-1);
.\Governance\Blo.sol: if (spender != src && spenderAllowance != uint96(-1)) {
.\UniswapOracle\UniswapConfig.sol: if (index != uint(-1)) {
.\UniswapOracle\UniswapConfig.sol: if (index != uint(-1)) {
.\UniswapOracle\UniswapConfig.sol: if (index != uint(-1)) {

Recommend replacing `uint(-1)` with `type(uint).max` and replacing `uint96(-1)` with `type(uint96).max`.

**[ghoul-sol (Based Loans) acknowledged](https://github.com/code-423n4/2021-04-basedloans-findings/issues/7#issuecomment-832828693):**

> Added to backlog for later refactoring, thanks!

<br />

## [[N-10] More readable constants](https://github.com/code-423n4/2021-04-basedloans-findings/issues/8)

Some constant values are difficult to read in one time because they have at lot of 0's. Solidity allows \_ to separate series of zeroes.

Recommend the following replacements:

- 1000000e18 with 1\_000\_000e18
- 4000000e18 with 4\_000\_000e18
- 100000000e18 with 100\_000\_000e18

### Comments:

**[ghoul-sol (Based Loans) confirmed](https://github.com/code-423n4/2021-04-basedloans-findings/issues/8#issuecomment-833640933):**

> Fixed as recommended

<br />

## [[N-11] function getUnderlyingPrice compares against "cETH"](https://github.com/code-423n4/2021-04-basedloans-findings/issues/26)

Contract CompoundLens functions `cTokenMetadata` and `cTokenBalances` compare against "bETH" while contract `SimplePriceOracle` function `getUnderlyingPrice` compares against "cETH". It is not clear if this `SimplePriceOracle` will be used in production, probably only for testing, but still would be nice to unify it across all the contracts.

Recommend replacing "cETH" with "bETH" in SimplePriceOracle function `getUnderlyingPrice`.

**[ghoul-sol (Based Loans) acknowledged](https://github.com/code-423n4/2021-04-basedloans-findings/issues/26#issuecomment-832787817):**

> This is not meant to be used on production, however, this contract is confusing and would not work if used so it was deleted. Thanks for pointing it out!

<br />

## [[N-12] Use 'interface' keyword for interfaces](https://github.com/code-423n4/2021-04-basedloans-findings/issues/27)

Interfaces are declared as contracts. For example, `ComptrollerInterface` name indicates that it should be an interface but it is declared as a contract. Solidity has a keyword "interface" that can be used here.

Recommend declaring interfaces with 'interface' keyword.

### Comments:

**[ghoul-sol (Based Loans) acknowledged](https://github.com/code-423n4/2021-04-basedloans-findings/issues/27#issuecomment-832778900):**

> Added to backlog, thanks!

<br />

## [[N-13] [Info] functions 'getUnderlyingPriceView' and 'price' are too similar](https://github.com/code-423n4/2021-04-basedloans-findings/issues/29)

Not a bug, just FYI:

Function `getUnderlyingPriceView` is too similar to function `price`. It would be best to avoid code duplication by extracting common code and using it where necessary. Less code duplication makes it easier to maintain it and improves readability.

**[ghoul-sol (Based Loans) acknowledged](https://github.com/code-423n4/2021-04-basedloans-findings/issues/29#issuecomment-832766933):**

> It's added to our backlog for refactoring, thanks!

<br />

## [[N-14] Requires a non-zero address check when deploying `CErc20` tokens and `CEther`.](https://github.com/code-423n4/2021-04-basedloans-findings/issues/39)

During the deployment of the contracts `CErc20` and `CErc20Immutable`, both input parameters `underlying_` and `ComptrollerInterface` lack a non-zero address check. In `CEther`, the `ComptrollerInterface` is not required to be non-zero either. If any of them were provided as `0` accidentally, there is no way to change the values, and the contract should be redeployed.

Recommend adding non-zero address checks in the constructor of the `CErc20`, `CErc20Immutable`, and `CEther` contracts.

**[ghoul-sol (Based Loans) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/39#issuecomment-835427401):**

> It's definitely a good practice to require non-zero address, however, it's not a threat. Severity should be 0.
>
> Added to backlog, thanks!

**[cemozerr (Judge) commented](https://github.com/code-423n4/2021-04-basedloans-findings/issues/39#issuecomment-839440805):**

> Both the impact and the likelihood of this bug is low, so rating this as non-critical.

<br />

## [[N-15] Missing event visibility in \_setCompAddress() function](https://github.com/code-423n4/2021-04-basedloans-findings/issues/13)

The `_setCompAddress()` function in the Comptroller contract does not emit an event when changing the comp address. While this does not impose any security risk, it does hinder a users ability to view any changes made to the comp address through the contract's lifetime.

It is recommended to emit an event indicating the old comp address, and the new comp address to be used when calling the `_setCompAddress()` function. An example of such an event is `event NewCompAddress(address oldCompAddress, address newCompAddress)`.

**[ghoul-sol (Based Loans) confirmed](https://github.com/code-423n4/2021-04-basedloans-findings/issues/13#issuecomment-832835787):**

> Fixed as recommended.

# Disclosures

C4 is an open organization governed by participants in the community.

C4 Contests incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher risk issues. Contest submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code, but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.
