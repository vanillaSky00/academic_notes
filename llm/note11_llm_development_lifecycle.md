# 📄 LLM Development Lifecycle: Pre-training, Fine-tuning & Alignment

**Tags:** #llm #pretraining #fine-tuning #instruction-tuning #rlhf #alignment #interview
**Links:** [[Prompt Engineering]], [[Training Stability]], [[Inference Optimization]]

---

## 🎯 The "Elevator Pitch"
> Building an LLM is a three-act story: first, teach it the entire structure of human language and knowledge by predicting billions of words (pre-training); then, teach it to follow instructions politely (instruction tuning); finally, teach it to be helpful, honest, and safe (alignment). Each phase uses different data, different objectives, and different compute budgets.

---

## 🧠 Core Mechanics

### 1. The Three Phases of LLM Development (Q88)

```
Phase 1: PRE-TRAINING
  "Learn everything about language and the world"
  Data: Trillions of tokens from the web (Common Crawl, books, code, Wikipedia...)
  Objective: Next-token prediction (self-supervised)
  Output: A base model that can continue any text — but doesn't yet "follow instructions"

         ↓

Phase 2: INSTRUCTION TUNING (Supervised Fine-tuning / SFT)
  "Learn to follow instructions helpfully"
  Data: High-quality (instruction, response) pairs — tens to hundreds of thousands
  Objective: Supervised cross-entropy on the response tokens
  Output: A model that understands and executes diverse user instructions

         ↓

Phase 3: ALIGNMENT TUNING (RLHF / DPO)
  "Learn to be helpful, harmless, and honest"
  Data: Human preference comparisons (response A vs response B)
  Objective: Maximize human preference via reinforcement learning (or distillation)
  Output: The production model — helpful, safe, well-calibrated
```

**Analogy:**
- Pre-training = sending a student to read every book in the library for years
- Instruction tuning = teaching that student how to answer exam questions properly
- Alignment = teaching that student professional ethics and how to interact with people respectfully

---

### 2. Pre-training Deep Dive

**Objective:** Given tokens `t₁, t₂, ..., tₙ`, predict `tₙ₊₁` at every position.
```
Loss = -Σ log P(tₜ | t₁, ..., tₜ₋₁)
```

**Scale of pre-training:**
- LLaMA-3 70B: trained on ~15 trillion tokens
- GPT-4: estimated ~10–13 trillion tokens
- Training duration: weeks to months on thousands of GPUs
- Cost: $10M–$100M+ for frontier models

**What the model learns:**
- Grammar, syntax, and semantics of language
- World knowledge: facts, relationships, common sense
- Code patterns, math reasoning, multiple languages
- In-context learning ability (emerges at scale)

**Key data quality matters:** A model trained on clean, diverse data generalizes better than one trained on more but lower-quality data (the "data flywheel" insight).

---

### 3. Types of Fine-tuning (Q89)

| Type | Purpose | Data | Objective |
|---|---|---|---|
| **Instruction Fine-tuning (SFT)** | Teach general instruction-following | Diverse (instruction, response) pairs | Cross-entropy on response tokens |
| **Task-specific Fine-tuning** | Maximize performance on one task | Domain-specific labeled dataset | Task loss (classification, extraction...) |
| **Alignment Tuning (RLHF/DPO)** | Align with human values and preferences | Human preference comparisons | Reward maximization / preference optimization |

**Full fine-tuning vs. Parameter-Efficient Fine-tuning (PEFT):**

| | Full Fine-tuning | PEFT (e.g., LoRA) |
|---|---|---|
| Parameters updated | All (~billions) | Small adapters (~millions) |
| Compute cost | Very high | Low |
| Memory required | Huge (gradients for all params) | Minimal |
| Risk of catastrophic forgetting | Higher | Lower |
| When to use | Large proprietary datasets, max performance | Resource-constrained; quick adaptation |

---

### 4. Instruction Fine-tuning (SFT) in Detail (Q90)

**The gap instruction tuning fills:**

A pre-trained base model is trained to *complete* text, not *answer* questions. If you ask it "What is the capital of France?", it might respond "What is the capital of Germany? What is the capital of Italy?" — because that's a natural text continuation.

Instruction tuning fixes this by training on pairs like:
```
Instruction: "What is the capital of France?"
Response: "The capital of France is Paris."
```

**What "high-quality" means for SFT data:**
- Diverse tasks (QA, summarization, coding, translation, math, creative writing)
- Correct, well-reasoned responses
- Responses at the right length — not too terse, not verbose
- Coverage of refusal cases (the model should know when NOT to answer)

**Key insight from the literature:** A small amount of very high-quality SFT data (e.g., 1K–10K examples) often outperforms large amounts of noisy data. LIMA (2023) showed a model fine-tuned on just 1,000 curated examples could match much larger SFT datasets in quality.

---

### 5. Alignment Tuning: RLHF & DPO (Q89)

**Why SFT isn't enough:** A model can learn to follow instructions but still produce harmful, biased, or sycophantic responses. Alignment tuning teaches the model to maximize *human preference*, not just imitate responses.

#### RLHF (Reinforcement Learning from Human Feedback)

**Three-step process:**
```
Step 1: SFT — Fine-tune base model on instruction data (produces SFT model)

Step 2: Reward Modeling
  - Collect human comparisons: for same prompt, humans rank response A vs B
  - Train a reward model (RM) to predict which response humans prefer
  - RM outputs a scalar reward score for any (prompt, response) pair

Step 3: RL Optimization (PPO)
  - Use the RM as the reward signal to fine-tune the SFT model via PPO
  - Model learns to generate responses that maximize the reward model score
  - KL penalty keeps the model from drifting too far from the SFT model
```

**Problems with RLHF:**
- Reward hacking: model finds responses that fool the reward model without being genuinely good
- PPO is unstable and compute-intensive
- Requires training and maintaining a separate reward model

#### DPO (Direct Preference Optimization)

**Insight:** RLHF's three-step process can be collapsed into a single supervised loss that directly optimizes on preference pairs — no reward model, no RL.

```python
# DPO loss (simplified concept)
# Given: prompt x, preferred response y_w, rejected response y_l
# Maximize: log P(y_w | x) - log P(y_l | x) relative to reference model

dpo_loss = -log(sigmoid(
    beta * (log_prob_preferred - log_prob_ref_preferred) -
    beta * (log_prob_rejected - log_prob_ref_rejected)
))
```

**Why DPO is increasingly preferred:**
- ✅ Simpler: no separate reward model
- ✅ More stable: standard supervised training
- ✅ Competitive quality with RLHF
- Used in: LLaMA-3, Mistral fine-tunes, many open models

---

### 6. The Full Development Stack in Context

```
Raw Text Corpus (internet, books, code)
     ↓  [Pre-training — weeks/months, thousands of GPUs]
Base Model (can complete text, has world knowledge)
     ↓  [Instruction Fine-tuning — days, hundreds of GPUs]
Instruction-following Model (answers questions, follows directions)
     ↓  [RLHF or DPO — days, human labelers + GPU]
Aligned Production Model (helpful, safe, well-calibrated)
     ↓  [Optional: task-specific fine-tuning or RAG]
Deployed Application Model
```

---

## ⚠️ Edge Cases & Constraints
- **Catastrophic forgetting:** Fine-tuning on a narrow task can degrade the model's general capabilities. Regularization or PEFT (LoRA) mitigates this.
- **Data contamination in pre-training:** If test sets from benchmarks leaked into training data, reported scores are inflated — a major evaluation validity concern.
- **Reward model collapse:** In RLHF, if RL runs too long, the model finds adversarial outputs that game the reward model while producing nonsense. KL divergence penalties and reward model updates are used to counteract this.
- **SFT data sycophancy risk:** If all fine-tuning responses are excessively agreeable, the model learns to flatter rather than inform. Diverse, honest response styles in training data are essential.
- **Common misconception:** "Fine-tuning" ≠ "training from scratch." Fine-tuning starts from existing pre-trained weights and performs a small number of additional gradient steps.

---

## 💻 Logical Code Snippet (Python)

```python
from transformers import AutoModelForCausalLM, AutoTokenizer, TrainingArguments
from trl import SFTTrainer, DPOTrainer

# ── Phase 2: Instruction Fine-tuning (SFT) ────────────────────────────
model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3-8B")
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3-8B")

# SFT dataset: list of {"instruction": ..., "response": ...} dicts
def format_instruction(sample):
    return f"""### Instruction:\n{sample['instruction']}\n\n### Response:\n{sample['response']}"""

sft_trainer = SFTTrainer(
    model=model,
    train_dataset=sft_dataset,
    formatting_func=format_instruction,
    args=TrainingArguments(
        output_dir="./sft_model",
        num_train_epochs=3,
        per_device_train_batch_size=4,
        learning_rate=2e-5,
    )
)
sft_trainer.train()  # SFT model saved to ./sft_model


# ── Phase 3: DPO Alignment ────────────────────────────────────────────
# DPO dataset: {"prompt": ..., "chosen": ..., "rejected": ...}
dpo_trainer = DPOTrainer(
    model=sft_model,          # start from SFT model
    ref_model=sft_model_ref,  # frozen reference (the SFT model before DPO)
    beta=0.1,                 # KL regularization strength
    train_dataset=dpo_dataset,
    tokenizer=tokenizer,
    args=TrainingArguments(
        output_dir="./aligned_model",
        num_train_epochs=1,
        per_device_train_batch_size=2,
        learning_rate=5e-7,
    )
)
dpo_trainer.train()  # Aligned model: helpful + safe


# ── LoRA: Parameter-Efficient Fine-tuning ─────────────────────────────
from peft import LoraConfig, get_peft_model

lora_config = LoraConfig(
    r=16,            # rank of the low-rank decomposition
    lora_alpha=32,   # scaling factor
    target_modules=["q_proj", "v_proj"],  # which layers to adapt
    lora_dropout=0.05,
    bias="none",
)
# Only ~0.1% of parameters are trainable — the rest are frozen
peft_model = get_peft_model(model, lora_config)
peft_model.print_trainable_parameters()
# trainable params: 4,194,304 || all params: 8,030,261,248 || trainable%: 0.05%
```

---

---

### 7. Alignment Tuning's Role in Usability (Q91)

**What alignment tuning actually achieves:**
- A model can be *capable* (can answer anything) but *not useful* (answers unsafely, rudely, or inaccurately) without alignment.
- Alignment tuning specifically targets the three H's: **Helpful, Harmless, Honest**.

| Without Alignment | With Alignment |
|---|---|
| May generate toxic/harmful content | Refuses dangerous requests |
| May be sycophantic or deceptive | Expresses uncertainty honestly |
| Response quality/tone varies wildly | Consistent, natural, trustworthy responses |
| No refusals — completes any prompt | Principled refusals with explanation |

**Key insight:** Alignment is what transforms a powerful but raw language model into a product that people actually trust and want to use daily.

---

### 8. Preventing Overfitting During Fine-tuning (Q92)

**Symptoms of overfitting during fine-tuning:**
- Training loss keeps dropping; validation loss rises or plateaus — the model memorizes training examples instead of generalizing.

**Prevention strategies:**

| Technique | Mechanism | When to Use |
|---|---|---|
| **Validation set + early stopping** | Stop training when val loss increases for N steps | Always — the primary defense |
| **L2 regularization (weight decay)** | Penalizes large weight magnitudes | General use |
| **Dropout** | Randomly zeroes activations during training | Dense layers; less common in Transformers now |
| **Data augmentation** | Paraphrase/back-translate training examples | Small datasets |
| **Smaller learning rate** | Prevents large weight updates that overfit | Always use smaller LR than pre-training |
| **LoRA / PEFT** | Fewer trainable params → structurally harder to overfit | Resource-constrained or small datasets |
| **Diverse, sufficient data** | Model sees more variation → better generalization | Always |

**Rule of thumb:** For fine-tuning LLMs, early stopping on a held-out validation set is the single most reliable overfitting defense.

---

### 9. Catastrophic Forgetting — Deep Dive (Q93)

**What happens mechanically:**
- Full fine-tuning performs gradient descent on a narrow loss from the new task.
- Gradient updates move weights toward the new task's optimum — but away from the pre-training optimum.
- Weights that encoded general knowledge get overwritten.

**Example:** A GPT model fine-tuned heavily on legal text may lose its ability to write poetry or answer general science questions.

**Mitigations ranked by effectiveness:**

| Strategy | How | Trade-off |
|---|---|---|
| **LoRA / PEFT** | Freeze base weights; only train small adapters | Best protection; slight quality ceiling |
| **Replay / data mixing** | Mix original pre-training data into fine-tuning | Computationally expensive |
| **Low learning rate** | Small weight changes → less forgetting | May underfit the new task |
| **Early stopping** | Stop before weights drift too far | May leave performance on the table |
| **Elastic Weight Consolidation (EWC)** | Penalizes changes to weights important for old tasks | Complex; rarely used in LLMs |

---

### 10. Full Fine-tuning vs. PEFT (Q94, Q95)

**Full fine-tuning:**
- Updates every parameter (billions of weights)
- Requires storing gradients + optimizer states for all params → 3–6× model size in GPU RAM during training
- Highest possible performance ceiling
- Risk: catastrophic forgetting, high cost, one checkpoint per task

**PEFT (Parameter-Efficient Fine-tuning):**
- Freezes base model; trains a tiny adapter (typically 0.01–1% of params)
- Can share one base model across many tasks — just swap adapters
- Near-full-fine-tuning performance at a fraction of the cost
- **LoRA is the dominant PEFT method** for LLMs

**Memory comparison for a 7B model fine-tuned with AdamW:**

| Method | GPU RAM Needed |
|---|---|
| Full fine-tuning (FP32) | ~112 GB (model + grads + optimizer) |
| Full fine-tuning (BF16 + mixed precision) | ~56 GB |
| LoRA (rank=16, BF16) | ~14–18 GB |
| QLoRA (4-bit + LoRA) | ~6–10 GB ← consumer GPU range |

---

### 11. LoRA Deep Dive (Q101, Q102)

**The core mathematical idea:**

A pre-trained weight matrix `W ∈ ℝ^(d×k)` (e.g., d=4096, k=4096) has 16M parameters.

LoRA hypothesizes that **the update ΔW lies in a low-rank subspace**:
```
ΔW = B · A    where B ∈ ℝ^(d×r), A ∈ ℝ^(r×k), r << d
```

For rank r=16: instead of 16M parameters, you train only `d×r + r×k = 4096×16 + 16×4096 = 131,072` parameters — **122× fewer**.

**During training:**
- `W` is frozen (no gradient flows through it)
- Only `A` and `B` receive gradients
- `A` is initialized randomly (small normal distribution); `B` is initialized to zero → ΔW starts at zero, meaning training begins from the original model's behavior

**During inference:**
- Merge: `W_effective = W + B·A` — no extra latency at inference time
- Or keep separate for multi-task serving (hot-swap adapters)

**Why low-rank works (Q102):**
- Large LLMs have massively redundant parameters — most of the weight space is "inactive" for any specific task.
- Task-specific knowledge can be compressed into a very low-dimensional subspace.
- Empirical finding: rank r=8 to r=64 covers most tasks; higher rank rarely helps and increases cost.

**LoRA hyperparameters to know:**
- `r` (rank): dimensionality of the update subspace. Higher = more capacity, more params.
- `alpha` (lora_alpha): scaling factor applied to ΔW. Effective learning rate ≈ `(alpha / r) × lr`.
- `target_modules`: which weight matrices to apply LoRA to (typically `q_proj`, `v_proj`, sometimes `k_proj`, `o_proj`, FFN layers).

---

### 12. QLoRA (Q103, Q104, Q105)

**What QLoRA adds on top of LoRA:**

```
Standard LoRA:   BF16 frozen weights  +  BF16 LoRA adapters
QLoRA:           4-bit NF4 frozen weights  +  BF16 LoRA adapters
```

**Three key QLoRA innovations (Dettmers et al., 2023):**
1. **4-bit NormalFloat (NF4):** A quantization format optimized for normally-distributed model weights — lower error than standard INT4.
2. **Double quantization:** Quantizes the quantization constants themselves, saving an additional ~0.5 GB per 7B model.
3. **Paged optimizers:** Offloads optimizer states to CPU RAM during memory spikes — prevents OOM crashes.

**Practical impact:**
- LLaMA-2 65B fine-tunable on a **single 48GB A40 GPU** with QLoRA — would need ~4× A100s with standard LoRA.
- Quality on most benchmarks: QLoRA ≈ full fine-tuning on the same tasks.

**Decision guide:**

```
Have ≥2 A100 80GB GPUs?  → Standard LoRA (faster, simpler)
Single consumer GPU (RTX 3090/4090, 24GB)?  → QLoRA
Very large model (65B+)?  → QLoRA (only practical option on limited hardware)
```

---

### 13. Gradient Accumulation (Q107)

**The problem:** You want to train with an effective batch size of 64, but fitting 64 samples into GPU RAM at once is impossible — GPU runs out of memory.

**The solution:** Process micro-batches of size 4, accumulate the gradients from each step without updating weights, and only perform the weight update after 16 micro-steps.

```
Effective batch size = micro_batch_size × accumulation_steps
64 = 4 × 16

Step 1:  compute loss on 4 samples, accumulate grads (no weight update)
Step 2:  compute loss on 4 samples, accumulate grads
...
Step 16: compute loss on 4 samples, accumulate grads → NOW update weights
```

**Trade-off:** Same final result as training with batch size 64, but 16 forward passes instead of 1 → ~16× slower per effective batch. Useful for correctness; doesn't save compute, only memory.

---

### 14. Fine-tuning Speedup Toolkit (Q108)

| Technique | What It Reduces | Typical Speedup |
|---|---|---|
| **LoRA / QLoRA** | Trainable parameters (→ gradient memory) | 2–8× memory reduction |
| **Mixed precision (BF16)** | Memory + compute | 2× memory; 2–3× faster matmuls |
| **Gradient accumulation** | Peak memory per step | Enables larger effective batches |
| **Gradient checkpointing** | Activation memory (re-compute on backward) | 2–4× memory reduction; ~20% slower |
| **Flash Attention** | Attention memory I/O | 2–4× faster attention |
| **Distributed training (DDP/FSDP)** | Wall-clock time (more GPUs) | Near-linear with GPU count |
| **Smaller sequence length** | Quadratic attention cost | Large impact for long-context data |

**Gradient checkpointing** (worth knowing): Instead of storing all intermediate activations during forward pass (needed for backprop), only store activations at "checkpoint" layers and recompute others during backward. Trades ~20% compute for large activation memory savings.

---

### 15. Pre-training Objectives Deep Dive (Q109, Q110, Q115)

#### Causal Language Modeling (CLM) — used by GPT, LLaMA, Claude
- **Task:** Predict the next token given all previous tokens (left-to-right only).
- **Attention:** Causal (masked) — each token attends only to past tokens.
- **Creates:** Decoder-only models, ideal for generation.
- **Self-supervised:** Labels are the input tokens shifted by one position — no human annotation needed.

#### Masked Language Modeling (MLM) — used by BERT, RoBERTa
- **Task:** Predict randomly masked tokens using bidirectional context (past AND future).
- **Attention:** Bidirectional (full) — every token sees every other.
- **Creates:** Encoder-only models, ideal for understanding tasks (classification, NER, QA extraction).
- **Not generative:** Cannot generate open-ended text — only "fill in the blank."

| | CLM (Causal) | MLM (Masked) |
|---|---|---|
| Context used | Left context only | Left + right context |
| Architecture | Decoder-only | Encoder-only |
| Ideal for | Text generation | Text understanding |
| Examples | GPT, LLaMA, Claude | BERT, RoBERTa, DistilBERT |
| Self-supervised? | ✅ Yes | ✅ Yes |

**Self-supervised learning significance (Q115):** Both CLM and MLM derive their training signal automatically from raw text — no human labelers needed. This is what makes trillion-token pre-training tractable. The "label" is already present in the data itself.

---

### 16. Scaling Laws (Q112)

**Empirical finding (Kaplan et al. 2020, Hoffmann et al. 2022 "Chinchilla"):**

Model performance (measured by loss) follows a **power law** with respect to:
- **N** — number of model parameters
- **D** — number of training tokens
- **C** — total compute budget (FLOPs ≈ 6 × N × D)

**Chinchilla scaling law (key result):**
> For a given compute budget C, optimal performance is achieved by training a model of size `N ∝ √C` on `D ∝ √C` tokens — i.e., **model size and data size should scale equally**.

**Practical implication:**
- Pre-Chinchilla: models were massively over-parameterized and under-trained (GPT-3 was too big for its data budget).
- Post-Chinchilla: Llama models are smaller but trained on far more tokens — often outperforming larger models trained on less data.

**Analogy:** Scaling laws are like a recipe ratio. You can't make a great cake by only adding more flour — you need to scale all ingredients together. More parameters without more data is wasted capacity.

---

### 17. MoE in Pre-training (Q113)

**How MoE replaces the FFN in a Transformer:**

```
Standard Transformer layer:
  Token → Self-Attention → FFN (dense, all params activated) → Output

MoE Transformer layer:
  Token → Self-Attention → Router → [Expert 1] [Expert 2] ... [Expert N]
                                         ↑ only top-k activated
                                    → Weighted sum of expert outputs → Output
```

**Pre-training benefits:**
- Total parameter count (capacity) can be scaled to hundreds of billions.
- Active parameters per token stay low (e.g., 14B active out of 140B total in Mixtral 8×7B).
- Pre-training compute budget is the same as a dense 14B model — but model quality approaches a dense 140B.

**Load balancing loss:** During training, a regularization term encourages all experts to be used roughly equally — preventing collapse where only 1–2 experts get all tokens.

---

### 18. Model Parallelism in Pre-training (Q114)

**Why it's needed:** LLaMA-3 405B has 405 billion parameters. At BF16 (2 bytes/param), that's 810 GB — far exceeding any single GPU's memory.

**Three parallelism strategies (revisited for training context):**

| Strategy | Split | Communication | Best For |
|---|---|---|---|
| **Tensor Parallelism (TP)** | Weight matrices split across GPUs within a layer | High (AllReduce after each layer) | Wide layers; NVLink required |
| **Pipeline Parallelism (PP)** | Different layers on different GPUs | Medium (pass activations between stages) | Very deep models |
| **Data Parallelism (DP/FSDP)** | Each GPU holds a shard of the model; different batches | Medium (gradient sync) | Standard distributed training |

**ZeRO (Zero Redundancy Optimizer):** A data parallelism optimization that shards optimizer states, gradients, and parameters across GPUs — eliminating redundant memory. Used in DeepSpeed, PyTorch FSDP.

**In practice:** Large models combine all three — e.g., TP=8 within a node (NVLink), PP=4 across nodes, DP=N over the full cluster.

---

## 💻 Logical Code Snippet (Python) — Extended

```python
# ── QLoRA fine-tuning setup ─────────────────────────────────────────
from transformers import BitsAndBytesConfig, AutoModelForCausalLM
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
import torch

# Step 1: Load model in 4-bit (NF4 quantization)
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",        # NormalFloat4 — optimal for LLM weights
    bnb_4bit_use_double_quant=True,   # double quantization for extra memory savings
    bnb_4bit_compute_dtype=torch.bfloat16,  # compute in BF16 even though stored in 4-bit
)
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3-8B",
    quantization_config=bnb_config,
    device_map="auto",
)

# Step 2: Prepare model for k-bit training (handles gradient casting, etc.)
model = prepare_model_for_kbit_training(model)

# Step 3: Attach LoRA adapters (only these train; base model stays frozen in 4-bit)
lora_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM",
)
model = get_peft_model(model, lora_config)
# Result: 8B model trainable on a single 24GB GPU


# ── Gradient Accumulation ─────────────────────────────────────────────
from transformers import TrainingArguments

training_args = TrainingArguments(
    output_dir="./output",
    per_device_train_batch_size=2,      # micro-batch: 2 samples
    gradient_accumulation_steps=16,    # effective batch: 2×16 = 32
    gradient_checkpointing=True,       # save activation memory at ~20% compute cost
    bf16=True,                         # mixed precision
    learning_rate=2e-4,
    num_train_epochs=3,
)


# ── Scaling law intuition ─────────────────────────────────────────────
def chinchilla_optimal_tokens(num_params: int) -> int:
    """
    Chinchilla law: optimal token count ≈ 20× number of parameters.
    e.g., 7B model → train on ~140B tokens for compute-optimal performance.
    """
    return 20 * num_params

print(chinchilla_optimal_tokens(7_000_000_000))  # → 140,000,000,000 tokens
```

---

## ❓ Active Recall
- [ ] What are the three phases of LLM development? What data and objective does each use?
- [ ] What does a pre-trained base model produce when prompted with a question? Why?
- [ ] What gap does instruction fine-tuning fill? What is the training data format?
- [ ] What is the difference between CLM and MLM? Which creates generative models?
- [ ] What is self-supervised learning? Why is it essential for pre-training at scale?
- [ ] What are scaling laws? What did Chinchilla show about optimal compute allocation?
- [ ] What is RLHF? Walk through its three steps.
- [ ] What is DPO? What are ORPO and KTO? How do alignment methods differ?
- [ ] What is catastrophic forgetting? List three mitigations ranked by reliability.
- [ ] What is the difference between full fine-tuning and PEFT? Give GPU RAM numbers for a 7B model.
- [ ] What is LoRA? Write the weight update formula. What does rank r control?
- [ ] What does QLoRA add on top of LoRA? What are the three key innovations?
- [ ] What is gradient accumulation? What problem does it solve?
- [ ] What is gradient checkpointing? What does it trade off?
- [ ] What is MoE? How does it appear in both pre-training and inference?
- [ ] What are the three model parallelism strategies? When is each used?
- [ ] What is ZeRO / FSDP?
- [ ] What is overfitting during fine-tuning? What is the primary defense?
- [ ] What role does alignment tuning play in making a model trustworthy?
- [ ] **Follow-up:** What is RLAIF / Constitutional AI?
- [ ] **Follow-up:** What is continual pre-training vs. domain-adaptive pre-training?
- [ ] **Follow-up:** What is the "data flywheel" and why does data quality matter more than quantity post-Chinchilla?
