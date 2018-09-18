---
eip: <to be assigned>
title: Human-Readable Signal Representation Architecture
author: Brooklyn Zelenka (@expede), Jennifer Cooper (@jenncoop)
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
human-readable strings in arbitrary languages.

## Abstract
A short (~200 word) description of the technical issue being addressed.

This standard provides a standard interface for fetching a string description of a machine signal in an arbitrary human language.

There are many cases where an end user needs feedback on, or instruction from, a smart contact. Returning a hard-coded string in some language (typically English) only serves a small segment of the global population.

By allowing users to register their own translations .............

## Motivation
<!--The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.-->

* User feedback is abysmal
* Want this to be a truly global system
* Abstracted out of ERC1066, but can also be slightly more general

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).-->

## Contract Architecture

Two types of contract: a `LocalePreferences`, and `Localization`s.

The `LocalePreferences` contract functions as a proxy for `tx.origin`.

```diagram
                                                                 +--------------+
                                                                 |              |
                                                        +------> | Localization |
                                                        |        |              |
                                                        |        +--------------+
                                                        |
                                                        |
+-----------+              +-------------------+        |        +--------------+
|           |              |                   | <------+        |              |
| Requestor | <----------> | LocalePreferences | <-------------> | Localization |
|           |              |                   | <------+        |              |
+-----------+              +-------------------+        |        +--------------+
                                                        |
                                                        |
                                                        |        +--------------+
                                                        |        |              |
                                                        +------> | Localization |
                                                                 |              |
                                                                 +--------------+
```

## `Localization`

Top-level discussion here

```solidity
interface Localization {
  function set(bytes32 _code, string _message) external nonpayable
  function stringFor(bytes32 _code) external view returns (bool _wasFound, string _message)
}
```

### `set`

What it's about yadda yadda

```solidity
function set(bytes32 _code, string _message) external nonpayable {
```

### `stringFor`

What it's about yadda yadda

```solidity
function stringFor(bytes32 _code) external view returns (bool _wasFound, string _message) {
```

## `LocalePreferences`

Top level discussion here

```solidity
interface LocalePreferences {
  function set(Localization _localization) external nonpayable returns (bool)
  function get(bytes32 _code) external view returns (bool, string)
}
```

### `set`

What it's about yadda yadda

```solidity
function set(Localization _localization) external nonpayable returns (bool)
```

### `get`

What it's about yadda yadda

```solidity
function get(bytes32 _code) external view returns (bool, string)
```

## Base String Format

* UTF8
* Super compatible with everything, all the languages, emoji, &c

## Format Strings

It can be very useful to insert use-case-specific data in a string

A user may want a high level message without detailed information.
They then just don't include the argument in the template, and it'll be ignored.
Other users will still get the data inserted.

The returned strings may either be simple strings, or contain

* String concatenation and interpolation on chain is notoriously expensive and inefficient
* Return a common format (probably IEEE Std 1003.1 / printf)
  * http://pubs.opengroup.org/onlinepubs/009696799/utilities/printf.html
  * Downside is that we're passing around type info. Useful when in JSON, &c
    * but not strictly needed? Maybe?

```solidity
"%1d bottles of beer on the wall, %1d bottles of beer. Take one down, pass it around, %2d bottles of beer on the wall"
```

```solidity
("%1s is an element with the atomic number %2d!", atomName, atomicNumber)

// For example
("%1s is an element with the atomic number %2d!", "Mercury", 80);
// => "Mercury is an element with the atomic number 80!"
```


## Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

* Why `bytes32` keys?
* Should not be locked into a specific UI
* Should be compatible with `revert`
* Return a bool, because it *is* just a boolean (found or not)

## Implementation
<!--The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

Link to the main Eth status codes repo. But for now...

```solidity
contract Localization {
  mapping(bytes32 => string) private dictionary_;

  constructor(string _missing) public {}

  // Currently overwrites anything
  function set(bytes32 _code, string _message) external nonpayable {
    dictionary_[_code] = _message;
    return true;
  }

  function stringFor(bytes32 _code) external view returns (bool _wasFound, string _message) {
    return ((dictionary_[_code] != ""), dictionary_[_code]);
  }
}

contract LocalePreferences {
  mapping(address => address) private registry_;
  address public defaultLocale;

  constructor(Localization _defaultLocale) public {
    defaultLocale = _defaultLocale;
  }

  // Should return a true or hex"01"?
  function set(Localization _localization) external nonpayable returns (bool) {
    registry_[tx.origin] = _localization;
    return true;
  }

  // The multiple return here adds a small amount of downstream complexity
  // Do we even care about showing the consumer that their translation wasn't found?
  // Better handled as an event, possibly? (ie: for the devs)
  //
  // Have to think about this some more
  function get(bytes32 _code) external view returns (bool, string) {
    return get(_code, _tx.origin);
  }

  // Primarily for testing
  function get(bytes32 _code, address _who) external view returns (bool, string) {
    return getLocale(_who).stringFor(_code);
  }

  function getLocaleFor(address _who) internal view returns (Localization) {
    if (registry_[_who_] == Localization(0)) {
      return getDefault();
    } else {
      return registry_[tx.origin];
    }
  }
}
```

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
