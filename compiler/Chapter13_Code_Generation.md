# 📄 Chapter 13 — Code Generation

**Tags:** #compiler #code-generation #register-allocation #basic-blocks #instruction-selection #TAC #descriptors #NCKU
**Links:** [[Intermediate Code Generation]], [[Runtime Environments]], [[DAG]], [[Three-Address Code]], [[Optimization]]

---

## 🎯 The "Elevator Pitch"
> Code generation is the final translation step: it takes Three-Address Code (TAC) and turns it into real machine instructions. The central challenge is **register allocation** — CPUs have only a handful of registers, but programs have many variables. The code generator must decide *which values to keep in registers, which to spill to memory, and when*. Getting this wrong wastes cycles on unnecessary loads and stores; getting it right is what separates a fast compiler from a slow one.

---

## 🧠 Part 1 — The Code Generator's Place in the Pipeline

```
Source Code
    │
    ▼ Front End (Lexer → Parser → Semantic Analyzer)
Intermediate Code (TAC / Quadruples)
    │
    ▼ Optimizer (optional)
Optimized IR
    │
    ▼ Code Generator  ← Chapter 13
Target Machine Code (assembly / object code)
```

### Core Tasks of the Code Generator

| Task | Description |
|---|---|
| **Input handling** | Consume IR (quadruples, TAC) one basic block at a time |
| **Instruction selection** | Choose the right target-language instruction for each TAC operation |
| **Register allocation** | Decide *which* variables go in *which* registers |
| **Register assignment** | Pick the *specific register* for each variable |
| **Instruction ordering** | Schedule instructions to minimize stalls and maximize throughput |

### Requirements of Generated Code

1. **Correctness** — semantics of the source program must be preserved
2. **Efficiency** — minimize instruction count, minimize memory accesses (loads/stores)
3. **Code quality** — use registers as much as possible; only spill to memory when necessary

---

## 🧠 Part 2 — The Naive Approach: Instruction Skeletons

### What is a Skeleton?

The simplest code generation strategy maps each TAC instruction to a fixed **skeleton** — a template of machine instructions. No intelligence about register reuse.

### Skeletons for Basic Operations

```
TAC instruction      →    Machine instruction skeleton
─────────────────────────────────────────────────────────
x = y op z (binary)  →    LOAD  R1, y          (load y into R1)
                           OP    R1, z          (R1 = R1 op z)
                           STOR  R1, x          (store result to x)

x = op y (unary)     →    LOAD  R1, y
                           OP    R1
                           STOR  R1, x

x = y (copy)         →    LOAD  R1, y
                           STOR  R1, x
```

### Worked Example: `D = (A * B) + C`

TAC (quadruples):
```
(0)  =*   A   B   T1       ← T1 = A * B
(1)  =+   T1  C   T2       ← T2 = T1 + C
(2)  =    T2      D        ← D  = T2
```

Naively applying the skeleton to each quadruple:

```asm
; From quad (0): T1 = A * B
LOAD  R1, A
MUL   R1, B
STOR  R1, T1

; From quad (1): T2 = T1 + C
LOAD  R1, T1       ← redundant! T1 is still in R1
ADD   R1, C
STOR  R1, T2

; From quad (2): D = T2
LOAD  R1, T2       ← redundant! T2 is still in R1
STOR  R1, D
```

**Result: 8 instructions** — but many are unnecessary loads and stores.

### 💡 Why Is the Naive Approach Bad?

The problem is the skeleton forgets that R1 *already holds* T1 after the multiply. It blindly reloads T1 from memory for the next instruction. The code generator needs **memory** — it must track what is currently in each register.

---

## 🧠 Part 3 — Pseudo-Operators: FX and SX

To improve on naive skeletons, the textbook introduces two pseudo-operators that encode conditional load/store behavior:

### FX — "Fetch if not already available"

> **FX R1, x** = load `x` into `R1` **only if** `x` is not already in `R1`.

The code generator must track register contents to implement FX correctly.

```
Updated skeletons with FX:

=*  =>   FX    R1, arg0      (load arg0 into R1 only if needed)
          MUL   R1, arg1
          STOR  R1, result

=+  =>   FX    R1, arg0
          ADD   R1, arg1
          STOR  R1, result

=   =>   FX    R1, arg0
          STOR  R1, result
```

Applying FX to our example: after `MUL`, R1 holds T1. The next quad's `FX R1, T1` sees T1 is already in R1 → **no load emitted**.

```asm
LOAD  R1, A        ; FX: A not in R1, so load
MUL   R1, B        ; T1 = A * B; R1 now holds T1
STOR  R1, T1       ; store T1 to memory
; FX for T1 in next quad → SKIPPED (T1 already in R1)
ADD   R1, C        ; T2 = T1 + C; R1 now holds T2
STOR  R1, T2       ; store T2
; FX for T2 in next quad → SKIPPED
STOR  R1, D        ; D = T2
```

**Result: 6 instructions** (down from 8). Saved 2 redundant loads.

### SX — "Store only if not used right away"

> **SX R1, x** = store `x` from `R1` to memory **only if**:
> - `x` is NOT a temporary variable (T1, T2, etc.), OR
> - the value in R1 won't be consumed by the very next instruction

The idea: if T1 is computed and then immediately consumed by the next TAC instruction (e.g., `T2 = T1 + C`), there's no need to store T1 to memory at all — just keep it in the register.

```
Updated skeletons with both FX and SX:

=*  =>   FX    R1, arg0
          MUL   R1, arg1
          SX    R1, result        ← conditional store

=+  =>   FX    R1, arg0
          ADD   R1, arg1
          SX    R1, result

=   =>   FX    R1, arg0
          SX    R1, result
```

Applying FX + SX to our example:

```asm
LOAD  R1, A        ; FX: load A (not in register)
MUL   R1, B        ; R1 = A * B (= T1)
; SX for T1: T1 is a temp, AND it's used immediately next → SKIP store
ADD   R1, C        ; R1 = T1 + C (= T2)
; SX for T2: T2 is a temp, AND it's used immediately next → SKIP store
STOR  R1, D        ; SX for D: D is NOT a temp → MUST store
```

**Result: 4 instructions** (down from 8). Eliminated all intermediate stores for temporaries!

### Instruction Count Comparison

| Strategy | Instructions for `D = (A*B)+C` |
|---|---|
| Naive skeletons | 8 |
| FX (avoid redundant loads) | 6 |
| FX + SX (avoid redundant stores) | **4** |

---

## 🧠 Part 4 — Register and Address Descriptors: The Data Structures

The improvements above require the code generator to *remember* what is in each register and where each variable's current value lives. This is tracked with two data structures:

### Register Descriptor (RD)

> A **register descriptor** maps each register to the set of variable names whose current value is stored in that register.

```python
# Initially, all registers are empty
register_descriptor = {
    'R0': set(),   # R0 holds: nothing
    'R1': set(),   # R1 holds: nothing
    'R2': set(),   # R2 holds: nothing
    # ...
}

# After LOAD R1, A:
register_descriptor['R1'] = {'A'}

# After MUL R1, B  (R1 now holds T1 = A*B):
register_descriptor['R1'] = {'T1'}    # R1 no longer holds A or B

# After ADD R1, C  (R1 now holds T2 = T1+C):
register_descriptor['R1'] = {'T2'}
```

### Address Descriptor (AD)

> An **address descriptor** maps each variable name to the set of locations where its current value can be found (could be a register, a memory location, or both).

```python
# Initially, all variables are in memory
address_descriptor = {
    'A':  {'mem_A'},    # A is only in memory
    'B':  {'mem_B'},
    'C':  {'mem_C'},
    'T1': {'mem_T1'},
    'T2': {'mem_T2'},
    'D':  {'mem_D'},
}

# After LOAD R1, A:
address_descriptor['A'] = {'mem_A', 'R1'}   # A is in BOTH memory AND R1

# After MUL R1, B  (R1 now holds T1, A's value in R1 is gone):
address_descriptor['T1'] = {'R1'}            # T1 is ONLY in R1 (not stored yet)
address_descriptor['A']  = {'mem_A'}         # A is back to memory only

# After STOR R1, T1:
address_descriptor['T1'] = {'mem_T1', 'R1'}  # T1 is in BOTH memory and R1
```

### How These Two Descriptors Work Together

```python
class RegisterDescriptor:
    def __init__(self, registers: list[str]):
        self._rd: dict[str, set[str]] = {r: set() for r in registers}

    def get_contents(self, reg: str) -> set[str]:
        return self._rd[reg]

    def set_contents(self, reg: str, var: str):
        """After LOAD: R holds var (and only var)."""
        self._rd[reg] = {var}

    def add_var(self, reg: str, var: str):
        """Add var to reg's contents (multiple vars can share a reg value)."""
        self._rd[reg].add(var)

    def remove_var(self, reg: str, var: str):
        self._rd[reg].discard(var)

    def find_register(self, var: str) -> str | None:
        """Is var currently in any register?"""
        for reg, contents in self._rd.items():
            if var in contents:
                return reg
        return None

    def free_registers(self) -> list[str]:
        """List of registers holding nothing."""
        return [r for r, c in self._rd.items() if not c]


class AddressDescriptor:
    def __init__(self):
        self._ad: dict[str, set[str]] = {}

    def locations(self, var: str) -> set[str]:
        return self._ad.get(var, {f'mem_{var}'})  # default: in memory

    def add_location(self, var: str, loc: str):
        self._ad.setdefault(var, set()).add(loc)

    def set_location(self, var: str, loc: str):
        self._ad[var] = {loc}

    def remove_location(self, var: str, loc: str):
        if var in self._ad:
            self._ad[var].discard(loc)

    def in_register(self, var: str) -> str | None:
        """Return which register holds var, or None."""
        for loc in self.locations(var):
            if loc.startswith('R'):
                return loc
        return None

    def best_location(self, var: str) -> str:
        """Prefer register over memory."""
        locs = self.locations(var)
        for loc in locs:
            if loc.startswith('R'):
                return loc   # register preferred
        return next(iter(locs))   # fallback to memory
```

---

## 🧠 Part 5 — Basic Blocks and Flow Graphs

Before generating code, the TAC is partitioned into **basic blocks**, which are the unit of local code generation.

### What is a Basic Block?

> A **basic block** is a maximal sequence of consecutive TAC instructions with the following properties:
> 1. Control enters only at the **first** instruction (the **leader**)
> 2. Control leaves only at the **last** instruction (no jumps in the middle)
> 3. Every instruction in the block executes together as a unit

### Finding Leaders (Starting Instructions of Blocks)

```python
def find_leaders(tac: list) -> set[int]:
    """
    A leader is:
    1. The first instruction of the program
    2. Any instruction that is a target of a conditional or unconditional jump
    3. Any instruction immediately following a jump
    """
    leaders = {0}   # instruction 0 is always a leader

    for i, instr in enumerate(tac):
        if is_jump(instr):           # goto L, if A relop B goto L
            target = get_jump_target(instr)
            leaders.add(target)      # jump target is a leader
            if i + 1 < len(tac):
                leaders.add(i + 1)   # instruction after jump is a leader

    return leaders


def partition_into_blocks(tac: list) -> list[list]:
    """
    Group TAC instructions into basic blocks.
    Each block runs from one leader up to (but not including) the next.
    """
    leaders = sorted(find_leaders(tac))
    blocks = []

    for i, start in enumerate(leaders):
        end = leaders[i + 1] if i + 1 < len(leaders) else len(tac)
        blocks.append(tac[start:end])

    return blocks
```

### Example: Partitioning TAC into Basic Blocks

```
TAC:
  (0)  i = 1                ← LEADER (first instruction)
  (1)  j = 1
  (2)  t1 = 10 * i
  (3)  t2 = t1 + j
  (4)  t3 = 8 * t2
  (5)  t4 = t3 - 88
  (6)  a[t4] = 0
  (7)  j = j + 1
  (8)  if j <= 10 goto 2    ← jump → next instr (9) is leader; target (2) is leader
  (9)  i = i + 1            ← LEADER
  (10) if i <= 10 goto 1    ← jump → target (1) is leader
  (11) …                    ← LEADER

Basic Blocks:
  B1: instructions 0 (entry)
  B2: instructions 1–8   (loop body)
  B3: instruction 9–10
  B4: instruction 11+
```

### Flow Graph

A **flow graph** is a directed graph where:
- **Nodes** = basic blocks
- **Edges** = possible control flow between blocks

```python
def build_flow_graph(blocks: list[list]) -> dict[int, list[int]]:
    """
    Build control flow graph.
    Returns: dict mapping block_idx -> list of successor block_idxs
    """
    successors = {}
    for i, block in enumerate(blocks):
        last = block[-1]
        succs = []

        if is_unconditional_jump(last):
            target_block = find_block_containing(get_jump_target(last), blocks)
            succs.append(target_block)
        elif is_conditional_jump(last):
            target_block = find_block_containing(get_jump_target(last), blocks)
            succs.append(target_block)    # taken branch
            if i + 1 < len(blocks):
                succs.append(i + 1)       # fall-through branch
        else:
            if i + 1 < len(blocks):
                succs.append(i + 1)       # sequential flow

        successors[i] = succs

    return successors
```

---

## 🧠 Part 6 — Next-Use Information: The Key to Smart Code Generation

### What is "Next-Use"?

> For each variable `x` at each program point (TAC instruction), the **next use** of `x` is the next instruction (if any) that reads `x` before it is redefined.

If a variable has no next use after being computed, it is **dead** — its value will never be read again. A dead value in a register can be discarded immediately, freeing the register.

### 💡 Intuition: Next-Use as a "Freshness Date"

Think of a register as a shelf spot in a kitchen. Next-use tells you: "this ingredient (variable) will be needed again in 3 steps." If the ingredient has no next-use, it's gone stale — throw it out (free the register). If it has an imminent next-use, keep it on the shelf (keep it in the register).

### Computing Next-Use: Backward Scan

The algorithm scans the basic block **backward** (from last instruction to first):

```python
def compute_next_use(block: list[tuple]) -> list[dict]:
    """
    Compute next-use and liveness for each variable at each instruction.
    block: list of TAC tuples (op, arg1, arg2, result)
    Returns: list of dicts {var: (next_use_instr, is_live)}
    """
    n = len(block)
    # Initially: all variables are live (conservative assumption)
    # In a real compiler, global liveness analysis would narrow this down
    live     = {var: True  for var in all_vars(block)}
    next_use = {var: None  for var in all_vars(block)}

    info = [None] * n   # will store per-instruction info

    # Scan backward
    for i in range(n - 1, -1, -1):
        op, arg1, arg2, result = block[i]

        # Record current liveness/next-use for this instruction
        info[i] = {
            var: (next_use[var], live[var])
            for var in all_vars(block)
        }

        # Update: result is DEFINED here → dead before this point
        if result:
            live[result]     = False   # kill: result's old value is overwritten
            next_use[result] = None    # no next-use for old value

        # Update: args are USED here → live before this point
        for arg in (arg1, arg2):
            if arg and not is_constant(arg):
                live[arg]     = True   # gen: arg is needed
                next_use[arg] = i      # next use of arg is at instruction i

    return info
```

### Example: Next-Use for a Simple Block

```
Block (backwards scan):
  (0)  t1 = a * b
  (1)  t2 = t1 + c
  (2)  d  = t2

Backward scan:
  After (2):  d defined  → live[d]=False; next_use[d]=None
              t2 used    → live[t2]=True; next_use[t2]=2

  After (1):  t2 defined → live[t2]=False; next_use[t2]=None (old t2 dead)
              t1 used    → live[t1]=True; next_use[t1]=1
              c  used    → live[c]=True; next_use[c]=1

  After (0):  t1 defined → live[t1]=False; next_use[t1]=None
              a  used    → live[a]=True; next_use[a]=0
              b  used    → live[b]=True; next_use[b]=0

Result: t1 and t2 are dead after their single use → no need to store them!
```

---

## 🧠 Part 7 — The getReg Function: Register Selection

> **getReg(instruction)** decides which register to use for the result of an operation, and ensures operands are in registers if needed.

### getReg Priority Rules

```python
def getreg(result: str, operands: list[str],
           rd: RegisterDescriptor, ad: AddressDescriptor,
           next_use_info: dict) -> str:
    """
    Select the best register L for storing the result of an operation.
    Returns: register name
    """
    # Rule 1: If result is already in a register and it's safe to use, use it
    r = ad.in_register(result)
    if r is not None:
        return r

    # Rule 2: Use a free (empty) register
    free = rd.free_registers()
    if free:
        return free[0]

    # Rule 3: Spill — find the register whose current occupant
    # (a) has no next-use (dead), OR
    # (b) has its value safely in memory, OR
    # (c) has the furthest next-use (Belady's optimal algorithm approximation)
    best_reg = None
    best_score = -1

    for reg in rd.all_registers():
        occupants = rd.get_contents(reg)
        spill_cost = 0

        for var in occupants:
            nu = next_use_info.get(var)
            if nu is None:
                # var is dead — free spill, score = infinity
                return reg   # immediately usable, no store needed
            elif f'mem_{var}' in ad.locations(var):
                # var is already in memory — free spill
                pass
            else:
                # Must store to memory — incurs cost
                spill_cost += 1

        # Choose register with least spill cost (furthest next use)
        score = (nu if nu else float('inf')) - spill_cost
        if score > best_score:
            best_score = score
            best_reg = reg

    # Spill the chosen register: store its contents to memory
    for var in rd.get_contents(best_reg):
        if f'mem_{var}' not in ad.locations(var):
            emit(f'STOR  {best_reg}, {var}')        # store to memory
            ad.add_location(var, f'mem_{var}')

    return best_reg
```

---

## 🧠 Part 8 — The Complete Code Generation Algorithm

### Algorithm for a Single Basic Block

```python
def generate_code_for_block(block: list[tuple],
                             rd: RegisterDescriptor,
                             ad: AddressDescriptor) -> list[str]:
    """
    Generate target code for one basic block.
    Each TAC instruction: x = y op z (binary) or x = y (copy) or x = op y (unary)
    """
    next_use_info = compute_next_use(block)
    output = []

    def emit(instr: str):
        output.append(instr)

    for i, (op, arg1, arg2, result) in enumerate(block):
        nu = next_use_info[i]

        if op in ('+', '-', '*', '/'):
            # Binary operation: result = arg1 op arg2
            L  = getreg(result, [arg1, arg2], rd, ad, nu)

            # Ensure arg1 is in L (load if needed)
            y_loc = ad.best_location(arg1)
            if y_loc != L:
                emit(f'LOAD  {L}, {y_loc}')
                rd.set_contents(L, arg1)
                ad.add_location(arg1, L)

            # Emit the operation: L = L op arg2
            z_loc = ad.best_location(arg2)
            machine_op = {'+'  : 'ADD',
                          '-'  : 'SUB',
                          '*'  : 'MUL',
                          '/'  : 'DIV'}[op]
            emit(f'{machine_op}  {L}, {z_loc}')

            # Update descriptors
            # L now holds result (and ONLY result)
            for old_var in list(rd.get_contents(L)):
                ad.remove_location(old_var, L)
            rd.set_contents(L, result)
            ad.set_location(result, L)

            # If arg1 or arg2 have no next-use and are only in L, free them
            for var in (arg1, arg2):
                if nu.get(var, (None,))[0] is None:   # no next-use → dead
                    if list(ad.locations(var)) == [L]:  # only in register
                        rd.remove_var(L, var)           # free from register

        elif op == '=':
            # Copy: result = arg1
            y_loc = ad.best_location(arg1)
            L = getreg(result, [arg1], rd, ad, nu)
            if y_loc != L:
                emit(f'LOAD  {L}, {y_loc}')
            rd.add_var(L, result)
            ad.add_location(result, L)

    # End of block: store all live variables back to memory
    for reg in rd.all_registers():
        for var in rd.get_contents(reg):
            if nu.get(var, (True,))[1]:   # var is live on exit
                if f'mem_{var}' not in ad.locations(var):
                    emit(f'STOR  {reg}, {var}')
                    ad.add_location(var, f'mem_{var}')

    return output
```

### Descriptor Update Rules (Summary)

After each instruction, update descriptors according to these rules:

| Instruction | Register Descriptor Update | Address Descriptor Update |
|---|---|---|
| `LOAD R, x` | `RD[R] = {x}` | `AD[x] += {R}` |
| `STOR R, x` | (unchanged) | `AD[x] += {mem_x}` |
| `OP Ry, Rz → Rx` (result `x=y op z`) | `RD[Rx] = {x}` (only x) | `AD[x] = {Rx}` (only in reg) |
| Free dead var from reg R | `RD[R].remove(var)` | `AD[var].remove(R)` |

---

## 🧠 Part 9 — Full Worked Example: `D = (A*B) + C`

Let's trace the full algorithm with **2 registers** (R1, R2) for:

```
TAC:
  (0)  T1 = A * B
  (1)  T2 = T1 + C
  (2)  D  = T2
```

**Next-use analysis** (backward scan):
```
Instruction (2): D = T2 → next_use[T2]=2, T2 is live; D: no next-use in block
Instruction (1): T2 = T1+C → next_use[T1]=1, next_use[C]=1; T2's old value dead
Instruction (0): T1 = A*B → next_use[A]=0, next_use[B]=0; T1's old value dead
```

**Initial state:**
```
RD: R1={}, R2={}
AD: A={mem_A}, B={mem_B}, C={mem_C}, T1={mem_T1}, T2={mem_T2}, D={mem_D}
```

**Processing instruction (0): T1 = A * B**
```
1. getReg(T1, [A,B]) → R1 is free → pick R1
2. A not in R1 → emit: LOAD  R1, mem_A
   RD: R1={A}; AD: A={mem_A, R1}
3. Emit: MUL  R1, mem_B
   T1 = A * B; R1 now holds T1
   Update: RD: R1={T1}; AD: T1={R1}
4. A has next_use=None after (0)? No, A has no more uses. A only in mem_A → ok.
   B has no next-use in block → nothing to free (B only in memory).

Generated: LOAD R1, A
           MUL  R1, B
```

**Processing instruction (1): T2 = T1 + C**
```
1. getReg(T2, [T1,C]) → T1 is in R1; R2 is free → pick R2? 
   Better: T1 is already in R1, use R1 for T2 (no need for a new reg)
   Actually: pick R1 (it has T1 which we need; result replaces T1)
   L = R1
2. T1 in R1 already → no LOAD needed (FX works!)
3. Emit: ADD  R1, mem_C
   T2 = T1 + C; R1 now holds T2
   Update: RD: R1={T2}; AD: T2={R1}; T1 is dead (no next-use) → AD: T1={mem_T1}
4. T1 has no next-use → remove from R1 (already done by update)
   C has no next-use after this → C only in mem_C, nothing to remove

Generated: ADD  R1, C
```

**Processing instruction (2): D = T2**
```
1. getReg(D) → R1 holds T2; result D goes into R1
2. T2 in R1 already → no LOAD needed
3. emit nothing for the operation (copy: D = T2)
   Update: RD: R1={D, T2}; AD: D={R1}; T2={R1}

End of block: D is live (it's a named variable) → must store
   if mem_D not in AD[D] → emit: STOR R1, D
```

**Final generated code:**
```asm
LOAD  R1, A      ; fetch A (not in register)
MUL   R1, B      ; R1 = A * B (= T1)
ADD   R1, C      ; R1 = T1 + C (= T2); no reload of T1 needed!
STOR  R1, D      ; D = T2; store only the final named result
```

**4 instructions** — matching the FX+SX optimal result from Part 3.

---

## 🧠 Part 10 — Register Spilling: When Registers Run Out

### The Spilling Problem

When all registers are occupied and a new value needs a register, we must **spill** — evict one variable from a register and save it to memory to free the register.

### Spill Selection Strategies

| Strategy | Rule | Quality |
|---|---|---|
| **Dead variable** | Evict a variable with no next-use (dead) — no store needed! | Best (free) |
| **Already in memory** | Evict a variable whose value is already in memory — no store needed | Free |
| **Furthest next-use** (Belady) | Evict the variable whose next use is farthest in the future | Near-optimal |
| **Least recently used (LRU)** | Evict the variable used least recently | Good heuristic |
| **FIFO** | Evict the oldest variable in the register | Simple, suboptimal |

```python
def spill_register(rd: RegisterDescriptor, ad: AddressDescriptor,
                   next_use: dict, code: list) -> str:
    """
    Choose a register to spill. Returns the freed register name.
    Priority: dead var > already in memory > furthest next-use.
    """
    # Pass 1: look for a dead variable
    for reg, vars_in_reg in rd.items():
        for var in vars_in_reg:
            if next_use.get(var) is None:
                rd.remove_var(reg, var)   # evict dead var (no store needed)
                return reg

    # Pass 2: look for a var already in memory
    for reg, vars_in_reg in rd.items():
        for var in vars_in_reg:
            if f'mem_{var}' in ad.locations(var):
                rd.remove_var(reg, var)   # evict (value safe in memory)
                return reg

    # Pass 3: Belady's approximation — evict variable with furthest next-use
    best_reg = None
    best_distance = -1
    for reg, vars_in_reg in rd.items():
        distance = min(
            (next_use.get(var, float('inf')) for var in vars_in_reg),
            default=float('inf')
        )
        if distance > best_distance:
            best_distance = distance
            best_reg = reg

    # Must store the evicted variable's value to memory
    for var in rd.get_contents(best_reg):
        if f'mem_{var}' not in ad.locations(var):
            code.append(f'STOR  {best_reg}, {var}')
            ad.add_location(var, f'mem_{var}')
    rd.set_contents_empty(best_reg)

    return best_reg
```

---

## 🧠 Part 11 — Peephole Optimization: Cleaning Up Generated Code

Even with a good code generator, some inefficiencies remain. **Peephole optimization** looks at a small sliding window of generated instructions and applies local improvements.

### Common Peephole Patterns

```python
def peephole_optimize(instructions: list[str]) -> list[str]:
    """
    Apply peephole optimizations on generated assembly.
    Looks at small windows of 1-3 instructions.
    """
    i = 0
    result = []

    while i < len(instructions):
        # Pattern 1: Redundant load after store
        #   STOR R1, x
        #   LOAD R1, x    ← redundant if nothing modifies x in between
        if (i + 1 < len(instructions) and
            is_store(instructions[i], reg='R1', var='x') and
            is_load(instructions[i+1],  reg='R1', var='x')):
            result.append(instructions[i])
            i += 2   # skip the redundant load
            continue

        # Pattern 2: Jump to immediate next instruction
        #   GOTO L
        #   L: ...        ← if L is the very next instruction
        if (is_goto(instructions[i]) and
            instructions[i+1].startswith(get_goto_target(instructions[i]) + ':')):
            i += 1   # skip the useless goto
            continue

        # Pattern 3: Algebraic identity
        #   ADD R1, 0     ← adding zero does nothing
        #   MUL R1, 1     ← multiplying by one does nothing
        if is_add_zero(instructions[i]) or is_mul_one(instructions[i]):
            i += 1   # skip the no-op instruction
            continue

        # Pattern 4: Strength reduction
        #   MUL R1, 2  →  ADD R1, R1   (multiply by 2 = add to itself)
        if is_mul_power_of_two(instructions[i]):
            n = get_multiplier(instructions[i])
            reg = get_dest_register(instructions[i])
            shift_amount = int(n).bit_length() - 1
            result.append(f'SHL  {reg}, #{shift_amount}')   # left shift
            i += 1
            continue

        result.append(instructions[i])
        i += 1

    return result
```

---

## 🧠 Part 12 — Global Register Allocation

Local (per-block) allocation is limited — it can't keep a variable in a register *across* basic blocks. **Global register allocation** addresses this.

### Cost-Benefit Analysis for Loop Variables

For a variable `x` inside a loop `L`, allocating a register for `x` has:

```
Benefit = (use(x, B) + 2 × live(x, B)) × loop_depth_factor

where:
  use(x, B) = number of times x is USED in blocks B within loop L
  live(x, B) = 1 if x is live on exit from block B (needs store at end), else 0
  loop_depth_factor = typically 10 per nesting level (inner loops more valuable)
```

Load cost = 2 (if x not in register at loop entry, must load once)
Store cost = 2 (if x live on exit, must store once)

**Net benefit** = `use(x,B) * 2 - 2 * (load_cost) - 2 * (store_cost × live(x,B))`

Allocate a dedicated register to variables with the highest net benefit.

### Graph Coloring: The Modern Approach

> **Register allocation by graph coloring**: model the problem as graph coloring where each variable is a node, an edge connects variables that are simultaneously live (interfere), and colors represent registers. If the graph can be colored with `k` colors, then `k` registers suffice — no spilling needed.

```python
class InterferenceGraph:
    """
    Build interference graph for register allocation.
    Two variables interfere if they are simultaneously live at any point.
    """
    def __init__(self):
        self.nodes = set()       # one node per variable
        self.edges = set()       # (u, v): u and v interfere

    def add_variable(self, var: str):
        self.nodes.add(var)

    def add_interference(self, u: str, v: str):
        if u != v:
            self.edges.add((min(u,v), max(u,v)))

    def build_from_liveness(self, liveness: dict[int, set[str]]):
        """
        Two variables interfere if they are both live at the same point.
        For each instruction, add edges between all simultaneously live variables.
        """
        for point, live_set in liveness.items():
            live_list = list(live_set)
            for i in range(len(live_list)):
                for j in range(i+1, len(live_list)):
                    self.add_interference(live_list[i], live_list[j])

    def color(self, k: int) -> dict[str, int] | None:
        """
        Try to color graph with k colors (= k registers).
        Uses greedy coloring with Kempe-chain spilling.
        Returns: {variable: color/register_number} or None if k colors insufficient.
        """
        # Simplification: try coloring; if a node has degree >= k, spill it
        coloring = {}
        stack = []
        graph = {v: set() for v in self.nodes}
        for u, v in self.edges:
            graph[u].add(v)
            graph[v].add(u)

        remaining = set(self.nodes)
        spills = set()

        while remaining:
            # Find a node with degree < k (can always be colored)
            simplifiable = [n for n in remaining if len(graph[n] & remaining) < k]
            if simplifiable:
                node = simplifiable[0]
                stack.append(node)
                remaining.remove(node)
            else:
                # Spill: pick a variable (e.g., least used)
                spill = min(remaining, key=lambda n: len(graph[n] & remaining))
                spills.add(spill)
                remaining.remove(spill)

        # Color the stack
        while stack:
            node = stack.pop()
            neighbor_colors = {coloring[n] for n in graph[node] if n in coloring}
            for color in range(k):
                if color not in neighbor_colors:
                    coloring[node] = color
                    break

        return coloring, spills
```

---

## ⚠️ Edge Cases & Constraints

- **Temporaries vs. named variables**: Temporaries (T1, T2, …) created by the compiler generally don't need to be stored to memory if they're consumed in the same basic block — this is exactly what SX exploits. Named variables often *do* need to be stored because they may be live across blocks.
- **Variables live across blocks**: Local (per-block) code generation must store all live-on-exit variables to memory at the end of each block. Without global liveness information, the code generator must conservatively assume all named variables are live on exit.
- **Register aliasing**: On some architectures, writing to `R1` (32-bit) also modifies `R0` (the low 16 bits). The code generator must be aware of register aliases to avoid descriptor inconsistencies.
- **Calling conventions**: When a procedure call is encountered in a basic block, all caller-saved registers must be spilled before the call. The code generator must insert stores for all caller-saved registers holding live variables.
- **NP-completeness of register allocation**: The general register allocation problem (for an arbitrary number of registers) is NP-complete. Practical compilers use heuristics (graph coloring with Chaitin's algorithm, or LLVM's greedy allocator).
- **FX and SX are compiler-internal abstractions**: They don't correspond to real machine instructions — they are resolved during code generation into actual LOAD/STOR instructions (or omitted entirely when the condition is met).
- **Address descriptors can have multiple locations**: A variable can be simultaneously in a register AND in memory (happens after a STOR). The code generator must track this carefully to avoid spurious stores.

---

## 🔗 Complete Concept Map

```
Chapter 13: Code Generation
│
├── Naive Code Generation
│     ├── Skeleton per TAC operator (LOAD, OP, STOR)
│     ├── 8 instructions for D = (A*B)+C
│     └── Problem: redundant loads and stores
│
├── Pseudo-operators
│     ├── FX: "load only if not in register already" → 6 instructions
│     └── SX: "store only if not immediately consumed or not a temp" → 4 instructions
│
├── Core Data Structures
│     ├── Register Descriptor (RD): register → set of variable names currently there
│     └── Address Descriptor (AD): variable → set of locations (register or memory)
│
├── Basic Blocks and Flow Graphs
│     ├── Basic block: maximal no-branch sequence
│     ├── Leaders: first instr, jump targets, instrs after jumps
│     └── Flow graph: directed graph of blocks (control flow)
│
├── Next-Use Information
│     ├── Computed by backward scan of basic block
│     ├── next_use[x] at instr i = next instr that reads x before redefining it
│     └── Used by getReg to choose which register to free (spill dead variables first)
│
├── Code Generation Algorithm
│     ├── For each TAC: x = y op z
│     │     ├── getReg: pick register L for result
│     │     ├── Load y if not in L (emit LOAD if needed)
│     │     ├── Emit OP L, z_location
│     │     └── Update RD and AD
│     └── End of block: store all live-on-exit variables
│
├── Register Spilling
│     ├── Priority: dead var > var in memory > furthest next-use (Belady)
│     └── Must emit STOR before evicting if not already in memory
│
├── Peephole Optimization
│     ├── Redundant load/store elimination
│     ├── Jump-to-next elimination
│     └── Strength reduction (MUL 2 → SHL 1)
│
└── Global Register Allocation
      ├── Loop variable cost-benefit: use(x,B) + live(x,B) weighting
      └── Graph coloring: interference graph with k-coloring = k registers
```

---

## 📚 References

1. **Aho, Lam, Sethi, Ullman** — *Compilers: Principles, Techniques, and Tools* (Dragon Book), 2nd ed., Chapter 8 (Code Generation). Sections 8.4 (Basic Blocks and Flow Graphs), 8.5 (Optimization of Basic Blocks), 8.6 (A Simple Code Generator), 8.7 (Peephole Optimization), 8.8 (Register Allocation and Assignment). The primary textbook for all concepts in this chapter.
2. **Cooper & Torczon** — *Engineering a Compiler*, 2nd ed., Chapter 13 (Register Allocation). Covers graph-coloring register allocation (Chaitin's algorithm) with excellent diagrams of interference graphs.
3. **Appel, A.** — *Modern Compiler Implementation in Java/C/ML*, Chapters 7–8. Provides a working Tiger compiler that implements basic block construction, next-use analysis, and register allocation — excellent for seeing real implementation.
4. **Chaitin, G.J. et al. (1981)** — *"Register Allocation via Coloring"*, Computer Languages 6(1). The landmark paper introducing graph-coloring register allocation, now used in GCC and most industrial compilers.
5. **Belady, L.A. (1966)** — *"A Study of Replacement Algorithms for a Virtual Storage Computer"*, IBM Systems Journal. Introduced the optimal page replacement algorithm (furthest future use) — the theoretical basis for optimal register spill selection.
6. **GeeksforGeeks — Compiler Design: Code Generation** — https://www.geeksforgeeks.org/compiler-design/code-generation/ — Concise verified summary of register/address descriptors and the code generation algorithm.
7. **NYU Compilers Lecture 14** — https://cs.nyu.edu/courses/spring09/G22.2130-001/lectures/lecture-14.html — Detailed treatment of next-use computation and getReg with worked examples, including important corrections to the Dragon Book's algorithm.
8. **CMU 15-411: Compiler Design Recitation 8** — https://www.cs.cmu.edu/~411/rec/08-solutions.pdf — Modern treatment of liveness analysis and its connection to code generation, including fixed-point iteration for global liveness.
9. **LLVM Code Generator Documentation** — https://llvm.org/docs/CodeGenerator.html — How a production code generator (used in Clang, Rust, Swift) handles register allocation, instruction selection, and scheduling in practice.

---

## ❓ Active Recall

### Level 1 — Definitions
- [ ] What is a basic block? What makes the first instruction of a basic block a "leader"?
- [ ] What is a register descriptor (RD)? What does it store?
- [ ] What is an address descriptor (AD)? What does it store?
- [ ] What is "next-use" information? How is it computed (what direction is the scan)?
- [ ] What does FX do? What does SX do? How do they differ?

### Level 2 — Mechanics
- [ ] Apply the naive skeleton approach to `E = (A + B) * (C - D)`. How many instructions are generated?
- [ ] Apply FX and SX optimizations to the same expression. How many instructions remain?
- [ ] Trace the code generation algorithm for `x = a + b; y = x * c` with 2 registers. Show the RD and AD after each instruction.
- [ ] Compute next-use information (backward scan) for the block: `t1=a*b; t2=t1+c; t3=t2-d; e=t3`. Which temporaries are dead after being used?
- [ ] When does SX emit a store? When does it suppress one?

### Level 3 — Analysis
- [ ] Why must all live variables be stored to memory at the end of a basic block? What assumption makes this necessary?
- [ ] A register holds variable `x` and a new value needs that register. Under what three conditions can we spill without emitting a STOR instruction?
- [ ] Why is register allocation NP-complete in general? What does graph coloring have to do with it?
- [ ] Explain Belady's optimal replacement strategy in the context of register spilling. Why can't compilers always use it?
- [ ] What is a peephole optimizer? Give three concrete patterns it can eliminate.

### Level 4 — Synthesis
- [ ] Design the complete RD and AD state evolution for `D = (A*B)+C` with only 1 register available. What happens? How many spill instructions are needed?
- [ ] Given a loop body that uses variables `a`, `b`, `c`, `d` with use counts 8, 5, 3, 1 respectively and all are live on exit, and only 2 registers are available: which variables should get dedicated registers? Show the cost-benefit calculation.
- [ ] Build the interference graph for the block: `t1=a+b; t2=a+c; d=t1+t2`. Which variables interfere? How many registers are needed to color this graph?
- [ ] Explain why the FX optimization requires the code generator to track register contents. What would happen if the code generator forgot what was in a register between two consecutive TAC instructions?
