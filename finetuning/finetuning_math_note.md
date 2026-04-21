# 📄 Fine-Tuning Math: Loss, Gradients & Weight Updates

**Tags:** #deep-learning #fine-tuning #LLM #optimization #cross-entropy #backprop  
**Links:** [[Supervised Learning]], [[Cross-Entropy Loss]], [[Backpropagation]], [[AdamW]], [[KL Divergence]], [[Teacher Forcing]], [[LoRA]]

---

## 🎯 The "Elevator Pitch"
> Fine-tuning is gradient descent on a pre-trained model, where you compute cross-entropy loss **only on the target output tokens**, then backpropagate to nudge the entire weight distribution toward generating the correct completions — not just ranking them, but assigning *well-calibrated probability mass* to them.

---

## 🧠 Core Mechanics

### 1. The Training Loop at a Glance
The fine-tuning loop is identical to pre-training in structure but differs in **what data you feed** and **where you compute loss**.

```
for each epoch:
    for each batch of (input, target_output) pairs:
        1. Forward pass  →  predicted token distribution
        2. Compute loss  →  cross-entropy on target tokens ONLY (loss mask on input)
        3. Backward pass →  gradients via backprop
        4. Weight update →  optimizer step (AdamW in practice)
```

Key difference from scratch training: you **start from a pre-trained checkpoint**, so the model is already reasonable — fine-tuning just steers it.

---

### 2. Loss: Cross-Entropy / Negative Log-Likelihood

**Intuition:** You don't just want the model to *rank* Sacramento above SF. You want it to assign **high probability mass** to Sacramento. Cross-entropy punishes low confidence on the correct token even if it's still top-ranked.

$$
\mathcal{L}_{\text{CE}} = -\log p_\theta(\text{Sacramento})
$$

- If $p = 0.12$: $\mathcal{L} \approx 2.12$ (bad)
- If $p = 0.68$: $\mathcal{L} \approx 0.39$ (much better)

**Why log?** Multiplying probabilities across a long sequence creates numerical underflow ($0.3 \times 0.7 \times 0.2 \times \ldots \to 0$). The log turns products into sums, keeping values numerically stable and making optimization easier.

$$
\log \prod_t p_t = \sum_t \log p_t
$$

**Information-theoretic view (the deeper why):**

- **Entropy $H(p)$**: minimum bits needed to encode tokens from the true distribution $p$ — a theoretical lower bound.
- **Cross-entropy $H(p, q)$**: actual bits consumed when you encode $p$-distributed events using your model $q$. Always $\geq H(p)$.
- **KL divergence $D_{KL}(p \| q) = H(p,q) - H(p)$**: the *wasted* bits — how many extra bits you're burning because your model $q$ disagrees with reality $p$. KL is **not symmetric** and is **not a distance metric**.

Minimizing cross-entropy $\equiv$ minimizing KL divergence from model $q$ to true $p$ (since $H(p)$ is fixed for the data).

> **Insight:** The log-probability isn't just a math convenience — it's literally the optimal code length for that token under your current model. Fine-tuning is compression.

---

### 3. Loss Mask & Teacher Forcing

**Loss mask:** During SFT, loss is computed **only on completion tokens**, not on the prompt/input. The `completion_only_loss=True` flag in HuggingFace SFTTrainer implements this. Computing loss on inputs would punish the model for not "predicting" the fixed prompt, which makes no sense.

**Teacher forcing:** Even if the model predicted the *wrong* token at step $t$, you still feed the *correct* token as input for step $t+1$. This enables:
- **Parallelization**: the entire target sequence is known ahead of time, so all token predictions can be computed in one forward pass (one big matrix multiply).
- **Stability**: without teacher forcing, errors compound over long sequences during training.

The tradeoff: a **train-test gap** — at inference, the model conditions on its own (possibly wrong) predictions. This gap is called *exposure bias*.

---

### 4. Backpropagation & Gradients

Backprop computes $\frac{\partial \mathcal{L}}{\partial w}$ for every weight $w$ via the chain rule, moving backward from the output layer to the first layer.

For cross-entropy, the gradient at the output layer is clean:

$$
\frac{\partial \mathcal{L}}{\partial \text{logit}_i} = p_i - y_i
$$

where $y_i = 1$ for the correct token, 0 otherwise. So:
- Correct token (Sacramento): gradient is **negative** → push logit up
- Wrong tokens (SF, LA, …): gradient is **positive** → push logit down

The gradient w.r.t. a weight matrix row $W_i$ connecting hidden state $h$ to output $i$:

$$
\frac{\partial \mathcal{L}}{\partial W_i} = (p_i - y_i) \cdot h
$$

The hidden state $h$ **scales** the gradient — tokens whose hidden representations are already large will receive proportionally larger updates.

**Mountain analogy:** Loss = altitude. Weights = your $(x,y)$ position. Gradient = slope/steepness. Gradient descent = take a step downhill.

---

### 5. Optimizers: From SGD to AdamW

**Vanilla SGD:**
$$
w \leftarrow w - \eta \cdot \nabla_w \mathcal{L}
$$

Problem: raw gradients from a single example are noisy and can be either huge or tiny.

**Adam / AdamW** (what LLMs actually use):
- Tracks a **momentum** term (exponential moving average of gradients) to smooth updates
- Tracks a **second moment** (moving average of squared gradients) to adaptively scale learning rate per parameter
- AdamW adds **decoupled weight decay** — L2 regularization is applied directly to weights, not folded into the gradient, which is empirically better

**Why momentum matters for transformers:** Attention weights and FFN weights have very different gradient scales. Adam's per-parameter learning rate rescaling handles this heterogeneity automatically.

---

### 6. Weight Update: What Actually Changes

In a model with billions of parameters, the update runs in parallel across all weights using **gradients averaged over the batch**. A batch average prevents the model from overfitting to a single example.

After one step, the full weight distribution shifts slightly — the histogram of all weights nudges in the direction that increases probability mass on correct tokens. A single update is small by design (controlled by $\eta$).

---

## ⚠️ Edge Cases & Constraints

**Catastrophic forgetting:** SFT on domain-specific data can cause the model to lose general capabilities. The gradient updates that help it specialize also damage neurons encoding general knowledge. Mitigation: mix domain data with general data, use LoRA (freeze most weights, only update small adapters), or regularize with proximal policy constraints.

**Exposure bias (teacher forcing gap):** Training with teacher forcing means the model never learns to recover from its own errors. At inference it compounds mistakes. Scheduled sampling (gradually replacing ground-truth tokens with model predictions during training) can help but complicates parallelization.

**Cross-entropy and output diversity collapse:** CE maximizes likelihood of observed outputs, but does *not* reward diversity. The model can degenerate into outputting highly repetitive high-probability sequences. Recent work (GEM, 2024) reformulates SFT as reverse KL minimization with entropy regularization to preserve diversity.

**Label smoothing:** Hard one-hot targets make the model overconfident. Label smoothing replaces the `[1, 0, 0, ...]` target with `[1-ε, ε/(V-1), ...]`, preventing probability mass from collapsing onto a single token. Calibrated Language Models (ICML 2025) shows this interacts with model size — large hidden dimensions can create a lower bound on output entropy that negates smoothing's benefits.

**Memory footprint of cross-entropy:** Computing CE over a vocabulary of 128K+ tokens requires materializing a huge logit matrix ($N \times |V|$). This becomes the dominant memory cost at large vocab sizes and requires fused kernel implementations (e.g., chunked cross-entropy) to handle.

---

## 💻 Logical Code Snippet (Python)

```python
import torch
import torch.nn.functional as F

# ----- Anatomy of one fine-tuning step -----

def sft_step(model, optimizer, input_ids, target_ids, loss_mask):
    """
    input_ids  : (batch, seq_len) — full sequence including prompt
    target_ids : (batch, seq_len) — same sequence, shifted by 1 for next-token prediction
    loss_mask  : (batch, seq_len) — 1 for completion tokens, 0 for prompt tokens
    """

    # 1. Forward pass — one matrix multiply covers the whole sequence (teacher forcing)
    logits = model(input_ids)          # (batch, seq_len, vocab_size)

    # 2. Cross-entropy loss — flat batch, then mask out input positions
    batch, seq_len, vocab = logits.shape
    loss_per_token = F.cross_entropy(
        logits.view(-1, vocab),        # (batch*seq_len, vocab)
        target_ids.view(-1),           # (batch*seq_len,)
        reduction='none'
    ).view(batch, seq_len)             # (batch, seq_len)

    # 3. Apply loss mask: only penalize completion tokens
    masked_loss = (loss_per_token * loss_mask).sum() / loss_mask.sum()

    # 4. Backprop
    optimizer.zero_grad()
    masked_loss.backward()             # chain rule backward through entire model

    # 5. Weight update (AdamW handles momentum + adaptive lr internally)
    optimizer.step()

    return masked_loss.item()


# ----- Cross-entropy from information theory perspective -----

def cross_entropy_bits(p_true, q_model):
    """
    H(p, q) = -sum(p * log2(q))
    KL(p||q) = H(p,q) - H(p)   [extra bits wasted due to wrong model]
    """
    import numpy as np
    p = np.array(p_true)
    q = np.array(q_model)
    H_p  = -np.sum(p * np.log2(p + 1e-12))           # entropy (lower bound)
    H_pq = -np.sum(p * np.log2(q + 1e-12))           # cross-entropy (actual cost)
    KL   = H_pq - H_p                                 # redundant bits
    return {"H(p)": H_p, "H(p,q)": H_pq, "KL(p||q)": KL}

# Example: p = true distribution, q = model distribution
print(cross_entropy_bits([0, 1, 0, 0], [0.12, 0.6, 0.18, 0.1]))
# {'H(p)': 0.0, 'H(p,q)': 2.07, 'KL(p||q)': 2.07}
# Model wastes 2.07 bits because it didn't put mass on the right token
```

---

## 🔬 Frontier Research & Papers

| Topic | Key Insight | Reference |
|---|---|---|
| **GEM (Game-theoretic SFT)** | Standard CE reduces output diversity. GEM reformulates SFT as reverse KL + entropy regularization, achieving 5-8pt gains in downstream tasks when diversity is needed for test-time scaling. | *Preserving Diversity in SFT*, ICLR 2025 |
| **Label Smoothing for LLMs** | LLMs can be over-confident; label smoothing helps but interacts with model size. Large hidden dimensions impose an entropy lower bound that can negate smoothing. Fused kernels needed for memory. | *Calibrated LMs*, ICML 2025 |
| **Proximal SFT (PSFT)** | PPO-style per-token probability ratio clipping added to the SFT objective prevents entropy collapse and policy drift during extended fine-tuning. | Zhu et al., 2025 |
| **RobustFT** | Multi-expert consistency + entropy-based filtering removes noisy labels from SFT data, improving accuracy under high-noise conditions. | Luo et al., 2024 |
| **FisherSFT** | Selects training subsets by maximizing the log-determinant of the Fisher information matrix — statistically principled data selection for SFT. | Deb et al., 2025 |
| **Perplexity-based filtering** | Pre-SFT model perplexity on training examples predicts downstream gain better than embedding similarity or response length. | Harada et al., 2025 |

---

## ❓ Active Recall

- [ ] Why is cross-entropy equivalent to minimizing KL divergence from the model to the data distribution?
- [ ] What is the difference between $H(p)$, $H(p,q)$, and $D_{KL}(p \| q)$ — and why is KL not a distance?
- [ ] Why do we apply a **loss mask** on the prompt tokens during SFT? What would happen if we didn't?
- [ ] What is **teacher forcing** and how does it enable parallelized training? What is its main downside?
- [ ] Derive the gradient of cross-entropy loss at the output logit layer. Why do correct tokens get a negative gradient and wrong tokens a positive one?
- [ ] Why is Adam better than SGD for transformer fine-tuning? What two statistics does Adam maintain per parameter?
- [ ] What is **catastrophic forgetting** in SFT, and name two strategies to mitigate it?
- [ ] Why does standard cross-entropy SFT cause **output diversity collapse**, and how does GEM address this?
- [ ] Why does label smoothing sometimes fail for large LLMs, according to the ICML 2025 calibration paper?
