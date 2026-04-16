
---

# Comprehensive Glossary: Airline Management System

### __Abstract Syntax Tree (AST)__
A hierarchical tree representation of the logical structure of a command or expression after it has been parsed. In the context of an __Airline Management System__, an __AST__ allows the engine to decompose complex flight queries—such as multi-city filtering with price constraints—into a machine-readable format that preserves operation order before execution.

### __Associativity__
The directional rule that dictates how operations of equal __precedence__ are grouped in the absence of parentheses. For instance, in flight distance calculations or fare subtractions, __associativity__ ensures that a sequence like `Total - Tax - Discount` is processed from left to right, yielding the mathematically correct `(Total - Tax) - Discount`.

### __Contract__
A rigorous, explicit agreement regarding a module’s behavior, defining its required inputs, expected outputs, and the specific __Status Object__ it will return upon failure. Adhering to a __contractual__ model ensures that when a __Booking Controller__ calls the __Database Module__, both components have a shared understanding of how to handle "Seat Unavailable" versus "System Offline".

### __Lexical Analysis (Tokenization)__
The first phase of the processing pipeline, where a raw string of input characters is converted into a sequence of __Tokens__. For an airline system, this involves identifying meaningful units such as __Flight Numbers__ (e.g., "AA123"), __Date Strings__, and __Airport Codes__ (e.g., "LHR") while discarding irrelevant whitespace or invalid characters.

### __Modular Decomposition__
The high-level architectural practice of partitioning a complex system into focused, independent, and interchangeable components. By utilizing __modular decomposition__, we ensure that a change in the __UI/REPL__ logic does not inadvertently corrupt the internal __Flight Evaluation__ logic, making the system easier to test and scale.

### __Post-order Evaluation__
A recursive strategy for traversing a tree where child nodes are fully processed before the parent operation is applied. In our system, this is used to calculate final ticket prices by first evaluating all base fares and surcharges (children) before applying the final taxes or discounts (parent node).

### __Precedence__
The set of rules determining the priority of different operations. In data filtering, __precedence__ ensures that logical "AND" operations are evaluated before "OR" operations, allowing a user to search for "Flights to Paris AND under $500" as a single, coherent unit.

### __Read-Eval-Print Loop (REPL)__
A continuous interactive environment that repeatedly reads user input, evaluates it against the system logic, and prints the result back to the console. The __REPL__ serves as the primary interface for administrators to interact with the __Airline Management System__ in real-time.


### __Recursive Descent Parser__
A top-down parsing technique implemented through a set of mutually recursive functions that directly mirror the project's formal grammar. This allows the system to validate complex schedules by breaking them down from a high-level `parse_expression` to granular units like `parse_term` and `parse_factor`.

### __Status Object__
A dedicated data structure (struct) used to communicate the outcome of a function call, containing a success flag (`ok`), a detailed error message, and the specific position of the failure. This is our primary mechanism for __Error Propagation__, replacing standard exceptions to make failure flows explicit and visible during the development process.

### __Syntax Analysis (Parsing)__
The process of validating a sequence of __Tokens__ against the system’s grammatical rules to ensure they form a meaningful command. This stage is responsible for detecting errors such as "Missing Destination" or "Invalid Date Order" and constructing the __AST__ for subsequent evaluation.

### __Token__
The smallest, indivisible lexical unit recognized by the system. Within our software, __tokens__ represent the building blocks of user commands, such as numeric values, operators, parentheses, or the __End marker__ that signals a complete input string.

---
