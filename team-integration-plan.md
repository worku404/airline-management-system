# Team Integration Plan: Airline Management System

## Overall delivery phases
1. **Foundation & contracts**
   - Finalize `Status` and `Money` contracts.
   - Establish error-code registry and logging conventions.
2. **Search & registry**
   - Implement flight registry and search filtering.
   - Validate IATA codes via shared validators.
3. **Inventory & booking**
   - Implement inventory guards and atomic booking workflow.
   - Introduce PNR generation and reservation registry.
4. **Operations**
   - Implement check-in, boarding pass synthesis, and flight status updates.
5. **Security & compliance**
   - Integrate validators across search, booking, and boarding paths.
   - Apply masking to all error logs and diagnostics.
6. **Revenue & analytics**
   - Add dynamic pricing hooks and financial audits.
   - Generate operational reports from check-in metrics.

## Integration order
1. `common_types` (Member 1)
2. `validator/security_utils` (Member 5) + `flight_manager` (Member 2)
3. `inventory_service` + `reservation_engine` (Member 3)
4. `boarding_controller` (Member 4)
5. `revenue_service` + `report_generator` (Member 6)

## Dependency map
- Member 1 → foundational for all modules.
- Member 5 → required by Members 2, 3, 4 for validation.
- Member 2 → provides search/registry for Member 3 and pricing inputs for Member 6.
- Member 3 → required by Members 4 and 6.
- Member 4 → supplies operational metrics for Member 6.

## Shared coding standards
- **Language**: C++17 only.
- **Error handling**: Status-based returns; no exceptions in baseline path.
- **Data integrity**: All currency in `long long amount_cents`.
- **Style**: PascalCase types, camelCase functions/variables, const-correctness.
- **Memory**: RAII; prefer `std::unique_ptr`.
- **State**: In-memory, no persistence in baseline.

## Branch strategy
- Feature branches named `[member]/[feature]`.
- Atomic commits with `feat:`, `fix:`, `docs:` prefixes.
- Rebase or squash before merge.

## Review flow
- At least one peer approval per PR.
- Header/interface changes require explicit review by affected owners.
- Validate error-model formatting before approval.

## Testing strategy
- Compile with `-Wall -Wextra -pedantic` under C++17.
- Validate search, booking, and check-in scenarios from the test matrix.
- Check negative paths: invalid IATA, overbooking, invalid PNR.
- Confirm REPL resilience after failures (no crash, state consistent).

## Final integration checklist
- [ ] `common_types` adopted across all modules.
- [ ] Validators wired into search, booking, and check-in.
- [ ] Inventory never underflows; booking rollback works.
- [ ] PNR generation unique and collision-checked.
- [ ] Boarding pass generation uses seat class and valid flight status.
- [ ] Dynamic pricing uses inventory without side effects.
- [ ] Revenue audit detects mismatches and returns failure Status.
- [ ] All modules compile with zero warnings.
- [ ] Documentation updated to match final behavior.

