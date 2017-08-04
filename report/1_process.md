# 1. Summary of the Review process and scope

<!-- **TODO** title of this section and content below should match up more -->

## 1.1 Process

Our review was conducted in two distinct phases, which we review to as initial and final reviews. The initial phase looked at the contract system code as it was provided to us by 0x upon commencement. The final review looked at the system code after our recommendations were incorporated. Thus each issue has a note about the **resolution**, and our assessment of it.

## 1.2 Scope

### Source code 

The code base reviewed was in the [0xProject/contracts](https://github.com/0xProject/contracts/tree/888d5a02573572240f4c55e03238be603c13c469) repository, with commit hash `888d5a02573572240f4c55e03238be603c13c469`.

Revisions made after the initial review, can be found in the `frozenUpdated` branch at commit hash `74728c404a1c7e9091074bd88abf454fd374228a`.

### Documentation

0x provided ConsenSys Diligence with the following documentation:

* The [0x Project Whitepaper](https://0xproject.com/pdfs/0x_white_paper.pdf)
* The [README](https://github.com/0xPro ject/contracts/blob/master/README.md) file for the [0xProject/contracts](https://github.com/0xProject/contracts/tree/frozen) repository.

### Dynamic tests

The pre-existing tests for [0xProject/contracts](https://github.com/0xProject/contracts/tree/frozen) repository were executed using the truffle framework, run against contracts deployed on a local instance of testrpc.

In order for the tests to succeed, the module `ethereumjs-testrpc` had to be uninstalled and reinstalled specifying version `3.0.2`.

```
$ npm uninstall -g ethereumjs-testrpc
$ npm install -g ethereumjs-testrpc@3.0.2
```

The revised code base now includes the script `npm run testrpc`,

### Review Goals

0x contracted ConsenSys Diligence to review the codebase for the following properties:

**Security**: identification of security related issues within each
contract and within the system of contracts.

**Sound Architecture**: evaluation of the architecture of this system through the lens of established smart contract best practices and general software best practices.

**Code Correctness and Quality**:
A full review of the contract source code.  The primary areas of focus include:

* Correctness (does it do what it is supposed to do)
* Readability (how easily it can be read and understood)
* Sections of code with high complexity
* Improving scalability
* Quantity and quality of test coverage
