---
eip: <to be assigned>
title: Human-Readable Signal Representation Archetecture
author: Brooklyn Zelenka (@expede), Jennifer Cooper (@jncoops)
discussions-to: <URL>
status: Draft
type: Standards Track
category: ERC
created: 2018-09-15
requires: 1066 [...maybe]
---

The title should be 44 characters or less.

## Simple Summary
If you can't explain it simply, you don't understand it well enough."
Provide a simplified and layman-accessible explanation of the EIP.

An on-chain system for registering and converting machine-efficient codes into
human-readable strings in arbitrary languages

## Abstract
A short (~200 word) description of the technical issue being addressed.

This standard provides a standard interface for fetching a string description of a machine signal in an arbitrary human language.

There are many cases where an end user needs feedback on, or instruction from, a smart contact. Returning a hard-coded string in some language (typically English) only serves a small segment of the global population.

By allowing users to register their own translations .............

## Motivation
<!--The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.-->

* User feedback is abysmal
* Want this to be a truly global system

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).-->

## Contract Architecture

Two types of contract: a `LocalePreferences`, and `Localization`s.

```solidity
contract Localization {
  mapping(bytes32 => string) private dictionary_;

  constructor() public {}

  // Currently overwrites anything
  function set(bytes32 _code, string _message) external nonpayable {
    dictionary_[_code] = _message;
    return true;
  }

  function get(bytes32 _code) external nonpayable returns (bool, string) {

  }

  function fallback() public nonpayable returns (string) {
  }
}

contract LocalePreferences {
  mapping(address => address) private registry_;
  address private defaultLocale_;

  constructor(Localization _defaultLocale) public {
    defaultLocale_ = _defaultLocale;
  }

  function getDefault() public returns (Localization) {
    return defaultLocale_;
  }

  // Should return a true or hex"01"?
  function set(Localization _localization) external nonpayable returns (bool) {
    registry_[tx.origin] = _localization;
    return true;
  }

  // Maaaaaybe too big?
  function get(bytes32 _code) external nonpayable returns (string) {
    return registry_[tx.origin].get(_code);
  }
}
```

## Template Strings

The returned strings may either be simple strings, or contain

## Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

* Should not be locked into a specific UI
* Should be compatible with `revert`

## Implementation
<!--The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
