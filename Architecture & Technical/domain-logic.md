
---

# Domain Logic & Business Rules Specification

## 1. Aeronautical Identifiers & Standards
To ensure interoperability with global aviation standards, the system enforces strict formatting for all geographical and logistical identifiers.

### __IATA Airport Code Constraints__
All origin and destination locations must conform to the __International Air Transport Association (IATA)__ 3-letter standard.
* __Validation Rule__: Must be exactly three characters, consisting only of uppercase alphabetic characters `[A-Z]`.
* __Business Logic__: The system treats "ADD" and "add" as distinct lexical units during the tokenization phase to enforce strict data entry standards.

### __Flight Identifier (Flight-ID) Schema__
Every flight in the registry is assigned a unique alphanumeric string.
* __Structure__: Typically follows the pattern `[Carrier Code]-[Numeric Sequence]` (e.g., `ET-901`).
* __Invariant__: The __Flight-ID__ serves as the immutable primary key for all inventory and search operations.

---

## 2. Passenger Name Record (PNR) Architecture
The __PNR__ is the critical digital contract between the airline and the passenger. It acts as the primary key for the entire __Reservation Lifecycle__.



### __PNR Generation Logic__
* __Format__: A 6-character alphanumeric string generated upon a successful booking commitment.
* __Entropy Requirement__: The generation algorithm must ensure high entropy to prevent "brute-force" discovery of passenger itineraries.
* __Lifecycle States__:
    1.  __Reserved__: Seat is deducted from inventory; PNR is generated.
    2.  __Checked-In__: Passenger identity is verified; boarding pass is synthesized.
    3.  __Boarded__: Final stage of the operational journey.

---

## 3. Aircraft Configuration & Inventory Logic
The system manages seat availability as a "Physical Asset" governed by the __Inventory Service__. Unlike retail inventory, airline seats are perishable assets with distinct class hierarchies.

### __Seat Class Stratification__
The system supports three distinct tiers of service, each with independent inventory pools:
* __Economy Class__: High-density seating with base-tier pricing models.
* __Business Class__: Mid-density seating with priority boarding pass synthesis.
* __First Class__: Low-density seating with premium `Money` valuation and maximum baggage allowances.

### __The "Physical Guard" Invariant__
The __Inventory Service__ enforces a strict mathematical floor.
* __Rule__: Occupied seats for a specific class can never exceed the `Total Capacity` defined in the aircraft configuration.
* __Atomic Linkage__: A PNR cannot be finalized unless the __Inventory Service__ successfully executes a decrement operation. This prevents the "Ghost Booking" phenomenon.

---

## 4. Financial Modeling & Currency Precision
To maintain "Aviation-Grade" financial integrity, the system rejects standard floating-point types (which suffer from rounding errors) in favor of a __Fixed-Point Monetary Model__.

### __The Money Struct Contract__
```cpp
struct Money {
    long long amount_cents; 
    std::string currency;   
};
```
* __Arithmetic Axiom__: All calculations—including dynamic pricing surcharges, taxes, and refunds—must be performed using __Integer Arithmetic__ on the `amount_cents` field.
* __Precision Benefit__: This ensures that a transaction of $10,000,000.00 is tracked with the same cent-perfect accuracy as a $1.00 fee.

---

## 5. Temporal Constraints & Scheduling
Flight scheduling is governed by __ISO-8601__ compatible timestamps (`std::time_t`) to ensure absolute precision in flight duration and date window filtering.

* __The Temporal Invariant__: `arrival_time` must always be strictly greater than `departure_time`. Any flight record violating this rule is rejected by the __Flight Manager__ during initialization.
* __Date Window Filtering__: When a user performs a `search`, the system applies a __Range Comparison__ against the departure timestamp to identify valid flights within the requested window.



---

## 6. Operational Workflow Invariants
The system enforces a specific __State Machine__ for all passenger interactions:
1.  __Search Phase__: Read-only access to the flight registry.
2.  __Commit Phase__: Atomic update to inventory and creation of the PNR.
3.  __Verification Phase__: Validation of the PNR to transition the passenger to a "Check-In" state.
* __Strict Sequence__: A passenger cannot transition to __Check-In__ without first satisfying the __Commit Phase__ requirements.

---

