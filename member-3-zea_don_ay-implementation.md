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



Perfect! Now let me provide a **comprehensive explanation** of the inventory_service module from @worku404/airline-management-codes:

---

# **Inventory Service Module - Complete Detailed Explanation**

## **What is the Inventory Service?**

The inventory service is the **seat and capacity management system** that tracks:
- How many seats are available per seat class (Economy, Business, First)
- Which specific seats are occupied/reserved
- Ensures no overbooking happens

**Think of it like a theater seat management system:** You have limited seats, and you need to track which ones are taken and which are available.

---

## **Member 3 Context (zea_don_ay)**

Member 3 is the **Engine & Inventory Specialist** responsible for:
- Ensuring bookings are atomic (all-or-nothing)
- Preventing overbooking
- Maintaining inventory integrity
- Managing the reservation lifecycle

---

# **Architecture Overview**

## **Data Structures**

### **1. SeatMap Struct**
```cpp
struct SeatMap {
    std::string flight_id;                    // Which flight
    std::map<std::string, bool> occupied_seats;  // Seat number → occupied?
};
```

**Purpose:** Track which specific seats are taken on a flight
**Example:**
```cpp
SeatMap for flight "AA100":
{
    flight_id: "AA100",
    occupied_seats: {
        "A1": true,    // Occupied
        "A2": false,   // Available
        "B1": true,    // Occupied
        "B2": false    // Available
    }
}
```

---

### **2. InventoryUpdate Struct**
```cpp
struct InventoryUpdate {
    std::string flight_id;
    int economy_delta;      // Change in economy seats (+/-)
    int business_delta;     // Change in business seats (+/-)
    int first_delta;        // Change in first class seats (+/-)
};
```

**Purpose:** Describe changes to available seats
**Example:**
```cpp
// Book 1 economy seat on flight "AA100"
InventoryUpdate update {
    flight_id: "AA100",
    economy_delta: -1,   // One fewer economy seat available
    business_delta: 0,   // No change
    first_delta: 0       // No change
};
```

---

### **3. InventorySnapshot Struct**
```cpp
struct InventorySnapshot {
    int economy_available;      // Seats available now
    int business_available;
    int first_available;
    int economy_capacity;       // Total seats in class
    int business_capacity;
    int first_capacity;
};
```

**Purpose:** Show current state of seats
**Example:**
```cpp
InventorySnapshot for "AA100":
{
    economy_available: 45,    // 45 economy seats left
    economy_capacity: 120,    // Total 120 economy seats
    business_available: 8,    // 8 business seats left
    business_capacity: 24,    // Total 24 business seats
    first_available: 2,       // 2 first class seats left
    first_capacity: 12        // Total 12 first class seats
}
```

---

### **4. InventoryState (Internal)**
```cpp
struct InventoryState {
    int economy_capacity;
    int business_capacity;
    int first_capacity;
    int economy_available;      // How many left?
    int business_available;
    int first_available;
    SeatMap seat_map;           // Which specific seats are taken?
};
```

**Purpose:** Internal tracking of all inventory for a flight
**Location:** Hidden in namespace { } (private to this module)

---

## **Global Storage**

```cpp
namespace {
    std::unordered_map<std::string, InventoryState> g_inventory;
    // Key: flight_id (e.g., "AA100")
    // Value: Complete inventory state for that flight
}
```

**Example:**
```cpp
g_inventory = {
    "AA100": InventoryState { ... },  // All data for flight AA100
    "AA200": InventoryState { ... },  // All data for flight AA200
    "BA300": InventoryState { ... }   // All data for flight BA300
}
```

---

# **Core Functions - Detailed Explanation**

## **Function 1: initialize_inventory()**

### **Purpose**
Set up seat pools for all flights when system starts

### **Function Signature**
```cpp
Status initialize_inventory(const std::vector<Flight>& flights,
                           int economy_capacity,
                           int business_capacity,
                           int first_capacity);
```

### **What It Does**

```
Input: List of flights + seat capacities
    ↓
For EACH flight:
    Create inventory with:
    - Economy: capacity available
    - Business: capacity available
    - First class: capacity available
    - Empty seat map
    ↓
Store in global registry
    ↓
Return: Success or Failure
```

### **Detailed Code Breakdown**

```cpp
Status initialize_inventory(const std::vector<Flight>& flights,
                           int economy_capacity,
                           int business_capacity,
                           int first_capacity) {
    
    // VALIDATION: Capacities must be positive
    if (economy_capacity <= 0 || business_capacity <= 0 || first_capacity <= 0) {
        return make_failure("INV_CAPACITY_INVALID", 
                          "Inventory capacity must be positive");
    }
    
    // Clear any old data
    g_inventory.clear();
    
    // For each flight, create inventory entry
    for (const auto& flight : flights) {
        InventoryState state{
            economy_capacity,      // Total capacity
            business_capacity,
            first_capacity,
            economy_capacity,      // Initially all available
            business_capacity,
            first_capacity,
            SeatMap{flight.flight_id, {}}  // Empty seat map
        };
        
        g_inventory.emplace(flight.flight_id, state);
    }
    
    return make_success();
}
```

### **Example Execution**

```
initialize_inventory(
    [Flight("AA100"), Flight("AA200"), Flight("BA300")],
    120,  // economy capacity
    24,   // business capacity
    12    // first capacity
);

Result:
g_inventory = {
    "AA100": {
        economy_capacity: 120,    economy_available: 120,
        business_capacity: 24,    business_available: 24,
        first_capacity: 12,       first_available: 12,
        seat_map: empty
    },
    "AA200": { ... same ... },
    "BA300": { ... same ... }
}
```

---

## **Function 2: check_availability()**

### **Purpose**
Check if seats are available for a specific seat class on a flight

### **Function Signature**
```cpp
Status check_availability(const std::string& flight_id, SeatClass seat_class);
```

### **What It Does**

```
Input: Flight ID + Seat Class (Economy/Business/First)
    ↓
Check: Does flight exist in inventory?
    ↓
Check: Is this seat class valid?
    ↓
Check: Are seats available? (available > 0)
    ↓
Return: Success or specific error
```

### **Detailed Code Breakdown**

```cpp
Status check_availability(const std::string& flight_id, SeatClass seat_class) {
    
    // STEP 1: Find flight in inventory
    auto it = g_inventory.find(flight_id);
    if (it == g_inventory.end()) {
        return make_failure("INV_FLIGHT_UNKNOWN", 
                          "Inventory not initialized for flight");
    }
    
    // STEP 2: Get available seats for this class
    int* available = select_available(it->second, seat_class);
    if (available == nullptr) {
        return make_failure("INV_CLASS_INVALID", "Invalid seat class");
    }
    
    // STEP 3: Check if seats available
    if (*available <= 0) {
        return make_failure("INV_FULL", "Seat class capacity exceeded");
    }
    
    return make_success();  // Seats available!
}
```

### **Helper Function: select_available()**

```cpp
int* select_available(InventoryState& state, SeatClass seat_class) {
    switch (seat_class) {
        case SeatClass::Economy:
            return &state.economy_available;    // Pointer to available count
        case SeatClass::Business:
            return &state.business_available;
        case SeatClass::First:
            return &state.first_available;
        default:
            return nullptr;  // Invalid class
    }
}
```

**Why use pointers?** So we can both check AND modify the value.

### **Examples**

```cpp
// Example 1: Economy seats available
check_availability("AA100", SeatClass::Economy);
// If economy_available > 0 → Success ✅

// Example 2: Business class full
check_availability("AA100", SeatClass::Business);
// If business_available = 0 → Failure (INV_FULL) ❌

// Example 3: Flight doesn't exist
check_availability("XX999", SeatClass::Economy);
// → Failure (INV_FLIGHT_UNKNOWN) ❌
```

---

## **Function 3: update_inventory()**

### **Purpose**
Deduct or add seats when booking/cancellation occurs

### **Function Signature**
```cpp
Status update_inventory(const InventoryUpdate& update);
```

### **What It Does**

```
Input: Flight ID + changes to apply
    ↓
For EACH seat class:
    Check: Would new_available go below 0?  → Reject
    Check: Would new_available exceed capacity?  → Reject
    ↓
If all pass:
    Apply all changes atomically
    ↓
Return: Success or Failure
```

### **Detailed Code Breakdown**

```cpp
Status update_inventory(const InventoryUpdate& update) {
    
    // Find flight
    auto it = g_inventory.find(update.flight_id);
    if (it == g_inventory.end()) {
        return make_failure("INV_FLIGHT_UNKNOWN", 
                          "Inventory not initialized for flight");
    }
    
    InventoryState& state = it->second;

    // Lambda to validate a single class change
    auto check_delta = [&](int available, int capacity, int delta, 
                          const std::string& code) -> Status {
        const int next = available + delta;
        
        // Can't go negative
        if (next < 0) {
            return make_failure(code, "Inventory cannot drop below zero");
        }
        
        // Can't exceed capacity
        if (next > capacity) {
            return make_failure(code, "Inventory cannot exceed capacity");
        }
        
        return make_success();
    };

    // Validate all three classes BEFORE applying any changes
    Status status = check_delta(state.economy_available, 
                               state.economy_capacity, 
                               update.economy_delta, 
                               "INV_ECONOMY_LIMIT");
    if (!status.success) return status;
    
    status = check_delta(state.business_available, 
                        state.business_capacity, 
                        update.business_delta, 
                        "INV_BUSINESS_LIMIT");
    if (!status.success) return status;
    
    status = check_delta(state.first_available, 
                        state.first_capacity, 
                        update.first_delta, 
                        "INV_FIRST_LIMIT");
    if (!status.success) return status;

    // All checks passed - apply changes atomically
    state.economy_available += update.economy_delta;
    state.business_available += update.business_delta;
    state.first_available += update.first_delta;
    
    return make_success();
}
```

### **Why Atomic?**

It validates ALL changes BEFORE applying ANY. This prevents partial updates:

```
BAD (Partial Update):
- Apply economy_delta ✓
- Check business (FAILS) ✗
- Business unchanged but economy was modified!

GOOD (Atomic):
- Check economy ✓
- Check business ✓
- Check first ✓
- If any fails: apply NOTHING
- If all pass: apply ALL together
```

### **Examples**

```cpp
// Example 1: Book 1 economy seat
InventoryUpdate book {
    flight_id: "AA100",
    economy_delta: -1,    // Deduct 1
    business_delta: 0,
    first_delta: 0
};
update_inventory(book);
// If economy > 0: Success ✅
// If economy = 0: Failure ❌

// Example 2: Cancel 1 economy seat
InventoryUpdate cancel {
    flight_id: "AA100",
    economy_delta: +1,    // Add 1 back
    business_delta: 0,
    first_delta: 0
};
update_inventory(cancel);
// If economy < capacity: Success ✅
// If economy = capacity: Can't exceed, Failure ❌
```

---

## **Function 4: reserve_seat()**

### **Purpose**
Mark a specific seat as occupied

### **Function Signature**
```cpp
Status reserve_seat(const std::string& flight_id, const std::string& seat_number);
```

### **What It Does**

```
Input: Flight ID + Seat Number (e.g., "12A")
    ↓
Check: Is seat number valid format?
    ↓
Check: Does flight exist?
    ↓
Check: Is seat already taken?
    ↓
If all pass: Mark seat as occupied
    ↓
Return: Success or Failure
```

### **Detailed Code Breakdown**

```cpp
Status reserve_seat(const std::string& flight_id, const std::string& seat_number) {
    
    // STEP 1: Validate seat format (using validator from Member 5)
    if (!is_valid_seat_number(seat_number)) {
        return make_failure("SEAT_INVALID", "Seat number format is invalid");
    }
    
    // STEP 2: Find flight
    auto it = g_inventory.find(flight_id);
    if (it == g_inventory.end()) {
        return make_failure("INV_FLIGHT_UNKNOWN", 
                          "Inventory not initialized for flight");
    }
    
    // STEP 3: Get seat map
    auto& seat_map = it->second.seat_map.occupied_seats;
    
    // STEP 4: Check if seat already taken
    auto existing = seat_map.find(seat_number);
    if (existing != seat_map.end() && existing->second) {
        return make_failure("SEAT_OCCUPIED", 
                          "Requested seat is already occupied");
    }
    
    // STEP 5: Reserve the seat
    seat_map[seat_number] = true;
    return make_success();
}
```

### **Example Execution**

```cpp
// Reserve seat "12A" on flight "AA100"
reserve_seat("AA100", "12A");

Before:
seat_map for AA100: {
    "12A": (doesn't exist),
    "12B": true
}

After:
seat_map for AA100: {
    "12A": true,         // ← Now occupied!
    "12B": true
}
```

---

## **Function 5: is_seat_taken()**

### **Purpose**
Check if a specific seat is occupied (returns boolean, not Status)

### **Function Signature**
```cpp
bool is_seat_taken(const std::string& flight_id, const std::string& seat_number);
```

### **What It Does**

```
Input: Flight ID + Seat Number
    ↓
Find flight
    ↓
Find seat in seat map
    ↓
Return: true if occupied, false if available or not found
```

### **Code**

```cpp
bool is_seat_taken(const std::string& flight_id, const std::string& seat_number) {
    
    // Try to find flight
    auto it = g_inventory.find(flight_id);
    if (it == g_inventory.end()) {
        return false;  // Flight not found, assume empty
    }
    
    // Try to find seat
    auto seat_it = it->second.seat_map.occupied_seats.find(seat_number);
    
    // Return true only if seat exists AND is marked true
    return seat_it != it->second.seat_map.occupied_seats.end() 
           && seat_it->second;
}
```

### **Examples**

```cpp
is_seat_taken("AA100", "12A");  // true if occupied, false if available
is_seat_taken("AA100", "99Z");  // false if never booked
is_seat_taken("XX999", "1A");   // false if flight doesn't exist
```

---

## **Function 6: get_inventory_snapshot()**

### **Purpose**
Get current state of all seats for a flight

### **Function Signature**
```cpp
InventorySnapshot get_inventory_snapshot(const std::string& flight_id);
```

### **What It Does**

```
Input: Flight ID
    ↓
Find flight inventory
    ↓
Return: Current available + capacity for all classes
```

### **Code**

```cpp
InventorySnapshot get_inventory_snapshot(const std::string& flight_id) {
    
    auto it = g_inventory.find(flight_id);
    
    // If flight not found, return all zeros
    if (it == g_inventory.end()) {
        return {0, 0, 0, 0, 0, 0};
    }
    
    const InventoryState& state = it->second;
    
    // Return snapshot of current state
    return {
        state.economy_available,
        state.business_available,
        state.first_available,
        state.economy_capacity,
        state.business_capacity,
        state.first_capacity
    };
}
```

### **Example**

```cpp
auto snapshot = get_inventory_snapshot("AA100");

snapshot contains:
{
    economy_available: 45,     // 45 seats left
    business_available: 10,    // 10 seats left
    first_available: 5,        // 5 seats left
    economy_capacity: 120,     // Total capacity
    business_capacity: 24,
    first_capacity: 12
}

// Can calculate occupancy:
// Occupied economy = 120 - 45 = 75 seats taken
```

---

# **Complete Usage Flow: Making a Booking**

```
User wants to book flight AA100, economy seat 12A

Step 1: Initialize System
┌─────────────────────────────────────┐
│ initialize_inventory(flights, ...)   │
│ Creates pools for all flights        │
│ AA100: 120 economy, 24 business, 12 first
└─────────────────────────────────────┘
         ↓

Step 2: Check Availability
┌─────────────────────────────────────┐
│ check_availability("AA100",          │
│                   SeatClass::Economy)│
│ "Are economy seats available?"       │
│ → Yes, 120 available ✅              │
└─────────────────────────────────────┘
         ↓

Step 3: Reserve Specific Seat
┌─────────────────────────────────────┐
│ reserve_seat("AA100", "12A")        │
│ Mark seat 12A as occupied            │
│ → Success ✅                         │
└─────────────────────────────────────┘
         ↓

Step 4: Deduct from Available Pool
┌─────────────────────────────────────┐
│ update_inventory({                  │
│   flight_id: "AA100",               │
│   economy_delta: -1,                │
│   business_delta: 0,                │
│   first_delta: 0                    │
│ })                                  │
│                                     │
│ Now AA100 has:                      │
│ - economy_available: 119 (was 120)  │
│ - seat 12A: occupied                │
└─────────────────────────────────────┘
         ↓

Result: Booking created, inventory updated, seat reserved!
```

---

## **Key Design Principles**

### **1. Atomicity**
```
Update all classes or update none.
No partial changes.
```

### **2. Validation Before Application**
```
Check EVERYTHING first.
Only modify if ALL checks pass.
```

### **3. Non-Negative Constraint**
```
available >= 0 (never negative)
available <= capacity (never over)
```

### **4. Separation of Concerns**
```
- Class pools: Track aggregate availability
- Seat map: Track specific seat assignments
- Both must stay in sync
```

---

## **Error Scenarios**

| Scenario | Error Code | Reason |
|----------|-----------|--------|
| Book when full | INV_FULL | No seats in class |
| Book unknown flight | INV_FLIGHT_UNKNOWN | Flight not in inventory |
| Reserve occupied seat | SEAT_OCCUPIED | Seat already taken |
| Invalid seat format | SEAT_INVALID | Wrong format (needs to be A1, 12B, etc.) |
| Deduct more than available | INV_ECONOMY_LIMIT | Would go negative |
| Add beyond capacity | INV_ECONOMY_LIMIT | Would exceed capacity |

---

## **Summary**

**Inventory Service = Seat Management System**

```
┌──────────────────────────────────────┐
│   Initialize Inventory               │
│   (Setup pools for all flights)       │
│                                      │
│   Check Availability                 │
│   (Are there seats?)                 │
│                                      │
│   Reserve Seat                       │
│   (Mark specific seat as taken)      │
│                                      │
│   Update Inventory                   │
│   (Deduct from available pool)       │
│                                      │
│   Get Snapshot                       │
│   (View current state)               │
└──────────────────────────────────────┘
```

Does this explanation make the inventory service clear? Any specific function you'd like me to dive deeper into? 🚀


Perfect! Now let me provide a **comprehensive, extremely detailed explanation** of boarding_controller with complete context from the entire system:

---

# **Boarding Controller - Comprehensive Deep Dive with Full System Context**

## **What is Boarding Controller?**

The boarding controller is the **final passenger journey stage** in the airline system. It's where passengers complete their travel lifecycle:

```
Passenger Journey Arc:
┌─────────────────────────────────────────────────────┐
│                                                     │
│ Search Flights → Book Flight → Check-in → Board    │
│                                                     │
│    (Member 2)   (Member 3)   (Member 4)   (Member 4)
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Member 4 (boarding_controller) handles the last TWO steps:** Check-in and Boarding

---

## **System Architecture Context**

```
┌──────────────────────────────────────────────────────────────┐
│                    AIRLINE SYSTEM                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Member 2: Flight Manager                              │ │
│  │ - Stores flights                                       │ │
│  │ - Enables search                                       │ │
│  │ - Tracks status (On Time, Delayed, Boarding)         │ │
│  └────────────────────────────────────────────────────────┘ │
│                          ↓                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Member 3: Reservation Engine                           │ │
│  │ - Creates bookings                                     │ │
│  │ - Generates PNR (Passenger Name Record)              │ │
│  │ - Allocates seats                                     │ │
│  │ - Tracks reservation status                           │ │
│  │                                                        │ │
│  │ Inventory Service:                                    │ │
│  │ - Manages seat availability per class                │ │
│  │ - Prevents overbooking                               │ │
│  │ - Reserves specific seats                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                          ↓                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Member 4: Boarding Controller  ← YOU ARE HERE         │ │
│  │ - Validates PNR                                        │ │
│  │ - Checks eligibility for boarding                      │ │
│  │ - Issues boarding pass                                 │ │
│  │ - Assigns gate and boarding group                      │ │
│  │ - Manages flight status                                │ │
│  └────────────────────────────────────────────────────────┘ │
│                          ↓                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Member 6: Revenue Service                              │ │
│  │ - Dynamic pricing                                      │ │
│  │ - Financial audit                                      │ │
│  │ - Operational reporting                               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

# **Data Structures in Detail**

## **1. BoardingPass Struct**

```cpp
struct BoardingPass {
    std::string pnr_id;        // Booking reference (6-char code)
    std::string gate;          // Gate assignment (e.g., "G5")
    int boarding_group;        // Priority group (1, 2, or 3)
};
```

### **Real-World Boarding Pass Example:**

```
┌────────────────────────────────────┐
│        BOARDING PASS               │
├────────────────────────────────────┤
│ Confirmation: ABC123               │ ← pnr_id
│ Gate: G5                           │ ← gate
│ Boarding Group: 2                  │ ← boarding_group
│                                    │
│ BOARDING INSTRUCTIONS:             │
│ Board when Group 2 is called       │
│ Proceed to Gate G5                 │
│ Bring boarding pass and ID         │
└────────────────────────────────────┘
```

**Boarding Groups Explained:**

```
GROUP 1 (First Class):
├─ Board FIRST
├─ VIP treatment
├─ Priority access
└─ Higher ticket price

GROUP 2 (Business Class):
├─ Board SECOND
├─ Good service
├─ Mid-tier pricing
└─ After First Class

GROUP 3 (Economy Class):
├─ Board THIRD/LAST
├─ Standard service
├─ Lower pricing
└─ Waits for other groups
```

---

## **2. CheckInResult Struct**

```cpp
struct CheckInResult {
    BoardingPass pass;         // Your actual boarding pass
    Status status;             // Success or error details
    int baggage_count;         // Number of bags checked
};
```

**Complete Result Example:**

```cpp
CheckInResult {
    pass: {
        pnr_id: "ABC123",
        gate: "G5",
        boarding_group: 2
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

# **Core Function: process_check_in() - DETAILED**

### **Function Signature**
```cpp
CheckInResult process_check_in(const std::string& pnr_id, int baggage_count);
```

### **Seven-Step Check-In Process**

```
START: Passenger arrives at check-in counter
│
├─ STEP 1: Format Validation
│  └─ Is PNR "ABC123" format valid? (6 chars, uppercase)
│     ✓ YES → Continue
│     ✗ NO → Return error CHECKIN_PNR_INVALID
│
├─ STEP 2: Baggage Validation
│  └─ Is baggage count between 0-5?
│     ✓ YES → Continue
│     ✗ NO → Return error CHECKIN_BAGGAGE_INVALID
│
├─ STEP 3: PNR Lookup
│  └─ Query reservation registry for PNR
│     ✓ FOUND → Get passenger details
│     ✗ NOT FOUND → Return error CHECKIN_PNR_MISSING
│
├─ STEP 4: Eligibility Check
│  └─ Is reservation status == "Reserved"?
│     (Not "Cancelled", not already "CheckedIn")
│     ✓ YES → Eligible
│     ✗ NO → Return error CHECKIN_STATE_INVALID
│
├─ STEP 5: Generate Boarding Pass
│  └─ Create pass with:
│     ├─ PNR from reservation
│     ├─ Gate based on flight_id
│     └─ Boarding group based on seat class
│
├─ STEP 6: State Transition
│  └─ Update reservation status: Reserved → CheckedIn
│     ✓ SUCCESS → Continue
│     ✗ FAILED → Return error
│
└─ STEP 7: Record & Return
   └─ Increment check-in counter
      Return boarding pass to passenger
```

### **Complete Code Breakdown**

```cpp
CheckInResult process_check_in(const std::string& pnr_id, int baggage_count) {
    
    // ================================================================
    // STEP 1: VALIDATE PNR FORMAT
    // ================================================================
    // PNR must be exactly 6 characters, uppercase letters/digits only
    // Examples: "ABC123" ✓, "abc123" ✗ (lowercase), "AB123" ✗ (5 chars)
    
    if (!is_valid_pnr(pnr_id)) {
        // PNR format is invalid - reject immediately
        return {
            {},  // Empty BoardingPass
            make_failure("CHECKIN_PNR_INVALID", "Invalid PNR format"),
            baggage_count
        };
    }
    
    // ================================================================
    // STEP 2: VALIDATE BAGGAGE COUNT
    // ================================================================
    // System allows 0-5 bags per passenger (business rule)
    // Examples: 0 ✓, 3 ✓, 5 ✓, 6 ✗, -1 ✗
    
    if (!is_valid_baggage_count(baggage_count)) {
        // Baggage count out of range
        return {
            {},
            make_failure("CHECKIN_BAGGAGE_INVALID", "Invalid baggage count"),
            baggage_count
        };
    }
    
    // ================================================================
    // STEP 3: LOOKUP RESERVATION
    // ================================================================
    // Query the reservation registry (maintained by Member 3)
    // Registry is a map: PNR → ReservationRecord
    
    const ReservationRecord* record = find_reservation(pnr_id);
    
    if (record == nullptr) {
        // PNR not found in system - never booked or typo
        return {
            {},
            make_failure("CHECKIN_PNR_MISSING", "PNR not found"),
            baggage_count
        };
    }
    
    // ================================================================
    // STEP 4: CHECK RESERVATION STATUS
    // ================================================================
    // Reservation must be in "Reserved" state
    // Valid states: Reserved, CheckedIn, Boarded, Cancelled
    // Only "Reserved" can check in
    
    if (record->status != ReservationStatus::Reserved) {
        // Reservation is not in correct state
        // Could be: already checked in, cancelled, etc.
        return {
            {},
            make_failure("CHECKIN_STATE_INVALID", 
                        "Reservation not eligible for check-in"),
            baggage_count
        };
    }

    // ================================================================
    // STEP 5: GENERATE BOARDING PASS
    // ================================================================
    // Create the boarding pass with:
    // - PNR from the reservation
    // - Gate derived from flight_id (deterministic)
    // - Boarding group based on seat class (First=1, Business=2, Economy=3)
    
    BoardingPass pass{
        pnr_id,                                          // e.g., "ABC123"
        gate_for(record->request.flight_id),           // e.g., "G5"
        boarding_group_for(record->request.preferred_class)  // e.g., 2
    };

    // ================================================================
    // STEP 6: UPDATE RESERVATION STATUS
    // ================================================================
    // Transition: Reserved → CheckedIn
    // This state change indicates passenger has physically checked in
    
    Status status = update_reservation_status(pnr_id, ReservationStatus::CheckedIn);
    
    if (!status.success) {
        // Status update failed (shouldn't happen normally)
        return {{}, status, baggage_count};
    }

    // ================================================================
    // STEP 7: RECORD CHECK-IN AND RETURN
    // ================================================================
    // Increment global counter for reporting
    ++g_total_checkins;
    
    // Return successful result with boarding pass
    return {pass, make_success(), baggage_count};
}
```

---

### **Error Scenarios with Detailed Analysis**

#### **Scenario 1: Successful Check-In** ✅

```
INPUT:
  pnr_id = "ABC123"
  baggage_count = 2

PROCESSING:
  Step 1: is_valid_pnr("ABC123")
          → 6 chars, all uppercase ✓
  
  Step 2: is_valid_baggage_count(2)
          → 2 is between 0-5 ✓
  
  Step 3: find_reservation("ABC123")
          → Found in registry ✓
          → Passenger: John Smith
          → Flight: AA100
          → Class: Business
  
  Step 4: Check status
          → status == Reserved ✓
          → Eligible for check-in ✓
  
  Step 5: Generate boarding pass
          → pnr_id: "ABC123"
          → gate: gate_for("AA100") → "G0"
          → boarding_group: boarding_group_for(Business) → 2
  
  Step 6: Update status
          → Reserved → CheckedIn ✓
  
  Step 7: Increment counter
          → g_total_checkins++ (now 1)

OUTPUT:
  CheckInResult {
    pass: {
      pnr_id: "ABC123",
      gate: "G0",
      boarding_group: 2
    },
    status: {
      success: true,
      error_code: "",
      message: ""
    },
    baggage_count: 2
  }

RESULT: ✅ BOARDING PASS ISSUED
        Passenger can proceed to Gate G0
        Will board with Group 2
```

---

#### **Scenario 2: Invalid PNR Format** ❌

```
INPUT:
  pnr_id = "abc123"  (lowercase!)
  baggage_count = 2

PROCESSING:
  Step 1: is_valid_pnr("abc123")
          → Has lowercase letters ✗
          → Format invalid

OUTPUT:
  CheckInResult {
    pass: {},
    status: {
      success: false,
      error_code: "CHECKIN_PNR_INVALID",
      message: "Invalid PNR format"
    },
    baggage_count: 2
  }

RESULT: ❌ REJECTION
        Agent: "Sir, your PNR appears incorrect. Please check it."
```

---

#### **Scenario 3: Too Much Baggage** ❌

```
INPUT:
  pnr_id = "ABC123"
  baggage_count = 7  (exceeds limit)

PROCESSING:
  Step 1: is_valid_pnr("ABC123")
          → Valid format ✓
  
  Step 2: is_valid_baggage_count(7)
          → 7 > 5 (max allowed) ✗
          → Baggage limit exceeded

OUTPUT:
  CheckInResult {
    pass: {},
    status: {
      success: false,
      error_code: "CHECKIN_BAGGAGE_INVALID",
      message: "Invalid baggage count"
    },
    baggage_count: 7
  }

RESULT: ❌ REJECTION
        Agent: "Ma'am, you can only check 5 bags. Please remove 2."
```

---

#### **Scenario 4: PNR Not Found** ❌

```
INPUT:
  pnr_id = "XYZ999"  (never booked)
  baggage_count = 2

PROCESSING:
  Step 1: is_valid_pnr("XYZ999")
          → Valid format ✓
  
  Step 2: is_valid_baggage_count(2)
          → Valid ✓
  
  Step 3: find_reservation("XYZ999")
          → Not in registry ✗
          → No such booking

OUTPUT:
  CheckInResult {
    pass: {},
    status: {
      success: false,
      error_code: "CHECKIN_PNR_MISSING",
      message: "PNR not found"
    },
    baggage_count: 2
  }

RESULT: ❌ REJECTION
        Agent: "I don't see this booking in our system. Did you complete your booking?"
```

---

#### **Scenario 5: Already Checked In** ❌

```
INPUT:
  pnr_id = "ABC123"  (same person checking in twice!)
  baggage_count = 2

PROCESSING:
  Step 1: is_valid_pnr("ABC123")
          → Valid ✓
  
  Step 2: is_valid_baggage_count(2)
          → Valid ✓
  
  Step 3: find_reservation("ABC123")
          → Found ✓
  
  Step 4: Check status
          → status == CheckedIn (not Reserved) ✗
          → Not eligible

OUTPUT:
  CheckInResult {
    pass: {},
    status: {
      success: false,
      error_code: "CHECKIN_STATE_INVALID",
      message: "Reservation not eligible for check-in"
    },
    baggage_count: 2
  }

RESULT: ❌ REJECTION
        Agent: "You already checked in earlier today! You're good to board."
```

---

#### **Scenario 6: Cancelled Reservation** ❌

```
INPUT:
  pnr_id = "ABC123"  (reservation was cancelled)
  baggage_count = 2

PROCESSING:
  Step 1-3: Valid inputs, reservation found
  
  Step 4: Check status
          → status == Cancelled ✗
          → Not eligible (reservation was cancelled)

OUTPUT:
  CheckInResult {
    pass: {},
    status: {
      success: false,
      error_code: "CHECKIN_STATE_INVALID",
      message: "Reservation not eligible for check-in"
    },
    baggage_count: 2
  }

RESULT: ❌ REJECTION
        Agent: "This reservation was cancelled. Would you like to rebook?"
```

---

# **Helper Functions in Detail**

## **boarding_group_for() Function**

```cpp
int boarding_group_for(SeatClass seat_class) {
    switch (seat_class) {
        case SeatClass::First:
            return 1;           // First class boards first
        case SeatClass::Business:
            return 2;           // Business class boards second
        case SeatClass::Economy:
        default:
            return 3;           // Economy class boards last
    }
}
```

### **Real-World Example**

```
At the gate:
  Agent: "We're now boarding Group 1 - First Class passengers"
  First Class passengers board
  
  Agent: "Now boarding Group 2 - Business Class"
  Business passengers board
  
  Agent: "Finally, Group 3 - Economy passengers, please board"
  Economy passengers board

Why this order?
- First Class: Fewer passengers, need time to settle
- Business: Medium number, good traffic flow
- Economy: Largest group, fills remaining space efficiently
```

---

## **gate_for() Function**

```cpp
std::string gate_for(const std::string& flight_id) {
    if (flight_id.empty()) {
        return "G0";  // Default if no flight ID
    }
    
    // Get the last character of flight ID
    char suffix = flight_id.back();
    
    // Must be a digit (0-9); otherwise default to '1'
    if (suffix < '0' || suffix > '9') {
        suffix = '1';
    }
    
    // Combine "G" prefix with digit
    return std::string("G") + suffix;
}
```

### **Gate Assignment Examples**

```
Flight ID → Gate Calculation

"AA100" 
  → Last char: '0' (is digit)
  → Gate: "G0"

"BA201"
  → Last char: '1' (is digit)
  → Gate: "G1"

"CX305"
  → Last char: '5' (is digit)
  → Gate: "G5"

"DELTA"
  → Last char: 'A' (not digit)
  → Default to '1'
  → Gate: "G1"

""
  → Empty flight ID
  → Gate: "G0"

Why this design?
- Deterministic: Same flight → Same gate (always)
- Simple: Quick calculation, no lookup needed
- Distributed: Spreads flights across gates
- Testable: Predictable behavior
```

---

# **Global State Management**

```cpp
namespace {
    int g_total_checkins = 0;  // Incremented each successful check-in
}
```

**Usage:**

```cpp
// First check-in
process_check_in("ABC123", 2);  // Succeeds
// g_total_checkins = 1

// Second check-in
process_check_in("DEF456", 1);  // Succeeds
// g_total_checkins = 2

// Query total
int total = get_total_checkins();  // Returns: 2
```

---

# **Integration Points: What boarding_controller Uses**

## **From Member 5 (validator.h)**

```cpp
// Validate PNR format
bool is_valid_pnr(const std::string& pnr_id);
// Validates: exactly 6 chars, all uppercase/digits
// Example: "ABC123" ✓, "abc123" ✗

// Validate baggage count
bool is_valid_baggage_count(int baggage_count);
// Validates: 0 ≤ count ≤ 5
// Example: 3 ✓, 6 ✗
```

---

## **From Member 3 (reservation_engine.h)**

```cpp
// Find a reservation by PNR
const ReservationRecord* find_reservation(const std::string& pnr_id);
// Returns pointer to reservation or nullptr
// Contains: passenger info, flight ID, seat class, status

// Update reservation status
Status update_reservation_status(const std::string& pnr_id, 
                                 ReservationStatus new_status);
// Changes: Reserved → CheckedIn
```

---

## **From Member 2 (flight_manager.h)**

```cpp
// Update flight status
Status set_flight_status(const std::string& flight_id, 
                        const std::string& new_status);
// Examples: "Boarding", "Delayed", "Cancelled"
// Wraps this function via update_flight_status()
```

---

# **What boarding_controller Provides**

```cpp
// Main check-in function
CheckInResult process_check_in(const std::string& pnr_id, int baggage_count);

// Update flight status (wrapper)
Status update_flight_status(const std::string& flight_id, 
                           const std::string& new_status);

// Get check-in statistics
int get_total_checkins();

// Get boarding statistics (currently same as checkins)
int get_total_boarded();
```

---

# **Real-World Passenger Journey**

```
1. SEARCH STAGE (Member 2)
   Passenger: "Show me flights from NYC to LA on May 20"
   System: Searches flight registry
   Result: "Flight AA100 at 2pm, $150"

2. BOOKING STAGE (Member 3)
   Passenger: "I'll take AA100"
   System: 
     ✓ Validates passport
     ✓ Checks inventory
     ✓ Deducts seat
     ✓ Calculates price (maybe with dynamic pricing from Member 6)
     ✓ Generates unique PNR
   Result: "Confirmed! Your PNR is ABC123"

3. CHECK-IN STAGE (Member 4 - boarding_controller)
   Passenger arrives at airport: "Checking in"
   System: process_check_in("ABC123", 2)
     ✓ Validate PNR format
     ✓ Lookup reservation
     ✓ Check reservation state
     ✓ Generate boarding pass
   Result: "Here's your boarding pass: Gate G0, Group 2"

4. BOARDING STAGE (Member 4)
   Agent: "Now boarding Group 2"
   Passenger: "That's me!" → Heads to Gate G0
   System: update_flight_status("AA100", "Boarding")
   
   Passenger: Boards plane
   System: increment g_total_checkins

5. FLIGHT (Members 2, 6)
   Plane departs
   If delayed: update_flight_status("AA100", "Delayed")
   Revenue service tracks if pricing was optimal

6. ARRIVAL
   Passenger disembarks
   Journey complete!
```

---

# **State Machine: Reservation Status**

```cpp
enum class ReservationStatus {
    Reserved,    // Initial state (after booking)
    CheckedIn,   // After check-in (after process_check_in)
    Boarded      // After boarding (theoretical, not used in current code)
};

State Transitions:

    ┌─────────────┐
    │  Reserved   │  ← Initial state (created by Member 3)
    └──────┬──────┘
           │
           │ process_check_in() succeeds
           │
           ↓
    ┌─────────────┐
    │  CheckedIn  │  ← After check-in
    └──────┬──────┘
           │
           │ [Theoretical - not implemented]
           │
           ↓
    ┌─────────────┐
    │   Boarded   │  ← After passenger boards
    └─────────────┘

Note: Cancellation not shown, but would create a separate state
```

---

# **Error Handling Strategy**

```
Fail-Fast Validation Pattern:

process_check_in("ABC123", 2)
    ↓
Check PNR format
    ├─ ✗ Invalid → Return immediately with error
    ↓ ✓ Valid
Check baggage
    ├─ ✗ Invalid → Return immediately with error
    ↓ ✓ Valid
Lookup PNR
    ├─ ✗ Not found → Return immediately with error
    ↓ ✓ Found
Check status
    ├─ ✗ Wrong state → Return immediately with error
    ↓ ✓ Eligible
Generate pass
    ↓
Update state
    ├─ ✗ Failed → Return error
    ↓ ✓ Success
Return boarding pass ✅
```

**Why fail-fast?**
- Catch errors early
- No wasted processing
- Clean error messages
- Atomic operation (all or nothing)

---

# **Summary Table: All Functions**

| Function | Input | Processing | Output | Use Case |
|----------|-------|-----------|--------|----------|
| `process_check_in()` | PNR, baggage | 7-step validation + boarding pass generation | CheckInResult | Passenger checks in at counter |
| `update_flight_status()` | Flight ID, status | Wrapper to flight_manager | Status | Update flight status (Boarding, Delayed) |
| `get_total_checkins()` | None | Read global counter | int | Statistics/reporting |
| `get_total_boarded()` | None | Read global counter | int | Statistics/reporting |
| `boarding_group_for()` | SeatClass | Switch statement | int (1, 2, or 3) | Determine boarding priority |
| `gate_for()` | Flight ID | Extract last digit | string ("G0"-"G9") | Assign gate deterministically |

---

## **Key Design Principles**

### **1. Validation Before Mutation**
```
✓ Check everything FIRST
✓ Only modify state if all checks pass
✓ Prevents partial/invalid states
```

### **2. Determinism**
```
✓ Same input → Same output (always)
✓ Same flight → Same gate (always)
✓ Makes system predictable
```

### **3. Atomic Operations**
```
✓ Check-in either fully succeeds or fully fails
✓ No partial states
✓ No data inconsistency
```

### **4. Clear Error Reporting**
```
✓ Specific error codes (CHECKIN_PNR_INVALID, etc.)
✓ Not generic "error occurred"
✓ Helps debugging and user communication
```

### **5. Separation of Concerns**
```
✓ boarding_controller doesn't manage reservations (Member 3 does)
✓ boarding_controller doesn't search flights (Member 2 does)
✓ boarding_controller only handles check-in and gate assignment
✓ Each module has clear responsibility
```

---

## **Visual Summary**

```
┌─────────────────────────────────────────────┐
│      BOARDING CONTROLLER (Member 4)         │
├─────────────────────────────────────────────┤
│                                             │
│  INPUT: PNR "ABC123" + Baggage 2           │
│           ↓                                 │
│  VALIDATION LAYER                          │
│  ├─ Format check: is_valid_pnr()          │
│  ├─ Baggage check: is_valid_baggage_count()
│  └─ Lookup: find_reservation()             │
│           ↓                                 │
│  ELIGIBILITY LAYER                         │
│  └─ Status check: Reserved?                │
│           ↓                                 │
│  GENERATION LAYER                          │
│  ├─ Gate: gate_for(flight_id)             │
│  └─ Group: boarding_group_for(class)      │
│           ↓                                 │
│  STATE TRANSITION LAYER                    │
│  └─ Update status: Reserved → CheckedIn   │
│           ↓                                 │
│  OUTPUT: Boarding Pass {                   │
│    pnr: "ABC123",                         │
│    gate: "G5",                            │
│    group: 2                               │
│  }                                         │
│                                             │
└─────────────────────────────────────────────┘
```

---

## **Key Takeaways**

✅ **boarding_controller = Airport Check-In Counter**

It:
1. **Validates** PNR format and baggage
2. **Looks up** passenger reservation
3. **Checks** if eligible for boarding
4. **Issues** boarding pass with gate and group
5. **Records** check-in event
6. **Manages** flight status updates

It's the **critical bridge between reservation and boarding**! 🎫✈️

---

Does this ultra-detailed explanation make the boarding_controller crystal clear? Any specific scenario or edge case you'd like me to explore? 🚀
