# Overview and description of the contract system

For readers not familiar with the inner working of the 0xProject's contract system, we provide a brief summary which we hope will provide greater context for understanding this report.

The contract system examined in this review forms the core of the 0x Protocol, for on-chain trading of ERC-20 compatible tokens. The system is deployed as 5 contracts, outlined below.

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [`Exchange`](#exchange)
- [`Proxy`](#proxy)
- [`TokenRegistry`](#tokenregistry)
- [`TokenDistributionWithRegistry`](#tokendistributionwithregistry)
- [`MultiSigWalletWithTimeLockExceptRemoveAuthorizedAddress`](#multisigwalletwithtimelockexceptremoveauthorizedaddress)

<!-- /TOC -->


## `Exchange`

The `Exchange` contract is the primary interface for users to trade through the 0x protocol, and provides a number of `public` functions which can be used to match or cancel orders.

The basic flow for completing an order is:

* A `maker` signs a message creating an order according to the protocol specification. This message specifies the parameters of the order, including the token pair (`makerToken` and `takerToken`) and the corresponding amounts to be traded.   Optionally, a `taker` address may be specified, which will ensure that only that address can fill the order.
* This order and signature can then be communicated off chain, by any means. The expectations is that 'Relayers' will maintain an order book with a list of orders. In order to be compensated for their services, Relayers may choose to list only orders which specify their address as a `feeRecipient`, along with a `takerFee` and/or `makerFee`, which must be paid in the 0x Protocol token (**ZRX**), this is a key element in the utility of the token.
* _NOTE: Everything up until this point is done off-chain._
* A `taker` who wishes to fill the order may then submit the parameters of the order and its signature to the `Exchange` contract. If both parties have sufficient transferable balances of their respective tokens (and `ZRX`), the `Exchange` will complete the transfer. Otherwise the call will error.
* Several different functions are available, which provide a `taker` with various options for filling one or more maker orders: `fillOrder()`, `fillOrKillOrder()`, `batchFillOrders()`, `batchFillOrKillOrders()`, `fillOrdersUpTo()`. It is important to note that the `taker` bears the full gas cost of execution, and risks losing their gas payment if a transaction is an error.
* `makers` may also cancel previously signed orders by calling `cancelOrder()`, or `batchCancelOrders()`.
* If an order has an amount either `filled`, or `cancelled`, the amount is recorded in the corresponding `mapping (bytes32 => uint)`. This is essential for preventing a maker's order from being filled more than once, or filled after cancellation.

Our investigation looked at several risks associated with this contract, including reentrancy, griefing, front running, and rounding errors, which are discussed later in this report.

## `Proxy`

The `Proxy` is the central contract in the system. Traders must `approve()` it to spend a sufficient amount of their token to successfully complete an order.

The `Proxy` has an `authorities` array which lists `Exchange` contract addresses. To fill an order, these addresses can call `Proxy::transferFrom()`, which in turn calls `transferFrom()` on both tokens in an exchange.

This `authorities` array can be updated by the `Proxy`'s `owner` address, making it the upgrade mechanism for deploying new versions of the `Exchange` contract, and deprecating old ones.

Thus the `Proxy` is the least changeable component in the system, and is necessarily simple in nature. _After our initial review, this contract was renamed to `TokenTransferProxy` for clarity._

## `TokenRegistry`

The `TokenRegistry` contract contains a curated list of contract addresses for popular, existing ERC-20 tokens, as well as related metadata including the name, ticker symbol, and a storage hash for the logo on both Swarm and IPFS. This token metadata is used to populate the UI developed by the 0xProject team.

It owned and controlled by single address (see Governance below), which can add or remove token addresses from the listing.

The `TokenRegistry` is separate from the core business logic of the protocol, and supports the use of the system's UI designed by the 0xProject team.  The protocol is not limited to tokens listed in the registry, and users are free to specify other tokens for exchange. It is not truly a part of the protocol, but has some curatorial influence on which tokens are likely to be traded most often.

## `TokenDistributionWithRegistry`

This contract takes a novel approach to the `ZRX` token sale, by making use of the 0x protocol itself, but also obscuring that fact from the end user so that the process closely resembles the format of other recent token sales.

The mechanism of the sale looks like this:

* Prior to the start of the sale
  * Prospective contributors must register to participate through an off-chain process
  * The `owner` of the contract adds these addresses to the `registered` mapping, which serves as a whitelist for allowed contributors
* To initialize the sale:
  * The `owner` calls the `init()` function, which sets the parameters for a large maker order, in which:
    * the `owner` address is the order `maker`
    * the `TokenDistributionWithRegistry` contract is the order `taker`
    * `makerFee` and `takerFee` are zero
* Despite the use of the 0x protocol, contributors can participate simply by send `ETH` directly to the contract. Internally, the process is more complex:
  * The fallback function calls `fillOrderWithEth()`
  * The contributor's `ETH` payment is deposited to an ERC-20 `EtherToken` contract, enabling it to be transferred using the ERC-20 interface. The deposit amount belongs to this `TokenDistributionWithRegistry` address.
  * The `Exchange::fillOrKillOrder()` function is called, transferring the `EtherToken` balance to the `owner` address, and the `ZRX` token balance to this `TokenDistributionWithRegistry` address.
  * The `ZRX` token balance is then forwarded to the contributor.
* There is also a `capPerAddress` value which increases with time during the sale. Initially, the maximum contribution amount will be set low enough so that all registered addresses can contribute, and over time this will increase until the full value of the `ZRX` token offering has been distributed. The same address can contribute multiple times up, so long as it does not exceed the current `capPerAddress` value.


_After our initial review, this contract was renamed to `TokenSale` for clarity._

## `MultiSigWalletWithTimeLockExceptRemoveAuthorizedAddress`

This contract adds two features to the [Gnosis Multisig-Wallet](https://etherscan.io/address/0x851b7f3ab81bd8df354f0d7640efcd7288553419#code).

One is a time lock, which introduces a minimum time delay of `secondsTimeLocked` between the submission and execution of any transaction, this feature is implemented in `MultiSigWalletWithTimeLock.sol`.

The second feature allows for this time delay to be bypassed in the specific case that the transaction is a call to `Proxy::removeAuthorizedAddress()`. This is intended to allow an `Exchange` contract to be quickly disabled in an emergency situation.
