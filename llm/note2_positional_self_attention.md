# 📄 Positional Embeddings & Self-Attention Mechanics

**Tags:** #llm #self-attention #positional-encoding #transformer #interview
**Links:** [[Tokenization & Embeddings]], [[Multi-Head Attention]], [[Encoder-Decoder Architecture]]

---

## 🎯 The "Elevator Pitch"
> Self-attention lets every token "look at" every other token simultaneously to understand context — but since it has no sense of order, positional embeddings are injected to tell the model *where* each token sits in the sequence.

---

## 🧠 Core Mechanics

### 1. Why Transformers Need Positional Embeddings (Q1)

| Model | Positional Awareness | Mechanism |
|---|---|---|
| RNN | ✅ Inherent | Processes tokens sequentially, one by one |
| CNN | ✅ Inherent | Convolutional filters slide over local windows |
| Transformer | ❌ None by default | Processes all tokens **in parallel** → order-agnostic |

- **The problem:** Self-attention treats the input as a *set*, not a *sequence*. "The cat sat" and "Sat the cat" would produce identical attention scores without position info.
- **The fix:** Add a **positional embedding** vector to each token embedding before feeding into the Transformer. This injects explicit order information.
- **Types:** Sinusoidal (original paper), learned positional embeddings, RoPE (Rotary Position Embedding), ALiBi — each with different extrapolation properties for long sequences.

---

### 2. Self-Attention: Step-by-Step (Q9, Q15)

**Why it's called "self"-attention:** The attention is computed *within* the same sequence — each token attends to itself and all others using vectors all derived from the same input. No external sequence involved.

**Step-by-step computation:**

1. **Linear Projections** — For each input token embedding `x`, compute three vectors:
   - **Query (Q):** "What am I looking for?"
   - **Key (K):** "What do I offer?"
   - **Value (V):** "What information do I carry?"
   - Each is produced by multiplying `x` by learned weight matrices `Wq`, `Wk`, `Wv`

2. **Compute Attention Scores** — Dot product of each Query with all Keys:
   - `scores = Q · Kᵀ`
   - Result shape: `[seq_len × seq_len]` — every token vs every token

3. **Scale** — Divide by `√d_k` (square root of key dimension):
   - Prevents dot products from growing too large in high-dimensional spaces
   - Large scores → very peaked softmax → vanishing gradients on non-peak tokens
   - `scaled_scores = scores / √d_k`

4. **Softmax** — Normalize scores into a probability distribution (attention weights):
   - `attention_weights = softmax(scaled_scores)`
   - Each row sums to 1.0 → tells the model how much to "attend" to each token

5. **Weighted Sum of Values** — Multiply attention weights by the Value vectors:
   - `output = attention_weights · V`
   - Each token's output is now a context-aware blend of all value vectors

---

### 3. Why Scaling by √d_k Matters (Q19)

**Analogy:** Imagine whispering in a noisy room. If you speak too loudly (large dot product), everyone else sounds silent by comparison. Scaling turns the volume back down so the model can hear subtle relationships.

- Without scaling: dot products in high dimensions grow proportional to `d_k`, making softmax produce near-one-hot distributions
- Near-one-hot softmax → almost all gradient flows to one token → **effectively kills learning** for other relationships
- `√d_k` scaling keeps the variance of the dot products at ~1, preserving gradient flow

---

## 🔗 Summary Formula

```
Attention(Q, K, V) = softmax( Q·Kᵀ / √d_k ) · V
```

This single formula is the entire self-attention operation.

---

## ⚠️ Edge Cases & Constraints
- **Quadratic complexity:** Self-attention computes `n × n` pairwise interactions → `O(n² · d)`. For a sequence of 10K tokens, this is 100M score computations per layer — very expensive.
- **Position embeddings don't generalize out-of-distribution:** Standard sinusoidal embeddings struggle beyond the training sequence length. RoPE and ALiBi were designed to address this.
- **Common misconception:** Q, K, V are *not* three separate inputs — they're all derived from the *same* input via three different learned linear projections.
- **Self-attention is permutation-equivariant without positional embeddings:** Shuffling the input order produces shuffled outputs in the same way — the model has no preference for sequence order until positional info is injected.

---

## 💻 Logical Code Snippet (Python)

```python
import torch
import torch.nn.functional as F
import math

def self_attention(X, Wq, Wk, Wv):
    """
    X: input embeddings, shape [seq_len, d_model]
    Wq, Wk, Wv: learned weight matrices, shape [d_model, d_k]
    """
    # Step 1: Project inputs into Q, K, V spaces
    Q = X @ Wq  # [seq_len, d_k]
    K = X @ Wk  # [seq_len, d_k]
    V = X @ Wv  # [seq_len, d_k]

    d_k = Q.shape[-1]

    # Step 2 & 3: Compute scaled attention scores
    scores = Q @ K.transpose(-2, -1) / math.sqrt(d_k)  # [seq_len, seq_len]

    # Step 4: Softmax → attention weights
    attention_weights = F.softmax(scores, dim=-1)  # [seq_len, seq_len]

    # Step 5: Weighted sum of values
    output = attention_weights @ V  # [seq_len, d_k]

    return output, attention_weights
```

---

## ❓ Active Recall
- [ ] Why do CNNs and RNNs not need positional embeddings but Transformers do?
- [ ] What are the three vectors computed in self-attention and what does each "mean"?
- [ ] Walk through the 5 steps of self-attention from scratch.
- [ ] What is the formula for scaled dot-product attention?
- [ ] Why do we divide by `√d_k`? What happens if we don't?
- [ ] Why is self-attention called "self"-attention?
- [ ] **Follow-up:** What is the difference between sinusoidal and learned positional embeddings? What are their trade-offs?
- [ ] **Follow-up:** What is RoPE, and why was it designed?
