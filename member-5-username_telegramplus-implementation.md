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



Perfect! Let me explain the **validator module** in complete detail:

---

# **Validator Module - Complete Detailed Explanation**

## **What is the Validator Module?**

The validator module is a **data validation utility** that checks if user inputs meet specific format and business rules before they're used in the system.

**Think of it like a gatekeeper:** Before allowing data into the system, validate it to prevent errors, corruption, or security issues.

---

## **File Structure**

```
validator.h  (Header)       → Function declarations
validator.cpp (Implementation) → Function implementations
```

---

# **Detailed Function Breakdown**
=== # (DELETED) ===
## **Function 1: is_valid_iata() **

### **Purpose**
Validate IATA airport codes (3-letter codes like JFK, LAX, DXB)

### **Function Signature**
```cpp
bool is_valid_iata(const std::string& code);
```

### **What it does**
```
Input: "JFK"
  ↓
Check: Is it exactly 3 characters?
  ↓
Check: Are ALL characters UPPERCASE letters?
  ↓
Output: true (valid) or false (invalid)
```

### **Detailed Code Breakdown**

```cpp
bool is_valid_iata(const std::string& code) {
    // RULE 1: Must be exactly 3 characters
    if (code.size() != 3) {
        return false;  // "JFKK" (4 chars) → invalid
                       // "JF" (2 chars) → invalid
    }
    
    // RULE 2: Every character must be UPPERCASE letter
    for (char ch : code) {
        if (!std::isupper(static_cast<unsigned char>(ch))) {
            return false;  // "jfk" (lowercase) → invalid
                           // "J1K" (digit) → invalid
                           // "J-K" (special char) → invalid
        }
    }
    
    // If we got here, all rules passed
    return true;
}
```

### **Examples**

| Input | Check | Result |
|-------|-------|--------|
| "JFK" | 3 chars, all UPPERCASE | ✅ **Valid** |
| "LAX" | 3 chars, all UPPERCASE | ✅ **Valid** |
| "DXB" | 3 chars, all UPPERCASE | ✅ **Valid** |
| "jfk" | 3 chars, but lowercase | ❌ Invalid |
| "JFk" | 3 chars, mixed case | ❌ Invalid |
| "JF" | Only 2 chars | ❌ Invalid |
| "JFKA" | 4 chars | ❌ Invalid |
| "J1K" | Has digit | ❌ Invalid |
| "J-K" | Has special char | ❌ Invalid |

### **Real-World Usage in System**

```cpp
// In flight_manager.cpp during search
if (!is_valid_iata(criteria.origin)) {
    return error("Invalid origin airport code");
}

if (!is_valid_iata(criteria.destination)) {
    return error("Invalid destination airport code");
}
```

---
=== # (DELETED) ===
## **Function 2: is_valid_passport()**

### **Purpose**
Validate passport numbers used for passenger identification

### **Function Signature**
```cpp
bool is_valid_passport(const std::string& passport_number);
```

### **What it does**
```
Input: "AB123456"
  ↓
Check: Is length 6-9 characters?
  ↓
Check: Does it contain ONLY uppercase letters and digits?
  ↓
Output: true (valid) or false (invalid)
```

### **Detailed Code Breakdown**

```cpp
bool is_valid_passport(const std::string& passport_number) {
    // RULE 1: Length must be between 6-9 characters
    if (passport_number.size() < 6 || passport_number.size() > 9) {
        return false;  // "AB123" (5 chars) → too short
                       // "AB1234567890" (12 chars) → too long
    }
    
    // RULE 2: Every character must be UPPERCASE letter or digit
    // Helper function is_upper_alnum() checks this
    for (char ch : passport_number) {
        if (!is_upper_alnum(ch)) {
            return false;  // "ab123456" (lowercase) → invalid
                           // "AB-12345" (special char) → invalid
                           // "AB 12345" (space) → invalid
        }
    }
    
    return true;
}
```

### **Helper Function: is_upper_alnum()**

```cpp
bool is_upper_alnum(char ch) {
    // Check if character is digit OR uppercase letter
    return std::isdigit(static_cast<unsigned char>(ch)) ||
           (std::isalpha(static_cast<unsigned char>(ch)) && 
            std::isupper(static_cast<unsigned char>(ch)));
}
```

**Breakdown:**
```
is_upper_alnum('5')   → true  (is digit)
is_upper_alnum('A')   → true  (is uppercase letter)
is_upper_alnum('a')   → false (is lowercase letter)
is_upper_alnum('-')   → false (is special char)
is_upper_alnum(' ')   → false (is space)
```

### **Examples**

| Input | Length | Chars | Result |
|-------|--------|-------|--------|
| "AB123456" | 8 | A,B,1,2,3,4,5,6 | ✅ **Valid** |
| "ABC123" | 6 | A,B,C,1,2,3 | ✅ **Valid** |
| "P1234567A" | 9 | P,1,2,3,4,5,6,7,A | ✅ **Valid** |
| "AB12345" | 7 | A,B,1,2,3,4,5 | ✅ **Valid** |
| "AB123" | 5 | Too short | ❌ Invalid |
| "AB12345678" | 10 | Too long | ❌ Invalid |
| "ab123456" | 8 | Has lowercase | ❌ Invalid |
| "AB-12345" | 8 | Has dash | ❌ Invalid |
| "AB 12345" | 8 | Has space | ❌ Invalid |

### **Real-World Usage**

```cpp
// In reservation_engine.cpp when booking
if (!is_valid_passport(passenger.passport_number)) {
    return error("Invalid passport number");
}
```

---

## **Function 3: is_valid_pnr()**

### **Purpose**
Validate PNR (Passenger Name Record) - the booking reference code

### **Function Signature**
```cpp
bool is_valid_pnr(const std::string& pnr_id);
```

### **What it does**
```
Input: "ABC123"
  ↓
Check: Is it exactly 6 characters?
  ↓
Check: Does it contain ONLY uppercase letters and digits?
  ↓
Output: true (valid) or false (invalid)
```

### **Detailed Code Breakdown**

```cpp
bool is_valid_pnr(const std::string& pnr_id) {
    // RULE 1: Must be EXACTLY 6 characters
    if (pnr_id.size() != 6) {
        return false;  // "ABC12" (5 chars) → invalid
                       // "ABC1234" (7 chars) → invalid
    }
    
    // RULE 2: Each character must be uppercase letter OR digit
    for (char ch : pnr_id) {
        if (!is_upper_alnum(ch)) {
            return false;  // "abc123" (lowercase) → invalid
                           // "ABC-23" (special char) → invalid
        }
    }
    
    return true;
}
```

### **Examples**

| Input | Length | Chars | Result |
|-------|--------|-------|--------|
| "ABC123" | 6 | A,B,C,1,2,3 | ✅ **Valid** |
| "XYZ789" | 6 | X,Y,Z,7,8,9 | ✅ **Valid** |
| "PNR001" | 6 | P,N,R,0,0,1 | ✅ **Valid** |
| "ABC12" | 5 | Too short | ❌ Invalid |
| "ABC1234" | 7 | Too long | ❌ Invalid |
| "abc123" | 6 | Lowercase | ❌ Invalid |
| "ABC-23" | 6 | Special char | ❌ Invalid |

### **Real-World Usage**

```cpp
// In controller.cpp when checking in
if (!is_valid_pnr(pnr_from_user)) {
    return error("Invalid PNR format");
}

// In boarding_controller.cpp
std::string generated_pnr = generate_pnr();
assert(is_valid_pnr(generated_pnr));  // Should always be valid
```

---

## **Function 4: is_valid_seat_number()**

### **Purpose**
Validate seat numbers in aircraft (like "12A", "B5", "25F")

### **Function Signature**
```cpp
bool is_valid_seat_number(const std::string& seat_number);
```

### **What it does**
```
Input: "12A"
  ↓
Check: Is length 2-5 characters?
  ↓
Check: Is it in format (A1 or 12A)?
  ↓
Output: true (valid) or false (invalid)
```

### **Detailed Code Breakdown**

```cpp
bool is_valid_seat_number(const std::string& seat_number) {
    // RULE 1: Length must be 2-5 characters
    if (seat_number.size() < 2 || seat_number.size() > 5) {
        return false;  // "A" (1 char) → invalid
                       // "ABC1234" (7 chars) → invalid
    }
    
    // RULE 2: Must be EITHER "Letter+Digits" OR "Digits+Letter"
    
    // Check format 1: Letter followed by digits (e.g., "A1", "B123")
    auto is_letter_followed_by_digits = [&]() {
        // First character must be a letter
        if (!std::isalpha(static_cast<unsigned char>(seat_number.front()))) {
            return false;  // First char not letter
        }
        
        // All remaining characters must be digits
        for (size_t i = 1; i < seat_number.size(); ++i) {
            if (!std::isdigit(static_cast<unsigned char>(seat_number[i]))) {
                return false;  // Non-digit found
            }
        }
        
        return true;  // Valid format: Letter+Digits
    };
    
    // Check format 2: Digits followed by letter (e.g., "1A", "123B")
    auto is_digits_followed_by_letter = [&]() {
        // Last character must be a letter
        if (!std::isalpha(static_cast<unsigned char>(seat_number.back()))) {
            return false;  // Last char not letter
        }
        
        // All preceding characters must be digits
        for (size_t i = 0; i + 1 < seat_number.size(); ++i) {
            if (!std::isdigit(static_cast<unsigned char>(seat_number[i]))) {
                return false;  // Non-digit found
            }
        }
        
        return true;  // Valid format: Digits+Letter
    };
    
    // Valid if either format matches
    return is_letter_followed_by_digits() || is_digits_followed_by_letter();
}
```

### **Examples**

| Input | Format | Length | Result |
|-------|--------|--------|--------|
| "A1" | Letter+Digit | 2 | ✅ **Valid** |
| "B12" | Letter+Digits | 3 | ✅ **Valid** |
| "C123" | Letter+Digits | 4 | ✅ **Valid** |
| "1A" | Digit+Letter | 2 | ✅ **Valid** |
| "12B" | Digits+Letter | 3 | ✅ **Valid** |
| "123C" | Digits+Letter | 4 | ✅ **Valid** |
| "A" | Too short | 1 | ❌ Invalid |
| "ABCDE" | All letters | 5 | ❌ Invalid |
| "A1B" | Letter+Digit+Letter | 3 | ❌ Invalid |
| "1A2" | Digit+Letter+Digit | 3 | ❌ Invalid |
| "A-1" | Special char | 3 | ❌ Invalid |

### **Real-World Usage**

```cpp
// In inventory_service.cpp when allocating seats
if (!is_valid_seat_number(requested_seat)) {
    return error("Invalid seat format");
}

// Typical valid seats in an aircraft:
// Column letters: A, B, C, D, E, F (6 seats per row)
// Row numbers: 1-100
// Valid seats: A1, B12, C123, 1A, 12B, etc.
```

---

## **Function 5: is_valid_baggage_count()**

### **Purpose**
Validate number of bags allowed for a passenger

### **Function Signature**
```cpp
bool is_valid_baggage_count(int baggage_count);
```

### **What it does**
```
Input: 3
  ↓
Check: Is it between 0-5 bags?
  ↓
Output: true (valid) or false (invalid)
```

### **Detailed Code Breakdown**

```cpp
bool is_valid_baggage_count(int baggage_count) {
    // RULE: Must be between 0 and 5 bags (inclusive)
    return baggage_count >= 0 && baggage_count <= 5;
}
```

### **Examples**

| Input | Check | Result |
|-------|-------|--------|
| 0 | 0 <= 0 <= 5 | ✅ **Valid** |
| 1 | 0 <= 1 <= 5 | ✅ **Valid** |
| 3 | 0 <= 3 <= 5 | ✅ **Valid** |
| 5 | 0 <= 5 <= 5 | ✅ **Valid** |
| -1 | -1 < 0 | ❌ Invalid (negative) |
| 6 | 6 > 5 | ❌ Invalid (too many) |
| 100 | 100 > 5 | ❌ Invalid (too many) |

### **Real-World Usage**

```cpp
// In boarding_controller.cpp during check-in
if (!is_valid_baggage_count(passenger_baggage)) {
    return error("Max 5 bags allowed per passenger");
}
```

---

## **Complete Validator Usage Map**

```
Validator Module
│
├─ is_valid_iata()
│  ├─ Used by: flight_manager.cpp, interactive_search_helper.cpp
│  ├─ Validates: Airport codes (JFK, LAX, DXB)
│  └─ Rules: 3 chars, all UPPERCASE
│
├─ is_valid_passport()
│  ├─ Used by: reservation_engine.cpp, boarding_controller.cpp
│  ├─ Validates: Passenger passport numbers
│  └─ Rules: 6-9 chars, uppercase letters + digits only
│
├─ is_valid_pnr()
│  ├─ Used by: controller.cpp, boarding_controller.cpp
│  ├─ Validates: Booking reference codes
│  └─ Rules: Exactly 6 chars, uppercase letters + digits only
│
├─ is_valid_seat_number()
│  ├─ Used by: inventory_service.cpp, reservation_engine.cpp
│  ├─ Validates: Seat assignments
│  └─ Rules: 2-5 chars, Letter+Digits or Digits+Letter format
│
└─ is_valid_baggage_count()
   ├─ Used by: boarding_controller.cpp, check-in process
   ├─ Validates: Number of bags
   └─ Rules: 0-5 bags per passenger
```

---

## **Why is Validation Important?**

### **Problem Without Validation**

```cpp
// Without validation
void search_flights(std::string origin, std::string dest) {
    // What if user enters:
    // origin = "jfk" (lowercase) → causes bug
    // dest = "LA" (only 2 chars) → causes bug
    // dest = "LAX123" (6 chars) → causes bug
    
    // Now we have corrupted data in system!
}
```

### **With Validation (Correct Approach)**

```cpp
void search_flights(std::string origin, std::string dest) {
    // Check inputs FIRST
    if (!is_valid_iata(origin)) {
        return error("Invalid origin");  // ✓ Fail fast
    }
    
    if (!is_valid_iata(dest)) {
        return error("Invalid destination");  // ✓ Fail fast
    }
    
    // Now safe to use origin and dest
    // ...
}
```

---

## **Test Cases for Validator Module**

```cpp
// Test: IATA validation
assert(is_valid_iata("JFK") == true);
assert(is_valid_iata("jfk") == false);
assert(is_valid_iata("JF") == false);
assert(is_valid_iata("JFKA") == false);

// Test: Passport validation
assert(is_valid_passport("AB123456") == true);
assert(is_valid_passport("AB12345") == true);
assert(is_valid_passport("AB123456789") == false);
assert(is_valid_passport("ab123456") == false);

// Test: PNR validation
assert(is_valid_pnr("ABC123") == true);
assert(is_valid_pnr("ABC12") == false);
assert(is_valid_pnr("abc123") == false);

// Test: Seat validation
assert(is_valid_seat_number("A1") == true);
assert(is_valid_seat_number("A123") == true);
assert(is_valid_seat_number("12A") == true);
assert(is_valid_seat_number("123B") == true);
assert(is_valid_seat_number("A1B") == false);
assert(is_valid_seat_number("A") == false);

// Test: Baggage validation
assert(is_valid_baggage_count(0) == true);
assert(is_valid_baggage_count(5) == true);
assert(is_valid_baggage_count(6) == false);
assert(is_valid_baggage_count(-1) == false);
```

---

## **Summary Table**

| Function | Input Type | Rules | Examples |
|----------|-----------|-------|----------|
| `is_valid_iata()` | string | Exactly 3 uppercase letters | JFK ✅, jfk ❌ |
| `is_valid_passport()` | string | 6-9 chars, uppercase+digits | AB123456 ✅, ab123456 ❌ |
| `is_valid_pnr()` | string | Exactly 6 chars, uppercase+digits | ABC123 ✅, ABC12 ❌ |
| `is_valid_seat_number()` | string | 2-5 chars, Letter+Digits or Digits+Letter | A1 ✅, A1B ❌ |
| `is_valid_baggage_count()` | int | Between 0-5 (inclusive) | 3 ✅, 6 ❌ |

---

## **Key Takeaway**

The validator module is a **gatekeeper** that ensures all inputs meet business requirements BEFORE they're processed. This prevents:
- ❌ Corrupted data
- ❌ System crashes
- ❌ Security vulnerabilities
- ❌ Unexpected behavior

It's used **everywhere** in the system to validate user input! 🛡️

Does this explain the validator module clearly? Any questions about specific validation rules? 🎯
