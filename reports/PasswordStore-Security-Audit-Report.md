---
title: PasswordStore Security Audit Report
author: Lead Security Researcher
date: \today
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.45\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{1.5cm}
    {\Huge\bfseries PasswordStore Protocol\par}
    \vspace{0.4cm}
    {\LARGE\bfseries Security Audit Report\par}
    \vspace{0.8cm}
    {\Large Version 1.0\par}
    \vspace{1.5cm}
    {\Large\itshape Prepared by: Lead Smart Contract Auditor\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Report Content Begins -->

Prepared by: Lead Security Researcher  
Audited Codebase: [Cyfrin / 3-passwordstore-audit](https://github.com/Cyfrin/3-passwordstore-audit)  
Commit Hash: `2e8f81e263b3a9d18fab4fb5c46805ffc10a9990`

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
  - [System Architecture](#system-architecture)
- [Executive Summary](#executive-summary)
  - [Issues Found Summary](#issues-found-summary)
- [Detailed Findings](#detailed-findings)
  - [High Severity Findings](#high-severity-findings)
    - [[H-01] Storing Plaintext Password in On-Chain Storage Violates Confidentiality](#h-01-storing-plaintext-password-in-on-chain-storage-violates-confidentiality)
    - [[H-02] Missing Access Control in `setPassword` Allows Unauthorized Overwriting](#h-02-missing-access-control-in-setpassword-allows-unauthorized-overwriting)
  - [Gas Optimizations](#gas-optimizations)
    - [[G-01] State Variable `s_owner` Should Be Declared `immutable`](#g-01-state-variable-s_owner-should-be-declared-immutable)
  - [Informational Findings](#informational-findings)
    - [[I-01] Incorrect Parameter Tag in `getPassword` NatSpec Comments](#i-01-incorrect-parameter-tag-in-getpassword-natspec-comments)
    - [[I-02] Event Naming Typo (`SetNetPassword` vs `SetNewPassword`)](#i-02-event-naming-typo-setnetpassword-vs-setnewpassword)
    - [[I-03] Solidity Compiler Version 0.8.18 Contains Known Issues](#i-03-solidity-compiler-version-0818-contains-known-issues)
- [Automated Testing and PoC Verification](#automated-testing-and-poc-verification)
- [Mitigation Architecture and Hardened Implementation](#mitigation-architecture-and-hardened-implementation)

\newpage

# Protocol Summary

`PasswordStore` is a smart contract protocol designed to provide a secure personal vault where an address (the owner) can store a private password that others should not be able to read. The protocol also intends to allow the owner to update this password at any time.

Key intended features include:
1. Only the contract owner can read the stored password.
2. Only the contract owner can update the stored password.
3. Passwords must remain strictly confidential from external observers.

# Disclaimer

The security audit team makes all effort to identify security vulnerabilities, architectural flaws, and optimization opportunities within the codebase during the allotted time period. However, a security audit is a time-scoped analysis and does not constitute a guarantee that the software is completely free of defects or vulnerabilities. Security reviews do not represent an endorsement of the project or its underlying business model.

# Risk Classification

We utilize the standard CodeHawks / Cyfrin severity matrix to evaluate vulnerabilities based on their Likelihood and Impact:

| Likelihood \ Impact | High | Medium | Low |
| :--- | :---: | :---: | :---: |
| **High** | High | High / Medium | Medium |
| **Medium** | High / Medium | Medium | Medium / Low |
| **Low** | Medium | Medium / Low | Low |

- **High Severity:** Critical bugs that lead to direct fund loss, irreversible protocol compromise, complete violation of primary invariants, or unauthorized state hijacking.
- **Medium Severity:** Vulnerabilities that affect contract availability, cause non-critical loss under specific circumstances, or allow logic manipulation.
- **Low Severity:** Minor issues or edge-case behavior deviations with limited financial or operational impact.
- **Gas Optimization:** Changes that reduce transaction execution gas costs without altering contract semantics.
- **Informational:** Code quality improvements, NatSpec documentation corrections, style guide adherence, or compiler recommendations.

\newpage

# Audit Details

## Scope

The audit focused on the core smart contract in the repository:

- **Repository:** `Cyfrin / 3-passwordstore-audit`
- **Target Commit:** `2e8f81e263b3a9d18fab4fb5c46805ffc10a9990`
- **In-Scope Contracts:**
  - `src/PasswordStore.sol` (42 lines total, 25 nSLOC)

## Roles

- **Owner:** Privileged actor who deploys the contract and is intended to have exclusive rights to read and write the stored password.
- **Public / External Users:** Unprivileged accounts that should be completely prohibited from viewing or altering the stored password.

## System Architecture

```text
+------------------------------------------------------------------+
|                       PasswordStore Contract                     |
|                                                                  |
|   State Variables:                                               |
|     - s_owner (address)       [Slot 0]                           |
|     - s_password (string)     [Slot 1]                           |
|                                                                  |
|   Functions:                                                     |
|     - setPassword(string)     [External: Writes Slot 1]          |
|     - getPassword()           [External View: Reads Slot 1]      |
|     - getOwner()              [External View: Reads Slot 0]      |
+------------------------------------------------------------------+
```

# Executive Summary

A thorough review of `PasswordStore.sol` was conducted using a combination of manual line-by-line inspection, adversarial threat modeling, static analysis with Slither, and Proof-of-Concept verification via Foundry test suites.

The audit revealed that **the protocol in its current implementation completely fails to satisfy both of its primary security guarantees**:
1. **Confidentiality Failure:** Storing raw plaintext data in Solidity state variables (`s_password`) does not keep the data secret, as all EVM state storage is publicly readable through JSON-RPC endpoints (`eth_getStorageAt`).
2. **Access Control Failure:** The `setPassword` function lacks access control verification, allowing any external caller to overwrite the stored password arbitrarily.

## Issues Found Summary

| Severity | Count | Open | Resolved |
| :--- | :---: | :---: | :---: |
| High | 2 | 2 | 0 |
| Medium | 0 | 0 | 0 |
| Low | 0 | 0 | 0 |
| Gas Optimization | 1 | 1 | 0 |
| Informational | 3 | 3 | 0 |
| **Total** | **6** | **6** | **0** |

\newpage

# Detailed Findings

## High Severity Findings

### [H-01] Storing Plaintext Password in On-Chain Storage Violates Confidentiality

**Severity:** High  
**Impact:** High  
**Likelihood:** High  
**Context:** `src/PasswordStore.sol#L14`

```solidity
string private s_password;
```

#### Description

The `PasswordStore` contract intends to store a private password that only the owner can retrieve. The contract declares the `s_password` state variable with the `private` visibility modifier.

However, in the Ethereum Virtual Machine (EVM), the `private` visibility modifier only prevents other smart contracts from calling compiler-generated getter functions. It does not provide any cryptographic confidentiality or encryption. All data stored in contract storage slots is completely transparent and publicly accessible to any entity with access to an Ethereum node, block explorer, or JSON-RPC endpoint (such as via `eth_getStorageAt`).

Additionally, passing plaintext passwords as parameters to `setPassword(string memory newPassword)` exposes the password in public transaction calldata recorded immutably on-chain.

#### Impact

Total compromise of confidentiality. Any third-party, MEV bot, or attacker can read the plaintext password from storage slot 1 or inspection of transaction calldata, completely undermining the purpose of the protocol.

#### Proof of Concept

The following Foundry test proves that an unprivileged attacker address can extract and decode the plaintext password directly from storage slot 1 using `vm.load` (the programmatic equivalent of `eth_getStorageAt` / `cast storage`):

```solidity
function test_poc_anyone_can_read_password_from_storage() public {
    address attacker = makeAddr("attacker");
    string memory secretPassword = "superSecretPassword123";

    // 1. Owner sets a secret password
    vm.prank(owner);
    passwordStore.setPassword(secretPassword);

    // 2. Attacker inspects storage slot 1 directly
    vm.startPrank(attacker);
    bytes32 slot1Value = vm.load(address(passwordStore), bytes32(uint256(1)));

    // 3. Attacker decodes string bytes from EVM storage slot
    uint8 length = uint8(slot1Value[31]) / 2;
    bytes memory passwordBytes = new bytes(length);
    for (uint256 i = 0; i < length; i++) {
        passwordBytes[i] = slot1Value[i];
    }
    string memory leakedPassword = string(passwordBytes);

    // 4. Assert that attacker recovered the exact secret password
    assertEq(leakedPassword, secretPassword);
    vm.stopPrank();
}
```

Using the `cast` command-line utility on any RPC endpoint:

```bash
# Query raw slot 1 from contract address
cast storage <CONTRACT_ADDRESS> 1 --rpc-url <RPC_URL>
# Output: 0x737570657253656372657450617373776f72643132330000000000000000002c

# Parse bytes32 to ASCII string
cast parse-bytes32-string 0x737570657253656372657450617373776f72643132330000000000000000002c
# Output: superSecretPassword123
```

#### Recommended Mitigation

Smart contracts operating on public blockchains should never store plaintext secrets.

1. **Client-Side Encryption:** Encrypt sensitive credentials off-chain with the owner's private key before broadcasting them to the network.
2. **Commitment / Hash Verification Scheme:** If the objective is verifying possession of a password rather than retrieving it, store a cryptographic hash (e.g., `keccak256(abi.encodePacked(password, salt))`) or employ zero-knowledge proofs (zk-SNARKs).

\newpage

### [H-02] Missing Access Control in `setPassword` Allows Unauthorized Overwriting

**Severity:** High  
**Impact:** High  
**Likelihood:** High  
**Context:** `src/PasswordStore.sol#L26-L29`

```solidity
/*
 * @notice This function allows only the owner to set a new password.
 * @param newPassword The new password to set.
 */
function setPassword(string memory newPassword) external {
    s_password = newPassword;
    emit SetNetPassword();
}
```

#### Description

The NatSpec documentation of `setPassword` explicitly specifies: *"This function allows only the owner to set a new password."*

However, the function implementation lacks any access control mechanism or conditional check verifying that `msg.sender == s_owner`. As a result, the function can be invoked by any arbitrary account on the network.

#### Impact

Any unauthenticated caller can overwrite the stored password at any time, destroying the integrity of the stored data and depriving the contract owner of exclusive control over their password vault.

#### Proof of Concept

The following Foundry test validates that a random non-owner address can invoke `setPassword` and overwrite the contract state:

```solidity
function test_poc_non_owner_can_set_password() public {
    address attacker = makeAddr("attacker");
    string memory maliciousPassword = "attacker_hacked_password";

    // 1. Attacker calls setPassword
    vm.prank(attacker);
    passwordStore.setPassword(maliciousPassword);

    // 2. Owner attempts to retrieve password and discovers it was overwritten
    vm.prank(owner);
    string memory actualPassword = passwordStore.getPassword();

    assertEq(actualPassword, maliciousPassword);
}
```

#### Recommended Mitigation

Add an access control modifier or require condition to enforce that only `s_owner` can invoke `setPassword`:

```diff
+ error PasswordStore__NotOwner();

+ modifier onlyOwner() {
+     if (msg.sender != s_owner) {
+         revert PasswordStore__NotOwner();
+     }
+     _;
+ }

- function setPassword(string memory newPassword) external {
+ function setPassword(string memory newPassword) external onlyOwner {
      s_password = newPassword;
      emit SetNewPassword();
  }
```

\newpage

## Gas Optimizations

### [G-01] State Variable `s_owner` Should Be Declared `immutable`

**Severity:** Gas Optimization  
**Context:** `src/PasswordStore.sol#L13`

```solidity
address private s_owner;
```

#### Description

The state variable `s_owner` is initialized once in the `constructor()` and is never modified afterwards. State variables that are assigned at deployment time and remain constant throughout the contract lifecycle should be marked with the `immutable` keyword.

#### Impact

Declaring `s_owner` as `immutable` avoids `SLOAD` operations (which cost 2,100 gas for cold access or 100 gas for warm access) when reading the owner in access control checks. Instead, the value is embedded directly into the deployed runtime bytecode, saving significant gas on every execution.

#### Recommended Mitigation

Update the state variable declaration and constructor assignment to use `immutable`:

```diff
- address private s_owner;
+ address private immutable i_owner;

  constructor() {
-     s_owner = msg.sender;
+     i_owner = msg.sender;
  }
```

---

## Informational Findings

### [I-01] Incorrect Parameter Tag in `getPassword` NatSpec Comments

**Severity:** Informational  
**Context:** `src/PasswordStore.sol#L31-L35`

```solidity
/*
 * @notice This allows only the owner to retrieve the password.
 * @param newPassword The new password to set.
 */
function getPassword() external view returns (string memory)
```

#### Description

The `getPassword()` function takes zero input parameters, yet its NatSpec documentation includes `@param newPassword The new password to set.`.

#### Impact

Inaccurate documentation can mislead integrators, auditors, and automated documentation generators.

#### Recommended Mitigation

Update the NatSpec comment to reflect the return value instead of a non-existent parameter:

```diff
  /*
   * @notice This allows only the owner to retrieve the password.
-  * @param newPassword The new password to set.
+  * @return The stored password.
   */
```

\newpage

### [I-02] Event Naming Typo (`SetNetPassword` vs `SetNewPassword`)

**Severity:** Informational  
**Context:** `src/PasswordStore.sol#L16`, `src/PasswordStore.sol#L28`

```solidity
event SetNetPassword();
```

#### Description

The event emitted upon updating the password is named `SetNetPassword` instead of `SetNewPassword`.

#### Impact

Typographical errors in event declarations degrade codebase readability and may cause integration discrepancies with off-chain indexing services (e.g., The Graph subgraphs).

#### Recommended Mitigation

Rename the event to `SetNewPassword`:

```diff
- event SetNetPassword();
+ event SetNewPassword();

  function setPassword(string memory newPassword) external onlyOwner {
      s_password = newPassword;
-     emit SetNetPassword();
+     emit SetNewPassword();
  }
```

---

### [I-03] Solidity Compiler Version 0.8.18 Contains Known Issues {#i-03-solidity-compiler-version-0818-contains-known-issues}

**Severity:** Informational  
**Context:** `src/PasswordStore.sol#L2`

```solidity
pragma solidity 0.8.18;
```

#### Description

The contract locks compiler version `0.8.18`. This version is subject to known compiler bugs documented by the Ethereum Foundation (e.g., `VerbatimInvalidDeduplication`, `FullInlinerNonExpressionSplitArgumentEvaluationOrder`).

#### Recommended Mitigation

Upgrade to a modern, stable compiler version such as `0.8.20` or `>=0.8.24`, verifying compatibility with target deployment networks.

\newpage

# Automated Testing and PoC Verification

All findings were validated using the Foundry testing framework. The full test suite confirms that all unit tests and exploit Proof of Concepts pass deterministically.

### Test Suite Execution Output

```text
$ forge test -vvv
[PASS] test_non_owner_reading_password_reverts() (gas: 13491)
[PASS] test_owner_can_set_password() (gas: 24167)
[PASS] test_poc_anyone_can_read_password_from_storage() (gas: 35612)
[PASS] test_poc_non_owner_can_set_password() (gas: 27032)
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 1.15ms
```

### Coverage Report

| File | % Lines | % Statements | % Branches | % Funcs |
| :--- | :---: | :---: | :---: | :---: |
| `script/DeployPasswordStore.s.sol` | 100.00% (6/6) | 100.00% (6/6) | 100.00% (0/0) | 100.00% (1/1) |
| `src/PasswordStore.sol` | 100.00% (9/9) | 100.00% (6/6) | 100.00% (1/1) | 100.00% (3/3) |
| **Total** | **100.00% (15/15)** | **100.00% (12/12)** | **100.00% (1/1)** | **100.00% (4/4)** |

# Mitigation Architecture and Hardened Implementation

Below is a reference hardened implementation incorporating all audit remediations, access control enforcements, gas optimizations, and cryptographic commitments:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

/**
 * @title PasswordStore
 * @author Secure Smart Contract Auditor
 * @notice Stores a password commitment hash securely. Only the owner can update.
 */
contract PasswordStore {
    /*//////////////////////////////////////////////////////////////
                                 ERRORS
    //////////////////////////////////////////////////////////////*/
    error PasswordStore__NotOwner();

    /*//////////////////////////////////////////////////////////////
                            STATE VARIABLES
    //////////////////////////////////////////////////////////////*/
    address private immutable i_owner;
    bytes32 private s_passwordHash;

    /*//////////////////////////////////////////////////////////////
                                 EVENTS
    //////////////////////////////////////////////////////////////*/
    event SetNewPassword();

    /*//////////////////////////////////////////////////////////////
                                MODIFIERS
    //////////////////////////////////////////////////////////////*/
    modifier onlyOwner() {
        if (msg.sender != i_owner) {
            revert PasswordStore__NotOwner();
        }
        _;
    }

    constructor(bytes32 initialPasswordHash) {
        i_owner = msg.sender;
        s_passwordHash = initialPasswordHash;
    }

    /**
     * @notice Allows only the owner to update the password hash.
     * @param newPasswordHash The new keccak256 hash of the password.
     */
    function setPasswordHash(bytes32 newPasswordHash) external onlyOwner {
        s_passwordHash = newPasswordHash;
        emit SetNewPassword();
    }

    /**
     * @notice Verifies if a given raw password matches the stored commitment.
     * @param rawPassword The plaintext password to verify.
     * @return bool True if the password matches the stored hash, false otherwise.
     */
    function verifyPassword(string memory rawPassword) external view returns (bool) {
        return keccak256(abi.encodePacked(rawPassword)) == s_passwordHash;
    }

    /**
     * @notice Returns the contract owner address.
     */
    function getOwner() external view returns (address) {
        return i_owner;
    }
}
```
