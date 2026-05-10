# 📄 Chapter 12 — Runtime Environments

**Tags:** #compiler #runtime #activation-record #stack-frame #calling-convention #static-allocation #stack-allocation #NCKU
**Links:** [[Intermediate Code Generation]], [[Symbol Tables]], [[Code Generation]], [[Procedure Calls]]

---

## 🎯 The "Elevator Pitch"
> A compiler doesn't just translate code — it must decide *where every piece of data lives in memory while the program runs*. The **runtime environment** is the complete system that manages this: how memory is divided (code, stack, heap, static), what information is bundled per function call (activation records), and the exact step-by-step protocol for calling and returning from procedures. Get this wrong and recursion breaks, local variables stomp on each other, and programs crash in mysterious ways.

---

## 🧠 Part 1 — Memory Organization at Runtime

### The Four Regions of Runtime Memory

Every compiled program organizes its memory into conceptually distinct regions:

```
High addresses
┌──────────────────────────────────┐
│           STACK                  │  ← grows downward
│    (activation records /         │
│       stack frames)              │
│                                  │
│    . . . . . . . . . . .        │
│         free space               │
│    . . . . . . . . . . .        │
│                                  │
│           HEAP                   │  ← grows upward
│    (dynamic / malloc data)       │
├──────────────────────────────────┤
│     STATIC / GLOBAL DATA         │  ← fixed at compile time
├──────────────────────────────────┤
│         CODE AREA                │  ← read-only machine instructions
└──────────────────────────────────┘
Low addresses
```

| Region | What Lives Here | When Allocated | Size Known At |
|---|---|---|---|
| **Code area** | Compiled instructions, one entry point per procedure | Load time | Compile time |
| **Static/global data** | Global variables, `static` locals, constants | Load time | Compile time |
| **Stack** | Activation records (local vars, params, saved registers) | Call time | Usually compile time |
| **Heap** | `malloc`/`new` dynamic allocations | Runtime (`malloc`) | Runtime |

### 💡 The Stack Grows Down, Heap Grows Up — Why?

They share the same pool of "free space" between them and grow toward each other. This lets both expand as needed without wasting memory with a fixed partition. The OS or runtime raises a "stack overflow" error when they collide.

### Code Area: Entry Points

```
Code area:
  [entry point 1] ── code for procedure 1 ──
  [entry point 2] ── code for procedure 2 ──
  [entry point 3] ── code for procedure 3 ──
  ...
  [entry point N] ── code for procedure N ──
```

Each procedure has a fixed, unique **entry point** address known at compile/link time. A `call p` instruction simply jumps to this address.

---

## 🧠 Part 2 — Activation Records (Stack Frames): The Core Concept

### What is an Activation?

> Each execution of a procedure is called an **activation**. If a procedure is recursive, multiple activations of the same procedure may be alive simultaneously — each with its own set of local variables.

### What is an Activation Record?

> An **activation record** (also called a **stack frame**) is a contiguous block of memory that stores all the state needed for one execution of one procedure.

Think of it as a C struct that the compiler builds on the stack at every function call:

```c
// Conceptual C struct for one activation record
struct ActivationRecord {
    // ─── Caller's responsibility (pushed before call) ───────────
    int         param_1;         // actual parameter 1
    int         param_2;         // actual parameter 2
    // ...
    int         param_n;         // actual parameter n
    int         arg_count;       // number of parameters (= n)
    void       *return_address;  // where to jump after return (l1)
    void       *return_value;    // space for callee to store result
    // ─── Callee's responsibility (set up by procbegin) ──────────
    void       *old_SP;          // saved SP of caller (control link)
    // (optional) void *access_link;  // for non-local variable access
    // (optional) void *saved_regs;   // saved machine registers
    int         local_data[];    // local variables + temporaries
};
```

### The Seven Fields of an Activation Record

The Dragon Book specifies these fields (not all compilers use all of them):

| Field | Contents | Direction from SP |
|---|---|---|
| **Temporaries** | Compiler-generated temporaries that can't go in registers | Negative (below SP) |
| **Local data** | Local variables declared in the procedure | Negative (below SP) |
| **Saved machine status** | Saved registers, old PC, etc. | Around SP |
| **Access link** (optional) | Pointer to AR of lexically enclosing scope (for non-local data) | At/near SP |
| **Control link** (optional) | Pointer to AR of caller (= old SP) | At SP |
| **Return value** | Space reserved for the function's return value | Positive (above SP) |
| **Actual parameters** | The argument values passed by the caller | Positive (above SP) |

### Stack Layout: Registers SP and TOP

Two key registers manage the runtime stack:

- **TOP** — points to the **lowest-numbered used location** on the stack (the absolute top of all active ARs)
- **SP** (Stack Pointer) — points to a **specific field** inside the *current* activation record (the old SP field, in this chapter's convention)

```
Low address
    0
    |
    |   ┌──────────────────────────┐
    |   │  Static area for program │
    |   │  and global data         │
    |   └──────────────────────────┘
    |
    |   ┌──────────────────────────┐ ← Activation record for P (first caller)
    |   │  Extra storage for P     │
    |   │  (display pointers, etc) │
    |   │  ─────────────────────── │ ← (fp for P)
    |   │  Local data for P        │
    |   │  Old SP                  │ ← SP points here when P is active
    |   │  Return value            │
    |   │  Return address          │
    |   │  Argument count          │
    |   │  Parameters of P         │
    |   └──────────────────────────┘
    |   ┌──────────────────────────┐ ← Activation record for Q (called by P)
    |   │  Extra storage for Q     │
    |   │  Local + temp data for Q │
    |   │  Old SP                  │ ← SP points here when Q is active
    |   │  Return value            │
    |   │  Return address          │
    |   │  Argument count          │
    |   │  Parameters of Q         │
    |   └──────────────────────────┘
    |   ┌──────────────────────────┐ ← Activation record for R (called by Q)
    |   │  Extra storage for R     │
    |   │  Local + temp data for R │
    |   │  Old SP (= Q's SP)       │ ← SP points here when R is active
    |   │  Return value            │
    |   │  Return address          │
    |   │  Argument count          │
    |   │  Parameters of R         │
    |   └──────────────────────────┘ ← TOP (top of stack)
    |
High address (2n - 1)
```

### Accessing Local Variables and Parameters

With SP pointing to the **old SP field** of the current AR:

- **Local data**: accessed by **negative offsets** from SP
  - `local_var_x` lives at `x[SP]` where `x` is a negative integer
- **Parameters**: accessed by **positive offsets** from SP
  - The `i`th parameter is at `(4 + n - i)[SP]`
  - (assuming each entry takes one unit of space, and there are `n` parameters)

```
SP + (-3) : temp_t2         ← local/temp data (negative offsets)
SP + (-2) : temp_t1
SP + (-1) : local_var_x
SP + (0)  : OLD_SP          ← SP points HERE
SP + (1)  : return_value
SP + (2)  : return_address
SP + (3)  : arg_count (= n)
SP + (4)  : param_n         ← last parameter
SP + (5)  : param_{n-1}
  ...
SP + (4+n-1): param_1       ← first parameter (positive offsets)
```

---

## 🧠 Part 3 — Static vs. Dynamic Storage Allocation

### Fully Static Runtime Environments

> In a **fully static** environment, *all* memory locations are fixed at compile time and remain unchanged for the entire program execution.

**Requirements for full static allocation:**
- No pointers or dynamic allocation
- Procedures cannot be called recursively (only one activation per procedure ever exists)

**Example language**: Early FORTRAN (pre-FORTRAN 90)

```python
# Fully static: every variable has exactly one fixed address forever
static_layout = {
    'x':    0x1000,   # global variable x always at address 0x1000
    'y':    0x1004,   # y always at 0x1004
    'main_local_a': 0x2000,   # local 'a' of main always at 0x2000
    'foo_local_b':  0x2004,   # local 'b' of foo always at 0x2004
    # foo cannot be called recursively — only one activation ever!
}
```

**Advantage**: Extremely fast — variable access is a single memory reference to a known address.  
**Disadvantage**: No recursion, no dynamic data structures.

### Stack-Based (Dynamic) Allocation

For languages supporting recursion, stack allocation is required. Each procedure call dynamically allocates a new activation record on the stack.

```python
# Stack allocation: each call creates a new frame
call_stack = []

def simulate_call(proc_name, params, local_size):
    """Push a new activation record onto the call stack."""
    frame = {
        'procedure': proc_name,
        'params':    params,
        'locals':    [None] * local_size,
        'return_addr': None,  # filled by calling sequence
        'saved_sp':  len(call_stack) - 1 if call_stack else -1
    }
    call_stack.append(frame)
    return frame

def simulate_return():
    """Pop the top activation record (the callee's frame)."""
    return call_stack.pop()

# Recursive factorial: each call gets its own frame
# fact(3) → fact(2) → fact(1) → fact(0)
# Stack at deepest point: [fact(3), fact(2), fact(1), fact(0)]
# Each has its own 'n' local variable — no interference!
```

---

## 🧠 Part 4 — The Calling Sequence: Step by Step

### Overview: Who Does What?

The work of a procedure call is split between **caller** and **callee**:

| Phase | Actor | Actions |
|---|---|---|
| **Before call** | Caller | Evaluate actual parameters, push them onto stack |
| **At call** | Caller | Push arg count, return address, reserve return value slot, push old SP, jump to callee |
| **Entry (procbegin)** | Callee | Set SP = TOP (point to old SP field), set TOP = SP + local_size |
| **Execute** | Callee | Body of the procedure runs |
| **Return** | Callee | Store return value, restore TOP and SP, jump back to return address |
| **After return** | Caller | Retrieve return value from known position relative to new TOP |

### Calling Sequence in Three-Address Code → Runtime Actions

For a call `p(T1, T2, …, Tn)`, the TAC `param` and `call` instructions translate to:

```
TAC:                           Runtime (at call time):
────────────────────────────────────────────────────────
param T1                  →    push(T1):   TOP = TOP - 1; *TOP = T1
param T2                  →    push(T2):   TOP = TOP - 1; *TOP = T2
...
param Tn                  →    push(Tn):   TOP = TOP - 1; *TOP = Tn
call p, n                 →    push(n):    TOP = TOP - 1; *TOP = n    (arg count)
                               push(l1):   TOP = TOP - 1; *TOP = l1   (return address)
                               push(_):    TOP = TOP - 1              (return value slot)
                               push(SP):   TOP = TOP - 1; *TOP = SP   (save old SP)
                               goto l2     (jump to first instr of p)
```

> **l1** = the address in the **caller** to return to after the call  
> **l2** = the address of the **first instruction** of the called procedure `p`

### Example: `p(A + B*C, D)`

```
TAC generated:                   Runtime stack after each push (grows downward):
─────────────────────────────────────────────────────────────────────────────
T1 = B * C
T2 = A + T1
param T2                   →   push T2         [... | T2 ]
param D                    →   push D          [... | T2 | D ]
call p, 2                  →   push 2          [... | T2 | D | 2 ]
                               push l1         [... | T2 | D | 2 | l1 ]
                               push _          [... | T2 | D | 2 | l1 | _ ]   (return val)
                               push SP_old     [... | T2 | D | 2 | l1 | _ | SP_old ]
                               goto l2
                                         ↑
                                    TOP and SP now point here (old SP field)
```

### `procbegin`: The First Instruction of Every Procedure

When the callee starts executing, it runs `procbegin`, which:

1. Sets `SP = TOP` — now SP points to the old SP field (the "anchor" for offset addressing)
2. Sets `TOP = SP + size_of_p` — reserves space for the callee's locals and temporaries

```python
def procbegin(size_of_p: int):
    """
    First instruction executed when entering a procedure.
    Sets SP to point at old SP field; advances TOP past local data.
    """
    global SP, TOP
    SP  = TOP               # SP now points to the old SP value on the stack
    TOP = SP + size_of_p    # reserve space for local data above SP
    # (remember: stack grows toward lower addresses, so "above" here
    #  means lower memory addresses than SP)
```

**After `procbegin`**, the AR layout from SP looks like:

```
SP + (-size_of_p) ... SP + (-1) :  local data + temporaries  ← callee fills these
SP + (0)                         :  OLD_SP                    ← saved by caller
SP + (1)                         :  return_value              ← callee fills this
SP + (2)                         :  return_address (l1)       ← set by caller
SP + (3)                         :  arg_count (n)             ← set by caller
SP + (4) ... SP + (3+n)          :  actual parameters         ← set by caller
```

---

## 🧠 Part 5 — The Return Sequence: Unwinding the Stack

The `return (expression)` statement compiles to a sequence of TAC instructions that must:
1. Store the return value
2. Restore TOP and SP to the caller's state
3. Jump back to the return address

```python
def return_sequence(expr_value, SP, TOP, memory):
    """
    Simulate the return sequence in three-address code.
    Offsets from SP (positive = above SP = toward lower addresses in this layout).

    Memory layout from SP:
      SP[0]  = OLD_SP (saved caller SP)
      SP[1]  = return value slot
      SP[2]  = return address (l1)
      SP[3]  = arg_count (number of parameters n)
      SP[4..3+n] = actual parameters
    """
    # Step 1: Store the return value into the reserved slot
    memory[SP + 1] = expr_value        # 1[SP] = T  (return value)

    # Step 2: Advance TOP to point at the return address field
    TOP = SP + 2                        # TOP now points to return_address

    # Step 3: Restore SP to the caller's SP (pop the callee's frame)
    SP_new = memory[SP]                 # SP = *SP  (dereference old SP)
    SP = SP_new

    # Step 4: Read the return address
    return_addr = memory[TOP]           # L = *TOP  (get return address l1)

    # Step 5: Advance TOP past return address, then past arg_count + params
    TOP = TOP + 1                       # TOP now points to arg_count field
    arg_count = memory[TOP]             # *TOP = number of parameters
    TOP = TOP + 1 + arg_count          # skip arg_count field + all parameters
    # TOP now restored to the top of the CALLER's extra storage

    # Step 6: Jump back to caller
    goto(return_addr)                   # goto *L (return to l1 in caller)

    return SP, TOP
```

### 💡 Intuition for the Return Sequence

Think of the return sequence as rewinding a cassette tape:
- You played forward (called the procedure, built the AR)
- Now you rewind backward through the same steps in reverse
- After rewinding, the caller is back exactly where it left off — but with the return value accessible just above its own AR

---

## 🧠 Part 6 — Activation Trees: The Call Graph at Runtime

### What is an Activation Tree?

> An **activation tree** represents all procedure activations during a program's execution. Each node is one activation; the parent of node `b` is the activation `a` that called it.

```
Rule: Node a is to the LEFT of node b iff the lifetime of a ends BEFORE b begins.
Rule: Node b is a CHILD of a iff a called b (and b hasn't returned yet when a returns).
```

### Example: `quicksort`

```c
main() {
    int n;
    readarray();
    quicksort(1, n);
}
quicksort(int m, int n) {
    int i = partition(m, n);
    quicksort(m, i-1);
    quicksort(i+1, n);
}
```

Activation tree (for input n=9):

```
                    main
                   /    \
          readarray     quicksort(1,9)
                        /     |     \
              partition(1,9) qs(1,3) qs(5,9)
                             /  \    /    \
                         p(1,3) ... p(5,9) ...
```

### Key Property: Stack = Path from Root to Current Node

At any moment, the runtime stack contains exactly the activation records corresponding to the **path from the root (main) to the currently executing node** in the activation tree.

```python
def simulate_activation_tree():
    """
    The control stack at any point = path from root to current node.
    DFS traversal of the activation tree = execution order.
    """
    # When control is at quicksort(1,3):
    stack = [
        'main',           # root
        'quicksort(1,9)', # called by main
        'quicksort(1,3)', # called by quicksort(1,9) ← currently active
    ]
    # quicksort(5,9) hasn't started yet (it's to the RIGHT in the tree)
    # partition(1,9) has already returned (it's to the LEFT and finished)
```

---

## 🧠 Part 7 — Access Links vs. Control Links

### Control Link (Dynamic Link)

> The **control link** (also called *dynamic link*) is a pointer from the current activation record to the **caller's** activation record. It is used to **restore SP** on return.

- Points to the activation record of the calling procedure
- Used mechanically during the return sequence to "pop" the stack
- Does NOT relate to lexical scope

### Access Link (Static Link)

> The **access link** (also called *static link*) is a pointer to the activation record of the **lexically enclosing** procedure. It is used to access **non-local variables** in lexically scoped languages (Pascal, Algol, nested functions in C-like languages).

```
Example (Pascal-like nested procedures):

procedure outer;
    var x: integer;       ← x lives in outer's AR
    procedure inner;
    begin
        x := x + 1;       ← inner needs to access outer's 'x'
    end;
begin
    ...
end;

AR layout when inner is executing:
┌─────────────────────────┐ ← inner's AR
│ local vars of inner     │
│ OLD_SP (control link)   │ → outer's AR  (who called inner)
│ access link             │ → outer's AR  (lexically enclosing outer)
│ ...                     │
└─────────────────────────┘
        │ access link
        ▼
┌─────────────────────────┐ ← outer's AR
│ x (= inner can read it) │
│ ...                     │
└─────────────────────────┘
```

```python
class ActivationRecord:
    def __init__(self, proc_name, local_size):
        self.proc_name    = proc_name
        self.locals       = [None] * local_size
        self.control_link = None   # → AR of the procedure that CALLED us
        self.access_link  = None   # → AR of the procedure that LEXICALLY ENCLOSES us
        self.return_addr  = None
        self.return_value = None
        self.params       = []

def lookup_nonlocal(name: str, current_ar: ActivationRecord,
                    lexical_depth: int):
    """
    Follow access links to find a non-local variable 'name'
    that is 'lexical_depth' levels up in the static scope chain.
    """
    ar = current_ar
    for _ in range(lexical_depth):
        ar = ar.access_link   # walk up the static scope chain
    return ar.locals[name]    # found the variable in the enclosing scope
```

### Access Link vs. Control Link Summary

| | Control Link (Dynamic Link) | Access Link (Static Link) |
|---|---|---|
| Points to | AR of the **caller** | AR of the **lexically enclosing** procedure |
| Used for | Restoring SP on return | Finding non-local variables |
| Follows | Dynamic call chain | Static scope chain |
| Changes each call? | Yes — depends on who called | Depends on static structure only |

---

## 🧠 Part 8 — Three Allocation Strategies Compared

### Strategy 1: Fully Static Allocation

```
Pros:  O(1) variable access (direct address), no runtime overhead
Cons:  No recursion, no dynamic sizing
Used:  Early FORTRAN, embedded systems with strict constraints
```

```asm
; Fully static: 'x' always at address 0x2000
LOAD R0, 0x2000      ; fast, known at compile time
ADD  R0, R0, #1
STORE R0, 0x2000
```

### Strategy 2: Stack Allocation

```
Pros:  Supports recursion, efficient (push/pop), automatic lifetime management
Cons:  Can't outlive the call (no returning local pointers!), stack overflow possible
Used:  C/C++ local variables, most compiled languages' default locals
```

```c
void foo(int n) {
    int local_arr[n];          // VLA: variable-length array on stack
    local_arr[0] = 42;
    // local_arr ceases to exist when foo returns
    // NEVER do: return local_arr; ← dangling pointer!
}
```

### Strategy 3: Heap Allocation

```
Pros:  Lifetime independent of call stack — data can outlive the function that created it
Cons:  Slower (malloc overhead), memory leaks if not freed, fragmentation
Used:  malloc/new, closures, objects, data structures that outlive their creator
```

```c
int* create_array(int n) {
    int *arr = malloc(n * sizeof(int));  // heap allocation
    arr[0] = 42;
    return arr;  // safe! heap data outlives this call
    // CALLER must free(arr) eventually
}
```

### 💡 The Golden Rule for Choosing

| Data property | Use |
|---|---|
| Fixed size, fixed lifetime (local var) | **Stack** |
| Fixed size, program lifetime (global, `static`) | **Static** |
| Variable size OR outlives its creator | **Heap** |

---

## 🧠 Part 9 — Complete Calling Convention: Detailed Code View

### Standard Calling Convention (x86-style conceptual model)

Real-world calling conventions (System V ABI for Linux x86-64, Windows x64) follow similar principles to what the slides describe, with registers used for the first few arguments:

```
x86-64 Linux (System V AMD64 ABI):
  - First 6 integer args: RDI, RSI, RDX, RCX, R8, R9
  - Remaining args:  pushed onto stack (right to left)
  - Return value:    RAX (or RAX:RDX for 128-bit)
  - Callee-saved:    RBX, RBP, R12-R15
  - Caller-saved:    RAX, RCX, RDX, RSI, RDI, R8-R11
  - Frame pointer:   RBP (optional but useful for debugging)
  - Stack pointer:   RSP (always 16-byte aligned before a call)
```

### Caller-Save vs. Callee-Save Registers

This is a critical concept for register allocation:

| Type | Convention | Who saves/restores? |
|---|---|---|
| **Caller-saved** | If caller needs the value after the call, it must save it before calling | Caller pushes to stack before call; restores after |
| **Callee-saved** | Callee promises to preserve these registers across the call | Callee pushes at entry; restores before return |

```python
# Conceptual model of caller-save vs callee-save behavior
def caller_function():
    # Step 1: Save caller-saved registers we need after the call
    save_register('RAX')   # caller-saved — we want RAX after the call
    save_register('RCX')   # caller-saved — we want RCX after the call

    # Step 2: Set up arguments and call
    set_arg(1, value_a)
    set_arg(2, value_b)
    call(callee_function)

    # Step 3: Restore caller-saved registers
    restore_register('RCX')
    restore_register('RAX')
    # return value is in RAX

def callee_function():
    # Step 1: Save callee-saved registers we'll use in this function
    save_register('RBX')   # callee-saved — must restore before return
    save_register('RBP')   # callee-saved — frame pointer

    # Body of function using RBX freely
    RBX = compute_something()

    # Step 2: Restore callee-saved registers before returning
    restore_register('RBP')
    restore_register('RBX')
    # return value goes into RAX
    return
```

---

## 🧠 Part 10 — Full Simulation: P calls Q calls R

Below is a Python simulation of the textbook's three-level call scenario, tracking SP and TOP precisely:

```python
class RuntimeStack:
    """
    Simulate the chapter's runtime stack model.
    Stack grows toward LOWER addresses (TOP decreases on push).
    SP = pointer to old SP field of current AR.
    """
    MEMORY_SIZE = 1000

    def __init__(self):
        self.mem   = [None] * self.MEMORY_SIZE
        self.TOP   = self.MEMORY_SIZE - 1  # start at high end
        self.SP    = None                   # undefined until first call

    def push(self, value) -> int:
        """Push value; return its address."""
        addr = self.TOP
        self.mem[self.TOP] = value
        self.TOP -= 1
        return addr

    def peek(self, offset_from_SP: int):
        """Read mem[SP + offset_from_SP]."""
        return self.mem[self.SP + offset_from_SP]

    def poke(self, offset_from_SP: int, value):
        """Write mem[SP + offset_from_SP] = value."""
        self.mem[self.SP + offset_from_SP] = value

    # ─── Calling sequence ─────────────────────────────────────
    def call(self, params: list, return_addr: str, local_size: int):
        """Execute calling sequence + procbegin for the callee."""
        # Caller: push params (right to left so first param at lowest offset)
        for p in reversed(params):
            self.push(p)
        n = len(params)

        # Caller: push arg_count, return_address, empty return value, old SP
        self.push(n)                        # arg_count
        self.push(return_addr)              # return address l1
        return_val_addr = self.TOP          # remember where return value is
        self.push(None)                     # reserve return value slot
        old_SP = self.SP                    # save current SP
        self.push(old_SP)                   # old SP (control link)

        # procbegin: set SP = TOP (SP now points at old SP we just pushed)
        self.SP  = self.TOP + 1             # SP = address of old_SP field
        self.TOP = self.SP - local_size     # reserve space for local data

        return return_val_addr

    # ─── Return sequence ─────────────────────────────────────
    def return_from(self, return_value):
        """Execute return sequence."""
        # Store return value
        self.poke(1, return_value)           # 1[SP] = return_value

        # Navigate back
        TOP_temp   = self.SP + 2            # point at return address
        return_addr = self.mem[TOP_temp]    # L = *TOP

        old_SP     = self.mem[self.SP]      # *SP = saved old SP
        arg_count  = self.mem[self.SP + 3]  # 3[SP] = arg_count

        # Restore
        self.SP    = old_SP
        self.TOP   = TOP_temp + 1 + arg_count  # skip return_addr + arg_count + params

        return return_addr


# ─── Example: P calls Q calls R ───────────────────────────────
stack = RuntimeStack()

# Initially: executing P (P's AR already on stack, SP set)
# P calls Q with params [10, 20]:
ret_addr_Q = stack.call(params=[10, 20], return_addr="P_after_Q", local_size=3)

# Q calls R with params [5]:
ret_addr_R = stack.call(params=[5], return_addr="Q_after_R", local_size=2)

# R executes and returns 42:
back_to = stack.return_from(return_value=42)
print(f"R returned to: {back_to}")   # → "Q_after_R"

# Q resumes and returns 100:
back_to = stack.return_from(return_value=100)
print(f"Q returned to: {back_to}")   # → "P_after_Q"
```

---

## 🧠 Part 11 — Variable-Length Data and Extra Storage

The "extra storage" area above each AR (between ARs) is used for:

1. **Display pointers** — an array of pointers to the ARs of all lexically enclosing scopes (a display is an alternative to access links for faster non-local access)
2. **Variable-length arrays (VLAs)** — when `int arr[n]` where `n` is a runtime value, the array itself can't go in the fixed-size AR; its descriptor goes in the AR but the data goes in extra storage
3. **Spilled registers** — when the register allocator can't fit all live values

```python
def allocate_vla(n: int, elem_size: int, stack: RuntimeStack):
    """
    Allocate a variable-length array at runtime.
    AR stores a descriptor (pointer + size); data goes in extra storage.
    """
    # Data lives in extra storage area (above this AR)
    data_addr = stack.TOP          # current TOP is in extra storage
    stack.TOP -= n * elem_size     # expand extra storage for the array

    # AR stores the descriptor (pointer to data, size)
    descriptor = {'base_addr': data_addr, 'size': n, 'elem_size': elem_size}
    return descriptor  # stored in AR's local data field
```

---

## ⚠️ Edge Cases & Constraints

- **Dangling pointers from stack**: Never return the address of a local variable. Once a function returns, its AR is gone — the memory will be overwritten by the next call. This is one of C's most common bugs.
- **Stack overflow**: Uncontrolled recursion (e.g., infinite recursion, very deep call trees) exhausts the stack region. Most OSes set a stack limit (e.g., 8MB on Linux by default).
- **The `procbegin`/`return` sequence is machine-state dependent**: The exact sequence varies by architecture and calling convention. The x86-64 System V ABI differs from the ARM AAPCS differs from RISC-V calling conventions.
- **Access links only needed for lexically nested procedures**: C doesn't have nested functions (GCC extension aside), so C compilers don't generate access links. Pascal and Algol require them.
- **Control link is optional if AR size is known at compile time**: Some compilers compute the caller's SP from `SP + fixed_offset` instead of storing it — saves one word per frame.
- **VLAs (variable-length arrays)**: C99 allows `int arr[n]` where n is a runtime value. This means the AR is not fully fixed-size — the compiler must use dynamic adjustment of TOP/SP and store descriptors in the AR itself.
- **Tail call optimization**: If the last action of a procedure is calling another procedure, the callee can reuse the caller's AR — avoiding a new push. This converts tail recursion into a loop, eliminating stack growth.

---

## 💻 Complete Worked Example: `factorial` via the Chapter's Model

```python
"""
Simulate factorial(3) using this chapter's exact AR layout.

Stack model: grows toward lower addresses.
TOP: next available slot (always < SP of current frame).
SP: points to old_SP field of current frame.

AR layout from SP (upward = positive offsets = toward lower addresses):
  SP+0 : OLD_SP (control link)
  SP+1 : return_value
  SP+2 : return_address
  SP+3 : arg_count (= 1 for factorial)
  SP+4 : parameter n
  SP-1 : local_temp (for intermediate computation)
"""

def factorial_call_trace():
    trace = []

    def call_factorial(n, caller_SP, return_label):
        """Returns the activation record pushed for factorial(n)."""
        # Caller pushes: param n, arg_count=1, return_addr, ret_val_slot, old_SP
        ar = {
            'param_n':       n,
            'arg_count':     1,
            'return_addr':   return_label,
            'return_value':  None,           # to be filled by callee
            'old_SP':        caller_SP,       # control link
            'local_temp':    None,           # local T for n * factorial(n-1)
        }
        trace.append(f"CALL factorial({n}): push AR, SP → AR['old_SP']")
        return ar

    def return_factorial(ar, result):
        ar['return_value'] = result
        trace.append(f"RETURN factorial({ar['param_n']}) = {result}: "
                      f"restore SP = {ar['old_SP']}")
        return result

    # ─── Execution ───────────────────────────────────────────
    # factorial(3):
    ar3 = call_factorial(3, caller_SP='main_SP', return_label='main_L1')
    # 3 != 0, so factorial needs factorial(2):
    ar2 = call_factorial(2, caller_SP='ar3_SP',  return_label='fact_L1')
    # 2 != 0, so factorial needs factorial(1):
    ar1 = call_factorial(1, caller_SP='ar2_SP',  return_label='fact_L1')
    # 1 != 0, so factorial needs factorial(0):
    ar0 = call_factorial(0, caller_SP='ar1_SP',  return_label='fact_L1')

    # factorial(0) = 1 (base case)
    r0 = return_factorial(ar0, 1)
    # factorial(1) = 1 * 1 = 1
    r1 = return_factorial(ar1, ar1['param_n'] * r0)
    # factorial(2) = 2 * 1 = 2
    r2 = return_factorial(ar2, ar2['param_n'] * r1)
    # factorial(3) = 3 * 2 = 6
    r3 = return_factorial(ar3, ar3['param_n'] * r2)

    for line in trace:
        print(line)
    print(f"\nfactorial(3) = {r3}")

factorial_call_trace()
# Output:
# CALL factorial(3): push AR, SP → AR['old_SP']
# CALL factorial(2): push AR, SP → AR['old_SP']
# CALL factorial(1): push AR, SP → AR['old_SP']
# CALL factorial(0): push AR, SP → AR['old_SP']
# RETURN factorial(0) = 1: restore SP = ar1_SP
# RETURN factorial(1) = 1: restore SP = ar2_SP
# RETURN factorial(2) = 2: restore SP = ar3_SP
# RETURN factorial(3) = 6: restore SP = main_SP
#
# factorial(3) = 6
```

---

## 🔗 Concept Map

```
Chapter 12: Runtime Environments
│
├── Memory Layout
│     ├── Code area       — compiled instructions, fixed entry points
│     ├── Static/global   — globals, static locals, compile-time fixed
│     ├── Stack           — activation records, grows downward
│     └── Heap            — dynamic malloc/new, grows upward
│
├── Allocation Strategies
│     ├── Fully static    — all addresses fixed, no recursion (FORTRAN)
│     ├── Stack-based     — AR pushed/popped per call, supports recursion
│     └── Heap-based      — data outlives creator, explicit dealloc needed
│
├── Activation Record (Stack Frame)
│     ├── Fields: temporaries, locals, machine status, access link,
│     │           control link, return value, actual parameters
│     ├── SP (stack pointer) → OLD_SP field of current AR
│     ├── TOP → lowest used address (top of stack)
│     ├── Local vars: negative offsets from SP
│     └── Parameters: positive offsets from SP
│
├── Activation Tree
│     ├── Nodes = activations; children = callees
│     ├── Stack = path from root to current node
│     └── DFS traversal = execution order
│
├── Calling Sequence
│     ├── Caller: push params, arg_count, return_addr, ret_val_slot, old_SP
│     ├── jump to callee l2
│     ├── procbegin: SP = TOP; TOP = SP + local_size
│     └── Shared responsibility: values close to boundary for efficiency
│
├── Return Sequence
│     ├── 1[SP] = return_value
│     ├── TOP = SP + 2; SP = *SP (restore caller SP)
│     ├── L = *TOP; TOP = TOP + 1 + arg_count
│     └── goto *L (jump back to caller)
│
└── Links
      ├── Control link (dynamic link) → AR of CALLER
      └── Access link (static link)  → AR of LEXICALLY ENCLOSING scope
```

---

## 📚 References

1. **Aho, Lam, Sethi, Ullman** — *Compilers: Principles, Techniques, and Tools* (Dragon Book), 2nd ed., Chapter 7 (Run-Time Environments). Primary source for all calling sequences, AR layout, activation trees, and static/stack/heap allocation discussed in this chapter.
2. **Cooper & Torczon** — *Engineering a Compiler*, 2nd ed., Chapter 6 (The Procedure Abstraction). Especially clear treatment of calling conventions, caller-save vs. callee-save, and the division of responsibilities between caller and callee.
3. **Appel, A.** — *Modern Compiler Implementation in Java/C/ML*, Chapter 6 (Activation Records). Provides a clean, typed abstraction of frames; shows how the Frame module is designed in a real compiler project (the Tiger compiler).
4. **Intel/AMD x86-64 ABI** — *System V Application Binary Interface: AMD64 Architecture Processor Supplement* — https://refspecs.linuxbase.org/elf/x86-64-abi.pdf. The actual calling convention used on Linux x86-64; every concept from Chapter 12 appears here in production form.
5. **GeeksforGeeks — Runtime Environments in Compiler Design** — https://www.geeksforgeeks.org/compiler-design/runtime-environments-in-compiler-design/ — Concise modern summary of activation records, control stack, and allocation strategies, verified April 2026.
6. **Princeton COS 320 — Activation Records** (Prof. David Walker) — https://www.cs.princeton.edu/courses/archive/spr04/cos320/notes/7-1.pdf — Lecture notes showing step-by-step AR push/pop for recursive calls, including factorial trace.
7. **MIT 6.004 Computation Structures — Procedures and Stacks** — https://computationstructures.org/lectures/stacks/stacks.html — Interactive step-by-step walkthrough of the stack frame protocol; especially clear on why arguments are pushed in reverse order for varargs support.
8. **Grune, van Reeuwijk, Bal, Jacobs, Langendoen** — *Modern Compiler Design*, 2nd ed., Chapter 7 (Runtime Systems). Covers heap management and garbage collection in addition to stack allocation.

---

## ❓ Active Recall

### Level 1 — Definitions
- [ ] Name the four regions of runtime memory. What lives in each? Which ones grow at runtime?
- [ ] What is an activation? What is an activation record? How are they related?
- [ ] List all seven fields of an activation record. Which fields are the caller's responsibility? Which are the callee's?
- [ ] What does `SP` point to in the chapter's model? What does `TOP` point to?
- [ ] Define: control link. Define: access link. What is the difference?

### Level 2 — Mechanics
- [ ] Trace the calling sequence for `p(A+B, C)`. List every operation in order (including TAC instructions and their runtime translations).
- [ ] What does `procbegin` do? Write out its two assignments to SP and TOP.
- [ ] Trace the return sequence step by step. How is SP restored? How is the return address recovered?
- [ ] For a procedure with 3 parameters and local variable `x`, what is the offset of `x` from SP? What is the offset of the 2nd parameter?
- [ ] Draw the activation tree for: `main calls foo(3); foo(n) calls bar(n-1) if n > 0`.

### Level 3 — Analysis
- [ ] Why does fully static allocation prevent recursion? Give a concrete example of what goes wrong.
- [ ] Why does the stack grow downward (toward lower addresses) in most architectures? Why does the heap grow upward?
- [ ] If a C function returns a pointer to a local variable, what happens? Why is this always a bug?
- [ ] When is an access link needed but a control link is not sufficient? Give a language example.
- [ ] What is tail call optimization? Why does it not need to push a new activation record?

### Level 4 — Synthesis
- [ ] Simulate the call stack for `factorial(4)` using this chapter's exact AR layout. Draw each AR showing all fields and their values when factorial(0) is at the top.
- [ ] Design the complete calling sequence (TAC + runtime actions) for a call `result = q(x, y+z)` where `q` has 2 parameters and 3 local variables. Show the stack before and after `procbegin`.
- [ ] In languages like Python where functions are first-class objects (closures), why is stack allocation of activation records insufficient? What must the runtime do instead?
- [ ] Why are arguments pushed onto the stack in **reverse order** (rightmost first)? What feature does this enable? (Hint: think about `printf`.)
