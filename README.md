# Agent to Solve the Model Selection Problem Nobody Talks About

*Every team picks the wrong LLM. Here's why — and a framework for fixing it.*

---

Nobody chose the wrong LLM on purpose. They just never defined what "right" meant.

And here is how the possible scenario looks like:

A team is six months into AI product development. Their AI feature is working — kind of.

The model they picked is too slow for the latency requirements.

Or the cost is 4× what they projected.

Or they tried to swap to a better model and discovered their eval stack was so tightly coupled to the original model's output format that migration would delay everything.

---

## The actual problem

Model selection gets treated as a single question: *which model should we use?* It isn't. It's three separate questions that require different information, different stakeholders, and different timeframes.

**Can this model do what we need?** That's a capability question. It requires task-specific benchmark data, not overall leaderboard rank.

**Can our team actually operate this model?** That's an infrastructure question. A 2-engineer pre-PMF startup and a 20-engineer enterprise with data sovereignty requirements are making completely different bets — on cost structure, compliance, and who owns the problem when it breaks.

**What does this model look like in 12 months?** That's a trajectory question. AI shifts every 3–6 months. The model you choose today should be one you can migrate away from without rebuilding your entire eval stack.

Most teams answer the first question, handle the second ad hoc, and never ask the third. That's where the expensive mistakes come from.

I built an agent that helps select the right LLM for your product using the Three-Gate Framework, and this post is a walkthrough of how it works.

---

## What the tool actually does

You type one sentence describing what you are building. Something like: *"Two engineers, pre-revenue legal tech startup, building a contract extraction pipeline from uploaded PDFs."*

The agent infers eleven parameters from that description, runs them through the three gates, searches [artificialanalysis.ai](https://artificialanalysis.ai) for current benchmark scores and pricing, and produces a recommendation with full reasoning — primary model, runner-up, gate-by-gate analysis, a replaceability score, red flags, and a documentation checklist.

Every inferred parameter is then exposed as an adjustable control on the dashboard. You can change the task type, the budget, the team size, the time horizon — and the recommendation recalculates live.

One input. Live benchmark data. Interactive output.

---

## The Three-Gate Framework

Every model has to pass all three gates to reach the recommendation.

Failing Gate 1 means the model can't do the job.
Failing Gate 2 means the team can't operate it.
Failing Gate 3 means the team is betting on something they may not be able to exit when the landscape shifts — which it will.

---

### Gate 1 — Capability

**The question: Can this model actually do what we need?**

The most common mistake here is opening artificialanalysis.ai or a similar leaderboard, sorting by the overall score, and picking the top result. That composite score is a weighted average across dozens of benchmarks spanning reasoning, science, coding, and agentic tasks. For most specific use cases, the majority of those benchmarks are irrelevant.

The model ranked #1 on the overall intelligence index may rank #6 on the metrics that actually matter for contract extraction.

A contract extraction workflow is a retrieval and structured output problem. The relevant benchmarks are AA-LCR (long-context reasoning across documents from 10k to 100k tokens) and IFBench (whether the model reliably conforms to a specified output format).

Gate 1 has three parameters.

**Task Type** is the most consequential input in the framework because it determines which benchmarks the agent uses. Fourteen task types are supported, each mapped to specific scores from artificialanalysis.ai:

*Retrieval* uses AA-LCR and AA-Omniscience — the latter combining factual accuracy and hallucination rate into a single score, because surfacing information that isn't in the source corpus is the most common retrieval failure mode.

*Extraction* uses AA-LCR and IFBench — instruction and format compliance matters most when the extraction output must conform to a downstream schema.

*Reasoning and Q&A* use GPQA Diamond (198 graduate-level questions designed to be unsolvable by search), HLE (Humanity's Last Exam — 2,158 expert-level questions across mathematics and science, currently the hardest publicly available benchmark), and MMLU-Pro (12,000 graduate questions across 14 subject areas).

*Code* uses LiveCodeBench — a contamination-free benchmark that continuously harvests fresh problems from LeetCode, AtCoder, and Codeforces so models cannot have memorised the answers during training — alongside SciCode (288 scientist-curated subproblems from laboratory challenges) and the AA Coding Index.

*Agentic Workflow and Multi-Agent Orchestration* use GDPval-AA (220 real-world tasks across 44 occupations where models are given shell access and web browsing to complete end-to-end work, scored by ELO from blind pairwise comparisons), τ²-Bench Telecom (114 dual-control agent-user simulation tasks in a technical support context), Terminal-Bench Hard (44 terminal-based agentic tasks covering software engineering and system administration), and APEX-Agents-AA (452 professional-service tasks in realistic application environments).

*RAG Pipelines* use AA-LCR, AA-Omniscience, and GPQA Diamond — the synthesis reasoning step in RAG is where model quality matters most, and AA-Omniscience's hallucination rate catches the most common failure: generating plausible content not present in the retrieved context.

*Function Calling, Browser Use, Agent Memory and State* each map to task-specific benchmark subsets in the same way.

The point is not the specific benchmarks. The point is that the relevant benchmarks are different for every task type, and using the general intelligence index as a proxy for task-specific capability is the root cause of most bad model selection decisions.

**Context Window** is evaluated as a hard filter before any benchmarks are consulted. There are four tiers: under 8k tokens (short documents and single-turn tasks), 8k–32k (standard document processing, multi-turn conversations), 32k–128k (full contracts, extended codebases), and 128k+ (full repository ingestion, book-length documents, long-horizon agent sessions). If the model physically cannot fit your inputs, its benchmark scores are irrelevant.

**Accuracy Priority** determines how benchmark performance is weighted against cost and latency when Gate 2 data comes in. At Low, speed and cost dominate and smaller models like Haiku and Gemini Flash are viable. At Critical — medical, legal, financial, safety contexts — the agent recommends the highest task-benchmark scorer regardless of what it costs. The four levels map directly to the severity of the consequences when the model is wrong.

Gate 1 output is a shortlist of 2–3 models that clear the capability threshold, filtered by context window, ranked by the task-specific benchmarks that actually matter.

---

### Gate 2 — Infrastructure

**The question: Where does this run, and what does that decision actually cost?**

Two teams can be running the same model and making completely different bets.

A 2-engineer startup calling GPT-4o through the OpenAI API is betting on simplicity and speed.
A 20-engineer enterprise running GPT-4o through Azure OpenAI is betting on compliance, data residency, and SLA guarantees.

Gate 2 is where those differences become decisive — and where technically excellent models get eliminated because the team cannot support the operational requirements, or because the hosting path fails compliance before the cost question is even asked.

**Team Size** is the most directional parameter in this gate. The framework enforces a hard rule:

If the team has 1–2 engineers, the recommendation will always be a closed-source API. No exceptions. At that team size, the infrastructure work of self-hosting — provisioning, model updates, monitoring, incident response — consumes the team disproportionately. The cost savings from open-source do not compensate for the velocity loss.

At 3–10 engineers, closed-source remains the default unless there's demonstrated cost pressure and dedicated infrastructure capacity.

Open-source self-hosting becomes viable at 11–50 with the right team structure.

At 50+, the full range of options including fine-tuned open-source models is in scope.

**Company Stage** shapes how aggressively to optimise for speed versus cost versus compliance.

Pre-PMF means avoiding infrastructure complexity that slows deployment cycles — closed-source pay-as-you-go, no fine-tuning, no bespoke deployment pipelines.

Post-PMF Scaling is where monthly API costs are now predictable enough to model and open-source evaluation starts to pay back.

Enterprise is where compliance and data residency requirements often dominate the entire decision before the technical evaluation begins.

**Monthly Budget** is cross-referenced against live pricing data from artificialanalysis.ai across five tiers.

Under $500/month covers lightweight models at limited volume — Haiku, Gemini Flash.
$500–$1k covers mid-tier at moderate volume — GPT-4o Mini, Sonnet 3.5.
$1k–$5k covers frontier models at production volume — GPT-4o, Sonnet 3.7, Gemini 1.5 Pro.
$5k–$20k covers high-volume or reasoning-heavy workloads.
$20k+ is enterprise contract territory. The agent flags any shortlisted model where realistic usage would breach the stated budget.

**Data Sovereignty**, when required, is a hard filter that runs before cost is even considered. It eliminates any hosting path where data leaves the team's own cloud environment — which means direct API calls to OpenAI, Anthropic, Google, and others are removed from the shortlist entirely. The only viable paths are AWS Bedrock, Azure OpenAI Service, or GCP Vertex AI. This is relevant for any product handling personal data under GDPR, health data under HIPAA, financial data under SOC 2 or PCI-DSS, or government data under FedRAMP.

**Cloud Provider** determines the specific managed deployment path.

AWS maps to Bedrock — managed access to Claude, Llama, Mistral, and others within your VPC, integrating natively with IAM and CloudTrail.

Azure maps to Azure OpenAI Service — GPT-4o and other OpenAI models with enterprise SLAs, European data residency options, and Active Directory integration, which is typically the path of least resistance for Microsoft-stack enterprises.

GCP maps to Vertex AI — the Gemini family plus Claude and Llama via Model Garden.

None means direct API calls: lowest overhead, pay-as-you-go, appropriate for pre-PMF teams not yet committed to a cloud.

Gate 2 output is a hosting path, a monthly cost estimate using live pricing, and a compliance assessment — applied to the Gate 1 shortlist. Models that cannot be hosted compliantly are eliminated before cost is considered.

---

### Gate 3 — Trajectory

**The question: What does this model look like in 12 months, and what does it cost to leave?**

This is the gate that almost never gets asked. It's also the one responsible for the most expensive problems. AI shifts every 3–6 months. A model that is the right technical choice today may be deprecated, superseded, or repriced within a year.

Gate 3 scores each remaining model on how easily the team could migrate away from it if they needed to. The score isn't about whether you will need to switch — it's about what it would cost if you did.

**Lock-in Tolerance** is a strategic question about how acceptable vendor dependency is.

At Low, the framework eliminates models with non-portable fine-tuning and penalises anything without compatibility with LiteLLM or Portkey — abstraction layers that let you swap models with a config change rather than a code change. Open-weight models like Llama and Mistral, which can be hosted anywhere independently of the original provider, score well at this tolerance level.

At High, proprietary fine-tuning and provider-specific features are in scope, and the replaceability score is de-weighted in the final recommendation.

**Time Horizon** determines how heavily trajectory risk is weighted against current benchmark performance.

At 3 months, benchmark scores dominate — the landscape won't shift enough to make today's right answer wrong within a quarter.

At 12 months, trajectory risk is significant and a model with an unclear roadmap scores lower than a slightly weaker model with a stable development path.

At 24 months, trajectory is the dominant factor once the capability threshold is cleared. A model scoring 7/8 on the replaceability index is routinely preferred over a slightly stronger model scoring 3/8 at this horizon.

**Migration Capacity** adjusts how heavily the replaceability index is weighted based on the team's realistic ability to execute a switch.

At None — no bandwidth for a migration — whatever is chosen is effectively permanent, so the framework weights replaceability heavily.

At Strong — where a model-agnostic abstraction layer already sits between the application and the API — any model can be exited in days, and the replaceability score is de-weighted accordingly.

These three parameters produce the **Replaceability Index**, scored 0–8:

| Factor | Score |
|---|---|
| Open weights available (model can be hosted anywhere, independent of the provider) | +2 |
| API abstraction layer compatible — LiteLLM or Portkey support means switching is a config change | +2 |
| Active model family, not deprecated — provider has committed to ongoing development | +2 |
| Provider financial stability — lowers probability of a forced migration | +1 |
| Community and OSS momentum — fine-tunes, adapters, and tooling exist independently of the provider | +1 |

7–8 is LOW migration risk. 4–6 is MEDIUM. 0–3 is HIGH and the recommendation is flagged with a warning. The final rating adjusts one level upward if Migration Capacity is Strong and one level downward if it is None.

Gate 3 output is a Replaceability Index score, a migration risk rating, and a 12-month trajectory assessment for each model that cleared Gates 1 and 2.

---

### Why the sequence matters

Gate 1 shortlists on capability.
Gate 2 filters by operational and financial reality.
Gate 3 scores on exit cost.

A model that tops Gate 1 but fails Gate 2 — the strongest reasoning model on GPQA Diamond, but requiring self-hosting that a 2-person pre-PMF team can't absorb — never reaches the recommendation.

A model that clears both Gates 1 and 2 but scores 2/8 on the Replaceability Index gets flagged as HIGH migration risk. That might be fine at a 3-month horizon. It is genuinely problematic at 24.

The sequence prevents the pattern that shows up most often in postmortems: a team picked a technically excellent model that created an operational, financial, or strategic problem six months later.

---

## What's next for the tool

The agent is a POC. These are the gaps that still matter.

**A real cost calculator.** Right now budget is estimated in tiers. The next version should take actual call volume — requests per day, average input and output token counts — and produce a monthly cost comparison across the shortlisted models using live pricing. The documentation checklist currently says "take a screenshot of the cost comparison." It should just be the cost comparison.

**An eval stack recommendation.** Model selection and eval design are the same decision. Once you know which model you're running and what task type it's handling, the agent should output a starter eval design — the specific metrics, test set structure, and evaluation framework (Ragas, LMQL, or custom) that fits the recommendation. Right now that work still happens separately and usually too late.

**Team profile memory.** Every session starts from scratch. A saved team profile — size, stage, cloud provider, sovereignty requirements — would let returning users focus immediately on the task-type specific question rather than re-entering infrastructure context they've already answered.

**A migration planner for teams already in production.** If you have a model running today and want to know what it would cost to move, the agent should take your current model as an input and produce an assessment: prompt engineering delta, fine-tuning portability, and eval rebuild effort. That's a different use case from initial selection, and it has a large addressable audience.

---
