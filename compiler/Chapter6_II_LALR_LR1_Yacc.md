# 📄 Bottom-Up Parsing II — LALR(k), LR(k), Yacc & Symbol Tables

**Tags:** #compiler #parsing #LALR #LR1 #Yacc #symbol-table #NCKU
**Links:** [[Bottom-Up Parsing I]], [[SLR Parsing]], [[First-Follow Sets]], [[Lexical Analysis]], [[CFG]]

---

## 🎯 The "Elevator Pitch"
> Chapter 6-II answers the question: *"SLR(1) was smart — can we do better without exploding the table size?"* The answer is **LALR(1)**: it computes tighter, per-state lookaheads instead of global Follow sets, giving it the power of full LR(1) at the cost of only LR(0)-sized tables. Then we see how **Yacc** turns this theory into a real parser generator, and how **symbol tables** connect the parser to the rest of the compiler.

---

## 🧠 Part 1 — Why SLR(1) Isn't Enough: The Motivation

Recall from Chapter 6-I:

| Method | Lookahead Source | Problem |
|---|---|---|
| LR(0) | None | Reduces on every token — too many conflicts |
| **SLR(1)** | Global `Follow(A)` | Follow sets are **too coarse** — they ignore *which state* we're in |
| **LALR(1)** | Per-state lookahead sets | Much tighter — resolves most real conflicts |
| LR(1) | Full 1-token lookahead per item | Perfect, but exponentially larger table |

### 💡 The Core Problem with SLR(1) — Intuition

Imagine the grammar has `A → α •` in state Iᵢ and `Follow(A) = {a, b}`. SLR says "reduce by `A → α` on both `a` and `b`". But what if in state Iᵢ, only `a` is a *reachable* follow token? `b` is in `Follow(A)` globally (from some other context), but not from *this particular* state.

SLR uses a **global telescope**. LALR uses a **per-state microscope**.

---

## 🧠 Part 2 — LR(1) Items: Adding Lookahead to Items

### The LR(1) Item Format

An **LR(1) item** is an LR(0) item **plus a lookahead token**:

```
[ A → α • β,  a ]
```

- `A → α • β` — the LR(0) core (dot position in the production)
- `a` — the **lookahead terminal**: we only reduce by `A → αβ` if the next input token is `a`

This lookahead is *specific to the derivation path that led to this state*, not a global Follow set.

### LR(1) Closure

The closure rule for LR(1) must propagate the lookahead:

```
If  [ A → α • B β,  a ]  is in the set,
Then for each production  B → γ,
    add  [ B → • γ,  b ]  for each b ∈ First(β a)
```

**Key insight**: the new lookahead `b` is derived from `First(βa)` — what could follow `B` given that after the `B` we still have `β` to parse, and then `a` beyond that.

```python
def lr1_closure(items, grammar, first):
    """
    items: set of (lhs, rhs_tuple, dot, lookahead)
    Compute LR(1) closure.
    """
    result = set(items)
    worklist = list(items)

    while worklist:
        lhs, rhs, dot, lookahead = worklist.pop()

        # If dot is before a nonterminal B: [A → α • B β, a]
        if dot < len(rhs) and rhs[dot] in grammar.nonterminals:
            B = rhs[dot]
            beta = rhs[dot + 1:]          # β (symbols after B)

            # Compute First(β a): what can follow B in this context?
            beta_a = beta + (lookahead,)
            lookaheads = first_of_string(beta_a, grammar, first)

            for B_rhs in grammar.productions[B]:
                for b in lookaheads:
                    new_item = (B, B_rhs, 0, b)
                    if new_item not in result:
                        result.add(new_item)
                        worklist.append(new_item)

    return frozenset(result)
```

### LR(1) GOTO

Same structure as LR(0) GOTO, but now we carry the lookahead through:

```python
def lr1_goto(item_set, symbol, grammar, first):
    """Advance dot past symbol, then LR(1) closure."""
    moved = frozenset(
        (lhs, rhs, dot + 1, la)
        for lhs, rhs, dot, la in item_set
        if dot < len(rhs) and rhs[dot] == symbol
    )
    return lr1_closure(moved, grammar, first) if moved else frozenset()
```

### LR(1) Table Construction

Fill ACTION and GOTO:

```python
def build_lr1_table(states, transitions, grammar):
    ACTION, GOTO_TABLE = {}, {}

    for i, state in enumerate(states):
        for lhs, rhs, dot, la in state:

            if dot < len(rhs):
                X = rhs[dot]
                j = transitions.get((i, X))
                if j is None:
                    continue
                if X in grammar.terminals:
                    ACTION[(i, X)] = ('shift', j)    # shift on terminal
                else:
                    GOTO_TABLE[(i, X)] = j            # goto on nonterminal

            else:
                # Reduce item: [A → α •, a] — reduce ONLY when next token == a
                if lhs == "S'":
                    ACTION[(i, '$')] = ('accept',)
                else:
                    key = (i, la)    # ← use the ITEM's lookahead, not Follow(A)!
                    ACTION[key] = ('reduce', lhs, rhs)

    return ACTION, GOTO_TABLE
```

> ⚠️ **This is the critical difference from SLR(1)**: reduce on `la` (the item's lookahead), not `Follow(lhs)`.

---

## 🧠 Part 3 — LALR(1): The Best of Both Worlds

### The Problem with Full LR(1)

LR(1) is maximally powerful — but it creates **too many states**. For a grammar with `n` LR(0) states, the LR(1) table can have **O(n × |terminals|)** states — impractically large for real languages.

### The LALR Insight: Merge States with the Same LR(0) Core

Two LR(1) states are **core-equivalent** if they have the same set of LR(0) items (same dot positions), differing only in their lookaheads. LALR merges these states by **unioning their lookahead sets**.

```
LR(1) states I₄ and I₇ both have core:
  { A → α •, ... }

I₄ lookaheads = {a, b}
I₇ lookaheads = {c, d}

LALR merged state I₄₇:
  { A → α •, a/b/c/d }     ← union of lookaheads
```

This collapses the LR(1) table back to LR(0) size, while retaining most of LR(1)'s precision.

### 💡 Analogy — LALR as "Smart Compression"

Think of LR(1) states as **high-resolution photos** and LR(0) states as **thumbnails**. LALR is like keeping the thumbnail layout but annotating each thumbnail with metadata (lookahead sets) that tells you when to reduce. You get most of the precision at a fraction of the storage cost.

---

## 🧠 Part 4 — LALR via Propagation Graph (The Practical Algorithm)

Rather than building the full LR(1) table and merging, the efficient approach is:

1. Build the LR(0) canonical collection (cheap).
2. Compute **spontaneous lookaheads** — lookaheads that appear directly from the LR(1) closure rule.
3. Build a **propagation graph** — edges that describe how lookaheads *flow* from one item to another as the DFA transitions.
4. Propagate until fixed-point (no new lookaheads).

### Step-by-Step

#### Step 1: Initialize

For the augmented start item `[S' → • S, $]` in state I₀:
- Seed lookahead `$` **spontaneously**.
- All other items start with **no** lookahead (empty set).

#### Step 2: Spontaneous vs. Propagated Lookaheads

For each kernel item `[A → α • X β, #]` in each state Iᵢ
(where `#` is a dummy placeholder token, not a real terminal):

Compute `lr1_closure({[A → α • X β, #]})`.

For each item `[B → γ • δ, b]` in the closure result:
- If `b ≠ #` → lookahead `b` is **spontaneously generated** for item `[B → γ • δ]` in `GOTO(Iᵢ, X)`.
- If `b = #` → there is a **propagation edge**: the lookahead of `[A → α • X β]` in Iᵢ propagates to `[B → γ • δ]` in `GOTO(Iᵢ, X)`.

```python
DUMMY = '#'   # sentinel token

def compute_lookaheads(states, transitions, grammar, first):
    """
    Compute spontaneous lookaheads and propagation edges for LALR.
    Returns: (lookaheads dict, propagation edges list)
    """
    # lookaheads[state_idx][item] = set of terminal lookaheads
    lookaheads = {i: {item: set() for item in state} for i, state in enumerate(states)}
    propagates = []   # list of ((src_state, src_item), (dst_state, dst_item))

    # Seed: S' → •S gets lookahead $
    start_item = next(it for it in states[0] if it[0] == "S'" and it[2] == 0)
    lookaheads[0][start_item].add('$')

    for i, state in enumerate(states):
        # Only process kernel items (dot not at position 0, except S'→•S)
        kernel = {it for it in state if it[2] > 0 or it[0] == "S'"}

        for item in kernel:
            lhs, rhs, dot = item
            # Try each symbol X that could follow the dot
            if dot >= len(rhs):
                continue
            X = rhs[dot]

            # Compute LR(1) closure using DUMMY as lookahead
            seed_item = (lhs, rhs, dot, DUMMY)
            closed = lr1_closure({seed_item}, grammar, first)

            for (B, B_rhs, B_dot, b) in closed:
                # This item will appear in GOTO(Iᵢ, X)
                dst_state_idx = transitions.get((i, X))
                if dst_state_idx is None:
                    continue
                dst_item = (B, B_rhs, B_dot + 1)   # after advancing dot

                if b != DUMMY:
                    # Spontaneous: b is generated regardless of src lookahead
                    lookaheads[dst_state_idx][dst_item].add(b)
                else:
                    # Propagation: src item's lookahead flows to dst item
                    propagates.append(((i, item), (dst_state_idx, dst_item)))

    return lookaheads, propagates


def propagate_lookaheads(lookaheads, propagates):
    """Fixed-point propagation: keep pushing lookaheads along edges."""
    changed = True
    while changed:
        changed = False
        for (src_state, src_item), (dst_state, dst_item) in propagates:
            new_las = lookaheads[src_state][src_item] - lookaheads[dst_state][dst_item]
            if new_las:
                lookaheads[dst_state][dst_item] |= new_las
                changed = True
    return lookaheads
```

### Building LALR Table from Propagated Lookaheads

After propagation, each LR(0) item `[A → α •]` in state Iᵢ has a precise set of lookaheads. Fill the table exactly like LR(1) but using these propagated sets:

```python
def build_lalr_table(states, transitions, grammar, final_lookaheads):
    ACTION, GOTO_TABLE = {}, {}

    for i, state in enumerate(states):
        for item in state:
            lhs, rhs, dot = item

            if dot < len(rhs):
                X = rhs[dot]
                j = transitions.get((i, X))
                if j is None:
                    continue
                if X in grammar.terminals:
                    ACTION[(i, X)] = ('shift', j)
                else:
                    GOTO_TABLE[(i, X)] = j
            else:
                # Reduce: use propagated lookaheads, not global Follow
                if lhs == "S'":
                    ACTION[(i, '$')] = ('accept',)
                else:
                    for la in final_lookaheads[i][item]:
                        key = (i, la)
                        new_act = ('reduce', lhs, rhs)
                        if key in ACTION and ACTION[key] != new_act:
                            print(f"LALR CONFLICT at state {i} on '{la}'")
                        ACTION[key] = new_act

    return ACTION, GOTO_TABLE
```

---

## 🧠 Part 5 — LALR Conflicts: The One Failure Mode

Merging states can introduce **reduce/reduce conflicts** that did not exist in the full LR(1) table. This happens when two LR(1) states with the same LR(0) core had *disjoint* lookahead sets — after merging, the lookaheads union and may now overlap.

### Classic Example: The Reduce/Reduce Conflict After Merging

```
LR(1) State A: [X → α •, a]   [Y → β •, b]    (a ≠ b → no conflict)
LR(1) State B: [X → α •, b]   [Y → β •, a]    (a ≠ b → no conflict)

LALR merged:   [X → α •, a/b] [Y → β •, a/b]  (both reduce on a and b → CONFLICT!)
```

**Implication**: LALR(1) is strictly less powerful than LR(1), but in practice the difference is negligible for programming language grammars.

### Power Hierarchy (memorize this!)

```
LR(0) ⊂ SLR(1) ⊂ LALR(1) ⊂ LR(1)
  ↑weak                        ↑strongest
```

All are subsets of unambiguous CFGs. LR(1) = all deterministically parseable languages. LALR(1) is what yacc/bison use.

---

## 🧠 Part 6 — Full LR(k) Table Construction

LR(k) generalizes the lookahead to k tokens. In practice k=1 is universal; k>1 is rarely needed and produces astronomical table sizes.

### LR(k) Item Format

```
[ A → α • β,  w ]    where w ∈ Σ*, |w| ≤ k
```

The lookahead string `w` has length at most `k`. For k=1, it's a single terminal.

### LR(k) Closure

```
If [ A → α • B β, w ] is in the set,
For each production B → γ,
  For each x ∈ Firstₖ(βw),
    Add [ B → • γ, x ]
```

Where `Firstₖ(βw)` = the set of strings of length ≤ k derivable from the beginning of `βw`.

```python
def firstk(symbols, k, grammar, memo={}):
    """
    Compute First_k(symbols): all strings of length ≤ k
    derivable from the beginning of symbols.
    """
    if not symbols:
        return {''}    # empty string (ε)

    key = (symbols, k)
    if key in memo:
        return memo[key]

    head, tail = symbols[0], symbols[1:]
    result = set()

    if head in grammar.terminals:
        for s in firstk(tail, k - 1, grammar, memo):
            combined = head + s
            if len(combined) <= k:
                result.add(combined)
    else:
        for rhs in grammar.productions[head]:
            for prefix in firstk(rhs + tail, k, grammar, memo):
                result.add(prefix[:k])

    memo[key] = result
    return result
```

---

## 🧠 Part 7 — Yacc: Theory Meets Practice

### What is Yacc?

> **Yacc** (Yet Another Compiler Compiler) is a LALR(1) parser generator. You write a grammar with semantic actions; Yacc generates a C parser (`y.tab.c`) that implements the LALR parse table.

### Yacc File Structure

```
%{
  /* C declarations, includes */
  #include <stdio.h>
%}

/* Token declarations */
%token DIGIT ID

/* Precedence / associativity declarations */
%left '+' '-'
%left '*' '/'
%right UMINUS

%%

/* Grammar rules with semantic actions */
expr : expr '+' expr   { $$ = $1 + $3; }
     | expr '*' expr   { $$ = $1 * $3; }
     | '(' expr ')'    { $$ = $2; }
     | DIGIT           { $$ = $1; }
     ;

%%

/* Supporting C code (yylex, main, etc.) */
```

### The `$$`, `$1`, `$2`, ... Notation

```
rule : A B C  { $$ = f($1, $2, $3); }
```

- `$$` = **synthesized attribute** of the LHS (the result)
- `$1`, `$2`, `$3` = attributes of the 1st, 2nd, 3rd RHS symbols

Yacc uses `yylval` as the channel between the lexer and parser:

```c
/* In the lexer (yylex): */
yylval = atoi(yytext);    // store token value before returning
return DIGIT;

/* In the parser: $1 etc. reads from the value stack */
```

### The Yacc Compilation Pipeline

```
grammar.y  ──yacc──►  y.tab.c  ──gcc──►
                       y.tab.h           ╮
tokens.l   ──lex───►  lex.yy.c ──gcc──► parser binary
                                         ╯ (links -ly -ll)
```

```bash
yacc -d grammar.y        # produces y.tab.c and y.tab.h
lex tokens.l             # produces lex.yy.c
gcc y.tab.c lex.yy.c -ly -ll -o parser
```

### Yacc's Disambiguating Rules (for Conflicts)

Yacc resolves conflicts automatically using these rules (in order):

| Rule | Conflict Type | Resolution |
|---|---|---|
| **1** | Shift/Reduce | **Default: shift** |
| **2** | Reduce/Reduce | **Default: reduce by earlier rule** (closer to top of grammar file) |
| **3** | Declared precedence | Higher precedence → preferred action |
| **4** | Tie in precedence | Use **associativity**: `left` → reduce, `right` → shift, `nonassoc` → error |
| **5** | `%prec` override | Forces production to use specified token's precedence |

### Precedence & Associativity Deep Dive

```yacc
%left  '+' '-'    /* lower precedence, left-associative */
%left  '*' '/'    /* higher precedence (later = higher), left-associative */
%right UMINUS     /* unary minus: highest, right-associative */

expr : expr '+' expr   { $$ = $1 + $3; }
     | expr '-' expr   { $$ = $1 - $3; }
     | expr '*' expr   { $$ = $1 * $3; }
     | '-' expr %prec UMINUS  { $$ = -$2; }  /* use UMINUS precedence */
     ;
```

**How conflict resolution works for `3 + 4 * 5`:**
- Stack has `expr + expr`, next token is `*`
- Production `expr + expr` has precedence of `+` (left)
- Lookahead `*` has higher precedence than `+`
- Shift wins → correct! (`4 * 5` grouped first)

**How conflict resolution works for `3 + 4 + 5`:**
- Stack has `expr + expr`, next token is `+`
- Same precedence as `+`, associativity is `left`
- Left → reduce → correct! (`(3 + 4) + 5`)

```python
def resolve_shift_reduce(action_prec, action_assoc, lookahead_prec):
    """
    Yacc's shift/reduce conflict resolution logic.
    Returns 'shift' or 'reduce'.
    """
    if action_prec is None or lookahead_prec is None:
        return 'shift'   # Rule 1: default shift

    if lookahead_prec > action_prec:
        return 'shift'   # lookahead has higher precedence
    elif lookahead_prec < action_prec:
        return 'reduce'  # production has higher precedence
    else:
        # Equal precedence → use associativity
        if action_assoc == 'left':
            return 'reduce'
        elif action_assoc == 'right':
            return 'shift'
        else:  # 'nonassoc'
            return 'error'
```

### Yacc and Symbol Table Integration

The parser doesn't just build a parse tree — it executes semantic actions that interact with the **symbol table**. The symbol table stores information about identifiers (variables, functions, types).

```c
/* Grammar rule with symbol table interaction */
declaration : type ID ';'
    {
        Symbol *s = lookup($2);          /* look up ID in symbol table */
        if (s != NULL) {
            yyerror("redeclaration of variable");
        } else {
            insert($2, $1);              /* insert ID with its type */
        }
    }
    ;
```

---

## 🧠 Part 8 — Symbol Tables

### What is a Symbol Table?

> A **symbol table** is a data structure that maps **identifiers** (names) to their **attributes** (type, scope, memory location, etc.). It is the central repository of semantic information built during parsing and used during code generation.

### Typical Symbol Table Entry

```c
typedef struct Symbol {
    char    *name;        // identifier string
    Type     type;        // int, float, char*, struct, function, ...
    Scope    scope;       // global, local, block level
    int      offset;      // memory offset from frame pointer
    int      line;        // source line (for error reporting)
    // for functions:
    Type    *param_types; // parameter type list
    int      n_params;    // number of parameters
} Symbol;
```

### Symbol Table Operations

| Operation | Description |
|---|---|
| `insert(name, attrs)` | Add a new identifier entry |
| `lookup(name)` | Find an identifier (search current scope inward) |
| `delete(name)` | Remove an identifier (on scope exit) |
| `open_scope()` | Push a new scope onto the scope stack |
| `close_scope()` | Pop the current scope |

### Scope Management with a Stack of Hash Tables

```python
class SymbolTable:
    """
    Scoped symbol table: a stack of hash tables.
    Each hash table represents one scope level.
    """
    def __init__(self):
        self.scopes = [{}]   # start with global scope

    def open_scope(self):
        """Enter a new block/function scope."""
        self.scopes.append({})

    def close_scope(self):
        """Exit the current scope — all locals are now gone."""
        if len(self.scopes) > 1:
            self.scopes.pop()
        else:
            raise RuntimeError("Cannot close global scope")

    def insert(self, name, attributes):
        """Insert into the CURRENT (innermost) scope."""
        current = self.scopes[-1]
        if name in current:
            raise SemanticError(f"Redeclaration of '{name}' in current scope")
        current[name] = attributes

    def lookup(self, name):
        """
        Search from innermost scope outward (dynamic scoping order).
        Returns attributes or None if not found.
        """
        for scope in reversed(self.scopes):
            if name in scope:
                return scope[name]
        return None   # undeclared identifier

    def lookup_current_scope(self, name):
        """Only look in the current scope (for redeclaration checks)."""
        return self.scopes[-1].get(name, None)
```

### Symbol Table in Lex + Yacc Pipeline

```
Source code
    │
    ▼
┌───────────────────────────────────────────────┐
│  Lexer (lex.yy.c)                             │
│  - tokenizes input                            │
│  - stores string value in yylval              │
│  - returns token type (DIGIT, ID, etc.)       │
└──────────────────────┬────────────────────────┘
                       │ token stream
                       ▼
┌───────────────────────────────────────────────┐
│  Parser (y.tab.c — LALR(1))                  │
│  - runs shift/reduce decisions                │
│  - executes semantic actions { $$ = ... }    │
│  - calls insert/lookup on symbol table        │
└──────────────────────┬────────────────────────┘
                       │ annotated parse tree / IR
                       ▼
              Code Generation / Type Checking
```

---

## ⚠️ Edge Cases & Constraints

- **LALR merge conflicts**: If merging two LR(1) states introduces a reduce/reduce conflict, the grammar is LALR(1) but not LR(1)... wait, that means the grammar *is* LR(1) but *not* LALR(1). You must use a full LR(1) parser in that rare case.
- **Yacc Rule 1 (default shift)** silently handles the dangling-else problem — but silently. Always check `yacc -v` output for conflict warnings.
- **`%prec` misuse**: placing `%prec` on the wrong rule can silently introduce parse errors. Always validate with test inputs.
- **Symbol table scope bugs**: forgetting to `close_scope()` on all exit paths (including error recovery) causes memory leaks and scope contamination.
- **`yylval` type conflicts**: in multi-type parsers, `yylval` is a C union — using the wrong field reads garbage. Always use `%union` and typed `%token`/`%type` declarations.
- **LR(k) for k>1** is almost never used — the table size grows as O(|states| × |Σ|ᵏ), which is intractable.

---

## 💻 Full Integration Sketch: Mini Calculator in Yacc Style

```c
/* calc.y — minimal calculator grammar */
%{
#include <stdio.h>
int yylex(void);
void yyerror(char *s);
%}

%union {
    int ival;    /* for integer tokens and expressions */
}

%token <ival> NUMBER
%type  <ival> expr

%left  '+' '-'
%left  '*' '/'
%right UMINUS

%%

program : expr '\n'   { printf("= %d\n", $1); }
        ;

expr : expr '+' expr        { $$ = $1 + $3; }
     | expr '-' expr        { $$ = $1 - $3; }
     | expr '*' expr        { $$ = $1 * $3; }
     | expr '/' expr        { $$ = ($3 == 0) ? (yyerror("div by zero"), 0) : $1 / $3; }
     | '(' expr ')'         { $$ = $2; }
     | '-' expr %prec UMINUS { $$ = -$2; }
     | NUMBER               { $$ = $1; }
     ;

%%

void yyerror(char *s) { fprintf(stderr, "Error: %s\n", s); }
```

```c
/* calc.l — lexer for the calculator */
%{
#include "y.tab.h"
%}

%%
[0-9]+   { yylval.ival = atoi(yytext); return NUMBER; }
[ \t]    { /* skip whitespace */ }
.        { return yytext[0]; }   /* return any other char as-is */
\n       { return '\n'; }
%%
```

```python
# Equivalent Python pseudocode showing the value stack mechanism
def execute_semantic_action(rule, value_stack):
    """
    Simulate how Yacc's value stack works during reductions.
    value_stack holds synthesized attributes of symbols.
    """
    if rule == "expr → expr '+' expr":
        right = value_stack.pop()   # $3
        value_stack.pop()           # '+' (no value)
        left  = value_stack.pop()   # $1
        value_stack.push(left + right)   # $$ = $1 + $3

    elif rule == "expr → '-' expr":     # UMINUS rule
        e = value_stack.pop()       # $2
        value_stack.pop()           # '-'
        value_stack.push(-e)        # $$ = -$2

    elif rule == "expr → NUMBER":
        # Already on stack as $1 = yylval from lexer
        pass   # $$ = $1 implicitly
```

---

## 🔗 Complete Concept Map

```
Chapter 6-II
│
├── LALR(1)
│     ├── Motivation: SLR Follow sets are too coarse
│     ├── LR(1) items: [A → α•β, a] — lookahead per item
│     ├── LR(1) Closure: propagate First(βa) as new lookaheads
│     ├── LALR = merge LR(1) states with same LR(0) core
│     ├── Propagation graph: efficient LALR construction
│     │     ├── Spontaneous lookaheads (from closure with dummy #)
│     │     └── Propagated lookaheads (fixed-point iteration)
│     └── LALR can introduce reduce/reduce conflicts (rare)
│
├── LR(k)
│     ├── Lookahead strings of length k
│     ├── Firstₖ(βw) used in closure
│     └── k > 1 impractical (table explosion)
│
├── Power Hierarchy
│     LR(0) ⊂ SLR(1) ⊂ LALR(1) ⊂ LR(1) ⊂ unamb. CFG
│
├── Yacc
│     ├── LALR(1) parser generator
│     ├── Grammar + semantic actions → C parser
│     ├── Conflict resolution: shift/reduce default=shift
│     ├── Precedence/associativity: %left %right %nonassoc
│     ├── %prec override for unary operators
│     └── yylval: lexer→parser value channel
│
└── Symbol Table
      ├── Maps identifiers → attributes (type, scope, offset)
      ├── Scope stack: open_scope / close_scope
      ├── Operations: insert / lookup / delete
      └── Integrated via semantic actions in Yacc rules
```

---

## 📚 References

1. **Aho, Lam, Sethi, Ullman** — *Compilers: Principles, Techniques, and Tools* (龍書 / Dragon Book), 2nd ed., Chapters 4.7 (LALR), 4.8 (LR(k)), 4.9 (Yacc), 2.7 (Symbol Tables). The canonical reference for all material in this chapter.
2. **Cooper & Torczon** — *Engineering a Compiler*, 2nd ed., Chapter 3.5–3.6. Exceptionally clear treatment of LALR propagation graphs with step-by-step examples.
3. **Appel, A.** — *Modern Compiler Implementation in Java*, Chapter 3. Clean pseudocode for LALR lookahead propagation.
4. **DeRemer & Pennello (1982)** — *"Efficient Computation of LALR(1) Look-Ahead Sets"*, ACM TOPLAS. The original paper behind the propagation graph algorithm taught in this course.
5. **GNU Bison Manual** — https://www.gnu.org/software/bison/manual/ — Real-world LALR(1) implementation; covers `%prec`, `%union`, conflict resolution in detail.
6. **Johnson, S. C. (1975)** — *Yacc: Yet Another Compiler-Compiler*, Bell Labs Technical Report. The original Yacc paper — historically important.
7. **Grune & Jacobs** — *Parsing Techniques: A Practical Guide* (freely available PDF). Chapter 9 covers LR variants exhaustively.
8. **PLY (Python Lex-Yacc)** — https://www.dabeaz.com/ply/ — Python implementation of Lex+Yacc; great for experimenting with LALR(1) grammars without C toolchain friction.

---

## ❓ Active Recall

### Level 1 — Definitions
- [ ] What additional component does an LR(1) item have compared to an LR(0) item? What is it used for?
- [ ] What does it mean for two LR(1) states to be "core-equivalent"?
- [ ] Define: spontaneous lookahead vs. propagated lookahead in the LALR propagation graph algorithm.
- [ ] What are the four components of a Yacc grammar file? Name them.
- [ ] What is `yylval` and why is it needed?

### Level 2 — Mechanics
- [ ] Trace the LR(1) closure of `{ [S' → • E, $] }` for grammar `E → E + T | T`, `T → id`. Show how lookaheads are computed.
- [ ] In the propagation graph algorithm, what does the "dummy token `#`" achieve? Why can't we use a real terminal?
- [ ] Explain the steps to build an LALR(1) table starting from the LR(0) collection.
- [ ] Given a shift/reduce conflict in Yacc with declared precedences, walk through the resolution algorithm step by step.
- [ ] How does `%prec UMINUS` work for the rule `expr → '-' expr`?

### Level 3 — Analysis
- [ ] Why can LALR(1) have reduce/reduce conflicts that LR(1) doesn't? Give a concrete example.
- [ ] A grammar is parsed by SLR(1) but conflicts arise — list two things you can try to resolve this, in order of preference.
- [ ] Compare: what does `lookup(name)` return in each case — (a) the name is in scope, (b) declared in an outer scope, (c) not declared at all?
- [ ] Why is `open_scope()` and `close_scope()` necessary? What bug occurs if `close_scope()` is not called on error paths?

### Level 4 — Synthesis
- [ ] Write a Yacc grammar fragment for a simplified `if-else` statement that correctly handles the dangling-else using Yacc's default conflict resolution. Explain *why* default shift resolves it correctly.
- [ ] Given the LALR power hierarchy `LR(0) ⊂ SLR(1) ⊂ LALR(1) ⊂ LR(1)`, construct a grammar that is SLR(1) but not LR(0). Prove it by showing the conflicting LR(0) state and how SLR resolves it.
- [ ] Design a symbol table for a language with nested functions (closures). What extra fields does each entry need? How does `lookup` change?
