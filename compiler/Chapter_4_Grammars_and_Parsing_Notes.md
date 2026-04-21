# 📄 Chapter 4: Grammars and Parsing

**Tags:** #compilers #parsing #CFG #grammars #syntax-analysis
**Links:** [[Lexical Analysis]], [[Context-Free Languages]], [[BNF Notation]], [[LL Parsing]], [[LR Parsing]]

---

## 🎯 The "Elevator Pitch"
> A **parser** reads a stream of tokens from the lexer and figures out *how they group together* — like solving a puzzle to reveal the grammar structure of your code. The result is a **parse tree** that the rest of the compiler uses to generate code.

---

---

# 📄 1. Parsing & Syntax Analysis

**Tags:** #parsing #syntax-analysis #parse-tree
**Links:** [[Lexical Analysis]], [[Intermediate Code Generation]]

---

## 🎯 The "Elevator Pitch"
> Parsing is the step after tokenizing: it groups tokens into meaningful structures (like sentences from words) and produces a tree showing how the code is organized.

## 🧠 Core Mechanics
* **Definition:** Syntax analysis (parsing) decides which parts of the incoming token stream should be grouped together and produces a **parse tree** as output.
* **Key Components:**
  * **Tokens → Parser:** The parser receives the token stream from the lexer.
  * **Parse Tree:** A hierarchical structure representing the grammatical structure of input.
  * **Intermediate Code Generator:** Transforms the parse tree into intermediate language (next compiler phase).

## ⚠️ Edge Cases & Constraints
* Parsing only checks **syntax**, not **semantics** (e.g., type errors are caught later).
* A valid parse tree doesn't guarantee a correct program — semantic analysis is still needed.

## 💻 Logical Code Snippet (Python)
```python
# Parsing pipeline: tokens → parse tree → intermediate code
def compile_pipeline(source_code):
    tokens = lexer(source_code)        # Step 1: Tokenize
    parse_tree = parser(tokens)        # Step 2: Syntax analysis → parse tree
    ir = intermediate_gen(parse_tree)  # Step 3: Transform to intermediate representation
    return ir
```

## ❓ Active Recall
* [ ] What is the input and output of a parser?
* [ ] How does the parser differ from the lexer?
* [ ] What does the intermediate code generator use as its input?

---

---

# 📄 2. Regular Expressions vs. Context-Free Grammars

**Tags:** #regular-expressions #CFG #comparison #modularization
**Links:** [[Finite Automata]], [[Context-Free Grammars]]

---

## 🎯 The "Elevator Pitch"
> Regular expressions are great for simple, flat patterns (like recognizing keywords and numbers), while context-free grammars handle the nested, recursive structure of full programming languages.

## 🧠 Core Mechanics
* **Definition:** Both formalisms describe languages, but CFGs are strictly **more powerful** — every RE can be expressed as a CFG, but not vice versa.
* **Key Components:**
  * **RE → FA → CFG:** Any finite automaton (from a regular expression) can be mechanically converted to a CFG.
  * **Why keep RE for tokens?**
    * RE ⇒ **easy & clear** description for tokens (e.g., identifiers, numbers).
    * RE ⇒ **efficient token recognizer** (DFA-based, O(n) time).
    * **Modularization:** Grammar rules use regular expressions as components — separation of concerns.

## ⚠️ Edge Cases & Constraints
* RE **cannot** describe nested/recursive structures (e.g., balanced parentheses `((()))`) — CFG is required.
* Using CFG for tokenization would be overkill and less efficient.

## 💻 Logical Code Snippet (Python)
```python
import re

# RE used for tokenization (fast, flat patterns)
TOKEN_PATTERNS = [
    ('NUMBER', r'\d+'),
    ('IDENT',  r'[a-zA-Z_]\w*'),
    ('PLUS',   r'\+'),
]

def tokenize(source):
    for name, pattern in TOKEN_PATTERNS:
        if re.match(pattern, source):
            return (name, re.match(pattern, source).group())

# CFG used at the grammar level (handles nesting)
# expression → expression + term | term
# (cannot be expressed cleanly with RE alone)
```

## ❓ Active Recall
* [ ] Can every CFG be expressed as a regular expression? Why or why not?
* [ ] Give two reasons why compilers still use regular expressions for lexing instead of CFGs.
* [ ] What does "modularizing components" mean in the context of RE + CFG?

---

---

# 📄 3. Context-Free Grammars (CFG) & BNF

**Tags:** #CFG #BNF #nonterminals #terminals #productions
**Links:** [[Regular Expressions]], [[Parse Tree]], [[Ambiguity]]

---

## 🎯 The "Elevator Pitch"
> A CFG is a set of rewriting rules that describe all valid sentences in a language. BNF (Backus-Naur Form) is just the notation we use to write those rules down.

## 🧠 Core Mechanics
* **Definition:** A **Context-Free Grammar** consists of:
  * **Terminals:** The actual tokens/symbols of the language (e.g., `+`, `id`, `(`)
  * **Nonterminals:** Abstract syntactic categories (e.g., `exp`, `term`, `factor`)
  * **Productions:** Rules of the form `A → α` (nonterminal → sequence of terminals/nonterminals)
  * **Start Symbol:** The nonterminal from which derivation begins

* **BNF Example (arithmetic expressions):**
  ```
  exp    → exp addop term | term
  addop  → + | -
  term   → term mulop factor | factor
  mulop  → *
  factor → ( exp ) | number
  ```

* **Key Capabilities of CFGs:**
  * Give **precise syntactic specification** of programming languages.
  * Parsers can be **constructed automatically** from a CFG.
  * Describe **nested structures**: balanced parentheses, matching `begin-end`, `if-then-else`, etc.
  * Syntax entities in CFG can guide **translation to object code**.

* **Historical Note:** BNF was introduced by **Backus & Naur in 1956** for describing natural language, then adopted for programming language specification.

## ⚠️ Edge Cases & Constraints
* **Flawed CFGs** can contain:
  * **Unreachable nonterminals:** Can't be reached from the start symbol (e.g., nonterminal `C` not reachable from `S`).
  * **Unproductive nonterminals:** Can never derive a terminal string (e.g., `B → Bb` with no base case).
* A CFG describes *syntax* only — it says nothing about *meaning*.

## 💻 Logical Code Snippet (Python)
```python
# Representing a CFG as a Python data structure
grammar = {
    'exp':    [['exp', 'addop', 'term'], ['term']],
    'addop':  [['+'], ['-']],
    'term':   [['term', 'mulop', 'factor'], ['factor']],
    'mulop':  [['*']],
    'factor': [['(', 'exp', ')'], ['number']],
}
start_symbol = 'exp'

def is_terminal(symbol):
    return symbol not in grammar  # terminals are not keys in grammar

def derive(nonterminal):
    # Return all possible right-hand sides for this nonterminal
    return grammar.get(nonterminal, [])
```

## ❓ Active Recall
* [ ] What are the four components of a CFG?
* [ ] What is BNF? How old is it, and what was it originally used for?
* [ ] Name two types of errors that can appear in a CFG.
* [ ] Why are CFGs useful for nested structures like `if-then-else`?

---

---

# 📄 4. Ambiguity in CFGs

**Tags:** #ambiguity #parse-tree #precedence #associativity #dangling-else
**Links:** [[CFG]], [[Disambiguation]], [[Operator Precedence]]

---

## 🎯 The "Elevator Pitch"
> An ambiguous grammar is one where the same string of tokens can be parsed in more than one way — like a sentence that has two different meanings. This is a problem because the compiler wouldn't know which meaning you intended.

## 🧠 Core Mechanics
* **Definition:** A CFG is **ambiguous** if there exists some sentence for which more than one **parse tree** (or equivalently, more than one leftmost/rightmost derivation) exists.

* **Classic Example:** The grammar `E → E + E | E * E | id` is ambiguous.
  * The sentence `id + id * id` has **two valid parse trees** — one grouping `+` first, one grouping `*` first.
  * This matters because `*` should have higher precedence than `+`.

* **Disambiguation Methods:**
  1. **Specify intention externally** (e.g., declare precedence and associativity rules separately):
     * Precedence (high → low): `( )` > negate > `↑` (exponent) > `* /` > `+ -`
     * Associativity: `↑` → **right**; all others → **left**
  2. **Rewrite the grammar** to encode precedence/associativity structurally:
     ```
     expression → expression + term | expression - term | term
     term       → term * factor | term / factor | factor
     factor     → primary ↑ factor | primary       ← right assoc
     primary    → -primary | element
     element    → ( expression ) | id
     ```
     Each level of nonterminal corresponds to one precedence level.

* **Dangling Else Problem:**
  * `stat → IF cond THEN stat | IF cond THEN stat ELSE stat` is ambiguous.
  * `if c1 then if c2 then s2 else s3` — which `if` does the `else` belong to?
  * **Unambiguous fix:**
    ```
    stat          → matched-stat | unmatched-stat
    matched-stat  → IF cond THEN matched-stat ELSE matched-stat | other-stat
    unmatched-stat→ IF cond THEN stat
                  | IF cond THEN matched-stat ELSE unmatched-stat
    ```

## ⚠️ Edge Cases & Constraints
* It is **undecidable** whether a given CFG is ambiguous (no general algorithm exists).
* Some languages are **inherently ambiguous** — no unambiguous CFG exists for them.
* Removing ambiguity by rewriting often makes the grammar larger and harder to read.

## 💻 Logical Code Snippet (Python)
```python
# Conceptual check: does a grammar produce multiple parse trees?
def is_ambiguous(grammar, start, sentence):
    parse_trees = find_all_parse_trees(grammar, start, sentence)
    return len(parse_trees) > 1  # ambiguous if more than one tree exists

# Disambiguation via precedence levels (structural encoding)
grammar_unambiguous = {
    'expression': [['expression', '+', 'term'], ['expression', '-', 'term'], ['term']],
    'term':       [['term', '*', 'factor'], ['term', '/', 'factor'], ['factor']],
    'factor':     [['primary', '↑', 'factor'], ['primary']],  # right-assoc
    'primary':    [['-', 'primary'], ['element']],
    'element':    [['(', 'expression', ')'], ['id']],
}
```

## ❓ Active Recall
* [ ] Define ambiguity in a CFG.
* [ ] Why is `id + id * id` ambiguous under the grammar `E → E+E | E*E | id`?
* [ ] What are the two strategies to resolve ambiguity?
* [ ] Explain the dangling else problem and how to fix it with grammar rewriting.
* [ ] Is it always possible to determine if a CFG is ambiguous? Why?

---

---

# 📄 5. Top-Down vs. Bottom-Up Parsing

**Tags:** #parsing #top-down #bottom-up #LL #LR #recursive-descent
**Links:** [[CFG]], [[Parse Tree]], [[LL Parsing]], [[LR Parsing]]

---

## 🎯 The "Elevator Pitch"
> Top-down parsing builds the parse tree from the root (start symbol) downward by making predictions; bottom-up parsing builds it from the leaves (tokens) upward by recognizing patterns.

## 🧠 Core Mechanics
* **Recognizer vs. Parser:**
  * **Recognizer:** Simply answers "Is this input syntactically valid?" (boolean).
  * **Parser:** Answers the above AND determines the structure (parse tree).

* **Two General Approaches:**

| Approach | Classic Method | Modern Method | Derivation Order |
|---|---|---|---|
| **Top-Down** | Recursive Descent | LL Parsing | Leftmost derivation |
| **Bottom-Up** | Operator Precedence | LR Parsing (shift-reduce) | Rightmost derivation (reversed) |

* **Top-Down Parsing:**
  * Expands the parse tree from root → leaves via **predictions**.
  * Performs a **preorder traversal** of the parse tree.
  * **Predictive in nature:** chooses which production to apply based on lookahead.
  * Named **LL**: scan input **L**eft-to-right, produce **L**eftmost derivation.

* **Bottom-Up Parsing:**
  * Starts at the leaves (terminal symbols) and reduces them to nonterminals.
  * Performs a **postorder traversal** of the parse tree.
  * Named **LR**: scan input **L**eft-to-right, produce **R**ightmost derivation (in reverse).

* **Naming Convention Summary:**
  * First letter = direction of input scan: **L** (left to right)
  * Second letter = derivation type: **L** (leftmost) or **R** (rightmost)

## ⚠️ Edge Cases & Constraints
* Not all CFGs can be parsed by LL or LR parsers without transformation.
* Left-recursive grammars break top-down (LL) parsers — must be eliminated.
* LR parsers are more powerful than LL but more complex to implement.

## 💻 Logical Code Snippet (Python)
```python
# Top-down: expand from start symbol downward (predict & match)
def top_down_parse(grammar, start, tokens):
    stack = [start]               # begin with start symbol
    index = 0
    while stack:
        top = stack.pop()
        if is_terminal(top):
            assert top == tokens[index]   # match terminal
            index += 1
        else:
            production = predict(top, tokens[index])  # choose rule via lookahead
            stack.extend(reversed(production))        # push RHS onto stack

# Bottom-up: shift tokens, then reduce when a rule's RHS is on top of stack
def bottom_up_parse(tokens):
    stack = []
    for token in tokens:
        stack.append(token)        # SHIFT
        while can_reduce(stack):
            stack = reduce(stack)  # REDUCE: replace RHS with LHS nonterminal
```

## ❓ Active Recall
* [ ] What is the difference between a recognizer and a parser?
* [ ] Describe the direction of tree construction in top-down vs. bottom-up parsing.
* [ ] What does "LL" stand for? What does "LR" stand for?
* [ ] Why do left-recursive grammars cause problems for top-down parsers?
* [ ] Which parsing approach is generally more powerful — LL or LR?

---

---

# 📄 6. Grammar Analysis: First & Follow Sets

**Tags:** #first-set #follow-set #grammar-analysis #LL-parsing #nullable
**Links:** [[CFG]], [[LL Parsing]], [[Parsing Table Construction]]

---

## 🎯 The "Elevator Pitch"
> FIRST and FOLLOW sets are computed from the grammar to answer two key questions: "What token can a nonterminal start with?" (FIRST) and "What token can appear right after a nonterminal?" (FOLLOW). These sets are essential for building LL parsing tables.

## 🧠 Core Mechanics

* **Nullable (⇒ ε):**
  * A nonterminal `A` is **nullable** if it can derive the empty string (`ε`).
  * Computed iteratively: mark `A` nullable if any of its productions is all-nullable.

* **FIRST(A):**
  * The set of **terminals** that can appear as the **first symbol** of any string derived from `A`.
  * If `A` is nullable, add `ε` to FIRST(A).
  * Algorithm: iterate until no new terminals are added.

* **FOLLOW(A):**
  * The set of **terminals** that can appear **immediately after** `A` in any sentential form.
  * Always includes `$` (end-of-input) for the start symbol.
  * If `A → αBβ`: add FIRST(β) \ {ε} to FOLLOW(B); if β is nullable, add FOLLOW(A) too.

* **Example Grammar G0:**
  ```
  S → ABc
  A → a | ε
  B → b | ε
  ```
  * FIRST(A) = {a, ε}, FIRST(B) = {b, ε}, FIRST(S) = {a, b, c}
  * FOLLOW(A) = {b, c}, FOLLOW(B) = {c}

## ⚠️ Edge Cases & Constraints
* If a nonterminal is **unreachable** from the start symbol, its FIRST/FOLLOW sets don't matter.
* If a nonterminal is **unproductive** (derives no terminal string), the grammar is flawed.
* FIRST/FOLLOW computation requires careful handling of **nullable chains** (e.g., `A → BCD` where B, C, D may all be nullable).

## 💻 Logical Code Snippet (Python)
```python
# Iterative algorithm to compute nullable, FIRST, and FOLLOW sets
def compute_nullable(grammar):
    nullable = set()
    changed = True
    while changed:
        changed = False
        for lhs, productions in grammar.items():
            for prod in productions:
                if all(sym in nullable or sym == 'ε' for sym in prod):
                    if lhs not in nullable:
                        nullable.add(lhs)
                        changed = True
    return nullable

def compute_first(grammar, nullable):
    first = {nt: set() for nt in grammar}
    changed = True
    while changed:
        changed = False
        for lhs, productions in grammar.items():
            for prod in productions:
                for sym in prod:
                    if is_terminal(sym):
                        added = first[lhs].add(sym)  # add terminal to FIRST
                        break
                    else:
                        before = len(first[lhs])
                        first[lhs] |= first[sym] - {'ε'}  # add FIRST of nonterminal
                        if sym not in nullable:
                            break  # stop if sym can't derive ε
                        if len(first[lhs]) != before:
                            changed = True
    return first

def compute_follow(grammar, first, nullable, start):
    follow = {nt: set() for nt in grammar}
    follow[start].add('$')
    changed = True
    while changed:
        changed = False
        for lhs, productions in grammar.items():
            for prod in productions:
                for i, sym in enumerate(prod):
                    if sym in grammar:  # sym is a nonterminal
                        beta = prod[i+1:]
                        before = len(follow[sym])
                        # Add FIRST(beta) \ {ε}
                        for b in beta:
                            if is_terminal(b):
                                follow[sym].add(b); break
                            else:
                                follow[sym] |= first[b] - {'ε'}
                                if b not in nullable: break
                        else:
                            follow[sym] |= follow[lhs]  # beta is nullable → add FOLLOW(lhs)
                        if len(follow[sym]) != before:
                            changed = True
    return follow
```

## ❓ Active Recall
* [ ] What does FIRST(A) represent? How is it computed?
* [ ] What does FOLLOW(A) represent? Why is it needed?
* [ ] What does it mean for a nonterminal to be "nullable"?
* [ ] In `S → ABc` with `A → a | ε` and `B → b | ε`, what is FIRST(S)?
* [ ] When computing FOLLOW(B) in `A → αBβ`, when do you add FOLLOW(A) to FOLLOW(B)?
