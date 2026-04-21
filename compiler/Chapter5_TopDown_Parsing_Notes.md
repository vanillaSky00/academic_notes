# 📄 Chapter 5: Top-Down Parsing

**Tags:** #compiler-design #parsing #LL1 #recursive-descent #syntax-analysis  
**Links:** [[Context-Free Grammars]], [[Lexical Analysis]], [[Syntax Trees]], [[Bottom-Up Parsing]]

---

## 🎯 The "Elevator Pitch"

> Top-down parsing is like **reading a recipe from the title down to the steps**: you start with the big picture (the start symbol) and keep expanding rules until you can match each actual ingredient (terminal token). The goal is to reconstruct how a sentence was built — without guessing wrong.

---

## 🗺️ Chapter Overview — The Big Picture

```
                    Source Code String
                           │
              ┌────────────▼────────────┐
              │     Top-Down Parser      │
              │  (starts from ROOT of    │
              │    parse tree, expands   │
              │    toward the LEAVES)    │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │        Two Forms         │
              └────────────┬────────────┘
          ┌────────────────┴────────────────┐
          ▼                                 ▼
  Recursive-Descent                  Table-Driven
  (with/without backtracking)       LL(k) Parser
```

---

## 🧠 Part 1: Recursive-Descent Parsing

### Core Idea

A **recursive-descent parser** has **one procedure per nonterminal**. Each procedure tries to match its nonterminal against the current input, calling other procedures recursively for sub-nonterminals.

**Key property:** The parse tree is built in **preorder** (root → left subtree → right subtree), which corresponds to a **leftmost derivation**.

### Example Grammar & Procedure

Given the grammar:
```
exp    → exp addop term | term
addop  → + | –
term   → term mulop factor | factor
mulop  → *
factor → ( exp ) | number
```

A naive recursive procedure for `factor`:

```c
// C-pseudocode for factor()
procedure factor():
    token = next_token()
    switch token:
        case '(':
            match('(')
            exp()          // recursive call!
            match(')')
        case NUMBER:
            match(NUMBER)
        else:
            error
```

### 🔧 Using EBNF to Eliminate Left Recursion

Notice `exp → exp addop term` is **left-recursive** (exp calls itself at the start). This causes infinite recursion! We rewrite using EBNF's `{...}` (zero-or-more) notation:

```
exp → term { addop term }    ← eliminates left recursion
```

Now the procedure becomes iterative:

```python
def exp():
    term()                     # match the first term
    while token in ('+', '-'): # zero or more addop-term pairs
        addop()
        term()

def term():
    factor()
    while token == '*':
        mulop()
        factor()

def factor():
    if token == '(':
        match('(')
        exp()
        match(')')
    elif token == NUMBER:
        match(NUMBER)
    else:
        error()
```

### 🌳 Syntax Tree Construction

For `3 + 4 + 5`, the parse tree looks like:

```
        +
       / \
      +   5
     / \
    3   4
```

This is a **left-associative** tree — the standard interpretation of repeated addition.

**Building a tree in code:**

```python
def exp() -> Node:
    newtemp = term()                    # left child
    while token in ('+', '-'):
        tok = token
        match(tok)
        newtemp2 = term()               # right child
        newtemp = makeOpNode(tok, newtemp, newtemp2)  # create internal node
    return newtemp
```

---

## 🧠 Part 2: Top-Down Parsing WITH Backtracking

### How it Works

Create **one procedure per nonterminal**. When a choice must be made (e.g., `A → ab | a`), try alternatives **left-to-right**, saving the input pointer so you can backtrack on failure.

**Example:** Grammar `S → cAd`, `A → ab | a`

```python
def S():
    if input_symbol == 'c':
        advance()
        if A():
            if input_symbol == 'd':
                advance()
                return True
    return False

def A():
    isave = input_pointer          # save position for backtracking
    if input_symbol == 'a':
        advance()
        if input_symbol == 'b':
            advance()
            return True            # matched 'ab'
    input_pointer = isave          # backtrack!
    if input_symbol == 'a':
        advance()
        return True                # matched 'a'
    return False
```

Language recognized: `L = {cabd, cad}`

### ⚠️ Problems with Backtracking

1. **Left recursion causes infinite loops.** A grammar is **left-recursive** if `A ⟹* Aα` for some nonterminal A. Example: `A → Aβ | δ` will loop forever.

2. **Backtracking also undoes semantic actions.** If your parser builds a symbol table while parsing, backtracking corrupts it.

3. **Order of alternatives matters.** For `A → w | ...`, if `w = ε` is applied first, *cod* would always be tried before anything useful.

**The fix: Use the lookahead symbol to pick the right alternative — no backtracking needed.**


We can Elimination of Left Recursion – Solved Problems
 https://www.youtube.com/watch?v=PFey5FpKlFM&t=404s

---

## 🧠 Part 3: The LL(1) Predict Function

### Motivation

Instead of trying alternatives blindly, we use a **single lookahead token** to *predict* which production to use.

**PREDICT(A → X₁X₂...Xₘ) =**
- `FIRST(X₁X₂...Xₘ)` — if it doesn't contain ε
- `FIRST(X₁X₂...Xₘ) \ {ε} ∪ FOLLOW(A)` — if it contains ε

### 📌 FIRST Sets

`FIRST(X)` = the set of terminals that can begin any string derived from X.

**Algorithm to compute FIRST:**
1. If X is a terminal: `FIRST(X) = {X}`
2. If X is a nonterminal and X → ε exists: add ε to FIRST(X)
3. If X → Y₁Y₂...Yₖ, add FIRST(Y₁) to FIRST(X). If ε ∈ FIRST(Y₁), also add FIRST(Y₂), etc.
4. Repeat until no changes.

```python
def compute_first(grammar):
    first = {sym: set() for sym in grammar.all_symbols}

    # Terminals: FIRST(t) = {t}
    for t in grammar.terminals:
        first[t].add(t)

    changed = True
    while changed:
        changed = False
        for (A, production) in grammar.productions:
            # production = [Y1, Y2, ..., Yk]
            i = 0
            while i < len(production):
                Yi = production[i]
                before = len(first[A])
                first[A] |= (first[Yi] - {'ε'})  # add FIRST(Yi) \ {ε}
                if 'ε' not in first[Yi]:
                    break    # Yi cannot vanish, stop here
                i += 1
            else:
                first[A].add('ε')  # all Yi can derive ε
            if len(first[A]) > before:
                changed = True

    return first
```

**Worked Example** (expression grammar):
```
E  → TE'           First(E)  = { (, id }
E' → +TE' | λ      First(E') = { +, λ }
T  → FT'           First(T)  = { (, id }
T' → *FT' | λ      First(T') = { *, λ }
F  → (E) | id      First(F)  = { (, id }
```

### 📌 FOLLOW Sets

`FOLLOW(A)` = the set of terminals that can appear **immediately to the right** of A in any sentential form.

**Algorithm to compute FOLLOW:**
1. Place `$` in FOLLOW(S) where S is the start symbol.
2. If there is a production `A → αBβ`:
   - Add `FIRST(β) \ {ε}` to FOLLOW(B)
   - If `ε ∈ FIRST(β)`, add FOLLOW(A) to FOLLOW(B)
3. If there is a production `A → αB`:
   - Add FOLLOW(A) to FOLLOW(B)
4. Repeat until no changes.

```python
def compute_follow(grammar, first):
    follow = {A: set() for A in grammar.nonterminals}
    follow[grammar.start_symbol].add('$')

    changed = True
    while changed:
        changed = False
        for (A, production) in grammar.productions:
            trailer = set(follow[A])   # start with FOLLOW(A)
            for B in reversed(production):
                if B in grammar.nonterminals:
                    before = len(follow[B])
                    follow[B] |= trailer
                    if len(follow[B]) > before:
                        changed = True
                    if 'ε' in first[B]:
                        trailer = trailer | (first[B] - {'ε'})
                    else:
                        trailer = first[B].copy()
                else:
                    trailer = {B}  # B is a terminal
    return follow
```

**Worked Example:**
```
Follow(E)  = { $, ) }
Follow(E') = { $, ) }
Follow(T)  = { +, $, ) }
Follow(T') = { +, $, ) }
Follow(F)  = { *, +, $, ) }
```

### 📌 LL(1) Predict Function — Full Definition

A grammar G is **LL(1)** if and only if for any two distinct productions `A → α` and `A → β`:
1. `FIRST(α) ∩ FIRST(β) = ∅` (no shared first symbols)
2. At most one of α, β can derive ε
3. If `β ⟹* ε`, then `FIRST(α) ∩ FOLLOW(A) = ∅`

*Intuition: Condition 1 and 3 together ensure you can always pick the **right** production using just **one lookahead token**.*

> ⚠️ **No ambiguous grammar and no left-recursive grammar can be LL(1).**

---

## 🧠 Part 4: The LL(1) Parse Table

### Construction Algorithm

Given a grammar G, build table M[A, a] (rows = nonterminals, columns = terminals + $):

```
For each production A → α:
  1. For each terminal a ∈ FIRST(α):
       M[A, a] = A → α
  2. If ε ∈ FIRST(α):
       For each terminal b ∈ FOLLOW(A):
         M[A, b] = A → α
       If $ ∈ FOLLOW(A):
         M[A, $] = A → α
  3. All remaining entries = ERROR
```

**Parse table for expression grammar:**

| | `id` | `+` | `*` | `(` | `)` | `$` |
|---|---|---|---|---|---|---|
| E | E→TE' | | | E→TE' | | |
| E' | | E'→+TE' | | | E'→λ | E'→λ |
| T | T→FT' | | | T→FT' | | |
| T' | | T'→λ | T'→\*FT' | | T'→λ | T'→λ |
| F | F→id | | | F→(E) | | |

```python
def build_parse_table(grammar, first, follow):
    table = {}  # table[(A, a)] = production

    for (A, production) in grammar.productions:
        first_of_prod = compute_first_of_string(production, first)

        for a in first_of_prod - {'ε'}:
            if (A, a) in table:
                raise Exception(f"Grammar is NOT LL(1)! Conflict at M[{A},{a}]")
            table[(A, a)] = production

        if 'ε' in first_of_prod:
            for b in follow[A]:
                if (A, b) in table:
                    raise Exception(f"Grammar is NOT LL(1)! Conflict at M[{A},{b}]")
                table[(A, b)] = production  # use ε-production when lookahead ∈ FOLLOW

    return table
```

---

## 🧠 Part 5: LL(1) Predictive (Table-Driven) Parsing

### Architecture

```
Input:  [ a  +  b  $  ]
          ↑
          |
     ┌────┴────────────────┐
     │  Predictive Parser  │──── Parse Table M[A,a]
     └────┬────────────────┘
          |
        Stack: [ S $ ]  (initially)
          |
        Output (leftmost derivation)
```

**Components:**
- **Input buffer** with end marker `$`
- **Stack** with end marker `$` on the bottom, start symbol S on top
- **Parse table** M[A, a]

### Algorithm

```python
def ll1_parse(input_string, table, grammar):
    stack = ['$', grammar.start_symbol]  # $ at bottom, S on top
    input_string += '$'
    ip = 0  # input pointer

    while stack[-1] != '$':
        X = stack[-1]           # top of stack
        a = input_string[ip]    # current input symbol

        if X == a:              # X is a matching terminal
            stack.pop()
            ip += 1             # advance input
        elif X in grammar.terminals:
            error()             # terminal mismatch
        elif (X, a) not in table:
            error()             # no rule for this combination
        else:
            production = table[(X, a)]   # look up M[X, a]
            stack.pop()                  # pop X
            if production != ['ε']:
                for sym in reversed(production):
                    stack.append(sym)    # push production in reverse
            print(f"Output: {X} → {' '.join(production)}")

    if input_string[ip] == '$':
        print("ACCEPT")
    else:
        error()
```

**Trace for `()$`, grammar `S → (S)S | ε`:**

| Step | Stack | Input | Action |
|------|-------|-------|--------|
| 1 | `$S` | `()$` | S→(S)S |
| 2 | `$S)S(` | `()$` | match `(` |
| 3 | `$S)S` | `)$` | S→ε |
| 4 | `$S)` | `)$` | match `)` |
| 5 | `$S` | `$` | S→ε |
| 6 | `$` | `$` | **ACCEPT** |

---

## 🧠 Part 6: Elimination of Left Recursion

### Why It's Needed

LL(1) parsers **cannot handle left recursion**. `A → Aα | β` causes infinite loops because the parser would push A forever without consuming input.

### Immediate Left Recursion

Transform: `A → Aα₁ | Aα₂ | ... | β₁ | β₂ | ...`

Into:
```
A  → β₁A' | β₂A' | ...
A' → α₁A' | α₂A' | ... | ε
```

**Example:**
```
Before:  E → E + T | T
After:   E  → T E'
         E' → + T E' | ε
```

```python
def eliminate_immediate_left_recursion(A, productions):
    """
    A → Aα₁|Aα₂|...|β₁|β₂|...
    →  A → β₁A'|β₂A'|...   and   A' → α₁A'|α₂A'|...|ε
    """
    alphas = []   # right sides of left-recursive rules (what follows A)
    betas  = []   # right sides of non-left-recursive rules

    for prod in productions:
        if prod[0] == A:            # starts with A: left-recursive
            alphas.append(prod[1:]) # drop the leading A
        else:
            betas.append(prod)

    if not alphas:
        return {A: productions}    # no left recursion, nothing to do

    A_prime = A + "'"
    new_A_prods  = [beta + [A_prime] for beta in betas]
    new_A_prime_prods = [alpha + [A_prime] for alpha in alphas] + [['ε']]

    return {A: new_A_prods, A_prime: new_A_prime_prods}
```

### General Left Recursion (More than 2 Steps)

Indirect left recursion: `A ⟹⁺ Aα` via multiple steps.
Example: `S → Aa | b`, `A → Ac | Sd | ε`
Here S can derive itself indirectly.

**General Algorithm (arrange nonterminals A₁...Aₙ in some order):**

```python
def eliminate_all_left_recursion(grammar):
    nonterminals = list(grammar.nonterminals)  # A1, A2, ..., An

    for i, Ai in enumerate(nonterminals):
        # Step 1: Substitute earlier Aj's into Ai's productions
        for j in range(i):
            Aj = nonterminals[j]
            new_prods = []
            for prod in grammar.productions[Ai]:
                if prod[0] == Aj:
                    # Replace Aj with each of Aj's productions
                    for aj_prod in grammar.productions[Aj]:
                        new_prods.append(aj_prod + prod[1:])
                else:
                    new_prods.append(prod)
            grammar.productions[Ai] = new_prods

        # Step 2: Eliminate immediate left recursion in Ai
        grammar.productions.update(
            eliminate_immediate_left_recursion(Ai, grammar.productions[Ai])
        )

    # Note: The grammar must have no cycles (A ⟹+ A) and no ε-productions
    # for this algorithm to work correctly.
    return grammar
```

**Example walkthrough:**
```
Original:  S → Aa | b      A → Ac | Sd | ε

Step 1 (i=1, j=0): Substitute S into A's productions
  A → Ac | Sd | ε
  → A → Ac | Aad | bd | ε   (replaced S with Aa|b)

Step 2: Eliminate immediate left recursion in A
  A  → bdA' | A'            (from β = bd, ε)
  A' → cA' | adA' | ε       (from α = c, ad)
```

---

## 🧠 Part 7: Left Factoring

When two productions for A share a **common prefix** α:
```
A → αβ₁ | αβ₂ | ...
```
An LL(1) parser can't decide which to use until *after* consuming α.

**Fix:** Factor out the common prefix:
```
A  → αA'
A' → β₁ | β₂ | ...
```

**Example:**
```
Before:  S → iCtS | iCtSeS | a     (if-then and if-then-else)
After:   S  → iCtSS' | a
         S' → eS | ε
```

```python
def left_factor(A, productions):
    """Find the longest common prefix of all productions for A."""
    from itertools import groupby

    # Group productions by their first symbol
    groups = {}
    for prod in productions:
        first = prod[0]
        groups.setdefault(first, []).append(prod)

    new_prods = []
    new_nonterminals = {}

    for first_sym, group in groups.items():
        if len(group) == 1:
            new_prods.append(group[0])  # no conflict, keep as is
        else:
            # Find longest common prefix
            prefix = group[0]
            for prod in group[1:]:
                i = 0
                while i < len(prefix) and i < len(prod) and prefix[i] == prod[i]:
                    i += 1
                prefix = prefix[:i]

            A_prime = A + "'"
            # A → prefix A'
            new_prods.append(prefix + [A_prime])
            # A' → remainder₁ | remainder₂ | ...
            remainders = [p[len(prefix):] or ['ε'] for p in group]
            new_nonterminals[A_prime] = remainders

    return new_prods, new_nonterminals
```

---

## 🧠 Part 8: Building Recursive-Descent Parsers from LL(1) Tables

Instead of hand-writing parsing procedures, we can **automatically generate** them from an LL(1) table.

### General Parsing Procedure Template

```c
void non_term(void) {
    token tok = next_token();
    switch (tok) {
        case TERMINAL_LIST:
            parsing_actions();  // actions for this production
            break;
        default:
            syntax_error(tok);
            break;
    }
}
```

### gen_actions() — Automatic Code Generation

```python
def gen_actions(production, nt_name):
    """Generate parsing actions for one production."""
    actions = []
    for symbol in production:
        if is_terminal(symbol):
            actions.append(f'match("{symbol}");')  # consume terminal
        else:
            actions.append(f'{symbol}();')          # recurse into nonterminal
    return actions

def generate_parser_procedure(A, table, grammar):
    """Generate the full switch-case procedure for nonterminal A."""
    print(f"void {A}(void) {{")
    print(f"    token tok = next_token();")
    print(f"    switch (tok) {{")
    for (NT, terminal), production in table.items():
        if NT == A:
            print(f"        case {terminal}:")
            for action in gen_actions(production, A):
                print(f"            {action}")
            print(f"            break;")
    print(f"        default: syntax_error(tok); break;")
    print(f"    }}")
    print(f"}}")
```

---

## ⚠️ Edge Cases & Constraints

| Problem | Description | Fix |
|---|---|---|
| **Left recursion** | `A → Aα` causes infinite loop | Eliminate via transformation |
| **Common prefixes** | Parser can't choose `A→αβ₁\|αβ₂` | Left factoring |
| **Multiply-defined entries** | Grammar is NOT LL(1) | Refactor grammar |
| **Ambiguity** | Same string has 2 parse trees | Ambiguous grammars ≠ LL(1) |
| **ε-productions** | Require FOLLOW sets for correct prediction | Careful FOLLOW computation |
| **Indirect left recursion** | `A→Bα, B→Aβ` | General left-recursion elimination |

### Multiply-Defined Entry — Definition

A parse table entry M[A, a] is **multiply-defined** if more than one production fills it. This means:
```
∃ productions A→α, A→β where a ∈ PREDICT(A→α) ∩ PREDICT(A→β)
```

This immediately means the grammar is **not LL(1)**.

---

## 🔗 Concept Connections

```
Grammar (CFG)
    │
    ├─── Compute FIRST sets ───────────┐
    │                                  │
    ├─── Compute FOLLOW sets ──────────┤
    │                                  │
    │                                  ▼
    │                         Build PREDICT sets
    │                                  │
    ├─── Left Recursion? ──── Yes ──► Eliminate it
    │                                  │
    ├─── Common Prefixes? ─── Yes ──► Left Factor
    │                                  │
    └─── Build LL(1) Table ◄───────────┘
              │
    ┌─────────┴────────────┐
    │   Multiply-defined?  │
    │  Yes → NOT LL(1)     │
    │  No  → LL(1) ✓       │
    └─────────┬────────────┘
              │
    ┌─────────▼────────────┐
    │  Generate Parser     │
    │  (recursive-descent  │
    │   OR table-driven)   │
    └──────────────────────┘
```

---

## ❓ Active Recall

- [ ] What does **LL(1)** stand for? What does each letter/number mean?
- [ ] What is the difference between **FIRST(α)** and **FOLLOW(A)**? When do you need FOLLOW?
- [ ] Why can't an LL(1) parser handle **left recursion**? What happens mechanically?
- [ ] Write the transformation that eliminates **immediate left recursion** from `A → Aα | β`.
- [ ] Given `A → αβ₁ | αβ₂`, how do you **left factor** this grammar?
- [ ] How do you construct the **LL(1) parse table**? Step by step.
- [ ] In the table-driven parsing algorithm, what happens when:  
  (a) top of stack is a terminal matching input?  
  (b) top of stack is a nonterminal?  
  (c) the table entry is ERROR?
- [ ] What makes a grammar **NOT LL(1)**? Give two conditions.
- [ ] Given the expression grammar `E → E+T | T`, `T → T*F | F`, `F → (E) | id`:
  - Eliminate left recursion
  - Compute FIRST and FOLLOW
  - Build the LL(1) table
- [ ] Can an **ambiguous grammar** ever be LL(1)? Why or why not?
- [ ] What is **left factoring** and when is it needed?
- [ ] In the predictive parsing algorithm, why is the stack initialized with `[S, $]`?
- [ ] Explain the **general algorithm for eliminating all left recursion**. What ordering constraint is required?

---

## 📚 Quick Reference

```
FIRST(X):  terminals that can start a string derived from X
FOLLOW(A): terminals that can appear right AFTER A in any sentential form
PREDICT(A→α): FIRST(α) ∪ (FOLLOW(A) if ε ∈ FIRST(α))

Grammar is LL(1) iff: for all A→α, A→β (α≠β):
  PREDICT(A→α) ∩ PREDICT(A→β) = ∅

Parse table: M[A, a] = A→α  iff  a ∈ PREDICT(A→α)

Left recursion fix:  A → Aα | β  becomes  A → βA'  and  A' → αA' | ε
Left factoring fix:  A → αβ₁ | αβ₂  becomes  A → αA'  and  A' → β₁ | β₂
```
