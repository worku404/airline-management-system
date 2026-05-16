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

Let me break down this airline management system in simple terms:

## What is an Airline Management System?

Think of it like a website where you can:
1. **Search for flights** ("I want to fly from New York to Los Angeles on May 20th")
2. **Book/reserve seats** (pick your flight and purchase a ticket)
3. **Check-in** (confirm you're arriving for your flight)

This document describes **Member 2's job** (kahi_h4x), who is responsible for the **Search & Pricing** part.

---

## What Does Member 2 Do? (The "Search & Pricing Lead")

### 🔍 **Their Main Mission:**
Make it easy for users to **find and discover flights** by:
- **Origin & Destination** (e.g., "flights from JFK to LAX")
- **Date** (e.g., "departing May 20 - May 27")
- **Show prices** (base ticket cost)

### Why is This Important?
**All booking and check-in workflows begin with accurate search results.** 
- Without a good search, users can't find flights
- Without finding flights, they can't book
- Without booking, the entire airline system falls apart

---

## Breaking Down Their Responsibilities

### **Phase 1: Registry Setup**
- **What**: Store all available flights in a database (in-memory list)
- **Why**: Need a place to store flight information before searching

### **Phase 2: Search Criteria Handling**
- **What**: When a user searches, validate their inputs:
  - Is "JFK" a real airport code? ✓
  - Is "XYZ" valid? ✗ (rejected)
  - Filter by origin → destination → date range
- **Why**: Prevent garbage data and ensure accurate results

### **Phase 3: Result Normalization**
- **What**: Return search results in a clean format
  - If flights found: show them
  - If no flights found: say "no results found" (not an error)

### **Phase 4: Pricing Exposure**
- **What**: Include the **base price** of each flight in results
- **Why**: Users want to see prices when searching

### **Phase 5: Testing & Documentation**
- **What**: Make sure everything works correctly
- **Why**: Prevent bugs that break the system

---

## Real-World Example

**User searches for flights:**
```
Origin: JFK (New York)
Destination: LAX (Los Angeles)  
Date: May 20, 2026
```

**Member 2's code does:**
1. ✓ Validates "JFK" is a real airport
2. ✓ Validates "LAX" is a real airport  
3. ✓ Validates May 20, 2026 is a valid date
4. 🔍 Searches the flight registry for matching flights
5. 📋 Returns results like:
   ```
   Flight AA100: JFK → LAX, May 20, 2:00 PM, $150
   Flight AA200: JFK → LAX, May 20, 5:30 PM, $120
   Flight UA300: JFK → LAX, May 20, 8:00 PM, $180
   ```

---

## How It Connects to Others

This member depends on:
- **Member 1**: Basic building blocks (Status codes, Money handling)
- **Member 5**: IATA code validation (airport code checking)

This member feeds into:
- **Member 3**: Uses these search results to let users book flights
- **Member 6**: Uses pricing info to apply special discounts

---

## Technical Jargon Explained

| Term | Meaning |
|------|---------|
| **IATA codes** | 3-letter airport codes (JFK, LAX, LHR) |
| **Flight registry** | Database of all available flights |
| **Search criteria** | What the user is looking for |
| **Flight query result** | The results returned to the user |
| **Base price** | The starting ticket price (before taxes/fees) |
| **Validation** | Checking if input data is correct |

---

## Summary

**Member 2 builds the "flight search engine"** — the thing that lets people find flights. Without it, you have an airline management system that users can't interact with!
