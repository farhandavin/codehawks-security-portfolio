# 🛡️ Smart Contract Security Audit Portfolio

Welcome to my Smart Contract Security Audit Portfolio. This repository showcases my published security audit reports, vulnerability research, exploit Proof-of-Concepts (PoCs), and protocol security reviews across competitive audits (CodeHawks, Sherlock, Code4rena) and private engagements.

**Auditor:** Farhan Davin  
**Methodology:** The Tincho Method (8-Phase Phase-Driven Adversarial Review)  
**Standards:** Cyfrin / CodeHawks Severity Classification  

---

## 📑 Security Audit Reports

| Protocol | Date | Scope | Findings Summary | Report PDF | Report MD |
| :--- | :---: | :--- | :---: | :---: | :---: |
| **PasswordStore** | Sep 2026 | `PasswordStore.sol` (25 nSLOC) | **2 High, 1 Gas, 3 Info** | [📄 Download PDF](./reports/PasswordStore-Security-Audit-Report.pdf) | [📝 Read Markdown](./reports/PasswordStore-Security-Audit-Report.md) |

---

## 🔍 Featured Audit Highlights

### 1. [PasswordStore Protocol Audit](./reports/PasswordStore-Security-Audit-Report.pdf)
- **Target:** Personal private password vault protocol on EVM.
- **Key Vulnerabilities Identified:**
  - **[H-01] Storing Plaintext Password in On-Chain Storage Violates Confidentiality:** Demonstrated extraction of private variables directly from EVM storage slot 1 using `vm.load` and `cast storage`.
  - **[H-02] Missing Access Control in `setPassword`:** Identified arbitrary state mutation allowing any unauthenticated account to overwrite the vault owner's password.
  - **[G-01] State Variable `s_owner` Non-Immutable:** Optimized SLOAD gas overhead by transitioning initialization to immutable bytecode storage.
  - **[I-01 - I-03] NatSpec Inconsistencies, Event Typo, & Solc Compiler Bugs:** Code quality hardening and event synchronization improvements.
- **Verification:** 100% Line, Branch, and Function coverage with Foundry reproducible exploit test cases.

---

## 🔬 Audit Methodology (The Tincho Method)

My security engagements follow a structured 8-phase auditing framework:

1. **Phase 0: Audit Readiness (The Rekt Test)** — Evaluating codebase maturity, documentation, test suite health, static analysis baselines, and dependency pinning.
2. **Phase 1: Scoping & Complexity Mapping** — Quantifying nSLOC, dependency mapping, and complexity tiering.
3. **Phase 2: Reconnaissance & Invariant Hypothesis** — Threat modeling, privileged actor analysis, and protocol invariant definitions.
4. **Phase 3: Line-by-Line Code Review** — Systematic manual inspection of control flows, state transitions, and arithmetic operations.
5. **Phase 4: Adversarial Attack Vectoring** — Modeling front-running/MEV, access control bypasses, storage transparency, reentrancy, and flash loan manipulations.
6. **Phase 5: Automated Testing & PoC Construction** — Writing deterministic Foundry exploit test cases proving vulnerability validity and financial impact.
7. **Phase 6 & 7: Classification & Executive PDF Reporting** — Formatting findings according to CodeHawks severity standards and compiling publication-grade PDF reports with Pandoc & LaTeX (Eisvogel).
8. **Phase 8: Mitigation Review & Re-Testing** — Verifying that client remediation patches resolve root causes without introducing regression issues.

---

## 🛠️ Security Tooling & Stack

- **Testing & Execution Frameworks:** Foundry (`forge`, `cast`, `anvil`), Hardhat
- **Static Analysis & Linters:** Slither, Aderyn, Solhint
- **Fuzzing & Invariant Testing:** Echidna, Foundry Invariant Testing (`vm.assume`, fuzz runs)
- **Report Generation:** Pandoc, Eisvogel LaTeX Engine, KaTeX

---

## 📬 Contact & Profiles

- **GitHub:** [@farhandavin](https://github.com/farhandavin)
- **CodeHawks:** [farhandavin](https://codehawks.com)
- **Specialization:** Solidity, EVM Architecture, DeFi Security, Access Control & Storage Security
