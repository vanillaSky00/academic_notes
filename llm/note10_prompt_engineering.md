# 📄 Prompt Engineering — Techniques, Strategies & Best Practices

**Tags:** #llm #prompt-engineering #cot #few-shot #zero-shot #react #in-context-learning #interview
**Links:** [[LLM Development Lifecycle]], [[Inference Optimization]], [[Decoding Strategies]]

---

## 🎯 The "Elevator Pitch"
> A prompt is the only knob a user has to steer an LLM without touching its weights. Prompt engineering is the craft of pulling the best possible performance out of a fixed model purely through how you write the input — the difference between a mediocre answer and a brilliant one can be a single sentence of instruction.

---

## 🧠 Core Mechanics

### 1. What Prompt Engineering Is (Q77)

**Definition:** The practice of designing, structuring, and refining the text inputs (prompts) given to an LLM to guide it toward accurate, relevant, and appropriately formatted outputs — without changing model weights.

**Why it matters:**
- LLMs are extremely sensitive to phrasing, ordering of examples, and level of specificity.
- A well-crafted prompt can close most of the performance gap between a mid-tier and a top-tier model.
- It's the fastest, cheapest path to improving LLM output — no retraining required.

**Anatomy of a well-structured prompt:**
```
[System Prompt]     → Define the model's role, tone, constraints
[Context/Background]→ Any relevant information the model needs
[Examples]          → (Optional) Few-shot demonstrations
[Task Instruction]  → Exactly what you want the model to do
[Input Data]        → The specific content to process
[Output Format]     → How you want the response structured
```

---

### 2. System Prompt vs. User Prompt (Q81)

| | System Prompt | User Prompt |
|---|---|---|
| **Purpose** | Defines the model's persona, role, constraints, and behavior globally | Provides the specific task or query for this turn |
| **Scope** | Persists across the entire conversation | Per-turn |
| **Who writes it** | Developer / product builder | End user |
| **Example** | "You are an expert data scientist. Always cite sources. Never make up numbers." | "Explain what gradient descent is." |

**Why system prompts matter:** They ground every subsequent response — a well-written system prompt reduces hallucinations, enforces output format, and sets appropriate scope boundaries.

---

### 3. Zero-Shot vs. Few-Shot Prompting (Q78)

| | Zero-Shot | Few-Shot |
|---|---|---|
| **Definition** | Task instruction only — no examples | Task instruction + `n` input-output demonstrations |
| **Model reliance** | Entirely on pre-trained knowledge | Pre-trained knowledge + in-context pattern matching |
| **Best when** | Task is simple or model is very capable | Task requires specific format, style, or domain |
| **Token cost** | Low | Higher (examples consume context window) |

**Example (Sentiment Classification):**
```
# Zero-shot
Classify the sentiment of the following review as Positive or Negative.
Review: "The product broke after two days."
Sentiment:

# Few-shot (2-shot)
Classify sentiment as Positive or Negative.
Review: "This is the best purchase I've ever made." → Positive
Review: "Terrible quality, waste of money." → Negative
Review: "The product broke after two days." → 
```

---

### 4. In-Context Learning (ICL) (Q82)

**Definition:** The ability of large language models to perform new tasks simply by conditioning on examples provided in the prompt — **without any gradient updates or fine-tuning**.

- Few-shot prompting is the most common form of ICL.
- ICL is an emergent property — smaller models do it poorly; large models (>10B parameters) do it well.
- **How it works mechanically:** Debated. Leading hypothesis: the model uses in-context examples to infer the task distribution and adapt its next-token predictions accordingly. It is not gradient-based learning.

**Key insight:** ICL works best when examples are:
- Representative of the target task distribution
- Correctly labeled (wrong labels hurt significantly)
- Ordered to show diversity of input types

---

### 5. Choosing Few-Shot Examples (Q79)

| Strategy | Mechanism | Best For |
|---|---|---|
| **Random sampling** | Pick examples randomly from the dataset | Quick baseline; general tasks |
| **Diversity-based** | Select examples covering a wide variety of input patterns | Preventing distribution bias |
| **Similarity-based** | Use vector embeddings to retrieve examples closest to the current query | High-precision tasks; RAG-style prompting |

**Similarity-based is generally best** for domain-specific tasks — retrieving semantically similar examples to the input consistently outperforms random or fixed few-shot selection.

---

### 6. Chain-of-Thought (CoT) Prompting (Q74, Q75, Q76)

**Definition:** Instructing the model to show its step-by-step reasoning before producing the final answer, mimicking how a human would think through a complex problem.

**Two variants:**

| Variant | How | When to Use |
|---|---|---|
| **Zero-shot CoT** | Add "Let's think step by step." to the prompt | Quick, no examples needed |
| **Few-shot CoT** | Provide examples that include full reasoning chains | Better quality, more control |

**Why CoT works (Q75):**
- Breaks complex problems into simpler intermediate steps — each step is easier to get right.
- Intermediate steps act as a "scratchpad" — the model can condition each next step on the previous correct one.
- Effectively gives the model more tokens to "think" before committing to a final answer.
- Strong gains on: math word problems, multi-hop reasoning, symbolic reasoning, commonsense QA.

**Trade-offs (Q76):**
- ✅ Significant accuracy gains on complex reasoning tasks
- ❌ More output tokens → higher latency and cost
- ❌ If the reasoning chain goes wrong early, errors compound (hallucinated reasoning)
- ❌ Requires careful prompt design — too verbose CoT chains can confuse the model

**Example:**
```
# Without CoT
Q: A train travels 60 mph for 2 hours then 80 mph for 3 hours. Total distance?
A: 360 miles.   ← often wrong with standard prompting

# With CoT  
Q: A train travels 60 mph for 2 hours then 80 mph for 3 hours. Total distance?
A: Let's think step by step.
   First segment: 60 mph × 2 hours = 120 miles.
   Second segment: 80 mph × 3 hours = 240 miles.
   Total: 120 + 240 = 360 miles.   ← correct and verifiable
```

---

### 7. Self-Consistency Prompting (Q83)

**Definition:** Generate multiple independent reasoning paths (via sampling) for the same question, then **majority-vote** on the final answer.

**Mechanism:**
1. Use CoT prompting with a non-zero temperature to generate `k` diverse reasoning chains.
2. Extract the final answer from each chain.
3. Return the answer that appears most frequently.

**Why it helps:** A single CoT chain can go off-rails. Multiple independent attempts are unlikely to all make the same mistake — the correct answer tends to win the majority vote.

**Cost:** `k×` more inference calls → `k×` higher latency and cost. Used selectively for high-stakes tasks.

---

### 8. Context Length Management in Prompts (Q80, Q84)

- **Context window is a hard limit:** Prompt + few-shot examples + system prompt + output must all fit within `max_tokens`.
- If the prompt overflows, early content is truncated — typically the *beginning* of the conversation, which often contains critical instructions.
- **Priority rule when designing prompts:** Most important information last (recency bias in attention).

**Practical prompt sizing guidelines:**
```
System prompt:        100–500 tokens  (be concise)
Few-shot examples:    200–1000 tokens (2–5 examples max)
Task instruction:     50–200 tokens
Input data:           remaining budget
Output space:         reserve ~20% of context window
```

**When context limits bite:**
- Long documents → use RAG (retrieve relevant chunks, don't dump the whole doc)
- Long conversations → summarize earlier turns periodically
- Many examples → switch to fine-tuning if few-shot is too expensive

---

### 9. Structured Output Prompting (Q86)

**Problem:** LLMs produce free-form text by default. Production pipelines need parseable structured data (JSON, XML, CSV).

**Strategies:**

1. **Explicit format instruction:**
   ```
   Respond ONLY with a valid JSON object. No explanation, no preamble.
   Schema: {"name": string, "sentiment": "positive"|"negative", "score": float}
   ```

2. **Provide a filled example:**
   ```
   Example output: {"name": "Alice", "sentiment": "positive", "score": 0.92}
   Now process: [your input]
   ```

3. **Constrained decoding (production):** Use libraries like Outlines or Guidance to enforce grammar-level constraints during generation — guarantees valid JSON without prompt tricks.

4. **Model-native JSON mode:** Many APIs (OpenAI, Anthropic) support a `response_format: {type: "json_object"}` parameter — forces valid JSON output at the API level.

---

### 10. Hallucination Reduction via Prompting (Q85)

**Key strategies:**

| Strategy | Prompt Example | Mechanism |
|---|---|---|
| **Grounding** | "Answer only using the provided document. If not found, say 'Not available'." | Constrains search space to provided context |
| **Uncertainty instruction** | "If you are not certain, express your uncertainty explicitly." | Allows calibrated hedging instead of confabulation |
| **Step-by-step verification** | "After your answer, verify each claim against the document." | Self-checking reduces confident errors |
| **RAG** | Inject retrieved facts before the question | Provides verifiable ground truth in context |

**Why grounding works:** Without constraints, the model samples from its full learned distribution — including patterns that sound plausible but are factually wrong. Grounding limits the sampling space to what's actually in the provided context.

---

### 11. ReAct Prompting for Agents (Q87)

**Definition:** ReAct (Reasoning + Acting) is a prompting framework for AI agents that interleaves reasoning steps with real-world actions (tool calls), grounded by observations from those actions.

**The ReAct loop:**
```
Thought:  "I need to find the current population of Taiwan."
Action:   search_web("Taiwan population 2024")
Observation: "Taiwan has a population of approximately 23.6 million."
Thought:  "Now I have the data. I can answer the question."
Answer:   "Taiwan's population is approximately 23.6 million."
```

**Why it reduces hallucinations:** At each step, the model's reasoning is anchored to actual tool outputs (observations), not generated from memory alone. Errors are correctable because the model can observe that an action failed.

**Contrast with pure CoT:** CoT reasons entirely from pre-trained knowledge. ReAct grounds reasoning in real-time external information.

**Common agent tools:** web search, code interpreter, calculator, database queries, API calls.

---

## ⚠️ Edge Cases & Constraints
- **Prompt sensitivity:** Small phrasing changes can dramatically shift LLM output — this is a known reliability concern. Prompts should be tested across paraphrases.
- **Few-shot label order matters:** The order of examples in few-shot prompts can bias results — the last example before the query has disproportionate influence.
- **CoT doesn't help smaller models:** Chain-of-thought prompting is most effective for models with >~100B parameters; smaller models may produce incoherent reasoning chains.
- **Self-consistency is expensive:** At `k=20` generations, it's 20× the inference cost — only justified for high-value or high-stakes queries.
- **System prompt injection attacks:** Malicious user inputs can attempt to override system prompt instructions ("Ignore all previous instructions...") — a security concern for production deployments.

---

## 💻 Logical Code Snippet (Python)

```python
# ── Zero-shot vs Few-shot prompt construction ─────────────────────────
def zero_shot_prompt(task_description: str, input_text: str) -> str:
    return f"""{task_description}

Input: {input_text}
Output:"""


def few_shot_prompt(task_description: str, examples: list[dict], input_text: str) -> str:
    example_block = "\n".join(
        f"Input: {ex['input']}\nOutput: {ex['output']}" for ex in examples
    )
    return f"""{task_description}

{example_block}

Input: {input_text}
Output:"""


# ── Chain-of-Thought prompt ───────────────────────────────────────────
def cot_prompt(question: str, zero_shot: bool = True) -> str:
    if zero_shot:
        return f"{question}\n\nLet's think step by step."
    # Few-shot CoT: prepend examples that include full reasoning chains
    return f"""Q: [Example question]
A: Let's think step by step. [Reasoning...] Therefore, the answer is [X].

Q: {question}
A: Let's think step by step."""


# ── Self-consistency: majority vote over multiple samples ─────────────
from collections import Counter

def self_consistent_answer(model, prompt: str, k: int = 10, temperature: float = 0.7):
    answers = []
    for _ in range(k):
        response = model.generate(prompt, temperature=temperature)
        answer = extract_final_answer(response)  # parse last line or "Answer: ..."
        answers.append(answer)
    # Return the most common answer
    return Counter(answers).most_common(1)[0][0]


# ── Structured JSON output prompt ────────────────────────────────────
def json_output_prompt(task: str, schema: dict, input_text: str) -> str:
    import json
    return f"""{task}

Respond ONLY with a valid JSON object matching this schema:
{json.dumps(schema, indent=2)}

Do not include any explanation, preamble, or markdown backticks.

Input: {input_text}
JSON Output:"""
```

---

## ❓ Active Recall
- [ ] What is prompt engineering, and why is it effective without modifying model weights?
- [ ] What is the difference between a system prompt and a user prompt?
- [ ] What is the difference between zero-shot and few-shot prompting?
- [ ] What is In-Context Learning (ICL)? How is it different from fine-tuning?
- [ ] What are the three strategies for selecting few-shot examples? Which performs best and why?
- [ ] What is Chain-of-Thought prompting? Give an example of when it helps.
- [ ] What are the trade-offs of using CoT prompting?
- [ ] What is self-consistency prompting? What is its main cost?
- [ ] Why does context length matter when designing prompts?
- [ ] What are three strategies to force LLM output into JSON format?
- [ ] Name two prompt-level strategies to reduce hallucinations.
- [ ] What is ReAct prompting? How does it differ from pure CoT?
- [ ] **Follow-up:** What is "prompt injection" and why is it a security concern?
- [ ] **Follow-up:** What is "Tree-of-Thought" prompting, and how does it extend CoT?
- [ ] **Follow-up:** What is the difference between ICL and RLHF as methods to steer model behavior?
