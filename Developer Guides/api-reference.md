# common_types.h

This module handles the core status-checking logic and shared data structures to ensure consistency across the system.

```cpp 
struct Status {
    bool success;
    std::string error_code;
    std::string message;
};

struct Money {
    long long amount_cents; // Prevents floating point errors
    std::string currency;   // ISO 4217 (e.g., "USD")
};

Status make_success();
Status make_failure(const std::string& code, const std::string& msg);
```

## flight_manager.h

Responsible for searching and managing flight schedules.
```cpp
struct Flight {
    std::string flight_id;
    std::string origin_iata;      // e.g., "ADD"
    std::string destination_iata; // e.g., "DXB"
    std::time_t departure_time;
    std::time_t arrival_time;
    Money base_price;
};

struct SearchCriteria {
    std::string origin;
    std::string destination;
    std::time_t date_window_start;
    std::time_t date_window_end;
};

struct FlightQueryResult {
    std::vector<Flight> available_flights;
    Status status;
};

FlightQueryResult search_flights(const SearchCriteria& criteria);
```

### reservation_engine.h

Handles the logic of holding seats and processing passenger data.
```cpp
enum class SeatClass {
    Economy,
    Business,
    First
};

struct Passenger {
    std::string first_name;
    std::string last_name;
    std::string passport_number;
};

struct BookingRequest {
    std::string flight_id;
    Passenger passenger;
    SeatClass preferred_class;
    std::string seat_number; 
};

struct BookingResult {
    std::string pnr_id; // Passenger Name Record (Confirmation Code)
    Money total_cost;
    Status status;
};

BookingResult create_booking(const BookingRequest& request);
```


### inventory_service.h

Manages the "physical" constraints of the aircraft, ensuring no overbooking occurs without authorization.
```cpp
struct SeatMap {
    std::string flight_id;
    std::map<std::string, bool> occupied_seats; // Seat ID -> Occupied Status
};

struct InventoryUpdate {
    std::string flight_id;
    int economy_delta;
    int business_delta;
};

Status check_availability(const std::string& flight_id, SeatClass s_class);
Status update_inventory(const InventoryUpdate& update);
```

## boarding_controller.h
The interface for day-of-flight operations, check-ins, and boarding passes.
```cpp
struct BoardingPass {
    std::string pnr_id;
    std::string gate;
    int boarding_group;
};

struct CheckInResult {
    BoardingPass pass;
    Status status;
};

CheckInResult process_check_in(const std::string& pnr_id, int baggage_count);
Status update_flight_status(const std::string& flight_id, std::string new_status);
```
## System Invariants

<!-- Atomicity: A booking cannot exist without a corresponding deduction from the flight inventory.

Financial Integrity: All costs are managed in long long cents to ensure 100% accuracy in multi-currency transactions.

Traceability: Every Status returned by a failed operation must include a unique error_code for logging and debugging.

Decoupling: The BoardingController does not need to know the price of the ticket, only the validity of the pnr_id.
 -->
