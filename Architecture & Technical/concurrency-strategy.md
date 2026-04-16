
---

# Concurrency & Atomicity Strategy: Airline Management System

## 1. The Architectural Challenge: Race Conditions
In a high-concurrency aviation environment, the primary technical risk is the __Race Condition__. This occurs when multiple independent service requests attempt to modify a shared resource—specifically the __Flight Inventory__—simultaneously. Without a robust atomicity strategy, the system risks overbooking a flight or generating a __PNR (Passenger Name Record)__ for a seat that no longer exists in the physical inventory.



## 2. The Atomic Transaction Pipeline
To achieve __Full Data Consistency__, the system implements a unidirectional commitment pipeline. Every booking request is treated as an __Atomic Unit of Work__; it either succeeds entirely across all modules or fails with zero side effects on the system state.

### __The Commitment Sequence:__
1.  __Inventory Probing (Pre-Lock)__: The `reservation_engine` queries the `inventory_service` to confirm that the specific `SeatClass` (Economy, Business, or First) has a strictly positive balance.
2.  __Physical Deduction (The Inventory Guard)__: The `inventory_service` decrements the seat count immediately. This acts as a __Soft Lock__, preventing other threads or requests from claiming that same unit during the remaining processing time.
3.  __Financial Validation (The Money Check)__: The system ensures the `Money` object is correctly initialized and that the `long long amount_cents` matches the dynamic pricing rules for that flight.
4.  __Entity Synthesis (PNR Generation)__: Once inventory and finance are secured, the system generates the unique __6-character PNR__. This is the "Point of No Return" for the transaction logic.
5.  __Registry Commitment__: The final `BookingResult` is written to the global reservation ledger, and the __Status Object__ returns `success: true`.



---

## 3. Compensating Transactions & Rollback Logic
A core principle of this strategy is __Fault Tolerance__. If the pipeline fails at any stage after the inventory has been deducted, the system must perform a __Compensating Transaction__ to restore the "Physical Truth" of the aircraft capacity.

* __The Reversion Protocol__: If the `reservation_engine` encounters an internal error (e.g., a PNR collision) after the seat has been taken, it must trigger an immediate `update_inventory` call with a positive delta to "give back" the seat.
* __Non-Persistent Leak Mitigation__: Since this is a memory-resident system, the __Controller__ ensures that any uncaught module failure results in an automatic state-audit to ensure the `SeatMap` accurately reflects the `ReservationRegistry`.

---

## 4. Module-Level Guarding Mechanisms

### __The Inventory Invariant (The "Floor" Rule)__
The `inventory_service` maintains a strict __Negative Floor Invariant__. It is mathematically impossible for a seat count to drop below zero.
* __Implementation__: The logic within `update_inventory` checks the `delta` against the current `occupied_seats` map. If the request would violate the floor, the service returns `Error: INV_INSUFFICIENT_CAPACITY` and aborts the write.

### __The PNR Collision Guard__
To ensure the __PNR__ remains a reliable primary key, the `reservation_engine` performs an existence check against the global map before commitment.
* __Deterministic Retries__: If a collision is detected, the system utilizes a high-entropy algorithm to generate a new ID, ensuring that the user never experiences a failure due to simple ID overlap.

---

## 5. Thread Safety & Future Scaling
While the baseline implementation utilizes a synchronous __REPL loop__, the architecture is "Concurrency-Ready" for future multi-threaded extensions.

* __Shared Resource Isolation__: All flight and inventory data is stored in dedicated service instances.
* __Thread-Safe Gateways__: Future integration of `std::mutex` or `std::shared_mutex` will be confined to the `inventory_service.cpp` implementation, meaning the rest of the team can continue building features without worrying about low-level locking logic.
* __Read-Write Separation__: The system is designed to allow multiple __Read (Search)__ operations simultaneously, only requiring a __Write (Exclusive)__ lock during the few milliseconds of PNR generation and inventory deduction.



---

## 6. Summary of Concurrency Invariants
| Invariant | Description | Module Owner |
| :--- | :--- | :--- |
| __Atomic Booking__ | No PNR exists without a corresponding seat deduction. | Reservation Engine |
| __Negative Floor__ | Seat counts never drop below zero. | Inventory Service |
| __PNR Uniqueness__ | No two reservations share a 6-character code. | Reservation Engine |
| __Financial Accuracy__ | Sum of PNR costs must match Inventory revenue logs. | Revenue Engineer |

_