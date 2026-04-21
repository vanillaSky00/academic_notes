# 📄 PPO & GRPO: Modern RL Algorithms for LLM Post-Training

**Tags:** #deep-learning #RL #PPO #GRPO #policy-gradient #LLM #DeepSeek #RLVR  
**Links:** [[RLHF Training Objective]], [[Rewards and Preference Learning]], [[LoRA PEFT]]

---

## 🎯 The "Elevator Pitch"
> **PPO** adds one key refinement on top of the basic policy gradient objective: it *clips* the importance ratio so no single update can move the model too far — small, stable steps over many iterations. **GRPO** then eliminates the expensive per-token value model by instead sampling a *group* of responses per prompt and using their average reward as the baseline, cutting training memory roughly in half.

---

## 🧠 Core Mechanics

### 1. From Basic Policy Gradient to PPO

Recall the basic training objective from the previous note:

$$\mathcal{J}_{\text{basic}} = \mathbb{E}\left[\rho_t \cdot A_t\right], \quad \rho_t = \frac{\pi_\theta(y_t)}{\pi_{\text{ref}}(y_t)}$$

**The problem:** If $\rho_t$ becomes very large (current policy assigns much higher probability to a token than the reference did), the gradient update is enormous — the model takes a huge step, possibly catastrophically destroying previous knowledge. This is training instability at its worst.

**PPO's fix: Clip the ratio.** Instead of using $\rho_t$ directly, use:

$$\mathcal{J}_{\text{PPO}} = \mathbb{E}\left[\min\left(\rho_t \cdot A_t, \; \text{clip}(\rho_t, 1-\epsilon, 1+\epsilon) \cdot A_t\right)\right]$$

**Reading the clip:** $\text{clip}(\rho_t, 1-\epsilon, 1+\epsilon)$ forces $\rho_t$ to stay within $[1-\epsilon, 1+\epsilon]$ — typically $\epsilon = 0.2$.

**Reading the min:** The `min` ensures we always take the *more conservative* (pessimistic) estimate:
- When $A_t > 0$ (good token): if $\rho_t > 1+\epsilon$, the unclipped value is larger → min picks the clipped one → update is capped → can't over-reward a token
- When $A_t < 0$ (bad token): if $\rho_t < 1-\epsilon$, the unclipped value is "more negative" → min picks the clipped one → can't over-penalize a token

**The intuition:** PPO enforces a "trust region" — the model can only move so far from the reference policy in a single update. It's the RL equivalent of a learning rate scheduler that prevents gradient explosion, but operating on the policy ratio level.

---

### 2. PPO's Advantage: The Value Model (GAE)

PPO computes per-token advantages using a **value model** — a separate neural network (usually another copy of the LLM with a scalar head, same as the reward model) that predicts:

$$V(x, y_{<t}) = \mathbb{E}[r \mid \text{context so far}]$$

"Given everything generated so far, what reward do I expect to get at the end?"

The per-token advantage is then:

$$A_t = r_t + \gamma V(s_{t+1}) - V(s_t)$$

And PPO uses **Generalized Advantage Estimation (GAE)** to smooth this across time steps — it propagates the final reward *backward* through earlier tokens with a discount factor $\lambda$, distributing credit more evenly. This helps early tokens that "set up" a good final answer receive positive advantage even if the reward only arrives at the last token.

$$A_t^{\text{GAE}} = \sum_{k=0}^{\infty} (\gamma \lambda)^k \delta_{t+k}, \quad \delta_t = r_t + \gamma V(s_{t+1}) - V(s_t)$$

**The cost:** The value model is typically as large as the policy model itself — you're training two LLMs simultaneously. This roughly doubles the memory requirement of the RL training loop.

---

### 3. Full PPO Model Inventory

| Model | Role | Memory status |
|---|---|---|
| **Policy LLM** ($\pi_\theta$) | The model being improved | Weights + grads + optimizer states |
| **Reference LLM** ($\pi_{\text{ref}}$) | Frozen; generates rollouts; provides $\rho_t$ denominator | Weights only (no grads) |
| **Reward Model** | Scores rollouts → scalar $r$ | Weights only (frozen during RL) |
| **Value/Critic Model** | Predicts expected reward per token → baseline for advantage | Weights + grads + optimizer states |

**Total memory:** ~4× inference memory. For a 7B model at bf16 that's ~14 GB inference → ~56 GB for PPO training. One A100 isn't enough.

---

### 4. GRPO: Kill the Value Model

**Key question GRPO asks:** "Why do we need an entire separate LLM just to estimate the average reward? Can't we just… compute the actual average reward?"

**GRPO's answer:** For each prompt $x$, sample $G$ different responses $\{y_1, y_2, \ldots, y_G\}$. Score them all. The baseline is literally their mean reward:

$$b = \bar{r} = \frac{1}{G} \sum_{i=1}^{G} r_i$$

The advantage for response $i$ is:

$$A_i = \frac{r_i - \bar{r}}{\sigma_r + \epsilon}$$

where $\sigma_r$ is the standard deviation of rewards within the group (for normalization). This is just z-score normalization.

**Concrete example with $G = 4$:**

| Response | Raw reward $r_i$ | Advantage $A_i = r_i - \bar{r}$ |
|---|---|---|
| A (great answer) | 0.9 | +0.35 |
| B (decent) | 0.7 | +0.15 |
| C (mediocre) | 0.4 | −0.15 |
| D (wrong) | 0.2 | −0.35 |

$\bar{r} = 0.55$. Now A and B get positive advantage → their probability increases. C and D get negative → their probability decreases. All centered around zero.

**Key difference from PPO:**
- PPO advantage: **per token**, predicted by a value model during generation
- GRPO advantage: **per sequence**, computed from group statistics after generation

GRPO has no per-token credit assignment. Every token in a winning response gets the same positive advantage, and every token in a losing response gets the same negative advantage. Coarser signal, but vastly cheaper.

---

### 5. GRPO Objective

GRPO keeps PPO's clipping (for stability), just swaps the advantage computation:

$$\mathcal{J}_{\text{GRPO}} = \mathbb{E}_{x, \{y_i\}} \left[ \frac{1}{G} \sum_{i=1}^{G} \sum_{t=1}^{T} \min\left(\rho_{i,t} \cdot A_i, \; \text{clip}(\rho_{i,t}, 1-\epsilon, 1+\epsilon) \cdot A_i\right) \right]$$

Plus the KL penalty (or it's sometimes removed — see DAPO).

---

### 6. PPO vs. GRPO: The Full Comparison

| Dimension | PPO | GRPO |
|---|---|---|
| **Baseline model** | Separate value model (trained) | Group mean reward (computed on-the-fly) |
| **Advantage granularity** | Per token (fine-grained, uses GAE) | Per sequence (coarse but unbiased) |
| **Number of models** | 4 (policy + ref + reward + value) | 3 (policy + ref + reward) |
| **Memory saving** | Baseline | ~50% reduction vs PPO |
| **Stability** | High (clipping + value smoothing) | High (clipping) |
| **Extra computation** | Value model inference at every step | Sample G responses per prompt |
| **Credit assignment** | Token-level (good for long reasoning) | Sequence-level (can misattribute) |
| **Best for** | General alignment, dialogue | Reasoning tasks with verifiable rewards |

---

### 7. RLVR: The Reasoning Revolution

**RLVR (Reinforcement Learning with Verifiable Rewards)** = GRPO + verifier rewards (no learned RM). This combination — pioneered by DeepSeek-R1-Zero — is what enabled training *reasoning models* without human labels:

1. Sample a math problem
2. Generate $G$ candidate solutions
3. Run a verifier: does each solution get the right answer?
4. Compute group-normalized advantages
5. Update policy with GRPO

Because verifiers are **hard to hack** (the model can't trick a Python evaluator), RL can run for many more steps than with a learned RM before training collapses. This allows the model to explore and develop capabilities like:
- **Chain-of-thought reasoning** (emerged spontaneously — not trained explicitly)
- **Self-reflection** ("Wait, let me reconsider...")
- **Longer responses** (GRPO + accuracy reward naturally incentivizes showing work)

None of these behaviors were explicitly programmed — they arose from optimizing a simple binary correctness signal.

---

### 8. The Historical Arc

```
2017: PPO introduced (Schulman et al.) — general RL algorithm for stability
2019–2020: OpenAI adapts PPO for language model alignment
2022: InstructGPT/ChatGPT — PPO + RLHF at scale; KL penalty formalized
2022: Constitutional AI (RLAIF) — AI feedback replaces human labelers
2023: DPO — bypass RL entirely with supervised preference optimization
2024: GRPO introduced in DeepSeekMath — eliminates value model
2025: DeepSeek-R1-Zero — GRPO + verifiers only, no SFT, emergent reasoning
2025: DAPO — removes KL penalty, adds entropy bonus for exploration
```

---

## ⚠️ Edge Cases & Constraints

**GRPO's group collapse problem:** If all $G$ responses get the same reward (all correct, or all wrong), the standard deviation $\sigma_r = 0$ and advantage computation divides by zero. In practice, clip $\sigma_r \geq \epsilon$ or skip the update for degenerate groups. This is most common at very early training (model always wrong) or very late training (model always right on easy problems).

**PPO's value model instability:** The value model is trained simultaneously with the policy, and their targets shift as the policy improves. This "moving target" problem can cause value estimates to diverge, destabilizing the whole training loop. A warmup period where only the value model trains (policy frozen) is common practice.

**GRPO's coarse credit assignment:** If a 500-token response earns a high reward, every single token gets positive advantage — including tokens that were filler or even slightly harmful. PPO's token-level advantage (from GAE) is more surgical about which tokens actually contributed. For long complex reasoning, this matters.

**DAPO removes KL entirely:** The DAPO paper (2025) argues the KL penalty actively hurts training on reasoning tasks — it prevents the exploration needed to discover novel solution strategies. Instead it uses a clip-higher trick + entropy bonus to maintain diversity. This is actively contested; the right choice depends on task.

**Sampling temperature matters for GRPO:** If temperature is too low, all $G$ samples are nearly identical → no diversity in rewards → noisy advantages. If too high, responses are incoherent. Typical range: temperature 0.6–1.0 for GRPO rollouts.

---

## 💻 Logical Code Snippet (Python)

```python
import torch
import torch.nn.functional as F

# ─────────────────────────────────────────────────────────────
# PPO Clipped Objective
# ─────────────────────────────────────────────────────────────

def ppo_loss(policy_log_probs, ref_log_probs, advantages, epsilon=0.2):
    """
    policy_log_probs: (batch, seq_len) — log π_θ(y_t | context)
    ref_log_probs:    (batch, seq_len) — log π_ref(y_t | context), detached
    advantages:       (batch, seq_len) — per-token advantages from GAE
    """
    # Importance sampling ratio ρ = π_θ / π_ref
    log_ratio = policy_log_probs - ref_log_probs   # log(ρ)
    ratio     = log_ratio.exp()                    # ρ

    # Clipped ratio: force ρ into [1-ε, 1+ε]
    ratio_clipped = ratio.clamp(1 - epsilon, 1 + epsilon)

    # Take the more conservative (min) objective
    pg_loss_unclipped = ratio         * advantages
    pg_loss_clipped   = ratio_clipped * advantages

    # Maximize the pessimistic bound → minimize the negative
    loss = -torch.min(pg_loss_unclipped, pg_loss_clipped).mean()
    return loss


# ─────────────────────────────────────────────────────────────
# GRPO Advantage Computation (the key innovation)
# ─────────────────────────────────────────────────────────────

def grpo_advantages(rewards: torch.Tensor) -> torch.Tensor:
    """
    rewards: (batch_size, G) — G rewards for each of the batch_size prompts
    Returns: advantages (batch_size, G) — z-score normalized within each group
    """
    # Baseline = mean reward within the group for each prompt
    baseline = rewards.mean(dim=-1, keepdim=True)   # (batch_size, 1)
    std      = rewards.std(dim=-1, keepdim=True)     # (batch_size, 1)

    # Advantage = how much better/worse than the group average
    advantages = (rewards - baseline) / (std + 1e-8) # (batch_size, G)
    return advantages


def grpo_training_step(policy_model, ref_model, prompts, reward_fn, G=8, temperature=0.8, epsilon=0.2):
    """
    Full GRPO training step for one batch of prompts.
    G: number of responses to sample per prompt
    """
    all_responses, all_log_probs_policy, all_log_probs_ref = [], [], []

    # ── Phase 1: Sample G responses per prompt ──────────────────
    with torch.no_grad():
        for prompt in prompts:
            group_responses = []
            for _ in range(G):
                # Sample with temperature to get diverse outputs
                response = policy_model.generate(prompt, temperature=temperature)
                group_responses.append(response)
            all_responses.append(group_responses)

    # ── Score all responses ──────────────────────────────────────
    rewards = torch.tensor([
        [reward_fn(resp) for resp in group]
        for group in all_responses
    ])  # (batch_size, G)

    # ── Compute group-normalized advantages ─────────────────────
    advantages = grpo_advantages(rewards)  # (batch_size, G)
    # Each token in a response inherits its sequence's advantage (coarse assignment)

    # ── PPO clipped update with GRPO advantages ──────────────────
    total_loss = 0
    for b, group in enumerate(all_responses):
        for g, response in enumerate(group):
            adv = advantages[b, g]  # scalar advantage for this response
            # Expand to token-level (every token gets same sequence advantage)
            token_adv = adv.expand(len(response))

            policy_lp = policy_model.log_probs(response)
            with torch.no_grad():
                ref_lp = ref_model.log_probs(response)

            total_loss += ppo_loss(policy_lp, ref_lp, token_adv, epsilon)

    total_loss /= (len(prompts) * G)
    total_loss.backward()
    return total_loss.item()


# ─────────────────────────────────────────────────────────────
# Using HuggingFace TRL — the practical way
# ─────────────────────────────────────────────────────────────

from trl import GRPOConfig, GRPOTrainer

def math_reward(response, ground_truth):
    """Verifiable reward: +1 if answer correct, 0 otherwise."""
    import re
    nums = re.findall(r'\d+\.?\d*', response)
    return 1.0 if nums and nums[-1] == ground_truth else 0.0

config = GRPOConfig(
    num_generations=8,          # G — responses per prompt
    temperature=0.8,
    max_new_tokens=512,
    learning_rate=1e-6,
    kl_coef=0.1,                # β — KL penalty coefficient
    cliprange=0.2,              # ε — PPO clip bound
)

trainer = GRPOTrainer(
    model=policy_model,
    ref_model=ref_model,
    reward_funcs=[math_reward],  # verifier as reward function
    args=config,
    train_dataset=dataset,
)
trainer.train()
```

---

## 🔬 Frontier Research & Papers

| Paper | Key Insight | Reference |
|---|---|---|
| **PPO (original)** | Clipped surrogate objective enables stable on-policy RL without trust-region constraints; generalizes across domains | Schulman et al., ICLR 2017 |
| **InstructGPT** | PPO + KL-penalized RLHF applied to GPT-3 at scale; first model to follow instructions reliably; defines the 3-stage pipeline | Ouyang et al., NeurIPS 2022 |
| **DeepSeekMath** | Introduces GRPO to replace PPO's value model with group-based advantage estimation; ~50% memory reduction | Shao et al., DeepSeek 2024 |
| **DeepSeek-R1-Zero** | Uses GRPO + verifiable rewards only (no SFT, no learned RM); discovers chain-of-thought, self-reflection, and length scaling spontaneously through RL | Guo et al., DeepSeek Jan 2025 |
| **DeepSeek-R1** | Full pipeline: cold-start SFT → GRPO/RLVR → distillation SFT → RLHF; demonstrates that RL on verifiable domains transfers to general reasoning | Guo et al., DeepSeek Jan 2025 |
| **DAPO** | Removes KL penalty entirely; adds clip-higher + token-level policy gradient + entropy bonus; improves AIME performance over standard GRPO | Yu et al., 2025 |
| **"It Takes Two: GRPO is Secretly DPO"** | Shows via gradient analysis that GRPO and DPO optimize similar contrastive objectives — GRPO is an online (exploratory) version of DPO; unifies the two families | 2025 |
| **RLVR Scaling Laws** | Verifiable reward RL obeys predictable compute scaling laws — more RL steps → better reasoning performance, analogous to pre-training scaling | Multiple labs, 2025 |

---

## ❓ Active Recall

- [ ] What is the fundamental instability in naive policy gradient that PPO's clipping solves? Draw what happens to the loss when $\rho_t \gg 1$ without clipping.
- [ ] Write the PPO clipped objective. Explain what the `min` operation achieves for $A_t > 0$ vs. $A_t < 0$ cases separately.
- [ ] What is the value model in PPO? What does it predict, and how is it trained? Why does it roughly double memory cost?
- [ ] Explain GAE (Generalized Advantage Estimation) in your own words — what problem does it solve, and what does the discount factor $\lambda$ control?
- [ ] What is GRPO's key innovation? Derive the advantage formula from scratch: what is the baseline, and how is it normalized?
- [ ] In the GRPO example with 4 responses scoring [0.9, 0.7, 0.4, 0.2], compute the baseline and each response's advantage.
- [ ] Why is GRPO's advantage "per sequence" while PPO's is "per token"? What are the practical implications for credit assignment?
- [ ] What happens in GRPO when all G responses get the same reward? How should you handle this?
- [ ] What is RLVR, and why does using verifiable rewards allow much longer RL training runs than using a learned reward model?
- [ ] DeepSeek-R1-Zero discovered chain-of-thought reasoning and self-reflection without those behaviors being explicitly trained. What does this suggest about the nature of emergent reasoning from RL?
- [ ] Compare DPO vs. PPO vs. GRPO on three dimensions: (1) whether they require a separate reward model, (2) whether they can explore beyond the training distribution, (3) memory cost relative to inference.
