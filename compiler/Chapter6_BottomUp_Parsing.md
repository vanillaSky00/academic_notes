# 📄 Bottom-Up Parsing (Shift-Reduce & LR Parsing)

**Tags:** #compiler #parsing #LR #shift-reduce #automata #NCKU
**Links:** [[Top-Down Parsing]], [[CFG]], [[First-Follow Sets]], [[DFA]], [[Rightmost Derivation]]

---

## 🎯 The "Elevator Pitch"
> Bottom-up parsing reads input left-to-right and reconstructs the parse tree by working **from leaves up to the root** — essentially running a derivation in reverse. Think of it like solving a jigsaw puzzle by finding and snapping together pieces (handles) until you have the complete picture (the start symbol).

---

## 🧠 Part 1 — Bottom-Up Parsing vs. Top-Down Parsing

| Property | Top-Down (LL) | Bottom-Up (LR) |
|---|---|---|
| Direction of tree build | Root → Leaves | Leaves → Root |
| Derivation traced | **Leftmost** derivation | **Rightmost** derivation (in reverse) |
| Rule application | Replace LHS with RHS | Replace RHS with LHS (reduce) |
| Grammar power | LL(k) grammars | LR(k) grammars — strictly more powerful |
| Conflict-free? | Needs no left recursion | Handles left recursion naturally |

### 💡 Intuition
Imagine parsing `id + id * id`:
- **Top-down**: Start with `E`, predict what to expand.
- **Bottom-up**: Start with `id`, recognize it's a valid `F`, build up to `T`, then to `E`.

---

## 🧠 Part 2 — Handle Pruning

### What is a Handle?

> A **handle** is a substring on the stack that matches the **right-hand side (RHS)** of some production, and reducing it represents **one valid step** in reversing a rightmost derivation.

**Formally:** If `S →*_rm αAw →_rm αβw`, then `β` at position after `α` is a handle of `αβw`.

### 🔑 Key Insight: The Handle Property
- The handle always appears at the **top of the stack** in shift-reduce parsing.
- There is **never a need to look deeper** into the stack to find the handle (this is why a stack is the right data structure!).
- Handle pruning = repeatedly finding and reducing handles until only the start symbol remains.

### 🧩 Analogy
Think of handles like "ripe fruit" on a tree. A bottom-up parser scans the input and "plucks" (reduces) ready groups from the top of the stack, working upward until the whole tree is built.

```python
def handle_pruning_concept(input_tokens, grammar):
    """
    Conceptual handle pruning: repeatedly find and reduce handles.
    A handle is a RHS substring at top of stack that can be reduced.
    """
    stack = []
    index = 0

    while True:
        # Try to find a handle (RHS of some rule) at top of stack
        handle = find_handle(stack, grammar)

        if handle:
            # REDUCE: pop RHS, push LHS
            for _ in range(len(handle.rhs)):
                stack.pop()
            stack.append(handle.lhs)

        elif index < len(input_tokens):
            # SHIFT: push next input token
            stack.append(input_tokens[index])
            index += 1

        elif stack == [grammar.start_symbol] and index == len(input_tokens):
            return "ACCEPT"
        else:
            return "ERROR"
```

---

## 🧠 Part 3 — Shift-Reduce Parsing

### The Four Actions

| Action | What it Does |
|---|---|
| **Shift** | Push the next input token onto the stack |
| **Reduce** | Pop the handle (RHS) off the stack; push the corresponding LHS nonterminal |
| **Accept** | Input fully consumed and stack holds only start symbol → success! |
| **Error** | No valid action possible → call error recovery |

### 📦 Stack + Input Buffer Model

```
STACK           INPUT           ACTION
$               id+id$          shift
$id             +id$            reduce by F → id
$F              +id$            reduce by T → F
$T              +id$            reduce by E → T
$E              +id$            shift
$E+             id$             shift
$E+id           $               reduce by F → id
$E+F            $               reduce by T → F
$E+T            $               reduce by E → E+T
$E              $               ACCEPT
```

> **`$`** is the **bottom-of-stack marker** / end-of-input marker.

```python
def shift_reduce_parser(tokens, action_table, goto_table):
    """
    Shift-Reduce parser using a state stack (LR engine skeleton).
    action_table[(state, token)] -> ('shift', s) | ('reduce', rule) | 'accept' | 'error'
    goto_table[(state, nonterminal)] -> next_state
    """
    stack = [0]             # state stack (starts with state 0)
    sym_stack = ['$']       # symbol stack for readability
    tokens.append('$')      # end-of-input marker
    i = 0

    while True:
        state = stack[-1]
        token = tokens[i]
        action = action_table.get((state, token), 'error')

        if action == 'error':
            raise SyntaxError(f"Unexpected token '{token}' in state {state}")

        elif action == 'accept':
            print("ACCEPT")
            return True

        elif action[0] == 'shift':
            next_state = action[1]
            stack.append(next_state)
            sym_stack.append(token)
            i += 1  # consume input token

        elif action[0] == 'reduce':
            lhs, rhs = action[1]           # e.g., ('E', ['E', '+', 'T'])
            for _ in range(len(rhs)):
                stack.pop()                # pop |rhs| states
                sym_stack.pop()
            top_state = stack[-1]
            goto_state = goto_table[(top_state, lhs)]
            stack.append(goto_state)
            sym_stack.append(lhs)
            # Note: i does NOT advance on reduce!
```

---

## 🧠 Part 4 — LR Parsers: The Big Picture

### Why LR is King

1. **Most powerful** deterministic shift-reduce parser — recognizes every unambiguous CFG.
2. **No backtracking** needed — decisions made in `O(n)` time.
3. **Proper superset** of LL(k): anything LL parsers can handle, LR can too — and more.
4. **Early error detection**: errors caught as soon as the erroneous token is seen.

### The LR Parsing Engine

```
┌──────────────────────────────────────────────────┐
│                  LR PARSER ENGINE                │
│                                                  │
│  INPUT:  a₁  a₂  a₃  ...  aₙ  $                │
│                   ↓                              │
│  ┌─────────────────────────┐                    │
│  │   Stack: [s₀, s₁, s₂…] │  ← State Stack     │
│  └────────────┬────────────┘                    │
│               │                                  │
│  ┌────────────▼────────────┐                    │
│  │     LR Parsing Table    │                    │
│  │  ┌──────────┬────────┐  │                    │
│  │  │  ACTION  │  GOTO  │  │                    │
│  │  └──────────┴────────┘  │                    │
│  └────────────┬────────────┘                    │
│               │                                  │
│        shift / reduce / accept / error           │
└──────────────────────────────────────────────────┘
```

### Structure of the LR Parsing Table

The table has two parts:
- **ACTION[state, terminal]** → shift sⱼ | reduce by rule | accept | error
- **GOTO[state, nonterminal]** → next state (used after a reduction)

---

## 🧠 Part 5 — LR(0) Items and Table Construction

### What is an LR(0) Item?

> An **LR(0) item** is a grammar production with a **dot (•)** indicating how far parsing has progressed through the rule's RHS.

For production `A → XYZ`, the items are:
```
A → • X Y Z    (haven't seen anything yet)
A → X • Y Z    (seen X, expecting Y then Z)
A → X Y • Z    (seen XY, expecting Z)
A → X Y Z •    (completed! — this is a "reduce item")
```

### 💡 Item Types
- **Kernel item**: dot is NOT at the leftmost position (or it's the augmented start item `S' → • S`)
- **Nonkernel item**: dot is at the leftmost position — added by closure
- **Reduce item**: dot at the rightmost position — triggers a reduce action

### Computing the Closure

> **Closure(I)**: If `A → α • B β` is in the set and `B → γ` is a production, add `B → • γ` to the set. Repeat until no new items can be added.

```python
def closure(items, grammar):
    """
    Compute the closure of a set of LR(0) items.
    items: set of (lhs, rhs_tuple, dot_position)
    """
    result = set(items)
    changed = True

    while changed:
        changed = False
        for (lhs, rhs, dot) in list(result):
            # If dot is before a nonterminal B: A → α • B β
            if dot < len(rhs) and is_nonterminal(rhs[dot]):
                B = rhs[dot]
                # Add all productions B → • γ
                for production_rhs in grammar.productions[B]:
                    new_item = (B, production_rhs, 0)  # dot at position 0
                    if new_item not in result:
                        result.add(new_item)
                        changed = True

    return result
```

### Computing GOTO

> **GOTO(I, X)**: Move the dot past symbol X for all items in I where the dot is before X, then compute closure.

```python
def goto(items, symbol, grammar):
    """
    Compute GOTO(items, symbol): advance dot past 'symbol' then take closure.
    """
    moved = set()
    for (lhs, rhs, dot) in items:
        # If dot is before 'symbol': A → α • symbol β
        if dot < len(rhs) and rhs[dot] == symbol:
            moved.add((lhs, rhs, dot + 1))   # A → α symbol • β

    return closure(moved, grammar) if moved else set()
```

### Building the Collection of LR(0) Item Sets

```python
def build_lr0_collection(grammar):
    """
    Build the canonical collection of LR(0) item sets (the DFA states).
    """
    # Augment grammar: S' → S
    start_item = ("S'", ("S",), 0)
    I0 = closure({start_item}, grammar)

    collection = [I0]
    transitions = {}   # (state_index, symbol) -> state_index

    i = 0
    while i < len(collection):
        current = collection[i]
        # Find all symbols after a dot in this state
        symbols = {rhs[dot] for (_, rhs, dot) in current if dot < len(rhs)}

        for X in symbols:
            goto_set = goto(current, X, grammar)
            if goto_set not in collection:
                collection.append(goto_set)
            j = collection.index(goto_set)
            transitions[(i, X)] = j

        i += 1

    return collection, transitions
```

---

## 🧠 Part 6 — The Characteristic Finite-State Machine (CFSM)

### What is the CFSM?

> The **CFSM** (Characteristic Finite-State Machine) is the DFA built from LR(0) item sets. It recognizes **viable prefixes** of right sentential forms.

### Key Concept: Viable Prefix

> A **viable prefix** is any prefix of a right sentential form that **does not extend past the handle**. Intuitively: the stack at any valid point in parsing always holds a viable prefix.

- CFSM states = LR(0) item sets
- CFSM transitions = GOTO function
- **Double-boxed (accepting) states** = states with a complete item (`A → α •`) — these are reduce states

### 💡 Why Viable Prefixes Matter
The parser never puts an "illegal" string on the stack — it's always a viable prefix. This is the mathematical guarantee that shift-reduce parsing is sound.

---

## 🧠 Part 7 — Completing the LR(0) Parse Table

### Algorithm

```
For each item set Iᵢ in the canonical collection:
  For each item A → α • a β in Iᵢ  (a is a terminal):
    ACTION[i, a] = shift j   where GOTO(Iᵢ, a) = Iⱼ

  For each item A → α • in Iᵢ  (dot at end, A ≠ S'):
    ACTION[i, a] = reduce A → α   for ALL terminals a (LR(0) rule)

  For item S' → S • in Iᵢ:
    ACTION[i, $] = accept

  For each nonterminal A where GOTO(Iᵢ, A) = Iⱼ:
    GOTO[i, A] = j
```

> ⚠️ **LR(0) limitation**: reduce actions apply to ALL terminals — no lookahead used. This causes conflicts for many practical grammars.

---

## 🧠 Part 8 — Conflict Diagnosis

### Two Types of Conflicts

| Conflict | Description | When it Occurs |
|---|---|---|
| **Shift/Reduce** | In state Iᵢ, given token `a`: should we shift or reduce? | `A → α •` AND `B → β • a γ` both in Iᵢ |
| **Reduce/Reduce** | In state Iᵢ: which rule to reduce by? | `A → α •` AND `B → β •` both in Iᵢ |

### Root Causes

1. **Grammar is ambiguous** → No table construction method can fix this without extra rules.
2. **Grammar is not LR(0)** but might be SLR(1) or LALR(1) → More lookahead resolves it.

### Classic Example: Dangling Else

```
S → if E then S
S → if E then S else S
S → other
```

After `if E then S`, seeing `else` → shift (attach else to nearest if) or reduce? → Shift/reduce conflict. Most languages resolve by convention: **shift wins** (innermost match).

### Ambiguous Grammar Example: Arithmetic

```
E → E + E | E * E | id
```

State where `E → E + E •` is complete but next token is `+` or `*`:
- **Reduce** → left-associativity for `+`
- **Shift** → right-associativity / higher precedence for `*`

Resolving via **operator precedence and associativity rules** is standard practice (used in yacc/bison).

---

## 🧠 Part 9 — SLR(k) Parsing

### Motivation: LR(0) is Too Weak

LR(0) says: "if there's a reduce item in this state, reduce **always**." This creates conflicts even for unambiguous grammars.

### The SLR Insight

> **SLR(k)** (Simple LR with k-token lookahead) resolves reduce actions by only reducing when the next token is in **Follow(A)** for the rule `A → α •`.

```
SLR(1) Rule:
  If A → α • is in Iᵢ:
    ACTION[i, a] = reduce A → α
    ONLY IF a ∈ Follow(A)
```

### SLR(1) Table Construction

```python
def build_slr1_table(collection, transitions, grammar, follow_sets):
    """
    Build SLR(1) parse table by using Follow sets to restrict reduce actions.
    """
    ACTION = {}
    GOTO = {}

    for i, item_set in enumerate(collection):
        for (lhs, rhs, dot) in item_set:

            if dot < len(rhs):
                X = rhs[dot]
                j = transitions.get((i, X))
                if j is not None:
                    if is_terminal(X):
                        # Shift action
                        if (i, X) in ACTION and ACTION[(i, X)] != ('shift', j):
                            raise ConflictError(f"Shift/Reduce conflict at state {i}, token {X}")
                        ACTION[(i, X)] = ('shift', j)
                    else:
                        # GOTO for nonterminals
                        GOTO[(i, X)] = j

            else:
                # Dot at end → reduce item
                if lhs == "S'":
                    ACTION[(i, '$')] = 'accept'
                else:
                    # SLR key: only reduce for tokens in Follow(lhs)
                    for a in follow_sets[lhs]:
                        if (i, a) in ACTION:
                            raise ConflictError(f"Conflict at state {i}, token {a}")
                        ACTION[(i, a)] = ('reduce', (lhs, rhs))

    return ACTION, GOTO
```

### SLR vs. LR(0) vs. LALR vs. LR(1)

| Method | Lookahead | Power | Practicality |
|---|---|---|---|
| **LR(0)** | None | Weakest | Few real grammars |
| **SLR(1)** | Follow sets | Moderate | Simple to build |
| **LALR(1)** | Computed lookahead per state | Strong | Most tools (yacc, bison) |
| **LR(1)** | Full 1-token lookahead | Strongest | Huge tables |

### 💡 Intuition for SLR Lookahead

LR(0) is like a guesser who reduces as soon as it sees any completed rule.  
SLR(1) is smarter: "I'll only reduce by `A → α` if the next token could *legally* follow `A` in this grammar." It uses **Follow(A)** as a filter.

---

## ⚠️ Edge Cases & Constraints

- **LR(0) conflicts cannot always be solved** — if the grammar is inherently ambiguous, no amount of lookahead in standard LR resolves all conflicts.
- **SLR can still fail** on grammars that need LALR or LR(1): Follow sets are computed globally and may be too coarse-grained.
- **Augmented grammar** is always required: add `S' → S` to ensure a unique accept state.
- **Reduce/reduce conflicts** are more dangerous than shift/reduce: they indicate fundamental ambiguity in the grammar's structure.
- **Grammars that are not LR(k) for any k**: these exist (e.g., requiring unbounded lookahead) and cannot be handled by table-driven LR parsers.
- `GOTO` is only defined for **nonterminals**; `ACTION` is only for **terminals** (including `$`).

---

## 💻 Comprehensive Code Sketch

```python
# ─────────────────────────────────────────────
# FULL LR(0) / SLR(1) SKELETON
# ─────────────────────────────────────────────

class Grammar:
    def __init__(self, productions, start):
        self.productions = productions  # dict: lhs -> [rhs_tuple, ...]
        self.start = start
        self.terminals = self._get_terminals()
        self.nonterminals = set(productions.keys())

    def _get_terminals(self):
        all_syms = {s for rhss in self.productions.values()
                      for rhs in rhss for s in rhs}
        return all_syms - set(self.productions.keys())


def closure(items, grammar):
    """Expand item set by adding B → •γ for each A → α•Bβ."""
    result = set(items)
    worklist = list(items)
    while worklist:
        lhs, rhs, dot = worklist.pop()
        if dot < len(rhs) and rhs[dot] in grammar.nonterminals:
            B = rhs[dot]
            for rhs_B in grammar.productions[B]:
                new_item = (B, rhs_B, 0)
                if new_item not in result:
                    result.add(new_item)
                    worklist.append(new_item)
    return frozenset(result)


def goto(item_set, symbol, grammar):
    """Move dot past symbol, then take closure."""
    moved = frozenset(
        (lhs, rhs, dot + 1)
        for lhs, rhs, dot in item_set
        if dot < len(rhs) and rhs[dot] == symbol
    )
    return closure(moved, grammar) if moved else frozenset()


def build_collection(grammar):
    """Build canonical LR(0) item sets and transitions."""
    augmented_start = ("S'", (grammar.start,), 0)
    I0 = closure({augmented_start}, grammar)

    states = [I0]
    state_index = {I0: 0}
    transitions = {}

    i = 0
    while i < len(states):
        current = states[i]
        all_symbols = {rhs[dot] for _, rhs, dot in current if dot < len(rhs)}

        for X in all_symbols:
            g = goto(current, X, grammar)
            if g not in state_index:
                state_index[g] = len(states)
                states.append(g)
            transitions[(i, X)] = state_index[g]

        i += 1

    return states, transitions


def compute_follow(grammar):
    """Compute Follow sets (needed for SLR)."""
    follow = {A: set() for A in grammar.nonterminals}
    follow[grammar.start].add('$')

    changed = True
    while changed:
        changed = False
        for lhs, rhss in grammar.productions.items():
            for rhs in rhss:
                for i, sym in enumerate(rhs):
                    if sym in grammar.nonterminals:
                        beta = rhs[i+1:]
                        # First(β) - {ε} goes into Follow(sym)
                        first_beta = compute_first_of_string(beta, grammar)
                        before = len(follow[sym])
                        follow[sym] |= (first_beta - {'ε'})
                        # If β can derive ε, Follow(lhs) → Follow(sym)
                        if 'ε' in first_beta or len(beta) == 0:
                            follow[sym] |= follow[lhs]
                        if len(follow[sym]) > before:
                            changed = True
    return follow


def build_slr1_table(states, transitions, grammar, follow):
    """Build SLR(1) ACTION and GOTO tables."""
    ACTION = {}
    GOTO_TABLE = {}

    for i, state in enumerate(states):
        for lhs, rhs, dot in state:
            if dot < len(rhs):
                X = rhs[dot]
                j = transitions.get((i, X))
                if j is None:
                    continue
                if X in grammar.terminals:
                    # Shift
                    key = (i, X)
                    if key in ACTION and ACTION[key] != ('shift', j):
                        print(f"CONFLICT at state {i} on '{X}'")
                    ACTION[key] = ('shift', j)
                else:
                    GOTO_TABLE[(i, X)] = j
            else:
                # Reduce item
                if lhs == "S'":
                    ACTION[(i, '$')] = ('accept',)
                else:
                    for a in follow[lhs]:
                        key = (i, a)
                        new_action = ('reduce', lhs, rhs)
                        if key in ACTION and ACTION[key] != new_action:
                            print(f"CONFLICT at state {i} on '{a}'")
                        ACTION[key] = new_action

    return ACTION, GOTO_TABLE


def slr1_parse(tokens, ACTION, GOTO_TABLE):
    """Run the SLR(1) parser on a token stream."""
    stack = [0]
    tokens = list(tokens) + ['$']
    i = 0

    while True:
        state = stack[-1]
        tok = tokens[i]
        action = ACTION.get((state, tok))

        if action is None:
            raise SyntaxError(f"Error: unexpected '{tok}' in state {state}")
        elif action[0] == 'accept':
            print("Input accepted ✓")
            return True
        elif action[0] == 'shift':
            stack.append(action[1])
            i += 1
        elif action[0] == 'reduce':
            _, lhs, rhs = action
            for _ in rhs:
                stack.pop()
            top = stack[-1]
            stack.append(GOTO_TABLE[(top, lhs)])
```

---

## 🔗 Concept Map

```
Bottom-Up Parsing
│
├── Key Mechanism: Handle Pruning
│     └── Handle = RHS at top of stack matching a production
│
├── Implementation: Shift-Reduce Parsing
│     ├── Actions: Shift | Reduce | Accept | Error
│     └── Data Structure: Stack + Input Buffer
│
├── LR Parser Family (power increases →)
│     ├── LR(0)    — no lookahead, weakest
│     ├── SLR(1)   — uses Follow sets as lookahead filter
│     ├── LALR(1)  — per-state computed lookahead (yacc/bison)
│     └── LR(1)    — full 1-token lookahead, most powerful
│
├── Construction Tools
│     ├── LR(0) Items  — productions with dot marker
│     ├── Closure(I)   — expand dots before nonterminals
│     ├── GOTO(I, X)   — advance dot past X + closure
│     └── CFSM         — DFA over viable prefixes
│
└── Conflicts
      ├── Shift/Reduce — grammar ambiguity or insufficient lookahead
      └── Reduce/Reduce — fundamental grammar ambiguity
```

---

## 📚 References

1. **Aho, Lam, Sethi, Ullman** — *Compilers: Principles, Techniques, and Tools* (龍書 / "Dragon Book"), 2nd ed. — Chapters 4.5–4.7. The primary textbook for LR parsing theory and SLR table construction.
2. **Appel, A.** — *Modern Compiler Implementation in Java/ML/C* — Chapter 3. Alternative treatment of LR parsing with clean pseudocode.
3. **Cooper & Torczon** — *Engineering a Compiler*, 2nd ed. — Chapter 3. Very readable treatment of viable prefixes and the CFSM.
4. **Grune & Jacobs** — *Parsing Techniques: A Practical Guide* (Free PDF) — Chapters 9–11. Encyclopedic treatment of all LR variants.
5. **GNU Bison Manual** — https://www.gnu.org/software/bison/manual/ — Real-world SLR/LALR implementation reference.
6. **3Blue1Brown "Essence of Linear Algebra"** (conceptual analogy for state spaces).

---

## ❓ Active Recall

### Level 1 — Definitions
- [ ] What is the key difference in how top-down and bottom-up parsers trace derivations?
- [ ] Define a "handle" precisely. What property does a handle always have regarding the stack?
- [ ] What are the four actions in shift-reduce parsing? Describe each.
- [ ] What is an LR(0) item? Write all items for the production `E → E + T`.

### Level 2 — Mechanics
- [ ] Explain the `closure` algorithm. Why do we need it?
- [ ] Explain the `GOTO` function. What are its inputs and output?
- [ ] How do you fill the `ACTION` table for an LR(0) parser? What rule triggers a shift vs. a reduce?
- [ ] What is a viable prefix? Why is it important that the stack always holds one?
- [ ] What is the CFSM and how does it relate to the LR(0) item sets?

### Level 3 — Analysis & Conflict Resolution
- [ ] What is a shift/reduce conflict? Give a concrete example.
- [ ] What is a reduce/reduce conflict? Which is more severe and why?
- [ ] How does SLR(1) improve upon LR(0)? What sets does it use to restrict reduce actions?
- [ ] Why can an ambiguous grammar never be parsed without conflicts by any LR method?
- [ ] If a grammar has a conflict in SLR(1), what are your options?

### Level 4 — Synthesis
- [ ] Trace the shift-reduce parse of `id * id + id` using a grammar `E → E+T | T`, `T → T*F | F`, `F → id`. Show the stack, input, and action at each step.
- [ ] Given the SLR rule "reduce only when next token ∈ Follow(A)", why is this still sometimes insufficient (i.e., why do we need LALR)?
- [ ] Design a grammar that is SLR(1) but not LR(0). Explain why it needs the Follow set check.
