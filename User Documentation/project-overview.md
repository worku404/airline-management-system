## 1. Project Idea
Build a modular, backend-focused Airline Management System in C++17 that simulates flight scheduling, seat reservations, and passenger check-ins using explicit status objects and clear architectural boundaries.

## 2. What We Are Building
Flight Manager: A module to search and maintain a registry of available flights.

Reservation Engine: A service to process booking requests and generate unique Passenger Name Records (PNR).

Inventory Service: A low-level guard that prevents overbooking by tracking seat availability in real-time.

Boarding Controller: A day-of-flight operation module that handles check-ins and simulates boarding pass generation.

REPL Controller: A command-line interface to interact with the system (e.g., searching, booking, and checking status).

## 3. Core Functionality
Search & Registry: Query flights by IATA airport codes (e.g., ADD, DXB) and date ranges.

Booking Pipeline: Create reservations with passenger details and specific seat classes (Economy, Business, First).

Inventory Logic: Atomic updates to seat counts to ensure data consistency.

Check-in Logic: Assign boarding groups and gates once a PNR is confirmed.

Error Reporting: Explicit Status returns for every operation, detailing why a booking or check-in failed.

## 4. User Experience
Users interact via a command loop: search <origin> <dest> or book <flight_id>.

Successful operations return a result: Booking Confirmed: PNR-8293.

Failures provide clear context: Error: Inventory: Seat 12A is already occupied.

The system maintains state in-memory across the session.

## 5. Constraints
No Exceptions: All modules must return result objects containing a Status.

Data Integrity: Use long long for currency (cents) to avoid floating-point inaccuracies.

Separation of Concerns: Inventory logic must remain decoupled from the Boarding/Check-in logic.

Modern C++: Utilization of C++17 features like std::unique_ptr and std::optional.

## 6. Out of Scope (Baseline)
Persistent Database: All data is wiped when the program closes.

Real-time Payment Gateway: Financial transactions are simulated by checking if the "Money" struct is valid.

Graphical User Interface (GUI): Interaction is strictly via the REPL.

Multi-leg/Connecting Flights: Only direct point-to-point flights are supported.

## 7. Team Deliverables
API Reference: Full .h and .cpp documentation for all modules.

System Logic Map: A clear guide on how a request flows from the Controller to the Inventory.

Validation Suite: Manual or automated tests verifying that inventory cannot drop below zero.

Demo Script: A sequence of commands showing a search, a successful booking, and a failed overbooking attempt.