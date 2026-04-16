## Documentation Hub: Airline Management System
This folder contains the full documentation set for building a modular Airline Management System (AMS) in C++17.

## What This Project Is
The Airline Management System is a backend simulation project designed to teach modular system design, data integrity, and strict error handling. It uses a decoupled service pipeline to manage the lifecycle of a flight reservation:

`flight_manager -> inventory_service -> reservation_engine -> boarding_controller`

This documentation serves as a blueprint for a team-based build, prioritizing engineering rigor and clear module boundaries over complex UI.

## Documentation Map
project-overview.md – High-level goals, core functionality, and project scope.

requirements.md – Functional (booking, searching) and non-functional (thread-safety, accuracy) requirements.

concepts.md – CS and domain concepts: IATA codes, PNR generation, and atomic inventory updates.

user-guidance.md – CLI command references and target behavior for the REPL.

technical.md – System architecture, state management, and data flow design.

api-reference.md – Public header contracts for each service module.

error-model.md – The Status object contract and domain-specific error codes (e.g., OVERBOOKED, INVALID_IATA).

testing.md – Verification matrix for seat allocation and check-in logic.

developer.md – Modern C++ coding standards and implementation guidance.

roadmap.md – A staged execution plan from basic registry to full boarding simulation.

README.md – Acts as the comprehensive entry point for the repository, defining the project identity , offering a quick-start guide, and mapping all documentation for stakeholders.

collaboration.md – Governs the human-to-human interface of the project by establishing branching strategies, peer review protocols, and conflict resolution frameworks for the development team.

glossary.md – Standardizes terminology across the project to prevent linguistic drift between modules by defining specific aviation, technical, and financial concepts.

functionality.md – Provides a detailed specification of the system’s operational capabilities, including the booking lifecycle, inventory controls, and search filtering rules.

benchmarks.md – Offers empirical evidence of system performance through quantitative analysis of execution latency, resource consumption, and scalability projections.



## Feature Scope

Supported

Flight Discovery: Search by origin/destination IATA codes.

Reservation Lifecycle: - Holding seats in different classes (Economy, Business, First).

Generating unique 6-character PNR (Passenger Name Record) codes.

Seat Management: Real-time availability checks and inventory guards.

Check-in Operations: Assigning gates and boarding groups based on seat class.

Error Handling Examples
Validation Error: `Invalid IATA code: "LONDON" `(Expected 3-letter code).

Availability Error: `Flight ADD-102 is full in Business Class`.

Logic Error: Cannot check-in: `Reservation PNR-XYZ123 not found`.

System Error: `Database locked during inventory update`.

Explicitly Deferred
Persistence (No SQL/NoSQL databases; in-memory only).

Real-world payment processing (Simulated via status objects).

Multi-city "layover" routing.

Graphical User Interface (GUI).

# Architecture
# Pipeline Data Flow

```User Input (REPL)``
       |
       v
``[Flight Manager]  <-->  [Inventory Service]``
       |                      |
       v                      v
``[Reservation Engine] --> [Boarding Controller]``
       |                      |
       v                      v
  ``` PNR Record           Boarding Pass```
  