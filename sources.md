# Red-Teaming Reading List

As of 2026-11-09. Links marked **(opened)** were fetched on that date; the rest are cited from memory and
their IDs were not re-checked, so confirm a link before you rely on it.

## Gray Swan

- [Gray Swan blog](https://www.grayswan.ai/blog) **(opened)**: competition announcements and results.
- [Your AI Agent Can Be Compromised. You'd Never Know.](https://www.grayswan.ai/blog/your-ai-agent-can-be-compromised-youd-never-know)
  **(opened)**: indirect prompt injection arena results, Mar 2026.
- [How Vulnerable Are AI Agents to Indirect Prompt Injections?](https://arxiv.org/abs/2603.15714)
  **(opened)**: the full paper behind that post (Dziemian, Lin, ... Kolter, 2026).
- [GraySwanAI/ipi_arena_os](https://github.com/GraySwanAI/ipi_arena_os): open-source scenarios, judging
  and sample attacks from that arena (linked from the post above).
- [UK AISI x Gray Swan Agent Red-Teaming Challenge: Results Snapshot](https://www.grayswan.ai/blog/uk-aisi-x-gray-swan-agent-red-teaming-challenge-results-snapshot)
  **(opened)**: 22 models, 44 behaviors, ~1.8M attempts, May 2025.
- [Security Challenges in AI Agent Deployment](https://arxiv.org/abs/2507.20526): the paper behind that
  challenge (NeurIPS 2025 Datasets & Benchmarks).
- [Gray Swan Introduces the Dangerous Reasoning Arena](https://www.grayswan.ai/blog/gray-swan-introduces-the-dangerous-reasoning-arena-competition)
  **(listed on the blog)**: the reasoning-model challenge.
- [nanoGCG](https://www.grayswan.ai/blog/nanogcg) **(listed on the blog)**: their GCG implementation.
- [AI red-teaming prompt engineering resources guide](https://app.grayswan.ai/arena/blog/ai-red-teaming-prompt-engineering-resources-guide):
  Gray Swan's own reading guide for red teamers (returned HTTP 429 when fetched; open it in a browser).
- [Fortune: Gray Swan Arena 2.0 and IPI Black](https://fortune.com/2026/10/20/exclusive-ai-security-company-gray-swan-red-team-arena-contest-to-jailbreak-ai-agents/)
  **(opened)**: Oct 2026 relaunch, $300k prize pool, rules on human-driven agents.
- [Gray Swan docs](https://docs.grayswan.ai/): check challenge rules here and on each challenge page.

## Inject Prompt

- [Archive](https://www.injectprompt.com/archive) **(opened)**: per-model jailbreak write-ups, Nov 2025 –
  Jul 2026. The posts' claims are the author's own informal tests.

## Pliny (elder-plinius)

- [GitHub profile](https://github.com/elder-plinius) **(opened)**
- [L1B3RT4S](https://github.com/elder-plinius/L1B3RT4S) **(opened)**: prompt collection, one file per vendor.
- [CL4R1T4S](https://github.com/elder-plinius/CL4R1T4S): collected system prompts of deployed products.

## Foundational papers

- [Jailbroken: How Does LLM Safety Training Fail?](https://arxiv.org/abs/2307.02483) (Wei et al. 2023)
- [Universal and Transferable Adversarial Attacks on Aligned LLMs (GCG)](https://arxiv.org/abs/2307.15043) (Zou et al. 2023)
- [Refusal in Language Models Is Mediated by a Single Direction](https://arxiv.org/abs/2406.11717) (Arditi et al. 2024) — the basis of abliteration
- [Safety Alignment Should Be Made More Than Just a Few Tokens Deep](https://arxiv.org/abs/2406.05946) (Qi et al. 2024)
- ["Do Anything Now": Characterizing In-The-Wild Jailbreak Prompts](https://arxiv.org/abs/2308.03825) (Shen et al. 2023)

## Automated attackers

- [PAIR: Jailbreaking Black Box LLMs in Twenty Queries](https://arxiv.org/abs/2310.08419) (Chao et al. 2023)
- [TAP: Tree of Attacks with Pruning](https://arxiv.org/abs/2312.02119) (Mehrotra et al. 2023)
- [AutoDAN](https://arxiv.org/abs/2310.04451) (Liu et al. 2023)
- [Jailbreaking Leading Safety-Aligned LLMs with Simple Adaptive Attacks](https://arxiv.org/abs/2404.02151) (Andriushchenko et al. 2024)
- [Best-of-N Jailbreaking](https://arxiv.org/abs/2412.03556) (Hughes et al. 2024)

## Multi-turn and long-context

- [Many-shot jailbreaking](https://www.anthropic.com/research/many-shot-jailbreaking) (Anthropic 2024)
- [Crescendo multi-turn jailbreak](https://arxiv.org/abs/2404.01833) (Russinovich et al. 2024)
- [LLM Defenses Are Not Robust to Multi-Turn Human Jailbreaks Yet](https://arxiv.org/abs/2408.15221) (Li et al. 2024)

## Reformulation, encoding, persuasion

- [Does Refusal Training in LLMs Generalize to the Past Tense?](https://arxiv.org/abs/2407.11969) (Andriushchenko & Flammarion 2024)
- [Low-Resource Languages Jailbreak GPT-4](https://arxiv.org/abs/2310.02446) (Yong et al. 2023)
- [GPT-4 Is Too Smart to Be Safe: Stealthy Chat with LLMs via Cipher](https://arxiv.org/abs/2308.06463) (Yuan et al. 2023)
- [ArtPrompt: ASCII Art-based Jailbreak Attacks](https://arxiv.org/abs/2402.11753) (Jiang et al. 2024)
- [How Johnny Can Persuade LLMs to Jailbreak Them](https://arxiv.org/abs/2401.06373) (Zeng et al. 2024)
- [DrAttack: Prompt Decomposition and Reconstruction](https://arxiv.org/abs/2402.16914) (Li et al. 2024)
- [Exploiting Programmatic Behavior of LLMs](https://arxiv.org/abs/2302.05733) (Kang et al. 2023)

## Reasoning models

- [H-CoT: Hijacking the Chain-of-Thought Safety Reasoning](https://arxiv.org/abs/2502.12893) (Kuo et al. 2025)

## Agents and prompt injection

- [Not What You've Signed Up For: Indirect Prompt Injection](https://arxiv.org/abs/2302.12173) (Greshake et al. 2023)
- [The Instruction Hierarchy](https://arxiv.org/abs/2404.13208) (Wallace et al. 2024)
- [AgentHarm](https://www.grayswan.ai/blog/agentharm) **(listed on the Gray Swan blog)**

## Defenses (know what you're up against)

- [Circuit Breakers: Representation Rerouting](https://arxiv.org/abs/2406.04313) (Zou et al. 2024)
- [Constitutional Classifiers](https://arxiv.org/abs/2501.18837) (Anthropic 2025)
- [StrongREJECT](https://arxiv.org/abs/2402.10260) (Souly et al. 2024): why a jailbreak that yields vague output scores as a failure

## Industry write-ups

- [Microsoft: Skeleton Key](https://www.microsoft.com/en-us/security/blog/2024/06/26/mitigating-skeleton-key-a-new-type-of-generative-ai-jailbreak-technique/) (2024)
- [Unit 42: Deceptive Delight](https://unit42.paloaltonetworks.com/jailbreak-llms-through-camouflage-distraction/) (2024)
- [HiddenLayer: Policy Puppetry](https://hiddenlayer.com/innovation-hub/novel-universal-bypass-for-all-major-llms/) (2025)
