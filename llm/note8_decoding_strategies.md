# 📄 LLM Decoding Strategies

**Tags:** #llm #decoding #greedy-search #beam-search #sampling #temperature #interview
**Links:** [[Autoregressive Generation & Context Window]], [[Inference Optimization]], [[Softmax]]

---

## 🎯 The "Elevator Pitch"
> Every time an LLM generates a word, it has a probability distribution over its entire vocabulary. A decoding strategy is the decision rule that picks which word to actually output — choose the wrong strategy and your model produces boring repetition or incoherent nonsense.

---

## 🧠 Core Mechanics

### 1. What Decoding Strategies Are (Q42)

At each generation step, the model outputs a vector of **logits** (raw scores) over all `vocab_size` tokens. These are converted to probabilities via softmax. The decoding strategy decides how to **select the next token** from this distribution.

**The core dial: quality ↔ diversity ↔ speed**

---

### 2. Strategy Overview Table (Q43, Q51, Q52)

| Strategy | Type | Token Selection | Best For | Weakness |
|---|---|---|---|---|
| **Greedy Search** | Deterministic | argmax (highest prob) | Speed, factual tasks | Repetitive, locally optimal only |
| **Beam Search** | Deterministic | Top-k beams tracked jointly | Translation, summarization | Slow, conservative |
| **Temperature Sampling** | Stochastic | Sample from scaled distribution | Creative writing, diversity | Can be incoherent at high T |
| **Top-k Sampling** | Stochastic | Sample from top-k tokens | General creative tasks | Fixed k can be too wide or narrow |
| **Top-p (Nucleus) Sampling** | Stochastic | Sample from smallest set summing to p | Best balance of quality + diversity | Slightly more compute per step |
| **Speculative Decoding** | Hybrid | Draft model proposes; main model verifies | Low-latency serving | Requires a matched draft model |

---

### 3. Greedy Search (Q45, Q49)

**Mechanism:** At each step, pick the single token with the highest probability.
```
next_token = argmax( softmax(logits) )
```

**Temperature = 0.0 is greedy decoding** — scaling logits toward 0 makes the softmax distribution infinitely peaked at the maximum, collapsing to argmax.

**Pros:**
- ✅ Fastest possible decoding
- ✅ Deterministic — same input always produces same output

**Cons:**
- ❌ **Shortsighted:** Makes the locally best choice at each step, which may not lead to the globally best sequence.
- ❌ Prone to repetitive loops ("The cat sat on the cat sat on the...")
- ❌ Misses high-quality sequences that start with a slightly lower-probability token

**Analogy:** Navigating a mountain range by always walking uphill — you'll reach a local peak quickly, but it may not be the highest mountain.

---

### 4. Beam Search (Q46, Q47, Q48, Q50)

**Mechanism:** Instead of tracking 1 candidate, track the **top-k candidates (beams)** at every step. At the end, output the sequence with the highest overall probability.

```
Beam width k=3:
Step 1: ["The", "A", "This"]           ← top-3 first tokens
Step 2: ["The cat", "The dog", "A cat"] ← extend each beam, keep top-3
Step 3: ["The cat sat", ...]            ← keep expanding...
Final:  pick beam with highest total log-prob
```

**Beam width parameter:**
- Larger `k` → more candidates → better quality → slower and more memory
- `k=1` → greedy search

**Beam Search vs BFS/DFS (Q50):**
- BFS explores *all* nodes at each level → exponential memory
- DFS explores one path fully → may miss optimal paths
- Beam Search keeps only **top-k at each level** → heuristic pruning; not guaranteed optimal, but practically efficient

**When to prefer Beam Search (Q47):**
- Machine translation, summarization, code generation
- When **reproducibility** and **factual accuracy** matter over diversity
- Legal/medical document generation (deterministic, reliable)

**Greedy vs Beam trade-off (Q48):**

| | Greedy | Beam (k=5) |
|---|---|---|
| Memory | O(1) | O(k) |
| Speed | ✅ Fastest | ❌ k× slower |
| Output quality | Suboptimal | Better globally |
| Deterministic | ✅ | ✅ |

---

### 5. Stochastic Sampling Methods (Q43, Q52)

#### Temperature Sampling (Q55 preview)
- Divide logits by temperature `T` before softmax:
  ```
  probs = softmax(logits / T)
  next_token = sample(probs)
  ```
- **T > 1:** Flattens distribution → more randomness, more diversity
- **T < 1:** Sharpens distribution → more deterministic, less diverse
- **T → 0:** Approaches greedy decoding

#### Top-k Sampling
- Keep only the `k` highest-probability tokens; zero out the rest; renormalize; sample.
- Prevents sampling very unlikely tokens that would be incoherent.
- **Problem:** Fixed `k` doesn't adapt to the distribution shape — if the model is very confident, `k=50` still allows bad tokens; if uncertain, `k=5` may be too restrictive.

#### Top-p (Nucleus) Sampling
- Keep the *smallest set* of tokens whose cumulative probability ≥ `p` (e.g., 0.9).
- Dynamically adjusts the candidate set based on how spread out the distribution is.
- If the model is confident → small nucleus (few tokens). If uncertain → large nucleus.
- Generally preferred over Top-k in practice.

**Visualizing the difference:**
```
Vocabulary probabilities (sorted): [0.5, 0.3, 0.1, 0.05, 0.03, 0.02, ...]

Top-k (k=3): always sample from top 3 → [0.5, 0.3, 0.1]
Top-p (p=0.9): sample from tokens summing to 0.9 → [0.5, 0.3, 0.1] = 0.9 ✓ (same here)
              But if distribution is flat: [0.2, 0.15, 0.12, 0.11, 0.1, 0.09, 0.08, 0.07, 0.08]
              Top-k still takes 3; Top-p (0.9) takes all 9 — adapts correctly
```

---

### 6. Speculative Decoding (Q43, Q44)

**The problem it solves:** LLM decoding is slow because each token requires a full forward pass through a large model.

**Mechanism:**
1. A small, fast **draft model** generates `k` candidate tokens quickly.
2. The large **verifier model** checks all `k` tokens in *parallel* (one forward pass).
3. Accept all tokens the verifier agrees with; reject and resample from the first disagreement.
4. Net result: multiple tokens accepted per verifier forward pass.

**Why it's efficient:** The verifier's forward pass over `k` tokens is barely more expensive than over 1 token (due to parallelism) — but you accept multiple tokens if the draft is good.

**Quality guarantee:** Output distribution is identical to running the large model alone — no quality loss.

---

### 7. Choosing the Right Strategy (Q51, Q44)

```
Is output quality/accuracy critical?
  ├── Yes, and it must be reproducible → Beam Search (or Greedy for speed)
  │     Examples: translation, summarization, medical/legal docs, code gen
  └── Yes, but creativity/diversity is also needed → Top-p or Top-k Sampling
        Examples: chatbots, story writing, dialogue systems

Is latency critical?
  ├── Extreme latency constraint → Greedy Search
  └── Can use a draft model → Speculative Decoding

Is this an offline batch job?
  └── Beam Search for quality, Greedy for speed
```

---

## ⚠️ Edge Cases & Constraints
- **Beam Search can be "too safe":** It often produces generic, bland text because the most probable sequences tend to be common phrases. This is why creative tasks prefer sampling.
- **Repetition penalty:** Often added on top of any strategy to penalize tokens already generated — helps Greedy/Beam avoid loops.
- **Top-p and Top-k are often combined:** `top_k=50, top_p=0.9` is a common default in practice (filter by k first, then by p).
- **Speculative decoding requires draft/main model alignment:** If the draft model is too different in quality from the main model, acceptance rate drops and it offers no speedup.
- **Common misconception:** Higher temperature ≠ better quality. Very high temperatures (>1.5) produce nearly random text. Good creative outputs typically use T ∈ [0.7, 1.0].

---

## 💻 Logical Code Snippet (Python)

```python
import torch
import torch.nn.functional as F

def greedy_decode(logits):
    """Always picks the highest probability token."""
    return logits.argmax(dim=-1)

def temperature_sample(logits, temperature=1.0):
    """Scale logits by temperature, then sample."""
    scaled = logits / max(temperature, 1e-8)  # avoid division by zero
    probs = F.softmax(scaled, dim=-1)
    return torch.multinomial(probs, num_samples=1).squeeze(-1)

def top_k_sample(logits, k=50):
    """Zero out all but top-k tokens, then sample."""
    top_k_values, _ = torch.topk(logits, k)
    threshold = top_k_values[..., -1, None]  # k-th largest value
    filtered = logits.masked_fill(logits < threshold, float('-inf'))
    probs = F.softmax(filtered, dim=-1)
    return torch.multinomial(probs, num_samples=1).squeeze(-1)

def top_p_sample(logits, p=0.9):
    """Keep smallest set of tokens whose cumulative probability >= p."""
    sorted_logits, sorted_indices = torch.sort(logits, descending=True)
    cumulative_probs = torch.cumsum(F.softmax(sorted_logits, dim=-1), dim=-1)
    # Remove tokens once cumulative prob exceeds p
    sorted_indices_to_remove = cumulative_probs > p
    sorted_indices_to_remove[..., 1:] = sorted_indices_to_remove[..., :-1].clone()
    sorted_indices_to_remove[..., 0] = 0  # always keep top token
    indices_to_remove = sorted_indices_to_remove.scatter(-1, sorted_indices, sorted_indices_to_remove)
    filtered = logits.masked_fill(indices_to_remove, float('-inf'))
    probs = F.softmax(filtered, dim=-1)
    return torch.multinomial(probs, num_samples=1).squeeze(-1)
```

---

## ❓ Active Recall
- [ ] What is a decoding strategy, and why is it necessary?
- [ ] What does setting temperature = 0 do, and which strategy does it correspond to?
- [ ] What is Greedy Search's main failure mode? Give a concrete example.
- [ ] How does Beam Search differ from Greedy Search? What is the beam width parameter?
- [ ] How is Beam Search different from BFS and DFS?
- [ ] When would you choose Beam Search over Top-p sampling? Give a use case for each.
- [ ] What is the difference between Top-k and Top-p sampling? Which adapts better to distribution shape?
- [ ] How does speculative decoding work? What is the quality guarantee?
- [ ] What does temperature `T > 1` do to the probability distribution? What about `T < 1`?
- [ ] **Follow-up:** What is "repetition penalty" and how is it applied?
- [ ] **Follow-up:** In Beam Search, what is "length normalization" and why is it important?
- [ ] **Follow-up:** Why does Beam Search tend to produce generic text for open-ended tasks?
