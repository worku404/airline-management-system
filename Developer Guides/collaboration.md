
# Engineering Governance & Collaboration Framework
## Airline Management System (AMS) Project

This document establishes the mandatory professional standards for the development and integration of the AMS modules. All contributors are expected to uphold these protocols to ensure system reliability and architectural purity.

---

## 1. Architectural Mandates
The project enforces a **strictly modular, backend-first architecture**. To ensure consistency across the Booking Pipeline and Inventory Logic, the following rules apply:

__Error Handling (Zero-Exception Policy)__: All modules must utilize explicit `Status` return objects to communicate success or failure states.
__Data Integrity__: Financial data must be represented as `long long` (cents) to eliminate floating-point precision issues.
__Modern C++ Standards__: Implementation must leverage C++17 features, specifically `std::unique_ptr` for resource management and `std::optional` for state queries.
__State Management__: The system is transient and session-based; internal state must be managed strictly in-memory without external database dependencies.

## 2. Interface Control & Module Ownership
Each engineer is the "Owner" of their assigned module's interface. Changes to header files (`.h`) require a formal review by the team.

| Functional Module | Domain Responsibility | Interface Requirement |
| __Flight Manager__| Registry maintenance & IATA search logic. | Must validate all airport codes. |
| __Inventory Service__ | Atomic seat tracking & concurrency guards. | Strict decoupling from Check-in logic. |
| __Reservation Engine__ | PNR lifecycle & booking validation. | Must verify inventory before confirmation. |
| __Boarding Controller__ | Check-in operations & boarding group logic. | Triggers pass generation upon PNR validation. |

---

## 3. Deployment & Git Workflow
We utilize a __Restricted Merge Strategy__ to maintain the integrity of the core system logic.

### 3.1 Branching & Merging
__Feature Isolation:__ Work must be performed on branches following the `[member]/[feature]` convention.
__Peer Review:__ A minimum of one peer approval is required for all Pull Requests (PRs).
__History Maintenance:__ Branches must be rebased or squashed before merging to maintain a clean project history.

### 3.2 Commit Standards
Commits should be atomic and descriptive. Use prefixes such as `feat:`, `fix:`, or `docs:` to categorize changes for the automated project timeline.

---

## 4. Communication & Synchronization Protocols
__Daily Status (Async):__ A concise update on progress and immediate blockers.
__Bi-Weekly Sync:__ A mid-week blocker check followed by a formal end-of-week demo session.
__Decision Logging:__ All design trade-offs (e.g., choice of PNR algorithm) must be documented in the **Decision Log** within `docs/technical.md`.

---

## 5. Quality Assurance Checklist
Before submitting a PR, ensure the following criteria are met:
__Wall/Werror:__ The code compiles without warnings on target compilers.
__Documentation:__ All public-facing API functions are documented with parameter descriptions and return states.
__Contract Adherence:__ Error messages must follow the project's standard error-model formatting (e.g., `Error: [Module]: [Description]`).
Validation: The code has been tested against the **Validation Suite** to ensure inventory cannot be overdrawn.

