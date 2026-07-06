# Day 1 — V8 Engine & JIT Compilation

## What is V8?

V8 is a JavaScript engine written in C++ by Google.
It runs in two environments:

- **Chrome / Edge** — executes JS in the browser (has access to DOM, fetch, localStorage)
- **Node.js** — executes JS on the server (has access to files, network, processes)

Other engines: SpiderMonkey (Firefox), JavaScriptCore (Safari). Same idea, different implementation.

**Core problem V8 solves:** your JS file is plain text. A CPU only understands binary machine instructions. V8 is the translator between your code and the processor.

---

## The Pipeline — 4 Stages

```
Your code  →  Parser  →  Ignition  →  TurboFan  →  CPU
 (text)        (AST)    (bytecode)  (machine code)
```

---

## Stage 1 — Parser builds the AST

The parser works in two steps:

**Step 1 — Tokenizer** breaks source text into tokens:
```
function add(a, b) { return a + b; }

→ KEYWORD(function)  IDENTIFIER(add)  PUNCT(()
  IDENTIFIER(a)      PUNCT(,)         IDENTIFIER(b)
  PUNCT())           PUNCT({)         KEYWORD(return)
  IDENTIFIER(a)      PUNCT(+)         IDENTIFIER(b)
  PUNCT(;)           PUNCT(})
```

**Step 2 — Parser** builds an Abstract Syntax Tree (AST) from tokens:
```
FunctionDeclaration
  ├── name: "add"
  ├── params: ["a", "b"]
  └── body: ReturnStatement
              └── BinaryExpression (operator: "+")
                    ├── left:  Identifier "a"
                    └── right: Identifier "b"
```

> **SyntaxError = parser failure.** If the parser can't build a valid tree,
> it throws SyntaxError — before any code executes.

---

## Stage 2 — Ignition produces bytecode

Ignition is an **interpreter**. It reads the AST and produces **bytecode** —
simplified, portable instructions that V8 understands (not the CPU directly).

```
// add(a, b) as bytecode:
LdaNamedProperty a    // load 'a' into accumulator
Star r0               // store in register r0
LdaNamedProperty b    // load 'b' into accumulator
Add r0                // add register r0 to accumulator
Return                // return accumulator value
```

Bytecode is fast to generate and fast enough for code that runs infrequently.

**Ignition also collects type feedback.** On every call it records the types
of arguments. After 1000 calls of `add(1, 2)` it knows: "a and b are always
`number`". This data is passed to TurboFan.

---

## Stage 3 — TurboFan and JIT compilation

When a function executes often enough it becomes **hot**.
Ignition hands it off to **TurboFan** — the JIT compiler.

**JIT (Just-In-Time)** means compilation to machine code happens
*while the program is running*, not before.

TurboFan uses type feedback to make assumptions and skip runtime type checks:

```javascript
// General code (what Ignition runs):
if (typeof a === 'number' && typeof b === 'number') { ... }
else if (typeof a === 'string') { ... }
// ...handles every possible type

// Optimized machine code (what TurboFan emits for number + number):
MOVSD xmm0, [a]   // load float64 a
ADDSD xmm0, [b]   // add float64 b
RET               // return — no type checks at all
```

**Speed difference: 10–100× faster than the interpreter.**

---

## Deoptimization

TurboFan optimizes based on **assumptions**. If an assumption breaks,
the optimized code is **thrown away immediately**.

```javascript
function add(a, b) { return a + b; }

// 10 000 calls with numbers → TurboFan optimizes for number + number
for (let i = 0; i < 10000; i++) add(i, i + 1);

// Suddenly a different type:
add(1, "hello"); // ← DEOPTIMIZATION
                 // TurboFan discards the machine code
                 // V8 falls back to Ignition / interpreter
                 // if the function goes hot again — full cycle repeats
```

---

## Practical Rules

**1. Keep types consistent inside functions**
```javascript
// ✗ bad — mixed types across calls → deoptimization
function process(val) {
  if (typeof val === 'string') return val.toUpperCase();
  if (typeof val === 'number') return val * 2;
}
process("abc"); process(123); // deopt!

// ✓ good — one function, one type
function processString(val) { return val.toUpperCase(); }
function processNumber(val) { return val * 2; }
```

**2. Don't mix types in arrays**
```javascript
const fast = [1, 2, 3];        // PACKED_SMI_ELEMENTS — fastest
const slow = [1, 2, "three"];  // PACKED_ELEMENTS — slower
const hole = [1, , 3];         // HOLEY_SMI_ELEMENTS — slower (gap!)
```

**3. Initialize objects in one place**
```javascript
// ✗ V8 has to change the object's internal "shape" each time
const obj = {};
obj.x = 1;
obj.y = 2;

// ✓ one shape from the start — V8 optimizes field access
const obj = { x: 1, y: 2 };
```

---

## Connection to TypeScript

TypeScript enforces type consistency at compile time.
This maps directly to V8 stability:

- TS type error → potential deoptimization prevented
- Consistent types → TurboFan can optimize aggressively and keep the optimization
- `any` → V8 can't make assumptions → slower code

---

## Key Terms

| Term | Definition |
|---|---|
| **V8** | JavaScript engine by Google, runs in Chrome and Node.js |
| **AST** | Abstract Syntax Tree — in-memory tree structure of your code |
| **Bytecode** | Portable V8 instructions, fast to generate |
| **Ignition** | V8's interpreter, executes bytecode, collects type feedback |
| **TurboFan** | V8's JIT compiler, generates optimized machine code |
| **Hot function** | A function called often enough to be worth JIT compiling |
| **Type feedback** | Data Ignition collects about argument types on each call |
| **Deoptimization** | Discarding machine code when type assumptions break |
| **JIT** | Just-In-Time — compilation happens during execution, not before |

---

## Questions to Think About

- Why is `[1, 2, 3]` faster than `[1, 2, "three"]`?
- What happens on the very first call to a function vs the 10 000th?
- Why does TypeScript's `any` type hurt performance?
