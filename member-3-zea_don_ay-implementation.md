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
