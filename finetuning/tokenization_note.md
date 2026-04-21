# 📄 Tokens, Tokenizers & Embeddings: How LLMs Read and Write Text

**Tags:** #deep-learning #tokenization #BPE #embeddings #LLM #NLP  
**Links:** [[Byte-Pair Encoding]], [[Embedding Layer]], [[Sampling Strategies]], [[Beam Search]], [[Fine-Tuning]], [[Vocabulary Size]]

---

## 🎯 The "Elevator Pitch"
> Before a neural network can do anything with text, that text must become numbers. **Tokenization** is the algorithm that maps raw characters → compact integer IDs, and **embeddings** are the learned vectors those IDs look up to enter the model. Every design choice here — vocabulary size, algorithm, freezing strategy — cascades into training cost, multilingual equity, and model capabilities.

---

## 🧠 Core Mechanics

### 1. Why Not Just Characters or Words?

| Scheme | Problem |
|---|---|
| **Character-level** | Sequences become absurdly long; attention cost scales quadratically with length |
| **Word-level** | Vocabulary explodes; out-of-vocabulary (OOV) words unseen at training time break inference |
| **Subword (BPE/Unigram)** | Sweet spot: short sequences, rare words can be decomposed into known subword pieces |

---

### 2. Byte-Pair Encoding (BPE) — The Dominant Algorithm

**Algorithm:**
1. Start with vocabulary = all individual characters (or bytes)
2. Count all adjacent symbol pairs in the corpus
3. Merge the most frequent pair into a new symbol
4. Repeat until vocabulary reaches target size $|V|$

**Intuition:** Common patterns get compressed into single tokens. `ING`, `TION`, `##ing` become first-class citizens. Rare words decompose into known sub-pieces rather than becoming `<UNK>`.

**GPT-2/3/4 tokenizer:** ~50K BPE tokens over bytes (so truly no OOV — any byte sequence is encodeable).  
**Llama 3:** 128K vocabulary — longer merges, shorter token sequences, cheaper attention.

**Merge rules are learned, deterministic, and frozen at inference** — the tokenizer is not a neural network, it's a lookup table + merge priority list.

---

### 3. Tokenizer Variants

| Tokenizer | Algorithm | Space handling | Used by |
|---|---|---|---|
| GPT-2 / GPT-4 | Byte-level BPE | Special G-char | GPT series |
| BERT | WordPiece | `##` prefix for continuations | BERT, DistilBERT |
| T5 / SentencePiece | Unigram LM | Underscore for spaces | T5, mT5 |
| DeepSeek | BPE | Special space glyph | DeepSeek |

**Unigram LM (SentencePiece):** Instead of bottom-up merging, starts with a huge seed vocabulary and iteratively *prunes* tokens whose removal least increases total corpus loss. This probabilistic approach naturally recovers morphological suffixes (`-ly`, `-ing`, `-tion`) and handles morphologically rich languages (Turkish, Finnish) better than BPE.

---

### 4. Embeddings: From Token ID → Model Input

The model doesn't operate on integers directly. Each token ID indexes into an **embedding matrix** $E \in \mathbb{R}^{|V| \times d}$, where $d$ is the model's hidden dimension (e.g., 4096 for Llama 3-8B).

```
Token ID 15324  →  E[15324]  →  float32 vector of size d
```

This is not just a lookup table trick — the embedding matrix is **trained** alongside the model. The vector for `Sacramento` should be geometrically close to `California capital`, `city`, `government`, etc. The embedding layer is where the model first encodes semantic meaning.

The final layer (the **language model head**) does the reverse: it projects the model's hidden state back into a distribution over all $|V|$ tokens using a weight matrix that often shares weights with the embedding layer (**weight tying**).

---

### 5. Sampling the Next Token

After the forward pass, the model produces logits $z \in \mathbb{R}^{|V|}$, which are converted to a probability distribution via softmax. Then you pick:

| Strategy | Mechanism | Trade-off |
|---|---|---|
| **Greedy** | $\arg\max(p)$ | Deterministic, fast, repetitive |
| **Temperature sampling** | Scale logits: $p_i \propto \exp(z_i / T)$ | $T \to 0$: greedy; $T \to \infty$: uniform |
| **Top-k sampling** | Sample from top-$k$ tokens only | Reduces noise; $k$ is a hyperparameter |
| **Top-p (nucleus)** | Sample from smallest set whose cumulative prob $\geq p$ | Adapts to distribution shape |
| **Beam search** | Maintain $B$ candidate sequences in parallel | Better quality; no randomness; slower |

**Temperature intuition:** Low $T$ → peaky distribution → conservative (vanilla ice cream). High $T$ → flat distribution → creative/noisy (random flavor).

---

### 6. Batching & Padding

GPUs require fixed-size tensors for parallel matrix multiplications. When batching sequences of different lengths, you pad shorter sequences with a special `[PAD]` token and use **attention masks** to prevent attending to padding positions.

Padding is prepended (left-pad) or appended (right-pad) depending on the model architecture and inference framework.

---

### 7. When to Freeze vs. Train the Tokenizer/Embeddings

| Scenario | What to do | Why |
|---|---|---|
| Lightweight SFT / RLHF (same vocab) | Freeze embeddings & tokenizer | Small changes don't need semantic drift |
| Domain adaptation (legal, medical) | Train embeddings | New jargon needs updated semantic geometry |
| Adding new special tokens or tags | Retrain tokenizer + resize embedding layer | New tokens need vocabulary slots and initialization |
| Vocab size change | Resize embedding layer + warmup new embeddings | Shape mismatch would crash; new entries start random |

**Practical pattern for new tokens:** Initialize new token embeddings as the mean of semantically related existing embeddings, then train. Cold-starting from random noise causes instability.

---

## ⚠️ Edge Cases & Constraints

**The "strawberry" problem:** `strawberry` is tokenized as `[straw, berry]`. The model can't look *inside* a token — it treats each token as atomic. So counting letters inside a token is hard. The letter `r` spans across two opaque integers, neither of which encodes its characters explicitly. Fix: character-tokenized prompts or models trained on byte-level representations.

**Tokenization and arithmetic:** `12345 + 67890` may tokenize as `[12, 345, +, 678, 90]` — misaligned digit groupings. Singh & Strouse (ICLR 2025) showed that **right-to-left tokenization** improves arithmetic accuracy by over 22 percentage points, because it aligns least-significant digits correctly.

**Multilingual inequity:** English text might tokenize to 10 tokens; equivalent Arabic text tokenizes to 17–34 tokens because the tokenizer was trained on an English-dominant corpus. More tokens = more attention FLOPs = higher API cost = slower inference. This is a **structural bias** baked into the vocabulary.

**Vocabulary size trade-off:** Larger vocab → shorter sequences (less attention cost) but bigger embedding tables. Tao et al. (NeurIPS 2024) showed a **log-linear relationship** between vocabulary size and training loss: there's an optimal vocab size that grows with model scale and compute budget.

**Tokenizer–model coupling:** You cannot swap tokenizers without retraining (or extensive re-initialization). The model's embedding matrix is literally indexed by token IDs — changing the mapping breaks everything. AutoTokenizer in HuggingFace enforces this coupling.

---

## 💻 Logical Code Snippet (Python)

```python
from transformers import AutoTokenizer
import torch

tokenizer = AutoTokenizer.from_pretrained("meta-llama/Meta-Llama-3-8B")

# ------ Encode → token IDs ------
text = "What is the capital of California?"
token_ids = tokenizer(text, return_tensors="pt")["input_ids"]
# tensor([[128000,  What,  is,  the,  capital,  of,  California,  ?]])

tokens = tokenizer.convert_ids_to_tokens(token_ids[0])
print(tokens)  # ['<|begin_of_text|>', 'What', ' is', ' the', ' capital', ' of', ' California', '?']

# ------ Decode → text ------
decoded = tokenizer.decode(token_ids[0])
print(decoded)  # '<|begin_of_text|>What is the capital of California?'

# ------ Batching with padding ------
batch = ["Short.", "This is a much longer sentence that needs more tokens."]
encoded = tokenizer(batch, padding=True, return_tensors="pt")
print(encoded["input_ids"].shape)   # (2, seq_len)  — padded to same length
print(encoded["attention_mask"])    # 0 where padding, 1 where real tokens

# ------ Temperature sampling intuition ------
import torch.nn.functional as F

logits = torch.tensor([2.0, 1.0, 0.5, 0.1])  # raw model outputs

def sample_with_temperature(logits, temperature=1.0):
    scaled = logits / temperature
    probs  = F.softmax(scaled, dim=-1)
    return torch.multinomial(probs, num_samples=1).item(), probs

idx_greedy, p_low  = sample_with_temperature(logits, T=0.1)  # almost always picks index 0
idx_random, p_high = sample_with_temperature(logits, T=2.0)  # much more uniform

# ------ Freezing embeddings in HuggingFace ------
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained("your-model")

# Freeze: embedding layer won't receive gradient updates
for param in model.model.embed_tokens.parameters():
    param.requires_grad = False

# Adding a new special token and resizing safely
tokenizer.add_special_tokens({"additional_special_tokens": ["<|tool_call|>"]})
model.resize_token_embeddings(len(tokenizer))
# New token row is randomly initialized — optionally warm-start it:
with torch.no_grad():
    model.model.embed_tokens.weight[-1] = \
        model.model.embed_tokens.weight[:-1].mean(dim=0)
```

---

## 🔬 Frontier Research & Papers

| Topic | Key Insight | Reference |
|---|---|---|
| **Byte Latent Transformer (BLT)** | Replaces tokenization entirely with **dynamic byte patching** based on next-byte entropy. Simple regions get large patches (cheap); complex regions get small patches (expensive). Matches Llama 3 at 8B with up to 50% fewer inference FLOPs. | Pagnoni et al., Meta / ACL 2025 |
| **SuperBPE** | Two-pass BPE that first learns subword tokens, then learns cross-word "superword" tokens spanning whitespace — achieves 33% sequence length reduction at 200K vocab sizes. | Liu et al., COLM 2025 |
| **BoundlessBPE** | Removes the word-boundary constraint from BPE merges entirely, achieving ~15% improvement in bytes-per-token and better Rényi efficiency. | COLM 2025 |
| **LiteToken** | Identifies and removes "intermediate merge residues" — tokens frequent during BPE training but rarely in final output (~10% of vocabulary). Reduces fragmentation, plug-and-play on existing tokenizers. | February 2026 |
| **Right-to-left tokenization** | Tokenizing digits right-to-left aligns least-significant digits, improving arithmetic accuracy by 22+ percentage points. | Singh & Strouse, ICLR 2025 |
| **Optimal vocabulary size scaling** | Log-linear relationship between vocab size and training loss; optimal vocab size grows with model scale. Explains the jump from 32K (Llama 2) → 128K (Llama 3) → 262K (Gemini). | Tao et al., NeurIPS 2024 |
| **ADAT (Dynamic tokenization)** | Iteratively refines vocabulary based on model feedback during training itself — the tokenizer co-evolves with the model. | NeurIPS 2024 |
| **Tokenization as design decision** | Argues that treating tokenizers as interchangeable defaults causes silent performance degradation in morphologically rich languages, code-switched text, and domain-specific corpora. Advocates principled tokenizer-model co-design. | ACL 2025 position paper |

---

## ❓ Active Recall

- [ ] Explain BPE in your own words: what are the inputs, the merge criterion, and the stopping condition?
- [ ] Why is Unigram LM (SentencePiece) often better than BPE for morphologically rich languages like Turkish or Finnish?
- [ ] What is the **embedding matrix**? Why is it trained rather than fixed, and how does the language model head relate to it?
- [ ] Explain the difference between **greedy decoding**, **temperature sampling**, and **beam search**. When would you prefer each?
- [ ] Why does tokenization cause the "strawberry letter counting" problem? How would byte-level tokenization fix it?
- [ ] What is the **vocabulary size trade-off**? What happens if you set $|V|$ too small or too large?
- [ ] You're fine-tuning a base LLM for a legal domain and need to add new acronyms as special tokens. Walk through exactly what steps you'd take (tokenizer, embedding layer, training strategy).
- [ ] Why is **multilingual tokenizer bias** a problem, and what structural property causes it?
- [ ] What is the Byte Latent Transformer's core innovation, and how does "dynamic patching based on entropy" differ from standard BPE?
- [ ] Why can you **not** simply swap the tokenizer of a pre-trained model without any additional training?
