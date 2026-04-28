# Member 3 Implementation Plan (zea_don_ay)

## 1. Member identity
- Name: zea_don_ay
- Member number: 3
- Role title: Engine & Inventory Specialist (Back-end Logic)

## 2. Mission
Ensure the reservation lifecycle is atomic: no booking is created without a corresponding inventory deduction. This role matters because it protects the system from overbooking and inconsistent state.

## 3. Scope of ownership
- **Owned features/modules**
  - Reservation engine and booking lifecycle.
  - Inventory guard and seat map integrity.
  - PNR generation and collision handling.
- **Likely files/directories**
  - `include/reservation_engine.h`
  - `src/reservation_engine.cpp`
  - `include/inventory_service.h`
  - `src/inventory_service.cpp`
- **Out of scope**
  - Search/filter logic, boarding pass creation, security validation logic (beyond using validators).

## 4. Project context
- **Fit in system**: Central transaction engine between search and boarding.
- **Depends on**: Member 1 for `Status`/`Money`; Member 2 for flight registry access; Member 5 for validation rules.
- **Downstream dependents**: Member 4 (check-in validation), Member 6 (revenue audits).

## 5. Detailed implementation tasks
**Phase 1: Inventory model**
1. Implement seat pools per `SeatClass` and per-flight `SeatMap`.
2. Implement `check_availability(flight_id, seat_class)` with strict floor rules.

**Phase 2: Inventory updates**
1. Implement `update_inventory` with atomic checks and non-negative constraints.
2. Return explicit failure `Status` on insufficient capacity or unknown flight.

**Phase 3: Booking lifecycle**
1. Implement `create_booking` to validate input and reserve inventory.
2. Generate a 6-character PNR after successful inventory deduction.
3. If PNR generation fails, rollback inventory with compensating update.

**Phase 4: Seat assignment**
1. Enforce unique `seat_number` within a flight via `SeatMap`.
2. Validate seat class compatibility and seat availability.

**Phase 5: Registry integration**
1. Store `BookingResult` in a reservation registry keyed by PNR.
2. Expose lookup methods for check-in workflows.

## 6. Technical design expectations
- **Data models**: `BookingRequest`, `BookingResult`, `Passenger`, `SeatClass`, `SeatMap`.
- **APIs**: `create_booking`, `check_availability`, `update_inventory`.
- **PNR generation**: High-entropy 6-char alphanumeric with collision checks.
- **Error handling**: Use `Status` for all failure paths; no partial bookings.

## 7. Security and quality requirements
- Validate `flight_id`, `seat_number`, and passenger fields via validator utilities.
- Enforce atomicity and rollback on any failure after inventory change.
- No inventory underflow, ever.

## 8. Definition of done
- Booking fails gracefully on full inventory with correct error codes.
- Successful booking yields unique PNR and updates inventory.
- Seat numbers never duplicate in a flight.
- Registry lookup supports check-in validation.

## 9. Collaboration and handoff
- **Inputs needed**: Flight registry access from Member 2; validation rules from Member 5.
- **Outputs delivered**: Booking/inventory APIs and reservation registry interfaces.
- **Merge expectations**: Coordinate with Member 4 for PNR lookup semantics.

## 10. Risks and mitigation
- **Risk**: PNR collisions or non-deterministic failures.  
  **Mitigation**: Collision checks and deterministic retry strategy.
- **Risk**: Inventory and booking state divergence.  
  **Mitigation**: Strict atomic update sequence with rollback on failure.

