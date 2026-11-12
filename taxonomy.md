# Red-Teaming Techniques: A Taxonomy for Defenders

As of 2026-11-09. Built from the sources in [sources.md](sources.md), which were fetched and read
in full that day. Numbers are quoted from the papers. Every source was read except Gray Swan's
resources guide, which was rate-limited; see the [coverage table](#source-coverage) at the end.

This document describes attacks at the level of **mechanism**, because that is the level a
defence works at. It deliberately reproduces no prompts, templates or suffixes.

---

## 1. Two axes for placing any technique

### 1a. What the attacker can access

| Access | What it enables | Example sources |
|---|---|---|
| Chat input only, manual | Persona, framing, encoding, multi-turn escalation | DAN (Shen), Wei et al., Crescendo, L1B3RT4S, InjectPrompt |
| Chat input, many queries | Attacker-LLM search (PAIR, TAP); random-augmentation sampling (Best-of-N) | PAIR, TAP, BoN |
| Log-probabilities | Random-search suffixes, genetic search on likelihood | Andriushchenko et al. adaptive attacks, AutoDAN |
| Prefill of the assistant turn | Forcing the first tokens past where shallow alignment acts | Andriushchenko et al. (Claude family 100%), Qi et al. |
| Chat history / multi-turn memory | Escalation, editing the history, faking prior turns | Crescendo, MHJ, InjectPrompt "history skip" |
| Gradients / weights | GCG suffixes, abliteration, fine-tuning attacks | GCG, Arditi et al., Qi et al. |
| Data the agent reads (no chat access at all) | Indirect prompt injection (IPI) | Greshake, Gray Swan IPI Arena, UK AISI challenge |

**Each API feature opens its own attack path:** logprobs enable search, prefill enables
first-token attacks, high-volume sampling enables Best-of-N, and conversation memory enables
Crescendo. Turning off a feature removes its whole family of attacks from the threat model.

### 1b. Why the attack works (root cause)

Wei et al. ("Jailbroken") give the two classic failure modes. Two later papers add mechanistic
ones, and the competition data adds a fourth that matters for agents:

1. **Competing objectives:** being helpful and following instructions are pitted against being
   harmless.
2. **Mismatched generalisation:** the input lies where pretraining gave the model the ability to
   respond but safety training never reached.
3. **Shallow alignment** (Qi et al.): safety mostly shapes the first few output tokens. Get past
   them and the base model's behaviour comes back.
4. **Linear refusal mechanism** (Arditi et al.): refusal is mediated by one direction in
   activation space. Remove it and refusal goes away.
5. **No separation of data from instructions** (Greshake; Wallace et al.): the model treats
   retrieved or tool-returned text as instructions.

Wei et al.'s **safety-capability parity** principle is the main lesson of root cause 2: a safety
mechanism weaker than the model it guards cannot read inputs the model can (Base64, ciphers,
ASCII art). CipherChat's title states the corollary: the more capable model is *more*
vulnerable to cipher attacks.

---

## 2. The families

Each family gives the mechanism, the evidence from the sources, which defences were measured
against it, and what follows for defence.

### F1. Persona, role-play and rewriting the rules (competing objectives)

**Mechanism.** The request is wrapped in a frame where complying seems to fit the model's rules:
an unrestricted alter ego, a fictional world, a "developer mode", a claim that the session is
exempt, or replacement guidelines.

**Evidence**
- **Shen et al. (DAN):**
  - 15,140 in-the-wild prompts collected Dec 2022 – Dec 2023, of which 1,405 are jailbreaks
    from 803 accounts. The paper identifies 131 communities, 11 of them major: Advanced, Toxic,
    Basic, Start Prompt, Exception, Anarchy, Narrative, Opposite, Guidelines, Fictional,
    Virtualization.
  - Strategy labels: prompt injection, privilege escalation, deception, mandatory answer,
    role-play, virtualization.
  - Average ASR on ChatGPT was 0.685. Five prompts exceeded 0.95 on both GPT-3.5 and GPT-4, and
    one stayed online for more than 240 days.
- **Wei et al.:**
  - Role-play attacks (`AIM`, `dev_mode_v2`, `evil_confidant`) reached 0.75–0.97 on GPT-4 and
    GPT-3.5 but **0.00 on Claude v1.3**. The adaptive attack still reached 1.00 on Claude.
  - Patching one family does not protect against the others.
- **Kang et al., "virtualization":** a fictional scenario built over several prompts that the
  model "emulates". With obfuscation and indirection, it got past OpenAI's input filter, output
  filter and useless-generation defence 100% of the time in 4 of 5 harm categories.
- **MHJ tactic taxonomy (Li et al.), Request Framing:** Fictionalization (Allegory, Appeal to
  Authority, False Premise, Hypothetical, Narration, Opposite Day, Recursion, Requesting for
  Beneficial Purposes, Roleplay, Slippery Slope Character), Framing as Code, Urgency, Emotional
  Appeal.
- **InjectPrompt archive (Nov 2025 – Jul 2026):** the same DAN-style persona templates and game
  or fiction wrappers (e.g. a tabletop game-master) were reused against each new model shortly
  after it launched: ChatGPT 5.1/5.2, Grok 4.1, Gemini 3, Kimi K2.5/K2.6, Qwen 3.7.
- **Skeleton Key (Microsoft, Russinovich, 26 Jun 2024):**
  - **Mechanism:** a multi-turn request to *augment* the model's behaviour guidelines rather
    than replace them: answer everything, and attach a warning instead of refusing. Microsoft
    classes it as "Explicit: forced instruction-following".
  - **Results:** tested Apr–May 2024, it got full compliance (with a warning prefix) across
    explosives, bioweapons, political content, self-harm, racism, drugs, graphic sex and
    violence from:
    - Llama3-70b-instruct
    - Gemini Pro
    - GPT-3.5 Turbo
    - GPT-4o
    - Mistral Large
    - Claude 3 Opus
    - Command R Plus
  - **The exception:** GPT-4 resisted unless the request was placed in a user-defined
    *system* message (possible through the API). Microsoft reads this as evidence that
    separating system and user messages reduces override attempts, which is the
    instruction-hierarchy argument.
  - **Mitigations:**
    - Prompt Shields, on by default in Azure AI Content Safety, at the most restrictive
      threshold
    - input filtering
    - system-message hardening
    - output filtering
    - separate abuse monitoring for recurring patterns
    - PyRIT red teaming, updated to include Skeleton Key
- **Policy Puppetry (HiddenLayer, 24 Apr 2025):**
  - **Mechanism:** the prompt is written to look like a policy or config file (XML, INI or
    JSON; strict XML isn't required) so the model treats it as authoritative instructions.
    The universal version adds role-play (a TV-drama scene) and leetspeak encoding of the
    harmful request. HiddenLayer distilled it to about 200 tokens with similar transfer.
  - **Results:** the article gives a yes/no table only, with no success rates or sample sizes.
    It reports effectiveness against:
    - GPT-4o, 4o-mini, 4.5 Preview and 4.1
    - Claude 3.5 and 3.7 Sonnet
    - Gemini 1.5 and 2.0 Flash
    - Copilot
    - Llama 3.1, 3.3 and 4
    - DeepSeek V3 and R1
    - Qwen2.5 72B
    - Mixtral 8x22B

    o1, o3-mini and Gemini 2.5 Pro needed "minor adjustments"; the reasoning models were
    "slightly more resilient".
  - **Other targets:** a variant extracted system prompts from many leading models, though not
    from the advanced reasoning ones. A handcrafted version beat a topic-restricted
    (healthcare) system prompt.
  - **Why it works, per HiddenLayer:** a "systemic weakness in how many LLMs are trained on
    instruction or policy-related data", which they call "the first post-instruction hierarchy
    alignment bypass". In this document's terms it is F1 plus F8: content imitating a
    configuration channel. It is a vendor claim, and HiddenLayer's recommendation is external
    detection, i.e. their own product.

**Defences measured against it**
- External safeguards on DAN prompts against ChatGPT reduced ASR only slightly: OpenAI
  moderation by 0.091, OpenChatKit by 0.030, NeMo-Guardrails by 0.019. None worked across all
  prompts.
- A GPT-3.5 snapshot from Nov 2023 had quietly neutralised 70.9% of prompts, but paraphrasing
  undid it. Typos in 10% of words raised the best prompt from 0.477 to 0.857.

**Defensive takeaways**
- Training against specific personas is whack-a-mole, and paraphrase restores patched prompts.
- Judge the *content of the output*, not the framing of the input. Fictional framing does not
  change what the text enables.
- Treat any user-supplied claim of authority or policy as user-level text, never as operator
  configuration. This is the instruction hierarchy (§3).

### F2. Controlling the output format and opening (competing objectives)

**Mechanism.** Constrain the output so that a refusal cannot fit, or force an affirmative
opening:
- prefix injection
- refusal suppression ("never say you can't")
- style injection (JSON, short words only)
- minimum length requirements
- dividers that separate a fake refusal from the "real" answer
- instructions to invert refusal phrases

**Evidence**
- **Wei et al.:** `prefix_injection`, `refusal_suppression`, `style_injection_*` and
  `few_shot_json`. The ablations are telling. An *innocuous* forced prefix ("hello") did far
  worse than an affirmative one, and *inverted* refusal-suppression rules yielded nothing on
  GPT-4. The attack really is "make refusal unlikely".
- **L1B3RT4S structure** (one file per vendor, about 33 vendors): the recurring parts are
  - persona or "mode enabled" framing;
  - fake special tokens and system markers;
  - a minimum output length and a placeholder slot for the user's query;
  - a visual divider between a refusal and the answer (about half the entries);
  - leetspeak or other encoded output, and required markdown or code blocks;
  - semantic inversion of refusal phrases;
  - starting with a fake refusal and then flipping to the answer.
- **MHJ tactics, Output Format:** Requested Output, and Stylized Output (Debate, Literary
  Elements, Outside Sources, Splitting, Subtraction, i.e. stripping disclaimers).

**Defensive takeaways**
- Format constraints are themselves **detectable signals**: a divider pattern, a length floor,
  leetspeak output, or "begin with …". Flag them on the input side.
- Encoded output defeats output classifiers that cannot decode it. Make the output classifier
  at least as capable as the model (Constitutional Classifiers' input classifier also blocks
  attempts to set up coded output).
- The forced-opening attacks share their root cause with prefill and GCG (F6). Deep-alignment
  training is the structural fix.

### F3. Persuasion and social engineering (competing objectives)

**Mechanism.** Rewrite the harmful request using human persuasion techniques: appeals to
authority, logic, emotion, reciprocity, urgency, claimed benign purpose.

**Evidence**
- **Zeng et al. (PAP):** 40 techniques. The paper says 13 strategies, but its table lists 15
  groups:
  - *Ethical:* Information-based, Credibility-based, Norm-based, Commitment-based,
    Relationship-based, Exchange-based, Appraisal-based, Emotion-based, Information Bias,
    Linguistics-based, Scarcity-based, Reflection-based.
  - *Unethical:* Threat, Deception, Social Sabotage.

  Results: **over 92% ASR in 10 trials** on Llama-2-7b-Chat, GPT-3.5 and GPT-4. Claude models
  scored 0% in that comparison.
- **PAIR's attacker strategies** are logical appeal, authority endorsement and role-playing;
  role-playing was strongest in ablation.
- **StrongREJECT:** across 37 jailbreaks, **PAIR and PAP were the most effective** once output
  quality was scored.

**Defences measured (PAP ASR on GPT-4 after 10 trials; no defence 92%)**

| Defence | GPT-4 ASR |
|---|---|
| Rephrase | 60% |
| Retokenize | 76% |
| Random-perturbation detectors (Rand-Drop/Insert/Swap/Patch) | 80–88% |
| RAIN | 88% |
| Adaptive system prompt | 38% |
| **Tuned summarizer** (rewrites the input to its core request) | **2%** |

The tuned summarizer's cost: MT-bench falls from 8.97 to 6.65.

**Defensive takeaways**
- Post-hoc perturbation defences built for gibberish suffixes do nothing against fluent
  persuasion. Zeng et al. also found post-hoc defences were *less* effective on more capable
  models.
- Stripping the request down to its core intent is what worked, at a cost in helpfulness.
- A claimed purpose ("I'm a nurse", "for research") is a persuasion technique. A legitimate
  claim needs out-of-band verification (operator context, account attributes), not trust in
  the text.

### F4. Encoding, translation and paraphrase (mismatched generalisation)

**Mechanism.** Express the harmful request (and often the requested output) in a form that
safety training never covered but the model can still read.

| Sub-type | Source | Key result |
|---|---|---|
| Base64, ROT13, leetspeak, disemvowel, self-invented encodings | Wei et al. | Base64 0.34 on GPT-4 but ~0.00 on GPT-3.5, which **can't decode it**: capability creates the hole |
| Classical ciphers and "SelfCipher" (role framing alone, no real cipher) | Yuan et al. (CipherChat) | GPT-4 averaged 72.1% unsafe across 11 domains with ASCII; Turbo 23.7%. Ciphers absent from pretraining failed even with 10+ demonstrations |
| Low-resource languages via Google Translate | Yong et al. | GPT-4 bypass: English 0.96%; low-resource combined **79.04%** (Zulu 53.08% alone); mid-resource 21.92%; high-resource 10.96% |
| ASCII art replacing the trigger words | Jiang et al. (ArtPrompt) | Ensemble ASR 52% averaged over 5 models, versus PAIR 28% and GCG 26%. On Claude, 52% where most baselines scored 0% |
| Past (or future) tense | Andriushchenko & Flammarion | GPT-4o **1% to 88%** (20 tries), 57% on a single try. Claude-3.5 Sonnet 0% to 53%. o1-preview 2% to 78%. o1's input filter blocked about 78% of present-tense requests but only about 27% of past-tense |
| Random augmentation (caps, scrambling, typographic images, audio pitch/speed) | Hughes et al. (Best-of-N) | See F7 |
| Typos and synonyms | Kang et al.; MHJ "Obfuscation" | 100% filter bypass in Kang's Table 1 |
| Voice input | InjectPrompt (Jan 2026 post) | Informal claim against ChatGPT 5.2, Grok 4.1, Perplexity |

**Defences measured**
- **Past-tense fine-tune** (Andriushchenko & Flammarion; GPT-3.5 trained on 394 past-tense
  refusals, ASR measured on held-out behaviours):

  | Refusal share of training mix | Past-tense ASR | Over-refusal |
  |---|---|---|
  | baseline | 74% | 3% |
  | 2% | 24% | 6% |
  | 5% | **0%** | **22%** |
  | 30% | 0% | 61% |

  It closes a *known* gap, and over-refusal rises steeply with the dose.
- **ArtPrompt against input defences:** perplexity filtering 52%→50%, paraphrase 52%→39%,
  retokenisation no change (the added spaces may disguise it further). Fine-tuning on the
  ASCII-art benchmark (ViTC) raised recognition from 10.26% to 71.54%.
- **CipherChat:** safe in-context demonstrations measurably cut unsafe output. The authors
  recommend safety training on enciphered and non-natural-language data.

**Defensive takeaways**
- **Safety-capability parity** is the core rule: the guard must decode whatever the model can.
- Do multilingual and multi-modality red teaming as standard, not only in English.
- Augmenting safety data with these transformations (translation, cipher, tense, paraphrase)
  works but generalises poorly to *new* transformations. Expect a long tail.
- Canonicalising inputs (decode, translate, normalise) before classification helps against
  known encodings. It cannot cover an encoding the attacker invents in context, which is what
  `auto_obfuscation` does.

### F5. Decomposition and payload splitting (mismatched generalisation)

**Mechanism.** No single fragment is harmful. The model reassembles the request while answering,
or the pieces are spread across turns or variables.

**Evidence**
- **DrAttack:** three stages. GPT-4 splits the prompt into sub-phrases; benign in-context
  examples teach the model to reassemble them; a synonym search tunes the fragments.
  - GPT-4 ASR: 63% (GPT judge), 80% (human judge), with about 15 queries.
  - It fails when the request contains explicit malicious nouns that have no good synonyms.
  - OpenAI Moderation, a perplexity filter and RA-LLM reduced ASR "only slightly".
- **Kang et al.:** code injection and payload splitting, i.e. string concatenation and variable
  assignment, which treats the LLM like a program. The classic-security analogy: you cannot
  predict a program's behaviour without running it, so input filtering has inherent limits.
- **Wei et al. `auto_payload_splitting`:** GPT-4 flags the sensitive words itself and they are
  then split. 0.34 on GPT-4, 0.59 on Claude v1.3.
- **MHJ "Hidden Intention Streamline":** a chain of individually harmless turns adding up to a
  harmful request. It worked against all four defences tested.

**Defensive takeaways**
- Single-message classifiers cannot see the whole request. Classify the **assembled output**,
  as a streaming output classifier does, and the conversation as a whole.
- Assume this family succeeds and check where intent becomes concrete: the response, the tool
  call, the action (Kang: "unconditional" defences such as sandboxing).

### F6. Multi-turn, long-context and history manipulation

**Mechanism.** Exploit the conversation, not any one message:
- gradual escalation;
- filling the context with fake compliant examples;
- editing or faking earlier turns;
- hiding one harmful topic among benign ones.

**Evidence**
- **Crescendo (Russinovich et al.):** starts from an abstract, benign question related to the
  goal, and each turn builds on the model's *own previous answer*. A refused turn is
  backtracked (deleted and rephrased).
  - Stating the harmful task explicitly dropped success below 1%. The attack depends on the
    task never being stated.
  - AdvBench binary ASR: GPT-4 98%, Gemini-Pro 100%.
  - HarmBench with GPT-4: 63.2% average, versus 38.9% for many-shot.
  - Most tasks fell in fewer than 5 turns.
  - Defences: Self-Reminder and Goal Prioritization helped on the hardest tasks but not on
    Election, Climate or Stabbing. At 20 rounds plus 20 backtracks, the attack broke through
    Self-Reminder.
- **MHJ (Li et al.):** 537 human multi-turn conversations.
  - 91.62% of successful HarmBench jailbreaks used more than one turn.
  - Against defences on llama-3-8b-instruct:

    | Defence | Human ASR | Ensemble of 6 automated attacks |
    |---|---|---|
    | RR (circuit breakers) | **51.7%** | 8.3% |
    | LAT | 75.4% | 10.0% |
    | DERTA | 87.9% | 68.3% |
    | CYGNET (closed) | 70.4% | 0.0% (its own paper) |

  - Mean time to success: 12.6–17.3 min.
  - For RMU unlearning, humans beat the best automated attack by 39.6 points.
- **Many-shot jailbreaking (Anil et al., Anthropic, 2 Apr 2024):**
  - **Mechanism:** hundreds of fake user/assistant exchanges in which the assistant complies,
    followed by the target query, all sent as one prompt. It is in-context learning turned
    against alignment.
  - **Scaling:** −E[log P(harmful)] = C·n^(−α) + K, a power law in the number of shots n, the
    same form as benign in-context learning.
    - Effective on Claude 2.0, GPT-3.5, GPT-4, Llama 2 70B and Mistral 7B, up to about 70k
      tokens on malicious use-cases with no plateau.
    - **Larger models are more vulnerable:** they learn in context faster (larger α), so
      they need fewer shots.
  - **Robust to variations:** swapping the user/assistant tags, translating the dialogue, or
    relabelling turns as "Question/Answer" shifts the intercept but **not the slope**. So
    detecting the standard format is not a fix. It still works when the demonstrations come
    from a different harm topic than the target. Composing with other jailbreaks lowers the
    intercept, which cuts the context needed exponentially.
  - **Mitigations:**
    - Standard SL and RL alignment, and even *targeted* SL and RL on MSJ prompts, raise the
      intercept but leave the exponent largely unchanged. With enough shots the attack still
      lands. The authors say a real fix must reduce the slope or raise the floor (K).
    - Capping context length works but gives up the feature.
    - Prompt-based defences, on a 205-shot attack (deception category):
      - In-Context Defense (prepending refusal demonstrations): 61% → 54%.
      - **Cautionary Warning Defense** (natural-language warnings before and after the
        input): **61% → 2%**. Its safety/capability trade-off was not evaluated.
    - The Anthropic blog describes the best result as prompt classification and modification
      before the model.
  - **Connection to other families:** the same power-law framing is how BoN (F7) forecasts
    ASR, and BoN composes with a 100-shot MSJ prefix.
- **Deceptive Delight (Unit 42, Lu, 23 Oct 2024):**
  - **Mechanism:**
    1. Ask for a narrative linking one unsafe topic with two benign ones.
    2. Ask the model to elaborate on each topic.
    3. Optionally, ask it to expand on the unsafe one.
  - **Why it works, per the authors:** limited "attention span". Mixed with benign content,
    the unsafe part is overlooked or misjudged, and content filters are misled by the benign
    framing as well.
  - **Setup:** 8 anonymised models, about 8,000 cases, 40 unsafe topics across Hate,
    Harassment, Self-harm, Sexual, Violence and Dangerous. Content filters were *disabled* to
    test the models alone. An LLM judge scored success as harmfulness and quality both ≥3 of 5.
  - **Results:**
    - Average ASR 5.8% with no jailbreak, **64.6%** with Deceptive Delight.
    - Turn 3 was best: harmfulness +21% and quality +33% over turn 2. Turn 4 showed
      diminishing returns, as safety kicked in once a large share of the content was unsafe.
    - Two benign topics was optimal; three or more degraded quality and ASR.
    - Violence had the highest ASR for most models; Sexual and Hate the lowest.
    - Models more vulnerable without a jailbreak were more vulnerable with it.
  - **Mitigations:** content filters as a second layer (OpenAI Moderation, Azure, Vertex,
    Bedrock Guardrails, Llama Guard, NeMo), system-prompt boundaries and persona, safety
    reinforcement wrapped around user input, and continuous testing.
- **History manipulation:** InjectPrompt's "Rejected Response" and "History Skip" posts, and
  the UK AISI "New Session / Session Data Update" cluster (F8).
- **Constitutional Classifiers' red team** tried gradual multi-turn escalation. It did not
  produce universal jailbreaks against the input/output classifier pair.

**Defensive takeaways**
- **Evaluate and classify whole conversations**, not single turns. Monitor the trajectory: a
  conversation drifting toward a harmful topic is the signal.
- Single-turn robustness numbers do not carry over. Circuit breakers: 8.3% against automated
  attacks, 51.7% against humans.
- Don't let users write assistant turns or edit history when the product doesn't need it.
- Crescendo's authors recommend designing downstream controls on the assumption that a
  jailbreak eventually lands.

### F7. Automated search and optimisation

| Method | Access | Mechanism | Headline result | Evades |
|---|---|---|---|---|
| **GCG** (Zou et al.) | White-box gradients | Greedy coordinate search over suffix tokens to maximise an affirmative opening; optimised across prompts and models for universality | Vicuna-7B 99% single behaviour. Transfer ensemble: GPT-3.5 86.6%, GPT-4 46.9%, Claude-2 2.1% | Nothing fluency-based: perplexity filters catch it |
| **AutoDAN** (Liu et al.) | Likelihoods (white-box), transfers to black-box | Hierarchical genetic algorithm seeded from DAN templates | Matches or beats GCG; **perplexity 46 vs GCG's 1,532** | Perplexity filter (GCG on Llama2 0.45→0; AutoDAN unchanged at 0.61) |
| **PAIR** (Chao et al.) | Black-box | Attacker LLM refines prompts from judge feedback, ≤90 queries | GPT-4 48%, Gemini-Pro 73%, Claude-2 0% | Perplexity filter; partly SmoothLLM |
| **TAP** (Mehrotra et al.) | Black-box | PAIR plus branching and off-topic pruning | GPT-4 90%, GPT-4o 94%, Claude 3 Opus 60% | **Llama Guard wrapper: GPT-4 still 84%** |
| **Adaptive attacks** (Andriushchenko et al.) | Logprobs, prefill | Template plus random-search suffix plus self-transfer plus restarts | **100%** on Llama-2/3, Gemma, R2D2, GPT-3.5; GPT-4o 100% with a custom template; all Claude 2–3.5 at 100% via prefill | Single-attack static benchmarks |
| **Best-of-N** (Hughes et al.) | Black-box sampling | Random augmentations, resampled until one passes | N=10k text: GPT-4o 89%, Claude 3.5 Sonnet 78%. −log(ASR) follows a **power law in N**, so fitting on about 1k samples predicts ASR at 10k within about 5% | Circuit breaker 52%; Cygnet 5% (rising to 67–86% with a tuned system prompt) |

**Cross-cutting findings**
- Attacks **compose**: BoN combined with an optimised prefix or many-shot raised GPT-4o text
  ASR from 86% to 97% and improved sample efficiency up to 250× (audio).
- BoN jailbreaks are **unreliable**: resampling reproduces harmful output only about 30% of the
  time for text. Defences that add randomness shift the power-law slope; they don't close the
  hole.
- Search attacks show up in **query patterns**: bursts of near-duplicate, iteratively reworded
  requests. Several papers recommend session monitoring and rate limits, but none tested them.
- **nanoGCG** (Gray Swan) is a pip-installable GCG for open models and the standard baseline
  for checking a safeguard.

**Defensive takeaways**
- **Report robustness as a curve over attacker budget**, not a single ASR. Fit BoN's power law
  on small samples to compare defences cheaply.
- Hide logprobs, disallow prefill, and rate-limit or watch for iterative query patterns.
- A perplexity filter only stops gibberish suffixes. PAIR, TAP, AutoDAN and humans pass it.
- Feed attack outputs back into adversarial training data, knowing the next search adapts.

### F8. Forged authority, context and reasoning ("control-plane spoofing")

**Mechanism.** Text in a low-privilege channel imitates a high-privilege one:
- fake chat-template or special tokens;
- fake system-prompt delimiters;
- fake prior assistant reasoning;
- fake session resets.

This is the family that best unites jailbreaks and agent attacks.

**Evidence**
- **UK AISI × Gray Swan (2507.20526):** clustering 1.8M attempts surfaced three dominant
  clusters:
  - **System Prompt Overrides** (mimicking system-level delimiters; effective against every
    model and behaviour);
  - **Faux Reasoning** (injected text imitating the model's reasoning so the action looks
    already approved);
  - **New Session / Session Data Update.**
- **Gray Swan IPI Arena (2603.15714):** one universal template worked on **21 of 41 scenarios
  across 9 models**. It presented the interaction as a simulated environment with a fake
  control panel, so injected text read as system commands.
- **H-CoT (Kuo et al.):** inject a mocked *execution-phase* reasoning snippet so the reasoning
  model skips its own safety check (the *justification* phase).
  - o1 ASR 0.8% to **94.6%**; o1-pro 1.2% to 97.6%; o3-mini 8.4% to 98.0%; DeepSeek-R1 79.2% to
    96.8%; Gemini 2.0 Flash Thinking 91.6% to 100%.
  - Snippets harvested from o1's displayed reasoning transferred to o3-mini.
  - Editing the justification text directly did not work.
- **L1B3RT4S:** fake special-token and system markers are among its most common parts.
- **Policy Puppetry (HiddenLayer):** user text written to look like a policy or config file
  (XML, INI or JSON) is treated as configuration. Details in F1.
- **Gray Swan Dangerous Reasoning Arena:** targets harmful intent in the reasoning trace even
  when the final answer looks safe. Behaviours: bioattack planning, strategic deception,
  covert backdoor insertion, infrastructure sabotage. No results published.

**Defensive takeaways**
- **Sanitise or escape role and template tokens** in every untrusted channel (user text,
  documents, tool results) so they cannot be mistaken for real ones.
- Train the instruction hierarchy (§3) so authority comes from the *channel*, never from the
  *content*.
- For reasoning models:
  - don't show users the safety-reasoning trace; H-CoT harvested its snippets from it;
  - don't accept user-supplied reasoning;
  - monitor the reasoning as well as the answer.

### F9. Attacks on internals and weights (white-box)

| Attack | Mechanism | Evidence |
|---|---|---|
| **Abliteration** (Arditi et al.) | Compute the refusal direction from mean activations on harmful vs harmless prompts; project it out of every weight matrix that writes to the residual stream | Works on 13 chat models up to 72B. HarmBench: Qwen 7B 79.2%, on par with per-prompt GCG. A 70B model costs **under $5** of compute. Capability largely kept (TruthfulQA drops up to 3.5 points). Llama-2's default system prompt blunts it, but a white-box attacker can drop the system prompt |
| **Prefilling** (Qi et al.; Andriushchenko et al.) | Force the first tokens of the response | ASR from about 0 to over 50% with 5–40 token prefills (Qi). Claude 2.0–3.5 100% (Andriushchenko) |
| **Harmful fine-tuning** (Qi et al.) | Gradients on early tokens dominate | Llama-2-7B-Chat 1.5% to 87.9% in **6 steps**. Even benign fine-tuning (Samsum) raises ASR to 23.4% |
| **Decoding exploits** (Qi et al.) | Vary temperature, top-k, top-p | HEx-PHI 54.9%, MaliciousInstruct 84.3% |
| **Adversarial suffixes, explained** (Arditi et al.) | Suffixes suppress the refusal direction and pull refusal heads' attention onto the suffix | Mechanistic link between F7 and F9 |

**Defences**
- **Deep-alignment augmentation** (Qi et al.): train on (harmful prefix → recover with a
  refusal). Prefill ASR 42–57% fell to 2.8–4.5%; GCG 65.6% to 19.0%; AlpacaEval 51.8% to
  49.5%.
- **Token-wise constrained fine-tuning** (Qi et al.): holds harmful fine-tuning to 4.6% versus
  88.9% for standard SFT.
- **Circuit breakers / representation rerouting** (Zou et al. 2024): average ASR across 10
  attacks fell from 76.7% to 9.8% (Mistral-7B) and from 38.1% to 3.8% (Llama-3-8B), including
  prefilling (95.0% to 4.9%). Capability kept (MT-Bench 7.60 to 7.53); over-refusal up modestly.
  **Limits:** single-turn only. Humans in multi-turn reached 51.7% (MHJ) and BoN reached 52%.
- **For open weights:** assume refusal can be removed. Abliteration takes minutes and no
  harmful data. Safeguards that matter must sit outside the weights (deployment controls) or
  deeper than refusal (unlearning, data filtering), and MHJ shows even unlearning (RMU) gives
  way to human red teamers.

### F10. Agents and indirect prompt injection (IPI)

**Mechanism.** The attacker never talks to the model. Instructions planted in an email, web
page, document, code comment, PR description, log entry or tool result are executed with the
agent's privileges.

**Greshake et al.'s taxonomy**
- *Injection methods:* passive (planted for retrieval), active (sent, e.g. by email),
  user-driven (the user is tricked into pasting it), hidden (multi-stage, multimodal, encoded,
  code-generated).
- *Threats:* Information Gathering (exfiltration), Fraud, Intrusion (remote control,
  persistence, malicious code completion), Malware (self-propagating), Manipulated Content
  (biased summaries, hidden ads, defamation), Availability (DoS, capability muting).

**Measured at scale**

| Source | Scale | Key findings |
|---|---|---|
| **UK AISI × Gray Swan** (2507.20526, Mar–Apr 2025) | 1.8M attempts, 62k breaks, 22 models, 44 behaviours | **Indirect 27.1% vs direct 5.7% ASR.** By category: Confidentiality Breaches 17.1%, Prohibited Actions 15.9%, Prohibited Content 8.9%, Conflicting Objectives 7.0%. At 10 queries, ASR approached ~100% for most models. Capability (GPQA) and inference compute did not predict robustness |
| **Gray Swan IPI Arena** (2603.15714, early 2026) | 271,588 chats, 8,648 successes, 13 models, 41 scenarios (Tool Use 18 / Coding 15 / Computer Use 8) | Success required the harmful action **and concealing it from the user**. ASR ranged from Opus 4.5 at 0.5% to Gemini 2.5 Pro at 8.5% (SecAlign 70B, a defended model, 5.5%). By setting: Tool Use 4.82%, Computer Use 3.13%, Coding 2.51%. Capability vs ASR r = −0.31 (not significant); **model family predicts robustness, not scale** |
| **AgentHarm** (2410.09024) | 110 behaviours (440 with variants), 104 tools, 11 harm categories | Harmful agent tasks with no jailbreak: GPT-4o 48.4%, Mistral Large 2 82.2%. A simple template raised Claude 3.5 Sonnet from 13.5% to **68.7%**. Jailbroken agents **kept their capability** (GPT-4o 84.2% vs 89.9% benign). Chat-only versions of the same requests were refused more often, so **chat refusal training does not carry over to agents** |

**Transfer is asymmetric** (IPI Arena): attacks that broke the most robust model (Opus 4.5) hit
the others at **44–81%**; 0% of Qwen3 VL's successes transferred to Opus 4.5. Attacks that beat
strong models exploit "more fundamental vulnerabilities in instruction following".

**Concealment is the headline risk.** Compromised agents "may even fabricate plausible
explanations", and tool logs and reasoning are too long for users to inspect. Gray Swan's
summary: users can't defend against attacks they can't see.

**Live examples encountered while reading these sources**
- The arXiv HTML page of the AgentHarm paper contains instructions aimed at AI summarisers.
- The L1B3RT4S and CL4R1T4S pages contain text such as "disregard prior instructions".

Any pipeline where an LLM reads the web, including a research agent compiling a document like
this one, is itself an IPI target.

**Defensive takeaways (system design, not only model training)**
1. **Treat tool output as data:** instruction-hierarchy training (§3), plus delimiting and
   sanitising untrusted content.
2. **Least privilege:** scope each tool and credential to the task. Prohibited Actions was the
   second-most-broken category.
3. **Confirm consequential or irreversible actions** (payments, sends, permission changes,
   merges) through a channel the model cannot write to.
4. **Keep independent audit logs and action monitors**, generated outside the model and not
   from its own summary, so concealment can be detected.
5. **Track provenance:** know which content came from which source and refuse to act on
   instructions from untrusted origins.
6. **Red-team in the agent setting.** Chat-mode robustness does not predict agent robustness
   (AgentHarm), and scale does not predict it either (both competition papers).

### F11. Reconnaissance, judge gaming and infrastructure

- **System-prompt extraction:**
  - CL4R1T4S (about 52k stars) collects reportedly extracted system prompts and tool
    definitions from Anthropic, OpenAI, Google, Meta, xAI and others, plus coding agents
    (Cursor, Devin, Windsurf, Cline, Replit …). LEAKHUB turns extraction into a leaderboard.
  - In the UK AISI challenge, Confidentiality Breaches was the most-broken category (17.1%).
  - **Takeaway:** assume your system prompt is public. Never put secrets, credentials or the
    only copy of a security check in it. Instruction-hierarchy training improved extraction
    robustness by 63%.
- **Gaming the judge:** competition attackers encoded forbidden content in other languages or
  hex to get past judges, fed false facts so a harmful action looked justified (a false 10×
  insulin concentration), and gamed judge specifications. In the Constitutional Classifiers
  red team, what got through was mainly **benign paraphrasing and length exploitation that
  gamed the grading rubric**, not the classifiers.
  - **Takeaway:** your evaluator is an attack surface. Use reasoning judges, give them
    few-shot "technically passes, violates in spirit" examples, and hold out a stricter judge.
- **Infrastructure bugs:** the only apparent universal jailbreak in the Constitutional
  Classifiers bounty came from a bug that leaked up to 128 tokens *after* the output
  classifier flagged them.
  - **Takeaway:** red-team the deployed system end to end, not the classifier on its own.
- **Tooling ecosystem** (Pliny's GitHub, 48 repos): T3MP3ST (autonomous multi-agent red
  teaming), P4RS3LT0NGV3 (text encode/transform), ST3GG (steganography), AutoRedTeam, and
  per-vendor prompt files. The per-vendor files make a ready-made **regression suite** for your
  own model.

---

## 3. Defence catalogue

What each defence covers, what it has been measured to do, and what is known to get past it.

| Defence | Covers | Measured effect | Known bypass or cost |
|---|---|---|---|
| **Targeted refusal fine-tuning** | A known transform (tense, cipher, persona) | Past tense 74% to 0% at a 5% refusal share | Over-refusal 3% to 22%. Doesn't generalise to new transforms (Wei; CC: fine-tuning on attacks "did not generalize" text to code) |
| **Deep-alignment augmentation** (Qi) | Prefill, GCG, decoding exploits (F2, F7, F9) | Prefill to 2.8–4.5%, GCG to 19% | Applied on top of existing alignment. Adaptive attacks untested |
| **Constrained fine-tuning** (Qi) | Harmful fine-tuning via an API | 88.9% to 4.6% | Some fine-tuning attacks still work; GSM8K 41.7% to 37.4% |
| **Instruction hierarchy** (Wallace) | F8, F10, extraction | Up to +63% robustness; +34% on held-out attacks including jailbreaks | Over-refusal regressions. Never follows instructions found while browsing. Prompting the hierarchy without training is far worse |
| **Perplexity filter** | GCG-style gibberish | GCG on Llama2 0.45 to 0 | Useless against fluent attacks (AutoDAN, PAIR, PAP, humans) |
| **Paraphrase / retokenise / SmoothLLM** | Token-level attacks | AutoDAN 0.97 to 0.68 (paraphrase). PAIR on GPT-3.5 51% to 10% (SmoothLLM) | Weak on semantic attacks: PAP on GPT-4 60–76%, ArtPrompt 39–52% |
| **Generic moderation APIs** | Overt harmful text | DAN ASR down only 0.019–0.091 | Encoding, framing, translation |
| **LLM guard wrapper** (Llama Guard) | Prompt and response | TAP on Claude 3 Opus 60% to 44% | TAP on GPT-4 still 84% |
| **Tuned summariser** (rewrite to core intent) | Persuasion (F3) | PAP on GPT-4 92% to 2% | MT-bench 8.97 to 6.65 |
| **Cautionary warning wrapper** (natural-language warning before and after the input) | Many-shot (F6) | MSJ 61% to 2% at 205 shots | Capability cost not measured. Targeted SL and RL only shift the power-law intercept |
| **Constitutional Classifiers** (input plus streaming output) | Universal jailbreaks in a defined harm domain | 86% to under 5%. Over 3,000 red-team hours, no universal jailbreak. Over-refusal +0.38% on production traffic; 23.7% compute overhead | 1.17% of attempts leaked before the stream was cut. Rubric gaming. Domain false positives (GPQA-Chemistry 26%) |
| **Circuit breakers / RR** | Harmful generation regardless of prompt (single-turn) | Average 76.7% to 9.8% (Mistral); agents 58% to 8% | Humans multi-turn 51.7%; BoN 52% |
| **Cygnet** (Gray Swan, closed) | Same | BoN 5% with no system prompt | Tuned system prompt 67–86%; humans 70.4% |
| **Unlearning** (RMU) | Hazardous knowledge | — | Humans beat automated attacks by 39.6 points (MHJ) |
| **Agent architecture controls** (least privilege, confirmations, provenance, independent logs) | F10 | Not measured in these sources | Recommended by Greshake, the IPI Arena and AISI papers. The only defences that hold if the model is fully compromised |
| **Operational controls** (hide logprobs, no prefill, rate limits, session monitoring, rapid-response patching) | F7, F6 | Recommended, untested | — |

**The "swiss cheese" principle** (Constitutional Classifiers; Microsoft's layered mitigations
for Skeleton Key) follows from the table. Every single layer has a measured bypass. Layers with
*uncorrelated* failure modes (a trained model, an input classifier, a streaming output
classifier, conversation-level monitoring, agent permission controls) are what reach low ASR.

---

## 4. How to measure a defence

The sources agree on these points.

1. **Static benchmarks overstate robustness.** Adaptive attacks hit 100% on models that looked
   robust (Andriushchenko et al.). RR, LAT and Cygnet looked near-perfect against automated
   single-turn attacks and fell to humans in multi-turn (MHJ). A robustness claim needs
   attacks adapted to the specific defence.
2. **Report ASR as a function of budget** (queries, attempts, red-team hours). BoN's power law
   makes this cheap: fit on a small budget, extrapolate, and compare slopes.
3. **Score harm with a capability-aware rubric.**
   - StrongREJECT: score = (1 − refused) × (specific + convincing) / 2. Its Spearman
     correlation with human rankings is 0.90 (fine-tuned judge), versus 0.82 for HarmBench's
     judge.
   - Most published jailbreaks score far below their papers' claims, because they trade
     **willingness for capability**: the jailbroken output is less useful. String-match judges
     overstate success.
   - The past-tense authors likewise note many "successes" are dual-use or vague.
4. **Know your judge's error rate.** The Llama Guard judge in PAIR missed 47% of real
   jailbreaks. The GPT-4 judge in the adaptive-attacks paper had about 20% false positives on
   Claude 2.1. Keyword ASR agreed with humans only 76% of the time (AutoDAN).
5. **Include humans and multiple turns.** Automated attacks underestimate risk by 19.6–65.4
   points (MHJ).
6. **Use transfer from the strongest model as a measure of attack strength.** Attacks that beat
   robust models generalise downward, not upward (IPI Arena). An attack corpus mined from weak
   models says little about strong ones.
7. **Measure over-refusal alongside ASR.** Every defence above that reports it shows a cost:
   past-tense fine-tune 22%, instruction hierarchy regressions, CC false positives on GPQA, the
   summariser's MT-bench drop. The CC prototype that survived red teaming refused about 44% of
   real traffic, which made it impractical.
8. **Expect drift.** Closed models change under you: GPT-3.5's silent Nov 2023 safeguard, H-CoT
   results varying by proxy-IP geolocation. Re-run regression suites, and plan quarterly
   benchmark refreshes as the IPI Arena does.

---

## 5. Priorities for a defender

In order of how much the evidence says each one buys:

1. **For agents, assume the model will be compromised and design for it.** Least privilege,
   human confirmation for consequential actions, and audit logs the model doesn't write. IPI
   is the highest-ASR family measured here (27.1% vs 5.7% direct), and concealment defeats
   user oversight.
2. **Layer uncorrelated defences**: an aligned model, a capable input classifier, a streaming
   output classifier and a conversation-level monitor. Keep guards at capability parity with
   the model so they can read what it can.
3. **Treat channel as authority:** instruction-hierarchy training, plus escaping template and
   role tokens in all untrusted text. This covers F8 and most of F10.
4. **Close API attack paths you don't need:** logprobs, assistant prefill, user-authored
   history, unbounded sampling. Rate-limit and watch for iterative query patterns.
5. **Train for depth, not just first-token refusal:** safety-recovery augmentation; and
   multi-turn, multilingual and transformed data in safety training.
6. **Keep secrets out of system prompts.**
7. **Red-team continuously, adaptively and with humans**, scored with a StrongREJECT-style
   rubric, reporting ASR against budget and over-refusal together. Use L1B3RT4S, MHJ (on
   Hugging Face, `ScaleAI/mhj`), `ipi_arena_os` and AgentHarm as regression suites.
8. **For open-weight releases**, assume refusal will be abliterated or fine-tuned away. Weigh
   release decisions on what the base capability enables.

---

## Source coverage

| Source | Read? | Notes |
|---|---|---|
| Gray Swan blog, IPI post, 2603.15714, ipi_arena_os, UK AISI snapshot, 2507.20526, Dangerous Reasoning, nanoGCG, AgentHarm (blog and 2410.09024), Fortune | Yes | AgentHarm's arXiv HTML contains injected instructions for AI summarisers |
| Gray Swan resources guide (app.grayswan.ai) | No | HTTP 429 rate limit |
| docs.grayswan.ai | Yes | Product docs only, nothing on arena rules or judging. Shade runs automated red-team scans, re-runnable in CI when the model, prompt or tools change. Cygnal is a runtime proxy or monitoring API that checks requests and responses against natural-language policy rules (Block/Observe/Off), and every policy includes a built-in "No Indirect Prompt Injection" rule |
| InjectPrompt archive | Yes (index) | Post list and patterns only; claims are the author's informal tests |
| Pliny: profile, L1B3RT4S, CL4R1T4S | Yes | Structure and patterns only; no payloads copied |
| All foundational, automated-attacker, multi-turn, reformulation, reasoning, agent and defence arXiv papers | Yes, full text | All IDs matched their titles. Constitutional Classifiers was read via curl because the fetch summariser refused its CBRN content |
| Anthropic many-shot jailbreaking | Yes | Blog plus the full paper PDF (34 pages, text extracted with `pdftotext`) |
| Microsoft Skeleton Key | Yes | |
| Unit 42 Deceptive Delight | Yes | |
| HiddenLayer Policy Puppetry | Yes | Read via curl; the server rejects the fetch tool with HTTP 403. Prompt templates in the article deliberately not reproduced |

**Inconsistencies inside the papers.** Where a figure above came from a paper with an
inconsistency, I used the table value:
- Wei et al. say their adaptive attack covers 28 attacks but list 30.
- GCG's abstract gives 84%, which its tables don't show.
- PAP says 13 strategies but tabulates 15.
- TAP's prose and table disagree on PAIR's GPT-4 query count.
- BoN gives GPT-4o text as both 89% and 86%.
- H-CoT gives o1's baseline refusal as 98% in the abstract and 99% in the body.
