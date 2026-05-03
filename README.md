# LLM Selector — A Three-Gate Framework Agent for Model Selection

**by Ram Joshi** · AI Product Manager, Munich

---

## 1. Why This Exists

Every team I've talked to in the last 12 months has made the same mistake: they picked an LLM the same way they'd pick a SaaS tool. They went to a leaderboard, sorted by the top score, and shipped it. Six months later they're either locked into a model that doesn't fit their task type, running a $40k/month API bill they didn't plan for, or trying to migrate off a fine-tuned model with no abstraction layer in place.

The problem isn't that teams pick the wrong model. The problem is that **nobody defined what "right" meant before the choice was made.** Model selection is treated as a single decision when it's actually three separate decisions — capability, infrastructure, and trajectory — each requiring different inputs and carrying different long-term consequences. Collapsing them into one call ("which model should we use?") is where the errors compound.

This agent exists because that decision deserves a framework, not a vibe. And because a framework embedded in a tool that re-evaluates in real time is more useful than a framework in a doc nobody reads.

---

## 2. What It Does

LLM Selector is a single-page browser agent that takes one input — a plain-language description of what you're building — and returns a structured model recommendation with full reasoning across three decision gates.

**Input:** One sentence to one paragraph describing your use case, team, and constraints.

**Output:** A live dashboard showing:
- Primary model recommendation with rationale
- Runner-up model and when to switch to it
- Three-gate analysis (Capability / Infrastructure / Trajectory)
- Replaceability score (0–8) and migration risk rating
- Red flags detected in your requirement profile
- Documentation checklist — the two screenshots every PM should have before sign-off
- Decision log you can paste directly into a requirements doc

**Key behaviour:** After the initial recommendation, every parameter on the dashboard is adjustable — task type, context window, team size, budget, data sovereignty, cloud provider, lock-in tolerance, time horizon. Change any of them and the recommendation recalculates automatically. This turns a one-time output into an interactive decision tool.

The agent runs entirely in the browser. Your Anthropic API key is stored in `localStorage` and never leaves your device except in direct calls to Anthropic's API.

---

## 3. The Framework

The Three-Gate Framework treats model selection as three sequential decisions, each with a distinct set of inputs and a clear failure mode.

### Gate 1 — Capability

**The question:** Can this model actually do what we need?

The most common mistake here is sorting a general leaderboard and picking the top result. A contract extraction task is a retrieval and structured output problem. The model ranked #1 overall may rank #6 on retrieval. That's the model you were about to ship.

The agent evaluates capability by task type — not general intelligence. Supported task types include retrieval, extraction, Q&A, reasoning, code, summarization, tool use, multimodal, agentic workflow, multi-agent orchestration, function calling, browser/computer use, RAG pipelines, and agent memory and state.

Context window is evaluated separately from benchmark rank. A model with a strong retrieval score but an 8k context window is the wrong call for a pipeline that processes 50-page legal documents.

### Gate 2 — Infrastructure

**The question:** Where do we host it, and what does that bet mean for our team?

A 2-engineer startup running GPT-4 through the OpenAI API and a 20-engineer enterprise running the same model through Azure are making fundamentally different bets — on reliability, security, cost structure, and who owns the problem when it breaks.

The agent evaluates team size, company stage, monthly budget, data sovereignty requirements, and cloud provider preference together. The output isn't just a hosting recommendation — it's a cost estimate at your volume and a compliance assessment.

Key rule enforced by the agent: if you are pre-PMF with fewer than three engineers, it will always recommend a closed-source API. The infrastructure overhead of self-hosting open-source at that stage will slow you down more than the cost savings are worth.

### Gate 3 — Trajectory

**The question:** What does this model look like in 12 months, and can we migrate if we need to?

This is the gate that almost never gets discussed in PM-engineering conversations. AI shifts every 3–6 months. The model you choose today should be one you can migrate away from without rebuilding your entire eval stack.

The agent produces a Replaceability Index scored 0–8 based on:

| Factor | Score |
|---|---|
| Open weights available | +2 |
| API abstraction layer compatible (LiteLLM, Portkey) | +2 |
| Active model family, not deprecated | +2 |
| Provider financial stability | +1 |
| Community / OSS momentum | +1 |

Migration risk (LOW / MEDIUM / HIGH) is derived from the replaceability score combined with the team's stated migration capacity and time horizon.

### Why the gates are sequential

Gate 1 produces a shortlist. Gate 2 filters it by operational reality. Gate 3 scores the survivors on long-term risk. A model that passes Gate 1 brilliantly but fails Gate 2 (wrong hosting model for your compliance requirements) never reaches the recommendation. This prevents the common failure of picking a technically excellent model that creates an operational or commercial problem six months later.

---

## 4. What's Next

These are the next meaningful increments — not feature bloat, but gaps that still limit the tool's usefulness in a real product decision context. Live leaderboard data via web search has shipped; these are what remain.

**Cost calculator.** The infrastructure gate estimates cost in buckets. It should take actual call volume (requests/day, average token count) and output a monthly cost comparison across the shortlisted models using live pricing. The two-screenshot checklist becomes one automated output.

**Eval stack recommendation.** After the model is selected, the next decision is how to evaluate it. The agent should output a starter eval design — task-specific metrics, a test set structure, and a suggested framework (Ragas, LMQL, custom) — as a second tab on the dashboard.

**Team profile memory.** Right now every session starts fresh. A saved team profile (size, stage, cloud provider, sovereignty requirements) would let returning users skip re-entering the infrastructure context and focus on the task-type specific question.

**Model migration planner.** For teams that already have a model in production, an input for "current model" should trigger a migration cost assessment — estimating prompt engineering delta, fine-tuning portability, and eval rebuild effort.

**Export to requirements doc.** The decision log and three-gate analysis should export as a formatted Word or Notion doc that drops directly into a PRD. The agent already produces all the content; the export is just packaging.

---

## 5. Insights from Building This Agent POC

These are the things I learned that I didn't expect going in. Not principles — concrete findings that would have changed how I scoped this if I'd known them upfront.

**Structured JSON output is fragile in ways you don't anticipate.** Even with explicit instructions to return only raw JSON, the model frequently wraps responses in markdown code fences. You need a fence-stripping pre-processor on every parse call. The fix is one function, but if you don't know to expect it, you'll spend an hour debugging a `SyntaxError` on a response that looks correct in the console.

**One input is a better UX than ten, but it shifts complexity into the prompt.** Moving from a 10-question intake to a single free-text description made the product dramatically easier to use. But all the inference work that the intake questions were doing has to move into the system prompt and output schema. The prompt engineering surface area roughly doubled when the UX surface area halved. This is the right trade — users shouldn't pay for model complexity — but the PM has to carry that cost somewhere.

**Parameter controls change how users think, not just what they select.** When users can adjust parameters after seeing the initial recommendation, they don't just tune the output — they build a mental model of the decision space. Watching the recommendation change when you toggle "data sovereignty required" teaches you something about the infrastructure gate that reading about it doesn't. Interactivity is a learning mechanism, not just a personalisation feature.

**Debounce is a product decision, not just a performance optimisation.** Setting the recalculation delay to 1.2 seconds after a parameter change wasn't arbitrary. Too short (< 500ms) and every slider drag fires multiple API calls and the UI feels unstable. Too long (> 2s) and the connection between the parameter change and the recommendation update feels broken. The delay is part of the perceived responsiveness of the product.

**The replaceability index made abstract risk concrete.** Before building the 0–8 score, trajectory risk was a narrative paragraph that most users skimmed. Once it became a number with a visible bar, users engaged with it — asking why it was 5 and not 7, comparing it across models. Quantifying something fuzzy doesn't make it precise; it makes it discussable. That's the goal for a decision-support tool.

**The two-screenshot checklist is the most underrated feature.** It's not technically interesting — it's just two strings in a list. But in user testing, it was consistently cited as immediately actionable. Most PM frameworks produce analysis. This one produces a task: go capture these two artefacts before you make the final call. The handoff from insight to action is where most decision tools break down.

**Embedding live data retrieval in the analysis call is better than a separate fetch.** The first implementation used two sequential API calls: one to fetch benchmark data from artificialanalysis.ai via web_search, then one for the actual analysis. This reliably hit rate limits (HTTP 429) and introduced a second failure mode. The correct architecture is one call with the `web_search` tool attached — Claude searches and analyses in a single agentic turn. The CORS problem (you can't call artificialanalysis.ai directly from a browser) disappears entirely, there's no second API key to manage, and the failure surface halves. The rule generalises: when you're tempted to chain two LLM calls, ask whether the second call can be a tool in the first.

---

## Stack

- Vanilla HTML / CSS / JavaScript — no framework, no build step
- IBM Plex Sans / Serif / Mono — type system
- Anthropic Claude Sonnet 4 (`claude-sonnet-4-20250514`) via direct browser API call
- `web_search_20250305` tool for live leaderboard data from artificialanalysis.ai
- `localStorage` for API key persistence
- Deployable to any static host (GitHub Pages, Netlify, Vercel)

## Usage

1. Clone the repo
2. Open `index.html` in a browser
3. Enter your Anthropic API key when prompted (stored locally, never transmitted elsewhere)
4. Describe what you're building
5. Adjust parameters on the dashboard to explore how the recommendation changes

## License

MIT
