## Airline Management System: General Functionality
The Airline Management System (AMS) is a modular, backend-focused simulation designed to handle the lifecycle of flight reservations. It prioritizes data integrity, separation of concerns, and robust error handling using explicit status objects rather than C++ exceptions.

## 1. System Pipeline & Data Flow
The project follows a decoupled service pipeline where data flows through specialized modules to ensure that a reservation is only finalized if all physical and financial constraints are met.

<Flight Discovery>: Users search for flights using IATA airport codes (e.g., ADD, DXB).

<Inventory Guard>: The system performs atomic updates to seat counts to prevent overbooking.

<Reservation Engine>: Upon successful validation, the system generates a unique 6-character Passenger Name Record (PNR).

<Day-of-Flight Operations>: The boarding controller processes check-ins and generates boarding passes based on the validated PNR.

## 2. Module Responsibilities
According to the Modular Programming Group Division, the system is split into six distinct areas of responsibility:

## Core Infrastructure

<Financial Math>: Uses long long cents to manage all currency transactions, avoiding floating-point rounding errors.


<Unified Error Model>: Every function returns a Status object containing a success boolean, a unique error code, and a descriptive message.

## Search & Pricing

<Registry Management>: Maintains a searchable database of flights with departure/arrival times and base prices.


<Dynamic Adjustments>: Prices can be adjusted dynamically based on real-time inventory levels.

## Transaction & Inventory Logic

<Atomic Bookings> : Ensures "Atomicity"—a booking cannot be created unless a seat is successfully deducted from the flight inventory.


<Seat Mapping>: Manages physical seat layouts (Economy, Business, First) to prevent double-booking.

## Operations & Security

<Check-In Workflow> : Validates PNRs, assigns boarding groups, and allocates gates.


<Input Validation>: A dedicated security layer sanitizes all user inputs (like passport numbers and IATA codes) before they reach the backend


## System Functionality & Member Responsibilities
The system is engineered as a decoupled backend pipeline where each module serves as a critical link in the reservation lifecycle.

## Member 1: Core Infrastructure & Safety Architect
This member builds the "rules of engagement" for the entire system, ensuring that data is handled with mathematical precision and errors are never silent.

``Financial Precision (The Money Model)``: Implements the Money struct using long long amount_cents to ensure 100% accuracy in financial transactions, effectively eliminating the rounding errors inherent in floating-point numbers.

``Arithmetic Safety``: Develops helper functions that manage currency addition and subtraction with overflow protection, ensuring that even high-volume bookings do not corrupt the financial state.

``Uniform Status Reporting``: Designs the Status object—a mandatory return type for every function—which forces the system to explicitly report whether an operation succeeded or failed.

``Traceability Logic``: Creates a standardized factory for make_failure that attaches unique error_code strings to every issue, allowing developers to pinpoint exactly where a booking failed in the pipeline.

## Member 2: Search & Discovery Lead
This member manages the "entry point" of the system, transforming raw flight data into a searchable, user-friendly registry.

``IATA Query Engine``: Implements the search_flights logic, which filters a database of Flight objects by strictly matching 3-letter IATA codes (e.g., "ADD" or "DXB").

``Temporal Filtering``: Writes the comparison logic to check if flight departure_time and arrival_time fall within the specific date windows requested by the user.

``Registry Stability``: Maintains the FlightQueryResult structure to ensure the system returns a graceful "No Flights Found" status rather than crashing when search criteria are not met.

## Member 3: Engine & Inventory Specialist
This member owns the "Brain" of the system, ensuring that seat availability and booking records are always perfectly synchronized.

``Atomic Booking Lifecycle``: Implements the create_booking function, which guarantees that a passenger record is never created unless a seat is successfully "deducted" from the physical inventory.

``Seat Map Integrity``: Manages a std::map of seat IDs to boolean occupancy values, ensuring that no two passengers can be assigned the same seat number simultaneously.

``Class-Based Allocation``: Handles logic for distinct inventory pools (Economy, Business, and First Class), preventing a "Business Class full" error from affecting other seating tiers.

``PNR Generation``: Develops the algorithm to produce unique 6-character Passenger Name Records (PNR), which serve as the primary key for the entire reservation lifecycle.

## Member 4: Operations & Boarding Lead
This member manages the "Day-of-Flight" workflow, handling everything from gate assignments to baggage tracking.

``PNR Validation Workflow``: Implements process_check_in, which cross-references the user's PNR against the Reservation Engine to verify identity and seat status.

``Boarding Pass Synthesis``: Generates the BoardingPass object, dynamically assigning gate numbers and boarding groups based on the passenger's seat class and flight status.

``Live Flight Status Updates``: Manages the update_flight_status function, allowing the airline to transition flights between "On Time," "Delayed," or "Boarding" in real-time.

## Member 5: Security & Compliance Officer
This member acts as the "Gatekeeper," ensuring that only valid and sanitized data enters the system's core services.

``Input Sanitization Layer``: Writes strict validators for passport_number and IATA codes, preventing malformed data from causing logic errors in the search or booking modules.

``Authorization Controls``: Implements logic to check if a passenger is eligible for a CheckInResult based on the current status of their PNR and local airline regulations.

``Privacy-First Auditing``: Enhances the Status system to log errors for developers while ensuring that sensitive passenger data is masked or removed from log files.

## Member 6: Revenue & Analytics Engineer
This member focuses on the "Business Intelligence" of the airline, using data to optimize pricing and track performance.

``Dynamic Pricing Algorithm``: Integrates with the inventory_service to implement logic that automatically increases the base_price of a flight as seats become scarce.

``Financial Audit Engine``: Periodically scans all BookingResult objects to ensure the sum of total_cost values perfectly matches the inventory records, verifying the "long long" integrity.

``Performance Metrics``: Uses data from the BoardingController to generate reports on boarding efficiency, tracking how many passengers successfully checked in per flight.