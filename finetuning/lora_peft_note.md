# 📄 LoRA & Parameter-Efficient Fine-Tuning (PEFT)

**Tags:** #deep-learning #fine-tuning #LoRA #PEFT #LLM #linear-algebra #optimization  
**Links:** [[Fine-Tuning Math]], [[SVD]], [[Backpropagation]], [[AdamW]], [[DoRA]], [[QLoRA]], [[GaLore]]

---

## 🎯 The "Elevator Pitch"
> Instead of updating every single weight in a giant model (expensive), LoRA freezes all original weights and only trains two tiny matrices whose *product* approximates what the full update would have been — exploiting the fact that useful weight changes during fine-tuning are inherently low-dimensional.

---

## 🧠 Core Mechanics

### 1. The Problem: Full Fine-Tuning Is Memory-Brutal

When you do full fine-tuning, you need to hold in GPU memory:

| What | Memory cost |
|---|---|
| Base model weights $W$ | 1× model size |
| Gradients $\nabla W$ | 1× model size |
| Optimizer states (Adam: momentum + variance) | 2× model size |
| **Total** | **~4× model size** |

For a 7B parameter model in fp32, that's already ~28 GB just for weights — and you need 4× that = ~112 GB. Not a single consumer GPU in sight.

---

### 2. The Key Empirical Finding: Weight Updates Are Low-Rank

Here is the insight that makes LoRA work, and it's worth sitting with:

When you fine-tune a pre-trained model, the **weight update matrix** $\Delta W$ (the change applied to an existing weight matrix $W$) turns out to be **approximately low-rank** — meaning most of the "signal" in $\Delta W$ lives in a tiny subspace.

**How do we see this?** Apply **SVD** (Singular Value Decomposition) to $\Delta W$ from a real fine-tuning run:

$$
\Delta W = U \Sigma V^T
$$

$\Sigma$ is a diagonal matrix of singular values sorted largest to smallest. If you plot them, the first few are large and the rest decay rapidly toward zero. This means:

> The update that actually matters can be expressed as a combination of very few "directions" in weight space. Everything else is noise.

**Important nuance:** This doesn't mean the *base weights* $W$ are low-rank (they're not — they encode rich general knowledge). It only means the *change* $\Delta W$ you'd want to apply for a specific task is low-rank.

---

### 3. LoRA: The Mechanism — Step by Step

Normally in a forward pass through a linear layer:

$$
h = W x
$$

In **full fine-tuning**, after training you end up with:

$$
h = (W + \Delta W) x
$$

**LoRA says:** Instead of learning $\Delta W$ directly (same size as $W$, very expensive), approximate it as:

$$
\Delta W \approx B A
$$

where:
- $A \in \mathbb{R}^{r \times d_{\text{in}}}$ — a small matrix (projects input *down* to rank $r$)
- $B \in \mathbb{R}^{d_{\text{out}} \times r}$ — a small matrix (projects *back up* to output dimension)
- $r \ll \min(d_{\text{in}}, d_{\text{out}})$ — the **rank**, the critical hyperparameter

So the forward pass becomes:

$$
h = W x + B A x = \underbrace{W x}_{\text{frozen, no grad}} + \underbrace{B A x}_{\text{trained, tiny}}
$$

**The base model $W$ is completely frozen.** Zero gradients flow through it. Only $A$ and $B$ receive gradient updates.

---

### 4. Why Can You Freeze $W$ and Still Learn? (The Part That's Confusing)

This is the crux. Let's build the intuition carefully.

**Think of it this way:** The pre-trained $W$ already encodes general knowledge. When you fine-tune for a specific task, you're not replacing that knowledge — you're *adding a correction signal on top of it*.

The correction $\Delta W = BA$ gets added to every forward pass output. Backprop computes $\frac{\partial \mathcal{L}}{\partial B}$ and $\frac{\partial \mathcal{L}}{\partial A}$ — the chain rule flows through $B$ and $A$ as normal. $W$ is just a constant in the computation graph — it contributes to the output but has `requires_grad=False`, so no gradient is stored for it.

**Linear algebra view:** You're parameterizing $\Delta W$ in a constrained rank-$r$ subspace. The gradient descent is searching for the best rank-$r$ approximation to the "ideal" $\Delta W$ you would've learned with full fine-tuning. Because the ideal $\Delta W$ is empirically close to low-rank, the approximation is very good.

**Memory view:** $W$ is frozen → no gradient tensor allocated for it → optimizer state is only for $A$ and $B$ → massive memory savings.

---

### 5. Initialization: Why $B = 0$ at Start

At the beginning of LoRA training:
- $A$ is initialized from a **Gaussian distribution** (random small values)
- $B$ is initialized to **all zeros**

This means $\Delta W = BA = 0$ at step 0. The model starts *exactly* as the pre-trained model, with no random noise injected. This is critical for training stability — you don't want to corrupt the pre-trained representation before training even begins.

---

### 6. The Rank $r$ Hyperparameter

| $r$ | Parameters trained | When to use |
|---|---|---|
| 1 | Minimal | RL, tiny tasks, surprising how often works |
| 4–8 | Low (original LoRA default) | Good starting point for most SFT tasks |
| 16–64 | Medium | Larger domain shifts, more data |
| Equal to $\min(d_{\text{in}}, d_{\text{out}})$ | Same as full fine-tuning | You've defeated the purpose |

**Concrete example:** A weight matrix $W \in \mathbb{R}^{1000 \times 1000}$ has $10^6$ parameters.  
With $r = 2$: $A \in \mathbb{R}^{2 \times 1000}$, $B \in \mathbb{R}^{1000 \times 2}$ → only $4000$ parameters → **250× reduction**.

---

### 7. The $\alpha$ Scaling Hyperparameter

The actual update applied is:

$$
\Delta W = \frac{\alpha}{r} \cdot B A
$$

$\alpha$ controls how *much* the adapter output is weighted relative to the frozen base model. Think of it as a "volume knob" on the adapter signal.

- If $\alpha = r$: scaling = 1.0, standard contribution
- If $\alpha > r$: adapter dominates more — useful when rank is increased
- **Rule of thumb:** When you double $r$, scale $\alpha$ proportionally

---

### 8. Where LoRA Is Placed in the Transformer

The original LoRA paper added adapters only to the **Query ($Q$) and Value ($V$)** weight matrices in the self-attention blocks, keeping the Key, Output projections, and FFN layers frozen.

Later research found that applying LoRA to **all linear layers** (including FFN up/down projections) often improves performance. In practice, you tune this as a hyperparameter.

```
Transformer Decoder Block:
├── Self-Attention
│   ├── W_Q  ← LoRA here (original paper)
│   ├── W_K  ← sometimes LoRA
│   ├── W_V  ← LoRA here (original paper)
│   └── W_O  ← sometimes LoRA
└── FFN
    ├── W_up    ← sometimes LoRA (recent work)
    └── W_down  ← sometimes LoRA (recent work)
```

---

### 9. Adapter Merging at Inference: Zero Cost

After training, you can merge the adapter back into the base weights:

$$
W_{\text{merged}} = W + BA
$$

Once merged, there's no additional computation at inference — the model is just a regular model again. You pay the LoRA cost **only during training**, not at serving time.

But if you want to **hot-swap adapters** (switch between task-specific LoRAs at runtime without reloading the model), you keep $W$ and the adapter separate and add/subtract them dynamically.

---

## ⚠️ Edge Cases & Constraints

**LoRA learns less but forgets less** (Biderman et al., 2024): In code and math domains, LoRA achieves slightly lower peak performance than full fine-tuning, but preserves more of the base model's general capabilities. If you need maximum task accuracy and don't care about forgetting, full fine-tuning wins. If you need to retain general intelligence, LoRA is safer.

**Uniform rank is suboptimal:** Using the same $r$ for every layer ignores that different layers have different intrinsic dimensionalities of their update. Attention layers often need higher rank than FFN layers for certain tasks.

**Learning rate is different from full fine-tuning:** LoRA typically needs a **~10× higher learning rate** than full fine-tuning because gradients flow through fewer parameters and need to "do more work" per step.

**LoRA does not help with *forward pass* memory:** The base model still must fit on the GPU. LoRA only compresses the *training overhead* (gradients + optimizer states), not the model's inference footprint. Use quantization (QLoRA) for that.

**Rank-1 LoRA in RL:** Surprisingly, rank $r = 1$ has shown strong results in reinforcement learning fine-tuning contexts. The update distribution in RL may be even lower-rank than in SFT. Don't immediately dismiss tiny ranks.

---

## 💻 Logical Code Snippet (Python)

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# ──────────────────────────────────────────────
# 1. LoRA Layer — manual implementation to see the mechanics
# ──────────────────────────────────────────────

class LoRALinear(nn.Module):
    def __init__(self, base_weight: torch.Tensor, rank: int = 4, alpha: float = 8.0):
        super().__init__()
        d_out, d_in = base_weight.shape

        # Frozen base weight — requires_grad=False means no gradient stored,
        # no optimizer state allocated. Pure constant in the compute graph.
        self.W = nn.Parameter(base_weight, requires_grad=False)

        # Trainable adapter matrices — tiny!
        # A: random init (breaks symmetry), B: zero init (ΔW = 0 at start)
        self.A = nn.Parameter(torch.randn(rank, d_in) * 0.01)
        self.B = nn.Parameter(torch.zeros(d_out, rank))

        self.scale = alpha / rank  # volume knob on adapter signal

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # Base model output — W is frozen, this computes but grad doesn't flow into W
        base_out = F.linear(x, self.W)

        # Adapter output — grad DOES flow into A and B
        adapter_out = F.linear(F.linear(x, self.A), self.B) * self.scale

        return base_out + adapter_out  # ΔW(x) added on top


# ──────────────────────────────────────────────
# 2. Wrapping a real HuggingFace model with PEFT LoRA
# ──────────────────────────────────────────────

from peft import LoraConfig, get_peft_model
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained("meta-llama/Meta-Llama-3-8B")

config = LoraConfig(
    r=8,                           # rank: controls expressiveness vs. efficiency
    lora_alpha=16,                 # alpha: scale factor (usually 2r is a good start)
    target_modules=["q_proj", "v_proj"],  # where to inject adapters
    lora_dropout=0.05,             # optional regularization inside adapter path
    bias="none",                   # don't adapt bias terms
    task_type="CAUSAL_LM",
)

peft_model = get_peft_model(model, config)
peft_model.print_trainable_parameters()
# → trainable params: 4,194,304 || all params: 8,034,344,960 || trainable%: 0.0522

# Only 0.05% of parameters will have gradients. The rest are frozen constants.


# ──────────────────────────────────────────────
# 3. Why frozen means no memory for gradients — illustrated
# ──────────────────────────────────────────────

W = torch.randn(1000, 1000, requires_grad=False)   # frozen: no grad tensor
A = torch.randn(4, 1000,    requires_grad=True)    # trained: grad tensor allocated
B = torch.zeros(1000, 4,    requires_grad=True)    # trained: grad tensor allocated

x = torch.randn(32, 1000)  # batch of 32 inputs

out = x @ W.T + x @ A.T @ B.T  # forward pass

loss = out.mean()
loss.backward()

print(W.grad)  # None — no gradient computed or stored for W ✓
print(A.grad)  # tensor(...) — gradient exists, optimizer will update this ✓
print(B.grad)  # tensor(...) — gradient exists, optimizer will update this ✓


# ──────────────────────────────────────────────
# 4. Merging adapter back into base model (zero inference overhead)
# ──────────────────────────────────────────────

def merge_lora(W_frozen, A, B, scale):
    """After training: bake ΔW into W permanently. No adapter at inference."""
    delta_W = (B @ A) * scale          # reconstruct the rank-r update
    return W_frozen + delta_W          # single merged weight, no extra compute

W_merged = merge_lora(W, A, B, scale=2.0)
# Now just use W_merged in a standard forward pass — identical to before but specialized
```

---

## 🔬 Frontier Research & Papers

| Method | Core Idea | Key Result | Reference |
|---|---|---|---|
| **LoRA** (baseline) | Freeze $W$, train $\Delta W \approx BA$ with rank $r$ | Up to 250× parameter reduction with competitive accuracy | Hu et al., ICLR 2022 |
| **DoRA** | Decompose $W$ into *magnitude* + *direction*; apply LoRA only to direction; train magnitude separately | Closes the gap to full fine-tuning; +3.7% reasoning on LLaMA-7B over LoRA; more robust to rank choice; accepted ICML 2024 Oral (1.5% rate) | Liu et al., ICML 2024 |
| **QLoRA** | 4-bit quantized base model + LoRA adapters in fp16; introduces NF4 data type | Fine-tune a 65B model on a single 48GB GPU | Dettmers et al., NeurIPS 2023 |
| **GaLore** | Instead of constraining $\Delta W$ to low-rank, project the *gradients* to low-rank before Adam update; allows full-parameter updates with low-rank optimizer states | Enables memory-efficient pre-training (not just fine-tuning) of 7B models on 24GB | Zhao et al., ICML 2024 |
| **LoRA+** | Use different learning rates for $A$ and $B$ matrices (currently LoRA uses the same $\eta$ for both, which is suboptimal) | Better convergence; theoretically grounded | Hayou et al., 2024 |
| **rsLoRA** | Replace the $\alpha/r$ scaling with $\alpha/\sqrt{r}$ — more stable gradient magnitude as rank increases | High-rank LoRA no longer destabilizes training | Kalajdzievski, 2023 |
| **LoRA learns less and forgets less** | Empirical study comparing LoRA vs. full fine-tuning; LoRA is slightly weaker at peak task performance but significantly better at preserving base model capabilities | Key finding: for continual use cases, prefer LoRA; for single-task maximization, consider full FT | Biderman et al., 2024 |
| **LoRA-Pro** | Adjust gradients of $A$ and $B$ so the *effective* gradient on $W$ matches the true full fine-tuning gradient | Surpasses full fine-tuning on 3/5 GLUE tasks due to implicit regularization | ICLR 2025 |

---

## 🔬 DoRA Deep Dive (Since It Directly Extends Your LoRA Knowledge)

**The observation that motivates DoRA:** When you track how $W$ changes during full fine-tuning vs. LoRA fine-tuning, you find:

- **Full fine-tuning:** magnitude and direction of weight vectors change with a *negative correlation* — large directional shift → small magnitude change, and vice versa. The model is flexible.
- **LoRA:** magnitude and direction changes are coupled and entangled — they can't vary independently, which limits learning capacity.

**DoRA's solution:** Explicitly decompose $W$ into two learnable components:

$$
W = \underbrace{m}_{\text{magnitude (scalar per row)}} \cdot \underbrace{\frac{V}{\|V\|_c}}_{\text{direction (unit-normalized)}}
$$

Then:
- $m$ is a small learnable vector (1 scalar per output dimension) — trained directly
- $V$ is updated via LoRA (rank-$r$ approximation of directional change)

This adds only **~0.01% more parameters** over LoRA, yet consistently outperforms it — even when DoRA uses *half the rank* of LoRA.

```python
# DoRA forward (conceptual)
def dora_forward(x, W_frozen, A, B, m, scale):
    # Reconstruct updated directional matrix
    V_updated = W_frozen + (B @ A) * scale        # LoRA handles direction
    # Normalize each row to unit length (separates magnitude from direction)
    V_normed  = V_updated / V_updated.norm(dim=1, keepdim=True)
    # Apply learned magnitude scaling (m is a trainable vector, not frozen)
    W_eff     = m.unsqueeze(1) * V_normed
    return F.linear(x, W_eff)
```

In HuggingFace PEFT, DoRA is one argument away:
```python
config = LoraConfig(r=8, lora_alpha=16, use_dora=True, ...)
```

---

## ❓ Active Recall

- [ ] In your own words: why does LoRA freeze $W$ instead of training it? What property of fine-tuning weight updates makes this valid?
- [ ] Walk through the math: if $W \in \mathbb{R}^{4096 \times 4096}$ and $r = 8$, what are the shapes of $A$ and $B$? How many parameters does LoRA train vs. full fine-tuning?
- [ ] Why is $B$ initialized to zero and $A$ initialized randomly? What breaks if you swap them?
- [ ] In the forward pass $h = Wx + BAx$, which terms have `requires_grad=True`? Sketch what the compute graph looks like and where gradients stop.
- [ ] What does the $\alpha / r$ scaling factor actually control? What happens to the model output if you set $\alpha = 0$?
- [ ] Full fine-tuning needs ~4× model memory; what does LoRA reduce to, and why (concretely: what tensor is no longer allocated)?
- [ ] What is adapter merging, and when would you merge vs. keep adapters separate?
- [ ] Explain SVD and why it reveals $\Delta W$ is low-rank. What do singular values represent?
- [ ] What is the key empirical observation that motivates DoRA? What's the difference in how magnitude and direction change during LoRA vs. full fine-tuning?
- [ ] What does GaLore do differently from LoRA — and why does it enable pre-training rather than just fine-tuning?
- [ ] You're building a product that serves one base model with 50 different task-specific adapters. Argue why LoRA is architecturally better than keeping 50 full fine-tuned copies.
