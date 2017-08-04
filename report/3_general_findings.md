# 3 - General Findings

## 3.1 Critical

ConsenSys Diligence found no general issues of critical severity during our review.

## 3.2 Major

### Reentrancy risk from malicious tokens

Any call to an external/untrusted contract has risks which can be difficult to quantify given the dynamic and evolving nature of smart contracts. Although we have not found evidence of an exploit, one which we'd like to discuss is the `Proxy` contract calling the `transferFrom()` function on an untrusted contract, intended to be a token ([/blob/888d5a/contracts/Proxy.sol#L101](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Proxy.sol#L101)).

A malicious token contract could implement its `transferFrom()` to reenter the `Exchange` contract, for example to `fillOrder()` or `cancelOrder()`.

Ref: [Best Practices: Reentrancy](https://github.com/ConsenSys/smart-contract-best-practices#reentrancy)

**Recommendation**

A possible approach would be `require()` that tokens are registered in the `TokenRegistry` to be trade-able.

**Resolution**

After discussing with the 0x team, and re-examining the logic, it appears that the only potential outcome of a reentrant call to `fillOrder()` would be to completely fill a maker order (or multiple maker orders) by reentering with multiple taker orders.

A reentrancy issue would be mitigated as there are no changes being made to internal state variables modifications after these external calls.

We note that a Malicious Token was added in [commit/8ae467](https://github.com/0xProject/contracts/commit/8ae467ad24c29f86209d342ec7a7fc3e124258c9) for testing, but the added test does not verify the outcome of this case of reentry.

Another existing mitigation is the use of `require` checks on all token transfers, as it would revert all changes if any transfer within and call to `fillOrder()` fails.
<br/><br/><br/>



### Front running

Miners have ultimate control over transaction ordering and inclusion of transactions on the Ethereum Blockchain. This means that miners are ultimately able to decide which transactions are filled or canceled. Given this inherent power of miners it opens up a possible form of front running. [Front running](http://www.investopedia.com/terms/f/frontrunning.asp) is the practice of stepping in front of orders placed (or about to be placed) by others to gain a price advantage.

Although 0x uses an off-chain orderbook it is still susceptible to front running as orders are cleared on the blockchain. For example, in Exchange.sol the function fill is susceptible to front running as a miner can always include their transaction first which results in the taker's fill being rejected or only partially filled. Similarly, when a miner sees a maker call cancel they can just fill the order first. Hence, if a maker accidentally places an unfavorable trade they may not be able to cancel it.

`fillOrdersUpTo()` and `batchFillOrders()` help mitigate front running by providing backup orders to fill so the taker is not just left with an error log, but nothing prevents a miner from iterating through the orders and filling them all or profiting from slippage.  For instance, a miner might see a taker call `fillOrdersUpTo()` with a large order and then call fill on the lowest priced order and then profit on the additional slippage from the taker's order.

Additionally, given the nature of blockchains, miners are not the only ones able to front run. As an example, a taker could see another taker broadcast a fill order. The malicious taker could instantly broadcast a competing fill order with a higher gas price to increase the probability of their order being filled first.

Ref: [Best Practices: Transaction-Ordering Dependence (TOD) / Front Running](https://github.com/ConsenSys/smart-contract-best-practices/blob/master/README.md#transaction-ordering-dependence-tod--front-running)

**Recommendation**

For users concerned about front running, a mitigation discussed with 0x is to specify a taker for orders.  More specifically, if one trusts that a relayer will not front run, then specifying the relayer as a taker will not make it possible for such orders to be front runned by other takers.  However, this mitigation does centralize orders towards relayers.

<br/><br/><br/>



### Lack of specifications and documentation

Ref: [Best Practices: specifications and documentation](https://github.com/ConsenSys/smart-contract-best-practices#security-related-documentation-and-procedures)


Our review found a lack of specifications and documentation, without which we are forced to make inferences about what is correct and desired behavior.

The primary documentation we received was the white paper which did not cover many of the interactions and components of the system. For example, the [`Critical` issue of rounding](../report/4_specific_findings.md#41-critical) lacks a specification and originally had [no tests](https://github.com/0xProject/contracts/issues/92).

Another example is the [Token Distribution contract](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/TokenDistributionWithRegistry.sol). We have written up some descriptions in the Overview and Appendix to provide readers with more context.

There was no documentation about the deployment or rollout process.  A good guideline for a [responsible token sale](https://medium.com/@matthewdif/towards-responsible-token-sales-icos-291e69cc9ccf) is that **"Under no circumstance should the money raised be released all at once to the development team"**; there is no documentation how this will be achieved.  The [`Major` issue of how large amounts of ETH will be held](../report/4_specific_findings.md#holding-large-amounts-of-eth-with-multisigwallet) is also unclear.

<br/><br/><br/>

### Rounding errors

Given the 0x protocol allows for partial fills, rounding errors are extremely pertinent to the protocol as they can act as a large hidden cost to takers and ultimately result in the loss of tokens. Rounding errors affect "partial fills", ie. when the remainder (`(fillTakerTokenAmount*makerTokenAmount)%takerTokenAmount`) does not equal 0, and increases linearly as the remainder increases. Furthermore, since the EVM does not support floating point numbers rounding errors need to be approximated. However, the precision of this approximation can be increased by multiplying the remainder (i.e - multiplying the remainder by 10 increases the precision by 1 decimal point).

0x’s [original implementation of the `isRoundingError`](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L477) function was

```
  return (target < 10**3 && mulmod(target, numerator, denominator) != 0);
```

which incorrectly assumed that if the `order.makerTokenAmount` is greater than 1000 there will never be a rounding error. This conflicts with the [comment for the function](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L468) stating:
`Checks if rounding error > 0.1%.`

**Recommendation**

1. As there's no other **documentation** about the system that describes factors such as desired behaviors of the system and the interaction of components and functions like `isRoundingError`, this issue has been classified `Critical` as users of a financial system should have defined and precise behavior. We recommend more documentation on `isRoundingError` including descriptions as they apply to presumably providing users with guarantees and protections, and the limits to those protections and when they will not hold. For example, if certain protections are only provided to users if they trade 1000+ tokens, then those should be clearly documented.

2. **Fix** the implementation using the standard definition of [approximation error](https://en.wikipedia.org/wiki/Approximation_error).

3. **Thoroughly test** `isRoundingError` [[issues/92]](https://github.com/0xProject/contracts/issues/92), including when the rounding error is exactly 10/10000 (0.1%), below at 9/10000, and above at 11/10000. Push `isRoundingError` to determine its limits, for example rounding errors at 1e27/1e30, (1e27-1)/1e30, (1e27+1)/1e30 and beyond.  While some limits may not be reachable in usual practice, it does not mean that they should remain unknown, and most scenarios are possible in testing.  Furthermore, some numbers which may seem large, may still have practical impact and importance for testing if one considers that ETH itself is divisible to 18 decimals, and there are no limits defined in any token standards about limits to decimals.

**Resolution**

1. There's no **documentation** describing limits of `isRoundingError` or its behavior apart from original code comments mentioning 0.1%.

2. 0x used an [exact mathematical equivalent](https://www.wolframalpha.com/input/?i=((a*b%2Fc)+-+floor(a*b%2Fc))+%2F+(a*b%2Fc)+%3D+((a*b)%25c)%2F(a*b)) of approximation error and [implemented `isRoundingError()`](https://github.com/0xProject/contracts/pull/132/commits/c1c4befeac352eaa144cc9c2185e618bea505c82) accordingly.

3. [6 tests](https://github.com/0xProject/contracts/blob/74728c404a1c7e9091074bd88abf454fd374228a/test/ts/exchange/helpers.ts#L81-L136) were added for `isRoundingError()`.  There is no test code with large values to explore its behavior further and determine its limits.

<br/><br/><br/>



### Paying less fees by taking advantage of rounding errors

Another concern consequentially derived from rounding error is a strategy takers may take advantage of to minimize the transaction fees he/she pays. Specifically paidTakerFee is calculated using getPartialAmount function which truncates the decimal. First of all, `fillOrder()` doesn't check the error percentage to make sure it was below 0.1%. Secondly, even if it does, takers could still try to construct his order into smaller chunks to "enjoy a 0.1% discount" in every chunk. Thirdly, since relayer will most likely pick orders with high transaction fees, makers could claim to pay a slightly higher fee in the order while in fact they're not, to get a prioritized inclusion onto the order book.

**Recommendation**

Check for rounding errors in the fees, like [[pull/156]](https://github.com/0xProject/contracts/pull/156), and investigate the impact of such a change (for example how often will fills encounter a rounding error) by running simulations of the exchange.

**Resolution**

None.
<br/><br/><br/>



### Unfillable Orders

Rounding errors cause unfillable orders, which arise when all potential fills result in too high of a rounding error, so the order is essentially bricked. An example of such an order is outlined below:

Alice creates an order of 1001 token A for 3 token B. Bob then fills this order with fillTakerTokenAmount = 2. This order only has a .05% error, so the order goes through without any problems. However, now if any other taker tries to fill the remaining 1 token B `isRoundingError` will always return true as it has a .19% error. Now, this order is in a perpetual limbo and will waste potential takers' gas until Alice cancels the order.

[[pull/146]](https://github.com/0xProject/contracts/pull/146) is a test case demonstrating this issue.

**Recommendation**

More test cases and further quantitative analysis of how frequent unfillable orders can occur.  An ideal would be a mathematical proof of some kind to quantify the issue: for example a rounding error of X% can cause at most Y% unfillable orders.

Develop a tool for takers and relayers that checks for unfillable orders. This will minimize takers wasting gas on unfillable orders.

**Resolution**

None.

<br/><br/><br/>



## 3.3 Medium

### Makers "griefing attack" on takers

"Griefing" attack of creating many orders is possible, allowing a maker to burn people's gas.  For example: a taker is "griefed" by a maker whose order is no longer valid since the maker tokens no longer exist. This is hard to defend against, as it requires constantly monitoring the maker's token allowance given to the Exchange, (and possibly also balance).

**Recommendation**

Provide tools, scripts, libraries to make it easy for people to identify orders which are no longer valid.
<br/><br/><br/>

### Pragma statements should set a precise version [[issues/102]](https://github.com/0xProject/contracts/issues/102)

Ref: [Best Practices: Locking the pragmas to a specific compiler version](https://github.com/ConsenSys/smart-contract-best-practices#lock-pragmas-to-specific-compiler-version).

Contracts should be deployed using a specific compiler version and flags that they have been tested with. Locking the pragmas to a specific compiler version helps ensure this and prevents contracts from accidentally being deployed with a newer compiler version which may have a higher risk of undiscovered bugs.

**Recommendation**

Lock the pragmas to a specific version in all contracts that will be deployed.

Prefer a more "mature" version of the compiler than using the latest most recent compiler released: this reduces the risk of encountering new compiler bugs.

**Resolution** [pull/117](https://github.com/0xProject/contracts/pull/117)

0.4.11 compiler version will be used.  This is a reasonable choice as 0.4.11 has been in the wild for a couple of months (as opposed to 0.4.14 that has been out for 2 weeks).
<br/><br/><br/>



### `ecrecover()` issue in `solidity <0.4.14`

An issue with the implementation of `ecrecover()` in the Solidity compiler was recently fixed on July 31. As Christian Reitwießner [explained](https://www.reddit.com/r/ethereum/comments/6qph9k/solidity_0414_released_security_bugfix_related_to/):

> Some inputs (invalid v value, for example) are considered invalid by the ecrecover precompiled contract. In these situations, it returns the empty byte array instead of an address. Due to the architecture of the EVM, the caller cannot detect whether the precompiled contract returned the empyt byte array or an actual address (this will change in Metropolis). If the precompiled contract returns the empty byte array, nothing is written to memory and whatever was there before stays there. If you just ignore these situations, then this memory content will be the return value of the Solidity ecrecover function. The compiler tried to zero out this memory area before the call since 0.4.0, but this protective measure turned out to be ineffective.

> If you want to check whether your contracts are affected, then debug a transaction (or better, all possible transactions that might cause ecrecover to be called) and see where the memory contents that are supposed to be overwritten by the ecrecover call originate from. Since the usual use-case for ecrecover is to compare the return value to a specific address, the contract could be vulnerable, if an attacker can cause exactly this address to appear at exactly this location in memory.

#### Recommendation

Exchange.sol uses `ecrecover` to validate orders and during its deployment we recommend it be compiled with 0.4.14: there is no time to thoroughly debug to ascertain if this is actually needed.  For the rest of the system, a qualitative evaluation of the benefits of jumping to a week-old 0.4.14 compiler, do not seem to outweigh the risks of potential compiler bugs introduced in 0.4.12, 0.4.13, and 0.4.14.

We do not recommend any permanent code changes as Truffle currently does not support running parallel versions of the Solidity compiler.

There is no documentation about the deployment process, but it appears that Exchange.sol is [currently deployed](https://github.com/0xProject/contracts/blob/74728c404a1c7e9091074bd88abf454fd374228a/migrations/4_configure_proxy.ts#L23) near the end of the process, so it should be possible to ignore the Exchange.sol contract that is deployed with 0.4.11 by Truffle.  An out-of-band compile with 0.4.14 and deploying Exchange.sol, and then calling `tokenTransferProxy.addAuthorizedAddress` with it, should allow the system to function as intended.  The deployment of TokenSale.sol should also reference the 0.4.14 Exchange.sol.  We recommend these steps be documented.

TokenSale.sol also uses `ecrecover`, but it is only active for the duration of the sale and we think the risks are acceptable for it to remain on 0.4.11.  Exchange.sol and TokenSale.sol are the only two contracts that use `ecrecover`.

**Resolution**

Unknown.
<br/><br/><br/>



### Losses related to token base units

It is not mentioned in the Solidity code or white paper that token denominations are all expressed in the smallest possible denominations. This should be explicit as it can result in orders being rejected from relayers' order books due to too low of fees, or users accidentally placing, filling, or cancelling smaller amounts of an order than anticipated. 

It is worth noting that 0x.js provides thorough documentation on token base units, but we believe this same level of documentation should be available to those interfacing directly with the smart contracts.

**Recommendation**

Include more documentation on token denominations in the Solidity code and in the white paper.

**Resolution**

None.
<br/><br/><br/>



### Keep test contracts separate

It is a common (and good) practice to create contracts external to the contract system for testing purposes. However, these contracts should be kept in a separate directory for clarity.

**Recommendation**

Move test contracts to a separate directory such as `contracts/test` or `test/contracts`.

**Resolution**

Fixed. Test contracts were moved to `contracts/test` in [[pull/133]](https://github.com/0xProject/contracts/pull/133/files#diff-879bc9679d7af5c32e07a561199bf7c0).

<br/><br/><br/>



### Avoid the Solidity optimizer

It was discussed with the 0x team, who were receptive to the recommendation that contracts should be compiled without the Solidity optimizer.  This reduces risks of unknown unknowns that can be introduced by the Solidity optimizer.

<br/><br/><br/>



## 3.4 Minor

### Negative maker fees are not possible

The `makerFee` value is a uint, making it impossible to give negative fees, which is sometimes useful for incentivizing liquidity.

**Recommendation**

None. This is a design decision with no impact on security.
<br/><br/><br/>

### Use `enum` for error codes [[issues/105]](https://github.com/0xProject/contracts/issues/105)

From [contracts/Exchange.sol#L29](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L29) onwards, error codes are declared as 8-bit constants. However, it is advisable that this is accomplished instead with an `enum` type as per Solidity specs.

**Recommendation**

Substituting the declaration of error codes as storage variables for an `enum` type.

The purpose of this latter type is exactly to address an enumerable feature through a textual definition throughout the code (with the purpose to not lose readability) which is exactly the case here.

**Resolution**

Fixed by implementing our recommendation in [[pull/111]](https://github.com/0xProject/contracts/pull/111).
<br/><br/><br/>

### Usage of capitalized names for variables

We applaud the attempts to highlight and distinguish the use of certain state variables such as [`ZRX_TOKEN_CONTRACT`](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L35-L36).

In our opinion, accessing state variables, especially writing to them, should be "highlighted" in code to prevent confusion against local variables.  We strongly encourage the community to improve clarity around state variables. In this instance, we would just note that the Solidity style guide recommends that capitals be used for constants (akin to other languages).

**Recommendation**

None. At this stage close to release, we do not think the code churn from renaming these variables is worthwhile.
<br/><br/><br/>



### Spelling, names, grammar...

Although above we have reported the Major issue of a lack of specifications and documentation, and have reviewed them and code comments, we note that there are some spelling mistakes and flawed grammar, and some variable names in code that could be improved, but we do not attempt to record or rectify these.
