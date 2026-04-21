# 📄 Multi-Head Attention & Masking

**Tags:** #llm #multi-head-attention #masking #transformer #interview
**Links:** [[Self-Attention Mechanics]], [[Encoder-Decoder Architecture]], [[Softmax]]

---

## 🎯 The "Elevator Pitch"
> Multi-head attention runs self-attention several times in parallel with different "lenses," then merges the views — giving the model richer understanding. Masking blocks certain tokens from being seen, enforcing that during generation the model can only look backwards, not forwards.

---

## 🧠 Core Mechanics

### 1. Why Multiple Attention Heads? (Q20)

**Analogy:** A single spotlight can only illuminate one aspect of a stage. Multi-head attention is like many spotlights, each pointed at a different relationship in the sentence.

- A **single attention head** can only learn one type of relationship per layer (e.g., subject-verb agreement).
- **Multi-head attention** runs `h` independent attention computations in parallel, each in a lower-dimensional subspace:
  - Head 1 might learn **syntactic** relationships (subject → verb)
  - Head 2 might learn **coreference** (pronoun → noun)
  - Head 3 might learn **positional** proximity
- Each head operates on a lower-dimensional projection: `d_k = d_model / h`
- Result: richer, more expressive contextual representations

---

### 2. How Multi-Head Outputs Are Combined (Q21)

**Step-by-step:**
1. Each of the `h` heads independently computes `Attention(Q_i, K_i, V_i)` → produces output of shape `[seq_len, d_k]`
2. All head outputs are **concatenated** along the feature dimension → `[seq_len, h × d_k]` = `[seq_len, d_model]`
3. The concatenated result is passed through a **learned linear projection** `W_o` → projects back to `[seq_len, d_model]`

```
MultiHead(Q, K, V) = Concat(head_1, ..., head_h) · W_o
where head_i = Attention(Q·W_qi, K·W_ki, V·W_vi)
```

---

### 3. Masked Self-Attention vs. Regular Self-Attention (Q22, Q24, Q25)

| Feature | Regular Self-Attention (Encoder) | Masked Self-Attention (Decoder) |
|---|---|---|
| Token visibility | All tokens (bidirectional) | Only past tokens (causal/unidirectional) |
| Use case | Understanding input context | Autoregressive text generation |
| Where used | Encoder layers | Decoder layers |

**Why masking is necessary:**
- During training, the decoder receives the entire target sequence at once (for efficiency).
- Without masking, the model could "cheat" by looking at future tokens when predicting the current one.
- Masking enforces that position `t` can only attend to positions `≤ t` → this is the **autoregressive property**.

**How masking works mechanically (Q25):**
1. Compute raw attention scores as usual: `scores = Q · Kᵀ / √d_k`
2. Create a **causal mask** — an upper-triangular matrix of `-∞` values
3. Add the mask to the scores (future positions become `-∞`)
4. Apply softmax → `softmax(-∞) = 0` → future tokens get zero attention weight
5. The model effectively cannot see or attend to any token to its right

**Visualizing the mask (4-token sequence):**
```
Token:    t1    t2    t3    t4
t1:       ✅    ❌    ❌    ❌
t2:       ✅    ✅    ❌    ❌
t3:       ✅    ✅    ✅    ❌
t4:       ✅    ✅    ✅    ✅
```

---

### 4. The Softmax Function (Q27)

- **Definition:** Converts a vector of raw scores (logits) into a probability distribution where all values ∈ (0, 1) and sum to 1.
- **Formula:** `softmax(zᵢ) = exp(zᵢ) / Σ exp(zⱼ)`
- **Applied in Transformers in two places:**
  1. **Inside self-attention:** normalizes attention scores → attention weights
  2. **Output layer:** converts final logits over vocabulary → next-token probability distribution

---

## ⚠️ Edge Cases & Constraints
- **Head collapse:** In practice, many attention heads can converge to similar patterns — not all `h` heads learn distinct relationships. Pruning redundant heads is an active research area.
- **d_k must divide evenly:** Since `d_k = d_model / h`, you can't arbitrarily choose `h` without ensuring divisibility.
- **Masking at inference vs. training:** At inference, tokens are generated one at a time, so masking is automatically enforced by the autoregressive loop. The mask matters most during *training* when the full target sequence is fed in.
- **Common misconception:** Masking is *additive* (subtract `-∞`), not multiplicative (zero out). The subtraction happens *before* softmax, not after.

---

## 💻 Logical Code Snippet (Python)

```python
import torch
import torch.nn.functional as F
import math

def multi_head_attention(X, W_q, W_k, W_v, W_o, num_heads, mask=None):
    """
    X: [batch, seq_len, d_model]
    mask: optional causal mask [seq_len, seq_len] with -inf for blocked positions
    """
    batch, seq_len, d_model = X.shape
    d_k = d_model // num_heads

    # Step 1: Project and reshape into heads
    Q = (X @ W_q).view(batch, seq_len, num_heads, d_k).transpose(1, 2)  # [b, h, s, d_k]
    K = (X @ W_k).view(batch, seq_len, num_heads, d_k).transpose(1, 2)
    V = (X @ W_v).view(batch, seq_len, num_heads, d_k).transpose(1, 2)

    # Step 2: Scaled dot-product attention per head
    scores = Q @ K.transpose(-2, -1) / math.sqrt(d_k)  # [b, h, s, s]

    # Step 3: Apply causal mask if provided (masked self-attention)
    if mask is not None:
        scores = scores + mask  # mask contains -inf for future positions

    attn_weights = F.softmax(scores, dim=-1)  # [b, h, s, s]

    # Step 4: Weighted sum of values
    head_outputs = attn_weights @ V  # [b, h, s, d_k]

    # Step 5: Concatenate heads and project
    concat = head_outputs.transpose(1, 2).reshape(batch, seq_len, d_model)  # [b, s, d_model]
    output = concat @ W_o  # [b, s, d_model]

    return output


def make_causal_mask(seq_len):
    """Upper-triangular matrix of -inf (zeros on/below diagonal)"""
    mask = torch.triu(torch.full((seq_len, seq_len), float('-inf')), diagonal=1)
    return mask
```

---

## ❓ Active Recall
- [ ] Why does the Transformer use multiple attention heads instead of one?
- [ ] How are the outputs of multiple heads combined? What is the role of `W_o`?
- [ ] What is the difference between regular self-attention and masked self-attention?
- [ ] Where is masked self-attention used, and why?
- [ ] How exactly does the mask enforce that future tokens get zero attention weight?
- [ ] Where is softmax applied inside the Transformer (name both locations)?
- [ ] **Follow-up:** If d_model = 512 and num_heads = 8, what is d_k per head?
- [ ] **Follow-up:** What is "attention head pruning" and why might it be useful?
