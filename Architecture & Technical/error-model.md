Here is a professional, expanded **Error Model** for your **Airline Management System**, formatted with `__` for bolding and integrated with the technical standards from your provided documentation.

# System Integrity & Error Modeling Guide: Airline Management System

## 1. Architectural Philosophy
The reliability of an __Airline Management System__ depends on the predictability of its failure states. Following our core philosophy, this project utilizes __explicit status-return contracts__ rather than standard C++ exceptions. This approach ensures that every developer can trace the exact flow of a failure across modules—from the __Flight Scheduler__ down to the __Database Persistence__ layer—without hidden control transfers.

## 2. The Core Status Structure
Every critical function must return a __Status object__ to maintain a uniform communication protocol between system components.

```cpp
struct Status {
    bool ok;                    // Boolean flag indicating operational success
    std::string message;         // Human-readable diagnostic data
    std::size_t errorCode;       // System-specific error identifier
    std::string moduleContext;   // The specific module (e.g., "Booking", "Payment") where the error originated
};
```

__Field Definitions:__
* __ok__: A binary success flag. If `false`, the system must halt the current transaction immediately.
* __message__: A deterministic explanation of the failure. This should never be empty if `ok` is false.
* __errorCode__: Used for programmatic handling (e.g., error code `404` for "Flight Not Found").
* __moduleContext__: Essential for debugging in complex, multi-threaded environments to pinpoint failure ownership.

## 3. Mandatory Propagation Rules
To prevent the processing of __invalid intermediate states__, all layers of the application must follow the __Immediate Return Rule__:
1.  __Invoke Dependency__: Execute the required sub-module function.
2.  __Validate Output__: Check the `ok` flag of the returned status.
3.  __Upward Propagation__: If the dependency fails, return that failure to the caller immediately. __Do not__ attempt to "guess" or "repair" data unless a specific fallback strategy is documented.

## 4. Domain-Specific Failure Ownership
Errors must be handled by the module closest to the data source to ensure __separation of concerns__:
* __Schedule Module__: Responsible for lexical errors in flight codes, invalid date formats, and time-zone overflows.
* __Booking Engine__: Responsible for "Seat Already Occupied," "Invalid Passenger ID," or "Class Mismatch" errors.
* __Financial Controller__: Handles "Transaction Timeout," "Currency Conversion Failure," and "Unauthorized Access".
* __Interface Layer (CLI/GUI)__: Responsible for formatting error displays to the end-user but must __not__ contain deep business logic.

## 5. Professional Message Style Guidelines
Messages should be __concise__, __deterministic__, and __context-aware__. Avoid vague language like "System Error".

__Standardized Examples:__
* __Incorrect__: "Something went wrong with the flight."
* __Professional__: `[Schedule] Invalid IATA code format: 'XX123'`.
* __Professional__: `[Booking] Requested seat '12B' is unavailable for Flight 404`.
* __Professional__: `[Database] Read timeout on 'PassengerManifest' table`.

## 6. Contract Compliance Checklist
Before committing code to the __Airline Management System__ repository, verify your error handling against this checklist:
* [ ] __Non-Empty Messages__: If `ok == false`, the message provides enough detail for a developer to reproduce the issue.
* [ ] __Static Success__: Success statuses carry an empty message and default error code `0` to save memory.
* [ ] __Boundary Respect__: The error is generated in the correct module (e.g., a database error is not being "renamed" by the UI).
* [ ] __Resource Safety__: Ensure all local resources (files, memory) are released before returning a failure status.

## 7. Rationale: Status Objects vs. Exceptions
While exceptions are a standard C++ feature, we prioritize __Status Objects__ for the baseline implementation of this project for two reasons:
1.  __Explicit Learning Flow__: For contributors, seeing the `if (!status.ok)` check at every call site makes the logic easier to audit and understand.
2.  __Performance Determinism__: In high-frequency systems like __Flight Search Engine__ iterations, status returns provide more predictable performance overhead than stack unwinding.
