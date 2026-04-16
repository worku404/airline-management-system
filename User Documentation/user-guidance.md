

---

# User Guidance & Operational Manual: Airline Management System

## 1. System Purpose & Operational Scope
The __Airline Management System__ is a high-performance, terminal-based backend designed to simulate the mission-critical workflows of a modern airline. This includes high-accuracy flight searching, atomic seat reservations, and day-of-flight operations such as passenger check-in. The system is built on a __Modular Service Architecture__, ensuring that each operational domain—from inventory guarding to financial tracking—operates within strict, deterministic boundaries.

## 2. Technical Prerequisites
Before attempting to build or execute the system, ensure the following environment specifications are met:
* __Toolchain__: A C++ compiler supporting the __C++17 Standard__ (e.g., `g++` version 9.0+ or `clang++` version 8.0+).
* __Environment__: A standard terminal emulator (Linux Bash, macOS Zsh, or Windows PowerShell).
* __Memory Model__: The system utilizes __64-bit Integer Arithmetic__; ensure the target architecture supports `long long` types for financial precision.

## 3. Build & Compilation Procedures
The system utilizes a __Flat Directory Structure__ (or structured `src/` layout) and must be compiled with strict diagnostic flags to ensure portfolio-grade code quality.

### __Standard Compilation (Linux/macOS)__
Execute the following command from the project root:
```bash
g++ -std=c++17 -Wall -Wextra -pedantic *.cpp -o airline_sim
```

### __Standard Compilation (Windows PowerShell)__
```powershell
g++ -std=c++17 -Wall -Wextra -pedantic *.cpp -o airline_sim.exe
```

__Flag Definitions:__
* `-std=c++17`: Enforces modern language features like structured bindings and optional types.
* `-Wall -Wextra -pedantic`: Enables the highest level of compiler warnings to catch potential logic flaws during build-time.

---

## 4. Execution & The REPL Environment
Once compiled, the system enters a __Read-Eval-Print Loop (REPL)__, which serves as the primary interface for all airline operations.

### __Launching the System__
```bash
./airline_sim
```

### __Global System Commands__
These commands manage the operational state of the terminal session:
* __`:help`__: Displays the comprehensive command reference and syntax guide.
* __`:quit`__: Initiates a graceful shutdown of the system, ensuring all memory-resident state is cleared.



---

## 5. Functional Command Reference (Target Behavior)
The system processes commands through a __Command-Line Orchestrator__. Below are the primary interaction patterns supported by the backend.

### __Flight Discovery & Search__
Users can query the global flight registry using __IATA Airport Codes__.
* __Syntax__: `search <origin_iata> <destination_iata>`
* __Example__: `search ADD DXB`
* __Expected Output__: A formatted table of available flights, including __Flight IDs__, departure times, and base pricing.

### __Reservation & PNR Lifecycle__
The booking pipeline creates a persistent reservation record and updates flight inventory.
* __Syntax__: `book <flight_id> <seat_class>`
* __Example__: `book FL-102 Business`
* __Expected Output__: `Booking Confirmed: PNR-A9B8C7`.

### __Operational Check-In__
Validates a PNR and prepares the passenger for boarding.
* __Syntax__: `checkin <pnr_id>`
* __Example__: `checkin A9B8C7`
* __Expected Output__: A synthesized __Boarding Pass__ containing gate assignment and boarding group details.

---

## 6. Diagnostic & Error Reporting Protocol
In alignment with the __Zero-Footprint Error Model__, the system provides explicit feedback when a command fails. Errors are reported with a specific __Phase Identifier__ and an __Error Code__.



### __Interpreting Output__
* __Result Output__: Prefixed with `Result:`, signifying a successful logic execution and state change.
* __Error Output__: Formatted as `Error: <Module>: <Message> [Code]`.

### __Common Error Scenarios__
* __Lexical Error__: `search ADD123` $\rightarrow$ `Error: Tokenizer: Invalid IATA format at position 7`.
* __Inventory Failure__: `book FL-101 Economy` (when full) $\rightarrow$ `Error: Inventory: Seat class capacity exceeded [INV_FULL]`.
* __Validation Failure__: `checkin NULL-00` $\rightarrow$ `Error: Boarding: PNR record not found [VAL_PNR_MISSING]`.

---

## 7. Operational Troubleshooting
### __Build Failures__
* __"C++17 not supported"__: Update your compiler to a modern version (GCC 9+ or Clang 8+).
* __"Undefined reference"__: Ensure you are including all `.cpp` files in your compile command (use `*.cpp`).

### __Runtime Anomalies__
* __No Flights Found__: Ensure you are using valid __3-letter IATA codes__. The registry is case-sensitive (use uppercase ADD, not add).
* __Session State__: Note that the baseline system is __Memory-Resident__; closing the REPL will reset all flight and reservation data.

---

__Operational Note:__ This system is designed for __Resilience__. If a command produces an error, the __REPL Controller__ will automatically recover, allowing the user to attempt a corrected command without restarting the application.