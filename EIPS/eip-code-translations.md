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

An on-chain system for providing user feedback to by converting machine-efficient
codes into human-readable strings in any language or phrasing. The system does not
impose a list of languages, but rather lets users write, share, and use the
localization of their choice.

## Abstract

This standard provides a standard interface for fetching a string description of a machine signal in an arbitrary human language.

There are many cases where an end user needs feedback on, or instruction from, a smart contact. Returning a hard-coded string in some language (typically English) only serves a small segment of the global population.

By allowing users to register their own translations, we enable richer messaging that is more culturally and linguistically accurate.

We also set the template string encoding to the widely used IEEE Std 1003.1.

## Motivation

If Ethereum is to be a truly global system usable by experts and lay persons alike,
systems to provide feedback on what happened during a transaction are needed in
as many languages as possible.

User feedback is a challenge in computing generally, and especially on Ethereum.

There are several machine efficient ways of representing intent, status,
state transition, and other semantic signals including enums or ERC-1066 codes.

The developer experience is enhanced by returning easier to consume information
with more context. End user experience is enhanced by providing text that can be
propagated up to the UI.

### User Feedback

The current state of user feedback involves either reverting "with reason" (often in English), or returning a boolean pass/fail status, neither of which provide much capacity to communicate with a diverse end-user base.

### A Truly Global System

By enabling developers to provide translations, we empower them to supply culturally and linguistically suitable messaging, leading to broader and more distributed access to information.

### Abstracted out of ERC1066

The concept of status translations was originally proposed as part of ERC1066. We feel it should be its own standard as it is potentially applicable in other circumstances outside of ERC1066.

## Specification

### Contract Architecture

Two types of contract: a `LocalePreference`, and `Localization`s.

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

A contract that holds a simple mapping of codes to their text representations.

```solidity
interface Localization {
  function textFor(bytes32 _code) external view returns (bool _wasFound, string _text)
}
```

### `textFor`

Fetches the localized text representation.

```solidity
function textFor(bytes32 _code) external view returns (bool _wasFound, string _text) {
```

## `LocalizationPreference`

A proxy contract that allows users to set their preferred `Localization`.
Text lookup is then passed on to their preferred contract.

A fallback `Localization` MUST be provided, and routed to if the requester has
not explicitly set a preferred `Localization`.

```solidity
interface LocalePreferences {
  function set(Localization _localization) external nonpayable returns (bool)
  function textFor(bytes32 _code) external view returns (bool _wasFound_, string _text)
}
```

#### `set`

Sets a user's preferred `Localization`. The registering user SHOULD be considered `tx.origin`.

```solidity
function set(Localization _localization) external nonpayable returns (bool)
```

#### `get`

Retrieve text for a code found at the user's preferred `Localization` contract.

The first return value (`bool _wasFound`) represents if the text was available at
that contract, or if a fallback was used. This information is useful for some UI
cases, where there is a desire to explain why the fallback localization was used.

Given a code, retrieves a success status, and mapped human-readable string based on the local preferences set in the contract, or in the event of a missing code, false and an empty string.

```solidity
function get(bytes32 _code) external view returns (bool _wasFound, string _text)
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

```c
"Knock knock. Who's there? %1s. %1s who? %2s!"
```

#### Interpolation Strategy

Please not that it is highly advisable to return the template string _as is_,
with arguments as multiple return values, leaving the actual interpolation
to be done off chain.

```c
("%1s is an element with the atomic number %2d!", atomName, atomicNumber)

// For example
("%1s is an element with the atomic number %2d!", "Mercury", 80);
// => "Mercury is an element with the atomic number 80!"
```

## Rationale

### `bytes32` Keys

While ERC1066 codes are stored as a `byte`, in order to provide maximal flexibility to the user, codes in this standard are `bytes32`.

### UI Independent

Localization logic should be UI independent in order to maintain consistency across many different interfaces.

The base string format will be UTF-8, as it's compatible with all means of strings including all languages, emojis and special characters.

### Boolean Return Values

Setting or retrieving a human-readable string returns with it a boolean value. This is to represent success or failure when looking for a code, and is meant to be used as an alternative to checking if the string is empty. This is also useful as a fallback. In the event that a code has not been mapped for the localization in use, the default localization can be applied instead with more ease.

* UTF8 / UTF-16?!??!?!
* Super compatible with everything, all the languages, emoji, &c

It can be very useful to insert use-case-specific data in a string

A user may want a high level message without detailed information.
They then just don't include the argument in the template, and it'll be ignored.
Other users will still get the data inserted.

The returned strings may either be simple strings, or contain

* String concatenation and interpolation on chain is notoriously expensive and inefficient
* Return a common format (probably IEEE Std 1003.1 / printf)
  *
  * Downside is that we're passing around type info. Useful when in JSON, &c
    * but not strictly needed? Maybe?


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

contract LocalePreference {
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
