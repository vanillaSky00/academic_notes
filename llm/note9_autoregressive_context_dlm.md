# 📄 Autoregressive Generation, Temperature, Context Windows & Beyond

**Tags:** #llm #autoregressive #temperature #context-window #diffusion-lm #streaming #interview
**Links:** [[Decoding Strategies]], [[Inference Optimization]], [[Encoder-Decoder Architecture]]

---

## 🎯 The "Elevator Pitch"
> LLMs generate text one token at a time — each new word depends on everything before it. Temperature controls how adventurous those choices are. The context window limits how much the model can remember. And newer approaches like diffusion language models challenge the entire one-token-at-a-time paradigm.

---

## 🧠 Core Mechanics

### 1. Autoregressive Generation (Q56, Q57)

**Definition:** A generation process where each output token is conditioned on all previously generated tokens (and the original input). The model's output at step `t` becomes part of the input at step `t+1`.

**Formally:**
```
P(y₁, y₂, ..., yₙ | x) = ∏ P(yₜ | y₁, ..., yₜ₋₁, x)
```
Each token is drawn from a conditional probability distribution over the vocabulary.

**Step-by-step:**
1. Feed prompt tokens → model predicts next token distribution → sample token `y₁`
2. Append `y₁` to context → model predicts next → sample `y₂`
3. Repeat until `<EOS>` token or max length

**Strengths:**
- ✅ Highly coherent text — each token is conditioned on the full accumulated context
- ✅ Flexible — works for any length output
- ✅ Simple training objective (predict next token = cross-entropy loss)

**Limitations:**
- ❌ **Inherently sequential:** Token `t` can't be generated until token `t-1` is done → no output-side parallelism → slow for long outputs
- ❌ **Error accumulation:** A wrong early token shifts the distribution for all future tokens (exposure bias)
- ❌ **Quadratic KV cache growth:** Each new token adds to the KV cache that all subsequent steps must read

**Analogy:** Like writing a novel by always continuing from the last sentence you wrote — coherent and natural, but you can't write chapter 5 until chapters 1–4 are done.

---

### 2. Temperature in Detail (Q55)

**What temperature does mechanically:**
```
probs = softmax(logits / T)
```
- Divides every logit by `T` before softmax.
- This **rescales the spread** of the distribution — it does not change which token is most likely, only by how much more likely it is than the others.

**Visual intuition:**

```
Original logits: [3.0, 1.5, 0.5, -0.5]

T = 1.0 (normal):    probs ≈ [0.59, 0.29, 0.10, 0.04]  ← moderate spread
T = 0.5 (low):       probs ≈ [0.87, 0.12, 0.01, 0.00]  ← very peaked (near-greedy)
T = 2.0 (high):      probs ≈ [0.39, 0.29, 0.21, 0.11]  ← much flatter (more random)
T → 0:               probs → [1.00, 0.00, 0.00, 0.00]  ← pure greedy
T → ∞:               probs → [0.25, 0.25, 0.25, 0.25]  ← uniform random
```

**Practical temperature ranges:**

| Temperature | Output Character | Typical Use Case |
|---|---|---|
| 0.0 | Deterministic (greedy) | Factual Q&A, code generation |
| 0.1–0.4 | Focused, conservative | Summarization, extraction |
| 0.7–0.9 | Balanced creativity | General chatbot, dialogue |
| 1.0 | Neutral (model's raw distribution) | Baseline |
| 1.2–1.5 | Creative, experimental | Brainstorming, fiction |
| > 1.5 | Often incoherent | Rarely useful |

**Key insight:** Temperature doesn't add "intelligence" — it adjusts how much the model trusts its own high-confidence predictions vs. exploring less likely options.

---

### 3. Context Window Deep Dive (Q53, Q54)

**Definition:** The maximum total number of tokens (prompt + generated output) the model can attend to in a single forward pass.

**Why it's limited:**
- Self-attention is O(n²) in memory — doubling context length quadruples attention memory.
- The KV cache also grows linearly with context length — very long contexts exhaust GPU RAM.

**What happens at the boundary:**
- Tokens exceeding the window are **silently truncated** — the model cannot see them.
- This is not an error message — the model simply acts as if those tokens never existed.
- In multi-turn conversations, this means early turns are forgotten first (left-truncation is common).

**Context window evolution:**

| Model | Context Window |
|---|---|
| GPT-2 (2019) | 1,024 tokens |
| GPT-3 (2020) | 4,096 tokens |
| GPT-4 (2023) | 8K → 32K → 128K tokens |
| Gemini 1.5 (2024) | 1M tokens |
| Claude 3 (2024) | 200K tokens |

**Mitigations for context limits:**
- **RAG (Retrieval-Augmented Generation):** Retrieve only relevant chunks from a large document corpus and inject them into the context — effectively extending the "effective memory."
- **Context compression:** Summarize or compress earlier turns before they fall off the window.
- **Sliding window attention:** Only attend to a local window of recent tokens at each layer (Longformer).

**Large vs. Small context window trade-offs:**

| | Large Window | Small Window |
|---|---|---|
| Use case | Long documents, complex reasoning | Simple Q&A, real-time apps |
| Memory | ❌ Very high (KV cache) | ✅ Low |
| Latency | ❌ Higher (more tokens to process) | ✅ Lower |
| Attention cost | ❌ O(n²) expensive | ✅ Manageable |
| "Memory" capacity | ✅ Retains more history | ❌ Forgets early content |

---

### 4. Diffusion Language Models (DLMs) vs. LLMs (Q58, Q59)

**How LLMs generate (autoregressive):**
- Left to right, one token at a time.
- Fast per-step, but slow for long outputs (sequential bottleneck).

**How DLMs generate (denoising):**
1. Start with a sequence of pure random noise tokens.
2. Iteratively **denoise** the entire sequence over `T` steps.
3. At each step, the model predicts a cleaner version of the whole sequence simultaneously.
4. After `T` steps, coherent text emerges.

**Key comparison:**

| | LLM (Autoregressive) | DLM (Diffusion) |
|---|---|---|
| Generation direction | Left to right, sequential | Entire sequence in parallel |
| Context modeling | Causal (past only) | Global (all positions at once) |
| Single step cost | Cheap | Heavier (full sequence) |
| Total steps | n (one per output token) | T (fixed denoising steps, T << n) |
| TPOT (latency) | Slow for long outputs | ✅ Faster for bulk/long outputs |
| Output quality (2024) | ✅ State of the art | Still catching up |
| Controllability | Moderate | ✅ Potentially higher |

**For latency-sensitive applications (Q59):**
- Short outputs (<50 tokens): LLMs are competitive or better (fewer steps overall).
- Long outputs (500+ tokens): DLMs can win on TPOT because they generate many tokens per denoising step.
- **Current state (2024–2025):** DLMs are rapidly improving but LLMs still dominate production use cases.

**Analogy:** LLMs are like a sculptor carving a statue from a block of marble, chip by chip, left to right. DLMs are like a sculptor who starts with a blob of clay representing the whole figure, and refines the entire shape simultaneously with each pass.

---

### 5. Token Streaming (Q60)

**What it is:** Sending each generated token to the user *as soon as it's produced*, rather than waiting for the full response to complete.

**Why it dramatically improves perceived latency:**
- Without streaming: User waits ~10s → receives full 500-word response at once.
- With streaming: User starts reading after ~0.1s (first token arrives) → response feels instant and "live."

**Technical mechanism:**
- Server uses Server-Sent Events (SSE) or WebSockets to push tokens incrementally.
- The model's computation is unchanged — streaming only affects *when* the client receives each token.

**Analogy:** The difference between waiting for a full pizza delivery and watching a chef prepare your food in front of you at a restaurant — same end result, very different experience.

**Implementation concept:**
```python
# Streaming with Anthropic SDK (conceptual)
with client.messages.stream(model="claude-...", messages=[...]) as stream:
    for text_chunk in stream.text_stream:
        print(text_chunk, end="", flush=True)  # display each token immediately
```

---

## ⚠️ Edge Cases & Constraints
- **Exposure bias in autoregressive models:** During training, the model sees gold (correct) tokens at each step. At inference, it sees its own (potentially wrong) predictions. This mismatch can compound errors — models trained with techniques like scheduled sampling mitigate this.
- **Temperature is not the only randomness control:** Top-p and Top-k also control diversity independently. Using temperature alone without Top-p/k can still produce low-probability tokens.
- **Context window ≠ memory:** The model doesn't "remember" like a human. Within the context window it has perfect recall; outside it, it has zero recall. There is no gradual forgetting — it's a hard cutoff.
- **DLMs in text vs. images:** Diffusion is dominant in image generation (Stable Diffusion, DALL-E 3) but still maturing for text. The discrete nature of tokens makes denoising harder than continuous pixel space.
- **Streaming and tool use:** Streaming complicates agentic pipelines — if the model calls a tool mid-generation, streaming must pause until the tool result arrives.

---

## 💻 Logical Code Snippet (Python)

```python
import torch
import torch.nn.functional as F

# ── Autoregressive Generation Loop ───────────────────────────────────
def autoregressive_generate(model, tokenizer, prompt: str, max_tokens=200, temperature=0.8, top_p=0.9):
    input_ids = tokenizer.encode(prompt, return_tensors="pt")
    generated = input_ids.clone()

    for _ in range(max_tokens):
        with torch.no_grad():
            logits = model(generated).logits[:, -1, :]  # only last position's logits

        # Temperature scaling
        logits = logits / temperature

        # Top-p (nucleus) filtering
        sorted_logits, sorted_idx = torch.sort(logits, descending=True)
        cumulative_probs = torch.cumsum(F.softmax(sorted_logits, dim=-1), dim=-1)
        remove_mask = cumulative_probs > top_p
        remove_mask[..., 1:] = remove_mask[..., :-1].clone()
        remove_mask[..., 0] = False
        sorted_logits[remove_mask] = float('-inf')
        logits = sorted_logits.scatter(-1, sorted_idx, sorted_logits)

        # Sample next token
        probs = F.softmax(logits, dim=-1)
        next_token = torch.multinomial(probs, num_samples=1)

        # Append and check for EOS
        generated = torch.cat([generated, next_token], dim=-1)
        if next_token.item() == tokenizer.eos_token_id:
            break

        # --- Token streaming: yield here in a real implementation ---
        yield tokenizer.decode(next_token[0])

    return tokenizer.decode(generated[0], skip_special_tokens=True)
```

---

## ❓ Active Recall
- [ ] What is autoregressive generation? Write out the joint probability formula.
- [ ] What are the two main weaknesses of autoregressive generation?
- [ ] What does temperature do mechanically to the logits?
- [ ] What temperature would you use for: (a) factual Q&A, (b) creative story writing?
- [ ] What happens when an input exceeds the context window? Is there a warning?
- [ ] What is the difference between context window size and model "memory"?
- [ ] What are two techniques for handling content that exceeds the context window?
- [ ] How do DLMs differ from LLMs in the generation process?
- [ ] For long-form generation, why might DLMs have a latency advantage?
- [ ] What is token streaming and how does it affect perceived vs. actual latency?
- [ ] **Follow-up:** What is "exposure bias" in autoregressive models? How does scheduled sampling address it?
- [ ] **Follow-up:** What is MDLM or Plaid — recent discrete diffusion language models?
- [ ] **Follow-up:** How does RAG extend effective context without expanding the context window?
