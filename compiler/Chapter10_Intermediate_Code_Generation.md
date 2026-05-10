# 📄 Chapter 10 — Intermediate Code Generation

**Tags:** #compiler #intermediate-code #TAC #quadruples #triples #DAG #backpatching #type-checking #symbol-table #NCKU
**Links:** [[Syntax-Directed Translation]], [[Bottom-Up Parsing II]], [[Symbol Tables]], [[Code Optimization]], [[Code Generation]]

---

## 🎯 The "Elevator Pitch"
> After parsing and semantic analysis, the compiler still speaks the *source language*. Intermediate Code Generation translates that into a language-neutral, machine-neutral **Intermediate Representation (IR)** — close enough to the machine to be optimized, but far enough from hardware to be portable. The star of this chapter is **Three-Address Code (TAC)**: one operation, at most two operands, one result. Think of it as a universal assembly language that no real CPU speaks, but every compiler understands.

---

## 🧠 Part 1 — Why Intermediate Representations?

### The Compiler Pipeline

```
Source Code
    │
    ▼  Lexer + Parser + Semantic Analyzer
Annotated AST
    │
    ▼  Intermediate Code Generator   ← Chapter 10
Three-Address Code (IR)
    │
    ▼  Optimizer
Optimized IR
    │
    ▼  Code Generator
Target Assembly / Machine Code
```

### Why Not Go Directly to Machine Code?

An IR decouples the **front end** (language-specific) from the **back end** (machine-specific):

```
N source languages × M target machines
  = N×M compilers    (without IR)
  = N+M components   (with IR: N front ends + M back ends)
```

A single well-designed IR (like LLVM's IR) lets you add new languages and new targets independently.

### IR Spectrum: High-Level to Low-Level

```
Higher (closer to source)          Lower (closer to machine)
────────────────────────────────────────────────────────────
AST / DAG → Three-Address Code → Stack machine code → Assembly
```

This chapter focuses on the **middle tier**: DAGs and Three-Address Code.

---

## 🧠 Part 2 — Syntax Tree Variants: DAGs

### What is a DAG?

> A **Directed Acyclic Graph (DAG)** for an expression is a syntax tree where **common subexpressions share nodes** rather than being duplicated. Each node is computed only once.

### AST vs. DAG for `a + a * (b - c) + (b - c) * d`

```
AST (redundant):                    DAG (shared nodes):

        +                                   +
       / \                                 / \
      +   *                               +   *
     / \ / \                             / \ / \
    a  a *   d                          a   *   d
        / \                                / \
       a  b-c                             a  b-c
         also b-c again                     (shared!)
```

The DAG saves computation: `(b - c)` is computed **once**, not twice.

### Value-Number Method: Building a DAG

The key insight: if two subtrees compute the same thing, give them the same **value number** (node identity). Use a hash table from `(op, left_vn, right_vn)` → `node`.

```python
class DAGNode:
    def __init__(self, op, left=None, right=None, val=None):
        self.op    = op       # operator or leaf type ('id', 'const')
        self.left  = left     # left child node (or None for leaves)
        self.right = right    # right child node
        self.val   = val      # for leaves: variable name or constant value
        self.label = None     # computed value label (for CSE)


class DAGBuilder:
    """
    Build a DAG for an expression using the value-number method.
    Identical subexpressions map to the same node (sharing).
    """
    def __init__(self):
        # hash table: (op, left_id, right_id) -> existing DAGNode
        self._table: dict = {}
        self._leaves: dict = {}   # name/const -> leaf node

    def make_leaf(self, kind: str, val) -> DAGNode:
        """Get or create a leaf node for a variable or constant."""
        key = (kind, val)
        if key not in self._leaves:
            self._leaves[key] = DAGNode(op=kind, val=val)
        return self._leaves[key]   # ← sharing: same var → same node

    def make_node(self, op: str, left: DAGNode, right: DAGNode = None) -> DAGNode:
        """Get or create an interior node. Reuse if identical subtree exists."""
        key = (op, id(left), id(right))
        if key not in self._table:
            self._table[key] = DAGNode(op=op, left=left, right=right)
        return self._table[key]   # ← sharing: same computation → same node

    def build(self, expr_ast) -> DAGNode:
        """Recursively build DAG from an AST node."""
        if expr_ast.is_leaf:
            return self.make_leaf(expr_ast.kind, expr_ast.val)
        left  = self.build(expr_ast.left)
        right = self.build(expr_ast.right) if expr_ast.right else None
        return self.make_node(expr_ast.op, left, right)
```

### Hash Table Lookup for Value Numbers

```python
def find_or_create(op, left_vn, right_vn, node_array, hash_table):
    """
    Value-number method: look up (op, left_vn, right_vn) in hash table.
    If found, return existing node number. Otherwise, create new node.

    node_array: list of (op, left, right) records (index = value number)
    hash_table: dict mapping (op, left_vn, right_vn) -> value_number
    """
    key = (op, left_vn, right_vn)

    if key in hash_table:
        return hash_table[key]   # common subexpression found!

    # Create new node
    vn = len(node_array)
    node_array.append({'op': op, 'left': left_vn, 'right': right_vn})
    hash_table[key] = vn
    return vn
```

---

## 🧠 Part 3 — Three-Address Code (TAC): The Universal IR

### What is TAC?

> **Three-Address Code** is an IR where each instruction has at most **one operator**, **two operand addresses**, and **one result address**. Every complex expression is broken into a sequence of simple steps, each using compiler-generated temporaries.

```
a + b * c  becomes:
  t1 = b * c
  t2 = a + t1
```

### Addresses in TAC

An **address** is one of:
1. **Name** — a source variable (pointer to symbol table entry in implementation)
2. **Constant** — a literal value (`3`, `3.14`, `'x'`)
3. **Compiler-generated temporary** — `t1`, `t2`, … (also in symbol table)

### All 10 Kinds of TAC Instructions

| # | Pattern | Meaning | Notes |
|---|---|---|---|
| 1 | `A = B op C` | Binary arithmetic/logic | `op` ∈ `{+, -, *, /, %, &, |, ^, …}` |
| 2 | `A = op B` | Unary operation | `op` ∈ `{-, ~, !}` or type conversion |
| 3 | `goto L` | Unconditional jump | Go to label/quadruple L |
| 4 | `if A relop B goto L` | Conditional jump | `relop` ∈ `{<, >, ==, !=, <=, >=}` |
| 5 | `param A` / `call P, n` | Procedure call | Pass argument / invoke `P` with `n` args |
| 6 | `A = B[i]` | Array read | Load element at index `i` |
| 7 | `A[i] = B` | Array write | Store element at index `i` |
| 8 | `A = &B` | Address-of | `A` gets the address of `B` |
| 9 | `A = *B` | Dereference | `A` gets the value pointed to by `B` |
| 10 | `*A = B` | Indirect store | Store `B` into memory pointed to by `A` |

---

## 🧠 Part 4 — TAC Representations: Quadruples, Triples, Indirect Triples

### Quadruples (4-Tuple)

Each instruction stored as `(operator, arg1, arg2, result)`. Entries are symbol table indices in a real implementation.

```
Example: D = A * B + C

Quadruple table:
 Index  Operator  Arg1  Arg2  Result
   0      =*        A     B     T1      ← T1 = A * B
   1      =+       T1     C     T2      ← T2 = T1 + C
   2       =       T2           D       ← D = T2

In real implementation (indices into symbol table):
 Index  Operator  Arg1  Arg2  Result
   0       8        6     7     9       ← sym[6]=A, sym[7]=B, sym[9]=T1
   1      15        9     8    11       ← sym[8]=C, sym[11]=T2
   2       3       11          10       ← sym[10]=D
```

```python
class Quadruple:
    """Single TAC instruction stored as a 4-tuple."""
    def __init__(self, op, arg1=None, arg2=None, result=None):
        self.op     = op       # operator code (int in real impl)
        self.arg1   = arg1     # symtab index or None
        self.arg2   = arg2     # symtab index or None
        self.result = result   # symtab index or None

    def __repr__(self):
        return f"({self.op}, {self.arg1}, {self.arg2}, {self.result})"


class QuadrupleTable:
    def __init__(self):
        self._quads: list[Quadruple] = []

    @property
    def nextquad(self) -> int:
        """Index of the next quadruple to be emitted (NEXTQUAD)."""
        return len(self._quads)

    def emit(self, op, arg1=None, arg2=None, result=None) -> int:
        """Emit one quadruple; return its index."""
        idx = self.nextquad
        self._quads.append(Quadruple(op, arg1, arg2, result))
        return idx

    def patch(self, index: int, target: int):
        """Backpatch: fill in the 'result' (goto target) of quadruple at index."""
        self._quads[index].result = target

    def __getitem__(self, i):
        return self._quads[i]
```

### Triples (3-Tuple)

Drop the `result` field — instead, reference a previous instruction **by its index** (position in the table serves as the implicit name).

```
Example: D = A * B + C

Triples:
 Index  Operator  Arg1    Arg2
   0      =*        A       B        ← (0) = A * B
   1      =+       (0)      C        ← (1) = (0) + C  [ref to triple 0]
   2       =       (1)      D        ← D   = (1)
```

**Advantage**: Smaller representation (no result field).  
**Disadvantage**: Moving an instruction changes all references to it → optimization is hard.

### Indirect Triples

Store triples in a separate array; use a **pointer list** to specify execution order. Moving an instruction = updating one pointer, not all references.

```python
# Triples array (immutable, stable indices)
triples = [
    (Op.MUL, 'A', 'B'),     # triple 0
    (Op.ADD, Ref(0), 'C'),   # triple 1 — Ref(0) means "result of triple 0"
    (Op.ASSIGN, Ref(1), 'D') # triple 2
]

# Indirect list (execution order — can be reordered for optimization)
execution_order = [0, 1, 2]  # pointers into triples array
```

### Comparison Table

| Property | Quadruples | Triples | Indirect Triples |
|---|---|---|---|
| Result field | Explicit (`t1`) | Implicit (index) | Implicit (index) |
| Reordering for opt. | Easy (name persists) | Hard (indices shift) | Easy (update pointer) |
| Memory | 4 fields per instr | 3 fields per instr | 3 fields + pointer list |
| Used in practice | Most common (Yacc/Bison) | Historical | RISC compilers |

---

## 🧠 Part 5 — Type Expressions and Type Checking

### Type Expressions

> A **type expression** is the structured representation of a type, built from basic types and type constructors.

```
Basic types:     integer, real, char, boolean, void
Type constructors:
  array(n, T)    — array of n elements of type T
  pointer(T)     — pointer to T
  struct(fields) — record/struct
  function(T₁×T₂×…→Tₙ) — function type
```

**Example**: `int[2][3]` → `array(2, array(3, integer))`

```python
# Type expression tree
class TypeExpr:
    pass

class BasicType(TypeExpr):
    def __init__(self, name: str, width: int):
        self.name  = name    # 'integer', 'real', 'char', 'boolean'
        self.width = width   # bytes needed: char=1, int=2, float=4, double=8, ptr=4

class ArrayType(TypeExpr):
    def __init__(self, size: int, element_type: TypeExpr):
        self.size    = size
        self.element = element_type
        self.width   = size * element_type.width   # synthesized!

class PointerType(TypeExpr):
    def __init__(self, base_type: TypeExpr):
        self.base  = base_type
        self.width = 4    # all pointers are the same width (32-bit)

class StructType(TypeExpr):
    def __init__(self, fields: list[tuple[str, TypeExpr]]):
        self.fields = fields  # [(name, type), ...]
        self.width  = sum(f[1].width for f in fields)

# Example: int[2][3]
t_int = BasicType('integer', width=2)
t_arr3 = ArrayType(size=3, element_type=t_int)    # array(3, integer), width=6
t_arr23 = ArrayType(size=2, element_type=t_arr3)  # array(2, array(3,int)), width=12
```

### Width Computation via SDT (Synthesized Attribute)

```
Grammar Rule                Semantic Rule
──────────────────────────────────────────────────────
T → int                     T.width = 2
T → float                   T.width = 4
T → char                    T.width = 1
T → ptr                     T.width = 4
T → T₁[num]                 T.width = num.val × T₁.width
T → struct { FL }           T.width = FL.width

FL → F ;                    FL.width = F.width;  O_enter(F.name, 0)
FL → FL₁ F ;                FL.width = FL₁.width + F.width
                             O_enter(F.name, FL₁.width)

F → T id                    F.width = T.width;   W_enter(id, T.width)
F → F₁[num]                 F.width = F₁.width × num.val
                             D_enter(F₁.name, num.val)
```

### Storage Layout for Local Variables

```python
class StorageAllocator:
    """Assign relative memory offsets to local variables at compile time."""
    def __init__(self):
        self.offset = 0       # current offset from frame base

    def allocate(self, name: str, type_expr: TypeExpr, symtab) -> int:
        """Assign offset to 'name'; update offset by type's width."""
        addr = self.offset
        symtab.set_offset(name, addr)
        symtab.set_type(name, type_expr)
        self.offset += type_expr.width
        return addr


# SDT for declarations: D → T id ;
def sem_declaration(type_node, id_name, allocator, symtab):
    """
    Semantic action for:  D → T id ;
    Computes the offset (storage layout) for the declared variable.
    """
    allocator.allocate(id_name, type_node.type_expr, symtab)
```

---

## 🧠 Part 6 — Translation of Arithmetic Expressions

### Semantic Rules for Expressions (SDT)

```
Grammar Rule          Semantic Action
──────────────────────────────────────────────────────
A → id = E            GEN(id.addr = E.addr)

E → E₁ + E₂           T = NEWTEMP()
                       E.addr = T
                       GEN(T = E₁.addr + E₂.addr)

E → E₁ * E₂           T = NEWTEMP()
                       E.addr = T
                       GEN(T = E₁.addr * E₂.addr)

E → - E₁              T = NEWTEMP()
                       E.addr = T
                       GEN(T = - E₁.addr)

E → ( E₁ )            E.addr = E₁.addr     ← no new temp needed!

E → id                 E.addr = id.addr     ← lookup in symtab
```

### Type Coercion (int ↔ float)

When operand types differ, the compiler must insert **type conversion** instructions:

```python
def gen_binop(E1, E2, op, quad_table, symtab):
    """
    Generate TAC for E1 op E2 with automatic type coercion.
    Returns (result_temp, result_type).
    """
    T = symtab.new_temp()

    if E1.type == 'int' and E2.type == 'int':
        quad_table.emit(f'int{op}', E1.addr, E2.addr, T)
        result_type = 'int'

    elif E1.type == 'float' and E2.type == 'float':
        quad_table.emit(f'float{op}', E1.addr, E2.addr, T)
        result_type = 'float'

    elif E1.type == 'int' and E2.type == 'float':
        # Widen E1: int → float first
        U = symtab.new_temp()
        quad_table.emit('inttofloat', E1.addr, None, U)
        quad_table.emit(f'float{op}', U, E2.addr, T)
        result_type = 'float'

    elif E1.type == 'float' and E2.type == 'int':
        # Widen E2: int → float first
        U = symtab.new_temp()
        quad_table.emit('inttofloat', E2.addr, None, U)
        quad_table.emit(f'float{op}', E1.addr, U, T)
        result_type = 'float'

    return T, result_type
```

### Full Expression Translator

```python
def translate_expr(node, qt, symtab):
    """
    Recursively translate an expression AST node to TAC.
    Returns the 'place' (address/temp name) holding the result.
    """
    if node.kind == 'id':
        sym = symtab.lookup(node.name)
        if sym is None:
            raise SemanticError(f"Undeclared: {node.name}")
        return sym.addr   # E.addr = id.addr

    elif node.kind == 'num':
        return str(node.value)   # constant directly usable as address

    elif node.kind == 'unary_minus':
        e = translate_expr(node.child, qt, symtab)
        t = symtab.new_temp()
        qt.emit('uminus', e, None, t)   # t = - e
        return t

    elif node.kind == 'paren':
        return translate_expr(node.child, qt, symtab)   # transparent

    elif node.kind in ('+', '-', '*', '/'):
        e1 = translate_expr(node.left, qt, symtab)
        e2 = translate_expr(node.right, qt, symtab)
        t  = symtab.new_temp()
        qt.emit(node.kind, e1, e2, t)   # t = e1 op e2
        return t

    elif node.kind == 'assign':
        e = translate_expr(node.right, qt, symtab)
        qt.emit('=', e, None, node.left.name)   # id = e
        return node.left.name
```

---

## 🧠 Part 7 — Addressing Array Elements

### Row-Major Layout (C/most languages)

For a 2D array `A[low₁..high₁][low₂..high₂]` stored row-major:

```
Address of A[i][j] = base_address
                   + (i - low₁) × (high₂ - low₂ + 1) × width
                   + (j - low₂) × width
```

**Generalizing to k dimensions**:

```
For A[i₁][i₂]…[iₖ] with bounds nⱼ = highⱼ - lowⱼ + 1:

addr = base + ((i₁-low₁)×n₂×n₃×…×nₖ
             + (i₂-low₂)×n₃×…×nₖ
             + …
             + (iₖ-lowₖ)) × element_width
```

### 💡 Intuition: Row-Major as Nested Loops

Think of a 2D array as a grid stored row by row:
```
A[0][0] A[0][1] A[0][2] | A[1][0] A[1][1] A[1][2] | …
         ← row 0 →                ← row 1 →
```
To reach row `i`, skip `i × columns` elements. Then skip `j` more for the column.

### TAC for Array References

```
Grammar:                     Semantic Action:
─────────────────────────────────────────────────────
L → id [ E ]                 L.array = id.array_info
                              L.addr = E.addr
                              L.type = id.element_type

L → L₁ [ E ]                 t = NEWTEMP()
                              L.array = L₁.array
                              L.type = L₁.type.element_type
                              GEN(t = L₁.addr * L₁.type.width)
                              GEN(t = t + E.addr)     [running index]
                              L.addr = t

E → L                         t = NEWTEMP()
                              GEN(t = L.array.base_addr [ L.addr ])
                              E.addr = t
                              [Access: t = A[offset]]
```

```python
def translate_array_access(array_node, index_exprs, qt, symtab):
    """
    Translate A[i][j][k] into TAC.
    Computes running offset: ((i * n2 + j) * n3 + k) * elem_width + base
    """
    array_info = symtab.lookup(array_node.name)
    dims       = array_info.dimensions   # list of (low, high, n)
    elem_width = array_info.base_type.width

    running_offset = None

    for dim_idx, (idx_expr, (low, high, n)) in enumerate(
            zip(index_exprs, dims)):
        idx_place = translate_expr(idx_expr, qt, symtab)

        # Adjust for lower bound: (iₖ - lowₖ)
        if low != 0:
            t_adj = symtab.new_temp()
            qt.emit('-', idx_place, str(low), t_adj)
            idx_place = t_adj

        if running_offset is None:
            running_offset = idx_place
        else:
            # running = running * n + idx
            t_mul = symtab.new_temp()
            t_add = symtab.new_temp()
            qt.emit('*', running_offset, str(n), t_mul)
            qt.emit('+', t_mul, idx_place, t_add)
            running_offset = t_add

    # Multiply by element width
    t_byte_offset = symtab.new_temp()
    qt.emit('*', running_offset, str(elem_width), t_byte_offset)

    # Final access: result = array_base[byte_offset]
    result = symtab.new_temp()
    qt.emit('=[]', array_node.name, t_byte_offset, result)
    return result
```

---

## 🧠 Part 8 — Boolean Expressions & Backpatching

This is the **most conceptually complex** part of the chapter. Read carefully.

### The Two Approaches

| Approach | How | When |
|---|---|---|
| **Numerical encoding** | Compute 0 or 1 into a temp; test with `if` | When boolean result is used as a value (e.g., `x = a > b`) |
| **Control-flow (jump) encoding** | Emit conditional jumps; use **backpatching** | When boolean controls flow (`if`, `while`, `&&`, `||`) |

We focus on the control-flow approach because it avoids redundant comparisons.

### Key Attributes

```
E.true  = list of quadruple indices that should jump to the
          "true" successor (goto target unfilled = goto _)

E.false = list of quadruple indices that should jump to the
          "false" successor (goto target unfilled = goto _)

M.quad  = NEXTQUAD at the moment M → ε is reduced
          (records the index of the next instruction to be emitted)
```

### 💡 Intuition: Backpatching is "IOU for Jump Targets"

When we emit `if p < q goto _`, we don't yet know *where* the "true" branch leads — we haven't parsed it yet. So we write an IOU: "this quadruple needs a true-target, to be filled in later." We store these IOUs in `E.true` and `E.false` lists. Later, when we know the target (`BACKPATCH(list, target)`), we go back and fill in all the blanks.

### Functions

```python
class BackpatchEngine:
    def __init__(self, qt: QuadrupleTable):
        self.qt = qt

    def makelist(self, i: int) -> list:
        """Create a singleton list containing quad index i."""
        return [i]

    def merge(self, *lists) -> list:
        """Concatenate multiple IOU lists."""
        result = []
        for lst in lists:
            result.extend(lst)
        return result

    def backpatch(self, patch_list: list, target: int):
        """Fill in the goto target for all quads in patch_list."""
        for quad_idx in patch_list:
            self.qt.patch(quad_idx, target)
            # This fills the '_' in 'goto _' with the actual target
```

### Boolean Expression Grammar + Semantic Actions

```python
"""
Grammar (with marker M for recording NEXTQUAD):

    M  → ε
    E  → E₁ or M E₂
       | E₁ and M E₂
       | not E₁
       | ( E₁ )
       | id
       | id₁ relop id₂

Semantic actions:
"""

def sem_M(qt) -> int:
    """M → ε   :   M.quad = NEXTQUAD"""
    return qt.nextquad   # record current position

def sem_E_or(E1, M_quad, E2, bp):
    """
    E → E₁ or M E₂
    Meaning: if E₁ is true → skip E₂ (E₁.true list already jumps past M)
             if E₁ is false → try E₂ (backpatch E₁.false to M.quad)
    """
    bp.backpatch(E1.false, M_quad)   # E₁.false jumps to start of E₂
    return {
        'true':  bp.merge(E1.true, E2.true),  # either condition true
        'false': E2.false                       # both conditions false
    }

def sem_E_and(E1, M_quad, E2, bp):
    """
    E → E₁ and M E₂
    Meaning: if E₁ is true → check E₂ (backpatch E₁.true to M.quad)
             if E₁ is false → skip E₂ (short-circuit)
    """
    bp.backpatch(E1.true, M_quad)    # E₁.true jumps to start of E₂
    return {
        'true':  E2.true,
        'false': bp.merge(E1.false, E2.false)  # either condition false
    }

def sem_E_not(E1):
    """E → not E₁  :  swap true and false lists"""
    return {'true': E1.false, 'false': E1.true}

def sem_E_relop(id1_addr, relop, id2_addr, qt, bp):
    """
    E → id₁ relop id₂
    Emits two quads: conditional jump (true branch) and goto (false branch).
    Both targets are left as '_' (IOU).
    """
    true_quad  = qt.nextquad
    qt.emit(f'if_{relop}_goto', id1_addr, id2_addr, None)  # goto _
    false_quad = qt.nextquad
    qt.emit('goto', None, None, None)                       # goto _
    return {
        'true':  bp.makelist(true_quad),
        'false': bp.makelist(false_quad)
    }
```

### Concrete Example: `if p < q || r < s && t < u  x = y + z; k = m - n;`

```
NEXTQUAD = 100

Step 1: Reduce (p < q)
  Emit quad 100: if p < q goto _    → E₁.true  = [100]
  Emit quad 101: goto _             → E₁.false = [101]

Step 2: M → ε  (M₁.quad = 102)

Step 3: Reduce (r < s)
  Emit quad 102: if r < s goto _   → E₃.true  = [102]
  Emit quad 103: goto _            → E₃.false = [103]

Step 4: M → ε  (M₂.quad = 104)

Step 5: Reduce (t < u)
  Emit quad 104: if t < u goto _   → E₅.true  = [104]
  Emit quad 105: goto _            → E₅.false = [105]

Step 6: Reduce E₃ AND M₂ E₅:
  BACKPATCH([102], 104)  → quad 102 becomes: if r < s goto 104
  E_and.true  = [104]
  E_and.false = merge([103], [105]) = [103, 105]

Step 7: Reduce E₁ OR M₁ E_and:
  BACKPATCH([101], 102)  → quad 101 becomes: goto 102
  E_or.true  = merge([100], [104]) = [100, 104]
  E_or.false = [103, 105]

Step 8: NEXTQUAD = 106
  BACKPATCH(E_or.true, 106)  → quads 100,104 jump to 106
  Emit quad 106: t1 = y + z
  Emit quad 107: x = t1
  NEXTQUAD = 108
  E_or.false list [103,105] patched to 108:
    quad 103: goto 108
    quad 105: goto 108
  Emit quad 108: t2 = m - n
  Emit quad 109: k = t2

Final TAC:
  100: if p < q goto 106
  101: goto 102
  102: if r < s goto 104
  103: goto 108
  104: if t < u goto 106
  105: goto 108
  106: t1 = y + z
  107: x = t1
  108: t2 = m - n
  109: k = t2
```

---

## 🧠 Part 9 — Control Flow: if, if-else, while

### Marker and Jump Nonterminals

```
M → ε   { M.quad = NEXTQUAD; }   ← records current quad position
N → ε   { N.next = MAKELIST(NEXTQUAD);
           GEN(goto _); }          ← emits unconditional jump, IOU
```

### Grammar + Semantic Rules for Flow Control

```python
"""
Grammar:
  S → if E then M₁ S₁ N else M₂ S₂
  S → if E then M S₁
  S → while M₁ E do M₂ S₁
  S → A                            (assignment)
  L → S
  L → L₁ ; M S
"""

def sem_if_then_else(E, M1_quad, S1, N, M2_quad, S2, bp):
    """
    S → if E then M₁ S₁ N else M₂ S₂
    Flow:
      E.true  → M₁ (start of then-branch)
      E.false → M₂ (start of else-branch)
      N       → past S₂ (end of entire if-else)
    """
    bp.backpatch(E['true'],  M1_quad)   # true  → then branch
    bp.backpatch(E['false'], M2_quad)   # false → else branch
    S_next = bp.merge(S1['next'], N['next'], S2['next'])
    return {'next': S_next}

def sem_if_then(E, M_quad, S1, bp):
    """
    S → if E then M S₁
    E.true  → M (then branch)
    E.false → falls through (past S₁)
    """
    bp.backpatch(E['true'], M_quad)
    S_next = bp.merge(E['false'], S1['next'])
    return {'next': S_next}

def sem_while(M1_quad, E, M2_quad, S1, qt, bp):
    """
    S → while M₁ E do M₂ S₁
    M₁ = re-evaluate condition each iteration
    M₂ = start of body
    Flow:
      E.true   → M₂ (enter body)
      S₁.next  → M₁ (loop back to condition)
      goto M₁  (explicit back-edge)
      E.false  → exit loop
    """
    bp.backpatch(E['true'],  M2_quad)   # true  → loop body
    bp.backpatch(S1['next'], M1_quad)   # end of body → re-test
    qt.emit('goto', None, None, M1_quad) # unconditional back-edge
    return {'next': E['false']}          # exit when E is false

def sem_stmt_sequence(L1, M_quad, S, bp):
    """
    L → L₁ ; M S
    The ';' between statements: patch L₁'s dangling jumps to M (start of S).
    """
    bp.backpatch(L1['next'], M_quad)
    return {'next': S['next']}
```

### While Loop Example: `while (A < B) do if (C < D) then X = Y + Z;`

```
NEXTQUAD = 100

M₁.quad = 100

Reduce (A < B):
  100: if A < B goto _   E.true=[100], E.false=[101]
  101: goto _

M₂.quad = 102

Reduce (C < D):
  102: if C < D goto _   E2.true=[102], E2.false=[103]
  103: goto _

Reduce  "if E then M S":
  BACKPATCH([102], 104)   → 102: if C < D goto 104
  S₁.next = merge([103], [])  = [103]

  104: T = Y + Z
  105: X = T

Reduce "while M₁ E do M₂ S₁":
  BACKPATCH(E.true=[100], M₂=102)  → 100: if A < B goto 102
  BACKPATCH(S₁.next=[103], M₁=100) → 103: goto 100
  EMIT(goto M₁=100)                → 106: goto 100
  S.next = E.false = [101]
  BACKPATCH([101], 107)             → 101: goto 107

Final:
  100: if A < B goto 102
  101: goto 107
  102: if C < D goto 104
  103: goto 100
  104: T = Y + Z
  105: X = T
  106: goto 100
  107: ...
```

---

## 🧠 Part 10 — Procedure Calls

### Grammar and Semantic Actions

```
Grammar:
  call → id ( args )
  args → args , E
  args → E
```

A **QUEUE** accumulates argument addresses left-to-right. After all args are parsed, emit `param` for each, then `call`.

```python
class ProcedureCallTranslator:
    def __init__(self, qt: QuadrupleTable):
        self.qt = qt
        self.queue = []   # accumulates E.addr for each argument

    def args_single(self, E_addr):
        """args → E  :  initialize QUEUE with first arg."""
        self.queue = [E_addr]

    def args_append(self, E_addr):
        """args → args , E  :  append to QUEUE."""
        self.queue.append(E_addr)

    def call_procedure(self, proc_name_addr):
        """
        call → id ( args )
        Emit: param p  for each arg p in QUEUE (in order)
        Then: call id, n
        """
        n = len(self.queue)
        for arg_addr in self.queue:
            self.qt.emit('param', arg_addr)
        self.qt.emit('call', proc_name_addr, n)
        self.queue.clear()

# Example: f(a, b+c, d)
# args → E:       queue = [a]
# args → args, E: queue = [a, t1]   (where t1 = b + c)
# args → args, E: queue = [a, t1, d]
# call → id(args):
#   param a
#   param t1
#   param d
#   call f, 3
```

---

## 🧠 Part 11 — Struct/Record Type Declarations

### SDT Functions for Structs

```
W_enter(name, width)  — store element width for field 'name'
D_enter(name, size)   — add a dimension of size 'size' to array field
O_enter(name, offset) — store memory offset of field 'name'
```

### Grammar and Semantic Rules

```
field → T id          { W_enter(id, T.width);
                         field.width = T.width;
                         field.name  = id }

field → field₁[num]   { D_enter(field₁.name, num.val);
                          field.width = field₁.width × num.val;
                          field.name  = field₁.name }

fieldlist → field ;   { O_enter(field.name, 0);
                          fieldlist.width = field.width }

fieldlist → fieldlist₁ field ;
                        { O_enter(field.name, fieldlist₁.width);
                          fieldlist.width = fieldlist₁.width + field.width }

type → struct { fieldlist }
                        { type.width = fieldlist.width }
```

### Example: `struct { int x; float y; char k[10]; }`

```python
# Field by field layout:
# x:    int    → width=2, offset=0
# y:    float  → width=4, offset=2
# k:    char   → width=1, char[10] → width=10, offset=6
# struct total width = 2 + 4 + 10 = 16 bytes

struct_layout = {
    'x': {'type': 'int',   'width': 2,  'offset': 0},
    'y': {'type': 'float', 'width': 4,  'offset': 2},
    'k': {'type': 'char',  'width': 1,
          'dims': [10],    'array_width': 10, 'offset': 6},
}
struct_total_width = 16
```

---

## 🧠 Part 12 — Switch Statements

### Translation Strategy: Linear If-Chains

```c
switch (E) {
    case V1: S1; break;
    case V2: S2; break;
    ...
    default: Sn;
}
```

Compiles to:

```
100:  T = [evaluate E]
101:  if T == V1 goto 104    // compare and jump for each case
102:  ...
      ...
      if T == Vₙ₋₁ goto ...
      [code for Sₙ₋₁]
      goto end
default:
      [code for Sₙ (default)]
end:  ...
```

```python
def translate_switch(expr_place, cases, default_stmts, qt, bp):
    """
    Translate switch statement to TAC.
    cases: list of (value, statements)
    """
    T = qt.symtab.new_temp()
    qt.emit('=', expr_place, None, T)   # T = E

    end_jumps = []   # all 'goto end' quads (to backpatch at end)

    for value, stmts in cases:
        # Compare: if T != value, skip this case
        skip_quad = qt.nextquad
        qt.emit(f'if_neq_goto', T, str(value), None)   # goto next case

        # Emit case body
        for stmt in stmts:
            translate_stmt(stmt, qt, bp)

        # Emit: goto end (break)
        end_jumps.append(qt.nextquad)
        qt.emit('goto', None, None, None)

        # Backpatch the skip jump to here
        qt.patch(skip_quad, qt.nextquad)

    # Default case
    for stmt in default_stmts:
        translate_stmt(stmt, qt, bp)

    # Patch all 'break' jumps to end
    end_target = qt.nextquad
    bp.backpatch(end_jumps, end_target)
```

---

## 🧠 Part 13 — Symbol Table: Full Reference

### Structure

```python
class SymbolTable:
    """
    Scoped symbol table implemented as a stack of hash maps.
    Most recently declared name always shadows outer declarations.
    """
    def __init__(self):
        self.scopes: list[dict] = [{}]   # global scope at bottom
        self._temp_count = 0
        self._offset = 0

    # ── Scope management ──────────────────────────────────────────
    def open_scope(self):
        """Enter a new block (function body, if block, etc.)."""
        self.scopes.append({})
        self._offset = 0     # reset offset for new frame

    def close_scope(self):
        """Exit current block; all locals are now invisible."""
        if len(self.scopes) == 1:
            raise RuntimeError("Cannot close global scope")
        self.scopes.pop()

    # ── Insert / lookup / delete ──────────────────────────────────
    def insert(self, name: str, sym: Symbol):
        """
        Insert into current (innermost) scope.
        Does NOT overwrite — shadows the outer declaration.
        """
        current = self.scopes[-1]
        if name in current:
            raise SemanticError(f"Redeclaration: '{name}'")
        current[name] = sym

    def lookup(self, name: str) -> Symbol | None:
        """
        Search innermost → outermost scope.
        Most Closely Nested rule: inner scope shadows outer.
        """
        for scope in reversed(self.scopes):
            if name in scope:
                return scope[name]
        return None   # not declared

    def lookup_current(self, name: str) -> Symbol | None:
        """Lookup only in the current (innermost) scope."""
        return self.scopes[-1].get(name, None)

    def delete(self, name: str):
        """
        Remove the MOST RECENT declaration of 'name'.
        Uncovers any outer declaration.
        """
        for scope in reversed(self.scopes):
            if name in scope:
                del scope[name]
                return
        raise KeyError(f"'{name}' not found in any scope")

    # ── Compiler temporaries ──────────────────────────────────────
    def new_temp(self) -> str:
        """Generate a fresh compiler temporary (t0, t1, t2, …)."""
        name = f"t{self._temp_count}"
        self._temp_count += 1
        self.insert(name, Symbol(name=name, dtype=None,
                                  offset=self._offset))
        return name
```

### Underlying Data Structures

| Structure | Insert | Lookup | Delete | Notes |
|---|---|---|---|---|
| Linear list | O(1) (front insert) | O(n) | O(n) | Simple, slow for large tables |
| Binary search tree | O(log n) avg | O(log n) | O(log n) | Needs key ordering |
| AVL / B-tree | O(log n) guaranteed | O(log n) | O(log n) | Balanced → no worst-case |
| **Hash table** | O(1) avg | **O(1) avg** | O(1) avg | Used in real compilers |

### Scope Rules Summary

```
Most Closely Nested Rule:
  A name reference resolves to the nearest enclosing declaration.

Declaration Before Use:
  A name must be declared before any reference to it.
  Enables single-pass parsing: build symtab and resolve simultaneously.

Block Structure:
  Each {} block creates a new scope.
  On exit: all declarations in that block are removed (close_scope).
  Outer declarations become visible again.
```

---

## ⚠️ Edge Cases & Constraints

- **Backpatching with no true/false lists**: An assignment `S → A` has `S.next = null` (empty list) because assignments don't produce conditional jumps — nothing to backpatch.
- **N → ε for if-then-else**: The `N` nonterminal emits an unconditional `goto _` at the end of the then-branch to skip the else-branch. Without `N`, control falls through into the else-branch even when the condition was true.
- **NEWTEMP() must be in symtab**: Every compiler-generated temporary is an actual symbol table entry, indexed in quadruples just like named variables.
- **Type coercion is unidirectional**: `int → float` (widening) is implicit; `float → int` (narrowing) requires an explicit cast. Mishandling this causes precision bugs.
- **Row-major vs. column-major**: C uses row-major; Fortran uses column-major. The address formula differs — using the wrong one produces incorrect array accesses silently.
- **Struct alignment padding**: Real compilers add padding bytes between struct fields to satisfy hardware alignment requirements (e.g., `float` at a 4-byte-aligned address). The simplified grammar above ignores padding.
- **Switch jump tables**: For dense integer cases, compilers often generate a **jump table** (array of code addresses indexed by the switch value) instead of a linear if-chain — O(1) dispatch vs. O(n).
- **Backpatch with already-patched quads**: Calling BACKPATCH on a list twice is a bug — the second call overwrites the first target. Lists should be cleared after patching.

---

## 🔗 Complete Concept Map

```
Chapter 10: Intermediate Code Generation
│
├── Intermediate Representations
│     ├── DAG — syntax tree with shared common subexpressions
│     │     └── Value-number method: hash (op, left_vn, right_vn) → node
│     └── Three-Address Code (TAC)
│           ├── 10 instruction types (binary, unary, goto, if, param/call, array, pointer)
│           ├── Quadruples: (op, arg1, arg2, result) — 4 fields, result is explicit
│           ├── Triples: (op, arg1, arg2) — implicit result = position
│           └── Indirect Triples: triples + pointer list (reorderable)
│
├── Types and Declarations
│     ├── Type expressions: basic, array(n,T), pointer(T), struct
│     ├── Width = storage units (synthesized attribute)
│     └── Storage layout: offset allocated left-to-right in declaration order
│
├── Expression Translation
│     ├── NEWTEMP() → fresh temporary; E.addr = that temp
│     ├── GEN(...) → emit one quadruple
│     └── Type coercion: emit inttofloat before mixed-type ops
│
├── Array Addressing
│     └── Row-major formula → running index computation in TAC
│
├── Boolean Expressions — Backpatching
│     ├── E.true / E.false = lists of quads needing target (IOUs)
│     ├── M → ε captures NEXTQUAD (checkpoint)
│     ├── BACKPATCH(list, target) fills in all IOUs
│     ├── MAKELIST(i) creates singleton IOU list
│     └── MERGE(a,b) concatenates IOU lists
│
├── Control Flow
│     ├── if-then: backpatch E.true to M; S.next = merge(E.false, S₁.next)
│     ├── if-then-else: N emits extra goto; three backpatches
│     └── while: M₁ before condition, M₂ before body, GEN(goto M₁) at end
│
├── Procedure Calls
│     └── QUEUE collects arg addresses; emit param+call on reduction
│
├── Struct Declarations
│     └── W_enter / D_enter / O_enter → width, dims, offset in symtab
│
├── Switch Statements
│     └── Linear if-chain or jump table
│
└── Symbol Table
      ├── Operations: insert / lookup / delete / open_scope / close_scope
      ├── Scoped hash maps (stack): Most Closely Nested rule
      └── Implementations: linear list, BST, AVL, hash table
```

---

## 📚 References

1. **Aho, Lam, Sethi, Ullman** — *Compilers: Principles, Techniques, and Tools* (Dragon Book), 2nd ed., Chapters 6 (Intermediate Code Generation), 2.8 (Symbol Tables). Definitive reference for all material in this chapter; backpatching algorithm is directly from §6.6–6.7.
2. **Cooper & Torczon** — *Engineering a Compiler*, 2nd ed., Chapter 5 (Intermediate Representations). Especially clear on DAGs and value numbering (§5.3) and array addressing (§5.4).
3. **Appel, A.** — *Modern Compiler Implementation in Java/C/ML*, Chapter 7 (Translation to Intermediate Code). Shows a clean functional implementation of TAC generation with explicit attribute passing.
4. **Knuth, D. E. (1968)** — *"Semantics of Context-Free Languages"*, Mathematical Systems Theory. The mathematical foundation for attribute grammars (used for TAC generation via SDT).
5. **LLVM Language Reference Manual** — https://llvm.org/docs/LangRef.html — A real-world, production IR. LLVM IR is essentially typed TAC with SSA form; studying it illuminates the design space.
6. **GCC Internals Manual** — https://gcc.gnu.org/onlinedocs/gccint/ — How GCC's GIMPLE and RTL IRs are structured; shows how academic TAC scales to production compilers.
7. **Grune, van Reeuwijk, Bal, Jacobs, Langendoen** — *Modern Compiler Design*, 2nd ed., Chapter 7 (Intermediate Code Generation). Alternative treatment with extensive worked examples of backpatching.
8. **PLY (Python Lex-Yacc)** — https://www.dabeaz.com/ply/ — Experiment with TAC generation via Python semantic actions without a C toolchain.

---

## ❓ Active Recall

### Level 1 — Definitions
- [ ] What is a DAG? How does it differ from an AST? What problem does it solve?
- [ ] What are the three kinds of addresses in Three-Address Code?
- [ ] What are the four fields of a **quadruple**? How do **triples** differ?
- [ ] Define `E.true` and `E.false` in the backpatching framework. What do they contain?
- [ ] What does `M → ε` do? What attribute does `M` have, and what is its value?
- [ ] What does `BACKPATCH(list, target)` do concretely? What field does it fill?

### Level 2 — Mechanics
- [ ] Trace the value-number method to build a DAG for `a + b + a + b`. How many nodes result?
- [ ] Translate `x = (a + b) * (a + b)` into quadruples using NEWTEMP(). Show the symbol table entries for temporaries.
- [ ] For `int A[3][4]`, write the TAC to compute the address of `A[i][j]` (row-major, lower bound 0, element width 2).
- [ ] Trace the backpatching algorithm for `a < b && c < d`. Show NEXTQUAD at each step, the emitted quads, and the final E.true/E.false lists.
- [ ] Trace the while-loop translation for `while (x < 10) do x = x + 1;`. Show every quadruple emitted and every BACKPATCH call.

### Level 3 — Analysis
- [ ] Why is `goto _` emitted with an unfilled target? What prevents filling it immediately?
- [ ] Why does `S → A` set `S.next = null`? What would go wrong if it weren't null?
- [ ] In the `if-then-else` rule, what is the role of `N → ε`? What would happen without it?
- [ ] Why are **indirect triples** easier to optimize than plain triples?
- [ ] Explain the "Most Closely Nested Rule" for symbol table lookup. What data structure naturally implements it?

### Level 4 — Synthesis
- [ ] Design the complete semantic actions (SDT) for `for (init; cond; update) body`. What marker nonterminals do you need? Draw the backpatch flow.
- [ ] Given `struct { int x; char y[5]; float z; }`, compute the offset and width of each field. What is the total struct width? (char=1, int=2, float=4)
- [ ] Write the quadruples for `if (a < b && c > d) then x = y; else x = z;`. Show all backpatch calls.
- [ ] Modify the `gen_binop` function to also handle `int` × `double` coercions. What new conversion instruction do you need to emit?
