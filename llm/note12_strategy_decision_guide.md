# 📄 Strategy Decision Guide: Prompt Engineering vs. Fine-tuning vs. RAG

**Tags:** #llm #prompt-engineering #fine-tuning #rag #strategy #decision #interview
**Links:** [[Prompt Engineering]], [[LLM Development Lifecycle]], [[Inference Optimization]]

---

## 🎯 The "Elevator Pitch"
> You have three main levers to make an LLM useful for a specific task: craft better prompts (free, instant), train the model on new data (expensive but powerful), or give it access to a retrieval system (flexible and updatable). Knowing *which* lever to pull — and when to combine them — is one of the most practical skills in applied LLM work.

---

## 🧠 Core Mechanics

### 1. The Three Approaches at a Glance

| Dimension | Prompt Engineering | Fine-tuning | RAG |
|---|---|---|---|
| **Changes model weights?** | ❌ No | ✅ Yes | ❌ No |
| **Requires labeled data?** | ❌ No | ✅ Yes | ❌ (just docs) |
| **Handles new/live info?** | ❌ No | ❌ No (static) | ✅ Yes |
| **Latency** | Longer prompt → higher | Lower (no prompt overhead) | Higher (retrieval step) |
| **Cost to implement** | ✅ Lowest | ❌ Highest | Medium |
| **Cost per query** | Medium–High (long prompts) | ✅ Lowest | Medium |
| **Source traceability** | ❌ No | ❌ No | ✅ Yes (cite chunks) |
| **Custom behavior/style** | Partial | ✅ Full control | ❌ Limited |

---

### 2. When to Choose Prompt Engineering (Q96)

**Use prompt engineering when:**
- ✅ You need to move fast — no data collection or training time
- ✅ The task is complex and open-ended, requiring the model's full breadth of general knowledge
- ✅ You have scarce or no labeled training data
- ✅ One LLM must handle many diverse tasks (no per-task model needed)
- ✅ You're prototyping and want to iterate in minutes, not days
- ✅ The base model already performs the task acceptably with good prompts

**Not suitable when:**
- ❌ The task requires knowledge the base model never saw during pre-training
- ❌ You need guaranteed output format consistency at high volume
- ❌ Long prompts become cost-prohibitive at scale (e.g., 4K-token system prompts × 1M queries/day)
- ❌ The model needs a fundamentally different tone, style, or persona that prompting alone can't achieve

**Analogy:** Prompt engineering is like giving detailed instructions to a smart generalist employee. It works well when the employee already has the skills — but can't teach them skills they fundamentally lack.

---

### 3. When to Choose Fine-tuning (Q100)

**Use fine-tuning when:**
- ✅ The task requires deep domain expertise the base model lacks (medical, legal, specialized code)
- ✅ You need consistent output style or format at scale — fine-tuning bakes this in
- ✅ Prompt engineering has hit a quality ceiling and you need the extra performance
- ✅ Low-latency, cost-efficient inference is critical (smaller prompts, faster responses)
- ✅ You have a high-volume use case where per-query prompt length is expensive
- ✅ You need behavior that's hard to elicit with prompts (e.g., specific refusal patterns, custom persona)

**Not suitable when:**
- ❌ Your knowledge needs to be updated frequently (fine-tuning captures a static snapshot)
- ❌ You have insufficient or low-quality training data
- ❌ You lack the GPU budget or engineering expertise for training
- ❌ Risk of catastrophic forgetting is unacceptable (use PEFT/LoRA to mitigate)

**Analogy:** Fine-tuning is like sending an employee to a specialized training course that changes how they think. After training, they're better at the target skill permanently — but the course is expensive and what they learned is frozen in time.

---

### 4. When to Choose RAG (Q97)

**Use RAG when:**
- ✅ Information changes frequently (news, product catalogs, regulatory documents, internal wikis)
- ✅ You need source attribution / citations — users need to verify answers
- ✅ You're working with proprietary information that shouldn't be baked into model weights
- ✅ You need to extend beyond the model's training cutoff
- ✅ You want to avoid the cost of retraining whenever knowledge updates

**Not suitable when:**
- ❌ Retrieval quality is low — bad retrieval → bad answers (garbage in, garbage out)
- ❌ Latency is extremely tight — the retrieval step adds 50–500ms
- ❌ The task requires deep reasoning across many connected concepts (RAG retrieves fragments, not holistic understanding)
- ❌ Documents don't exist — the knowledge needs to come from the model's internal representation

**Analogy:** RAG is like giving the employee a library card before answering — they can look things up in real time. Great for facts; less great if the task requires deep understanding built up over years of study.

---

### 5. RAG vs. Fine-tuning: Head-to-Head (Q97, Q98, Q99)

#### Limitations of RAG over Fine-tuning (Q98)
| Limitation | Details |
|---|---|
| **Retrieval dependency** | Output quality is bounded by retriever quality — wrong chunks → wrong answer |
| **No behavior change** | RAG doesn't change how the model reasons, writes, or refuses — only what it knows |
| **Added latency** | Retrieval step (embedding + vector search + fetch) adds 100–500ms per query |
| **Context fragmentation** | Retrieved chunks are disconnected — model may struggle to synthesize across many chunks |
| **No style/persona control** | Can't teach a model to write in a specific domain tone via RAG alone |

#### Limitations of Fine-tuning over RAG (Q99)
| Limitation | Details |
|---|---|
| **Static knowledge** | Knowledge is frozen at training time; can't access new information without retraining |
| **Expensive updates** | Any knowledge change requires a new fine-tuning run |
| **No source traceability** | Can't cite where a fact came from — a critical issue in regulated domains |
| **Catastrophic forgetting risk** | Learning new domain may overwrite general capabilities |
| **High upfront cost** | Data collection, training infrastructure, evaluation |
| **No "I don't know"** | Fine-tuned models can hallucinate confidently — RAG can respond "not found in documents" |

---

### 6. The Decision Flowchart

```
START: I need an LLM to do [task]
         │
         ▼
Does the base model already do this acceptably with a good prompt?
  YES → Use Prompt Engineering (iterate fast, no training cost)
  NO  → Continue ↓
         │
         ▼
Does the task require frequently updated or proprietary information?
  YES → Does it also need domain-specific reasoning style?
          YES → RAG + Fine-tuning combined
          NO  → RAG alone
  NO  → Continue ↓
         │
         ▼
Is the task domain-specific with static knowledge and high query volume?
  YES → Fine-tuning (task-specific or SFT)
  NO  → Use Few-shot Prompt Engineering or RAG
         │
         ▼
Is hardware/budget extremely constrained for fine-tuning?
  YES → QLoRA (consumer GPU fine-tuning)
  NO  → LoRA (standard PEFT) or Full Fine-tuning
```

---

### 7. Combining Approaches (The Real World)

In practice, the best production systems often layer all three:

```
[System Prompt]        ← Prompt Engineering: define persona, output format, tone
[Retrieved Context]    ← RAG: inject relevant live documents
[Task Instruction]     ← Prompt Engineering: specific task framing
      ↓
[Fine-tuned Model]     ← Fine-tuning: domain expertise baked in, consistent style
      ↓
[Structured Output]    ← Prompt Engineering: JSON/XML format enforcement
```

**Example: Enterprise medical Q&A system**
- Fine-tune on medical literature → domain reasoning
- RAG over hospital-specific guidelines → current, proprietary info
- System prompt → safe, formal clinical tone + "always recommend consulting a physician"

---

## ⚠️ Edge Cases & Constraints
- **Prompt engineering is not free at scale:** A 2K-token system prompt across 10M queries/day = 20 billion tokens/day in prompt overhead → significant cost.
- **RAG hallucination is different from fine-tuning hallucination:** RAG models may hallucinate by incorrectly synthesizing from retrieved chunks. Fine-tuned models may hallucinate from baked-in but wrong weights. Both need mitigation.
- **Fine-tuning on bad data is worse than no fine-tuning:** A fine-tuned model confidently wrong is more dangerous than a base model that expresses uncertainty.
- **"Fine-tune first" anti-pattern:** Many teams fine-tune prematurely before exhausting prompt engineering. Start with prompts, then fine-tune only when prompt engineering hits a ceiling.
- **Knowledge distillation as an alternative:** Sometimes the goal isn't task adaptation but model compression — distilling a large model's behavior into a smaller one. This is a separate use case from task-specific fine-tuning.

---

## 💻 Logical Code Snippet (Python)

```python
# ── Similarity-based RAG: retrieve then generate ──────────────────────
from sentence_transformers import SentenceTransformer
import numpy as np

# Offline: embed your document corpus
embedder = SentenceTransformer("all-MiniLM-L6-v2")
corpus = ["Document 1 text...", "Document 2 text...", "Document 3 text..."]
corpus_embeddings = embedder.encode(corpus)  # shape: [n_docs, 384]

def rag_retrieve(query: str, top_k: int = 3) -> list[str]:
    """Retrieve top-k most relevant documents for a query."""
    query_embedding = embedder.encode([query])  # shape: [1, 384]
    # Cosine similarity
    scores = np.dot(corpus_embeddings, query_embedding.T).flatten()
    top_indices = scores.argsort()[::-1][:top_k]
    return [corpus[i] for i in top_indices]

def rag_generate(query: str, llm) -> str:
    """Full RAG pipeline: retrieve → inject into prompt → generate."""
    retrieved_docs = rag_retrieve(query, top_k=3)
    context = "\n\n".join(f"[Doc {i+1}]: {doc}" for i, doc in enumerate(retrieved_docs))

    prompt = f"""Answer the question using ONLY the provided documents.
If the answer is not in the documents, say "Not found in provided documents."

Documents:
{context}

Question: {query}
Answer:"""

    return llm.generate(prompt)


# ── Decision heuristic: estimate if fine-tuning is justified ─────────
def should_fine_tune(
    prompt_engineering_accuracy: float,   # 0.0 - 1.0
    daily_query_volume: int,
    avg_prompt_tokens: int,
    token_cost_per_1k: float = 0.001,     # USD
    fine_tune_cost_usd: float = 500.0,    # one-time
    target_accuracy: float = 0.90,
) -> dict:
    # Is there a performance gap to close?
    needs_improvement = prompt_engineering_accuracy < target_accuracy

    # Daily prompt cost
    daily_token_cost = (daily_query_volume * avg_prompt_tokens / 1000) * token_cost_per_1k
    # Days to break even (fine-tuning reduces prompt length → lower per-query cost)
    break_even_days = fine_tune_cost_usd / (daily_token_cost * 0.5)  # assume 50% prompt reduction

    return {
        "recommend_fine_tuning": needs_improvement or break_even_days < 30,
        "performance_gap": target_accuracy - prompt_engineering_accuracy,
        "break_even_days": round(break_even_days, 1),
        "daily_prompt_cost_usd": round(daily_token_cost, 2),
    }

# Example
print(should_fine_tune(
    prompt_engineering_accuracy=0.78,
    daily_query_volume=100_000,
    avg_prompt_tokens=800,
))
# → {'recommend_fine_tuning': True, 'performance_gap': 0.12, 'break_even_days': 3.1, ...}
```

---

## ❓ Active Recall
- [ ] What are the three main strategies for adapting an LLM to a task? What does each change (or not change)?
- [ ] When should you prefer prompt engineering over fine-tuning?
- [ ] When should you use fine-tuning over prompt engineering?
- [ ] What are the two key advantages of RAG that fine-tuning cannot replicate?
- [ ] What are the two key advantages of fine-tuning that RAG cannot replicate?
- [ ] What is the main limitation of RAG when retrieval quality is poor?
- [ ] What is the "fine-tune first" anti-pattern?
- [ ] Walk through the decision flowchart for choosing between the three approaches.
- [ ] Give an example of a real system that would benefit from combining all three approaches.
- [ ] **Follow-up:** What is "retrieval-augmented fine-tuning" (RAFT)?
- [ ] **Follow-up:** What is knowledge distillation, and when is it preferred over fine-tuning?
- [ ] **Follow-up:** How does long-context LLM capability (e.g., 1M token window) affect the RAG vs. prompt engineering trade-off?
