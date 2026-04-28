# Member 2 Implementation Plan (kahi_h4x)

## 1. Member identity
- Name: kahi_h4x
- Member number: 2
- Role title: Search & Pricing Lead (Discovery)

## 2. Mission
Deliver the flight discovery experience and make the registry searchable by IATA codes and date windows. This role matters because all booking and check-in workflows begin with accurate search results.

## 3. Scope of ownership
- **Owned features/modules**
  - Flight registry storage and search logic.
  - `FlightQueryResult` consistency and failure handling.
  - Base pricing exposure in search results.
- **Likely files/directories**
  - `include/flight_manager.h`
  - `src/flight_manager.cpp`
- **Out of scope**
  - Inventory mutation, booking, check-in, and security validation rules (beyond using validators).

## 4. Project context
- **Fit in system**: First functional module in the pipeline after REPL; feeds booking flow.
- **Depends on**: Member 1 for `Status`/`Money`; Member 5 for IATA validation utilities.
- **Downstream dependents**: Member 3 uses flight registry for bookings; Member 6 uses base pricing.

## 5. Detailed implementation tasks
**Phase 1: Registry setup**
1. Define in-memory container (e.g., `std::vector<Flight>`) for registry state.
2. Implement read-only accessors for other modules.

**Phase 2: Search criteria handling**
1. Validate IATA origin/destination via validator utilities.
2. Apply sequential filtering: origin → destination → date window.

**Phase 3: Result normalization**
1. Populate `FlightQueryResult` with matches and a success `Status`.
2. If no matches, return success with empty results or a non-fatal status per spec.

**Phase 4: Pricing exposure**
1. Return `Flight.base_price` in query results without mutating inventory.
2. Coordinate with Member 6 on dynamic pricing hooks (read-only view).

**Phase 5: Testing & docs**
1. Add tests for valid searches, no-result searches, and invalid IATA inputs.
2. Ensure errors map to the error-model formatting.

## 6. Technical design expectations
- **Data models**: `Flight`, `SearchCriteria`, `FlightQueryResult`.
- **APIs**: `search_flights(const SearchCriteria&)`.
- **Error handling**: Return `Status` for invalid IATA or invalid date windows.
- **Performance**: Use const references and avoid copying flight records when possible.

## 7. Security and quality requirements
- Validate input via `validator.h` before search.
- Maintain strict uppercase IATA rules.
- No mutation of inventory in search paths.
- Compile with strict flags and zero warnings.

## 8. Definition of done
- Search returns deterministic results for matching and non-matching criteria.
- Invalid IATA results in a clear failure `Status`.
- No crashes or exceptions on empty registry.

## 9. Collaboration and handoff
- **Inputs needed**: Validator utilities (Member 5), Money contract (Member 1).
- **Outputs delivered**: Flight registry API and `search_flights` behavior.
- **Merge expectations**: Avoid changing `Flight` struct without notifying Member 3 and Member 6.

## 10. Risks and mitigation
- **Risk**: Date window filtering inconsistencies.  
  **Mitigation**: Use ISO-8601 time comparisons and clear test cases.
- **Risk**: Search returns invalid flights due to unsanitized IATA.  
  **Mitigation**: Enforce validation and fail fast on invalid codes.

