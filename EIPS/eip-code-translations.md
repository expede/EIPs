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

## Simple Summary
An on-chain system for registering and converting machine-efficient codes into
human-readable strings in arbitrary languages.

## Abstract
This standard provides a standard interface for fetching a string description of a machine signal in an arbitrary human language.

There are many cases where an end user needs feedback on, or instruction from, a smart contact. Returning a hard-coded string in some language (typically English) only serves a small segment of the global population.

By allowing users to register their own translations, we enable richer messaging that is more culturally and linguistically accurate.

## Motivation
<!--The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.-->

### User Feedback

The current state of user feedback involves either reverting "with reason" (often in English), or returning a boolean pass/fail status, neither of which provide much capacity to communicate with a diverse end-user base.

### A Truly Global System

By enabling developers to provide translations, we empower them to supply culturally and linguistically suitable messaging, leading to broader and more distributed access to information.

### Abstracted out of ERC1066

The concept of status translations was originally proposed as part of ERC1066. We feel it should be its own standard as it is potentially applicable in other circumstances outside of ERC1066.


## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).-->

### Contract Architecture

This standard includes two types of contract:  `LocalePreferences`, and `Localization`.

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

### `Localization`

A `Localization` contract is meant to maintain a mapping of codes to human-readable strings. There should be a `Localization` contract for each language a developer wants to support.

```solidity
interface Localization {
  function set(bytes32 _code, string _message) external nonpayable
  function stringFor(bytes32 _code) external view returns (bool _wasFound, string _message)
}
```

#### `set`

Set a human-readable string for the given code.

```solidity
function set(bytes32 _code, string _message) external nonpayable {
```

#### `stringFor`

Get the human-readable string for any given code. Returns a `bool` retrieval status, as well as the message itself, if found, otherwise an empty string.

```solidity
function stringFor(bytes32 _code) external view returns (bool _wasFound, string _message) {
```

### `LocalePreferences`

A `LocalePreferences` contract maintains a registry of `Localization`s, as well any related preferences (ie. default localization).

```solidity
interface LocalePreferences {
  function set(Localization _localization) external nonpayable returns (bool)
  function get(bytes32 _code) external view returns (bool, string)
}
```

#### `set`

Stores a mapping of the provided `Localization` contract. Returns `true` if retrieved successfully, `false` otherwise.

```solidity
function set(Localization _localization) external nonpayable returns (bool)
```

#### `get`

Given a code, retrieves a success status, and mapped human-readable string based on the local preferences set in the contract, or in the event of a missing code, false and an empty string.

```solidity
function get(bytes32 _code) external view returns (bool, string)
```

### Base String Format

The base string format will be UTF-8, as it's compatible with all means of strings including all languages, emojis and special characters.

### Format Strings

It can be useful to insert use-case-specific data into a string. We propose using IEEE Std 1003.1 / `printf` common format for including data.

A user may want a high level message without detailed information. In order to achieve this, they can omit the argument in the template, and it'll be ignored.
Other users will still receive the argument data.

The returned strings may either be simple strings, or contain argument data.

Examples with arguments:

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

### `bytes32` Keys

While ERC1066 codes are stored as a `byte`, in order to provide maximal flexibility to the user, codes in this standard are `bytes32`.

### UI Independent

Localization logic should be UI independent in order to maintain consistency across many different interfaces.

### Boolean Return Values

Setting or retrieving a human-readable string returns with it a boolean value. This is to represent success or failure when looking for a code, and is meant to be used as an alternative to checking if the string is empty. This is also useful as a fallback. In the event that a code has not been mapped for the localization in use, the default localization can be applied instead with more ease.

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
