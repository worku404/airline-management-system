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








Perfect! Now let me provide a **deep, comprehensive explanation** of the boarding_controller module:

---

# **Boarding Controller Module - Complete Deep Dive**

## **What is the Boarding Controller?**

The boarding controller is the **final stage of the passenger journey** - it handles:
- ✈️ **Check-in validation** - Is your PNR valid and eligible to check in?
- 🎫 **Boarding pass generation** - Creates your boarding pass with gate and boarding group
- 📊 **Flight status tracking** - Updates flight status (On Time, Delayed, Boarding)
- 🛄 **Baggage management** - Records baggage count during check-in

**Think of it like an airport check-in counter:** You show your PNR (Passenger Name Record), we validate it, assign you a gate and boarding group, and you're ready to board!

---

## **Member 4 Context (eloicry)**

Member 4 is the **Operations & Boarding Lead** responsible for:
- Day-of-flight operations
- Producing boarding passes
- Managing flight status
- Closing the passenger lifecycle

**Dependencies:**
- ✅ Member 1 (Status, Money, SeatClass)
- ✅ Member 5 (Validators)
- ✅ Member 3 (Reservation registry, PNR lookup)
- ✅ Member 2 (Flight status updates)

---

# **Data Structures**

## **1. BoardingPass Struct**

```cpp
struct BoardingPass {
    std::string pnr_id;        // Booking reference (e.g., "ABC123")
    std::string gate;          // Gate number (e.g., "G5")
    int boarding_group;        // Priority group (1=First, 2=Business, 3=Economy)
};
```

**Purpose:** Your actual boarding pass information
**Example:**
```cpp
BoardingPass {
    pnr_id: "ABC123",
    gate: "G5",           // Board at gate 5
    boarding_group: 1     // First class boards first
}
```

**Boarding Groups Explained:**
```
Group 1: First Class        (Highest priority - board first)
Group 2: Business Class     (Medium priority)
Group 3: Economy Class      (Lower priority - board last)
```

---

## **2. CheckInResult Struct**

```cpp
struct CheckInResult {
    BoardingPass pass;      // Your boarding pass
    Status status;          // Success or error details
    int baggage_count;      // How many bags you're checking
};
```

**Purpose:** Complete result of check-in operation
**Example:**
```cpp
CheckInResult {
    pass: {
        pnr_id: "ABC123",
        gate: "G5",
        boarding_group: 1
    },
    status: {
        success: true,
        error_code: "",
        message: ""
    },
    baggage_count: 2
}
```

---

# **Core Functions - Deep Explanation**

## **Function 1: process_check_in()**

### **Purpose**
Validate PNR, check eligibility, and generate boarding pass

### **Function Signature**
```cpp
CheckInResult process_check_in(const std::string& pnr_id, int baggage_count);
```

### **The Complete Check-In Process**

```
User arrives at airport with PNR "ABC123" and 2 bags
            ↓
┌──────────────────────────────────────────────────────┐
│ STEP 1: Validate PNR Format                         │
├──────────────────────────────────────────────────────┤
│ is_valid_pnr("ABC123")?                             │
│ Must be exactly 6 chars, uppercase letters/digits   │
│ ✓ YES → Continue                                    │
│ ✗ NO → Return error "CHECKIN_PNR_INVALID"         │
└──────────────────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────────────────┐
│ STEP 2: Validate Baggage Count                      │
├──────────────────────────────────────────────────────┤
│ is_valid_baggage_count(2)?                          │
│ Must be 0-5 bags                                    │
│ ✓ YES (2 ≤ 5) → Continue                           │
│ ✗ NO (e.g., 10) → Return error "CHECKIN_BAGGAGE_INVALID"
└──────────────────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────────────────┐
│ STEP 3: Lookup Reservation                          │
├──────────────────────────────────────────────────────┤
│ find_reservation("ABC123")                          │
│ Search in reservation registry                      │
│ ✓ FOUND → Get passenger details                     │
│ ✗ NOT FOUND → Error "CHECKIN_PNR_MISSING"          │
└──────────────────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────────────────┐
│ STEP 4: Check Reservation Status                    │
├──────────────────────────────────────────────────────┤
│ Is status == ReservationStatus::Reserved?           │
│ (Not Cancelled, Not Already Checked In)             │
│ ✓ YES → Eligible for check-in                       │
│ ✗ NO → Error "CHECKIN_STATE_INVALID"               │
└──────────────────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────────────────┐
│ STEP 5: Generate Boarding Pass                      │
├──────────────────────────────────────────────────────┤
│ Create BoardingPass with:                           │
│ - PNR: "ABC123"                                     │
│ - Gate: Determined by flight_id (e.g., "G5")      │
│ - Boarding Group: Determined by seat class         │
└──────────────────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────────────────┐
│ STEP 6: Update Reservation Status                   │
├──────────────────────────────────────────────────────┤
│ Change status from "Reserved" to "CheckedIn"        │
│ ✓ SUCCESS → Continue                                │
│ ✗ FAILED → Return error                             │
└──────────────────────────────────────────────────────┘
            ↓
┌──────────────────────────────────────────────────────┐
│ STEP 7: Record Check-In                             │
├──────────────────────────────────────────────────────┤
│ Increment g_total_checkins counter                  │
│ Return CheckInResult with boarding pass             │
└──────────────────────────────────────────────────────┘
```

### **Detailed Code Breakdown**

```cpp
CheckInResult process_check_in(const std::string& pnr_id, int baggage_count) {
    
    // STEP 1: Validate PNR format (using Member 5's validator)
    if (!is_valid_pnr(pnr_id)) {
        return {
            {},  // Empty BoardingPass
            make_failure("CHECKIN_PNR_INVALID", "Invalid PNR format"),
            baggage_count
        };
    }
    
    // STEP 2: Validate baggage count (using Member 5's validator)
    if (!is_valid_baggage_count(baggage_count)) {
        return {
            {},
            make_failure("CHECKIN_BAGGAGE_INVALID", "Invalid baggage count"),
            baggage_count
        };
    }
    
    // STEP 3: Lookup reservation (using Member 3's registry)
    const ReservationRecord* record = find_reservation(pnr_id);
    if (record == nullptr) {
        return {
            {},
            make_failure("CHECKIN_PNR_MISSING", "PNR not found"),
            baggage_count
        };
    }
    
    // STEP 4: Check if eligible for check-in
    if (record->status != ReservationStatus::Reserved) {
        return {
            {},
            make_failure("CHECKIN_STATE_INVALID", 
                        "Reservation not eligible for check-in"),
            baggage_count
        };
    }

    // STEP 5: Generate boarding pass
    BoardingPass pass{
        pnr_id,                                          // e.g., "ABC123"
        gate_for(record->request.flight_id),           // Determine gate
        boarding_group_for(record->request.preferred_class)  // Determine group
    };

    // STEP 6: Update reservation status
    Status status = update_reservation_status(pnr_id, ReservationStatus::CheckedIn);
    if (!status.success) {
        return {{}, status, baggage_count};
    }

    // STEP 7: Record successful check-in
    ++g_total_checkins;
    
    return {pass, make_success(), baggage_count};
}
```

---

### **Helper Functions**

#### **boarding_group_for()**

```cpp
int boarding_group_for(SeatClass seat_class) {
    switch (seat_class) {
        case SeatClass::First:
            return 1;           // First class boards first
        case SeatClass::Business:
            return 2;           // Business boards second
        case SeatClass::Economy:
        default:
            return 3;           // Economy boards last
    }
}
```

**Why this matters?**
- **First Class** = Higher priority (VIP treatment, more room)
- **Business** = Medium priority
- **Economy** = Lower priority

**Real-world example:**
```
Boarding announcement: "We now boarding group 1 - First Class passengers"
Your boarding group: 1 (First Class)
You board immediately! ✅

Later: "Now boarding group 3 - Economy"
Economy passenger: Waits their turn
```

---

#### **gate_for()**

```cpp
std::string gate_for(const std::string& flight_id) {
    if (flight_id.empty()) {
        return "G0";  // Default gate
    }
    
    // Get last character of flight ID
    // Example: "AA100" → '0' is last char
    char suffix = flight_id.back();
    
    // If not a digit, default to '1'
    if (suffix < '0' || suffix > '9') {
        suffix = '1';
    }
    
    // Combine "G" + digit
    // Example: "AA100" → gate "G0"
    return std::string("G") + suffix;
}
```

**Examples:**
```
gate_for("AA100") → "G0"  (last digit is 0)
gate_for("AA101") → "G1"  (last digit is 1)
gate_for("AA105") → "G5"  (last digit is 5)
gate_for("BRITISH") → "G1"  (no digit, default to 1)
gate_for("") → "G0"  (empty, default to 0)
```

**Why assign gates this way?**
- Deterministic (same flight always gets same gate)
- Simple algorithm
- Spreads flights across gates

---

### **Example Scenarios**

#### **Scenario 1: Successful Check-In**

```
Input:
  pnr_id: "ABC123"
  baggage_count: 2

Processing:
  ✓ PNR format valid (6 chars, uppercase)
  ✓ Baggage valid (0-5)
  ✓ PNR found in registry
  ✓ Reservation status = "Reserved"
  ✓ Status updated to "CheckedIn"

Output:
CheckInResult {
    pass: {
        pnr_id: "ABC123",
        gate: "G1",         (from flight_id "AA101")
        boarding_group: 2   (from Business class)
    },
    status: {
        success: true,
        error_code: "",
        message: ""
    },
    baggage_count: 2
}
```

---

#### **Scenario 2: Invalid PNR Format**

```
Input:
  pnr_id: "abc123"  (lowercase!)
  baggage_count: 2

Processing:
  ✗ PNR format invalid (has lowercase)

Output:
CheckInResult {
    pass: {},
    status: {
        success: false,
        error_code: "CHECKIN_PNR_INVALID",
        message: "Invalid PNR format"
    },
    baggage_count: 2
}
```

---

#### **Scenario 3: PNR Not Found**

```
Input:
  pnr_id: "XYZ999"
  baggage_count: 1

Processing:
  ✓ PNR format valid
  ✓ Baggage valid
  ✗ PNR NOT in reservation registry

Output:
CheckInResult {
    pass: {},
    status: {
        success: false,
        error_code: "CHECKIN_PNR_MISSING",
        message: "PNR not found"
    },
    baggage_count: 1
}
```

---

#### **Scenario 4: Already Checked In**

```
Input:
  pnr_id: "ABC123" (already checked in before)
  baggage_count: 2

Processing:
  ✓ PNR format valid
  ✓ Baggage valid
  ✓ PNR found
  ✗ Reservation status = "CheckedIn" (not "Reserved")

Output:
CheckInResult {
    pass: {},
    status: {
        success: false,
        error_code: "CHECKIN_STATE_INVALID",
        message: "Reservation not eligible for check-in"
    },
    baggage_count: 2
}
```

---

## **Function 2: update_flight_status()**

### **Purpose**
Update flight status (On Time, Delayed, Boarding, Cancelled)

### **Function Signature**
```cpp
Status update_flight_status(const std::string& flight_id, const std::string& new_status);
```

### **Code**

```cpp
Status update_flight_status(const std::string& flight_id, const std::string& new_status) {
    // Simply delegate to flight_manager's set_flight_status()
    // Member 2 handles validation of valid status values
    return set_flight_status(flight_id, new_status);
}
```

**This is a simple wrapper** that delegates to Member 2 (flight_manager) to update flight status.

### **Examples**

```cpp
// Update flight to "Boarding"
update_flight_status("AA100", "Boarding");

// Update flight to "Delayed"
update_flight_status("AA100", "Delayed");

// Update flight to "Cancelled"
update_flight_status("AA100", "Cancelled");

// Update flight to "On Time"
update_flight_status("AA100", "On Time");
```

---

## **Function 3: get_total_checkins()**

### **Purpose**
Get total number of check-ins so far

### **Function Signature**
```cpp
int get_total_checkins();
```

### **Code**

```cpp
int get_total_checkins() {
    return g_total_checkins;  // Global counter
}
```

**Example:**
```cpp
// After 5 successful check-ins
get_total_checkins();  // Returns: 5

// After 10 successful check-ins
get_total_checkins();  // Returns: 10
```

---

## **Function 4: get_total_boarded()**

### **Purpose**
Get total number of passengers boarded

### **Function Signature**
```cpp
int get_total_boarded();
```

### **Code**

```cpp
int get_total_boarded() {
    return g_total_checkins;  // Same as check-ins
}
```

**Note:** Currently, `get_total_boarded()` returns the same as `get_total_checkins()`. In a real system, boarding might be tracked separately (when passenger actually boards the plane).

---

# **Global State**

```cpp
namespace {
    int g_total_checkins = 0;  // Counter for all check-ins
}
```

**Incremented by:**
- Each successful call to `process_check_in()`

**Used by:**
- `get_total_checkins()`
- `get_total_boarded()`

---

# **Complete Check-In Workflow: User Perspective**

```
┌─────────────────────────────────────────────────────┐
│          USER ARRIVES AT CHECK-IN COUNTER           │
└─────────────────────────────────────────────────────┘
    
    Agent: "I'll need your PNR and baggage count"
    You: "My PNR is ABC123, I have 2 bags"
    
    ↓
    
┌─────────────────────────────────────────────────────┐
│     System: process_check_in("ABC123", 2)           │
└─────────────────────────────────────────────────────┘
    
    ✓ PNR format check: ABC123 (6 chars, uppercase)
    ✓ Baggage check: 2 (0-5 allowed)
    ✓ Lookup: Found in reservation registry
    ✓ Status check: Reservation is "Reserved"
    ✓ Create boarding pass:
        - PNR: ABC123
        - Gate: G1 (from flight AA101)
        - Group: 2 (Business class)
    ✓ Update status to "CheckedIn"
    ✓ Increment counter
    
    ↓
    
┌─────────────────────────────────────────────────────┐
│            RETURN BOARDING PASS                     │
└─────────────────────────────────────────────────────┘
    
    Agent: "Here's your boarding pass!"
    
    ┌────────────────────────────┐
    │   BOARDING PASS            │
    ├────────────────────────────┤
    │ PNR: ABC123                │
    │ Gate: G1                   │
    │ Boarding Group: 2          │
    │ Baggage: 2 bags            │
    └────────────────────────────┘
    
    You: "When do we board?"
    Agent: "Group 1 boarding now, you're Group 2, so very soon!"
    
    ↓ (Later)
    
    Agent: "Now boarding Group 2 passengers!"
    You: "That's me!" → Head to Gate G1
```

---

# **Integration Points**

## **Dependencies (What it uses)**

```
boarding_controller.cpp depends on:

1. common_types.h
   └─ BoardingPass, CheckInResult, Status, SeatClass

2. validator.h (Member 5)
   ├─ is_valid_pnr()
   └─ is_valid_baggage_count()

3. flight_manager.h (Member 2)
   └─ set_flight_status()

4. reservation_engine.h (Member 3)
   ├─ find_reservation()
   └─ update_reservation_status()
```

## **Who uses it**

```
controller.cpp uses boarding_controller:

1. When user types: checkin ABC123 2
   └─ Calls process_check_in("ABC123", 2)

2. When user types: status AA100 Boarding
   └─ Calls update_flight_status("AA100", "Boarding")

3. For reports/stats
   └─ Calls get_total_checkins()
```

---

# **Error Handling Flow**

```
process_check_in()
    ├─ Validation Error 1: Invalid PNR Format
    │  └─ Return: CHECKIN_PNR_INVALID
    │
    ├─ Validation Error 2: Invalid Baggage Count
    │  └─ Return: CHECKIN_BAGGAGE_INVALID
    │
    ├─ Logic Error 1: PNR Not Found
    │  └─ Return: CHECKIN_PNR_MISSING
    │
    ├─ Logic Error 2: Wrong Reservation State
    │  └─ Return: CHECKIN_STATE_INVALID
    │
    ├─ System Error: Failed to Update Status
    │  └─ Return: Error from update_reservation_status()
    │
    └─ Success ✅
       └─ Return: BoardingPass + Success
```

---

# **Key Design Principles**

### **1. Fail-Fast Validation**
```
Check everything BEFORE making changes:
1. Format validation (PNR, baggage)
2. Existence check (PNR in registry)
3. State check (is reservation valid?)
Only then: Update state
```

### **2. Deterministic Gate Assignment**
```
Same flight → Same gate (always)
Makes it predictable for users and staff
```

### **3. Boarding Priority**
```
First Class → Business → Economy
Matches real-world airline operations
```

### **4. State Transitions**
```
Reserved ──(check-in)──> CheckedIn
   ↓
 (cancel)
   ↓
Cancelled
```

### **5. Atomic Check-In**
```
Either fully succeeds OR fully fails
No partial states
```

---

# **Summary Table**

| Function | Input | Output | Purpose |
|----------|-------|--------|---------|
| `process_check_in()` | PNR + baggage | BoardingPass + Status | Validate PNR, generate pass |
| `update_flight_status()` | flight_id + status | Status | Update flight status |
| `get_total_checkins()` | None | Integer | Get check-in count |
| `get_total_boarded()` | None | Integer | Get boarded count |

---

# **Real-World Parallels**

```
Boarding Controller ≈ Airport Check-in Counter

┌─────────────────────────────────────────┐
│ Passenger arrives with boarding pass    │
│ (equivalent to PNR)                     │
│                                         │
│ ✓ Is it a valid format?                │
│ ✓ Do you have luggage? (baggage count) │
│ ✓ Is the booking in our system?        │
│ ✓ Haven't you checked in already?      │
│                                         │
│ If all pass:                            │
│ → Issue boarding pass                   │
│ → Assign gate and boarding group        │
│ → Log the check-in                      │
│ → Ready to board!                       │
└─────────────────────────────────────────┘
```

---

## **Key Takeaways**

✅ **Boarding Controller = Check-in Operations**

The boarding controller:
1. **Validates** PNR and baggage
2. **Looks up** reservation record
3. **Checks** if eligible for check-in
4. **Generates** boarding pass with gate and group
5. **Updates** reservation status
6. **Counts** successful check-ins

It's the **bridge between reservations and actual boarding**! 🎫✈️

---

Does this deep dive clarify the boarding_controller? Any specific part you'd like to explore further? 🚀
