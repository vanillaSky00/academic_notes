# 📄 LSTM Networks — Long Short-Term Memory

**Tags:** #deep-learning #sequence-modeling #RNN #LSTM #NLP #time-series
**Links:** [[Recurrent Neural Networks]], [[Gradient Descent]], [[Transformers]], [[GRU]], [[Backpropagation Through Time]]
**Source:** Colah's Blog (2015) + enriched with research papers and interview insights

---

## 🎯 The "Elevator Pitch"

> An LSTM is a special RNN that uses learned gates to selectively **remember or forget** information over long sequences — solving the core failure of vanilla RNNs where older memories fade away during training.

**Simple Analogy 🎬:** Imagine reading a mystery novel. A good reader remembers who committed the crime from Chapter 1 even when they're on Chapter 20. A bad reader (vanilla RNN) forgets Chapter 1 by Chapter 5. LSTM is the good reader — it has a **notepad (cell state)** it can write to, erase from, and selectively reveal at the right moment.

---

## 🧠 Core Mechanics

### Why Do We Need LSTMs At All? — The Vanishing Gradient Problem

- **Vanilla RNN** processes sequences step-by-step, passing a **hidden state** $h_t$ forward in time.
- During training (backpropagation through time, BPTT), gradients are multiplied at every timestep: $\frac{\partial L}{\partial W} = \prod_{t=1}^{T} \frac{\partial h_t}{\partial h_{t-1}}$
- Each of those partial derivatives involves sigmoid/tanh activations whose outputs lie in $(0, 1)$. **Multiplying many small numbers → gradient → 0**. Earlier layers stop learning. This is the **vanishing gradient problem**.
- **Analogy 🔦:** Imagine a game of telephone across 50 people. The original message ("She is a doctor") becomes garbled to nothing by the end. The RNN "hears" the final whisper but has lost the original signal.

### The LSTM Solution — Cell State as a Conveyor Belt

The key innovation: introduce a **cell state** $C_t$ — a separate memory track that flows relatively unchanged through the sequence. Unlike the hidden state $h_t$, it only gets modified via **additive updates**, not repeated multiplications. This prevents gradient vanishing.

**Conveyor Belt Analogy 🏭:** The cell state $C_t$ is a conveyor belt running through a factory. Workers (gates) can add boxes to the belt, remove boxes, or select which boxes to inspect. The belt itself is never crushed or squeezed — it just rolls along.

---

### The Three Gates (The Factory Workers)

Each gate = sigmoid layer (outputs 0–1) × pointwise multiplication. **0 = block everything, 1 = let everything through.**

#### 1. 🗑️ Forget Gate — "What should I discard?"

$$f_t = \sigma(W_f \cdot [h_{t-1}, x_t] + b_f)$$

- Looks at the previous hidden state $h_{t-1}$ and current input $x_t$.
- Outputs a value in $(0, 1)$ for each element of the cell state.
- **1 = keep this memory, 0 = erase this memory**.
- **Example:** Reading "The bank by the river" → when you hit "river," you forget the "financial institution" context of "bank" and update the meaning.

#### 2. 📝 Input Gate — "What new info should I write?"

$$i_t = \sigma(W_i \cdot [h_{t-1}, x_t] + b_i)$$
$$\tilde{C}_t = \tanh(W_C \cdot [h_{t-1}, x_t] + b_C)$$

- Two-part step:
  - **Input gate** ($i_t$, sigmoid): Decides *which* values to update.
  - **Candidate values** ($\tilde{C}_t$, tanh): Proposes *what* to write.
- **Example:** When you learn someone's new name, you write it to memory.

#### 3. 🔄 Cell State Update — "Actually update the notepad"

$$C_t = f_t * C_{t-1} + i_t * \tilde{C}_t$$

- **Erase** old info via $f_t$ (multiply → some memories fade).
- **Add** new info via $i_t * \tilde{C}_t$.
- **Critical insight:** This is an **additive** operation — gradients flow through addition cleanly (derivative = 1), which is why the vanishing gradient is mitigated. The forget gate $f_t$ allows the network to *learn* when to let gradients flow vs. when to stop them.

#### 4. 📤 Output Gate — "What should I reveal right now?"

$$o_t = \sigma(W_o \cdot [h_{t-1}, x_t] + b_o)$$
$$h_t = o_t * \tanh(C_t)$$

- Filters the cell state through tanh (maps to $(-1, 1)$) then masks via $o_t$.
- Produces the **hidden state** $h_t$ — the output you actually use for predictions.
- **Example:** If you just read a subject noun, you might "output" grammatical information (singular/plural) ready for the next verb.

---

### Summary: Two Memory Streams

| Stream | Name | Purpose | Lives Long? |
|---|---|---|---|
| $C_t$ | **Cell State** | Long-term episodic memory | ✅ Yes — conveyor belt |
| $h_t$ | **Hidden State** | Short-term working memory / output | ❌ Fades faster |

---

## 🌳 The Sequence Modeling Family Tree

Understanding *why* LSTM, and not something else, requires knowing the whole landscape:

```
Sequence Modeling Methods
│
├── Traditional: HMM, CRF (hand-crafted features, limited capacity)
│
├── Recurrent Neural Networks
│   ├── Vanilla RNN         → Suffers vanishing gradient. Good only for very short deps.
│   ├── LSTM (1997)         → Gated cell state. Long-range memory. More params.
│   └── GRU  (2014)         → Simpler LSTM. Combines forget+input gates. Faster.
│
└── Attention-based
    └── Transformer (2017)  → No recurrence. Full parallelism. O(n²) attention. 
                              Needs positional encoding. Dominates NLP at scale.
```

---

## ⚔️ LSTM vs Transformer — When To Use Which?

This is the most important architectural decision in modern sequence modeling.

### How Transformers Work (Brief)

Instead of processing one step at a time, Transformers process the **entire sequence at once** using **self-attention**: every token attends to every other token, computing relevance scores.

**Analogy 📬:** LSTM reads a letter word by word, remembering as it goes. Transformer lays out all the words on a table simultaneously and draws arrows between every pair of related words, all at once.

### Head-to-Head Comparison

| Dimension | LSTM | Transformer |
|---|---|---|
| **Processing** | Sequential (step-by-step) | Parallel (all at once) |
| **Long-range deps** | Weakens over very long sequences | Handles directly via attention |
| **Training speed** | Slow (can't parallelize) | Fast on GPUs/TPUs |
| **Data hunger** | Works on small datasets | Needs large data to shine |
| **Memory** | O(n) — scales linearly | O(n²) — attention is quadratic |
| **Interpretability** | Harder (hidden state is a black box) | Attention weights = interpretable |
| **Streaming / real-time** | ✅ Natural fit (processes one token at a time) | ❌ Awkward (needs full context) |
| **Inductive bias** | Strong sequential / temporal bias built-in | No sequential bias (needs positional encoding) |
| **When to pick** | Small data, real-time, time series with local patterns | Large NLP, long-range global context tasks |

### Research Evidence

- Greff et al. (2015) compared popular LSTM variants and found performance differences were **minimal** — the base LSTM architecture is quite robust.
- Jozefowicz et al. (2015) tested 10,000+ RNN architectures, finding some that **outperformed standard LSTMs** on specific tasks.
- Vaswani et al. (2017) — *"Attention Is All You Need"* — showed Transformers dominate at scale for NLP.
- In financial time series (Ruiru et al. 2024, IJCCI), LSTM showed **more consistent results** while Transformers had higher variance with fewer parameters.
- For short-horizon univariate forecasting, both models perform **similarly** — the dataset complexity matters more than the architecture.

### ✅ Decision Guide: LSTM vs Transformer

```
Is your dataset small (< tens of thousands of samples)?
    YES → Use LSTM (or GRU)
    NO  → Consider Transformer

Do you need real-time/streaming predictions?
    YES → Use LSTM (processes causally, one step at a time)
    NO  → Either works

Is the sequence very long (> 1000 tokens) with global dependencies?
    YES → Transformer (attention spans the whole sequence)
    NO  → LSTM is fine and cheaper

Do you have limited compute?
    YES → LSTM or GRU (linear memory, lower parameter count)
    NO  → Transformer scales better with scale
```

---

## 🔧 LSTM Variants Worth Knowing

### GRU (Gated Recurrent Unit) — Cho et al. 2014
- Merges forget + input gates into a single **update gate** $z_t$.
- Also merges cell state + hidden state into one stream.
- **Fewer parameters, faster to train**, comparable performance.
- **Formula:**
  $$z_t = \sigma(W_z \cdot [h_{t-1}, x_t]) \quad \text{(update gate)}$$
  $$r_t = \sigma(W_r \cdot [h_{t-1}, x_t]) \quad \text{(reset gate)}$$
  $$h_t = (1-z_t)*h_{t-1} + z_t*\tanh(W \cdot [r_t * h_{t-1}, x_t])$$

### Peephole Connections — Gers & Schmidhuber 2000
- Gate layers can "peek" at the current **cell state** $C_{t-1}$ directly, not just $h_{t-1}$.
- Useful when precise timing matters (e.g., counting beats in music).

### Bidirectional LSTM (BiLSTM)
- Run two LSTMs: one forward, one backward over the sequence.
- Concatenate hidden states: each output has context from **both past and future**.
- Great for text classification, NER — any task where you have the full sequence at inference time.
- **Not suitable for real-time/causal generation**.

### Stacked (Deep) LSTM
- Multiple LSTM layers stacked vertically — output of layer $l$ is input to layer $l+1$.
- Typical: 2–3 layers. A single layer with 100–200 hidden units is a good starting point.

---

## 💻 Logical Code Snippet (PyTorch)

```python
import torch
import torch.nn as nn

# ── Manual Gate Math (Conceptual, Single Timestep) ──────────────────────
def lstm_step_manual(x_t, h_prev, c_prev, W_f, W_i, W_c, W_o, b_f, b_i, b_c, b_o):
    """
    One forward step of an LSTM cell.
    x_t:    input at timestep t,  shape [input_size]
    h_prev: hidden state (h_{t-1}), shape [hidden_size]
    c_prev: cell state  (C_{t-1}),  shape [hidden_size]
    """
    combined = torch.cat([h_prev, x_t], dim=0)      # Concatenate h and x

    # Step 1 — Forget Gate: "What to erase from cell state?"
    f = torch.sigmoid(W_f @ combined + b_f)          # values in (0,1)

    # Step 2 — Input Gate + Candidate: "What new info to write?"
    i = torch.sigmoid(W_i @ combined + b_i)          # gate: which positions
    c_tilde = torch.tanh(W_c @ combined + b_c)       # candidates: what values

    # Step 3 — Cell State Update: additive (NOT multiplicative → no vanishing!)
    c_t = f * c_prev + i * c_tilde

    # Step 4 — Output Gate: "What to expose as h_t?"
    o = torch.sigmoid(W_o @ combined + b_o)
    h_t = o * torch.tanh(c_t)                        # filter cell state through tanh

    return h_t, c_t   # new hidden state, new cell state


# ── Practical PyTorch LSTM ───────────────────────────────────────────────
class SequenceClassifier(nn.Module):
    """
    LSTM that reads a variable-length sequence and classifies it.
    E.g., sentiment analysis: 'This movie was great' → positive
    """
    def __init__(self, vocab_size, embed_dim, hidden_size, num_classes, num_layers=2):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.lstm = nn.LSTM(
            input_size=embed_dim,
            hidden_size=hidden_size,
            num_layers=num_layers,
            batch_first=True,         # input shape: (batch, seq_len, features)
            dropout=0.3,              # regularize between stacked layers
            bidirectional=False       # set True for BiLSTM
        )
        self.classifier = nn.Linear(hidden_size, num_classes)

    def forward(self, token_ids):
        # token_ids: (batch, seq_len)
        x = self.embedding(token_ids)            # → (batch, seq_len, embed_dim)
        
        # lstm returns: output (all h_t), (h_n, c_n) — final hidden & cell states
        out, (h_n, c_n) = self.lstm(x)
        
        # Use the FINAL hidden state as the sequence summary
        last_hidden = h_n[-1]                    # → (batch, hidden_size)
        logits = self.classifier(last_hidden)    # → (batch, num_classes)
        return logits


# ── GRU (simpler, often comparable to LSTM) ─────────────────────────────
class GRUClassifier(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_size, num_classes):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.gru = nn.GRU(embed_dim, hidden_size, batch_first=True)  # no cell state!
        self.classifier = nn.Linear(hidden_size, num_classes)

    def forward(self, x):
        x = self.embedding(x)
        _, h_n = self.gru(x)       # GRU: no c_n, only h_n
        return self.classifier(h_n.squeeze(0))


# ── Comparing LSTM vs Transformer for time series ───────────────────────
# LSTM approach:
lstm_model = nn.LSTM(input_size=1, hidden_size=64, num_layers=2, batch_first=True)

# Transformer approach (encoder-only for regression/forecasting):
encoder_layer = nn.TransformerEncoderLayer(d_model=64, nhead=4, batch_first=True)
transformer_model = nn.TransformerEncoder(encoder_layer, num_layers=2)
# Key difference: transformer processes all timesteps simultaneously — parallelizable!
# LSTM processes one timestep at a time — inherently sequential.
```

---

## ⚠️ Edge Cases & Constraints

- **LSTM still struggles with very long sequences (>1000 timesteps):** Even with the cell state, the forget gate can become dominant and override earlier memories. This is where Transformers genuinely win.
- **Exploding gradients:** LSTMs mitigate vanishing gradients but NOT exploding ones. Solution: **gradient clipping** (`torch.nn.utils.clip_grad_norm_`).
- **Bidirectional LSTMs can't be used for autoregressive generation** (e.g., GPT-style text generation): the backward pass looks into the "future," which is not available at inference time.
- **GRU is preferred when data is small or compute is limited** — comparable performance with less complexity.
- **LSTM has ~4× more parameters than a vanilla RNN** for the same hidden size (due to 4 weight matrices for each gate).
- **Transformers with quadratic attention complexity** struggle with very long sequences (e.g., genomics with 100k+ tokens) — this is why Mamba (state space models) and linear attention variants have emerged.
- **Common misconception:** LSTMs don't fully "solve" the vanishing gradient — they make it *learnable*. The forget gate can still learn to vanish gradients if that's optimal for the task.

---

## 🔬 Key Papers to Know

| Paper | Year | What It Did |
|---|---|---|
| Hochreiter & Schmidhuber | 1997 | Original LSTM paper — introduced cell state + input/output gates |
| Gers, Schmidhuber & Cummins | 1999 | Added the **forget gate** — the version we use today |
| Gers & Schmidhuber | 2000 | Added **peephole connections** |
| Cho et al. | 2014 | Introduced **GRU** as a simpler LSTM alternative |
| Greff et al. | 2015 | Compared 8 LSTM variants — differences were minor |
| Vaswani et al. | 2017 | *"Attention Is All You Need"* — Transformer architecture |

---

## ❓ Active Recall (Interview-Grade Questions)

### Conceptual
- [ ] What is the vanishing gradient problem in vanilla RNNs, and *why* does multiplying activations cause it?
- [ ] How does the cell state in LSTM prevent the vanishing gradient compared to the hidden state in RNNs?
- [ ] Why is the additive cell state update ($C_t = f_t * C_{t-1} + i_t * \tilde{C}_t$) critical for gradient flow?
- [ ] Explain the role of each gate (forget, input, output) using a concrete analogy.
- [ ] What is the difference between the **cell state** $C_t$ and the **hidden state** $h_t$? Why do we need both?

### Architecture Decisions
- [ ] When would you choose LSTM over a Transformer? Give 3 concrete scenarios.
- [ ] What is a GRU, and how does it differ from LSTM? When would you prefer it?
- [ ] What is a BiLSTM and why can't you use it for autoregressive text generation?
- [ ] How does stacking LSTM layers (deep LSTM) improve performance?
- [ ] What is a peephole connection and when might it help?

### Practical / Interview Traps
- [ ] Does LSTM solve the **exploding** gradient problem? How do you handle it in practice?
- [ ] Why does LSTM still struggle with sequences > 1000 steps long?
- [ ] If you're building a real-time speech recognition system, would you use LSTM or Transformer? Why?
- [ ] Explain what happens if the forget gate always outputs 1. What about always 0?
- [ ] What's the computational complexity of self-attention in Transformers vs LSTM's processing of a sequence of length $n$?

### Code/Math
- [ ] Write the 4 LSTM equations from memory (forget gate, input gate, candidate values, output gate).
- [ ] In PyTorch, what does `nn.LSTM(batch_first=True)` return? What are `h_n` and `c_n`?
- [ ] What is the shape of the weight matrix $W_f$ in a forget gate if the input size is 10 and hidden size is 50?

---

## 💡 Mental Model Summary

```
Vanilla RNN:
  h_t = tanh(W · [h_{t-1}, x_t])          ← one stream, vanishes over time

LSTM:
  f_t = σ(...)                              ← forget: erase stale memory
  i_t = σ(...), C̃_t = tanh(...)           ← input: write new memory
  C_t = f_t * C_{t-1} + i_t * C̃_t        ← ADDITIVE UPDATE: gradients flow!
  o_t = σ(...), h_t = o_t * tanh(C_t)      ← output: expose relevant summary

GRU:
  z_t = σ(...)                              ← update gate (forget + input merged)
  h_t = (1-z_t)*h_{t-1} + z_t*tanh(...)   ← simpler, one stream

Transformer:
  Attention(Q,K,V) = softmax(QK^T/√d) · V  ← every token attends to every other
  No recurrence → fully parallel → dominant at scale
```
