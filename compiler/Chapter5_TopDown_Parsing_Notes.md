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

### What does "LL(1)" actually mean?

The cryptic name decodes letter by letter:

| Letter | Meaning |
|---|---|
| **L** (1st) | **L**eft-to-right scan of the input |
| **L** (2nd) | **L**eftmost derivation is constructed |
| **k** (here `1`) | `k` tokens of **lookahead** (in practice, almost always `k = 1`) |

So an **LL(1) parser** scans input left-to-right, builds a leftmost derivation, and uses **a single lookahead token** to decide which production to apply.

> **Key promise of predictive parsing:** at every step there is **exactly one** production we can use — no backtracking. If we have configuration `ω A γ` (where `A` is the leftmost nonterminal) and the next token is `t`, then there is at most one production `A → α` that could possibly succeed on input `t`. Any other choice is guaranteed wrong.

The theory is developed for arbitrary `k`, but `k = 1` is what real parser generators produce. Larger `k` quickly explodes the size of the parse table.

### Motivation

Instead of trying alternatives blindly, we use a **single lookahead token** to *predict* which production to use.

**PREDICT(A → X₁X₂...Xₘ) =**
- `FIRST(X₁X₂...Xₘ)` — if it doesn't contain ε
- `FIRST(X₁X₂...Xₘ) \ {ε} ∪ FOLLOW(A)` — if it contains ε

### 🔍 Why Two Cases? (FIRST vs. FOLLOW intuition)

Suppose the parser is staring at:
- leftmost nonterminal `A` on top of stack
- next input token `t`

Under what circumstances should we apply `A → α`?

**Case 1 — α can produce `t` in the first position.**
There is some derivation `α ⟹* t β`. Then expanding `A → α` is a great move — eventually the leading `t` of α's expansion will line up against the input. We say `t ∈ FIRST(α)`.

**Case 2 — α cannot produce `t`, but α can vanish entirely.**
That is, `α ⟹* ε`. The `A` on the stack has no hope of matching `t` itself, but if α erases, then *whatever symbol comes after A in the surrounding sentential form* gets a chance to match `t`. So we still want this move, provided that `t` is something that can legally appear *right after `A`* in some derivation. We say `t ∈ FOLLOW(A)`.

> ⚠️ **Subtlety students miss:** FOLLOW(A) has nothing to do with what A *produces*. It describes where A is *used* in the grammar — what tokens can legally appear immediately after A in some sentential form derived from the start symbol.

### 📌 FIRST Sets

`FIRST(X)` = the set of terminals that can begin any string derived from X.

**Algorithm to compute FIRST:**
1. If X is a terminal: `FIRST(X) = {X}`
2. If X is a nonterminal and X → ε exists: add ε to FIRST(X)
3. If X → Y₁Y₂...Yₖ, add FIRST(Y₁) to FIRST(X). If ε ∈ FIRST(Y₁), also add FIRST(Y₂), etc.
4. Repeat until no changes.

```python
def compute_first(grammar):
    """
    Dragon Book Algorithm (FIRST set construction).
    Iterates until a fixed point is reached.
    """
    first = {sym: set() for sym in grammar.all_symbols}

    # Rule 1: terminals: FIRST(t) = {t}
    for t in grammar.terminals:
        first[t].add(t)

    changed = True
    while changed:
        changed = False
        for (A, production) in grammar.productions:
            before = len(first[A])    # snapshot BEFORE processing this production

            # Walk symbols left-to-right; stop when one cannot derive ε.
            i = 0
            while i < len(production):
                Yi = production[i]
                first[A] |= (first[Yi] - {'ε'})    # Rule 3: add FIRST(Yi) \ {ε}
                if 'ε' not in first[Yi]:
                    break                          # Yi cannot vanish; stop here
                i += 1
            else:
                first[A].add('ε')                  # Rule 2: every Yi derives ε

            if len(first[A]) > before:
                changed = True

    return first


def first_of_string(symbols, first):
    """
    FIRST of an arbitrary string α = X₁X₂...Xₙ.
    Used when filling parse-table entries — see build_parse_table().
    """
    result = set()
    for X in symbols:
        result |= (first[X] - {'ε'})
        if 'ε' not in first[X]:
            return result
    result.add('ε')   # every Xi can derive ε ⇒ α can derive ε
    return result
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

### 🤔 What about FOLLOW sets for *terminals*?

FOLLOW is well-defined for **any grammar symbol**, not just nonterminals — it's purely a question of "what tokens can appear immediately to my right in some sentential form?". The same algorithm applies; you just don't restrict the iteration to nonterminals.

**However: the LL(1) parse-table construction never reads FOLLOW of a terminal.** Looking at the algorithm:
- Case (a) needs `FIRST(α)` of production right-hand sides.
- Case (b) needs `FOLLOW(A)` where `A` is the **left-hand side** of a production — which is always a nonterminal.

So most implementations (including [`compute_follow`](#) above) skip terminals as a micro-optimization. But conceptually, you can still compute them — and doing so is a great sanity check on your understanding.

**Worked example using the course grammar** `E → T X, X → +E|ε, T → (E)|int Y, Y → *T|ε`:

| Terminal | Where it appears | Reasoning | FOLLOW |
|---|---|---|---|
| `(` | `T → ( E )` | followed by `E` ⇒ FIRST(E) | `{ (, int }` |
| `)` | `T → ( E )` | end of production ⇒ FOLLOW(T) | `{ +, ), $ }` |
| `+` | `X → + E` | followed by `E` ⇒ FIRST(E) | `{ (, int }` |
| `*` | `Y → * T` | followed by `T` ⇒ FIRST(T) | `{ (, int }` |
| `int` | `T → int Y` | followed by `Y`, and Y⟹ε so also FOLLOW(T) | `{ *, +, ), $ }` |

Each result is a **sanity check on the language**:
- After a `+` you must see another expression → it must start with `(` or `int`. ✓
- After an `int` you can see an operator (`*`, `+`), or close-paren, or end-of-input — but **never another `(`** (no juxtaposition without an operator). ✓
- After `*` only operands can follow (no `*+`, no `**`). ✓

If your computed FOLLOW set for a terminal contradicts your linguistic intuition about the language, that's a strong signal you have a bug in the grammar (or in your algorithm).

### 📌 LL(1) Predict Function — Full Definition

A grammar G is **LL(1)** if and only if for any two distinct productions `A → α` and `A → β`:
1. `FIRST(α) ∩ FIRST(β) = ∅` (no shared first symbols)
2. At most one of α, β can derive ε
3. If `β ⟹* ε`, then `FIRST(α) ∩ FOLLOW(A) = ∅`

*Intuition: Condition 1 and 3 together ensure you can always pick the **right** production using just **one lookahead token**.*

> ⚠️ **No ambiguous grammar and no left-recursive grammar can be LL(1).**

---

## 🧠 Part 4: The LL(1) Parse Table

https://www.youtube.com/watch?v=DT-cbznw9aY

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
    """
    Dragon Book Algorithm 4.31 — Construction of a predictive parsing table.
    For each production A → α:
      (a) ∀ a ∈ FIRST(α), a ≠ ε:        M[A, a] = A → α
      (b) if ε ∈ FIRST(α):
              ∀ b ∈ FOLLOW(A):           M[A, b] = A → α   (covers $ too)
      Any cell filled twice ⇒ grammar is NOT LL(1).
    """
    table = {}  # table[(A, a)] = production

    for (A, production) in grammar.productions:
        first_of_prod = first_of_string(production, first)

        # Case (a)
        for a in first_of_prod - {'ε'}:
            if (A, a) in table:
                raise Exception(f"Grammar is NOT LL(1): conflict at M[{A},{a}]")
            table[(A, a)] = production

        # Case (b) — handles the special $ case naturally because $ ∈ FOLLOW(A)
        if 'ε' in first_of_prod:
            for b in follow[A]:
                if (A, b) in table:
                    raise Exception(f"Grammar is NOT LL(1): conflict at M[{A},{b}]")
                table[(A, b)] = production

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
    """
    Dragon Book Algorithm 4.34 — Table-driven predictive parsing.
    Stack grows to the right; stack[-1] is the top.
    """
    stack = ['$', grammar.start_symbol]   # $ marks bottom of stack
    input_string += '$'                   # $ marks end of input
    ip = 0                                # input pointer

    while stack[-1] != '$':
        X = stack[-1]
        a = input_string[ip]

        if X == a:                        # terminal on top matches input
            stack.pop()
            ip += 1
        elif X in grammar.terminals:      # terminal on top, but mismatch
            error(f"expected {X}, found {a}")
        elif (X, a) not in table:         # blank cell ⇒ syntax error
            error(f"no rule for M[{X}, {a}]")
        else:
            production = table[(X, a)]
            print(f"Output: {X} → {' '.join(production)}")   # leftmost derivation step
            stack.pop()                                      # pop X
            if production != ['ε']:
                for sym in reversed(production):             # push Yk, ..., Y2, Y1
                    stack.append(sym)                        # so Y1 ends up on top

    if input_string[ip] == '$':
        print("ACCEPT")
    else:
        error("unexpected trailing input")
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

### 🔬 End-to-End Worked Example (Course Grammar)

Let's run the full pipeline — from a "broken" grammar to a working LL(1) parser — on a small expression grammar.

**Step 0 — Original (not LL(1)) grammar:**
```
E → T + E | T
T → int * T | int | ( E )
```

This is **not** LL(1). Both productions for `E` start with `T` (common prefix), and the first two productions for `T` both start with `int`. With only one token of lookahead, the parser cannot pick between them.

> 🧭 **Why no left-recursion elimination here?** Recall the LL(1) checklist: a grammar must be **(a) left-factored** *and* **(b) free of left recursion**. The course grammar above is already **right-recursive** — `E → T + E` has the recursive `E` on the *right end*, not the front. So Part 6 has nothing to do; only Part 7 (left factoring) is needed.
>
> Compare with the textbook left-recursive expression grammar:
> ```
> E → E + T | T          ← left-recursive! E on the FRONT
> T → T * F | F          ← left-recursive!
> F → ( E ) | int
> ```
> This grammar would need **Part 6 first** (eliminate left recursion → produces `E → T E'`, `E' → + T E' | ε`, etc.), and **then** Part 7 if any common prefixes remained. The end result is the standard LL(1) form you saw in [Part 3's worked example](#-first-sets).
>
> **Trade-off worth noting:** eliminating left recursion forces the grammar to be right-recursive, which gives **right-associative** parse trees by default (e.g. `1+2+3` parses as `1+(2+3)`). For operators like `-` or `/` that's wrong, so you fix associativity later in semantic actions or AST construction — not in the grammar.

**Step 1 — Left factor:**
Pull out the common prefixes and introduce fresh nonterminals `X` (for E's tail) and `Y` (for T's tail):
```
E → T X
X → + E | ε
T → ( E ) | int Y
Y → * T | ε
```

**Step 2 — Compute FIRST sets:**

| Symbol | FIRST |
|---|---|
| `+`, `*`, `(`, `)`, `int` | each terminal is its own FIRST |
| `T` | `{ (, int }` |
| `E` | `{ (, int }` *(same as T, since T cannot derive ε)* |
| `X` | `{ +, ε }` |
| `Y` | `{ *, ε }` |

**Step 3 — Compute FOLLOW sets:**

Walk through where each nonterminal *appears* in the grammar:

| Nonterminal | Where it appears | FOLLOW |
|---|---|---|
| `E` | `(E)` and as start symbol | `{ ), $ }` |
| `X` | end of `E → T X` | inherits FOLLOW(E) = `{ ), $ }` |
| `T` | start of `E → T X` (so FIRST(X)\{ε}); X⟹ε so also FOLLOW(E) | `{ +, ), $ }` |
| `Y` | end of `T → int Y` | inherits FOLLOW(T) = `{ +, ), $ }` |

**Step 4 — Build the parse table:**

|     | `int` | `(` | `)` | `+` | `*` | `$` |
|-----|-------|-----|-----|-----|-----|-----|
| `E` | `T X` | `T X` |     |     |     |     |
| `X` |       |     | `ε` | `+ E` |   | `ε` |
| `T` | `int Y` | `( E )` |   |     |     |     |
| `Y` |       |     | `ε` | `ε` | `* T` | `ε` |

Every entry comes from **one simple rule** (the same Algorithm 4.31 from Part 4):

> For each production `A → α`:
> - **If α does not derive ε** → put `A → α` in `M[A, a]` for every `a ∈ FIRST(α)`.
> - **If α does derive ε** → put `A → α` in `M[A, b]` for every `b ∈ FOLLOW(A)`.

That's it. FIRST when the right-hand side produces something; FOLLOW when it vanishes. Applied to this grammar:

| Production | Derives ε? | Use which set | Cells filled |
|---|---|---|---|
| `E → T X`  | no  | FIRST(T X) = `{int, (}`     | `M[E, int]`, `M[E, (]` |
| `X → + E`  | no  | FIRST(+ E) = `{+}`          | `M[X, +]` |
| `X → ε`    | yes | FOLLOW(X) = `{), $}`        | `M[X, )]`, `M[X, $]` |
| `T → ( E )`| no  | FIRST((E)) = `{(}`          | `M[T, (]` |
| `T → int Y`| no  | FIRST(int Y) = `{int}`      | `M[T, int]` |
| `Y → * T`  | no  | FIRST(* T) = `{*}`          | `M[Y, *]` |
| `Y → ε`    | yes | FOLLOW(Y) = `{+, ), $}`     | `M[Y, +]`, `M[Y, )]`, `M[Y, $]` |

Empty cells are *errors*.

**Step 5 — Parse `int * int $`:**

| Step | Stack (top → right) | Input | Move |
|------|---------------------|-------|------|
| 1 | `E $` | `int * int $` | `M[E, int] = T X` → push `T X` |
| 2 | `T X $` | `int * int $` | `M[T, int] = int Y` → push `int Y` |
| 3 | `int Y X $` | `int * int $` | match `int`, advance |
| 4 | `Y X $` | `* int $` | `M[Y, *] = * T` → push `* T` |
| 5 | `* T X $` | `* int $` | match `*`, advance |
| 6 | `T X $` | `int $` | `M[T, int] = int Y` → push `int Y` |
| 7 | `int Y X $` | `int $` | match `int`, advance |
| 8 | `Y X $` | `$` | `M[Y, $] = ε` → pop `Y`, push nothing |
| 9 | `X $` | `$` | `M[X, $] = ε` → pop `X`, push nothing |
| 10 | `$` | `$` | **ACCEPT** |

Notice three things:

1. The **stack always holds the unprocessed frontier** of the parse tree, with the leftmost pending symbol on top.
2. **Terminals on top of the stack get matched** against input and consumed.
3. **Empty entries in the table are errors.** For example, `M[E, *]` is empty — if a `*` ever appeared where an expression was expected, the parser would reject immediately.

---

## 🧠 Part 6: Elimination of Left Recursion

https://www.youtube.com/watch?v=PFey5FpKlFM

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

## 🧠 Part 9: When LL(1) Isn't Enough

### Quick "is it definitely NOT LL(1)?" tests

These are sufficient (but **not** necessary) conditions to rule out LL(1):

| Property | Implication |
|---|---|
| Grammar is **left-recursive** | NOT LL(1) |
| Grammar is **not left-factored** (shared prefixes) | NOT LL(1) |
| Grammar is **ambiguous** | NOT LL(1) |
| Requires **>1 token of lookahead** | NOT LL(1) |

### The mechanical test

Even if a grammar passes all the quick checks above, the only way to *prove* it is LL(1) is to **actually build the parse table** and verify that **every entry contains at most one production**. A multiply-defined entry ⟺ grammar is not LL(1).

**Example of a non-LL(1) grammar:**
```
S → S A | B
```
The first production is left-recursive, so when the parser sees `S` on top of the stack and the next input begins with something in `FIRST(B)`, the table gets two entries — `S → S A` and `S → B` — both at `M[S, FIRST(B)]`. Conflict.

### The practical reality

Most real programming language grammars **are not LL(1)**. They have:
- ambiguous if/else,
- unbounded lookahead between declarations and expressions (`T x;` vs `f(x);`),
- left-recursive expression grammars that you really *do* want left-associative.

LL(1) is too weak to capture all of these constructs cleanly. This motivates more powerful formalisms — **LR(1)**, **LALR(1)**, **GLR**, **PEG** — which we cover in [[Bottom-Up Parsing]]. The good news: every concept in this chapter (FIRST, FOLLOW, parse tables, conflict analysis) carries over directly.

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

---

## 📖 References

### Primary Textbook (the "Dragon Book")
- Aho, A. V., Lam, M. S., Sethi, R., & Ullman, J. D. (2006). *Compilers: Principles, Techniques, and Tools* (2nd ed.). Addison-Wesley. — Chapter 4 ("Syntax Analysis"), §§4.4 ("Top-Down Parsing"), 4.4.2 (Recursive-Descent), 4.4.3 (FIRST and FOLLOW), 4.4.4 (LL(1) Grammars), 4.4.5 (Non-Recursive Predictive Parsing). [Companion PDF (Shanghai Tech mirror)](https://faculty.sist.shanghaitech.edu.cn/faculty/songfu/cav/Dragon-book.pdf) · [Wikipedia overview](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools)

### Lecture Series (source of the integrated transcripts)
- Aiken, A. *Compilers* (Stanford / Coursera). Lectures 7-01 to 7-04: Predictive Parsing, FIRST sets, FOLLOW sets, LL(1) Parsing Tables. [Stanford CS143 home](https://cs143.stanford.edu/) · [YouTube playlist](https://www.youtube.com/playlist?list=PLEAYkSg4uSQ3yc_zf_f1GOxl5CZo0LVBb) · [Internet Archive mirror](https://archive.org/details/academictorrents_e31e54905c7b2669c81fe164de2859be4697013a) · [edX self-paced](https://www.edx.org/learn/computer-science/stanford-university-compilers)

### Stanford CS143 Handouts
- Dill, D. L. *CS143 Notes: Parsing.* Stanford. [PDF](https://web.stanford.edu/class/archive/cs/cs143/cs143.1156/handouts/parsing.pdf) — formal treatment of nullable, FIRST, FOLLOW, and LL(1) table construction.
- Kjolstad, F. (instructor) & Aiken, A. (slide author). *CS143 Lecture 1.* [PDF](https://web.stanford.edu/class/cs143/lectures/lecture01.pdf)

### Additional Course Notes
- *Compilers: Class Notes* — NYU. [Fall 2006 edition](https://cs.nyu.edu/~gottlieb/courses/2000s/2006-07-fall/compilers/class-notes.html) · [Fall 2008 edition](https://cs.nyu.edu/~gottlieb/courses/2000s/2008-09-fall/compilers/class-notes.html)
- Karkare, A. *CS 335A: Compiler Design.* IIT Kanpur. [Course page](https://karkare.github.io/cs335/)
- Hasti, K. *CS 536 Reading: Top-Down Parsing.* University of Wisconsin–Madison. [HTML notes](https://pages.cs.wisc.edu/~hasti/cs536/readings/Topdown.html)
- Wilsey, P. *Parsing — Part II (Top-down parsing, left-recursion removal).* University of Cincinnati EECE 6083. [PDF](https://eecs.ceas.uc.edu/~wilseypa/classes/eece6083/lectureNotes/parsingLL.pdf)

### Left Recursion / Left Factoring (transformations)
- Moreno Maza, M. *Elimination of left recursion.* University of Western Ontario CS447. [Lecture notes](https://www.csd.uwo.ca/~mmorenom/CS447/Lectures/Syntax.html/node8.html)
- Cockett, R. *LL(1) Transformations.* University of Calgary CPSC 411. [Course page](http://pages.cpsc.ucalgary.ca/~robin/class/411/LL1.3.html)
- *Left recursion.* Wikipedia. [Article](https://en.wikipedia.org/wiki/Left_recursion)
- *Left Recursion (LR) and Left Factoring (LF).* TutorialsPoint. [Tutorial](https://www.tutorialspoint.com/automata_theory/left_recursion_and_left_factoring.htm)

### Visualization & Practice
- Almasri, F. & Aboutabl, A. (2016). *The Use of a Predictive Parsing Visualization Tool (PPVT) in a Compiler Construction Course.* ACM Inroads. [Article](https://dl.acm.org/doi/fullHtml/10.1145/3002136)
- *Eliminating Left Recursion — Solved Problems.* YouTube. [Video](https://www.youtube.com/watch?v=PFey5FpKlFM&t=404s)

