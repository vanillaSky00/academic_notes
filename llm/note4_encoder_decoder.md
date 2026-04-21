# 📄 Encoder-Decoder Architecture in Transformers

**Tags:** #llm #encoder #decoder #cross-attention #transformer #interview
**Links:** [[Multi-Head Attention & Masking]], [[Self-Attention Mechanics]], [[Training Stability]]

---

## 🎯 The "Elevator Pitch"
> The encoder reads the entire input at once and builds a rich understanding of it. The decoder then uses that understanding — plus what it's already generated — to produce the output one token at a time.

---

## 🧠 Core Mechanics

### 1. The Encoder (Q16)

- **Purpose:** Process the full input sequence and produce **context-aware representations** for every input token.
- **What it does:**
  - Applies **multi-head self-attention** (bidirectional — every token sees every other token)
  - Applies a **feed-forward neural network** (FFN) on top of the attention output
  - Both sublayers use **residual connections** and **layer normalization**
- **Output:** A sequence of rich, context-aware vectors (one per input token) called the **encoded state** — passed to every decoder layer via cross-attention.

**Analogy:** The encoder is like a scholar who reads an entire French document deeply before attempting to translate it.

---

### 2. The Decoder (Q17)

- **Purpose:** Generate the output sequence **token by token**, conditioned on:
  1. Previously generated output tokens (autoregressive)
  2. The encoder's final contextual representations (cross-attention)

- **Each decoder layer has three sublayers:**
  1. **Masked self-attention** — attends to previously generated tokens only (causal mask prevents peeking ahead)
  2. **Cross-attention (encoder-decoder attention)** — allows the decoder to look at the encoder's output
  3. **Feed-forward network (FFN)** — position-wise transformation

---

### 3. Cross-Attention: How Encoder and Decoder Talk (Q26)

**Why it's called "cross"-attention:** The queries and the keys/values come from *different* sequences (decoder vs. encoder), crossing the boundary between the two.

| Vector | Source |
|---|---|
| **Query (Q)** | Decoder's previous layer output |
| **Key (K)** | Encoder's final output |
| **Value (V)** | Encoder's final output |

- The decoder query asks: "Given what I've generated so far, which parts of the input should I focus on?"
- The encoder keys/values respond: "Here's the full input context — weigh it accordingly."

**Contrast with encoder self-attention:**
- Encoder self-attention: Q, K, V all from the *same* input sequence (internal dependencies)
- Cross-attention: Q from decoder, K and V from encoder (inter-sequence linking)

---

### 4. The Full Encoder-Decoder Flow (Q18)

```
Input Sequence
     ↓
[Embedding + Positional Encoding]
     ↓
┌─────────────────────────────┐
│  ENCODER (×N layers)        │
│  ┌──────────────────────┐   │
│  │ Multi-Head Self-Attn  │   │
│  │ + Residual + LayerNorm│   │
│  ├──────────────────────┤   │
│  │ Feed-Forward Network  │   │
│  │ + Residual + LayerNorm│   │
│  └──────────────────────┘   │
└─────────────────────────────┘
     ↓ Encoded State (Keys, Values)
     
Output (so far) → [Embedding + Positional Encoding]
     ↓
┌─────────────────────────────┐
│  DECODER (×N layers)        │
│  ┌──────────────────────┐   │
│  │ Masked Self-Attention │   │
│  │ + Residual + LayerNorm│   │
│  ├──────────────────────┤   │
│  │ Cross-Attention       │ ← Encoder State (K, V)
│  │ + Residual + LayerNorm│   │
│  ├──────────────────────┤   │
│  │ Feed-Forward Network  │   │
│  │ + Residual + LayerNorm│   │
│  └──────────────────────┘   │
└─────────────────────────────┘
     ↓
Linear + Softmax → Next Token Probability
```

---

### 5. Architecture Variants in Modern LLMs

| Architecture | Example Models | Notes |
|---|---|---|
| Encoder-Decoder | T5, BART, original Transformer | Best for seq2seq (translation, summarization) |
| Encoder-only | BERT, RoBERTa | Best for understanding tasks (classification, NER) |
| Decoder-only | GPT, LLaMA, Claude | Best for generation; uses only masked self-attention |

- **Decoder-only models** have no encoder-decoder cross-attention — they attend only to their own past tokens. This simplifies architecture and scales well for large language models.

---

## ⚠️ Edge Cases & Constraints
- **Encoder-decoder overhead:** Running both encoder and decoder at inference is more expensive than decoder-only. For open-ended generation, decoder-only is now dominant.
- **Cross-attention bottleneck:** Every decoder layer must attend over the entire encoder output — for very long inputs this is expensive even though encoder output is fixed.
- **Bidirectional vs. causal:** Encoder self-attention is bidirectional (sees all tokens); this makes encoders unsuitable for left-to-right generation without masking.
- **Common misconception:** Cross-attention is sometimes confused with the decoder's masked self-attention. They are *different sublayers*: masked self-attention handles the decoder's own generated sequence, cross-attention links to the encoder.

---

## 💻 Logical Code Snippet (Python)

```python
import torch.nn as nn

class TransformerDecoderLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff):
        super().__init__()
        # Sublayer 1: Masked self-attention (causal)
        self.masked_self_attn = nn.MultiheadAttention(d_model, num_heads, batch_first=True)
        self.norm1 = nn.LayerNorm(d_model)

        # Sublayer 2: Cross-attention (encoder-decoder)
        self.cross_attn = nn.MultiheadAttention(d_model, num_heads, batch_first=True)
        self.norm2 = nn.LayerNorm(d_model)

        # Sublayer 3: Feed-forward network
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_ff),
            nn.ReLU(),
            nn.Linear(d_ff, d_model)
        )
        self.norm3 = nn.LayerNorm(d_model)

    def forward(self, decoder_input, encoder_output, causal_mask):
        # Sublayer 1: Masked self-attention
        attn_out, _ = self.masked_self_attn(
            decoder_input, decoder_input, decoder_input,
            attn_mask=causal_mask  # prevents attending to future tokens
        )
        x = self.norm1(decoder_input + attn_out)  # residual

        # Sublayer 2: Cross-attention (Q from decoder, K/V from encoder)
        cross_out, _ = self.cross_attn(
            query=x,
            key=encoder_output,
            value=encoder_output
        )
        x = self.norm2(x + cross_out)  # residual

        # Sublayer 3: Feed-forward
        ff_out = self.ffn(x)
        x = self.norm3(x + ff_out)  # residual

        return x
```

---

## ❓ Active Recall
- [ ] What is the purpose of the encoder? What does it output?
- [ ] What are the three sublayers in a Transformer decoder layer?
- [ ] Why is masked self-attention used in the decoder but not the encoder?
- [ ] In cross-attention, where do Q, K, and V come from?
- [ ] How does the decoder "know" what the encoder read?
- [ ] What is the difference between encoder-only, decoder-only, and encoder-decoder Transformers? Give a model example for each.
- [ ] **Follow-up:** Why have decoder-only models (GPT, LLaMA) become dominant for large-scale LLMs?
- [ ] **Follow-up:** In a decoder-only model like GPT, there is no encoder. How does the model handle tasks like translation or summarization?
