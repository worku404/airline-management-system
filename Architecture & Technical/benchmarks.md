
---

# Concurrency & Atomicity Strategy: Airline Management System

## 1. The Architectural Imperative: State Integrity
In the domain of aviation software, the primary technical risk is the __Race Condition__. This occurs when multiple independent service calls attempt to modify the same shared resource—specifically the __Flight Inventory__—within the same execution window. Without a robust atomicity strategy, the system risks the "Double-Booking" phenomenon, where two separate __PNRs (Passenger Name Records)__ are issued for a single physical asset. 

This strategy ensures that the system transitions between valid states only, maintaining a strict __Single Source of Truth__ across all decoupled modules.



## 2. The Atomic Commitment Pipeline
To ensure __Full Data Consistency__, the system treats every booking and check-in request as an __Atomic Unit of Work__. This means the operation either completes in its entirety or fails with zero residual side effects on the inventory or financial registers.

### __The Commitment Sequence:__
1.  __Inventory Probing (Read-Lock Simulation)__: The `reservation_engine` initiates a request to the `inventory_service` to verify that the specific `SeatClass` (Economy, Business, or First) has a strictly positive balance.
2.  __Physical Deduction (The Write Guard)__: The `inventory_service` decrements the available seat count. This acts as a __Soft Lock__, effectively reserving the resource while the rest of the metadata (passenger names, payment simulation) is processed.
3.  __Financial Validation (The Money Check)__: The system verifies the `Money` object, ensuring the `amount_cents` is mathematically sound and consistent with the flight’s base price and dynamic scaling factors.
4.  __Entity Synthesis (PNR Generation)__: Once inventory and finance are secured, the system generates the unique __6-character PNR__. This is the "Commit Point"—once the PNR is generated, the transaction is functionally irreversible without a formal cancellation.
5.  __Registry Commitment__: The final `BookingResult` is pushed to the global reservation ledger, and the __Status Object__ returns a success flag to the __REPL Controller__.



---

## 3. Compensating Transactions & Rollback Logic
A critical component of this strategy is __Fault Tolerance__. If the pipeline encounters a failure at any stage after the inventory has been deducted (e.g., a PNR collision or a malformed passenger input), the system must execute a __Compensating Transaction__ to restore inventory balance.

* __The Reversion Protocol__: If the `reservation_engine` fails during PNR generation, it triggers an immediate `update_inventory` call with a positive delta. This "unlocks" the seat, returning it to the available pool for other users.
* __Non-Persistent Leak Mitigation__: Since the baseline system is memory-resident, the __Controller__ is designed to catch unexpected module failures and ensure that no "Ghost Seats" (seats deducted but not assigned to a PNR) remain in the system state.

---

## 4. Modular Guarding Mechanisms

### __The Inventory Invariant (The "Floor" Rule)__
The `inventory_service` maintains a strict __Negative Floor Invariant__. It is mathematically impossible for a seat count to drop below zero.
* __Implementation__: Within the `update_inventory` logic, the service checks the `delta` against the current occupancy map. If the request would result in an underflow, the service returns `Error: INV_INSUFFICIENT_CAPACITY` and aborts the write operation entirely.

### __The PNR Collision Guard__
To ensure the __PNR__ remains a reliable primary key, the `reservation_engine` performs an existence check against the global map before commitment.
* __Deterministic Retries__: If a collision is detected, the system utilizes a high-entropy generator to produce a secondary ID, ensuring that the user experience is never interrupted by a randomized overlap in the identification space.

---

## 5. Thread Safety & Future Scaling
While the baseline implementation utilizes a synchronous __REPL loop__, the architecture is "Concurrency-Ready" for transition to multi-threaded environments.

* __Resource Isolation__: All flight and inventory data is encapsulated within dedicated service instances, preventing global variable "pollution."
* __Thread-Safe Gateways__: Future implementations of `std::mutex` or `std::shared_mutex` will be localized within the `inventory_service.cpp` and `reservation_engine.cpp` modules. This ensures that the rest of the application remains agnostic to the low-level locking mechanics.
* __Read-Write Separation__: The system is optimized to allow multiple __Read (Search)__ operations to occur simultaneously, only requiring an __Exclusive Write Lock__ during the millisecond-window of PNR commitment.



---

## 6. Summary of Concurrency Invariants
| Invariant | Description | Technical Owner |
| :--- | :--- | :--- |
| __Atomic Booking__ | No PNR is created without a prior inventory deduction. | Reservation Engine |
| __Negative Floor__ | Inventory counts can never underflow below zero. | Inventory Service |
| __PNR Uniqueness__ | No two active reservations share the same 6-char code. | Reservation Engine |
| __Financial Accuracy__ | Sum of PNR valuations must match Inventory revenue logs. | Revenue Engineer |

