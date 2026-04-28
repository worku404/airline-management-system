# Member 4 Implementation Plan (eloicry)

## 1. Member identity
- Name: eloicry
- Member number: 4
- Role title: Operations & Boarding Lead (Execution)

## 2. Mission
Deliver day-of-flight operations: validate PNRs, produce boarding passes, and manage flight status updates. This role matters because it closes the passenger lifecycle and ensures operational correctness.

## 3. Scope of ownership
- **Owned features/modules**
  - Check-in workflow and boarding pass generation.
  - Flight status updates (On Time, Delayed, Boarding).
  - Baggage count handling in check-in results.
- **Likely files/directories**
  - `include/boarding_controller.h`
  - `src/boarding_controller.cpp`
- **Out of scope**
  - Pricing, inventory updates, search logic, and validation utilities (beyond using them).

## 4. Project context
- **Fit in system**: Final stage of pipeline after reservation engine.
- **Depends on**: Member 3 for reservation registry and PNR lookup; Member 5 for validation rules.
- **Downstream dependents**: Member 6 for performance reporting.

## 5. Detailed implementation tasks
**Phase 1: Check-in validation**
1. Implement `process_check_in(pnr_id, baggage_count)` with PNR lookup.
2. Enforce valid PNR format and ensure reservation state is eligible.

**Phase 2: Boarding pass synthesis**
1. Generate `BoardingPass` with gate assignment and boarding group.
2. Use seat class to determine boarding group priority.

**Phase 3: Flight status management**
1. Implement `update_flight_status(flight_id, new_status)` with valid state transitions.
2. Expose status to reports and REPL output.

**Phase 4: Baggage integration**
1. Validate `baggage_count` bounds and record in check-in result.

**Phase 5: Testing**
1. Add tests for valid check-in, missing PNR, and invalid transitions.

## 6. Technical design expectations
- **Data models**: `BoardingPass`, `CheckInResult`.
- **APIs**: `process_check_in`, `update_flight_status`.
- **Integrations**: Reservation registry lookup; validation utilities.
- **Error handling**: Use `Status` and return explicit errors for invalid PNR or state.

## 7. Security and quality requirements
- Validate PNR format before lookup.
- Do not expose passenger PII in boarding responses or logs.
- Ensure check-in is idempotent or returns consistent failure on duplicates.

## 8. Definition of done
- Successful check-in returns a boarding pass with gate and group.
- Invalid PNR or ineligible status returns a clear failure `Status`.
- Flight status updates propagate without breaking search/booking flows.

## 9. Collaboration and handoff
- **Inputs needed**: Reservation registry interface from Member 3; validator rules from Member 5.
- **Outputs delivered**: Check-in behavior contract and flight status update API.
- **Merge expectations**: Coordinate status enums or strings with Member 2 and Member 6.

## 10. Risks and mitigation
- **Risk**: Inconsistent PNR state handling across modules.  
  **Mitigation**: Define a shared reservation state machine with Member 3.
- **Risk**: Invalid flight status transitions.  
  **Mitigation**: Enumerate allowed transitions and test them explicitly.

