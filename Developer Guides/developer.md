
# Developer & Contributor Guide: Airline Management System

## 1. Purpose & Scope
This document provides the technical foundation for developers implementing, maintaining, or extending the Airline Management System. It ensures that all contributions align with our standards for safety, performance, and scalability in aviation software.

## 2. Environment & System Requirements
To maintain consistency across the development team, ensure your local environment meets these specifications:
__Compiler:__ C++17 or higher (GCC 9+, Clang 10+, or MSVC 19+).
__Build System:__ CMake 3.15+ (preferred for cross-platform compatibility).
__Database:__ SQLite or PostgreSQL (depending on the current branch requirements).
__Version Control:__ Git 2.25+.

__Initial Build Command:__
```bash
g++ -std=c++17 -Wall -Wextra -pedantic src/*.cpp -Iinclude -o airline_system
```

## 3. Core Architectural Principles
Developers must adhere to the following architectural constraints to prevent system regression:
__Separation of Concerns:__ Keep Business Logic (booking rules), Data Access (database queries), and UI/CLI components in distinct modules.
__Stateless Processing:__ Where possible, design flight-search and pricing logic to be stateless to allow for future horizontal scaling.
__Data Integrity:__ Never allow a "Partial Booking." All transaction-heavy operations must follow ACID properties (Atomicity, Consistency, Isolation, Durability).
__rchestration:__ Keep the `main()` function and high-level controllers focused solely on coordinating service calls rather than executing deep logic.

## 4. Coding & Style Standards
__Naming Conventions:__ Use `PascalCase` for Classes/Structs and `camelCase` for variables and functions.
__Resource Management:__ Use RAII (Resource Acquisition Is Initialization). Prefer `std::unique_ptr` and `std::shared_ptr` over raw pointers to prevent memory leaks in long-running system processes.
__Type Safety:__ Use `std::size_t` for all indexing and container sizes. Use `enum class` for flight statuses (e.g., `Delayed`, `OnTime`, `Cancelled`).
__Immutability:__ Mark member functions and variables as `const` by default unless mutation is strictly required.

## 5. Feature Implementation Workflow
Follow this 5-step pipeline for every new feature (e.g., adding a "Frequent Flyer" module):
1.  __Contract Definition:__ Define the module interface in a header file (`.h` or `.hpp`).
2.  __Logic Implementation:__ Develop the functional code in the corresponding source file (`.cpp`).
3.  __Compilation Check:__ Verify that the project compiles without warnings under strict flags.
4.  __Unit Testing:__ Run isolated tests for the new module (e.g., test seat allocation logic with zero remaining seats).
5.  __Documentation:__ Update the `README.md` and inline Doxygen comments to reflect API changes.

## 6. Implementation Order (Recommended)
When building out the system from scratch, follow this dependency-aware order:
* __Data Models:__ Define `Flight`, `Passenger`, and `Ticket` structures.
* __Persistence Layer:__ Implement file I/O or database connectors.
* __Search Engine:__ Implement flight filtering (by destination, date, price).
* __Booking Controller:__ Handle seat availability and ticket issuance.
* __Admin Tools:__ Implement flight scheduling and cancellation logic.

## 7. Error Handling & Validation
The system must be "Fail-Safe." A crash in the flight-search module should never crash the entire application.
* __Validation:__ All user inputs (dates, flight numbers) must be validated before reaching the logic layer.
* __Error Propagation:__ Use `std::optional` or custom `Result` types for expected failures (e.g., "Seat Taken") rather than throwing exceptions for routine logic.
* __Conciseness:__ Error messages should be clear, deterministic, and logged with a timestamp.

## 8. Pre-Merge Review Checklist
Before submitting a Pull Request to the `main` branch, verify the following:
* [ ] __Zero Warnings:__ Compiles cleanly under `-Wall -Wextra -pedantic`.
* [ ] __No Memory Leaks:__ Verified via Valgrind or AddressSanitizer.
* [ ] __Documentation Sync:__ Public headers and the Developer Guide are updated.
* [ ] __Boundary Respect:__ Logic is contained in the correct module (e.g., no UI code in the database class).
* [ ] __Test Coverage:__ All new logic paths are covered by a test case.

## 9. Common Pitfalls to Avoid
* __Eval-while-Parsing:__ Do not process ticket payments while still validating user input; separate the two stages.
* __Race Conditions:__ Avoid global variables for "Total Available Seats" to prevent double-booking during concurrent access.
* __Hardcoding:__ Never hardcode paths, API keys, or airline-specific pricing constants. Use a configuration file.

## 10. Future Refactoring Roadmap
* __Architecture:__ Move toward a clear `include/` and `src/` directory structure for better header visibility.
* __Testing:__ Migrate manual test scripts to a framework like GoogleTest or Catch2.
* __Observability:__ Add a debug mode to dump the current "System State" (active flights/memory usage) to a log file.

---