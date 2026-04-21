# 📄 LLM Inference Optimization: KV Cache, Quantization & Memory Management

**Tags:** #llm #inference #kv-cache #quantization #paged-attention #interview
**Links:** [[Self-Attention Mechanics]], [[Encoder-Decoder Architecture]], [[Training Stability]]

---

## 🎯 The "Elevator Pitch"
> Running a large language model at inference is expensive — it needs to compute attention over all previous tokens for every new word it generates. KV caching is the key trick that avoids re-doing this work; quantization shrinks the model to fit in memory; and PagedAttention manages that memory efficiently at scale.

---

## 🧠 Core Mechanics

### 1. The Full LLM Inference Pipeline (Q2)

**End-to-end steps when you send a query to an LLM:**

| Phase | What Happens |
|---|---|
| **1. Tokenization** | Raw text → token IDs via the model's tokenizer |
| **2. Prefill (Prompt Processing)** | Embedding lookup + all decoder layers process the *entire* prompt **in parallel** → computes and caches Keys and Values for all prompt tokens |
| **3. Decoding (Autoregressive Generation)** | Model generates output **one token at a time**; each step uses cached K/V from previous tokens + computes new K/V for the just-generated token |
| **4. Detokenization** | Output token IDs → human-readable text |

- **Why prefill is fast:** The prompt can be processed in parallel (like an encoder).
- **Why decoding is slow:** Each token requires a forward pass; generation is inherently sequential.

---

### 2. KV Cache: The Core Inference Trick (Q3)

**The problem without KV cache:**
- At each decoding step `t`, self-attention needs K and V from *all* previous tokens (1 to t-1).
- Recomputing K and V from scratch at every step → **O(t)** redundant computations per step → **O(t²)** total for the entire sequence.

**What KV cache does:**
- After computing K and V for token `t`, **store them in GPU memory**.
- At step `t+1`, retrieve cached K/V for tokens 1 to t, compute only the new token's K/V, and concatenate.
- Only the **Query for the current token** needs to be freshly computed.

**Analogy:** Imagine translating a book page by page. Without caching, you'd reread the entire book up to the current page before writing each sentence. With caching, you keep notes (K/V) from everything you've already read.

**Trade-off:**
- ✅ Several-fold speedup in token generation
- ❌ KV cache size grows **linearly** with sequence length and batch size → major memory bottleneck for long contexts or high-throughput serving

---

### 3. Handling Large KV Cache Memory (Q5)

**Techniques to manage KV cache memory:**

| Technique | Mechanism | Trade-off |
|---|---|---|
| **PagedAttention** | Divides KV cache into fixed-size "pages" (like OS virtual memory); allocates on demand | Reduces fragmentation; enables large batch sizes |
| **Grouped-Query Attention (GQA)** | Multiple query heads share a single K/V head group | Reduces K/V cache size; slight quality trade-off |
| **Multi-Query Attention (MQA)** | All query heads share *one* K/V head | Maximum cache reduction; more quality trade-off |
| **KV Cache Quantization** | Store K/V as INT8 instead of FP16 | 2× memory saving; minor accuracy impact |
| **KV Cache Offloading** | Move inactive cache pages to CPU RAM | Allows very long contexts; adds latency for swapping |

**GQA vs MQA vs MHA summary:**

```
MHA (Multi-Head):   Q1 K1 V1 | Q2 K2 V2 | Q3 K3 V3 | Q4 K4 V4   (4 KV pairs)
GQA (Grouped):      Q1 Q2 K1 V1 | Q3 Q4 K2 V2                    (2 KV pairs)
MQA (Multi-Query):  Q1 Q2 Q3 Q4 K1 V1                             (1 KV pair)
```

---

### 4. Quantization (Q4)

**What it is:** Reducing the numerical precision of model weights (and optionally activations and KV cache) from high-precision floats to lower-bit integers.

| Precision | Bits | Bytes per param | Relative size |
|---|---|---|---|
| FP32 (full) | 32 | 4 bytes | 1× (baseline) |
| FP16 / BF16 | 16 | 2 bytes | 0.5× |
| INT8 | 8 | 1 byte | 0.25× |
| INT4 | 4 | 0.5 bytes | 0.125× |

**Benefits:**
- ✅ Memory footprint reduced by 50–87.5%
- ✅ Faster inference — lower-precision ops are faster on modern hardware (especially INT8 tensor cores)
- ✅ Enables deployment on smaller GPUs or even consumer hardware

**Costs:**
- ❌ Small accuracy degradation — lower precision = more rounding error
- ❌ Requires calibration (PTQ) or retraining (QAT) to minimize quality loss

**Types:**
- **Post-Training Quantization (PTQ):** Quantize after training — fast, no retraining needed, some quality loss
- **Quantization-Aware Training (QAT):** Simulate quantization *during* training — better quality, expensive

---

### 5. Computational Complexity of Self-Attention (Q10, Q23)

- **Self-attention complexity:** `O(n² · d)` where `n` = sequence length, `d` = model dimension
- The `n²` term comes from computing attention scores between all `n × n` token pairs
- For long sequences (n = 100K), this becomes prohibitively expensive

**Comparison with RNNs:**
| | Self-Attention | RNN |
|---|---|---|
| Complexity | O(n² · d) | O(n · d²) |
| Parallelism | ✅ Full (all tokens at once) | ❌ Sequential |
| Long-range dependencies | ✅ Direct 1-hop connection | ❌ Fades over distance |

- Self-attention is *slower* than RNNs per parameter for long sequences but *faster* in practice due to GPU parallelism.

**Efficient attention alternatives for long contexts:**
- FlashAttention (IO-aware exact attention)
- Linear attention approximations
- Sliding window attention (Longformer)
- Sparse attention

---

## ⚠️ Edge Cases & Constraints
- **KV cache invalidation:** If you change the prefix (e.g., system prompt changes), the entire cache must be recomputed. Prefix caching architectures mitigate this.
- **Quantization sensitivity varies by layer:** Attention layers and early layers are often more sensitive to quantization than FFN layers.
- **PagedAttention is the foundation of vLLM:** Understanding it is important for LLM serving interviews.
- **Common misconception:** KV cache stores the raw embeddings, not the attention scores. It stores the *projected* K and V tensors, not Q (which is only needed for the current token).

---

## 💻 Logical Code Snippet (Python)

```python
# ── Conceptual KV Cache in Autoregressive Decoding ───────────────────

def autoregressive_generate(model, prompt_ids, max_new_tokens):
    kv_cache = {}  # Will store: layer_id → (K_accumulated, V_accumulated)

    # Step 1: PREFILL — process entire prompt in parallel
    token_ids = prompt_ids
    output, kv_cache = model.forward(token_ids, kv_cache=None, use_cache=True)
    # kv_cache now holds K, V for all prompt tokens

    generated = []
    for step in range(max_new_tokens):
        # Get the last generated token (or last prompt token on first step)
        last_token = output[:, -1:, :]  # shape: [batch, 1, d_model]

        # Step 2: DECODE — process only the newest token
        # The model uses cached K/V for all previous tokens
        output, kv_cache = model.forward(
            last_token,
            kv_cache=kv_cache,   # reuse previous keys and values
            use_cache=True       # extend cache with new token's K/V
        )

        # Sample / argmax next token from logits
        next_token_id = output[:, -1, :].argmax(dim=-1)
        generated.append(next_token_id)

        if next_token_id == model.eos_token_id:
            break

    return generated


# ── Quantization: conceptual post-training quantization ──────────────
import torch

def quantize_to_int8(weight_fp32):
    """Map float32 weights to int8 range [-128, 127]"""
    scale = weight_fp32.abs().max() / 127.0
    weight_int8 = (weight_fp32 / scale).round().clamp(-128, 127).to(torch.int8)
    return weight_int8, scale  # scale needed for dequantization at runtime
```

---

### 6. Training vs. Inference: Key Differences (Q36)

| Dimension | Training | Inference |
|---|---|---|
| **Goal** | Learn weights by minimizing loss | Use fixed weights to generate output |
| **Compute** | Very expensive (days/weeks on clusters) | Cheaper, but must be fast |
| **Frequency** | Done once (or periodically fine-tuned) | Every user query |
| **Memory** | Stores weights + gradients + optimizer states | Stores weights + KV cache only |
| **Batch size** | Large (maximizes GPU use) | Small to medium (latency vs throughput) |

---

### 7. Latency in LLM Serving (Q37)

**Key latency metrics:**
- **Time to First Token (TTFT):** Time from sending the prompt to receiving the first output token — driven by the **prefill** phase.
- **Time Per Output Token (TPOT):** Time between each subsequent generated token — driven by the **decode** phase.
- **End-to-end latency:** Total time from query submission to complete response.

**Why low latency matters:** Chatbots, coding assistants, and search systems all require near-real-time responses. High latency = frustrated users, abandoned sessions.

---

### 8. Batching Strategies for Inference (Q38, Q39, Q40)

**Single-query inference:** One request → one response. Lowest latency but poor GPU utilization.

**Batch inference:** Multiple requests grouped and processed together. Maximizes GPU parallelism → high throughput, but all requests in a batch must wait for the *longest* one to finish → higher latency.

**Continuous (Dynamic) Batching:**
- New requests can *join* a running batch at the token level — as soon as a sequence finishes, a new one fills its slot.
- No need to wait for the entire batch to complete.
- Used in production systems (vLLM, TGI) for best-of-both-worlds throughput + latency.

| Strategy | Throughput | Latency | Use Case |
|---|---|---|---|
| Single-query | Low | ✅ Lowest | Interactive chatbots |
| Static batching | High | ❌ High (waits longest) | Offline bulk jobs |
| Continuous batching | ✅ Highest | Medium | Production serving |

**Core trade-off:** Larger batches → higher throughput, higher latency. The right batch size depends on the SLA (Service Level Agreement) for your application.

---

### 9. Mixture-of-Experts (MoE) for Inference Efficiency (Q41)

**What MoE is:** Instead of activating *all* model parameters for every token, a lightweight **gating network** selects only a small subset of "expert" feed-forward sub-networks to process each token.

**Why it helps inference:**
- A 100B-parameter MoE model might only activate 10–20B parameters per token → same quality as a dense 100B model but at a fraction of the FLOPs per forward pass.
- Faster token generation (lower latency), higher throughput.

**Analogy:** Instead of asking every specialist in a hospital to examine every patient, a triage nurse routes each patient to only the 2 most relevant specialists. Same expertise, much less wasted time.

**Trade-offs:**
- ✅ Fewer FLOPs per token → faster inference
- ❌ All expert weights must still be loaded in memory → high *memory* cost
- ❌ Load balancing is tricky — experts must be routed evenly to avoid bottlenecks

**Examples:** Mixtral 8×7B, GPT-4 (rumored), Grok.

---

### 10. Context Window (Q53, Q54)

**Definition:** The maximum number of tokens (input + output) the model can process in a single forward pass — the model's "working memory."

| Context Window | Pros | Cons |
|---|---|---|
| **Large** (e.g., 128K tokens) | Handles long documents, multi-turn history, complex reasoning | Higher memory use, higher latency, quadratic attention cost |
| **Small** (e.g., 4K tokens) | Fast, memory-efficient | "Forgets" early content; unsuitable for long documents |

**What happens when input exceeds context window:**
- Older tokens are *truncated* — the model simply cannot see them.
- This can severely degrade coherence in long conversations.
- Mitigations: sliding window attention, RAG (retrieval-augmented generation), context compression.

---

## 💻 Logical Code Snippet (Python) — Continuous Batching Concept

```python
from collections import deque

# Simulated continuous batching scheduler
def continuous_batching_scheduler(request_queue: deque, max_batch_size: int, model):
    active_sequences = []  # currently running sequences

    while request_queue or active_sequences:
        # Fill batch with new requests up to max batch size
        while len(active_sequences) < max_batch_size and request_queue:
            new_request = request_queue.popleft()
            active_sequences.append({"tokens": new_request, "done": False, "output": []})

        # One decoding step for all active sequences in parallel
        next_tokens = model.decode_step(active_sequences)  # returns [token per seq]

        for i, token in enumerate(next_tokens):
            active_sequences[i]["output"].append(token)
            if token == model.eos_token_id:
                active_sequences[i]["done"] = True
                # yield completed result immediately
                yield active_sequences[i]["output"]

        # Remove completed sequences → slot opens for new request
        active_sequences = [s for s in active_sequences if not s["done"]]
```

---

### 11. Speculative Decoding (Deep Dive) (Q61)

The key bottleneck in autoregressive decoding is that **each token requires a full forward pass** through the large model — a serial, memory-bound process.

**Mechanism:**
1. A small, fast **draft model** (e.g., 7B) generates `k` candidate tokens cheaply and quickly.
2. The large **target model** (e.g., 70B) evaluates all `k` tokens in **one parallel forward pass**.
3. Tokens the target model agrees with are accepted; generation resumes from the first rejected token.
4. Net result: multiple tokens accepted per target model forward pass.

**Why it works without quality loss:** The output distribution is mathematically equivalent to running the target model alone — rejection sampling ensures this guarantee.

**When to use it:** Latency-sensitive applications (chatbots, code completion) where a good smaller model of the same family is available (e.g., LLaMA-7B as draft for LLaMA-70B).

**Acceptance rate matters:** If draft and target models diverge frequently, acceptance rate drops below ~50% and the speedup disappears. Draft model quality is critical.

---

### 12. Flash Attention (Q64)

**The standard attention memory problem:**
- Computing `Q·Kᵀ` for a sequence of length `n` produces an `n×n` matrix.
- For n=4096, this is 16M entries — written to GPU's slow High Bandwidth Memory (HBM).
- Every forward pass reads/writes this matrix repeatedly → **memory bandwidth bottleneck**, not compute bottleneck.

**What Flash Attention does:**
- Uses **tiling**: splits Q, K, V into small blocks that fit in on-chip SRAM (fast, ~100× faster than HBM).
- Computes attention incrementally block-by-block inside SRAM, **never materializing the full n×n matrix in HBM**.
- Result: mathematically identical to standard attention, but vastly fewer slow memory reads/writes.

**Benefits:**
- ✅ 2–4× faster wall-clock time for attention
- ✅ Sub-quadratic memory usage (O(n) instead of O(n²) in HBM)
- ✅ Enables much longer context windows on the same hardware
- ✅ Now the standard in all major inference engines (vLLM, TGI, TensorRT-LLM)

**Analogy:** Instead of writing all calculations on a slow whiteboard (HBM) and re-reading them, Flash Attention does all scratch work on a small fast notepad (SRAM) and only writes the final answer.

---

### 13. Mixed Precision Inference (Q66)

**What it is:** Using FP16 (or BF16) for most operations while keeping critical computations in FP32.

| Format | Bits | Range | Precision | Use |
|---|---|---|---|---|
| FP32 | 32 | Large | High | Training numerics, loss scaling |
| FP16 | 16 | Smaller | Lower | Inference weights, activations |
| BF16 | 16 | Same as FP32 | Lower | Training + inference (preferred on A100, H100) |
| INT8 | 8 | Integer only | Very low | Quantized inference |

**Why FP16/BF16 for inference:**
- ✅ 2× memory reduction vs FP32
- ✅ NVIDIA Tensor Cores run FP16/BF16 ops significantly faster
- ✅ Negligible quality loss for most tasks
- BF16 is generally preferred over FP16 because it has the same exponent range as FP32, reducing overflow risk

---

### 14. Distributed Inference Challenges (Q62)

**When a single GPU isn't enough** (e.g., LLaMA-405B won't fit on one A100 80GB):

| Parallelism Strategy | How It Works | Best For |
|---|---|---|
| **Tensor Parallelism** | Split individual weight matrices across GPUs; each GPU computes a slice | Large individual layers |
| **Pipeline Parallelism** | Different layers live on different GPUs; input flows through sequentially | Very deep models |
| **Data Parallelism** | Each GPU holds a full model copy; different batches processed in parallel | High throughput |

**Core challenges:**
- **Communication overhead:** After each layer, GPUs must synchronize (AllReduce) — this inter-GPU communication adds latency, especially over PCIe vs NVLink.
- **Load imbalance:** If one GPU finishes early, it waits — poor utilization.
- **Phase mismatch:** Prefill is compute-bound (benefits from tensor parallelism); decode is memory-bound (needs fewer GPUs per request, more concurrent requests).

---

### 15. Designing a Scalable LLM Serving System (Q63)

**Production system components:**

```
User Request
     ↓
Load Balancer (routes to least-loaded node)
     ↓
Request Queue + Continuous Batching Scheduler
     ↓
Inference Engine (vLLM / TensorRT-LLM)
  ├── PagedAttention KV Cache Manager
  ├── Flash Attention Kernels
  ├── Tensor Parallel across GPUs
  └── Prefix Cache (reuse KV for repeated system prompts)
     ↓
Token Streaming to Client
     ↓
Autoscaling (add GPU nodes on traffic spikes)
```

**Key design decisions:**
- **vLLM** for most use cases: best throughput + Hugging Face compatibility + PagedAttention
- **TensorRT-LLM** when lowest latency on NVIDIA hardware is paramount
- **Prefix caching** when many requests share the same system prompt (saves recomputing KV for the prefix)
- **Autoscaling** based on queue depth (requests waiting) rather than CPU/GPU %, which lags

---

### 16. Inference Performance Metrics (Q70)

| Metric | Definition | Driven By |
|---|---|---|
| **TTFT** (Time to First Token) | Latency from query submission to first output token | Prefill speed; proportional to prompt length |
| **TPOT** (Time Per Output Token) | Average time between consecutive output tokens | Decode speed; memory bandwidth bound |
| **E2E Latency** | Total time from query to complete response | TTFT + (TPOT × output length) |
| **Throughput** | Tokens/sec or requests/sec across all users | Batch size, continuous batching efficiency |
| **GPU utilization** | % of GPU compute being used | Batching strategy, scheduling |

**Online vs. Offline deployment (Q67):**

| | Online Inference | Offline Inference |
|---|---|---|
| Traffic pattern | Unpredictable, real-time | Pre-collected, batch jobs |
| Primary metric | Latency (TTFT, TPOT) | Throughput (tokens/sec) |
| Infrastructure | Cloud, autoscaling | On-premise, fixed resources |
| Examples | Chatbots, copilots | Bulk classification, embeddings |

---

### 17. Inference Acceleration Toolkit (Q73)

A complete map of acceleration levers:

| Technique | What It Reduces | Trade-off |
|---|---|---|
| **KV Caching** | Redundant K/V recomputation | GPU memory grows with context |
| **Quantization (INT8/INT4)** | Model memory footprint, compute | Minor accuracy loss |
| **Flash Attention** | HBM memory I/O | Requires kernel-level implementation |
| **Continuous Batching** | GPU idle time between requests | Slightly more complex scheduling |
| **Speculative Decoding** | Serial decode bottleneck | Requires draft model |
| **Pruning** | Model parameter count | Accuracy drop if over-pruned |
| **Knowledge Distillation** | Model size (student model) | Training cost; quality ceiling |
| **MoE / Sparse Activation** | Active FLOPs per token | High total memory for all experts |
| **Hardware (GPU/TPU)** | Wall-clock time | Cost |

**Common inference engines (Q71):**

| Engine | Strength | Best For |
|---|---|---|
| **vLLM** | PagedAttention, continuous batching, HF compatible | General cloud serving |
| **TensorRT-LLM** | Lowest latency, CUDA kernels | NVIDIA-only, latency-critical |
| **Text Generation Inference (TGI)** | HuggingFace ecosystem, production-ready | HF model serving |
| **llama.cpp** | CPU inference, quantized | Edge, local development |
| **LMDeploy** | Efficient serving, turbomind backend | Research / Chinese ecosystem |

---

## 💻 Logical Code Snippet (Python) — Continuous Batching Concept

```python
from collections import deque

def continuous_batching_scheduler(request_queue: deque, max_batch_size: int, model):
    active_sequences = []

    while request_queue or active_sequences:
        while len(active_sequences) < max_batch_size and request_queue:
            new_request = request_queue.popleft()
            active_sequences.append({"tokens": new_request, "done": False, "output": []})

        next_tokens = model.decode_step(active_sequences)

        for i, token in enumerate(next_tokens):
            active_sequences[i]["output"].append(token)
            if token == model.eos_token_id:
                active_sequences[i]["done"] = True
                yield active_sequences[i]["output"]

        active_sequences = [s for s in active_sequences if not s["done"]]


# ── Flash Attention: conceptual tiling logic ─────────────────────────
def flash_attention_tiled(Q, K, V, block_size=64):
    """
    Instead of computing full n×n attention matrix in slow HBM,
    process in SRAM-sized blocks and accumulate result incrementally.
    This is the key idea — actual implementation uses custom CUDA kernels.
    """
    n = Q.shape[0]
    output = torch.zeros_like(Q)

    for i in range(0, n, block_size):          # iterate over query blocks
        Q_block = Q[i:i+block_size]            # load query block into SRAM
        running_sum = torch.zeros(Q_block.shape[0])
        running_max = torch.full((Q_block.shape[0],), float('-inf'))

        for j in range(0, n, block_size):      # iterate over key/value blocks
            K_block = K[j:j+block_size]        # load KV block into SRAM
            V_block = V[j:j+block_size]

            # Compute partial attention scores in SRAM (never write to HBM)
            scores = Q_block @ K_block.T / math.sqrt(Q.shape[-1])
            # ... online softmax update (numerically stable, incremental)
            # ... accumulate weighted values
        
        output[i:i+block_size] = ...           # write only final result to HBM

    return output
```

---

## ❓ Active Recall
- [ ] Walk through the 4 phases of LLM inference from raw text to output text.
- [ ] What is the KV cache? What problem does it solve? What does it store?
- [ ] Why does KV cache size grow linearly with sequence length and batch size?
- [ ] What is PagedAttention, and what memory problem does it solve?
- [ ] What is the difference between GQA and MQA? How do they reduce KV cache size?
- [ ] What is quantization? Compare INT8 vs INT4 trade-offs.
- [ ] What is the computational complexity of self-attention? Why is it quadratic?
- [ ] What are TTFT and TPOT? Which inference phase drives each?
- [ ] What is continuous batching? How does it differ from static batching?
- [ ] What is MoE? How does it reduce FLOPs without reducing model quality?
- [ ] What are the trade-offs of large vs. small context windows?
- [ ] What is speculative decoding? What is the quality guarantee?
- [ ] What memory bottleneck does Flash Attention solve? What is "tiling"?
- [ ] What is the difference between FP16 and BF16? Which is preferred for inference?
- [ ] What are the three parallelism strategies for distributed inference?
- [ ] Name 3 inference engines and their primary strengths.
- [ ] List 5 techniques to accelerate LLM inference and what each reduces.
- [ ] What is the difference between online and offline LLM inference?
- [ ] **Follow-up:** What is "prefill-decode disaggregation"?
- [ ] **Follow-up:** What is prefix caching and when does it help?
- [ ] **Follow-up:** What is knowledge distillation and how does it differ from quantization?
