# Revised Interview Answer Script — Binance LLM Data Scientist (BAP)
**JD Focus:** LLM agents for Web3, token research/summarization, trading agents, prompt engineering, MCP, memory, agent coordination  
**Core Requirements:** Agent orchestration, LangGraph/LangChain, RAG, vector databases, agent evaluation  
**Framework Applied:** STAR / PREP / What–So What–Now What / 3-2-1

> **Key reframe from previous version:** The prior script over-indexed on post-training (SFT/RLHF) which is NOT this JD's core requirement. This role is about building and evaluating LLM agents for financial/Web3 scenarios — which is your strongest ground. Stop treating this as a gap interview. This is a strength interview. Your job is to go deep, not apologize.

---

## Framework Selection Logic

| Question Type | Best Framework |
|---|---|
| "Tell me about yourself / why this role" | **PREP** — positioning question |
| "Walk me through Paprika" | **STAR** — experience verification |
| "Hardest problem in RAG competition" | **STAR** — behavioral |
| "How did you validate your RAG system?" | **What / So What / Now What** |
| "How does your SOP caching work?" | **What / So What / Now What** |
| "What is RAG and when does it fail?" | **3-2-1** — conceptual, multiple components |
| "What is LangGraph / pgvector / LLM-as-judge?" | **3-2-1** — framework explanation |
| "What's your experience with agent evaluation?" | **STAR** |
| "How would you build a token research agent?" | **What / So What / Now What** — design question |
| "Why hire you?" | **PREP** |

---

## Q1 — Tell me about yourself and why you're applying.
**Framework: PREP**

> **Point:** I'm a CS student at NCKU who has spent the past year building production-grade LLM agent systems — agent orchestration, RAG pipelines, vector databases, and evaluation infrastructure. This role is the closest match to what I've been building that I've seen.

> **Reason:** I started in security research, which gave me a systems-level instinct for failure modes and trust boundaries — instincts I now apply directly to agent design. When I ask "what happens when the LLM misbehaves?", I'm not being academic. I'm designing the fallback loop. Over the past year and a half I've shifted that foundation fully into LLM agent systems, with two substantial projects as outputs.

> **Example:** In Paprika, I built the full agentic backend for a cooperative cooking game — LangGraph FSM with eight nodes, pgvector SOP caching, WebSocket real-time state push from Unity, LLM-as-judge evaluation on EC2. The core problem was state ambiguity when object transforms caused context loss — I solved it with context suppression and deterministic state injection, reducing decision latency from ~8s to ~1–2s. In the E.SUN Financial RAG competition, top 25% of 487 teams, building a financial Q&A system across three document domains each requiring completely different chunking strategies — FAQ, insurance policies, and financial reports including scanned PDFs where I built an LLM-based OCR fallback when Tesseract failed. I also placed 262nd globally out of 10,460 in picoCTF 2025, applying prompt injection and LLM jailbreak techniques — directly relevant to Web3 systems where adversarial inputs target both the contract layer and the LLM layer.

> **Point (restated):** Agent orchestration, RAG, vector databases, evaluation — these are not things I've read about. They're the systems I've been shipping. That's why I'm applying.

---

## Q1a — Your resume mentions LangGraph. How deeply have you used it and what did you learn about its limitations?
**Framework: PREP**

> **Point:** LangGraph is the core runtime for Paprika's agent — I've used it for a full production-grade FSM with eight nodes, conditional edges, and persistent state. And I've hit its real limitations in production.

> **Reason:** LangGraph gives you explicit state graph control, which is what you need when agent behavior has to be deterministic and inspectable. The alternative — letting the LLM decide what to do next at every step — fails in production because you can't debug it or guarantee loop termination. The FSM model forces you to define exactly what transitions are legal, which surfaces design problems early.

> **Example:** The limitation I hit hardest was state schema design. LangGraph's state is a typed dict passed through the graph — anything not explicitly in the schema gets dropped between nodes. In Paprika, when the cooking sequence spanned many steps, we had to carefully decide what persisted: the SOP sequence, the critic's failure history, the current game state snapshot from Unity. Getting the schema wrong caused a subtle bug where the agent would forget a previous failure and retry the same broken action. The fix was treating the state schema as a first-class architectural decision, not an afterthought.

> **Point (restated):** LangGraph is powerful precisely because it's explicit — but that explicitness means the state schema and transition logic are where the real engineering work lives, not the prompts.

---

## Q2 — Walk me through Paprika's architecture. Focus on how the agent makes decisions.
**Framework: STAR**

> **Situation:** We were building a cooperative cooking game where an LLM-driven NPC needed to work alongside a human player in real time. The core constraint was latency — pure LLM reasoning at every step produced ~8s decision time, which broke the cooperative loop entirely. We were also inspired by the Voyager paper's skill library concept.

> **Task:** I needed to design an agent architecture that could make fast, reliable decisions in a partially observable environment — game state arrives from Unity over WebSocket, object positions change mid-task, and the agent needs to recover gracefully when actions fail.

> **Action:** I built a four-stage FSM in LangGraph: curriculum (goal planning), skill (SOP retrieval), action (tool execution), and critic (success verification) — eight nodes total. The decision flow:
> 1. Curriculum receives the current goal and queries pgvector with a semantic embedding of the planning output. If a stored SOP matches above threshold, the skill stage retrieves it. If not, the action stage reasons from scratch.
> 2. Each action call returns a result from Unity via WebSocket. The critic evaluates success. Two consecutive failures trigger fallback to curriculum with the failure context appended — forcing a re-plan rather than retrying blindly.
> 3. Successful novel sequences get stored back to pgvector, so the SOP library grows through play — the skill accumulation loop from Voyager.
>
> The state ambiguity problem — the LLM losing track of object transforms when cooked meat moved from pot to hand — was solved through context suppression: inject "cooking succeeded" and suppress the now-irrelevant pot status. This prevented context flood while keeping reasoning grounded.

> **Result:** Decision latency dropped from ~8s to ~1–2s for cached tasks after warmup. CI/CD runs LLM-as-judge evaluation on EC2 with LangSmith tracing for end-to-end behavior monitoring.

---

## Q2b — How does the SOP caching work at the retrieval level?
**Framework: What / So What / Now What**

> **What:** The retrieval key is the curriculum stage's natural language planning output — "prepare a hamburger starting with the meat" — embedded with the same model used at storage time, queried against pgvector by cosine similarity. A stored SOP is a named ordered list of tool calls: "prepare_cooked_meat: [go_to_refrigerator, pick_meat, go_to_pot, cook_meat, pick_up_meat, go_to_preparation_table, put_meat]."

> **So What:** Using semantic embedding of the planning output — rather than exact task name matching — gives robust retrieval even when the same task is described differently across game contexts. The SOP isn't executed blindly: each step goes through the critic, so a stale or partially-wrong SOP fails gracefully rather than cascading.

> **Now What:** The known limitation is that SOPs store concrete object references (preparation_table_2) but our context suppression layer removes specific identifiers from the prompt. We're moving to abstract SOPs — store "go_to_preparation_table," let a deterministic state injection layer ground it to the correct instance at execution time. The more interesting extension: let the agent revise SOPs on failure, which is essentially a lightweight policy update on the skill library — the direction Voyager's RL component points toward.

---

## Q3 — RAG Competition: What was the hardest technical problem and how did you solve it?
**Framework: STAR**

> **Situation:** E.SUN Financial RAG competition — Q&A system across three domains: bank FAQ, insurance policies, and financial reports. 487 teams. The naive assumption: documents in the same category share the same structure.

> **Task:** Documents within the same category had wildly different internal structures. Insurance policies especially — some indexed by article number, some by section header, some with tables mid-clause. Our recursive text splitter fused the tail of article 50 with the header of article 51. The retrieval embedding correctly matched the malformed chunk — so retrieval "worked" but the answer was contaminated. The binary competition score told us something was wrong but not where.

> **Action:** Two-stage approach. First, I built a validation pipeline: logged the full retrieval chain per question and manually inspected every zero-scored question. That's how I identified the boundary fusion pattern. Then document-level forensics on my ~100 assigned documents — I found law articles consistently used a dollar-sign prefix as a section marker, wrote custom regex to split on that delimiter, and validated every chunk started with a valid boundary. For outlier files, per-file manual inspection and one-off rules. Six full re-chunk → re-embed → re-evaluate cycles, ~2–3 hours each.

> **Result:** Finance domain 84%, FAQ 96%, insurance 94%. The finance gap came from scanned PDFs — Tesseract failed on low-quality accounting table scans, so we fed the pages directly to an LLM. The reason LLM outperforms Tesseract here is fundamental: Tesseract is pixel-pattern matching — when scan quality degrades, glyph templates fail hard with no fallback. An LLM reads with comprehension — it infers a partial digit from surrounding financial context the same way a human squints at a blurry printout and reconstructs the meaning. The tradeoff: LLM OCR can hallucinate a plausible-looking number silently, where Tesseract fails loudly. In production that silent failure mode is more dangerous — you'd need a confidence-flagging layer routing low-certainty extractions to human review. Top 25% of 487 teams overall.

---

## Q3b — How would you evaluate a RAG system's quality beyond a single accuracy score?
**Framework: What / So What / Now What**

> **What:** The competition's 0/1 score is an end-to-end pass/fail signal — tells you the system failed, not why. In our pipeline I manually classified failures into four types: boundary fusion (malformed chunk), missing chunk (correct content not retrieved), OCR failure, and genuine hard cases (ambiguous question even with the correct chunk).

> **So What:** The taxonomy let us prioritize by systemic impact. Boundary fusion and OCR failures were systematic and fixable. Genuine hard cases weren't worth engineering time — no chunking improvement would fix them. Without the taxonomy, you treat every failure as equally worth iterating on and waste cycles.

> **Now What:** A production RAG evaluation stack has three separate layers: retrieval quality (recall@k on a labeled set — did the right chunk make top-k?), generation quality (LLM-as-judge on answer correctness given the chunk — is the LLM using the chunk correctly?), and end-to-end quality. Separating these lets you localize failures: high retrieval recall but low end-to-end score means the generation layer is the problem, not retrieval. If retrieval recall is low, no generation tuning will fix it.

---

## Q4 — How would you design a token research and summarization agent?
**Framework: What / So What / Now What**

> **What:** A token research agent needs to answer questions like "what is the current narrative around token X, what are the risks, and what does on-chain data say about holder concentration?" That requires three distinct data sources: structured on-chain data (holder distribution, transaction volume, whale movements), unstructured text (project docs, news, sentiment), and real-time market data (price, volume, liquidity). A single retrieval strategy won't cover all three.

> **So What:** The architecture I'd build: a router agent classifies the query type and dispatches to specialized sub-agents. An on-chain data agent queries a structured blockchain indexer (The Graph, Dune Analytics) and returns structured results. A document agent runs RAG over a vector store of project whitepapers, audit reports, and news — with recency-weighted retrieval since stale token research is often worse than no research. A market data agent hits a price API. A synthesis agent takes all three outputs and produces the final summary with source citations. The critical design decision is making the router's classification explicit and logged — you need to know which sub-agent contributed what to each summary so you can evaluate and improve them independently.

> **Now What:** The hardest operational problem isn't the retrieval — it's keeping the vector store fresh for thousands of tokens and deciding what to index versus query live. I'd implement a freshness layer: content older than N days gets flagged for re-indexing, and breaking news triggers an immediate ingestion pipeline rather than waiting for the scheduled refresh. Evaluation uses LLM-as-judge on a human-labeled test set of token research questions with known correct answers — same three-layer evaluation structure as the RAG system.

---

## Q5 — What's your experience with agent evaluation and what makes it hard?
**Framework: STAR**

> **Situation:** In Paprika, I built the LLM-as-judge evaluation pipeline. The agent makes tool calls in a game environment, and the judge evaluates whether the action sequence was correct given the game state. This ran on EC2, triggered by CI/CD on every push, with results stored on S3.

> **Task:** Game agent behavior is hard to evaluate with deterministic metrics — multiple valid action sequences can accomplish the same goal. I needed a judge that evaluated intent and outcome, not string matching.

> **Action:** I used GPT-4 as judge and Claude as the agent model, deliberately different families to reduce self-enhancement bias. The judge received the Unity game state, the agent's action sequence, and a rubric for success. LangSmith traced every agent run — full prompt chain, tool call history, judge verdict — making failure debugging tractable. Key failure mode I designed against: verbosity bias. An agent can hallucinate a plausible-sounding action sequence and score well. I mitigated this by including the actual Unity game state in the judge's context, giving it direct evidence to contradict false action claims.

> **Result:** The pipeline caught two regression categories: prompt changes that broke tool call format (judge scored correctly), and state management bugs where the agent took correct actions in the wrong order (judge missed some — a calibration gap I'd address with adversarial test cases in a production version). Honest limitation: we didn't calibrate the judge against human labels on a held-out set. Without that calibration number, you don't know if your judge is signal or noise.

---

## Q6 — What's your familiarity with Web3 and crypto?
**Framework: 3-2-1**

> **3 areas, 2 practical touchpoints, 1 honest gap:**

> **Blockchain data structures:** I understand Merkle trees, SHA-256, and blockchain data structures at the implementation level — studied as part of my security background. I understand PoW vs PoS consensus at a conceptual level and why they matter for data integrity and finality guarantees.

> **Web3 security:** My CTF background (262nd globally, picoCTF 2025) included prompt injection and LLM jailbreak techniques — directly relevant to Web3 AI systems where adversarial inputs can target both the smart contract layer and the LLM layer simultaneously. I understand the trust model difference between on-chain and off-chain computation.

> **Token economics:** I understand supply mechanics, liquidity pools, and on-chain data availability at a conceptual level — enough to reason about what a token research agent needs to retrieve and why.

> **1 honest gap:** I haven't built on blockchain infrastructure directly — no smart contract development, no DeFi protocol integration, no backtesting experience. My Web3 knowledge is stronger on data and security than protocol. For the agent use cases in this role, the agent architecture and RAG skills are what matters most — and those I have. The domain knowledge I can close fast.

---

## Q7 — Why hire you over a candidate with equivalent agent experience?
**Framework: PREP**

> **Point:** Most agent engineers can build a pipeline. Fewer have built one that has to work reliably in a real-time environment with a human depending on it — and then built the evaluation infrastructure to prove it works.

> **Reason:** In Paprika the agent wasn't running in a sandbox — it was playing alongside a human in a live game. That constraint made every design decision production-grade: latency had to be acceptable, failures had to recover gracefully, and the evaluation pipeline had to catch regressions before they hit the game. I built all three layers: the agent (LangGraph FSM, SOP caching, fallback loops), the retrieval (pgvector semantic lookup, domain-specific chunking in the RAG competition), and the evaluation (LLM-as-judge on EC2, LangSmith tracing, CI/CD). Full-stack ownership across all three is unusual.

> **Example:** The E.SUN RAG competition forced evaluation-driven development from scratch — six iteration cycles, failure taxonomy, per-failure-type prioritization. That's the same discipline you need when building a token research agent: you need to know whether a bad answer came from retrieval failure, generation failure, or a genuinely hard question — and you need to iterate on the right layer. I've done that under competitive pressure against a real external ground truth signal. And my security background gives me an adversarial lens on Web3 AI systems that most agent engineers don't have.

> **Point (restated):** Agent architecture, RAG, vector databases, evaluation, and a security background relevant to Web3 adversarial inputs — I'm not describing a profile I'm growing toward. That's what I've been building for the past year. This role is the natural next step.

---

## Master Framework Reference

| Signal Word in Question | Framework |
|---|---|
| "Tell me about a time..." / "Walk me through..." / "Describe how you built..." | **STAR** |
| "Why do you want..." / "What is your strength..." / "Why should we hire you..." | **PREP** |
| "What happened with X?" / "How did you handle Y?" / "How would you design Z?" | **What / So What / Now What** |
| "Explain how X works" / "What does X do?" / Multiple concepts at once | **3-2-1** |

---

## Tone Recalibration for This JD

The previous script had you in defensive mode — constantly apologizing for post-training gaps. This JD does not require post-training. Recalibrate completely:

| Previous posture (wrong JD) | Correct posture (this JD) |
|---|---|
| "I'll be transparent, my gap is SFT/RLHF..." | "Agent orchestration and RAG are my core strength — let me go deep." |
| "I'm ramping on Llama Factory and TRL..." | "LangGraph, pgvector, LLM-as-judge — I've shipped all three in production." |
| "I can close the gap quickly..." | "I don't have a gap on the core requirements. Let me show you." |
| Mentioning post-training first | Lead with Paprika and E.SUN — those are the exact requirements |

**The one honest gap to acknowledge proactively:** Web3/crypto domain knowledge and trading systems (listed as nice-to-have, not required). Frame it as: "My Web3 knowledge is stronger on data and security than protocol — and the agent architecture skills are what this role needs most."
