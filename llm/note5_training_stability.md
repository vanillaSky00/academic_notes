# 📄 Training Stability: Residual Connections, Layer Normalization & Loss

**Tags:** #llm #residual-connections #layer-norm #cross-entropy #training #interview
**Links:** [[Encoder-Decoder Architecture]], [[Inference Optimization]], [[Gradient Flow]]

---

## 🎯 The "Elevator Pitch"
> Training deep networks is like passing a whispered message through 96 people in a line — by the end, the signal is garbled or gone. Residual connections create a "phone bypass" so the original message always gets through; layer normalization keeps everyone speaking at the same volume; and cross-entropy loss measures how wrong each whispered word is.

---

## 🧠 Core Mechanics

### 1. Residual (Skip) Connections (Q11, Q28)

**The vanishing gradient problem recap:**
- During backpropagation, gradients are multiplied layer by layer.
- In very deep networks (dozens to hundreds of layers), gradients shrink exponentially → **vanishing gradients** → early layers barely update → training fails.

**What residual connections do:**
- Instead of `output = layer(input)`, compute:
  - `output = input + layer(input)`
- The gradient can now flow *directly* through the `+` (identity path) without passing through `layer`.
- This creates a **gradient highway** — even if `layer`'s gradient is tiny, the identity branch preserves gradient magnitude.

**Intuition:** The layer doesn't need to learn a full transformation from scratch. It only needs to learn a **small additive correction (residual)** to the input. This is easier and more stable.

```
x ──────────────────────────── (+) ──▶ output
 \                              ↑
  └──▶ [Self-Attn / FFN] ──────┘
           (learns the residual)
```

---

### 2. Layer Normalization (Q29)

**Why normalize?**
- Without normalization, activations can drift to very large or very small values as training progresses.
- Large activations → large gradients → **exploding gradients**
- Small activations → small gradients → **vanishing gradients**

**How Layer Normalization works:**
- For each token's feature vector, normalize across the *feature dimension* (not batch):
  - Compute mean `μ` and variance `σ²` across all `d_model` features
  - Normalize: `x̂ = (x - μ) / √(σ² + ε)`
  - Apply learned scale and shift: `y = γ · x̂ + β`
- Result: activations always have mean ≈ 0, variance ≈ 1 at each layer

**Contrast with Batch Normalization:**
| | Layer Norm | Batch Norm |
|---|---|---|
| Normalizes across | Feature dimension | Batch dimension |
| Works with batch size 1 | ✅ Yes | ❌ Unstable |
| Works with variable seq len | ✅ Yes | ❌ Problematic |
| Used in | Transformers | CNNs |

**Pre-LN vs Post-LN (important interview topic):**

| Variant | Formula | Notes |
|---|---|---|
| **Post-LN** (original paper) | `LayerNorm(x + Sublayer(x))` | Can produce unstable gradients; requires careful warm-up |
| **Pre-LN** (modern default) | `x + Sublayer(LayerNorm(x))` | More stable training; no warm-up needed; used in GPT, LLaMA |

---

### 3. Cross-Entropy Loss (Q30)

**Definition:** Measures how different the model's predicted probability distribution is from the true one-hot target distribution.

**Formula:**
```
L = -log( P(correct_token) )
```
- If the model assigns probability 0.9 to the correct token → Loss ≈ 0.105 (good)
- If the model assigns probability 0.01 to the correct token → Loss ≈ 4.6 (bad)
- The log makes large mistakes very costly, encouraging the model to be *confident* and *correct*.

**During Transformer training:**
1. For each position in the sequence, the model outputs logits over the entire vocabulary (`[vocab_size]`)
2. Softmax converts logits → probability distribution
3. Cross-entropy compares this distribution to the true next token (one-hot)
4. Loss is averaged across all positions and all sequences in the batch
5. Backpropagation updates all parameters to minimize this loss

**Intuition:** Cross-entropy is basically asking "How surprised was the model when it saw the correct answer?" Lower surprise = better model.

---

## 🔗 How These Three Work Together

```
Forward Pass:
  Input → [Layer] → Residual Add → LayerNorm → Next Layer → ... → Logits → Softmax → Cross-Entropy Loss
  
Backward Pass:
  Gradient flows back through:
  - Cross-entropy: tells the model how wrong it was
  - LayerNorm: keeps gradient scale stable
  - Residual connections: guarantees gradient reaches early layers
```

These three mechanisms are the core reason Transformers can be trained to 96+ layers without gradient failure.

---

## ⚠️ Edge Cases & Constraints
- **Residuals don't fix everything:** If the layer learns something counterproductive, the residual still adds it. The benefit is specifically gradient flow, not output quality directly.
- **LayerNorm has learnable parameters:** `γ` (scale) and `β` (shift) are learned. Without them, all layers would have identical normalized activations — losing expressive capacity.
- **Cross-entropy requires softmax probabilities:** Raw logits cannot be fed directly — softmax must be applied first. In PyTorch, `nn.CrossEntropyLoss` combines LogSoftmax + NLLLoss internally for numerical stability.
- **Label smoothing:** A common modification to cross-entropy that replaces the hard one-hot target with a soft distribution (e.g., 0.9 for the correct token, 0.1 spread over others) — improves calibration and generalization.

---

## 💻 Logical Code Snippet (Python)

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# ── Residual Connection ──────────────────────────────────────────────
class ResidualBlock(nn.Module):
    def __init__(self, sublayer):
        super().__init__()
        self.sublayer = sublayer

    def forward(self, x):
        return x + self.sublayer(x)  # input + layer(input) = residual

# ── Layer Normalization (Pre-LN style) ───────────────────────────────
class PreLNBlock(nn.Module):
    def __init__(self, d_model, sublayer):
        super().__init__()
        self.norm = nn.LayerNorm(d_model)
        self.sublayer = sublayer

    def forward(self, x):
        # Normalize first, then sublayer, then residual
        return x + self.sublayer(self.norm(x))

# ── Cross-Entropy Loss ───────────────────────────────────────────────
# During training: logits shape [batch, seq_len, vocab_size]
# Targets: shape [batch, seq_len] with integer class indices

criterion = nn.CrossEntropyLoss()

logits = torch.randn(2, 10, 50_000)        # batch=2, seq=10, vocab=50K
targets = torch.randint(0, 50_000, (2, 10)) # true next token IDs

# Reshape for loss computation
loss = criterion(
    logits.view(-1, 50_000),   # [batch*seq, vocab]
    targets.view(-1)            # [batch*seq]
)
# Backprop: loss.backward() updates all model parameters
```

---

## ❓ Active Recall
- [ ] What is the vanishing gradient problem, and why does it occur?
- [ ] How do residual connections mitigate vanishing gradients?
- [ ] What is the formula for a residual connection? What does the layer actually learn?
- [ ] What does layer normalization normalize across, and why is that different from batch normalization?
- [ ] What is Pre-LN vs. Post-LN? Which is more commonly used today and why?
- [ ] What does cross-entropy loss measure? Write out the formula.
- [ ] What does it mean when cross-entropy loss is very high? Very low?
- [ ] **Follow-up:** What is label smoothing, and why does it help?
- [ ] **Follow-up:** Why does PyTorch's `nn.CrossEntropyLoss` accept raw logits instead of probabilities?
