# Member 5 Implementation Plan (username_telegramplus)

## 1. Member identity
- Name: username_telegramplus
- Member number: 5
- Role title: Security & Compliance Officer (Gatekeeper)

## 2. Mission
Protect the system from bad input and sensitive data leakage by enforcing strict validation and secure logging. This role matters because it shields all modules from malformed requests and privacy violations.

## 3. Scope of ownership
- **Owned features/modules**
  - Input validation for IATA codes, passports, PNRs, and baggage counts.
  - Access-control checks for check-in eligibility.
  - Sanitized error logging and traceability rules.
- **Likely files/directories**
  - `include/validator.h`
  - `src/validator.cpp`
  - `include/security_utils.h`
- **Out of scope**
  - Core booking logic, pricing calculations, and flight registry internals.

## 4. Project context
- **Fit in system**: Cross-cutting security layer used by all modules.
- **Depends on**: Member 1 for `Status` contracts and error-code conventions.
- **Downstream dependents**: Members 2–4 consume validator utilities; Member 6 consumes audit outputs.

## 5. Detailed implementation tasks
**Phase 1: Validation utilities**
1. Implement `is_valid_iata(code)` (exactly 3 uppercase letters).
2. Implement `is_valid_passport(passport_number)` per agreed format.
3. Implement `is_valid_pnr(pnr_id)` (6-character alphanumeric).

**Phase 2: Access control**
1. Implement `can_check_in(pnr_id, reservation_state)` with clear failure `Status`.
2. Ensure revoked or missing reservations return deterministic errors.

**Phase 3: Secure logging**
1. Implement masking utilities (e.g., partially redact passport numbers).
2. Ensure all logged errors use masked identifiers and required error codes.

**Phase 4: Integration hooks**
1. Provide lightweight helpers for Member 2 (IATA), Member 3 (passenger/seat), Member 4 (PNR/baggage).

**Phase 5: Testing**
1. Add tests for malformed inputs and masking behavior.

## 6. Technical design expectations
- **Utilities**: Pure functions in `validator` and `security_utils` without module side effects.
- **Error handling**: Return `Status` or boolean plus `Status` for validation failures.
- **Integrations**: No direct mutation of other module state.

## 7. Security and quality requirements
- Reject any non-uppercase IATA codes.
- Mask sensitive data in logs and errors.
- Avoid logging full passport numbers or passenger names.
- Compile under strict warnings with no exceptions.

## 8. Definition of done
- Validation helpers available to all modules and used in critical flows.
- Access-control logic prevents unauthorized check-in attempts.
- Logs are sanitized and traceable via error codes.

## 9. Collaboration and handoff
- **Inputs needed**: Error-code registry from Member 1; reservation state definitions from Member 3.
- **Outputs delivered**: Validator API, masking utilities, access-control checks.
- **Merge expectations**: Communicate any validation rule change to all module owners.

## 10. Risks and mitigation
- **Risk**: Overly strict validation blocks legitimate input.  
  **Mitigation**: Document format rules and align with domain-logic spec.
- **Risk**: PII leakage via logs.  
  **Mitigation**: Centralized masking utilities and mandatory use in error reporting.

