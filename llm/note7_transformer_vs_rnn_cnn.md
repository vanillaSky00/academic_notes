# 📄 Transformers vs. CNNs & RNNs — Architecture Comparison & Limitations

**Tags:** #llm #transformer #rnn #cnn #architecture #feed-forward #interview
**Links:** [[Self-Attention Mechanics]], [[Training Stability]], [[Inference Optimization]]

---

## 🎯 The "Elevator Pitch"
> RNNs read text like a person reading one word at a time — they're slow and forget early words. CNNs are like scanning a sentence through a small window — they see local patterns but miss distant relationships. Transformers read the whole sentence at once and let every word directly "talk" to every other word.

---

## 🧠 Core Mechanics

### 1. RNNs and Their Core Weakness (Q31, Q33)

**How RNNs work:**
- Process sequence tokens **one at a time**, left to right.
- Carry a hidden state `h_t` forward: `h_t = f(h_{t-1}, x_t)`
- To relate token 1 to token 100, the model must propagate information through **99 sequential steps**.

**Problems:**
- **Vanishing gradients over distance:** Gradient from position 100 must flow back 99 steps. Each multiplication by a weight <1 shrinks it exponentially → early tokens barely influence learning.
- **No parallelism:** Step `t` can't be computed until step `t-1` finishes → slow training on GPUs.
- **Long-range amnesia:** In practice, RNNs "forget" tokens more than ~50–100 steps back, even with LSTM/GRU improvements.

---

### 2. CNNs and Their Core Weakness (Q33)

**How CNNs are used for sequences:**
- Apply fixed-size convolutional filters (windows) that slide over the input.
- A filter of width 3 captures relationships between 3 adjacent tokens.
- To capture relationships across 100 tokens, you need many stacked layers.

**Problems:**
- **Fixed local receptive field:** Each filter can only see a small window. Distant relationships require stacking many layers — and even then, connectivity is implicit, not direct.
- **Not designed for variable-length sequences:** Input padding and masking add complexity.
- **No natural notion of sequence order:** Positional information must be baked in separately.

---

### 3. How Transformers Address Both (Q31, Q33)

| Property | RNN | CNN | Transformer |
|---|---|---|---|
| Long-range dependency | ❌ Degrades with distance | ❌ Requires many layers | ✅ Direct in 1 step (self-attention) |
| Parallelism (training) | ❌ Sequential | ✅ Parallel | ✅ Fully parallel |
| Complexity per layer | O(n · d²) | O(n · k · d²) | O(n² · d) |
| Handles variable-length | ✅ | ❌ Without tricks | ✅ |
| Positional awareness | ✅ Inherent | ✅ Partial | ❌ Needs positional embeddings |

**The key Transformer advantage:** Self-attention creates a **direct O(1) connection** between any two tokens regardless of distance. Token 1 and token 1000 interact in a single attention step with equal access.

**Analogy:** RNNs are like passing a note hand-to-hand across a room — information degrades with each pass. Transformers give every person in the room a direct phone line to every other person.

---

### 4. Fundamental Limitations of Transformers (Q32)

Despite their superiority, Transformers have real weaknesses:

| Limitation | Details |
|---|---|
| **Quadratic attention cost** | O(n² · d) makes very long sequences (>100K tokens) expensive or impossible without approximation |
| **High memory footprint** | Weights, activations, KV cache, and optimizer states all consume GPU RAM |
| **Data hunger** | Transformers need massive, diverse training corpora to learn well — they don't generalize from small datasets like structured models can |
| **Sensitivity to data quality** | Biased or low-quality training data propagates into model behavior — garbage in, garbage out |
| **No inherent sequential structure** | Positional embeddings are a workaround; the model has no native sense of order |
| **Interpretability** | Attention weights are often cited as explanations, but they don't reliably indicate causal importance |

---

### 5. The Position-Wise Feed-Forward Sublayer (Q35)

After self-attention, each Transformer layer applies a **feed-forward network (FFN)** independently to each token position.

**Structure:**
```
FFN(x) = ReLU(x · W₁ + b₁) · W₂ + b₂
```
- Two linear transformations with a ReLU (or GeLU in modern models) in between.
- Applied identically to **each token's vector independently** — hence "position-wise."
- Typically: `d_ff = 4 × d_model` (e.g., d_model=512 → d_ff=2048)

**What it does:**
- Self-attention mixes information **across** tokens (contextual blending).
- FFN processes each token's representation **individually** — adds non-linearity and increases representational capacity.
- Think of attention as "deciding what to pay attention to" and FFN as "thinking deeper about each token after gathering context."

**Analogy:** After a team meeting (self-attention) where everyone shares updates, each person goes away to process the information privately and develop their own deeper understanding (FFN).

---

## ⚠️ Edge Cases & Constraints
- **LSTMs partially solve RNN limitations** via gating mechanisms — but still can't match Transformers on long-range tasks and parallelism.
- **Efficient Transformers** (Longformer, BigBird, Mamba) attempt to break the O(n²) barrier with sparse or linear attention — important for long-document tasks.
- **FFN is not cross-token:** Unlike self-attention, the FFN at position `i` has no access to position `j`. It only refines what self-attention already gathered.
- **GeLU vs. ReLU:** Modern LLMs (GPT, LLaMA) use GeLU or SwiGLU instead of ReLU — smoother activation improves training stability and performance.

---

## 💻 Logical Code Snippet (Python)

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# ── Position-wise Feed-Forward Network ──────────────────────────────
class PositionWiseFFN(nn.Module):
    def __init__(self, d_model, d_ff):
        super().__init__()
        self.linear1 = nn.Linear(d_model, d_ff)
        self.linear2 = nn.Linear(d_ff, d_model)

    def forward(self, x):
        # x: [batch, seq_len, d_model]
        # Applied identically to each position — no cross-token mixing
        return self.linear2(F.relu(self.linear1(x)))

# ── Contrast: RNN processes sequentially, Transformer in parallel ────
class SimpleRNN(nn.Module):
    def __init__(self, d_model):
        super().__init__()
        self.rnn = nn.GRU(d_model, d_model, batch_first=True)

    def forward(self, x):
        # Must process token-by-token internally — inherently sequential
        output, hidden = self.rnn(x)
        return output  # info at position t depends on all steps 0..t

# For a transformer layer, all positions are processed in parallel via
# matrix multiplications in self-attention + FFN — no sequential dependency
```

---

## ❓ Active Recall
- [ ] Why do RNNs struggle with long-range dependencies?
- [ ] How does self-attention solve the long-range dependency problem in O(1) steps?
- [ ] Why can't CNNs capture relationships between distant tokens without many layers?
- [ ] What are the three main limitations of Transformers themselves?
- [ ] What does the position-wise FFN do, and how does it differ from self-attention?
- [ ] What is the typical relationship between `d_ff` and `d_model` in the FFN?
- [ ] Why is the FFN called "position-wise"?
- [ ] **Follow-up:** What is the difference between ReLU and GeLU? Why do modern models prefer GeLU?
- [ ] **Follow-up:** What is Mamba/SSM (State Space Model), and how does it challenge the Transformer's dominance?
- [ ] **Follow-up:** If self-attention is O(n²), how do sparse attention methods like Longformer reduce this?
