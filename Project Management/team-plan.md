
---

# Team Ownership & Technical Responsibility Matrix

## __Member 1: Core Infrastructure & Safety Architect__
This member establishes the foundational "Rules of Engagement" for the entire system, ensuring that data is handled with mathematical precision and failures are never silent.
* __Financial Precision (The Money Model)__: Implements the `Money` structure utilizing `long long amount_cents` to ensure __100% accuracy__ in financial transactions. This architectural choice effectively eliminates the cumulative rounding errors inherent in floating-point mathematics.
* __Arithmetic Safety__: Develops robust helper functions for currency addition and subtraction that utilize __overflow protection__, ensuring that even high-volume booking data does not corrupt the system's financial state.
* __Uniform Status Reporting__: Designs the mandatory `Status` return type—a required contract for every function—which forces the system to __explicitly report__ whether an operation succeeded or failed.
* __Traceability Logic__: Creates a standardized factory for `make_failure` that attaches unique __Error Codes__ to every issue, allowing developers to pinpoint exactly where a transaction failed in the pipeline.

## __Member 2: Search & Discovery Lead__
This member manages the primary "Entry Point" of the system, transforming raw aeronautical data into a searchable, high-performance flight registry.
* __IATA Query Engine__: Implements the `search_flights` logic, which filters a repository of `Flight` objects by strictly matching __3-letter IATA codes__ (e.g., "ADD" or "DXB").
* __Temporal Filtering__: Develops the comparison logic to verify if `departure_time` and `arrival_time` fall within the specific __date windows__ requested by the user.
* __Registry Stability__: Maintains the `FlightQueryResult` structure to ensure the system returns a graceful __"No Flights Found"__ status rather than a system crash when search criteria are not met.

## __Member 3: Engine & Inventory Specialist__
This member owns the "Brain" of the system, ensuring that seat availability and booking records remain perfectly synchronized at all times.
* __Atomic Booking Lifecycle__: Implements the `create_booking` function, which guarantees __atomicity__: a passenger record is never finalized unless a seat is successfully "deducted" from the physical inventory.
* __Seat Map Integrity__: Manages a `std::map` of seat identifiers to boolean occupancy values, ensuring that no two passengers can be assigned the same __Seat Number__ simultaneously.
* __Class-Based Allocation__: Handles complex logic for distinct inventory pools (Economy, Business, and First Class), preventing a __"Business Class Full"__ state from erroneously affecting other seating tiers.
* __PNR Generation__: Develops the algorithm to produce unique __6-character Passenger Name Records (PNR)__, which serve as the primary key for the entire reservation lifecycle.

## __Member 4: Operations & Boarding Lead__
This member manages the "Day-of-Flight" workflow, handling the transition from a digital reservation to a physical boarding event.
* __PNR Validation Workflow__: Implements `process_check_in`, which cross-references the user's __PNR__ against the __Reservation Engine__ to verify passenger identity and current booking status.
* __Boarding Pass Synthesis__: Generates the `BoardingPass` object, dynamically assigning gate numbers and boarding groups based on the passenger's __Seat Class__ and real-time flight status.
* __Live Flight Status Updates__: Manages the `update_flight_status` function, allowing the airline to transition flights between __"On Time," "Delayed," or "Boarding"__ in real-time.

## __Member 5: Security & Compliance Officer__
This member acts as the "Gatekeeper," ensuring that only sanitized and authorized data enters the system's core services.
* __Input Sanitization Layer__: Writes strict validators for __Passport Numbers__ and __IATA Codes__, preventing malformed data from causing logic overflows in the search or booking modules.
* __Authorization Controls__: Implements logic to check if a passenger is eligible for a `CheckInResult` based on the current status of their __PNR__ and local airline security regulations.
* __Privacy-First Auditing__: Enhances the __Status System__ to log errors for internal developer review while ensuring that sensitive passenger data is __masked or removed__ from all public log files.

## __Member 6: Revenue & Analytics Engineer__
This member focuses on the "Business Intelligence" of the airline, using data to optimize pricing and track system performance.
* __Dynamic Pricing Algorithm__: Integrates with the __Inventory Service__ to implement logic that automatically increases the `base_price` of a flight as the available seat count becomes scarce.
* __Financial Audit Engine__: Periodically scans all `BookingResult` objects to ensure the sum of `total_cost` values perfectly matches the __Inventory Records__, verifying the __Long Long Integrity__ of the financial model.
* __Performance Metrics__: Extracts data from the __Boarding Controller__ to generate reports on efficiency, tracking the ratio of __Successful Check-ins__ versus total reservations per flight.

---