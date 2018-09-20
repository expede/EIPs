---
eip: <to be assigned>
title: Localized Signal-to-Text
author: Brooklyn Zelenka (@expede), Jennifer Cooper (@jenncoop)
discussions-to: <URL>
status: Draft
type: Standards Track
category: ERC
created: 2018-09-15
---

## Simple Summary

An on-chain system for providing user feedback by converting machine-efficient codes into human-readable strings in any language or phrasing. The system does not impose a list of languages, but rather lets users write, share, and use the localization of their choice.

## Abstract

This standard provides a standard interface for fetching a string description of a machine signal in an arbitrary human language.

There are many cases where an end user needs feedback on, or instruction from, a smart contact. Returning a hard-coded string in some language (typically English) only serves a small segment of the global population.

By allowing users to register their own translations, we enable richer messaging that is more culturally and linguistically accurate.

We also set the template string encoding to the widely used IEEE Std 1003.1.

## Motivation

If Ethereum is to be a truly global system usable by experts and lay persons alike, systems to provide feedback on what happened during a transaction are needed in as many languages as possible.

User feedback is a challenge in computing generally, and especially on Ethereum.

There are several machine efficient ways of representing intent, status, state transition, and other semantic signals including enums or ERC-1066 codes.

The developer experience is enhanced by returning easier to consume information with more context. End user experience is enhanced by providing text that can be propagated up to the UI.

### User Feedback

The current state of user feedback involves either reverting "with reason" (often in English), or returning a boolean pass/fail status, neither of which provide much capacity to communicate with a diverse end-user base.

### A Truly Global System

By enabling developers to provide translations, we empower them to supply culturally and linguistically suitable messaging, leading to broader and more distributed access to information.

### Abstracted out of ERC-1066

The concept of status translations was originally proposed as part of ERC-1066. We feel it should be its own standard as it is potentially applicable in other circumstances outside of ERC-1066.

## Specification

### Contract Architecture

Two types of contract: `LocalizationPreferences`, and `Localization`s.

The `LocalizationPreferences` contract functions as a proxy for `tx.origin`.

```diagram
                                                                   +--------------+
                                                                   |              |
                                                          +------> | Localization |
                                                          |        |              |
                                                          |        +--------------+
                                                          |
                                                          |
+-----------+          +-------------------------+        |        +--------------+
|           |          |                         | <------+        |              |
| Requestor | <------> | LocalizationPreferences | <-------------> | Localization |
|           |          |                         | <------+        |              |
+-----------+          +-------------------------+        |        +--------------+
                                                          |
                                                          |
                                                          |        +--------------+
                                                          |        |              |
                                                          +------> | Localization |
                                                                   |              |
                                                                   +--------------+
```


### `Localization`

A contract that holds a simple mapping of codes to their text representations.

```solidity
interface Localization {
  function set(bytes32 _code, string _message) external returns (bool);
  function textFor(bytes32 _code) external view returns (bool _wasFound, string _text);
}
```

#### `set`

Sets a localized text representation for any given `bytes32` code.

```solidity
function set(bytes32 _code, string _message) external returns (bool);
```

#### `textFor`

Fetches the localized text representation.

```solidity
function textFor(bytes32 _code) external view returns (bool _wasFound, string _text);
```

### `LocalizationPreferences`

A proxy contract that allows users to set their preferred `Localization`. Text lookup is then passed on to their preferred contract.

A fallback `Localization` MUST be provided, and routed to if the requester has not explicitly set a preferred `Localization`.

```solidity
interface LocalizationPreferences {
  function set(Localization _localization) external returns (bool);
  function get(bytes32 _code) external view returns (bool _wasFound, string _text);
}
```

#### `set`

Sets a user's preferred `Localization`. The registering user SHOULD be considered `tx.origin`.

```solidity
function set(Localization _localization) external nonpayable returns (bool _success);
```

> [name=brooke] Perhaps this should explode if it fails, rather than return a bool?

> [name=jenn] What would be the benefit of that? Oh wait, this is `set` - yes that makes sense. We would want to stop execution if adding the Localization fails.

#### `get`

Retrieve text for a code found at the user's preferred `Localization` contract.

The first return value (`bool _wasFound`) represents if the text was available at that contract, or if a fallback was used. This information is useful for some UI cases, where there is a desire to explain why the fallback localization was used.

Given a code, retrieves a success status, and mapped human-readable string based on the local preferences set in the contract, or in the event of a missing code, false and an empty string.

```solidity
function get(bytes32 _code) external view returns (bool _wasFound, string _text);
```

## String Format

All strings MUST be encoded as UTF-8.

```solidity
"Špeĉiäl chârãçtérs are permitted"
"As are non-Latin characters: アルミ缶の上にあるみかん。"
"Emoji are legal: 🙈🙉🙊🎉"
"Feel free to be creative: (ﾉ◕ヮ◕)ﾉ*:･ﾟ✧"
```

### Templates

Template strings are allowed, and MUST follow the [C `printf`](http://pubs.opengroup.org/onlinepubs/009696799/utilities/printf.html) conventions.

```solidity
"Knock knock. Who's there? %1s. %1s who? %2s!"
```

#### Interpolation Strategy

Please note that it is highly advisable to return the template string _as is_, with arguments as multiple return values, leaving the actual interpolation to be done off chain.

```solidity
("%1s is an element with the atomic number %2d!", atomName, atomicNumber)

// For example
("%1s is an element with the atomic number %2d!", "Mercury", 80);
// => "Mercury is an element with the atomic number 80!"
```

## Rationale

### `bytes32` Keys

`bytes32` is very efficient since it is the EVM's base word size. Given the enormous number of elements (|A| > 1.1579 × 10<sup>77</sup>), it can embed nearly any practical signal, enum, or state. In cases where an application's key is longer than `bytes32`, hashing that long key can map that value into the correct width.

Designs with widths smaller than `bytes32`, such as `bytes1` in [ERC-1066](https://eips.ethereum.org/EIPS/eip-1066), can be very directly embedded into the larger width. This case is simply a one-to-one mapping of the smaller set into the the larger one.

### UI Independent

Localization logic should be UI independent in order to maintain consistency across many different interfaces.

The base string format will be UTF-8, as it's compatible with all means of strings including all languages, emojis and special characters.

### Boolean Return Values

Setting or retrieving a human-readable string returns with it a boolean value. This is to represent success or failure when looking for a code, and is meant to be used as an alternative to checking if the string is empty. This is also useful as a fallback. In the event that a code has not been mapped for the localization in use, the default localization can be applied instead with more ease.

## Implementation

```solidity
pragma solidity ^0.4.25;

contract Localization {
  mapping(bytes32 => string) private dictionary_;

  constructor() public {}

  // Currently overwrites anything
  function set(bytes32 _code, string _message) external returns (bool) {
    dictionary_[_code] = _message;
    return true;
  }

  function textFor(bytes32 _code) external view returns (bool _wasFound, string _message) {
    return (keccak256(abi.encodePacked(dictionary_[_code])) != keccak256(abi.encodePacked("")), dictionary_[_code]);
  }
}

contract LocalizationPreference {
  mapping(address => address) private registry_;
  address public defaultLocalization;

  constructor(Localization _defaultLocalization) public {
    defaultLocalization = _defaultLocalization;
  }

  function set(Localization _localization) external returns (bool) {
    registry_[tx.origin] = _localization;
    return true;
  }

  function get(bytes32 _code) external view returns (bool, string) {
    return get(_code, tx.origin);
  }

  // Primarily for testing
  function get(bytes32 _code, address _who) public view returns (bool, string) {
    return getLocalizationFor(_who).textFor(_code);
  }

  function getLocalizationFor(address _who) internal view returns (Localization) {
    if (Localization(registry_[_who]) == Localization(0)) {
      return Localization(defaultLocalization);
    } else {
      return Localization(registry_[tx.origin]);
    }
  }
}
```

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
