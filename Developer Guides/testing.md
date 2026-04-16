
---

# Validation & Testing Strategy: Airline Management System

## 1. Quality Assurance Objectives & Philosophies
The primary mission of the __Testing Strategy__ is to provide a "Hard Guard" against data corruption, ensuring that the __Airline Management System__ operates with mathematical certainty.
* __Deterministic Failure Flow__: Validating that every module correctly propagates a `Status{success=false}` rather than crashing or returning silent, stale data.
* __Atomic Transaction Verification__: Ensuring that a passenger __Reservation__ is never finalized unless a corresponding seat increment has been successfully committed in the __Inventory Service__.
* __Systemic Resilience__: Confirming that the __REPL Controller__ and core services can recover gracefully from malformed input or edge-case failures without terminating the application session.

---

## 2. Hierarchical Verification Stages
The system follows a __Six-Stage Validation Lifecycle__, ensuring that each architectural layer is verified before the next dependency is integrated.

### __Stage 1: Build & Initialization Infrastructure__
* __Compiler Rigor__: Confirms the project compiles under strict standards (`-Wall -Wextra -pedantic`) using __C++17__ toolchains.
* __Session Orchestration__: Validates that the __REPL__ initializes the flight registry correctly and responds to administrative commands like `:help` and `:quit`.

### __Stage 2: Lexical Analysis & Data Sanitization (Tokenizer)__
* __IATA Compliance__: Tests the scanner’s ability to identify valid __3-letter airport codes__ (e.g., "ADD", "DXB") while rejecting illegal characters or malformed strings.
* __Numeric Boundary Checks__: Verifies that monetary values and seat counts are scanned without __Integer Overflow__, returning a precise error position upon failure.

### __Stage 3: Syntactic Validity & AST Construction (Parser)__
* __Grammar Enforcement__: Validates that the __Recursive Descent Parser__ correctly structures queries and booking requests according to formal rules.
* __Structural Integrity__: Ensures that complex, nested inputs (such as filtered flight searches) build a logically sound __Abstract Syntax Tree (AST)__.

### __Stage 4: Execution Accuracy & Runtime Safety (Evaluator)__
* __Financial Precision__: Exercises the __Money Struct__ to confirm that 64-bit integer arithmetic remains __100% accurate__ across high-volume transactions.
* __Exception Mitigation__: Specifically targets runtime hazards, such as ensuring that an attempt to book a flight with zero available seats triggers the correct __Inventory Failure Status__.

### __Stage 5: Error Normalization & Propagation__
* __Diagnostic Consistency__: Audits all service modules to ensure that error messages and `error_code` strings follow the standardized __Status Model__.
* __Recovery Verification__: Guarantees that the system state remains consistent and the __REPL__ loop continues after a "Blocked" transaction (e.g., an unauthorized check-in attempt).

### __Stage 6: Advanced Feature Integration__
* __Ergonomic Verification__: Tests that helper overloads and inline predicates do not introduce regressions into the core processing pipeline.
* __Observability Audits__: Confirms that optional __Debug Toggles__ (such as AST dumping) provide accurate internal state visualizations for developers.

---

## 3. Core Functional Test Matrix (Target Scenarios)

### __Positive Validation ("The Happy Path")__
* __Search Execution__: `search ADD DXB` $\rightarrow$ System returns a populated `FlightQueryResult` with valid `Flight` objects.
* __Atomic Booking__: `book FL-101 Economy` $\rightarrow$ System generates a __6-character PNR__, creates a `BookingRecord`, and updates the `SeatMap`.
* __Check-In Lifecycle__: `checkin PNR-A1B2C3` $\rightarrow$ System synthesizes a `BoardingPass` with gate and boarding group data.

### __Negative Validation ("The Error Path")__
* __Invalid IATA Query__: `search AD123 DXB` $\rightarrow$ Triggers a __Tokenizer Failure__ (Invalid Character at Pos 2).
* __Inventory Depletion__: Booking the 11th seat on a 10-seat flight $\rightarrow$ Triggers an __Inventory Service Failure__ (`INV_FULL`).
* __Authentication Barrier__: `checkin PNR-999999` $\rightarrow$ Triggers a __Validation Failure__ (PNR Not Found).

---

## 4. Contract & Invariant Tests
To maintain the __Modular Decomposition__ of the project, each function is subjected to strict "Contract Audits":
* __Status Binary Check__: Every failure must result in `status.success == false`.
* __Message Non-Nullity__: Every failed operation must provide a human-readable `message` and a machine-readable `error_code`.
* __Position Precision__: When a failure occurs during input processing, the reported `position` must match the exact index of the offending token.

---

## 5. Automation & Regression Protocol
As the project moves toward a production-ready state, the manual testing script will be integrated into an __Automated Build Pipeline__:
* __Unit Testing Suite__: Implementation of __Catch2__ or __GoogleTest__ for isolating the __Reservation Engine__ and __Inventory Service__.
* __Integration Matrix__: Testing across a compiler matrix (__g++__ and __clang++__) to ensure standard-compliant, portable C++17 behavior.
* __Regression Focus__: Continuous monitoring of __PNR Generation__ and __Financial Accumulation__ to ensure that new feature branches do not compromise existing mission-critical logic.



---

__Certification Note:__ The system is only considered __Portfolio-Ready__ when the final verification suite demonstrates a 100% pass rate across the __Core Functional Test Matrix__ and adheres to all __System Invariants__.