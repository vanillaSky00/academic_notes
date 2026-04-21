# 📄 Chapter 3: Scanning (Lexical Analysis)

**Tags:** #compilers #scanning #lexical-analysis #tokens #finite-automata
**Links:** [[Chapter 4 - Grammars and Parsing]], [[Regular Expressions]], [[Finite Automata]], [[Thompson Construction]], [[Subset Construction]], [[DFA Minimization]]

---

## 🎯 The "Elevator Pitch"
> The **scanner** (lexical analyzer) is the compiler's first pass: it reads raw source code character-by-character and groups characters into meaningful chunks called **tokens** (like keywords, identifiers, numbers). Think of it as turning a wall of text into a stream of labeled words before grammar rules are applied.

---
---

# 📄 1. Tokens, Patterns & Lexemes — The Holy Trinity

**Tags:** #tokens #lexemes #patterns #scanner
**Links:** [[Regular Expressions]], [[Symbol Table]]

---

## 🎯 The "Elevator Pitch"
> Every chunk of source code has three faces: the **category** it belongs to (token), the **rule** describing what it looks like (pattern), and the **actual characters** matched from the source (lexeme). Like a word "running" belongs to category VERB, matches the pattern /[a-z]+ing/, and its lexeme is literally "running".

## 🧠 Core Mechanics

* **Token:** The abstract *type* of a lexical unit — defined as an enumerated type in the compiler.
  ```c
  typedef enum { IF, THEN, ELSE, PLUS, MINUS, NUM, ID, ... } TokenType;
  ```
  Two subtypes:
  * **Fixed tokens** (e.g., keywords like `if`, operators like `+`): the lexeme IS the token — no extra attribute needed.
  * **Value-carrying tokens** (e.g., `ID`, `NUM`): the lexeme varies, so the token carries an **attribute** — typically a pointer into the symbol table.

* **Lexeme:** The actual character string matched from the input. E.g., for the token `ID`, the lexeme might be `"myVariable"`.

* **Pattern:** The rule (regular expression) describing which strings qualify as a given token. E.g., the pattern for `ID` is `letter(letter|digit)*`.

* **Classic Example — scanning `E = M * C ** 2`:**
  ```
  <ID,   → symbol table entry for E >
  <assign_op>
  <ID,   → symbol table entry for M >
  <mult_op>
  <ID,   → symbol table entry for C >
  <expo_op>
  <NUM,  integer value 2>
  ```
  Notice: identifiers carry a pointer; operators carry no value; numeric literals carry their integer value.

* **The `getToken()` interface:**
  * The scanner does NOT tokenize the entire file at once.
  * It operates **on-demand** under parser control, returning one token at a time:
    ```c
    TokenType getToken(void);
    ```
  * This pull model keeps memory usage constant regardless of file size.

* **Trick — keywords vs. identifiers:**
  * Install all reserved keywords into the symbol table *first*.
  * When the scanner matches `letter(letter|digit)*`, look it up in the symbol table.
  * If found → return the keyword token. If not → return `ID`.
  * This elegantly avoids writing separate rules for every keyword.

## ⚠️ Edge Cases & Constraints
* **Longest match rule ("maximal munch"):** Always consume the longest possible string. E.g., `>=` should be one `GEQ` token, not `>` then `=`.
* **Precedence rule:** If two patterns match the same longest lexeme (e.g., `do` matches both identifier and keyword), the one listed first in the scanner spec wins.
* **Tie resolution for keywords:** The keyword-preloaded symbol table trick handles this automatically.

## 💻 Logical Code Snippet (Python)

```python
from enum import Enum, auto

class TokenType(Enum):
    IF = auto(); ELSE = auto(); PLUS = auto()
    MINUS = auto(); NUM = auto(); ID = auto()

# Symbol table pre-loaded with keywords
symbol_table = {"if": TokenType.IF, "else": TokenType.ELSE}

def get_token(source: str, pos: int) -> tuple:
    """Returns (TokenType, lexeme, new_pos). Pull-model: called by parser."""
    # Skip whitespace
    while pos < len(source) and source[pos].isspace():
        pos += 1
    if pos >= len(source):
        return None, None, pos

    ch = source[pos]

    # Match identifier or keyword: letter(letter|digit)*
    if ch.isalpha():
        start = pos
        while pos < len(source) and source[pos].isalnum():
            pos += 1
        lexeme = source[start:pos]
        # Keyword trick: look up in symbol table first
        tok = symbol_table.get(lexeme, TokenType.ID)
        return tok, lexeme, pos

    # Match number: digit+
    if ch.isdigit():
        start = pos
        while pos < len(source) and source[pos].isdigit():
            pos += 1
        return TokenType.NUM, int(source[start:pos]), pos

    if ch == '+': return TokenType.PLUS, '+', pos + 1
    if ch == '-': return TokenType.MINUS, '-', pos + 1
```

## ❓ Active Recall
* [ ] What is the difference between a token, a lexeme, and a pattern? Give one example of each.
* [ ] Why does the scanner return tokens on-demand instead of all at once?
* [ ] How does pre-loading keywords into the symbol table simplify scanner design?
* [ ] What is the "maximal munch" rule? Why is it necessary?
* [ ] What is a "value-carrying" token and why does it carry a symbol table pointer?

---
---

# 📄 2. Input Buffering

**Tags:** #input-buffering #lookahead #scanner-implementation
**Links:** [[Tokens]], [[DFA Implementation]]

---

## 🎯 The "Elevator Pitch"
> To figure out where one token ends and the next begins, the scanner often needs to "peek ahead" beyond the current character. Input buffering manages this lookahead efficiently, using two pointers on a character buffer.

## 🧠 Core Mechanics

* **The Problem:** Many tokens require lookahead to decide when they end.
  * Example: scanning `a[index] = 4 + 2`
  * After seeing `a`, we must check the *next* character to know if it's just `a` (ID) or the start of `a[...]` — but the `[` belongs to the *next* token (subscript operator).

* **Two-Pointer Technique:**
  * **Begin pointer:** Marks the start of the current token being scanned.
  * **Lookahead pointer:** Advances forward to find the end of the token.
  * When a complete token is recognized: extract `source[begin:lookahead]` as the lexeme, then advance `begin` to `lookahead`.

  ```
  a [ i n d e x ] = 4 + 2
  ^begin
    ^lookahead  (has gone past 'a', sees '[', backs up — 'a' is the token)
  ```

* **Why Buffering?** Reading character-by-character from disk is expensive. Buffering reads a block at a time. The two-pointer system operates on the buffer without re-reading from disk.

## ⚠️ Edge Cases & Constraints
* Some tokens require **retraction**: the lookahead pointer may advance past the token boundary and must back up by one character (e.g., matching `<` vs `<=` — after seeing `<`, you must peek at the next char; if it's not `=`, retract).
* The **`/` operator in some languages** (like Fortran) requires even more complex restart-position tracking (see §7 on the `/` slash trick).

## 💻 Logical Code Snippet (Python)
```python
class InputBuffer:
    def __init__(self, source: str):
        self.source = source
        self.begin = 0       # start of current token
        self.lookahead = 0   # current read position

    def peek(self) -> str:
        """Look at current char without consuming."""
        return self.source[self.lookahead] if self.lookahead < len(self.source) else '\0'

    def advance(self):
        """Move lookahead forward."""
        self.lookahead += 1

    def retract(self):
        """Back up lookahead by one (used when we went too far)."""
        self.lookahead -= 1

    def commit_token(self) -> str:
        """Extract current token lexeme and advance begin pointer."""
        lexeme = self.source[self.begin:self.lookahead]
        self.begin = self.lookahead
        return lexeme
```

## ❓ Active Recall
* [ ] What are the roles of the begin pointer and the lookahead pointer?
* [ ] When does the scanner need to "retract" the lookahead pointer?
* [ ] Why is buffering more efficient than reading character-by-character from disk?

---
---

# 📄 3. Regular Expressions & Finite Automata — The Theoretical Foundation

**Tags:** #regular-expressions #finite-automata #DFA #NFA #formal-languages
**Links:** [[Thompson Construction]], [[Subset Construction]], [[Scanner]]

---

## 🎯 The "Elevator Pitch"
> Regular expressions describe *patterns* in a human-readable way. Finite automata are *machines* that recognize those patterns. Every regular expression can be mechanically converted into an automaton, which is what your scanner actually runs.

## 🧠 Core Mechanics

### Strings & Languages
| Term | Definition | Example (Σ = {a, b}) |
|------|-----------|----------------------|
| **Alphabet (Σ)** | Finite set of symbols | {a, b} |
| **String** | Finite sequence over Σ | `aba`, `ε` (empty) |
| **Length** \|w\| | Number of symbols | \|aba\| = 3, \|ε\| = 0 |
| **Prefix** | Leading substring | `ab` is prefix of `aba` |
| **Suffix** | Trailing substring | `ba` is suffix of `aba` |
| **Substring** | Delete a prefix AND a suffix | `b` is substring of `aba` |
| **Language** | A (possibly infinite) set of strings | {a, ab, abb, abbb, ...} |

### Regular Expression Rules
| Operator | Meaning | Example |
|----------|---------|---------|
| `ε` | Empty string | matches "" only |
| `a` | Literal symbol | matches "a" only |
| `r\|s` | Union (alternation) | `a\|b` → {a, b} |
| `rs` | Concatenation | `ab` → {ab} |
| `r*` | Kleene star (0 or more) | `a*` → {ε, a, aa, aaa, …} |
| `r+` | One or more | `a+` = `aa*` |
| `r?` | Optional (0 or 1) | `a?` = `a\|ε` |
| `[abc]` | Character class | = `a\|b\|c` |
| `.` | Any character | Σ \ {newline} |

### The Equivalence Theorem
> **RE ≡ NFA ≡ DFA ≡ Regular Language ≡ Regular Grammar**
> All five formalisms describe *exactly* the same class of languages (the **regular languages**). This is one of the foundational results of formal language theory.

### DFA (Deterministic Finite Automaton)
* **Formal definition:** 5-tuple (Q, Σ, δ, q₀, F)
  * Q = finite set of states
  * Σ = input alphabet
  * δ: Q × Σ → Q (total, deterministic transition function — exactly one next state)
  * q₀ ∈ Q = start state
  * F ⊆ Q = accepting states
* **Key property:** For every state and every input symbol, there is **exactly one** next state. No ε-transitions. Easy to simulate in O(n) time.

### NFA (Nondeterministic Finite Automaton)
* Same 5-tuple, but δ: Q × (Σ ∪ {ε}) → **2^Q** (can transition to a *set* of states, including on ε — "free moves")
* **Key property:** Can be in multiple states simultaneously. More compact to construct (Thompson's algorithm), but needs conversion to DFA for efficient execution.
* **ε-closure(S):** The set of all states reachable from any state in S using only ε-transitions (zero or more). This is the fundamental operation in NFA-to-DFA conversion.

## ⚠️ Edge Cases & Constraints
* **DFA vs NFA power:** Identical — both recognize exactly the regular languages. DFAs are faster to simulate; NFAs are smaller and easier to construct.
* **Combining DFAs for multiple tokens is complex** — easier to build separate NFAs and merge them (then convert the combined NFA to a DFA).
* **Regular expressions CANNOT describe nested/recursive structures** (e.g., balanced parentheses). You need CFGs for that.

## 💻 Logical Code Snippet (Python)
```python
# DFA simulation: O(n) time, unambiguous
def run_dfa(dfa: dict, start: str, accepting: set, input_str: str) -> bool:
    """
    dfa: { state: { symbol: next_state } }
    Returns True if DFA accepts input_str.
    """
    state = start
    for ch in input_str:
        if ch not in dfa.get(state, {}):
            return False       # no transition → reject
        state = dfa[state][ch]
    return state in accepting

# Example: DFA for (a|b)*abb
dfa = {
    'q0': {'a': 'q1', 'b': 'q0'},
    'q1': {'a': 'q1', 'b': 'q2'},
    'q2': {'a': 'q1', 'b': 'q3'},
    'q3': {'a': 'q1', 'b': 'q0'},
}
assert run_dfa(dfa, 'q0', {'q3'}, 'aabb')   # True
assert not run_dfa(dfa, 'q0', {'q3'}, 'ab') # False
```

## ❓ Active Recall
* [ ] What are the five components of a DFA?
* [ ] What makes an NFA "nondeterministic"? What is an ε-transition?
* [ ] What is ε-closure(S)? How is it used?
* [ ] Why is `r* = r | rr | rrr | ...` potentially infinite, yet still "regular"?
* [ ] What is the Equivalence Theorem, and why does it matter for compiler design?

---
---

# 📄 4. Thompson's Construction — RE → NFA

**Tags:** #thompson-construction #NFA #regular-expressions #algorithm
**Links:** [[Regular Expressions]], [[Subset Construction]], [[NFA]]

---

## 🎯 The "Elevator Pitch"
> Thompson's Construction is a recursive algorithm that takes any regular expression and mechanically builds an NFA for it by assembling small NFA "lego bricks" — one brick for each operator (union, concatenation, Kleene star). It works by splitting an expression into its constituent subexpressions, constructing NFAs for each, then combining them using a fixed set of rules.

## 🧠 Core Mechanics

**Invariant:** Every NFA produced by Thompson's Construction has:
* Exactly **one start state** (no incoming transitions to it)
* Exactly **one accepting state** (no outgoing transitions from it)
* At most **2 states per symbol/operator** → an *n*-character regex produces at most **2n states**

### The 5 Construction Rules

**Rule 1 — Empty string (ε):**
```
→ [q_start] --ε--> ((q_accept))
```

**Rule 2 — Single symbol `a`:**
```
→ [q_start] --a--> ((q_accept))
```

**Rule 3 — Union `r|s` (N(r) OR N(s)):**
```
         ε    N(r)    ε
→ [new_s] ──→ [...]──→ ((new_f))
         ε    N(s)    ε
```
New start has ε-transitions into both sub-NFAs; both sub-NFAs' accept states have ε-transitions to a new single accept state.

**Rule 4 — Concatenation `rs` (N(r) THEN N(s)):**
```
→ N(r) --ε--> N(s) → (accept of N(s))
```
The accept state of N(r) merges (via ε) into the start state of N(s).

**Rule 5 — Kleene Star `r*` (N(r) zero or more times):**
```
         ε              ε
→ [new_s] → N(r) → ((new_f))
         ε    ↑←back-ε─┘
```
New start → N(r) start (ε); N(r) accept → N(r) start (ε, the loop); N(r) accept → new accept (ε); new start → new accept (ε, for zero repetitions).

### Time & Space Complexity
* The total number of states is at most 2m where m is the length of the regex, and each state has at most 2 outgoing transitions, so the total number of transitions is at most 4m.
* An NFA of m states matching a string of length n takes O(mn) time, so Thompson's NFA can do pattern matching in linear time for fixed-size alphabets.

## ⚠️ Edge Cases & Constraints
* The algorithm processes the regex in **postfix order** (convert infix → postfix first using Shunting-Yard algorithm to handle precedence correctly).
* A stack is used to store intermediate NFAs during construction — each operator pops operand NFAs off the stack and pushes the result.
* The resulting NFA has ε-transitions everywhere — it needs the Subset Construction to become a runnable DFA.

## 💻 Logical Code Snippet (Python)
```python
class State:
    _id = 0
    def __init__(self):
        self.id = State._id; State._id += 1
        self.transitions: dict[str, list['State']] = {}  # char → [states]
    def add(self, symbol: str, target: 'State'):
        self.transitions.setdefault(symbol, []).append(target)

class NFA:
    def __init__(self, start: State, accept: State):
        self.start = start
        self.accept = accept

def nfa_symbol(ch: str) -> NFA:
    """Rule 2: single character."""
    s, a = State(), State()
    s.add(ch, a)
    return NFA(s, a)

def nfa_union(n1: NFA, n2: NFA) -> NFA:
    """Rule 3: r | s."""
    s, a = State(), State()
    s.add('ε', n1.start); s.add('ε', n2.start)
    n1.accept.add('ε', a); n2.accept.add('ε', a)
    return NFA(s, a)

def nfa_concat(n1: NFA, n2: NFA) -> NFA:
    """Rule 4: rs."""
    # Merge n1's accept into n2's start via ε
    n1.accept.add('ε', n2.start)
    return NFA(n1.start, n2.accept)

def nfa_star(n: NFA) -> NFA:
    """Rule 5: r* (Kleene star)."""
    s, a = State(), State()
    s.add('ε', n.start)    # enter the loop
    s.add('ε', a)           # skip entirely (zero repetitions)
    n.accept.add('ε', n.start)  # loop back
    n.accept.add('ε', a)        # exit the loop
    return NFA(s, a)

# To build NFA for (a|b)*abb:
# 1. nfa_symbol('a') | nfa_symbol('b') → union_ab
# 2. nfa_star(union_ab) → star_ab
# 3. nfa_concat(star_ab, nfa_symbol('a')) → ...
# 4. nfa_concat(..., nfa_symbol('b')) → ...
# 5. nfa_concat(..., nfa_symbol('b')) → final NFA
```

## ❓ Active Recall
* [ ] What invariant does every Thompson-constructed NFA satisfy (start/accept states)?
* [ ] Draw the NFA for `a|b` using Thompson's rules. How many states does it have?
* [ ] Draw the NFA for `a*`. Which ε-transition provides "zero repetitions"?
* [ ] In what order should the regex operators be processed, and what data structure is used?
* [ ] What is the maximum number of states for a Thompson NFA built from a regex of length *m*?

---
---

# 📄 5. Subset Construction — NFA → DFA

**Tags:** #subset-construction #powerset-construction #NFA-to-DFA #epsilon-closure
**Links:** [[Thompson Construction]], [[DFA Minimization]], [[LL Parsing]]

---

## 🎯 The "Elevator Pitch"
> An NFA can be in multiple states at once (non-determinism). The Subset Construction resolves this by making each DFA state *represent a set of NFA states* — "where could the NFA possibly be right now?" It's like tracking all possible paths through a maze simultaneously.

## 🧠 Core Mechanics

### Key Operation: ε-closure
* **ε-closure(S):** All NFA states reachable from any state in set S by following ε-transitions *only* (zero or more).
* **move(S, a):** The set of NFA states reachable from any state in S by consuming input character `a` (not including ε-transitions).
* **Combined step:** To transition on `a` from a DFA state S: `ε-closure(move(S, a))`

### The Algorithm (Lazy/Reachable-only Powerset Construction)

```
1. Start: DFA_start = ε-closure({NFA_start})
2. Worklist = [DFA_start]; DFA_states = {DFA_start}
3. While worklist is not empty:
     T = worklist.pop()
     For each input symbol a in Σ:
         U = ε-closure(move(T, a))
         Record DFA transition: T --a--> U
         If U not in DFA_states:
             DFA_states.add(U)
             worklist.append(U)
4. DFA accepting states: any DFA state containing at least one NFA accepting state
```

### Why Not Full Powerset?
* An NFA with *n* states has 2ⁿ possible subsets. But most are **unreachable** from the start.
* The lazy construction only creates states actually reachable — often far fewer than 2ⁿ in practice.
* In pathological cases, the minimal DFA can be exponentially smaller than the DFA produced by the powerset construction.

### Accepting States in the Combined Lexer DFA
When building a DFA for *multiple* token patterns:
1. Build a separate NFA for each token type, tagged with its category.
2. Combine into one NFA with a new start state and ε-transitions to each sub-NFA's start.
3. Run subset construction. A DFA state is accepting for token type T if it contains any NFA accepting state tagged T.
4. If a DFA state accepts multiple token types (precedence conflict) → pick the **highest-priority** one (usually first-listed).

## ⚠️ Edge Cases & Constraints
* The resulting DFA may have up to 2ⁿ states — exponentially larger than the NFA — making the construction impractical for very large NFAs.
* A DFA state that contains NFA states for *multiple* token types → use **precedence** (first-listed token wins).
* The DFA produced is correct but **not necessarily minimal** — use DFA minimization next.

## 💻 Logical Code Snippet (Python)
```python
from collections import deque

def epsilon_closure(states: frozenset, nfa_transitions: dict) -> frozenset:
    """All states reachable via ε-transitions from 'states'."""
    closure = set(states)
    stack = list(states)
    while stack:
        s = stack.pop()
        for t in nfa_transitions.get((s, 'ε'), []):
            if t not in closure:
                closure.add(t)
                stack.append(t)
    return frozenset(closure)

def move(states: frozenset, symbol: str, nfa_transitions: dict) -> frozenset:
    """NFA states reachable from 'states' on 'symbol' (no ε)."""
    result = set()
    for s in states:
        result |= set(nfa_transitions.get((s, symbol), []))
    return frozenset(result)

def subset_construction(nfa_start, nfa_accept, nfa_transitions, alphabet):
    """Convert NFA to DFA using subset (powerset) construction."""
    start = epsilon_closure(frozenset([nfa_start]), nfa_transitions)
    dfa_transitions = {}
    dfa_states = {start}
    worklist = deque([start])

    while worklist:
        T = worklist.popleft()
        for a in alphabet:
            U = epsilon_closure(move(T, a, nfa_transitions), nfa_transitions)
            if not U:
                continue  # dead state — skip or add sink
            dfa_transitions[(T, a)] = U
            if U not in dfa_states:
                dfa_states.add(U)
                worklist.append(U)

    # DFA accepting states: any subset containing an NFA accept state
    dfa_accepting = {S for S in dfa_states if nfa_accept in S}
    return start, dfa_states, dfa_transitions, dfa_accepting
```

## ❓ Active Recall
* [ ] Define ε-closure(S). How is it computed?
* [ ] What is `move(S, a)`? How does it combine with ε-closure?
* [ ] Describe the subset construction algorithm step-by-step.
* [ ] How are DFA accepting states determined during subset construction?
* [ ] Why doesn't the construction always produce 2ⁿ DFA states even though the powerset has 2ⁿ elements?
* [ ] What happens when a DFA state contains NFA accepting states for two different tokens?

---
---

# 📄 6. DFA Minimization — Hopcroft's Algorithm

**Tags:** #DFA-minimization #Hopcroft #table-filling #distinguishable-states
**Links:** [[Subset Construction]], [[Scanner Optimization]]

---

## 🎯 The "Elevator Pitch"
> After subset construction, the DFA may have redundant states that behave identically. DFA minimization finds and merges them, producing the **unique smallest DFA** for a given regular language — fewer states means a faster, leaner scanner.

## 🧠 Core Mechanics

### Distinguishability
* Two states p and q are **distinguishable** if there exists some string w such that starting from p leads to an accepting state, but starting from q does not (or vice versa).
* Two states are **indistinguishable** (equivalent) if no such string exists — they can be safely merged.
* Two states q₁, q₂ are distinguishable if there exists a string w such that exactly one of δ̂(q₁, w) and δ̂(q₂, w) is an accepting state.

### The Table-Filling Algorithm (O(n²))

**Intuition:** Start from the obvious: any accepting state is distinguishable from any non-accepting state (distinguished by ε). Then propagate backwards.

```
Step 0: Remove unreachable states.
Step 1: Mark all (accepting, non-accepting) pairs as distinguishable.
Step 2: Repeat until no new pairs are marked:
         For each unmarked pair (p, q) and each symbol a:
           If (δ(p,a), δ(q,a)) is already marked → mark (p,q)
Step 3: All unmarked pairs are equivalent → merge them.
Step 4: Remove dead states and unreachable states.
         Make transitions to dead states undefined (or add one sink state).
```

### Hopcroft's Algorithm (O(n log n)) — The Efficient Version
* Based on partition refinement: initially partition states into {F, Q\F} (accepting vs. non-accepting). Repeatedly split partitions when states within a group behave differently on some input symbol.
* At termination, each partition class = one state in the minimal DFA.
* **Choosing the smaller split** when adding to the worklist is the key insight that gives the O(n log n) bound.

### Steps for the Lecture's Algorithm:
1. **Partition:** Start with two groups: {accepting states} and {non-accepting states}.
2. **Refine:** For each group G and symbol a: if some states in G go to group X on a and others don't → split G.
3. **Pick representatives:** One state per final group.
4. **Clean up:** Delete dead states (no path to accepting) and unreachable states.

## ⚠️ Edge Cases & Constraints
* The minimal DFA is unique up to state renaming — every language has exactly one minimal DFA.
* Dead states (states from which no accepting state is reachable) can be deleted, making the transition function partial. This is acceptable in practice — "no transition" means reject.
* Minimization applies to the DFA *after* subset construction. You cannot minimize an NFA directly (NFA minimization is PSPACE-complete).

## 💻 Logical Code Snippet (Python)
```python
def dfa_minimize(states, alphabet, transitions, accepting, start):
    """
    Table-filling / partition-refinement DFA minimization.
    transitions: dict { (state, symbol): next_state }
    Returns: mapping of old_state -> representative_state
    """
    non_accepting = states - accepting

    # Step 1: Initial partition {F, Q\F}
    partition = [accepting, non_accepting] if non_accepting else [accepting]
    partition = [p for p in partition if p]  # remove empty sets

    changed = True
    while changed:
        changed = False
        new_partition = []
        for group in partition:
            # Try to split 'group' based on transitions
            splits = {}
            for state in group:
                # Signature: which partition-group does each symbol lead to?
                sig = tuple(
                    next((i for i, g in enumerate(partition)
                          if transitions.get((state, a)) in g), -1)
                    for a in alphabet
                )
                splits.setdefault(sig, set()).add(state)

            if len(splits) > 1:
                changed = True  # group was split
            new_partition.extend(splits.values())
        partition = new_partition

    # Step 2: Build representative mapping (pick any element per group)
    rep = {}
    for group in partition:
        representative = next(iter(group))
        for state in group:
            rep[state] = representative

    return rep  # map each state → its representative (merged state)
```

## ❓ Active Recall
* [ ] Define "distinguishable states." What string is always sufficient to distinguish an accepting state from a non-accepting state?
* [ ] Describe the table-filling algorithm for DFA minimization in 4 steps.
* [ ] What is the initial partition in Hopcroft's algorithm, and why?
* [ ] What does it mean to "refine" a partition?
* [ ] Is there always a unique minimal DFA? Why or why not?
* [ ] Why can't we directly minimize an NFA the same way?

---
---

# 📄 7. The Full Lexical Analyzer Pipeline — Putting It All Together

**Tags:** #lexical-analyzer #pipeline #scanner-construction #Lex #longest-match
**Links:** [[Thompson Construction]], [[Subset Construction]], [[DFA Minimization]], [[Lex Tool]]

---

## 🎯 The "Elevator Pitch"
> Building a real scanner means combining all the theory: write regex patterns for each token → build NFAs → merge them → convert to DFA → minimize → simulate the DFA on source input with longest-match and precedence rules.

## 🧠 Core Mechanics

### The 5-Step Construction Pipeline

```
Step 1: Write a RE for each token type
            ID     →  letter(letter|digit)*
            NUM    →  digit+
            RELOP  →  < | > | <= | >= | == | !=

Step 2: Build a Thompson NFA for each RE (separate NFAs)

Step 3: Combine NFAs into one mega-NFA
            Add a new start state q₀
            Add ε-transitions from q₀ to each sub-NFA's start state
            Tag each sub-NFA's accept state with its token type

Step 4: Convert the mega-NFA to a DFA (Subset Construction)
            Each DFA state that includes a tagged NFA accept state → accepting
            If multiple tags → pick highest priority token

Step 5: Minimize the DFA (Hopcroft's Algorithm)
```

### Lexing Strategy: Longest Match + Precedence
* **Longest match (maximal munch):** Keep advancing as long as the DFA can transition. Accept only when the next character would cause a dead/failure transition.
* **Precedence:** If the current DFA state accepts multiple token types → take the one listed first in the scanner spec.
* **Retraction:** Sometimes you go past the real end of the token. Keep track of the last accepting state and its position; when you hit a dead state, retract to that position.

### The `/` Slash Trick (Fortran Problem)
* Problem: In Fortran, `DO 99 I = 1, 100` is a DO-loop, but `DO 99 I = 1.100` is an assignment to variable `DO99I`.
* You don't know which it is until you see the `,` vs `.`.
* Solution: Mark the **restart position** in the pattern with a `/`:
  ```
  do / ({letter}|{digit})* = ({letter}|{digit})* ,
  ```
  The `/` means: if this token is matched, scanning of the *next* token resumes at the `/` position (not after the full match).
* Implementation: create a special DFA state for the `/` position that records a "tentative restart point."

### Lex / Flex — The Practical Tool
* Lex (or Flex) automates the entire pipeline.
* **Input file format:**
  ```
  {definitions}     ← named regex macros, %{ ... %} C declarations
  %%
  {rules}           ← pattern  { action code }
  %%
  {auxiliary code}  ← helper C functions
  ```
* **Key Lex variables:**
  * `yylex()` — invoke the scanner (called by parser)
  * `yytext` — pointer to matched lexeme string
  * `yyleng` — length of matched string
  * `yylineno` — current line number

* **Example Lex rules:**
  ```lex
  letter(letter|digit)*  { yylval = install(); return ID;   }
  digit+                 { yylval = install(); return NUM;  }
  "<"                    { yylval = LT; return RELOP;       }
  ```

* **Compile workflow:**
  ```bash
  lex  myfile.l   # → generates lex.yy.c
  cc   lex.yy.c   # → a.out (executable scanner)
  ```

## ⚠️ Edge Cases & Constraints
* **Keyword vs. identifier conflict:** Always handle by installing keywords in the symbol table first, or by listing keyword patterns *before* the ID pattern (precedence rule).
* **Ambiguous longest match:** Two patterns can match the same longest string → precedence (first rule wins).
* **Scanner errors:** When no pattern matches and no transition is possible → scanner error (illegal character). Must handle gracefully (skip or report).
* **Combining many DFAs** directly is complex — the combined NFA → DFA route is much cleaner.

## 💻 Logical Code Snippet (Python)
```python
class Lexer:
    """Simulates a DFA-based lexer with longest-match and precedence."""

    def __init__(self, dfa, start, accepting_map):
        """
        dfa: { state: { char: next_state } }
        accepting_map: { state: TokenType }  (highest-priority token per state)
        """
        self.dfa = dfa
        self.start = start
        self.accepting_map = accepting_map

    def tokenize(self, source: str):
        tokens = []
        pos = 0
        while pos < len(source):
            # Skip whitespace
            while pos < len(source) and source[pos].isspace():
                pos += 1
            if pos >= len(source): break

            # Longest-match scan
            state = self.start
            last_accept_pos = -1
            last_accept_token = None
            i = pos

            while i < len(source):
                ch = source[i]
                next_state = self.dfa.get(state, {}).get(ch)
                if next_state is None:
                    break  # dead transition — stop
                state = next_state
                i += 1
                if state in self.accepting_map:
                    # Record this as a candidate (may still find longer match)
                    last_accept_pos = i
                    last_accept_token = self.accepting_map[state]

            if last_accept_token is None:
                raise SyntaxError(f"Illegal character '{source[pos]}' at position {pos}")

            # Retract to last accepting position
            lexeme = source[pos:last_accept_pos]
            tokens.append((last_accept_token, lexeme))
            pos = last_accept_pos  # resume from here

        return tokens
```

## ❓ Active Recall
* [ ] List the 5 steps to build a lexical analyzer from scratch.
* [ ] Why do we merge individual NFAs rather than individual DFAs?
* [ ] What is the "longest match" rule? What is "retraction"?
* [ ] How does the scanner handle a conflict where both `ID` and a keyword like `if` match?
* [ ] What do `yytext`, `yyleng`, and `yylineno` represent in Lex?
* [ ] What is the `/` slash trick? In which language was this problem first noted?
* [ ] What happens when no DFA transition is possible and no accepting state was seen? (Scanner error)

---

## 🗺️ Chapter 3 — Master Concept Map

```
Source Code (characters)
        │
        ▼
  ┌─────────────┐
  │ Input Buffer│  (begin ptr + lookahead ptr)
  └──────┬──────┘
         │ characters
         ▼
  ┌─────────────────────────────────────────┐
  │          DFA Simulation                 │
  │  (longest match + retraction logic)     │
  └──────────────┬──────────────────────────┘
                 │ token type + lexeme
                 ▼
         ┌───────────────┐
         │  Symbol Table │  (stores identifiers, literals)
         └───────────────┘
                 │
                 ▼
        Stream of Tokens → Parser

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
HOW THE DFA WAS BUILT:

RE patterns  →[Thompson's]→  NFA  →[Subset Construction]→  DFA  →[Hopcroft]→  Min DFA
```
