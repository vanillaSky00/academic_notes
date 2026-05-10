# 📄 Chapter 7 — Syntax-Directed Translation (SDT)

**Tags:** #compiler #SDT #attribute-grammar #semantic-analysis #LALR #intermediate-code #NCKU
**Links:** [[Bottom-Up Parsing II]], [[Symbol Tables]], [[CFG]], [[Parse Tree]], [[Intermediate Code Generation]]

---

## 🎯 The "Elevator Pitch"
> Syntax-Directed Translation answers: *"Once the parser recognizes structure, how do we attach meaning to it?"* Every grammar rule gets **semantic rules** (equations) that compute **attributes** — properties like data types, computed values, or generated code. The parse tree becomes a computation graph: values flow up (synthesized) or down (inherited), and at the end you've translated source code into something meaningful.

---

## 🧠 Part 1 — Semantic Analysis: The Big Picture

### Where Semantics Fits in the Compiler Pipeline

```
Source chars
    │
    ▼ Lexer (scanner)
Token stream
    │
    ▼ Parser (LALR)
Parse tree / AST
    │
    ▼ Semantic Analyzer  ← YOU ARE HERE
Annotated tree + Symbol Table
    │
    ▼ Intermediate Code Generator
IR (Three-address code / TAC)
    │
    ▼ Optimizer → Code Generator → Target Code
```

### Two Categories of Semantic Analysis

| Category | Purpose | Examples |
|---|---|---|
| **Correctness analysis** | Required by language rules to ensure valid execution | Type checking, scope rules, declaration checking |
| **Optimization analysis** | Improves execution efficiency | Constant folding, dead code detection, loop invariants |

### What Gets Checked in Correctness Analysis?

- **Type checking**: Is `x + y` valid if `x` is `int` and `y` is `float`?
- **Declaration checking**: Is every variable declared before use?
- **Scope rules**: Is this name visible in this scope?
- **Flow rules**: Does every execution path return a value?

### 💡 Why is Semantics *Syntax-Directed*?

All modern languages are designed so that **meaning follows structure**. The meaning of `a + b` is determined by the structure of the `+` node and the meanings of its children `a` and `b`. There is no need to look at the whole program to understand a local expression — this is **compositionality**, and it's what makes attribute grammars possible.

---

## 🧠 Part 2 — Attribute Grammars: The Formal Framework

### What is an Attribute Grammar?

> An **attribute grammar** is a CFG augmented with:
> 1. A set of **attributes** — named properties associated with each grammar symbol.
> 2. **Semantic rules** (attribute equations) — one per grammar rule, defining how attributes are computed from each other.

### Formal Definition

For each grammar rule `A → X₁ X₂ … Xₙ`, we write semantic rules of the form:

```
b = f(c₁, c₂, …, cₖ)
```

Where `b` is an attribute of some symbol in the rule and `c₁…cₖ` are attributes of other symbols in the same rule.

### Attributes Live on Parse Tree Nodes

Each node in the parse tree carries attribute *slots*. The semantic rules define how to fill those slots. Once all slots are filled, you've completed the translation.

---

## 🧠 Part 3 — Synthesized vs. Inherited Attributes

This is the **most important distinction** in the chapter. Get this right and everything else follows.

### Synthesized Attributes (Bottom-Up Flow)

> An attribute of node `A` is **synthesized** if its value is computed from the attributes of `A`'s **children** (and possibly `A` itself).

- Information flows **upward** in the parse tree: leaves → root.
- Easy to compute with bottom-up (LR) parsing — a natural fit!
- Example: the **value** of an arithmetic expression is synthesized from its subexpressions.

```
       expr.val = 7
       /         \
  expr.val=3   expr.val=4
      |             |
    num.val=3   num.val=4
```

**Rule**: `expr → expr₁ '+' expr₂` with semantic rule `expr.val = expr₁.val + expr₂.val`

### Inherited Attributes (Top-Down Flow)

> An attribute of node `X` is **inherited** if its value is computed from the attributes of `X`'s **parent** or **siblings** (left siblings, in left-to-right parsing).

- Information flows **downward** in the parse tree: root → leaves.
- Harder to compute with bottom-up parsing — requires special handling.
- Example: the **data type** in a declaration flows down from `type` to all the identifiers in `var-list`.

```
       decl
      /    \
  type     var-list
 (dtype=real)  (dtype=real ← inherited from parent)
               /    \
             id      var-list
           (dtype=real)  (dtype=real)
                          |
                         id
                       (dtype=real)
```

### 💡 Intuition: River vs. Waterfall

- **Synthesized** = water flowing *uphill* — each node aggregates its children's contributions and passes the result to its parent. Think of a river formed from tributaries.
- **Inherited** = water flowing *downhill* — a parent broadcasts information to its children. Think of a waterfall distributing water downward.

---

## 🧠 Part 4 — Dependency Graphs & Evaluation Order

### The Problem

Attributes depend on each other. Before computing attribute `b`, you must have already computed all the attributes it depends on. This creates a **partial order**.

### Dependency Graph

Build a directed graph:
- **Nodes**: each attribute instance on each parse tree node (e.g., `expr₁.val`, `expr₂.val`)
- **Edges**: `a → b` if computing `b` requires the value of `a`

```python
def build_dependency_graph(parse_tree, semantic_rules):
    """
    Build the attribute dependency graph for a parsed input.
    Returns: dict of attribute_instance -> set of attribute_instances it depends on
    """
    graph = {}   # attr_node -> set of attr_nodes it depends on (prerequisites)

    for tree_node in parse_tree.all_nodes():
        rule = tree_node.grammar_rule         # e.g., "expr → expr + expr"
        sem_rules = semantic_rules[rule]      # e.g., [("expr.val", ["expr1.val","expr2.val"])]

        for (lhs_attr, rhs_attrs) in sem_rules:
            target = (tree_node, lhs_attr)
            graph[target] = set()
            for dep in rhs_attrs:
                # Resolve dep to the actual tree node + attribute
                source = resolve_attribute(tree_node, dep)
                graph[target].add(source)

    return graph


def topological_sort(graph):
    """
    Topological sort of the dependency graph = correct evaluation order.
    If a cycle exists, the attribute grammar is circular (invalid).
    """
    visited = set()
    order = []

    def dfs(node):
        if node in visited:
            return
        visited.add(node)
        for dep in graph.get(node, []):
            dfs(dep)
        order.append(node)   # post-order = correct eval order

    for node in graph:
        dfs(node)

    return order   # evaluate attributes in this order
```

### Circular Dependencies = Bad

If the dependency graph has a **cycle**, the attribute grammar is **circular** and cannot be evaluated. Example:
```
A.x = B.y + 1
B.y = A.x - 1   ← cycle! which one do we compute first?
```

Well-designed attribute grammars are always acyclic.

---

## 🧠 Part 5 — Concrete Attribute Grammar Examples

### Example 1: Arithmetic Expression Evaluator (Synthesized Only)

**Grammar and semantic rules:**

```
expr  → expr₁ '+' term    { expr.val = expr₁.val + term.val }
expr  → term              { expr.val = term.val }
term  → term₁ '*' factor  { term.val = term₁.val * factor.val }
term  → factor            { term.val = factor.val }
factor → '(' expr ')'    { factor.val = expr.val }
factor → number           { factor.val = number.val }
number → number₁ digit    { number.val = number₁.val * 10 + digit.val }
number → digit            { number.val = digit.val }
digit → '0'|'1'|…|'9'    { digit.val = integer value of the character }
```

```python
# Synthesized attributes — compute bottom-up
class ExprNode:
    def evaluate(self):
        """Recursively evaluate the expression (synthesized attribute)."""
        if self.rule == "expr → expr + term":
            return self.children[0].evaluate() + self.children[2].evaluate()
        elif self.rule == "term → term * factor":
            return self.children[0].evaluate() * self.children[2].evaluate()
        elif self.rule == "factor → ( expr )":
            return self.children[1].evaluate()
        elif self.rule == "number → digit":
            return self.children[0].digit_value()
        elif self.rule == "number → number digit":
            return self.children[0].evaluate() * 10 + self.children[1].digit_value()
```

### Example 2: Based Number (Mixed Inherited + Synthesized)

Parse `345o` (octal) or `128d` (decimal). The **base** must be inherited from the suffix down to each digit.

```
num   → digits basesuffix    { num.val = digits.val(base=basesuffix.base) }
basesuffix → 'o'             { basesuffix.base = 8 }
basesuffix → 'd'             { basesuffix.base = 10 }
digits → digits₁ digit       { digits.base = digits₁.base [inherited from parent]
                               digit.base  = digits.base
                               digits.val  = digits₁.val * digits.base + digit.val }
digits → digit               { digit.base = digits.base
                               digits.val  = digit.val }
digit  → '0'|…|'7'|…|'9'   { digit.val  = char_to_int(digit.lexval) }
```

💡 **Notice**: `base` flows **down** (inherited), `val` flows **up** (synthesized). This is a **mixed** attribute grammar.

```python
def evaluate_based_number(digits_str, base_char):
    """
    Conceptual: base is inherited (top-down), val is synthesized (bottom-up).
    """
    base = 8 if base_char == 'o' else 10   # inherited attribute

    val = 0
    for ch in digits_str:
        digit_val = int(ch)
        if digit_val >= base:
            raise SemanticError(f"Digit {ch} invalid in base {base}")
        val = val * base + digit_val       # synthesized attribute

    return val
```

### Example 3: Type Declaration (Inherited, C-like)

`float x, y;` — the type `float` must be distributed to both `x` and `y`.

```
Grammar Rule                    Semantic Rule
─────────────────────────────────────────────────────────
decl  → type var-list           var-list.dtype = type.dtype
type  → int                     type.dtype = integer
type  → float                   type.dtype = real
var-list₁ → id, var-list₂      id.dtype       = var-list₁.dtype   [inherited]
                                var-list₂.dtype = var-list₁.dtype   [inherited]
                                addtype(id.entry, id.dtype)          [side effect]
var-list → id                   id.dtype = var-list.dtype            [inherited]
                                addtype(id.entry, id.dtype)          [side effect]
```

```python
# Inherited attribute: dtype flows from decl down to every id
def process_declaration(type_node, var_list_node):
    dtype = type_node.dtype                  # synthesized: 'int' or 'real'
    distribute_type(var_list_node, dtype)    # inherited: push dtype downward

def distribute_type(var_list_node, dtype):
    var_list_node.dtype = dtype              # receive inherited attribute

    if var_list_node.has_comma:
        id_node = var_list_node.first_child
        id_node.dtype = dtype                # inherit
        symbol_table.addtype(id_node.entry, dtype)   # side effect

        rest = var_list_node.second_child
        distribute_type(rest, dtype)         # recurse: propagate down
    else:
        id_node = var_list_node.only_child
        id_node.dtype = dtype
        symbol_table.addtype(id_node.entry, dtype)
```

---

## 🧠 Part 6 — Traversal Algorithms for Attribute Computation

### For Synthesized Attributes: Postorder Traversal

Compute children before parent — naturally matches bottom-up parsing.

```python
def postorder_eval(node):
    """Evaluate synthesized attributes — children first, then parent."""
    for child in node.children:
        postorder_eval(child)                # recurse into children first
    node.syn_attr = compute_synthesized(node)  # then compute this node's attr
```

### For Inherited Attributes: Preorder Traversal

Compute parent before children — naturally matches top-down (LL) parsing.

```python
def preorder_eval(node):
    """Evaluate inherited attributes — parent first, then children."""
    for child in node.children:
        compute_inherited(child, from_parent=node)  # push attrs down to child
        preorder_eval(child)                          # recurse
```

### For Mixed (Inherited + Synthesized): Dependency-Ordered Traversal

Use topological sort of the dependency graph to determine order. In practice, many compilers use multiple passes over the tree.

```python
def mixed_eval(parse_tree, semantic_rules):
    """
    Evaluate all attributes in dependency order.
    General solution using topological sort.
    """
    dep_graph = build_dependency_graph(parse_tree, semantic_rules)
    eval_order = topological_sort(dep_graph)

    attr_values = {}   # (node, attr_name) -> computed value

    for (node, attr) in eval_order:
        rule = node.grammar_rule
        sem_rule = semantic_rules[rule][attr]

        # Gather all dependency values
        deps = {dep: attr_values[dep] for dep in dep_graph[(node, attr)]}
        attr_values[(node, attr)] = sem_rule(**deps)

    return attr_values
```

---

## 🧠 Part 7 — L-Attributed Grammars

### Definition

> An attribute grammar is **L-attributed** if every inherited attribute of symbol `Xⱼ` in a rule `A → X₁ X₂ … Xₙ` depends only on:
> 1. Attributes of **A** (the parent), OR
> 2. Attributes of **left siblings** `X₁, X₂, …, Xⱼ₋₁` (symbols to the LEFT of Xⱼ)

### 💡 Why "L"?

"L" stands for **Left-to-right**: you can evaluate an L-attributed grammar in a single left-to-right pass over the parse tree. As you visit each node, all the information you need (from the parent and left siblings) is already available.

### Classes of Grammars

| Grammar Class | Constraint | Evaluation |
|---|---|---|
| **S-attributed** | All attributes are synthesized | Postorder traversal (single pass, LR parser friendly) |
| **L-attributed** | Inherited attrs depend only on parent + left siblings | Left-to-right single pass |
| **General** | Arbitrary dependencies (no cycle) | Topological sort (may need multiple passes) |

S-attributed ⊂ L-attributed ⊂ General attribute grammars.

### Computing Synthesized Attributes in LR Parsing

LR parsers naturally compute synthesized attributes using a **value stack** that mirrors the parsing stack:

```
Parsing stack:   [ s₀ | s₁ | s₂ | … | sₙ ]
Value stack:     [ v₀ | v₁ | v₂ | … | vₙ ]
```

When we **reduce** by `A → X₁ X₂ X₃`:
1. Pop 3 states from parsing stack.
2. Pop 3 values from value stack (these are `$1`, `$2`, `$3` in Yacc terms).
3. Compute `$$` = semantic rule using `$1`, `$2`, `$3`.
4. Push `$$` onto value stack.

```python
def lr_parse_with_attrs(tokens, action_table, goto_table, semantic_rules):
    """
    LR parser with parallel value stack for synthesized attribute computation.
    """
    state_stack = [0]
    value_stack = [None]          # parallel value stack
    tokens.append('$')
    i = 0

    while True:
        state = state_stack[-1]
        tok, tok_val = tokens[i]
        action = action_table.get((state, tok), 'error')

        if action == 'error':
            raise SyntaxError(f"Unexpected token '{tok}'")

        elif action == 'accept':
            return value_stack[-1]    # final synthesized value

        elif action[0] == 'shift':
            _, next_state = action
            state_stack.append(next_state)
            value_stack.append(tok_val)    # push token's attribute value
            i += 1

        elif action[0] == 'reduce':
            _, lhs, rhs = action
            n = len(rhs)

            # Pop n values — these become $n, $n-1, ..., $1
            popped_values = []
            for _ in range(n):
                state_stack.pop()
                popped_values.insert(0, value_stack.pop())

            # Apply semantic rule: compute synthesized attribute $$
            sem_rule = semantic_rules.get((lhs, tuple(rhs)), lambda vals: vals[0])
            result = sem_rule(popped_values)   # $$ = f($1, $2, ..., $n)

            top = state_stack[-1]
            state_stack.append(goto_table[(top, lhs)])
            value_stack.append(result)         # push $$ onto value stack
```

---

## 🧠 Part 8 — Semantic Actions: What They Actually Do

A **semantic action** is the code executed when a grammar rule is reduced. It can:

1. **Compute attribute values** — `$$ = $1 + $3`
2. **Maintain compiler variables** — update counters, temporary name generators
3. **Generate intermediate code** — emit three-address code (TAC) instructions
4. **Print error diagnostics** — type mismatch, undeclared variable, etc.
5. **Update the symbol table** — insert declarations, resolve references

### Generating Intermediate Code via Semantic Actions

```yacc
/* Generating three-address code for assignments */
assign : ID '=' expr ';'
    {
        char *tmp = new_temp();        // allocate a new temporary
        emit("%s = %s", $1, $3.place); // emit: x = temp
        $$.place = tmp;
    }
    ;

expr : expr '+' expr
    {
        char *tmp = new_temp();
        emit("%s = %s + %s", tmp, $1.place, $3.place);
        $$.place = tmp;
    }
    ;
```

```python
# Python conceptual equivalent: TAC generation as semantic actions
class TACGenerator:
    def __init__(self):
        self.code = []           # list of TAC instructions
        self.temp_count = 0

    def new_temp(self):
        name = f"t{self.temp_count}"
        self.temp_count += 1
        return name

    def emit(self, instruction):
        self.code.append(instruction)

    def gen_add(self, left_place, right_place):
        t = self.new_temp()
        self.emit(f"{t} = {left_place} + {right_place}")
        return t    # this is the synthesized 'place' attribute

    def gen_assign(self, dest, src_place):
        self.emit(f"{dest} = {src_place}")
```

---

## 🧠 Part 9 — Declaration Handling: Grammar Design Matters!

### The Problem with Naive Grammar for Declarations

Consider: `int x, y, z;`

**Avoided grammar** (`namelist → id | id ',' namelist`):

```
D → int namelist ;
namelist → id
namelist → id , namelist
```

**Problem**: When `id` is reduced to `namelist`, the type (`int`) hasn't been seen yet in a bottom-up parser! We can't immediately insert the type into the symbol table.

```
Parse stack evolution (bottom-up):
  int  →  int id  →  int id,  →  int id, id  →  int namelist  →  D
                                              ↑
                                    PROBLEM: type unknown when
                                    'id' is first reduced!
```

### The Hook Technique (Marker Nonterminals)

Insert a **marker nonterminal** `M` immediately after the type keyword. When `M` is reduced (which happens before seeing any ids), we can store the current type in a global/inherited variable.

```
D → type M namelist ';'   { /* type stored during M reduction */ }
M → ε                     { current_type = type.dtype }    ← hook!
namelist → id ',' namelist { addtype(id.entry, current_type) }
namelist → id              { addtype(id.entry, current_type) }
```

```python
# Hook implementation concept
current_type = None   # global compiler variable (shared state)

def reduce_M():
    """Empty production M → ε acts as a hook: type is now known."""
    global current_type
    current_type = type_stack[-1]    # read the type from value stack

def reduce_namelist_id(id_entry):
    """Now we can safely insert into symbol table."""
    symbol_table.addtype(id_entry, current_type)
```

### Cloned Productions: Another Solution

Duplicate grammar rules — one clone per type keyword. Avoids the need for inherited attributes entirely.

```
D → int  namelist_int  ';'
D → float namelist_float ';'

namelist_int  → namelist_int  ',' id { addtype($3, integer) }
namelist_int  → id                   { addtype($1, integer) }

namelist_float → namelist_float ',' id { addtype($3, real) }
namelist_float → id                    { addtype($1, real) }
```

**Trade-off**: Eliminates inherited attributes at the cost of grammar bloat — not practical for many types.

### The Right Grammar Design for `float x, y;`

**Preferred grammar** (right-recursive with type propagation):

```
decl → type var-list

type → int    { type.dtype = integer }
type → float  { type.dtype = real }

var-list₁ → id ',' var-list₂    { var-list₂.dtype = var-list₁.dtype   (inherited)
                                   id.dtype = var-list₁.dtype           (inherited)
                                   addtype(id.entry, id.dtype) }
var-list → id                    { id.dtype = var-list.dtype            (inherited)
                                   addtype(id.entry, id.dtype) }
```

### Why the `id_list ':' type` Grammar is Problematic

```
declaration → id_list ':' type
id_list → ID | id_list ',' ID
```

```
   Parse tree:
        declaration
       /      |    \
   id_list    :    type
   /    \           |
id_list  ,ID       REAL
   |
   ID
```

Problem: `id_list` is fully reduced *before* `type` is seen. When reducing `ID → id_list`, we don't yet know the type. This requires backpatching or a linked list of pending entries — complex and error-prone.

**Lesson**: Grammar design directly affects the complexity of semantic analysis. A grammar that's syntactically equivalent can be semantically easy or hard to translate.

---

## 🧠 Part 10 — Bottom-Up Translation of S-Attributed Grammars

For S-attributed grammars (all synthesized), the translation is straightforward during LR parsing:

```python
# Full example: arithmetic expression evaluator via LR + value stack

grammar = {
    # (lhs, rhs) -> semantic function
    ('E', ('E', '+', 'T')): lambda v: v[0] + v[2],
    ('E', ('T',)):          lambda v: v[0],
    ('T', ('T', '*', 'F')): lambda v: v[0] * v[2],
    ('T', ('F',)):          lambda v: v[0],
    ('F', ('(', 'E', ')')): lambda v: v[1],
    ('F', ('num',)):        lambda v: v[0],   # v[0] is number's lexval
}

def parse_and_evaluate(tokens_with_vals, action, goto):
    state_stack = [0]
    val_stack   = [None]
    tokens = list(tokens_with_vals) + [('$', None)]
    i = 0

    while True:
        s = state_stack[-1]
        tok, tok_val = tokens[i]

        act = action[(s, tok)]

        if act == 'accept':
            return val_stack[-1]   # synthesized value of the root

        elif act[0] == 'shift':
            state_stack.append(act[1])
            val_stack.append(tok_val)   # lex value from lexer
            i += 1

        elif act[0] == 'reduce':
            lhs, rhs = act[1], act[2]
            n = len(rhs)
            vals = val_stack[-n:]      # $1 ... $n
            del state_stack[-n:]
            del val_stack[-n:]
            result = grammar[(lhs, rhs)](vals)   # $$ = semantic_rule($1..$n)
            val_stack.append(result)
            state_stack.append(goto[(state_stack[-1], lhs)])

# Example trace for "3 + 4 * 2":
# Tokens: [('num',3), ('+',None), ('num',4), ('*',None), ('num',2)]
# Expected result: 3 + (4*2) = 11
```

---

## ⚠️ Edge Cases & Constraints

- **Circular attribute grammars**: if `A.x` depends on `B.y` and `B.y` depends on `A.x`, the grammar is unusable — there's no valid evaluation order. Well-designed grammars are always acyclic.
- **Inherited attributes in LR parsers**: LR parsers naturally handle synthesized attributes (via value stack), but inherited attributes require workarounds: marker nonterminals (hooks), global variables, or a separate top-down pass.
- **Side effects in semantic rules**: Semantic actions like `addtype()` or `emit()` have global side effects. If evaluation order is wrong, the symbol table or generated code will be incorrect. Always respect dependency order.
- **Grammar refactoring for SDT**: sometimes you must restructure a grammar (clone productions, introduce markers) solely to make attribute computation feasible — even if the language being recognized doesn't change.
- **`$0` in Yacc**: you can access `$0` (the value of the symbol *before* the current rule on the stack) as an unofficial way to peek at a "parent" attribute — but this is fragile and grammar-position dependent. Use with extreme caution.
- **L-attributed but not S-attributed**: an attribute grammar where at least one symbol has an inherited attribute that depends on left siblings cannot be evaluated by simple postorder traversal — it needs the left-to-right L-attributed evaluation strategy.

---

## 💻 Complete Synthesis: SDT for a Mini Typed Language

```python
# ─────────────────────────────────────────────────────────────
# COMPLETE SYNTAX-DIRECTED TRANSLATION SKETCH
# Handles: declarations (int/float x, y;) and arithmetic exprs
# ─────────────────────────────────────────────────────────────

from enum import Enum
from dataclasses import dataclass, field
from typing import Any

class Type(Enum):
    INTEGER = 'integer'
    REAL    = 'real'
    UNKNOWN = 'unknown'


@dataclass
class Symbol:
    name:   str
    dtype:  Type
    scope:  int = 0
    offset: int = 0


class SymbolTable:
    def __init__(self):
        self.scopes = [{}]          # stack of scopes
        self.offset = 0             # memory offset counter

    def open_scope(self):
        self.scopes.append({})

    def close_scope(self):
        self.scopes.pop()

    def addtype(self, name: str, dtype: Type):
        if name in self.scopes[-1]:
            raise SemanticError(f"Redeclaration: '{name}'")
        sym = Symbol(name=name, dtype=dtype, offset=self.offset)
        self.scopes[-1][name] = sym
        self.offset += (4 if dtype == Type.INTEGER else 8)
        return sym

    def lookup(self, name: str) -> Symbol | None:
        for scope in reversed(self.scopes):
            if name in scope:
                return scope[name]
        return None


class TACEmitter:
    def __init__(self):
        self.instructions = []
        self._tmp = 0

    def new_temp(self) -> str:
        t = f"t{self._tmp}"
        self._tmp += 1
        return t

    def emit(self, *parts):
        self.instructions.append(' '.join(str(p) for p in parts))
        return self.instructions[-1]

    def dump(self):
        for i, instr in enumerate(self.instructions):
            print(f"  {i:3d}: {instr}")


# Semantic rules as functions (called during reductions)
# ─────────────────────────────────────────────────────
sym_table = SymbolTable()
tac       = TACEmitter()
_current_type = None     # shared state via hook/marker

def rule_type_int():
    return Type.INTEGER

def rule_type_float():
    return Type.REAL

def rule_hook_M(dtype: Type):
    """Marker: store type before processing id list."""
    global _current_type
    _current_type = dtype

def rule_varlist_id_comma_varlist(id_name: str):
    """var-list → id ',' var-list"""
    sym_table.addtype(id_name, _current_type)

def rule_varlist_id(id_name: str):
    """var-list → id"""
    sym_table.addtype(id_name, _current_type)

def rule_expr_add(place1: str, place2: str) -> str:
    """expr → expr '+' expr  →  $$ = new_temp; emit t = $1 + $3"""
    t = tac.new_temp()
    tac.emit(t, '=', place1, '+', place2)
    return t

def rule_expr_mul(place1: str, place2: str) -> str:
    t = tac.new_temp()
    tac.emit(t, '=', place1, '*', place2)
    return t

def rule_expr_id(id_name: str) -> str:
    sym = sym_table.lookup(id_name)
    if sym is None:
        raise SemanticError(f"Undeclared variable: '{id_name}'")
    return id_name    # the 'place' attribute is just the variable name

def rule_assign(dest: str, src_place: str):
    tac.emit(dest, '=', src_place)


# ─── EXAMPLE EXECUTION ───────────────────────────────
# Source: int x, y; x = x + y;
rule_hook_M(rule_type_int())          # M → ε, type = int
rule_varlist_id('y')                  # var-list → id (y)
rule_varlist_id_comma_varlist('x')    # var-list → id , var-list (x)

# Now translate: x = x + y
p_x = rule_expr_id('x')              # 'x' is a place
p_y = rule_expr_id('y')              # 'y' is a place
p_sum = rule_expr_add(p_x, p_y)      # emit: t0 = x + y
rule_assign('x', p_sum)              # emit: x = t0

tac.dump()
# Output:
#     0: t0 = x + y
#     1: x = t0
```

---

## 🔗 Concept Map

```
Syntax-Directed Translation (Chapter 7)
│
├── Attribute Grammar = CFG + Attributes + Semantic Rules
│     ├── Attributes: properties of grammar symbols
│     │     ├── Synthesized: computed from CHILDREN (flow UP ↑)
│     │     └── Inherited:  computed from PARENT + left siblings (flow DOWN ↓)
│     └── Semantic Rules: equations linking attributes across a production
│
├── Evaluation
│     ├── Dependency graph → topological sort → correct order
│     ├── Synthesized only (S-attributed) → postorder traversal (LR-friendly)
│     ├── L-attributed → single left-to-right pass
│     └── General → may need multiple passes
│
├── LR Parsing + Attribute Computation
│     ├── Value stack mirrors state stack
│     ├── On reduce: pop n values, apply semantic rule, push $$
│     └── Synthesized attributes: trivially handled this way
│
├── Grammar Design for SDT
│     ├── Avoid grammar where type is unknown at reduction time
│     ├── Techniques: marker nonterminals (hooks), cloned productions
│     └── Right-recursive id lists preferred over left-recursive for type propagation
│
├── Semantic Actions Can:
│     ├── Compute attribute values
│     ├── Emit intermediate code (TAC)
│     ├── Insert into symbol table
│     └── Print error diagnostics
│
└── Symbol Table Integration
      ├── Type info flows via inherited attributes or hooks
      └── addtype(name, dtype) called during declaration reductions
```

---

## 📚 References

1. **Aho, Lam, Sethi, Ullman** — *Compilers: Principles, Techniques, and Tools* (Dragon Book), 2nd ed., Chapters 5 (Syntax-Directed Translation), 6 (Intermediate Code Generation). The primary textbook; all examples in these notes are grounded in Dragon Book formalism.
2. **Cooper & Torczon** — *Engineering a Compiler*, 2nd ed., Chapter 4 (Intermediate Representations) and Chapter 5 (Syntax-Driven Translation). Exceptionally clear diagrams of attribute flow in parse trees.
3. **Appel, A.** — *Modern Compiler Implementation in Java/C/ML*, Chapter 4. Shows practical SDT via recursive-descent with explicit inherited attribute passing.
4. **Knuth, D. E. (1968)** — *"Semantics of Context-Free Languages"*, Mathematical Systems Theory. The original paper defining attribute grammars — foundational reading.
5. **Deransart, P., Jourdan, M., Lorho, B.** — *Attribute Grammars: Definitions, Systems, and Bibliography* (Springer LNCS 323). Exhaustive reference on the theory of attribute grammars.
6. **GNU Bison Manual** — https://www.gnu.org/software/bison/manual/ — Sections on mid-rule actions (markers/hooks), `%union`, and value stack (`$$`, `$n`) — directly maps to the Yacc integration discussed here.
7. **PLY (Python Lex-Yacc)** — https://www.dabeaz.com/ply/ — Experiment with semantic rules and value stacks in pure Python without a C build chain.
8. **LLVM Tutorial: Kaleidoscope** — https://llvm.org/docs/tutorial/ — A real-world worked example of building a parser + attribute evaluator + IR emitter from scratch.

---

## ❓ Active Recall

### Level 1 — Definitions
- [ ] What is an attribute grammar? Name its three components.
- [ ] Define synthesized attribute. In which direction does it flow in the parse tree?
- [ ] Define inherited attribute. What can an inherited attribute of `Xⱼ` depend on?
- [ ] What is a dependency graph in the context of attribute grammars? What does a cycle mean?
- [ ] What is an L-attributed grammar? Why is "L" the right letter?

### Level 2 — Mechanics
- [ ] For the rule `expr → expr₁ '+' term`, write the semantic rule for `expr.val`. What kind of attribute is `expr.val`?
- [ ] For the declaration grammar `decl → type var-list`, `var-list.dtype` is an inherited attribute. Trace exactly how `dtype = real` reaches `id(x)` and `id(y)` in the parse tree for `float x, y;`.
- [ ] Explain the parallel value stack in LR parsing. What happens to the value stack on a shift? On a reduce?
- [ ] What is a "hook" (marker nonterminal)? Why is it needed for declaration grammars?
- [ ] Why is the grammar `D → int namelist ; / namelist → id , namelist | id` problematic for bottom-up SDT?

### Level 3 — Analysis
- [ ] Is every S-attributed grammar also L-attributed? Is every L-attributed grammar also S-attributed? Justify.
- [ ] Why can't inherited attributes be directly computed using the LR value stack? What workarounds exist?
- [ ] Given the grammar `declaration → id_list ':' type`, explain exactly what goes wrong during bottom-up parsing when you try to associate the type with each id.
- [ ] A semantic action has a side effect (e.g., calling `addtype()`). Why must you ensure the action executes in the correct dependency order?

### Level 4 — Synthesis
- [ ] Design an attribute grammar for a language with typed expressions: `int x; float y; x + y` should produce a type error. Show the grammar rules, attributes, and semantic rules.
- [ ] Given: `var x, y, z : real; u, v : integer;` — redesign the grammar so that type information can be correctly propagated to all variable names in a bottom-up LALR parse. Show the grammar and semantic rules.
- [ ] Write the semantic actions (in pseudocode or Yacc-style) to translate `a = b + c * d` into three-address code. Show the parse, the value stack evolution, and the emitted TAC instructions step by step.
- [ ] Why might a compiler designer choose "cloned productions" over "marker nonterminals" for handling type declarations? What are the trade-offs?
