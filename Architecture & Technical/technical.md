
---

# Technical Design Specification: Airline Management System

## 1. System Architecture Overview
The __Airline Management System__ is engineered as a high-integrity, modular backend utilizing a __Layered Service Architecture__. Unlike traditional monolithic CLI applications, this system separates user interaction, business logic, and low-level resource guarding into distinct, decoupled domains.


### 1.1 The Operational Pipeline
Every request, whether a flight search or a seat reservation, flows through a deterministic execution pipeline:
1.  __Ingestion Layer__: The __REPL Controller__ captures raw terminal input.
2.  __Orchestration Layer__: The __Controller__ delegates the request to the appropriate service (e.g., __Flight Manager__ or __Reservation Engine__).
3.  __Domain Logic Layer__: Services process the business rules (e.g., checking if a PNR exists or if a date window is valid).
4.  __Persistence & Guard Layer__: The __Inventory Service__ performs atomic checks to prevent overbooking.
5.  __Response Synthesis__: The system wraps the result in a __Status Object__ and returns it to the user.

---

## 2. Core Architectural Principles

### 2.1 Modular Decomposition
The system is partitioned into five primary modules to ensure __Separation of Concerns__:
* __Flight Manager__: Owns the registry of flight schedules and IATA-based search logic.
* __Reservation Engine__: Manages the lifecycle of a booking and the generation of unique confirm codes (PNR).
* __Inventory Service__: Acts as the "Physical Guard," tracking real-time seat occupancy.
* __Boarding Controller__: Handles day-of-flight operations like gate assignment and check-in.
* __REPL Controller__: Manages the user session and command-line interface.

### 2.2 Explicit Contract Modeling
To achieve aviation-grade reliability, the system abandons implicit behaviors in favor of __Explicit Contracts__:
* __No Exceptions__: All modules return a `Status` or `Result` object. This forces the caller to acknowledge potential failures (e.g., "Flight Full") immediately.
* __Immutable States__: Where possible, data structures are passed by `const` reference to prevent accidental side effects during search or check-in processes.

---

## 3. Data Integrity & Financial Modeling

### 3.1 The "Money" Contract
A critical requirement is __100% financial accuracy__. To avoid the precision loss common in floating-point math, the system implements a fixed-point model.

```cpp 
struct Money {
    long long amount_cents; // 12500 represents $125.00
    std::string currency;   // ISO 4217 standard
};
```
* __Constraint__: All currency calculations must occur within the `long long` domain.
* __Benefit__: Eliminates rounding errors during price scaling (e.g., adding taxes or dynamic pricing surcharges).

### 3.2 Atomic Inventory Updates
To prevent the "Double Booking" problem, the __Inventory Service__ and __Reservation Engine__ are linked by an atomic update contract:
* A __PNR__ cannot be generated unless the `update_inventory` status returns `success: true`.
* If the seat increment fails, the entire booking request is rolled back.


---

## 4. Module Specifications & API Design

### 4.1 Flight Registry & Search
The __Flight Manager__ utilizes a registry pattern to store `Flight` objects. 
* __Search Logic__: Filters are applied sequentially (Origin $\rightarrow$ Destination $\rightarrow$ Date Window).
* __Performance__: Queries return a `FlightQueryResult` which includes a vector of matches and a `Status` object.

### 4.2 Reservation & PNR Lifecycle
The __Reservation Engine__ transforms a `BookingRequest` into a permanent `BookingResult`.
* __PNR Generation__: CONFIRMATION codes are 6-character unique identifiers.
* __Class Validation__: The engine checks `SeatClass` (Economy, Business, First) against the aircraft's physical configuration.

### 4.3 Boarding & Day-of-Flight
The __Boarding Controller__ handles the final stage of the passenger journey.
* __Check-in__: Validates the `pnr_id` and assigns a `boarding_group`.
* __Status Sync__: Updates the flight status (e.g., "Boarding," "Delayed") across the global registry.

---

## 5. Error & Diagnostic Model
The __Status Object__ is the backbone of the system's diagnostic capability.

| Field | Purpose | Example |
| :--- | :--- | :--- |
| `success` | Binary result of the operation. | `false` |
| `error_code` | Unique string for programmatic handling. | `"INV_SEAT_OCCUPIED"` |
| `message` | Human-readable explanation for the user. | `"Seat 12A is already booked."` |

__Propagation Protocol:__
1.  Service detects an issue (e.g., `check_availability` fails).
2.  Service returns `make_failure("CODE", "MSG")`.
3.  Controller receives the failure and halts the pipeline.
4.  User is notified of the specific error context.

---

## 6. Development Workflow & File Layout
The project follows a __Stage-Based Implementation__ to ensure each module is verified before the next layer is built.

### 6.1 Planned File Structure
```text
include/
  common_types.h        // Money, Status, and shared Constants
  flight_manager.h      // Search and Registry interfaces
  inventory_service.h   // Real-time seat tracking guards
  reservation_engine.h  // Booking and PNR logic
  boarding_controller.h // Check-in and gate management
  controller.h          // REPL and system orchestration
src/
  *.cpp                 // Respective implementations
  main.cpp              // Minimal entry point
```

---

## 7. Quality Assurance & System Invariants
To maintain "Portfolio-Ready" status, the following invariants are enforced via a __Validation Suite__:
* __Zero Overbooking__: The inventory delta must never result in negative seat counts.
* __Financial Consistency__: Total revenue must perfectly match the sum of all confirmed `total_cost` values.
* __REPL Resilience__: The system must recover from every possible `Status{success=false}` without terminating the program.


---

## 8. Extension Roadmap (Phase 6+)
Once the baseline modular system is stable, the following enhancements are planned:
* __Persistence Layer__: Integration of a file-based or SQLite storage system to save flight state between sessions.
* __Dynamic Pricing__: A revenue management module that adjusts `base_price` based on seat scarcity.
* __Audit Logs__: A centralized logging service that records every PNR transaction and inventory shift.