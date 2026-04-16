

# System Requirements Specification: Airline Management System

## 1. Executive Scope & Objective
The primary objective of the __Airline Management System__ is to provide a robust, command-line driven interface for the evaluation, scheduling, and management of aeronautical data. The system is engineered as a high-integrity __C++17__ application that prioritizes __explicit error reporting__, __modular architecture__, and __deterministic performance__. By utilizing a custom __Status-based error model__ rather than standard exceptions, the system ensures that every failure state is traceable and manageable within a real-time __REPL__ environment.

---

## 2. Functional Requirements (FR)

### __FR-001: Interactive Command Processing__
* The system shall implement a __Read-Eval-Print Loop (REPL)__ to facilitate continuous user interaction.
* It must process input strings line-by-line from the standard input stream.
* The __REPL__ session shall persist until a termination signal (EOF) or an explicit `:quit` command is received.

### __FR-002: System Command Suite__
* __:help__: The system shall provide a comprehensive help module detailing available commands and syntax guidance.
* __:quit__: The system shall support a graceful shutdown sequence to ensure all resources are released.

### __FR-003: Lexical Tokenization__
* The __Tokenizer__ shall decompose raw input into a sequence of meaningful units, including __Integer Literals__, __Arithmetic Operators__ (`+`, `-`, `*`, `/`), and __Parentheses__.
* The system must intelligently ignore arbitrary __whitespace__ while maintaining strict tracking of character positions for error reporting.
* Every token stream must conclude with an explicit __End Token__ to signify a completed command.


### __FR-004: Syntactic Analysis & AST Construction__
* The __Parser__ must consume the token stream and construct an __Abstract Syntax Tree (AST)__ that represents the logical hierarchy of the expression.
* It must strictly enforce __Operator Precedence__ and __Associativity__ rules to ensure mathematical accuracy.
* Support for __Unary Minus__ operations must be implemented within the factor-level logic.

### __FR-005: Computational Evaluation__
* The __Evaluator__ module shall perform a recursive, post-order traversal of the __AST__.
* All numerical results shall be computed using the `long long` integer domain.
* Critical runtime failures, such as __Division by Zero__, must be caught and converted into a failure status rather than allowing a system crash.

### __FR-006: Diagnostic Integrity__
* Every module within the system must return a __Status Object__ containing a success flag, a descriptive message, and the exact source position of any failure.
* The __Controller__ layer shall format these diagnostics into a user-friendly output that pinpoints the error location.

---

## 3. Non-Functional Requirements (NFR)

### __NFR-001: Architectural Transparency__
* The codebase must prioritize __Clarity and Teachability__, favoring explicit __Contracts__ and readable logic over opaque optimizations.

### __NFR-002: Modular Decoupling__
* All system modules (Tokenizer, Parser, Evaluator) must remain strictly __independent__.
* Logic must be separated into header (`.h`) and source (`.cpp`) files to maintain a clean public API.

### __NFR-003: Technical Constraints & Portability__
* The system is restricted to the __C++17__ standard to ensure compatibility with modern toolchains like __g++__ and __clang++__.
* No external libraries beyond the __C++ Standard Library__ are permitted in the baseline implementation.

---

## 4. Development Constraints & Lifecycle

* __Language__: C++17.
* __Numeric Domain__: 64-bit signed integers (`long long`).
* __Error Protocol__: __Status Structs__ only; no `try-catch` blocks in the baseline path.
* __Parser Strategy__: __Recursive Descent__.


---

## 5. Acceptance Criteria by Development Stage

### __Stage 1: Foundation__
* The project successfully compiles with strict flags (`-Wall -Wextra -pedantic`).
* The __REPL__ successfully routes the `:help` and `:quit` commands.

### __Stage 2: Lexical Integrity__
* The __Tokenizer__ correctly identifies all valid tokens and reports errors for illegal characters or numerical overflows.

### __Stage 3: Syntactic Validity__
* The __Parser__ builds accurate __ASTs__ and correctly identifies syntax errors like "Unexpected Token" or "Missing Parenthesis".

### __Stage 4: Computational Accuracy__
* The __Evaluator__ produces correct results for complex nested expressions and gracefully handles mathematical edge cases.

### __Stage 5: Error Resilience__
* The system remains __stable__ and allows for immediate input recovery following any module-level error.

---

