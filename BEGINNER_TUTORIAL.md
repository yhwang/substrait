# Substrait Repository - Beginner Tutorial

## 🎯 What is Substrait?

**Substrait** is a cross-language specification for describing data compute operations. Think of it as a "universal translator" for query plans - it allows different data processing systems to communicate and exchange query execution plans in a standardized way.

### Key Analogy
- **Apache Arrow** standardizes how data is represented in memory (the "what")
- **Substrait** standardizes how operations on data are described (the "how")

### The Problem Substrait Solves

Without Substrait, every data system needs custom integrations with every other system. With Substrait:
- ✅ Each system only needs to support Substrait
- ✅ Instantly compatible with the entire ecosystem
- ✅ Upgradable components (swap in a faster engine!)
- ✅ Heterogeneous environments (mix different execution engines)

---

## 🏗️ Repository Structure

### **Core Components**

#### 1. **`proto/`** - Protocol Buffer Definitions (Source of Truth)
The heart of Substrait - these `.proto` files define the specification:
- **`plan.proto`** - Main plan structure and versioning
- **`algebra.proto`** - Relations (operations) and expressions
- **`type.proto`** - Complete type system definitions
- **`extensions/extensions.proto`** - Extension mechanisms for custom functions

#### 2. **`extensions/`** - Standard Function Definitions (YAML)
Pre-defined functions that systems can use:
- **`functions_arithmetic.yaml`** - Math operations (add, subtract, multiply, divide)
- **`functions_arithmetic_decimal.yaml`** - Decimal-specific arithmetic
- **`functions_comparison.yaml`** - Comparisons (equal, greater_than, is_null)
- **`functions_string.yaml`** - String operations (concat, substring, upper, lower)
- **`functions_boolean.yaml`** - Boolean logic (and, or, not, xor)
- **`functions_datetime.yaml`** - Date/time operations
- **`functions_set.yaml`** - Set operations (index_in, etc.)
- Many more specialized function sets

#### 3. **`site/docs/`** - Comprehensive Documentation
Your learning hub:
- **`tutorial/sql_to_substrait.md`** - ⭐ **START HERE!** Hands-on tutorial building a complete plan
- **`spec/specification.md`** - Full specification overview
- **`types/`** - Type system documentation
  - `type_system.md` - Core type concepts
  - `type_classes.md` - Simple, compound, and user-defined types
  - `type_variations.md` - Physical type variations
- **`expressions/`** - Expression documentation
  - `scalar_functions.md` - Regular functions
  - `aggregate_functions.md` - Aggregation operations
  - `field_references.md` - How to reference columns
- **`relations/`** - Relation (operation) documentation
  - `logical_relations.md` - Filter, join, aggregate, etc.
  - `physical_relations.md` - Physical execution variants
- **`serialization/`** - How plans are serialized
  - `binary_serialization.md` - Protobuf format
  - `text_serialization.md` - Human-readable format

#### 4. **`tests/`** - Test Cases
Comprehensive test suite for function implementations:
- **`cases/`** - Organized by function category
  - `boolean/` - Boolean function tests
  - `comparison/` - Comparison function tests
  - `string/` - String function tests
  - `datetime/` - Date/time function tests
  - And many more...
- **`coverage/`** - Test coverage analysis tools

#### 5. **`grammar/`** - ANTLR Grammar Definitions
Parser definitions for Substrait's text format:
- `SubstraitLexer.g4` - Lexical analysis
- `SubstraitType.g4` - Type parsing
- `FuncTestCaseLexer.g4` - Test case parsing
- `FuncTestCaseParser.g4` - Test case grammar

#### 6. **Root Level Files**
- **`README.md`** - Project overview
- **`go.mod`** - Go module definition (Go bindings)
- **`pyproject.toml`** - Python project configuration
- **`core.go`** - Core Go implementation

---

## 📚 Core Concepts

### 1. **Plans**
A Substrait plan describes a complete query execution strategy. It's the top-level container that includes:

- **Relations** - The operations to perform (filter, join, aggregate, etc.)
- **Expressions** - Calculations and field references
- **Types** - Data type definitions for all fields
- **Extensions** - References to function definitions from YAML files

**Structure:**
```json
{
  "version": "0.x.x",
  "extensionUrns": [...],
  "extensions": [...],
  "relations": [...]
}
```

### 2. **Relations** (Operations)
Relations are the nodes in a query plan tree. Each relation transforms data:

- **Read** - Read from a table, file, or virtual table
- **Filter** - Filter rows based on boolean conditions
- **Project** - Add new computed columns (keeps existing columns!)
- **Join** - Combine data from multiple sources (inner, left, right, outer, etc.)
- **Aggregate** - Group and summarize data (GROUP BY + aggregations)
- **Sort** - Order data
- **Fetch** - Limit/offset operations
- **Set** - Union, intersect, except operations

**Key Insight:** Relations form a tree where data flows from leaf nodes (usually Read) up to the root.

### 3. **Expressions**
Expressions compute values and can be nested to form expression trees:

- **Field References** - Reference columns by numeric index (NOT by name!)
  ```json
  {
    "selection": {
      "directReference": {
        "structField": { "field": 0 }
      }
    }
  }
  ```

- **Literals** - Constant values
  ```json
  {
    "literal": {
      "string": "Hello"
    }
  }
  ```

- **Scalar Functions** - Operations like `add()`, `concat()`, `substring()`
  ```json
  {
    "scalarFunction": {
      "functionReference": 1,
      "arguments": [...]
    }
  }
  ```

- **Aggregate Functions** - Operations like `sum()`, `count()`, `avg()`
- **Window Functions** - Operations like `rank()`, `row_number()`
- **Cast** - Type conversion (special expression type, not a function)

### 4. **Type System**
Substrait has a rich, precise type system:

**Simple Types:**
- Integers: `i8`, `i16`, `i32`, `i64`
- Floating point: `fp32`, `fp64`
- Text: `string`, `varchar`, `fixedchar`
- Binary: `binary`, `fixedbinary`
- Temporal: `date`, `time`, `timestamp`, `interval`
- Other: `bool`, `uuid`

**Compound Types:**
- `struct` - Named fields with different types
- `list` - Ordered collection of same type
- `map` - Key-value pairs

**Parameterized Types:**
- `decimal(precision, scale)` - Fixed-point decimal
- `varchar(length)` - Variable-length string with max length
- `fixedchar(length)` - Fixed-length string

**Nullability:**
Every type includes nullability information:
- `NULLABILITY_REQUIRED` - Cannot be null
- `NULLABILITY_NULLABLE` - Can be null
- `NULLABILITY_UNSPECIFIED` - Nullability unknown

### 5. **Extensions**
Functions are defined externally in YAML files and referenced by URI:

**Extension URI Format:**
```
extension:io.substrait:functions_arithmetic
```

**In a Plan:**
1. Declare the extension URI with an anchor ID
2. Declare each function used with a function anchor ID
3. Reference functions by their anchor ID in expressions

**Example:**
```json
{
  "extensionUrns": [
    {
      "extensionUrnAnchor": 1,
      "urn": "extension:io.substrait:functions_arithmetic"
    }
  ],
  "extensions": [
    {
      "extensionFunction": {
        "extensionUrnReference": 1,
        "functionAnchor": 1,
        "name": "add"
      }
    }
  ]
}
```

### 6. **Schemas and Field References**
**Critical Concept:** Substrait uses numeric indices, not names!

**NamedStruct:**
Schemas are represented as a `NamedStruct` with:
- `names` - Array of field names (depth-first for nested types)
- `struct` - The struct type containing all field types

**Field Indexing:**
- Fields are numbered starting from 0
- For nested structs, subfields appear in depth-first order
- Field references in expressions use these numeric indices

**Example:**
```sql
CREATE TABLE products (
  product_id: i64,
  details: struct<manufacturer: string, year: i32>,
  name: string
)
```

Field indices:
- 0: `product_id`
- 1: `details`
- 2: `manufacturer` (subfield of details)
- 3: `year` (subfield of details)
- 4: `name`

---

## 🚀 Getting Started - Recommended Learning Path

### Step 1: Read the Hands-On Tutorial ⭐
**File:** `site/docs/tutorial/sql_to_substrait.md`

This is THE best place to start! The tutorial walks you through building a complete Substrait plan from this SQL query:

```sql
SELECT
  product_name,
  product_id,
  sum(quantity * price) as sales
FROM orders
INNER JOIN products
  ON orders.product_id = products.product_id
WHERE
  INDEX_IN("Computers", categories) IS NULL
GROUP BY
  product_name,
  product_id
```

**What You'll Learn:**
1. How to define types and schemas
2. Building expressions step-by-step
3. Constructing relations (Read, Filter, Join, Aggregate)
4. Understanding field indices and how they change through relations
5. Using the emit property to reorder/subset columns
6. Assembling the complete plan with extensions

**Time Investment:** 30-45 minutes
**Outcome:** Complete understanding of how Substrait plans work

### Step 2: Explore the Specification
**File:** `site/docs/spec/specification.md`

Get the big picture:
- What components are complete vs. in progress
- Links to detailed documentation for each component
- Technology principles and design philosophy

### Step 3: Deep Dive into Components

Based on your interests:

**For Type System:**
- `site/docs/types/type_system.md`
- `site/docs/types/type_classes.md`
- `proto/substrait/type.proto` (source of truth)

**For Expressions:**
- `site/docs/expressions/scalar_functions.md`
- `site/docs/expressions/aggregate_functions.md`
- `site/docs/expressions/field_references.md`
- `proto/substrait/algebra.proto` (Expression message)

**For Relations:**
- `site/docs/relations/logical_relations.md`
- `site/docs/relations/basics.md`
- `proto/substrait/algebra.proto` (Rel message)

### Step 4: Study Function Definitions
**Directory:** `extensions/`

Pick a function category and examine the YAML:
- See how functions are defined with signatures
- Understand function options (overflow behavior, rounding, etc.)
- Learn about function overloading (same name, different argument types)

**Example:** Open `extensions/functions_arithmetic.yaml` and study the `add` function.

### Step 5: Explore Examples
**Directory:** `site/examples/`

- `extensions/` - Example extension definitions
- `types/` - Example user-defined types

### Step 6: Try the Validator Tool
**Tool:** [substrait-validator](https://github.com/substrait-io/substrait-validator)

Install and use it to validate and visualize plans:
```bash
# Install (requires Rust)
cargo install substrait-validator

# Validate and generate HTML report
substrait-validator plan.json --out-file output.html
```

The validator shows:
- Schema at each relation
- Field indices at each point
- Type information
- Validation errors

### Step 7: Read the FAQ
**File:** `site/docs/faq.md`

Answers common questions like:
- Why does project keep existing columns?
- Where are field names represented?
- What is the post-join filter for?

---

## 💡 Key Design Principles

### 1. Index-Based, Not Name-Based
**Why:** Eliminates ambiguity and makes plans easier to process programmatically.

Fields are always referenced by position (0, 1, 2...), never by name. Names are only present:
- In Read relation base schemas (for source lookup)
- In Root relation output (for final result naming)
- Optionally as hints in `RelCommon.output_names` (for debugging/round-tripping)

### 2. Protobuf Serialization
**Binary Format:** Compact and efficient for production use
**JSON Format:** Human-readable for debugging and testing

The JSON you see in tutorials is Protobuf's JSON representation, not an official Substrait text format (though one is planned).

### 3. Extension-Friendly
Custom functions and types can be added via:
- YAML extension files (for reusable functions)
- User-defined types (for custom data types)
- Embedded functions (for inline implementations)

### 4. Language-Agnostic
Works with any language supporting Protocol Buffers:
- C++, Java, Python, Go, Rust, JavaScript, C#, and more
- No language-specific assumptions in the spec

### 5. Semantic Clarity
Every operation has unambiguous semantics:
- Detailed documentation in proto comments
- Test cases demonstrating behavior
- Clear specification of edge cases

### 6. Separation of Concerns
- **Logical Plans:** What to compute (Substrait's focus)
- **Physical Plans:** How to compute (implementation-specific)
- **Data Format:** How data is represented (Arrow's focus)

---

## 🔧 Language Support in This Repository

### Go
- **Files:** `go.mod`, `go.sum`, `core.go`, `core_test.go`
- **Purpose:** Go language bindings and utilities
- **Module:** `github.com/substrait-io/substrait`

### Python
- **Files:** `pyproject.toml`, `requirements.txt`, `tests/`
- **Purpose:** Test infrastructure and coverage analysis
- **Tools:** pytest, black (formatter), ANTLR parser

### Protocol Buffers
- **Directory:** `proto/`
- **Purpose:** Language-neutral specification
- **Generated Code:** Not included in repo (generate for your language)

---

## 📖 Common Use Cases

### 1. SQL Parser → Execution Engine
**Scenario:** Parse SQL with Apache Calcite, execute with Arrow C++ compute kernels

**Flow:**
```
SQL Query → Calcite Parser → Substrait Plan → Arrow Execution
```

**Benefit:** Decouple parsing from execution, use best-in-class components

### 2. Cross-System Views
**Scenario:** Define a view once, use it in Spark, Trino, and DuckDB

**Flow:**
```
View Definition → Substrait Plan → Store in Iceberg → Use in any engine
```

**Benefit:** Consistent view semantics across all systems

### 3. Query Federation
**Scenario:** Send the same query to multiple engines, compare results

**Flow:**
```
Substrait Plan → Engine A (Postgres)
              → Engine B (DuckDB)
              → Engine C (DataFusion)
```

**Benefit:** Consistent interpretation, easy benchmarking

### 4. Alternative Frontends
**Scenario:** Execute Pandas operations inside a database

**Flow:**
```
Pandas API → Substrait Plan → SingleStore/Postgres/etc.
```

**Benefit:** Familiar API with database performance

### 5. Plan Visualization
**Scenario:** Build a D3-based query plan visualizer

**Flow:**
```
Substrait Plan → Visualization Tool → Interactive Diagram
```

**Benefit:** Universal visualizer works with any Substrait producer

### 6. Query Optimization
**Scenario:** Build a standalone query optimizer service

**Flow:**
```
Unoptimized Plan → Optimizer Service → Optimized Plan
```

**Benefit:** Reusable optimization logic across systems

---

## 🎓 Additional Learning Resources

### Official Resources
1. **Website:** [substrait.io](https://substrait.io)
2. **GitHub:** [github.com/substrait-io/substrait](https://github.com/substrait-io/substrait)
3. **Specification:** `site/docs/spec/specification.md`

### In This Repository
1. **Tutorial:** `site/docs/tutorial/sql_to_substrait.md` ⭐
2. **FAQ:** `site/docs/faq.md`
3. **About:** `site/docs/about.md`
4. **Proto Definitions:** `proto/substrait/*.proto` (with detailed comments)

### External Tools
1. **substrait-validator** - Validate and visualize plans
2. **Apache Calcite** - SQL parser that can produce Substrait
3. **Apache Arrow** - Data format that pairs well with Substrait

### Community
- **GitHub Issues** - Ask questions, report bugs
- **Governance:** `site/docs/governance.md`
- **Powered By:** `site/docs/community/powered_by.md`

---

## ⚡ Quick Example Walkthrough

Let's trace a simple query through Substrait:

### SQL Query
```sql
SELECT product_name, quantity
FROM orders
WHERE quantity > 10
```

### Logical Plan
```
Project(product_name, quantity)
  └─ Filter(quantity > 10)
      └─ Read(orders)
```

### Substrait Representation (Conceptual)

**1. Read Relation:**
```json
{
  "read": {
    "namedTable": { "names": ["orders"] },
    "baseSchema": {
      "names": ["product_id", "quantity", "order_date", "price"],
      "struct": { "types": [...] }
    }
  }
}
```

**2. Filter Relation:**
```json
{
  "filter": {
    "input": { /* Read relation */ },
    "condition": {
      "scalarFunction": {
        "functionReference": 1,  // References "greater_than"
        "arguments": [
          { "selection": { "field": 1 } },  // quantity (index 1)
          { "literal": { "i32": 10 } }
        ]
      }
    }
  }
}
```

**3. Root Relation (with column selection):**
```json
{
  "root": {
    "names": ["product_name", "quantity"],
    "input": {
      "filter": { /* Filter relation */ },
      "common": {
        "emit": {
          "outputMapping": [0, 1]  // Select first two columns
        }
      }
    }
  }
}
```

**4. Complete Plan:**
```json
{
  "version": { "major": 0, "minor": 42, "patch": 0 },
  "extensionUrns": [
    {
      "extensionUrnAnchor": 1,
      "urn": "extension:io.substrait:functions_comparison"
    }
  ],
  "extensions": [
    {
      "extensionFunction": {
        "extensionUrnReference": 1,
        "functionAnchor": 1,
        "name": "gt"
      }
    }
  ],
  "relations": [
    { "root": { /* Root relation */ } }
  ]
}
```

---

## 🎯 Next Steps

### For Beginners
1. ✅ Read this tutorial
2. 📖 Work through `site/docs/tutorial/sql_to_substrait.md`
3. 🔍 Explore the proto files in `proto/substrait/`
4. 🧪 Look at test cases in `tests/cases/`

### For Implementers
1. 📚 Study the specification thoroughly
2. 🛠️ Use substrait-validator to test your plans
3. 🔌 Implement producer or consumer for your system
4. 🤝 Join the community and contribute

### For Contributors
1. 📋 Check GitHub issues for areas needing help
2. 📝 Improve documentation
3. ➕ Add new function definitions
4. 🧪 Contribute test cases

---

## 🤔 Common Questions

### Q: Why not just use SQL?
**A:** SQL is great for humans but lacks the precision needed for systems. Substrait provides:
- Unambiguous semantics
- Machine-readable format
- Support for optimizations and transformations
- Cross-system compatibility

### Q: How does Substrait relate to Apache Arrow?
**A:** They're complementary:
- **Arrow:** Standardizes data representation (memory format)
- **Substrait:** Standardizes operations on data (query plans)
- Together they enable complete interoperability

### Q: Can I use Substrait with my existing system?
**A:** Yes! You can:
- Build a producer (generate Substrait from your system)
- Build a consumer (execute Substrait in your system)
- Or both!

### Q: Is Substrait production-ready?
**A:** The core specification is stable and used in production by several projects. Check `site/docs/community/powered_by.md` for examples.

---

## 📝 Summary

**Substrait is:**
- ✅ A cross-language specification for query plans
- ✅ Based on Protocol Buffers for efficiency
- ✅ Extensible with custom functions and types
- ✅ Designed for interoperability between data systems
- ✅ Complementary to Apache Arrow

**This repository contains:**
- 📋 The specification (proto files)
- 📚 Comprehensive documentation
- 🔧 Standard function definitions
- 🧪 Test cases and examples
- 🛠️ Language bindings (Go, Python)

**Start your journey:**
1. Read `site/docs/tutorial/sql_to_substrait.md`
2. Explore the proto definitions
3. Try the validator tool
4. Join the community!

---

**Happy Learning! 🚀**

For questions or contributions, visit [github.com/substrait-io/substrait](https://github.com/substrait-io/substrait)