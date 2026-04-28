# Member 1 Implementation Plan (worku404)

## 1. Member identity
- Name: worku404
- Member number: 1
- Role title: Core Architect (Infrastructure & Safety)

## 2. Mission
Build the foundational types and safety contracts that every module depends on. This role matters because all booking, search, and check-in workflows rely on consistent Money arithmetic and Status reporting.

## 3. Scope of ownership
- **Owned features/modules**
  - `Money` fixed-point financial model and arithmetic helpers.
  - `Status` object contract and factory helpers.
  - Error-code traceability and logging conventions for failures.
- **Likely files/directories**
  - `include/common_types.h`
  - `src/common_types.cpp`
  - Shared error-code registry (location coordinated with team, default in `common_types.*`).
- **Out of scope**
  - Flight search, inventory updates, booking logic, boarding operations, and REPL command parsing.

## 4. Project context
- **Fit in system**: Provides the universal data and error-contract layer used by all service modules.
- **Depends on**: None for initial definitions; may align with Member 5 on sanitization/logging rules.
- **Downstream dependents**: Members 2–6 use `Money`, `Status`, and error-code conventions.

## 5. Detailed implementation tasks
**Phase 1: Contract definition**
1. Define `Status` fields (`success`, `error_code`, `message`) to match the repo’s error model.
2. Define `Money` with `long long amount_cents` and `std::string currency`.

**Phase 2: Safe arithmetic**
1. Implement `safe_add_money` and `safe_sub_money` helpers using overflow checks.
2. Enforce currency consistency checks (same ISO-4217 code) before arithmetic.

**Phase 3: Status factories**
1. Implement `make_success()` with empty message and `error_code` default.
2. Implement `make_failure(code, msg)` ensuring non-empty code/message.

**Phase 4: Traceability logging**
1. Add a minimal error-code registry or audit hook that guarantees unique error codes are logged.
2. Ensure logging sanitizes sensitive fields (coordinate with Member 5’s masking rules).

**Phase 5: Validation**
1. Add unit-level validation (or sample assertions) for overflow protection and error-code uniqueness.
2. Update developer documentation if the Status/Money contract changes.

## 6. Technical design expectations
- **Data models**: `Status`, `Money`.
- **Services/Utilities**: Status factories; Money arithmetic helpers.
- **Error handling**: Never throw exceptions; return `Status` and propagate failures immediately.
- **Logging/monitoring**: Centralized failure logging with error_code and module context (no PII).

## 7. Security and quality requirements
- Overflow checks on all Money arithmetic.
- Enforce non-empty error codes for every failure.
- No exceptions in baseline path.
- Compile cleanly under `-Wall -Wextra -pedantic`.

## 8. Definition of done
- `common_types.h/.cpp` compile without warnings.
- Money arithmetic fails safely on overflow and currency mismatch.
- `make_success`/`make_failure` used consistently by other modules.
- Error-code logging documented and verified with sample failures.

## 9. Collaboration and handoff
- **Inputs needed**: Error-code naming conventions from team; sanitization rules from Member 5.
- **Outputs delivered**: Stable `Status` and `Money` contracts; helper APIs; error-code logging spec.
- **Merge expectations**: Coordinate with all module owners before breaking changes to `common_types.h`.

## 10. Risks and mitigation
- **Risk**: Inconsistent error codes across modules.  
  **Mitigation**: Maintain a shared registry or documented code list.
- **Risk**: Overflow bugs in Money helpers.  
  **Mitigation**: Add overflow checks and simple test coverage.
- **Risk**: Logging leaks PII.  
  **Mitigation**: Centralize masking rules with Member 5 and enforce via helper API.

