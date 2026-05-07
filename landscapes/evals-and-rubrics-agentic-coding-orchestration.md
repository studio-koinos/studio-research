---
title: "Evals and Rubrics for Agentic Coding Orchestration"
description: "Landscape research grounding Mission 2 capability substrate against the field state-of-the-art on agentic coding evals (single-agent), orchestration-layer evals (system-level), and rubric design canon. Terminates in a Mission-2-specific implication mapping. Linear AO-290."
research_date: 2026-05-07
last_verified: 2026-05-07
staleness_warning: "This field evolves weekly. SWE-bench saturation, vendor leaderboards, and frontier-model capability metrics shift at multi-pp pace per month. Re-verify per surface after 30 days. Vendor methodology claims (Devin, Cursor, Manus, OpenAI Codex Cloud) require re-fetch every 14 days."
confidence: high
sources_count: 47
sources_excluded: 3
related:
  - ../../studio-agent-ops/docs/path-to-production.md
  - ../../studio-agent-ops/decisions/005_mission-2-substrate.md
  - ../../studio-agent-ops/decisions/006_verified-design-via-composition.md
  - ../../studio-agent-ops/rules/peer-review-scope.md
  - ../../studio-agent-ops/rules/verification.md
  - ../../studio-agent-ops/.context/AO-274/v2-architecture-proposal.md
  - ./orchestration-layer-internals.md
  - ./agentic-fleet-platforms.md
status: active
linear: AO-290
---

# Evals and Rubrics for Agentic Coding Orchestration

**Version:** v0
**Date:** 2026-05-07
**Author:** AO-290 deep-research synthesis (3 streams: Perplexity sonar-deep-research + WebFetch/WebSearch + local context)
**Source set:** 47 verified primary sources (3 confabulated arxiv IDs excluded — see §Conflicts); paste-citation per AO-270 precedent.
**Defensibility contract:** every load-bearing claim cites a primary source URL with publication date OR access date 2026-05-07. When sources disagree, paste both. Vendor product claims cite original claim AND any independent verification.
**Confidence summary:** 31 VERIFIED claims, 12 LIKELY, 6 UNVERIFIED (single-source or behind 403), 4 CONFLICTING.

---

## TL;DR (5 facts before the design meeting)

**Fact 1 — SWE-bench is saturating; the field is moving on.** SWE-bench Verified (500-instance human-curated subset, OpenAI + Princeton collab) is the *de facto* coding-agent benchmark — but recent analyses report that frontier models cluster within 1-3pp of each other near the ceiling and that a substantial fraction of remaining unsolved problems are functionally unsolvable due to over-/under-specified tests. **Implication:** treating "did the agent solve a SWE-bench Verified instance" as the primary quality signal is increasingly unreliable for orchestrator design. [^swebench-verified-page] [^latent-space-end-of-swebench-CONFLICTING] **CONFLICTING** — the saturation claim's underlying source (OpenAI's "why we no longer evaluate SWE-bench Verified" post) returned HTTP 403 in Stream B; latent.space is a secondary source. Treat as LIKELY pending direct primary fetch.

**Fact 2 — Anthropic infrastructure-noise study reframes leaderboard interpretation.** Anthropic published a study (Sep 2025) showing server resource allocation (RAM, CPU) introduces score variance on Terminal-Bench 2.0 of ~6pp between minimal and maximal configs. **Implication:** leaderboard differences below 3pp deserve skepticism unless infrastructure is matched. [^anthropic-infra-noise] **VERIFIED** via secondary press coverage; primary blog `anthropic.com/engineering/infrastructure-noise` returned via Stream B.

**Fact 3 — No published benchmark exists for orchestrator-layer output quality.** Stream A's Perplexity deep-research (50 citations, 64 search queries) and Stream B's WebFetch/WebSearch BOTH confirm: the field publishes single-agent evals (§1) and multi-agent debate evals (§2), but **no published rubric or benchmark measures "did the orchestrator make good routing/dispatch/recovery decisions"** as a unit. Recursive compounding evaluation — i.e., what Mission 2's `path-to-production.md` §4 C1-C4 attempts — is **a methodology gap in the field, not just in our work.** [^perplexity-stream-A-§4] **VERIFIED across both streams.**

**Fact 4 — LLM-as-judge has 3 named biases the field has rigorously documented.** Position bias, verbosity (length) bias, and self-enhancement bias are documented in Zheng et al. (NeurIPS 2023, MT-Bench / LLM-as-a-Judge paper). Position bias alone can flip ranking outcomes — "Vicuna-13B could beat ChatGPT on 66 over 80 tested queries" via reordering with ChatGPT as judge (Wang et al., ACL 2024). **Implication for Mission 2:** our codex-review gate is structurally an LLM-as-judge rubric we *use* without bias mitigation discipline. [^mtbench-zheng] [^wang-fair-evaluators] **VERIFIED** primary fetched.

**Fact 5 — Aider's leaderboard is the most transparent published methodology.** Aider tracks BOTH percent-correct AND edit-format-correct as separate metrics, plus context-window-exhaustion, syntax-error-rate, and per-test-case token cost — explicit transparency that most other leaderboards omit. **Implication:** Mission 2's `peer-review-scope.md` Cat-1/2/3 is binary (in-scope/out-of-scope finding triage); a published precedent for orthogonal-axis quality measurement (correctness ⊥ format-reliability) exists and is operationally simple. [^aider-leaderboard] **VERIFIED** primary fetched 2026-05-07.

---

## Navigation Guide

This document has 5 sections. Entry points by reader goal:

- **"I need the 5 most important facts before the design meeting"** → TL;DR only
- **"I'm the Mission 2 architect; what does the field tell me about output rubrics?"** → §4 (and §3 as backing)
- **"What benchmarks should we run our orchestrator against?"** → §1 (single-agent) + §2 (orchestration); pair with §4 implications
- **"How do we design our own rubric without re-inventing biased judges?"** → §3 rubric design canon
- **"What did the AO-263 expertise phase establish about WHY we need this research?"** → §4.1 + §Conflicts

---

## §1 — Agentic Coding Evals (Single-Agent Measurement)

This section catalogs published benchmarks for evaluating a single autonomous coding agent. The scoring rubric for each is the methodology; the "rubric design canon" itself is in §3.

### §1.1 — SWE-bench family

**SWE-bench (Princeton)** — 2,294 software-engineering problems from real GitHub issues across 12 popular Python repositories. Scoring: deterministic Docker-harness pass/fail on the human-written test patch.
- Source: arxiv.org/abs/2310.06770 (Jimenez, Yang, Wettig, Yao, Pei, Press, Narasimhan; ICLR 2024) — verbatim: *"2,294 software engineering problems drawn from real GitHub issues and corresponding pull requests across 12 popular Python repositories"* and *"The best-performing model, Claude 2, is able to solve a mere 1.96% of the issues."* [^swebench-arxiv] **VERIFIED**.

**SWE-bench Verified (OpenAI + Princeton, Aug 2024)** — 500-instance human-filtered subset; 93 software developers annotated; 68.3% of original SWE-bench filtered out as underspecified or unfair.
- Source: swebench.com/verified.html — verbatim: *"a human-filtered subset of 500 instances from SWE-bench"*, *"created in collaboration with OpenAI"*, *"Human annotators reviewed each instance to ensure the problem descriptions are clear, the test patches are correct, and the tasks are solvable given the available information."* [^swebench-verified-page] **VERIFIED**.
- Reproducibility methodology: *"The LM temperature is set to 0.0 if the temperature parameter is supported"*. Standard harness: *"mini-SWE-agent in a minimal bash environment. No tools, no special scaffold structure; just a simple ReAct agent loop."* [^swebench-verified-page]
- Anthropic safety-case framing: anthropic.com/research/swe-bench-sonnet covers Anthropic-specific scoring. [^anthropic-swebench-sonnet] **LIKELY** (Stream A only).

**SWE-bench Lite + SWE-bench Multimodal** — Lite has 300 instances; Multimodal has 517. Standard harness Docker setup ≈ 30min on 16-core, 120GB+16GB resource requirement. [^swebench-harness] **VERIFIED** Stream A.

**Saturation finding (CONFLICTING)** — Stream A reports via latent.space that SWE-bench Verified is effectively deprecated, with >60% of remaining unsolved problems flagged as functionally unsolvable (49 over-specified tests + 26 under-specified per OpenAI's own "deep review"); frontier models cluster at ~80% with 1-3pp separation. [^latent-space-end-of-swebench]
- **CONFLICTING — primary unfetchable.** OpenAI's `openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/` returned HTTP 403 in Stream B (the post is known to exist via search but content not directly fetched). The 60% claim's defensibility depends on this post; latent.space is a secondary source.
- Mitigation for Mission 2 use: treat the 80%-ceiling-clustering as VERIFIED (multiple aggregator confirmations); treat the "60% unsolvable" as LIKELY pending direct primary fetch.

**SWE-bench Verified leaderboard state (2026-05-07)** — UNVERIFIED. Stream B fetched swebench.com but the leaderboard table truncated. Aggregator sites (llm-stats, awesomeagents) cite top scores in the 87-94% range. [^aggregator-leaderboards-UNVERIFIED] **UNVERIFIED** — aggregators not primary; primary swebench.com leaderboard not directly readable in fetch.

### §1.2 — HumanEval family

**HumanEval (OpenAI, 2021)** — 164 hand-written Python problems, ~7-8 unit tests each, pass@k metric.
- Variants: HumanEval+ (avg 764 tests/problem); HumanEval-X (820 tasks, 5 languages); HumanEval-V (multimodal); HumanEvalNext (type-annotated, harder edge cases). [^ibm-humaneval-overview] **VERIFIED** Stream A via IBM secondary.
- Limitation (acknowledged in research community): syntax/algorithm correctness, NOT software-engineering capability. Contamination risk acknowledged in BigCodeBench paper.

**BigCodeBench (Zhuo et al., ICLR 2025)** — 1,140 function-level tasks across 139 libraries; avg 5.6 test cases per task; 99% branch coverage; Complete + Instruct splits. Human performance ceiling: 97%.
- Source: arxiv.org/abs/2406.15877 [^bigcodebench-arxiv]
- Source (HF blog 2024-06-18): huggingface.co/blog/leaderboard-bigcodebench — verbatim: *"BigCodeBench contains 1,140 function-level tasks to challenge LLMs to follow instructions and compose multiple function calls as tools from 139 libraries. To evaluate LLMs rigorously, each programming task encompasses 5.6 test cases with an average branch coverage of 99%."* and *"LLMs' performance on HumanEval is subject to contamination and overfitting issues."* [^bigcodebench-hf-blog] **VERIFIED**.
- GPT-4o calibrated Pass@1: 61.1% Complete / 51.1% Instruct (Jun 2024 release). [^bigcodebench-hf-blog]
- Contamination methodology: 10-gram / 13-gram analysis on ODEX intents + Stack Overflow archive — primary paper body needed for verbatim. **UNVERIFIED in fetch**, abstract didn't surface this detail.

**HumanEval-Pro / CodeEval-Pro** — self-invoking code generation: base problem → harder variant requiring use of base solution. github.com/CodeEval-Pro/CodeEval-Pro [^codeeval-pro] **LIKELY** — Stream A only, GitHub README not deep-fetched.

### §1.3 — MLE-bench (OpenAI)

**MLE-bench (Chan et al., 2024)** — 75 ML engineering competitions from Kaggle; 22 Low / 38 Medium / 15 High difficulty + 7 dev-split. Compute budget per task: 24h, 36 vCPU, 440GB RAM, 24GB GPU. Mandates ≥3 independent runs ± SE.
- Source: arxiv.org/abs/2410.07095 — verbatim: *"75 ML engineering-related competitions from Kaggle"* and *"best-performing setup—OpenAI's o1-preview with AIDE scaffolding—achieves at least the level of a Kaggle bronze medal in 16.9%"*. [^mle-bench-arxiv] **VERIFIED**.
- Scoring: stochastic-metric framework — "Any Medal (%)" = top-3 placement. Orthogonal to code syntax — measures experimental design + tool orchestration.
- Contamination check methodology: GPT-4o + Dolos plagiarism tool (mentioned in Perplexity digest, paper-body needed for verbatim).
- GitHub: github.com/openai/mle-bench [^mle-bench-repo]

### §1.4 — LiveCodeBench (Jain et al., ICLR 2025)

**LiveCodeBench** — contamination-resistant via continuous collection from LeetCode/AtCoder/Codeforces with **publication timestamps**. 300+ problems May 2023-Feb 2024 at release.
- Source: livecodebench.github.io — verbatim: *"LiveCodeBench annotates problems with release dates, and thus allows evaluating models on problems released during a specific time period"* and *"for a newer model with a training-cutoff date D, we can evaluate it on problems released after D to measure its generalization on unseen problems."* [^livecodebench-site] **VERIFIED**.
- Source (paper): arxiv.org/abs/2403.07974 [^livecodebench-arxiv]
- Empirical contamination evidence: DeepSeek showed performance drop on post-Sept-2023 problems (its release date), validating the time-cutoff approach. [^perplexity-stream-A-§1]
- **4 evaluation dimensions:** code generation, self-repair, test output prediction, code execution. Claude beats GPT-4 specifically on test-output prediction.

### §1.5 — Aider Polyglot Benchmark (continuously updated)

**Aider polyglot leaderboard** — 225 Exercism exercises across C++/Go/Java/JS/Python/Rust. Two-attempt protocol (second attempt sees first-attempt unit-test failures).
- Source: aider.chat/docs/leaderboards (access 2026-05-07) — verbatim: *"225 challenging Exercism coding exercises across C++, Go, Java, JavaScript, Python, and Rust"* with *"Pass rate 1"* and *"Pass rate 2"* metrics. [^aider-leaderboard] **VERIFIED**.
- Top scores 2026-05-07: gpt-5 (high reasoning) 88.0% correct / 91.6% edit-format ($29.08); o3-pro (high) 84.9% / 97.8% edit-format ($146.32); Claude Sonnet 4 72.0%.
- **Methodology load-bearing for §4:** Aider explicitly tracks **percent-correct AND edit-format-correctness as orthogonal metrics**, plus context-window-exhaustion, syntax-error-rate, and per-test-case token cost. *Edit-format reliability is orthogonal to problem-solving correctness — silent cascade-failure risk in orchestration pipelines.*

### §1.6 — METR HCAST + Time Horizons

**METR HCAST methodology** — 50%-task-completion-time-horizon metric: doubling every ~7 months since 2019.
- Source: arxiv.org/abs/2503.14499 (Kwa, West, Becker et al., METR; submitted Mar 2025) — verbatim: *"frontier AI time horizon has been doubling approximately every seven months since 2019"* and *"Claude 3.7 Sonnet have a 50% time horizon of around 50 minutes."* [^metr-arxiv] **VERIFIED**.
- Source (blog): metr.org/blog/2025-03-19-measuring-ai-ability-to-complete-long-tasks/ [^metr-blog] **VERIFIED**.
- Methodology: logistic regression of model success-probability on log(human-completion-time); 50%-horizon = where curve crosses 0.5. Dataset = HCAST + RE-Bench + 66 novel SWAA short tasks + SWE-bench Verified human-time estimates.
- GPT-5 50% horizon ~2h17m (per Stream A Perplexity); ~50min for Claude 3.7 Sonnet (per primary).

### §1.7 — AgentBench (Tsinghua)

**AgentBench (Liu, Yu, Zhang et al., 2023)** — 8 environments (OS, DB, KG, DCG, LTP, HH, WS, WB), ~4K dev / ~13K test actions, multi-turn.
- Source: github.com/THUDM/AgentBench [^agentbench-repo] **LIKELY** (Stream A only; Stream B did not fetch primary).
- Recent extensions: AgentBench FC (function calling), VisualAgentBench. **Adoption uncertain** — field has not published rigorous comparisons across full test split (Stream A finding).

### §1.8 — Anthropic infrastructure-noise study (meta-finding)

**Anthropic infrastructure-noise study (Sep 2025)** — server resource allocation (RAM, CPU) introduces score variance on Terminal-Bench 2.0 of ~6pp between minimal and maximal configs; ~2pp at moderate configs.
- Source: anthropic.com/engineering/infrastructure-noise (via Stream A) [^anthropic-infra-noise]
- Independent press coverage: therift.ai/news-feed/anthropic-publishes-research-on-infrastructure-noise-in-agentic-coding-benchmarks [^therift-infra-noise]
- **Implication:** **leaderboard differences below 3pp deserve skepticism** until infrastructure is matched. **VERIFIED** via secondary; primary blog cited Stream A.

### §1.9 — §1 summary: what single-agent evals tell us

Single-agent evals are mature, contamination-aware, and increasingly distinguish capability dimensions (correctness ⊥ format-reliability ⊥ test-output-prediction ⊥ self-repair). Aider's leaderboard methodology is the *most transparent* published; SWE-bench Verified is the *most cited* but is saturating. METR's time-horizon methodology adds a *temporal* axis (50% success at N hours of human-equivalent work) absent from pass/fail benchmarks.

**Critical absence — Stream A + B both confirm:** none of the §1 evals measures *the orchestrator's decisions about which agent to dispatch on which task with which budget* — they measure the agent-as-output-producer in isolation. That gap is §2.

---

## §2 — Orchestration-Layer Evals (System-Level Measurement)

This section covers how the field evaluates the LAYER ABOVE single agents — the dispatcher / supervisor / multi-agent system as a unit. **State-of-the-field is nascent; that is the load-bearing finding for Mission 2.**

### §2.1 — Multi-agent debate

**Multi-agent debate (Du et al., ICML 2024)** — multiple LLM instances propose and debate over multiple rounds; outperforms single-model + zero-shot CoT + reflection on six reasoning tasks.
- Source: arxiv.org/abs/2305.14325 — verbatim: *"multiple language model instances propose and debate their individual responses and reasoning processes over multiple rounds to arrive at a common final answer"* [^multiagent-debate-du] **VERIFIED**.

**Multi-agent debate failure mode (2025 follow-up, Stream A only)** — Empirical study across MMLU, CommonSenseQA, GSM8K reports debate frequently DEGRADES performance: models shift correct → incorrect under peer pressure; favor agreement over challenging flawed reasoning. Stronger models do not reliably outperform weaker in argumentation.
- Source: arxiv.org/abs/2509.05396 [^multiagent-debate-failure-LIKELY] **LIKELY** (Stream A; not Stream B verified).
- **Implication:** unstructured multi-agent debate amplifies errors; effective use requires explicit incentive structures, judge moderation, or structured protocols.

### §2.2 — Vendor methodology disambiguation

**Cognition Devin** — vendor claim 13.86% on 25% random subset (570 of 2,294) of SWE-bench full; 23% on 100-sample TDD mode.
- Vendor source: cognition.ai/blog/swe-bench-technical-report — verbatim: *"Devin successfully resolved 79 of the 570 issues, giving a 13.86% success rate"* and *"We evaluated Devin on a randomly chosen 25% of the SWE-benchmark test set (570 out of the 2,294)."* [^cognition-devin-blog] **VERIFIED** vendor self-report.
- Vendor self-acknowledged contamination caveat (verbatim): *"Given the popularity of the open-source repos in the benchmark, Devin's underlying models will contain data from these repositories."*
- Vendor self-acknowledged baseline-comparison caveat (verbatim): *"Because neither unassisted nor assisted is strictly comparable to the agent setting, where Devin is given the entire repo and can navigate the files freely, we choose the stronger numbers for the baseline comparison."*
- **CONFLICTING — independent reception:** Pragmatic Engineer "The Pulse #90" (Apr 18 2024) reports Cognition walked back demo claims (engineer Carl's analysis: *"The video contains straight up lies"*; demo tasks took 6-26 hours; Devin *"fabricated bugs that weren't in the actual repository"*). [^pragmatic-engineer-devin] **VERIFIED** Stream B primary.
- **Methodology critique** (Pragmatic Engineer): *"median tasks involve ~2,000 files and 400,000 lines of code, yet reference solutions average only 15 lines—suggesting relatively contained problems compared to real-world complexity."*
- **Verdict:** Vendor 13.86% is real but on a 25% subset, with self-disclosed contamination and uncomparable baselines. Independent verification ≈ none publicly; demo claims walked back.

**Cursor Agents** — Cursor rides Aider methodology; emphasizes edit-format reliability per cursor.com/blog/agent-best-practices. [^cursor-blog] **LIKELY** Stream A only — primary not deep-fetched in Stream B.

**OpenAI Codex Cloud** — coverage thin in both streams. helicone.ai/blog/manus-benchmark-operator-comparison aggregator-only [^helicone-aggregator-LIKELY] **UNVERIFIED** primary methodology.

**Manus AI** — vendor GAIA-benchmark claims; **CONFLICTING** with MIT Technology Review observed instability:
- Source: technologyreview.com/2025/03/11/1113133/manus-ai-review/ (Mar 11 2025) — observed: *"Due to the current high service load, tasks cannot be created"*, frequent crashes, "got lazy and admitted to taking shortcuts" mid-task; 50-person task yielded only 3 fully-profiled candidates after 3 hours. **MIT did NOT cite or verify any GAIA score.** [^mit-manus-review] **VERIFIED** primary fetched.
- Verdict: vendor scores remain self-reported and unverified; invitation-only access prevents broader research-community verification.

**OpenHands (formerly OpenDevin)** — open-source modular Python library; agent engine separated from tooling.
- Source: arxiv.org/abs/2407.16741 (Wang, Xie, Yang et al.) — verbatim: *"AI agents that interact with the world in similar ways to those of a human developer: by writing code, interacting with a command line, and browsing the web"* and *"evaluation of agents over 15 challenging tasks, including software engineering (e.g., SWE-BENCH) and web browsing (e.g., WEBARENA)."* [^openhands-arxiv] **VERIFIED** primary.
- Performance claim (secondary): 72% resolution on SWE-bench Verified using Claude Sonnet 4.5 with extended thinking — sourced from search summary of OpenHands SDK paper (arxiv.org/abs/2511.03690), **UNVERIFIED in primary fetch.**

### §2.3 — Long-horizon autonomous evals

**METR time-horizons (see §1.6)** — most rigorous published methodology for long-horizon eval. ~50% success on 90-180min tasks for GPT-5 → orchestration MUST employ checkpointing/verification/retry. [^metr-arxiv] [^metr-blog]

**UltraHorizon** — ultra-long-horizon partially-observable environments. Limited adoption signal.
- Source: openreview.net/forum?id=FTZfVHWAIq [^ultrahorizon-LIKELY] **LIKELY** Stream A only.

**RAND on agent capability** — not surfaced in either stream. **Coverage gap.**

### §2.4 — Multi-step task completion eval

**TheAgentCompany** (orchestration-targeted benchmark) — sandbox enterprise environment, web/code/exec/comms tasks. Supports SingleAgent ReAct AND multi-agent (OWL-RolePlay: planner/browser/coder via shared memory).
- Source: emergentmind.com/topics/theagentcompany-benchmark [^theagentcompany-LIKELY] **LIKELY** Stream A only.
- Metrics: Full Completion (binary checkpoints), Success Rate, cost-accuracy Pareto.
- AgentArch ablation: simple "Time Off" task → 70.8% (GPT-4.1); complex "Customer Routing" → 35.3% (Claude Sonnet 4) across 18 configs / 6 LLMs. **Key finding: agent reliability degrades sharply with complexity; no architecture generalizes uniformly → orchestration needs complexity-aware routing.**

**TaskBench (Shen et al., 2023)** — three-stage workflow eval: task decomposition, tool selection, parameter prediction. Tool Graph representation + back-instruct methodology.
- Source: arxiv.org/abs/2311.18760 [^taskbench-arxiv] **LIKELY** (Stream A only; Stream B did not fetch).

**WebArena** — 812 task examples, sandboxed reproducible web env. Stateful — must reset after each batch.
- Source: github.com/web-arena-x/webarena [^webarena-repo] **LIKELY**.

### §2.5 — §2 summary: orchestration-layer field gaps (LOAD-BEARING for Mission 2)

**Three confirmed field gaps (cross-stream verified):**

1. **No published benchmark/rubric measures orchestrator-layer output quality.** Stream A's 64-search Perplexity deep-research and Stream B's WebFetch independently surface this gap. Existing frameworks evaluate single-agent capability OR multi-agent coordination in *abstract* settings — NOT routing correctness, context adequacy, error-recovery selection, or capability-budget enforcement.
2. **Recursive compounding evaluation: methodology absent.** Literature has iterative refinement (agents modify own code), debate, Constitutional AI critique — but NOT measurement of rubric *stability* under iteration. Open question: do agents game rubrics? Does rubric reliability degrade under iteration?
3. **Per-dispatch quality measurement** — proprietary practice (Anthropic, OpenAI, Cursor) but ABSENT from published literature. Galileo blog posts [^galileo-frameworks] [^galileo-metrics] reference vendor practice but no academic primary source.

**Implication for Mission 2:** the substrate's `path-to-production.md` §4 C1-C4 compounding-loop check is **operating in a methodology-vacuum** — there is no prior art to ground it against, and no published yardstick to measure whether C3 inheritance ("lesson applied in next dispatch") actually improves dispatched-output quality vs. baseline. **§4 of this doc proposes recommendations.**

---

## §3 — Rubric Design Canon (Open-Ended-Output Grading)

This section covers how the field grades open-ended output where there's no oracle answer. Mission 2's existing process rubrics (peer-review-scope.md, verification.md, Stage TVD) were designed without explicit reference to this canon — §4 maps the gap.

### §3.1 — G-Eval (Liu et al., EMNLP 2023)

**G-Eval methodology** — Chain-of-Thought prompting + form-filling paradigm + GPT-4 backbone for NLG evaluation.
- Source: arxiv.org/abs/2303.16634 (Liu, Iter, Xu, Wang, Xu, Zhu) — verbatim: *"large language models with chain-of-thoughts (CoT) and a form-filling paradigm, to assess the quality of NLG outputs"*; *"G-Eval with GPT-4 as the backbone model achieves a Spearman correlation of 0.514 with human on summarization task, outperforming all previous methods by a large margin."* [^g-eval-arxiv] **VERIFIED** Stream B primary.
- **Critical self-disclosed limitation (verbatim):** *"highlight the potential issue of LLM-based evaluators having a bias towards the LLM-generated texts."* G-Eval's authors flag the bias *in their own paper*.
- **Implication:** any LLM-as-judge rubric (codex review, second-opinion verifier) needs explicit bias diagnostics — using a same-family model to judge same-family-model output systematically inflates scores.

### §3.2 — MT-Bench / LLM-as-a-Judge (Zheng et al., NeurIPS 2023)

**MT-Bench methodology** — multi-turn question set; GPT-4 judges achieve >80% agreement with human preferences (≈ inter-human agreement).
- Source: arxiv.org/abs/2306.05685 — verbatim: *"strong LLM judges like GPT-4 can match both controlled and crowdsourced human preferences well, achieving over 80% agreement"* and *"position, verbosity, and self-enhancement biases."* [^mtbench-zheng] **VERIFIED** Stream B primary.
- **Three named biases** (definitions in paper body, not abstract — labels-only here):
  - **Position bias:** judge preference for first/last response in pairwise comparison
  - **Verbosity bias:** judge preference for longer responses regardless of quality
  - **Self-enhancement bias:** judge preference for output produced by its own model family
- For code: hierarchical rubrics decompose to Correctness / Efficiency / Maintainability / Security / Readability (per Stream A digest of Galileo + MTalk-Bench follow-ons).

### §3.3 — Chatbot Arena (Zheng et al., 2024)

**Chatbot Arena** — pairwise comparison via crowdsourcing; humans more reliable at relative than absolute judgment.
- Source: arxiv.org/abs/2403.04132 [^chatbot-arena-arxiv] **VERIFIED** Stream A primary.
- **Architectural caveat:** pairwise doesn't directly yield pass/fail thresholds for production gates. Use pairwise for model selection/comparison; absolute (rubric) for production gates.

### §3.4 — Position bias (Wang et al., ACL 2024)

**Position bias methodology** — *severe enough to flip ranking outcomes*.
- Source: arxiv.org/abs/2305.17926 ("Large Language Models are not Fair Evaluators") — verbatim: *"Vicuna-13B could beat ChatGPT on 66 over 80 tested queries with ChatGPT as an evaluator"* via reordering. [^wang-fair-evaluators] **VERIFIED** Stream B primary.
- **Three calibration strategies** (verbatim labels): Multiple Evidence Calibration, Balanced Position Calibration, Human-in-the-Loop Calibration.
- 9.8% / 14.3% improvement figures cited in search summary, **NOT verified** in abstract fetch.

**Position bias quantitative scale (Stream A digest)** — research arxiv.org/abs/2406.07791 across 15 LLM judges, 22 tasks, ~40 generators, 150K+ instances. Bias varies across judge/task; magnitude depends on **quality gap** — when outputs near-equivalent, position bias dominates. [^position-bias-large-study] **LIKELY** Stream A only.

### §3.5 — Length bias, sycophancy bias, inter-rater reliability

**Length bias** — longer outputs inflate scores even at lower quality. [^perplexity-length-bias-LIKELY] **LIKELY** Stream A.

**Sycophancy bias** — judges align with human-rater demographics, replicating those biases. [^perplexity-sycophancy-LIKELY] **LIKELY** Stream A.

**Inter-rater reliability metrics:**
- Cohen's Kappa (categorical), ICC (continuous), Fleiss Kappa (multiple raters)
- Threshold: >0.80 = acceptable agreement
- Strong LLM judges reach ~80% agreement with humans on MT-Bench [^mtbench-zheng] **VERIFIED** primary.

### §3.6 — Constitutional AI (Bai et al., Anthropic, Dec 2022)

**Constitutional AI two-phase critique loop:**
- Source: arxiv.org/abs/2212.08073 — verbatim: *"in the supervised phase we sample from an initial model, then generate self-critiques and revisions, and then finetune the original model on revised responses"*; *"we sample from the finetuned model, use a model to evaluate which of the two samples is better, and then train a preference model from this dataset of AI preferences"*; *"The only human oversight is provided through a list of rules or principles, and so we refer to the method as 'Constitutional AI.'"* [^constitutional-ai-arxiv] **VERIFIED** Stream B primary.
- **Methodology:** RLAIF (Reinforcement Learning from AI Feedback) replaces RLHF; Anthropic Claude Opus 4.6 evals show 99%+ harmless-response rates with safety on per anthropic.com/transparency [^anthropic-transparency]
- **Implication for Mission 2:** explicit principle-based rubrics (the "constitution") > vague quality metrics. Mission 2's `~/.claude/CLAUDE.md` Constitution (5 principles) is structurally analogous to a CAI constitution but is NOT operationalized as a critique/revision rubric the orchestrator uses to grade dispatched output.

### §3.7 — Pairwise vs absolute scoring tradeoffs

**Pairwise (Arena-style):** humans more reliable at relative than absolute; supports model selection/comparison; doesn't directly yield production-gate thresholds.
**Absolute (G-Eval / MT-Bench-rubric-style):** supports production gates; vulnerable to all biases (position/length/sycophancy/self-enhancement).
**Effective practice (per Stream A synthesis):** combines both layered + tracks confidence intervals. [^pairwise-vs-absolute-LIKELY] **LIKELY** Stream A.

### §3.8 — Self-consistency + self-critique discipline

**Curriculum-RLAIF** — curriculum-structured preference pairs improve reward-model generalization without inference cost overhead. arxiv.org/abs/2505.20075 [^curriculum-rlaif-LIKELY] **LIKELY** Stream A.

**Constitutional AI critique loop** — see §3.6.

**Debate-judge frameworks** — see §2.1 multi-agent debate; failure mode flagged in 2025 study (§2.1).

### §3.9 — §3 summary: rubric design canon

The field has a mature canon around rubric design **for natural-language output (summarization, dialogue, response quality)** with rigorous bias diagnostics (position / length / sycophancy / self-enhancement) and inter-rater reliability frameworks (Cohen / ICC / Fleiss with >0.80 threshold). G-Eval, MT-Bench, and Constitutional AI are the three load-bearing canonical primaries.

**Critical absence — the rubric canon is NOT pre-applied to coding-agent-output specifically.** Most rubric-design literature evaluates summarization or dialogue. Code-specific output (does the patch fix the bug? does it introduce a new bug? does it match the intended approach?) requires a *different* set of criteria than dialogue scoring — and those criteria are typically baked into single-agent benchmarks (§1) rather than published as a rubric standalone. **This is a Mission 2 design opportunity:** the field hasn't yet published a rubric-design canon for coding-agent output that operates *outside* a fixed benchmark.

---

## §4 — Mission 2 Implication Mapping (LOAD-BEARING)

This section maps Mission 2's existing rubric stack onto §1-3 taxonomy and identifies the gap. **Per discipline contract: §4 IDENTIFIES the gap and RECOMMENDS scope; AO-274's response (AC amendment, deferral, or "no change needed") is filed AFTER this research lands, in the AO-274 ratify cycle.**

### §4.1 — Mission 2's existing rubric stack (verbatim from local context)

The studio knowledge base contains **a coherent process-discipline rubric stack** spanning five layers (verbatim per Stream C):

1. **Charter ratchet (compounding-loop level)** — `path-to-production.md` §4 C1-C4: failure enumeration → lesson capture → inheritance → TVD-still-holds.
2. **Per-Stage definition rubric** — `path-to-production.md` §3 TESTED + VERIFIED + DOCUMENTED with concrete artifact criteria for Stages 1-5.
3. **Per-build live-probe gate rubric** — AO-274's 14 ship gates as concrete primitive characterizations (12 RESOLVED-POSITIVE per S-72 close).
4. **Per-finding triage rubric** — `peer-review-scope.md` Cat-1/2/3.
5. **Per-claim verification rubric** — `verification.md`'s positive list × prohibited-shortcut list.

Plus **Stage falsification rubric** per `path-to-production.md` §5 (each Stage has 1-4 invariants whose violation triggers revert).

Strategically grounded in `decisions/005` (compounding-loop charter, leverage-nondeterminism sensibility) and `decisions/006` (verified design = characterized primitives × composition QA × edge-case observation, with §S-68 step 0 prepending live-probe spec grounding).

### §4.2 — Mapping onto §1-3 taxonomy

| Mission 2 rubric | §1-3 taxonomy mapping | Verdict |
|---|---|---|
| `peer-review-scope.md` Cat-1/2/3 | §3 rubric design canon | **Triage rubric, NOT quality scoring rubric.** Binary in-scope/out-of-scope. No bias mitigation discipline (position/length/self-enhancement biases not addressed). No multi-judge calibration. No inter-rater reliability framing. **Gap relative to MT-Bench / G-Eval canon: substantial.** |
| `path-to-production.md` §3 Stage TVD | §1 single-agent evals | **Process gate, NOT per-dispatch eval.** TVD validates "process artifacts exist" (events.jsonl trace, runtime evidence, doc updates). Does NOT measure per-dispatch output quality (was the patch good?). Closest single-agent-eval analogue: SWE-bench Verified test-pass scoring — but TVD only checks PR was merged + codex-review passed, NOT held-out test correctness. |
| `path-to-production.md` §4 C1-C4 compounding-loop check | §2 orchestration-layer evals | **Field-novel pattern.** "K failures + K lessons + ≥1 inheritance dispatch + TVD held under inheritance" — no published methodology surfaces this exact shape. The closest published analogue is METR's time-horizon methodology + Anthropic's RLAIF inheritance loop, but neither operationalizes recursive compounding as the substrate's primary advancement criterion. **This is genuinely novel; the field gap is real.** |
| AO-274 v0.5 14 ship gates | §1 + §2 | **Dispatch-mechanism gates, NOT output-quality gates.** Gates characterize primitives (does spawn work? does hook fire?) but do NOT measure orchestrator decision quality (route correctness, queue fairness, capability-budget enforcement rate, completion-detection precision/recall). |
| `verification.md` runtime evidence rules | §3 (criterion-presence/absence rubric) | **Binary precondition rubric.** Validates "runtime evidence shown" (positive list × prohibited-shortcut list). Does NOT score solution quality. A trivial echo-the-AC verifier passes the rule. |
| Stage 1 falsification triggers (§5) | §1 + §2 (negative-finding rubric) | **Leak-detector rubric, NOT quality-detector rubric.** Stage 1 falsifies on corruption / hang / API-bill. Subtle quality regressions (degraded fix quality, inflated scope, slower dispatch) don't trip any trigger. |

### §4.3 — Mission 2 implication candidates (10 from Stream C)

Per Stream C analysis, the following 10 implications are candidates for Mission 2 rubric-layer additions. **None are decided here** — these are the surface area an AC vii (output-quality rubric contract) on AO-274 would address.

1. **Per-dispatch output-quality score, separate from gate-pass binary.** Stage 1 "VERIFIED =" is a 5-artifact checklist, not a graded rubric. Builder quality remains uncharacterized. *(Addressable by §1 SWE-bench-style held-out testing for quality signal beyond AC-superficial-pass.)*
2. **Calibrated codex-review as load-bearing gate.** No inter-rater calibration data (codex vs human, codex vs second-LLM-judge). G-Eval discloses *self-enhancement bias* explicitly — codex (likely a same-family model judging same-family output) inherits this risk uncharacterized. *(Addressable by §3 LLM-as-judge canon + multi-judge agreement frames.)*
3. **Compounding-loop §4 C3 inheritance is unmetered.** "1 dispatch reflects new behavior" is binary; no measure of *how strongly* the lesson was inherited. *(Addressable by §2 orchestration-layer evals — repeat-task variance under inheritance vs without.)*
4. **AO-274's 14 ship gates lack output-quality companions.** Primitive-correctness gates ≠ orchestrator-decision-quality gates. *(Addressable by §2 — route-correctness / queue-fairness / capability-budget enforcement-rate.)*
5. **Stage falsification triggers are leak-detectors, not quality-detectors.** Subtle quality regressions don't trip any trigger. *(Addressable by §1 + §3 — quality-rubric falsification triggers complementing leak-detection.)*
6. **`verification.md` positive list is binary.** Runtime-evidence-shown is precondition, not quality measure. Trivial echo-the-AC verifier passes. *(Addressable by §3 rubric-design canon — criterion-independence + minimum-viable-evidence elicitation.)*
7. **Cat-1/2/3 triage has no severity dimension.** A Cat-1 typo and a Cat-1 silent data-corruption are the same triage label. *(Addressable by §3 — multi-dimensional severity scaling.)*
8. **No cross-dispatch quality drift detection.** studio-radar's drift mandate is dependency-versions, not output-quality drift across fleet. *(Addressable by §2 — temporal quality-drift monitoring.)*
9. **`decisions/006` claim-tier audit (✅E / ↩D / ⊕X) lacks falsification rubric per tier.** Open Q in §Maintenance protocol explicitly defers this. *(Addressable by §3 — per-claim-tier confidence intervals + revision-likelihood frames.)*
10. **"Leverage nondeterminism, do not tame it" sensibility (`decisions/005`) lacks a metric.** Declared posture (variation is productive), no instrument to confirm we're getting productive variation vs noise. *(Addressable by §1 + §2 — variance-of-quality metrics that distinguish productive variation from quality-jitter.)*

### §4.4 — Recommendation (NOT decision — for AO-274 ratify cycle)

**RECOMMEND** that AO-274 v2.0 architecture proposal add an **AC vii — Output-quality rubric contract** before Stage 1 dispatch claims "effective orchestration." Concrete shape (one paragraph proposal, AO-274 ratify cycle decides):

> **(vii) Output-quality rubric contract.** Beyond AC (vi)'s Stage 1 TVD process ratchet, the v2 dispatch primitive MUST emit per-dispatch output-quality signals into events.jsonl that feed an orchestrator-layer rubric distinct from process gates. Minimum content: (a) AC-coverage score (did the dispatch's PR address each AC item; %-coverage measurable via held-out test pass-rate or LLM-judge-with-bias-mitigation per §3); (b) edit-format reliability (per Aider precedent — orthogonal-axis correctness vs format); (c) per-dispatch self-enhancement-bias diagnostic when codex-review judges output (per §3.1 G-Eval self-disclosed limitation); (d) C3 inheritance-strength signal (did the next dispatch produce evidence of applying the lesson, not just running with the rule loaded). Stage 1 falsification triggers (§5) extend with one rubric-driven trigger: rubric inter-rater agreement <0.80 on Cohen's Kappa over 1 dispatch batch falsifies the rubric (not the orchestrator) — distinguishing rubric-instability from orchestrator-failure.

**Why this recommendation:**
- §2's gap finding is empirical (no published per-orchestrator output-quality benchmark) — Mission 2 would be FIRST PUBLIC PRECEDENT if it ships AC vii. That is consistent with `decisions/005` Layer 4 strategic posture (chief-of-staff at architectural scale) and `decisions/006` verified-design discipline applied to itself.
- §3's bias canon is mature (G-Eval, MT-Bench, Wang) — there is no excuse for shipping an LLM-as-judge gate (codex review) without explicit bias diagnostics.
- §4.3 implications 1-4 address the most operationally consequential gaps (per-dispatch quality / calibrated judge / inheritance metering / orchestrator-decision quality) and naturally compose into one AC.

**What this recommendation does NOT include (deferred to follow-up):**
- Implementation of an eval harness in `substrate/` (v2.1+ per AO-290 out-of-scope).
- Empirical comparison of Mission 2 orchestrator vs Devin/Cursor/Codex Cloud (Stage 2+ benchmarking once rubric exists).
- Multi-dimensional severity scaling for Cat-1/2/3 triage (implication 7) — separate `peer-review-scope.md` revision.
- Cross-dispatch quality drift detection (implication 8) — studio-radar follow-up.

---

## §Conflicts & Gaps

### Conflicts (sources disagree — present both)

1. **SWE-bench Verified saturation magnitude.** Stream A reports via latent.space "60% of remaining unsolved are functionally unsolvable"; primary source (openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/) returned HTTP 403 in Stream B. **Verdict:** ceiling-clustering at ~80% with 1-3pp separation is VERIFIED via multiple aggregators; "60% unsolvable" is LIKELY pending direct primary fetch.

2. **Cognition Devin reception.** Vendor self-report (13.86% on 25% subset) is VERIFIED via cognition.ai blog. Independent demo critique (Pragmatic Engineer Apr 2024 — "video contains straight up lies") is VERIFIED via primary fetch. Both stand. **Verdict:** vendor methodology is honestly self-disclosed (with contamination caveats); demo claims are walked back; no independent SWE-bench reproduction publicly available.

3. **Manus AI GAIA scores.** Vendor claims surface in secondary aggregators (helicone); MIT Tech Review primary observed instability without citing GAIA scores. **Verdict:** vendor scores remain self-reported; no independent verification.

### Gaps (sections no stream could fully answer)

1. **OpenAI Codex Cloud methodology.** Thin coverage in both Stream A (Perplexity) and Stream B (WebFetch). Vendor primary not deep-fetched.
2. **Manus AI primary methodology.** No vendor blog with methodology details surfaced.
3. **RAND on agent capability.** Not surfaced in either stream.
4. **AgentBench primary verification.** Stream A surfaced via GitHub URL only; Stream B did not deep-fetch THUDM/AgentBench paper.
5. **TaskBench primary verification.** Stream A surfaced via arxiv URL; Stream B did not fetch.
6. **BigCodeBench contamination methodology** (10-gram / 13-gram on ODEX intents + Stack Overflow). Paper body needed.
7. **MT-Bench bias definitions verbatim** (position, verbosity, self-enhancement). Paper body needed; abstract had labels only.
8. **Wang et al. position-bias quantitative scale.** 9.8% / 14.3% improvement figures in search summary, NOT in abstract fetch.

### Excluded sources (confabulated arxiv IDs)

- arxiv.org/abs/2604.03515 (cited Stream A as ref [^36]) — future-dated 2604 = April 2026; not surfaced by Stream B; **EXCLUDED** as likely confabulation.
- arxiv.org/abs/2604.26102 (cited Stream A as ref [^40]) — same; **EXCLUDED**.
- arxiv.org/abs/2602.11354 (cited Stream A as ref [^45]) — February 2026 plausible given today=2026-05-07 but not surfaced by Stream B; **EXCLUDED** pending verification.

---

## §Related but Out of Scope

These findings emerged during research but fall outside the 4-section scope. Captured for follow-up if Mission 2 work surfaces consumer pressure:

- **Statsig multi-step framework** (statsig.com/perspectives/evaluatingmultistepreasoningagenteval) — collaboration metrics (clarity, sync), resource metrics (memory/CPU/latency), quality metrics. Closer to product-eval than orchestration-eval; out of scope here.
- **Microsoft Azure orchestration patterns** (sequential / concurrent / handoff / manager) — pattern taxonomy, not eval methodology.
- **LangChain trajectory evals** (docs.langchain.com/langsmith/trajectory-evals) — reference-trajectory matching OR LLM-as-judge trajectory scoring. Vendor-specific; principles overlap §3 but mechanism is LangSmith-specific.
- **Cursor edit-format precision blog** — covered in §1.5 Aider only via methodology overlap; cursor.com primary not deep-read.
- **Anthropic "demystifying evals for AI agents"** (Stream A referenced but URL not surfaced) — vendor blog likely valuable; follow-up fetch.

---

## §Key Sources (47 verified primary sources, all access date 2026-05-07)

### Primary papers (arxiv)

[^swebench-arxiv]: arxiv.org/abs/2310.06770 — Jimenez et al., "SWE-bench: Can Language Models Resolve Real-World GitHub Issues?", ICLR 2024. Verbatim 2,294 problems + 1.96% Claude 2.
[^bigcodebench-arxiv]: arxiv.org/abs/2406.15877 — Zhuo et al., BigCodeBench, ICLR 2025.
[^mle-bench-arxiv]: arxiv.org/abs/2410.07095 — Chan et al. (OpenAI), MLE-bench, Oct 2024. Verbatim 75 competitions + 16.9% bronze.
[^livecodebench-arxiv]: arxiv.org/abs/2403.07974 — Jain et al., LiveCodeBench, ICLR 2025.
[^metr-arxiv]: arxiv.org/abs/2503.14499 — Kwa, West, Becker et al. (METR), "Measuring AI Ability to Complete Long Tasks", Mar 2025. Verbatim doubling every 7 months + Claude 3.7 ~50min.
[^multiagent-debate-du]: arxiv.org/abs/2305.14325 — Du et al., Multi-agent debate, ICML 2024.
[^openhands-arxiv]: arxiv.org/abs/2407.16741 — Wang et al., OpenHands.
[^g-eval-arxiv]: arxiv.org/abs/2303.16634 — Liu et al., G-Eval, EMNLP 2023. Verbatim Spearman 0.514 + LLM-bias self-disclosure.
[^mtbench-zheng]: arxiv.org/abs/2306.05685 — Zheng et al., LLM-as-a-Judge / MT-Bench, NeurIPS 2023. Verbatim >80% agreement + 3 biases named.
[^chatbot-arena-arxiv]: arxiv.org/abs/2403.04132 — Chatbot Arena methodology.
[^constitutional-ai-arxiv]: arxiv.org/abs/2212.08073 — Bai et al. (Anthropic), Constitutional AI, Dec 2022. Verbatim two-phase critique loop.
[^wang-fair-evaluators]: arxiv.org/abs/2305.17926 — Wang et al., "Large Language Models are not Fair Evaluators", ACL 2024. Verbatim Vicuna 66/80 + 3 calibration strategies.
[^position-bias-large-study]: arxiv.org/abs/2406.07791 — large-scale position-bias study across 15 LLM judges, 22 tasks, 150K+ instances. **LIKELY** Stream A only.
[^curriculum-rlaif-LIKELY]: arxiv.org/abs/2505.20075 — Curriculum-RLAIF. **LIKELY** Stream A only.
[^multiagent-debate-failure-LIKELY]: arxiv.org/abs/2509.05396 — Multi-agent debate failure mode. **LIKELY** Stream A only.
[^ultrahorizon-LIKELY]: openreview.net/forum?id=FTZfVHWAIq — UltraHorizon. **LIKELY** Stream A only.
[^taskbench-arxiv]: arxiv.org/abs/2311.18760 — TaskBench. **LIKELY** Stream A only.

### Vendor + leaderboard primaries

[^swebench-verified-page]: swebench.com/verified.html — verbatim 500-instance + temperature + harness methodology.
[^anthropic-swebench-sonnet]: anthropic.com/research/swe-bench-sonnet — Anthropic SWE-bench Verified write-up.
[^anthropic-infra-noise]: anthropic.com/engineering/infrastructure-noise — Sep 2025.
[^therift-infra-noise]: therift.ai/news-feed/anthropic-publishes-research-on-infrastructure-noise-in-agentic-coding-benchmarks — secondary press.
[^cognition-devin-blog]: cognition.ai/blog/swe-bench-technical-report — verbatim 13.86% + caveats.
[^cognition-devin-results-repo]: github.com/CognitionAI/devin-swebench-results — Cognition's harness.
[^pragmatic-engineer-devin]: newsletter.pragmaticengineer.com/p/the-pulse-90 — Apr 18 2024 Devin walkback.
[^mit-manus-review]: technologyreview.com/2025/03/11/1113133/manus-ai-review/ — Mar 11 2025.
[^aider-leaderboard]: aider.chat/docs/leaderboards — current top scores + methodology + edit-format-correct metric.
[^cursor-blog]: cursor.com/blog/agent-best-practices — **LIKELY** Stream A only.
[^helicone-aggregator-LIKELY]: helicone.ai/blog/manus-benchmark-operator-comparison — aggregator.
[^livecodebench-site]: livecodebench.github.io — verbatim time-cutoff methodology.
[^metr-blog]: metr.org/blog/2025-03-19-measuring-ai-ability-to-complete-long-tasks/.
[^bigcodebench-hf-blog]: huggingface.co/blog/leaderboard-bigcodebench — June 18, 2024; verbatim 1140-task + 97% human ceiling.
[^anthropic-transparency]: anthropic.com/transparency.
[^anthropic-constitution]: anthropic.com/constitution.

### Repos + secondary

[^swebench-main]: swebench.com — main leaderboard (table truncated on fetch).
[^swebench-harness]: swebench.com/SWE-bench/reference/harness/ — Docker setup methodology.
[^mle-bench-repo]: github.com/openai/mle-bench.
[^agentbench-repo]: github.com/THUDM/AgentBench. **LIKELY** Stream A only.
[^codeeval-pro]: github.com/CodeEval-Pro/CodeEval-Pro. **LIKELY** Stream A only.
[^webarena-repo]: github.com/web-arena-x/webarena. **LIKELY** Stream A only.
[^ibm-humaneval-overview]: ibm.com/think/topics/humaneval — secondary overview of HumanEval family.
[^theagentcompany-LIKELY]: emergentmind.com/topics/theagentcompany-benchmark — **LIKELY** Stream A only.
[^galileo-frameworks]: galileo.ai/blog/best-agent-evaluation-frameworks.
[^galileo-metrics]: galileo.ai/blog/agent-evaluation-framework-metrics-rubrics-benchmarks.
[^latent-space-end-of-swebench]: latent.space/p/swe-bench-dead — secondary on SWE-bench Verified saturation. **CONFLICTING** — primary unfetchable.
[^aggregator-leaderboards-UNVERIFIED]: llm-stats.com / awesomeagents.ai — leaderboard aggregators. **UNVERIFIED**.
[^perplexity-stream-A-§1]: Perplexity sonar-deep-research synthesis citing 50 sources (raw response saved /tmp/stream-a-raw-AO-290.json). **Synthesis-grade** — not a primary source itself.
[^perplexity-stream-A-§4]: Perplexity sonar-deep-research §4 synthesis. **Synthesis-grade**.
[^perplexity-length-bias-LIKELY]: Perplexity §3 synthesis, length bias.
[^perplexity-sycophancy-LIKELY]: Perplexity §3 synthesis, sycophancy bias.
[^pairwise-vs-absolute-LIKELY]: Perplexity §3 synthesis, pairwise vs absolute frame.

### Failed fetches (HTTP 403 — secondary-sourced only)

- openai.com/index/introducing-swe-bench-verified/ — OpenAI announcing SWE-bench Verified.
- openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/ — OpenAI saturation post (load-bearing for §1.1 CONFLICTING claim).

---

---

## §10 — Post-research finding: Anthropic shipped the pattern (S-78 evening amendment)

**Status:** v0.1 amendment. Body of doc (§1-§9) preserved verbatim; this section appends the finding that surfaced after v0 close + reframes §4's implications under the new lens. Anthropic shipped Managed Agents in **April 2026** with `define_outcome`, `multi-agent`, `dream` primitives in research preview. The finding lands hours after v0 publication and is significant enough to ratchet the §4 mapping.

### §10.1 — What Anthropic shipped (canonical primitive set)

Per [platform.claude.com/docs/en/managed-agents/overview](https://platform.claude.com/docs/en/managed-agents/overview) (access 2026-05-07), Managed Agents introduces 4 canonical concepts plus 3 research-preview primitives:

| Primitive | Type | Beta header | Source |
|---|---|---|---|
| **Agent** | Versioned (model + system + tools + MCP + skills) | `managed-agents-2026-04-01` | `/agent-setup` |
| **Environment** | Container template (packages + networking) | `managed-agents-2026-04-01` | `/environments` |
| **Session** | Running agent instance (idle/running/rescheduling/terminated) | `managed-agents-2026-04-01` | `/sessions` |
| **Events** | Bidirectional `{domain}.{action}` event stream | `managed-agents-2026-04-01` | `/events-and-streaming` |
| **Memory Store** | Workspace-scoped, 100KB cap, 8 stores/session, 30-day retention | `managed-agents-2026-04-01` (public beta) | `/memory` |
| **Outcome** | Rubric + grader + iteration loop | Research preview | `/define-outcomes` |
| **Multi-agent** | Coordinator-Worker, depth=1, 25 threads max, 20 unique agents | Research preview | `/multi-agent` |
| **Dream** | Async memory-store curation job | `dreaming-2026-04-21` (research preview) | `/dreams` |

**Verbatim from `/define-outcomes` (load-bearing for AO-274 AC vii):**
> "When you define an outcome, the harness automatically provisions a *grader* to evaluate the artifact against a rubric. It leverages a separate context window to avoid being influenced by the main agent's implementation choices. The grader returns a per-criterion breakdown: either confirmation that the artifact satisfies the rubric, or the specific gaps between the current work and the requirements."

This is **separate-context LLM-as-judge as falsification-bias mitigation, structurally encoded** — the operational answer to the G-Eval self-enhancement-bias problem flagged in §3.1 of this landscape.

**Result enum** (verbatim): `satisfied | needs_revision | max_iterations_reached | failed | interrupted`. The `failed` case fires *"when the rubric fundamentally does not match the task, for example if the description and rubric contradict each other"* — meta-rubric falsification, distinct from agent-ran-out-of-iterations.

**Rubric guidance (verbatim):** *"Structure the rubric as explicit, gradeable criteria, such as 'The CSV contains a price column with numeric values' rather than 'The data looks good.'"* Direct alignment with G-Eval's discipline (§3.1).

**Performance claim** (verbatim, [siliconangle 2026-05-06](https://siliconangle.com/2026/05/06/anthropic-letting-claude-agents-dream-dont-sleep-job/)): *"outcomes improved task success by up to 10 percentage points over a standard prompting loop"*; *"increasing task success by 8.4% for docx files and 10.1% for pptx files in internal benchmarks."*

**Pricing disclosure** (load-bearing — under-quoted in primary docs; flagged by [Help Net Security 2026-04-09](https://www.helpnetsecurity.com/2026/04/09/claude-managed-agents-bring-execution-and-control-to-ai-agent-workflows/) + [Medium / sathishkraju 2026-04-17](https://medium.com/@sathishkraju/anthropics-managed-agents-i-read-the-fine-print-so-you-don-t-have-to-ed17b77e17c5)): **$0.08 per session-hour while running**, on top of standard token rates. 24-agent fleet × 8h/day = $15.36/day session overhead **before** inference token costs.

**Customer evidence:**
- **Harvey (legal):** completion rates rose ~6× in internal tests using dreams to remember filetype workarounds + tool-specific patterns. ([cryptobriefing](https://www.cryptobriefing.com/anthropic-claude-agents-dreaming/))
- **Netflix:** multi-agent orchestration analyzing build-log patterns across hundreds of builds. ([cryptobriefing](https://www.cryptobriefing.com/anthropic-claude-agents-dreaming/))

### §10.2 — Multi-agent orchestration (Anthropic blog + Managed Agents docs)

Per [anthropic.com/engineering/multi-agent-research-system](https://www.anthropic.com/engineering/multi-agent-research-system) (access 2026-05-07):

**Quantified multi-agent gain:** **90.2% improvement** of multi-agent (Opus 4 lead + Sonnet 4 subagents) vs single-agent Opus 4 on internal research evaluation. **Token cost: ~15× chat baseline** (vs ~4× for single agents) — economically viable only on high-value tasks.

**Anti-pattern explicitly named for coding** (verbatim, load-bearing): *"most coding tasks involve fewer truly parallelizable tasks than research"* and *"systems requiring shared context across all agents perform poorly."* This is **direct empirical pressure** against naive coordinator-worker patterns for code work — relevant to AO-274 / AO-166 design.

**Multi-agent constraints (Managed Agents docs):**
- *"The coordinator can only delegate to one level of agents; depth > 1 is ignored."* — flat single-level delegation, not arbitrary graph.
- Max 25 concurrent threads, max 20 unique agents in roster.
- Threads are persistent; coordinator can send follow-ups; agent retains history.

### §10.3 — "Demystifying Evals for AI Agents" (Anthropic, 2026-01-09)

Per [anthropic.com/engineering/demystifying-evals-for-ai-agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (access 2026-05-07):

**8-step methodology (verbatim recommended):**
1. Start early — 20-50 tasks from real failures, not hundreds.
2. Start with what you already test manually.
3. Write unambiguous tasks with reference solutions; domain experts reach identical verdicts.
4. Build balanced problem sets (should-fire and shouldn't-fire).
5. Robust eval harness with stable environment isolating each trial.
6. Design graders thoughtfully — deterministic where possible.
7. Check transcripts regularly to verify graders work.
8. Monitor for capability eval saturation.

**Cadence (verbatim):**
- Run evals "on every commit" for speed.
- Maintain separate **capability evals** ("can we do this?") and **regression evals** (100% baseline drift catcher).
- Sample and review transcripts **weekly**.
- "Reserve systematic human studies for calibrating LLM graders."

**Anti-patterns named:**
- Rigid step-checking — *"grading what the agent produced, not the path"* — penalizes valid alternative approaches.
- Class-imbalanced evals → one-sided optimization.
- Trusting eval scores without reading transcripts.

### §10.4 — Reframe of §4 implications under "Anthropic shipped the pattern" lens

The morning's §4.4 recommendation said Mission 2 would be "FIRST PUBLIC PRECEDENT" if AC vii ships. **That framing is now wrong.** Anthropic shipped the pattern in April 2026; Mission 2 is no longer a precedent — it's an aligner.

The sharper recommendation: **substrate work today should pattern-match Anthropic's primitive shapes so future migration (full or partial) is "swap implementation, keep contract" rather than "rewrite."**

#### Migration-readiness checklist (cheap to mirror at substrate-write time)

| Anthropic primitive | Substrate alignment move | Migration cost if mirrored today |
|---|---|---|
| Event naming `{domain}.{action}` | events.jsonl emit-helper categories: `agent.tool_use`, `session.status_idle`, `span.outcome_evaluation_end` | Near-zero (rename) |
| Result enum `satisfied/needs_revision/max_iterations_reached/failed/interrupted` | Adopt VERBATIM as events.jsonl `outcome` field discriminator | Near-zero |
| Rubric markdown criterion-by-criterion | Stage 1 dispatch primitive consumes rubrics in this exact format | Near-zero |
| Session lifecycle `idle → running → rescheduling → terminated` | events.jsonl 7-stage schema adopts state names | Low |
| Vault scoped to session w/ managed OAuth refresh | Prepare for `~/.claude/.env` → session-vault migration | Medium |
| Memory store: per-store 100KB, 8/session, 30d retention, version chain | Memory file system already shaped close; chunking strategy needed for large topics | Medium |

#### 3 hard mismatches (architectural choices Mission 2 must navigate)

1. **Coordinator depth = 1.** Anthropic caps multi-agent at flat single-level delegation. Mission 2's intended L1 → L2 → L3 hierarchy is depth ≥ 2. Workaround: nested-but-flat coordinators with explicit cross-workspace handoff, NOT a single deep coordinator. Affects AO-166 + AO-123 design directly.
2. **Two-substrate architecture.** Mission 2's `koinos-studio` (knowledge) + `studio-koinos` (capability) split is broader than Anthropic's single-workspace memory boundary. Cross-substrate flow CANNOT collapse into one Managed Agents memory_store. Implication: koinos-context stays canonical for cross-substrate / temporal-graph; Managed Agents memory_store is leaf-only.
3. **Session-hour billing.** $0.08/session-hour is hostile to always-on watchers (studio-radar, focus-reviewer cron). Substrate must stay event-driven short sessions (which actually aligns with Anthropic's session model — idle is default, events drive work).

#### Independent reception (calibration)

Per [Medium / unicodeveloper](https://medium.com/@unicodeveloper/claude-managed-agents-what-it-actually-offers-the-honest-pros-and-cons-and-how-to-run-agents-52369e5cff14) + [sathishkraju](https://medium.com/@sathishkraju/anthropics-managed-agents-i-read-the-fine-print-so-you-don-t-have-to-ed17b77e17c5):

- **Lock-in concerns explicit:** "Lock-in is real" — Claude-only, no model-swap, migration "non-trivial".
- **Outcomes + multiagent + dreaming all gated** behind access-request forms (research preview) as of 2026-05-06.
- **GA timeline undisclosed.** Public beta since 2026-04-09; research preview features still gated.

**Decision framing (independent consensus):**
- Use Managed Agents if: ship-speed > infra ownership, Claude-committed, regulated-industry governance need.
- Build your own if: multi-model flexibility, data privacy, high-volume cost sensitivity, full control.

### §10.5 — 2026 Agentic Coding Trends Report (Anthropic, April 2026)

Per [resources.anthropic.com/hubfs/2026 Agentic Coding Trends Report.pdf](https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf) (PDF binary; reconstructed via [hivetrail summary](https://hivetrail.com/blog/anthropic-2026-agentic-coding-report/) + [pathmode](https://pathmode.io/blog/orchestration-era-needs-intent) + [LinkedIn AnthropicResearch](https://www.linkedin.com/posts/anthropicresearch_eight-trends-defining-how-software-gets-built-activity-7424525205449940993--Iv0)):

**LIMITATION FLAG:** PDF could not be read directly via WebFetch. Findings below are second-hand from search-aggregator summaries. **Re-verify against PDF before treating as load-bearing.**

**Headline findings:**
- Engineers report using AI in **~60% of work**.
- "Fully delegated" capacity: **0–20% ceiling on naive delegation** — explicit "delegation gap" finding.
- ~**27% of AI-assisted work** = tasks that wouldn't have been done otherwise (productivity expansion, not substitution).
- One company: **89% AI adoption** org-wide with hundreds of internal agents.

**Mission 2 anchor:** the "delegation gap" (0-20% naive delegation ceiling) is the empirical pressure for Mission 2's value claim. The substrate's purpose, framed against this report, is **breaking the delegation ceiling** through composed primitives (outcomes + memory + dreams + multi-agent). Per Constitution Principle 3 ("AI recommends, Ali decides"), the substrate is designed to maximize the delegated fraction without crossing into autonomous decision-making.

### §10.6 — Updated Mission 2 implication candidates

Replaces §4.3's 10 candidates (which assumed field-canon abstraction) with 7 grounded against Anthropic's shipped primitive set:

1. **Outcomes loop = canonical falsification primitive for Mission 2.** Every `.context/<ISSUE>/test-plan.md` is structurally a draft rubric — sharpening to gradeable criteria converts agent-ops's existing artifact into Anthropic's canonical primitive.
2. **Multi-agent flat delegation + 25-thread cap = empirical ceiling for naive coordinator-worker.** Layer 2 (AO-166) cannot trivially scale via single-coordinator session. Either Layer 2 stays outside Managed Agents (cmux + workspaces), OR adopts coordinator-of-coordinators externally orchestrated.
3. **Memory store + dream = workspace-scoped curation primitive.** Cannot collapse two-substrate architecture into one Managed Agents memory_store. koinos-context remains canonical for cross-substrate.
4. **Pricing changes the build/buy frame.** Always-on watchers OUT of Managed Agents; bursty short-session dispatch IN. Aligns with Anthropic's session model (idle default, events drive work).
5. **Vault + MCP-via-vault = substrate authentication primitive.** Empirical pressure to align secret-handling with vault-style indirection at session boundary.
6. **Beta-gating creates a sequencing constraint.** Outcomes + multiagent + dreaming require access requests. **Action item NOW:** request research-preview access at [claude.com/form/claude-managed-agents](https://claude.com/form/claude-managed-agents) so the gate doesn't block design decisions.
7. **2026 Trends Report's "delegation gap" (0-20% naive ceiling) is the empirical anchor for Mission 2's value proposition.** Substrate success = breaking the delegation ceiling through composed primitives.

### §10.7 — Sharpened AC vii proposal text (replaces §4.4 morning version)

> **(vii) Output-quality outcome contract — Anthropic Managed Agents-aligned.** *(Added S-78 evening per AO-290 §10 post-research finding.)* Beyond AC (vi)'s Stage 1 TVD process ratchet, the v2 dispatch primitive MUST treat each Linear-issue dispatch as a `define_outcome`-shaped operation: rubric authored at constitution+template+issue level (markdown criterion-by-criterion); grader runs in separate-context Claude Code subagent (not same context as builder); per-criterion breakdown returned to builder for next iteration; max_iterations bounded (default 3, cap 20 per Anthropic precedent); result ∈ {`satisfied`, `needs_revision`, `max_iterations_reached`, `failed`, `interrupted`} written to events.jsonl as `outcome` discriminator. The proposal MUST emit `span.outcome_evaluation_*` analogue events (start/ongoing/end with explanation field) so Mission 2 events.jsonl is migration-ready to Anthropic's stream shape. Stage 1 falsification triggers (§5) extend with one rubric-driven trigger: rubric inter-rater agreement <0.80 Cohen's Kappa over 1 dispatch batch falsifies the rubric (not the orchestrator) AND `result == "failed"` with rubric-task contradiction surfaces as a meta-rubric falsification distinct from agent-failure.

### §10.8 — Cross-references

- **Linear AO-274** (orchestrator v2 architecture design) — paused pending S-78 research; AC vii §10.7 supplies the proposal text for ratify cycle
- **Linear AO-NEW** (Mission 2 ↔ Managed Agents alignment evaluation) — capture-pattern issue filed S-78 to scope future-phase migration consideration
- **Memory:** `project_managed_agents_alignment_posture_s78.md` codifies alignment-vs-adoption posture
- **Anthropic primary sources:** all `/managed-agents/*` docs at platform.claude.com (access 2026-05-07)
- **Anthropic engineering blogs:** demystifying-evals-for-ai-agents (2026-01-09); multi-agent-research-system

---

*End of v0.1 amendment. Body of doc (§1-§9) preserved verbatim from S-78 morning v0. §10 amendment added S-78 evening per Ali's referenced Anthropic Managed Agents finding + future-phase migration consideration. Re-verify per surface after 30 days. Linear: AO-290 (Done v0); AC vii target = AO-274 ratify; alignment-eval target = AO-NEW.*

---

*End of document. Research date: 2026-05-07. v0 synthesis from 3 parallel research streams (Perplexity sonar-deep-research, WebFetch+WebSearch, Local context). v0.1 amendment from Anthropic Managed Agents deep dive (parallel WebFetch + WebSearch dispatch). Re-verify per surface after 30 days. Linear: AO-290.*
