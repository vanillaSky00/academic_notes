# 📄 Tokenization & Embeddings in LLMs

**Tags:** #llm #tokenization #embeddings #interview
**Links:** [[Transformer Architecture]], [[Self-Attention]], [[Positional Embeddings]]

---

## 🎯 The "Elevator Pitch"
> Before a language model can understand text, it has to convert raw words into numbers. Tokenization chops text into chunks; embeddings turn those chunks into rich numerical vectors the model can actually compute with.

---

## 🧠 Core Mechanics

### 1. What is Tokenization? (Q12)
- **Definition:** The process of splitting raw text into discrete units called **tokens** (subwords, words, or characters), then mapping each token to an integer ID from the model's vocabulary.
- **Why it's necessary:** Neural networks cannot process raw strings — they require numerical input. Tokenization is the bridge between human-readable text and the mathematical world of matrices.
- **Pipeline:** `Raw text → tokens → integer IDs → embeddings`

---

### 2. Subword Tokenization vs. Word-Level Tokenization (Q7)
- **Word-level problem:** Vocabulary explodes to millions of entries; rare or unseen words (OOV words) have no representation.
- **Subword tokenization (e.g., Byte-Pair Encoding / BPE):**
  - Breaks unknown words into known subword units → **no OOV problem**
  - Smaller, manageable vocabulary size
  - Lets the model learn **morphology** (prefixes, suffixes, roots)
  - Example: `"unhappiness"` → `["un", "happi", "ness"]`

| Approach | Vocab Size | OOV Handling | Morphology Awareness |
|---|---|---|---|
| Word-level | Huge | ❌ Fails | ❌ |
| Subword (BPE) | Moderate | ✅ Splits OOV | ✅ |
| Character-level | Tiny | ✅ | Minimal |

---

### 3. Trade-offs of Large Vocabulary Size (Q8)
| Factor | Large Vocabulary | Small Vocabulary |
|---|---|---|
| Rare word coverage | ✅ Better | ❌ More fragmentation |
| Embedding layer size | ❌ Larger | ✅ Smaller |
| Softmax cost | ❌ More expensive | ✅ Cheaper |
| Training efficiency | ❌ Slower | ✅ Faster |

- **Bottom line:** Models target a sweet spot (~30K–100K tokens) to balance linguistic coverage with computational efficiency.

---

### 4. The Embedding Layer (Q6, Q13, Q14)
- **Definition:** A lookup table (the **embedding matrix** of shape `[vocab_size × d_model]`) that converts integer token IDs into dense, continuous vectors.
- **Mechanism:**
  1. Token ID is used as an index into the embedding matrix
  2. The corresponding row vector is retrieved → this is the **token embedding**
  3. Token embeddings are **combined with positional embeddings** before entering the Transformer layers
- **What embeddings encode:** Semantic meaning + syntactic role of each token (e.g., "king" and "queen" are close in embedding space)
- **Note:** The embedding layer is shared with the output projection layer in many architectures (weight tying).

---

## 🔗 Analogy
Think of tokenization like breaking a sentence into LEGO bricks (tokens), and embeddings as painting each brick a unique color that encodes its meaning. The Transformer then looks at the arrangement of colored bricks to understand the sentence.

---

## ⚠️ Edge Cases & Constraints
- **BPE is greedy:** It may split words in linguistically unintuitive ways, especially in non-English languages.
- **Tokenization is model-specific:** GPT-4 and LLaMA use different tokenizers; you can't mix them.
- **Embeddings are learned, not fixed:** They update during training via backpropagation.
- **Rare tokens get sparse gradients:** In large vocabularies, infrequent tokens see fewer updates → weaker representations.
- **Common misconception:** Embeddings ≠ one-hot vectors. One-hot is the intermediate step; the embedding matrix *projects* the one-hot into a dense space.

---

## 💻 Logical Code Snippet (Python)

```python
import torch
import torch.nn as nn

# Vocabulary size and embedding dimension
vocab_size = 50_000
d_model = 512

# Embedding matrix: shape [vocab_size, d_model]
embedding_layer = nn.Embedding(vocab_size, d_model)

# Step 1: Tokenize text → integer IDs (simplified)
tokens = ["The", "cat", "sat"]
token_ids = [464, 3797, 3332]  # hypothetical vocab indices

# Step 2: Convert to tensor
token_id_tensor = torch.tensor(token_ids)  # shape: [seq_len]

# Step 3: Lookup embeddings
token_embeddings = embedding_layer(token_id_tensor)  # shape: [seq_len, d_model]

# Step 4: Add positional embeddings (see Positional Embeddings note)
# final_embeddings = token_embeddings + positional_embeddings
```

---

---

## 🔬 Deeper Dive: OOV Handling & Tokenization Algorithms (Q111)

### How Subword Methods Handle OOV Words

**The OOV problem:** Any word not in the model's fixed vocabulary is "out-of-vocabulary." Word-level models must mark these as `<UNK>` — losing all information about the word.

**Subword solution:** Decompose unknown words into known subword pieces. Since the subword vocabulary contains characters and common morphemes, *any* word can be represented.

```
Unknown word: "serendipitously"
BPE split:    ["ser", "end", "ip", "ito", "us", "ly"]
              ← all subwords are in vocabulary → no information loss
```

**Three dominant subword tokenization algorithms:**

| Algorithm | Used By | Key Mechanism |
|---|---|---|
| **BPE** (Byte-Pair Encoding) | GPT-2, GPT-4, LLaMA, Claude | Iteratively merge most frequent adjacent token pairs from a character-level start |
| **WordPiece** | BERT, DistilBERT | Similar to BPE but selects merges that maximize training data likelihood |
| **SentencePiece** | T5, LLaMA-2, Gemma | Language-agnostic; works directly on raw text (no pre-tokenization needed); supports BPE or Unigram internally |

**BPE algorithm (simplified):**
```
Start:      vocabulary = individual characters
            corpus = all training text split into characters

Repeat until vocab_size is reached:
  1. Count all adjacent token pairs in the corpus
  2. Find the most frequent pair (e.g., "e" + "r" → "er")
  3. Merge that pair everywhere in the corpus
  4. Add the merged token to the vocabulary

Result:     common words → single tokens ("the", "and")
            rare words → multiple subword tokens ("un" + "happi" + "ness")
```

**Why this is clever:** The algorithm naturally allocates vocabulary slots to the most useful units — common words become atomic tokens, rare words decompose into reusable morphemes.

### Tokenization Artifacts to Know for Interviews

- **Tokenization is language-dependent:** English-trained tokenizers are inefficient for Chinese/Japanese — a single Chinese character may tokenize into multiple BPE tokens, making context windows effectively shorter for non-Latin scripts.
- **Numbers are often fragmented:** "2024" might tokenize as ["20", "24"] or ["2", "0", "2", "4"] — this is one reason LLMs struggle with arithmetic.
- **Capitalization affects tokenization:** "Hello" and "hello" may map to different tokens — even though semantically identical.
- **Leading spaces matter:** In BPE, "cat" and " cat" (with a space) are often different tokens — the space is absorbed into the token.

---

## ❓ Active Recall
- [ ] Why can't LLMs process raw text directly? What does tokenization solve?
- [ ] What is an OOV word, and how does BPE handle it?
- [ ] Walk through the BPE algorithm step by step.
- [ ] What are the three main subword tokenization algorithms? Which model uses each?
- [ ] What are the trade-offs between large and small vocabularies?
- [ ] How does the embedding layer work mechanically? What is its shape?
- [ ] What is "weight tying" in the context of the embedding layer?
- [ ] Why are rare tokens potentially weaker than common ones, even with subword tokenization?
- [ ] Why do LLMs tend to struggle more with non-Latin scripts than English?
- [ ] **Follow-up:** If a model has vocab size 32K and d_model=768, how many parameters does the embedding matrix have?
- [ ] **Follow-up:** What is SentencePiece, and why is it "language-agnostic"?
- [ ] **Follow-up:** Why do numbers tokenize poorly, and how does this affect LLM arithmetic performance?
