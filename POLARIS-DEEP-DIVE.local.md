# CloudLabs Polaris - Complete Deep Dive (Team Lead Reference)

> Local, untracked working doc for Aryan (team lead). Not committed, not pushed.
> Goal: you understand every part of this product - what, why, and how - well
> enough to make decisions and answer any jury question, including the backend
> Girish built. House style: no em-dashes.
>
> Last updated: 2026-06-27. Source of truth for the team remains `context.md`;
> this doc is the narrated, in-depth companion.

---

## 0. How to read this doc

- Sections 1-5 are the "story": what we built and why. Read these to pitch it.
- **Section 1A** is Polaris explained with no jargon via one running example
  (Priya, stuck on step 8). Best thing to show non-technical people.
- Section 5 explains the **"agents are config"** idea in both plain-English (the
  Netflix analogy) and technical terms, with a comparison table vs the naive
  "one bot per lab" approach.
- Section 6 is the "machine": the backend, module by module (technical).
- **Section 6B** is the same machine in plain English, part by part, with
  examples. **Section 6C** is "the smart bits" - the wow factors to show off.
- Sections 7-9 cover cost, the ODL contract, and the portal.
- Sections 10-15 are status, risks, decisions, jury prep, a glossary, and a code
  map.
- Anywhere you see `file.py`, that is a real file under `/backend/app`.

---

## 1. The product in one page

**Polaris** is a per-lab AI support agent embedded inside CloudLabs hands-on labs.
When a learner gets stuck mid-lab, they ask Polaris. It does three things:

1. **Answers, grounded in that exact lab's guide** (plus an optional FAQ, the
   model's own knowledge, and web search for fresh Azure/product changes).
2. **Escalates intelligently** when it is not confident: instead of guessing, it
   produces a pre-packaged, structured support ticket (lab, step, question, what
   it already tried, an AI summary).
3. **Turns every question into a friction signal** tagged to a lab step.
   Aggregated across a cohort, this becomes a **lab-improvement digest** for the
   lab owner: "40 of 100 learners asked about step 7, here are the 3 root
   confusions and suggested guide fixes."

Two surfaces, one engine:
- **Learner side** (customer-facing): the Polaris chat embedded in the lab.
- **Builder side** (internal): a portal where a lab developer enters an ODL ID,
  clicks Build, and gets back a deployed agent endpoint + agent ID.

**Bucket:** Improve Customer Experience (primary); also Reduce Cost + Speed Up
Delivery.

**The one-line pitch:** Polaris cuts support load AND makes every lab get better
the more it is run.

---

## 1A. Polaris in plain English (one running example)

Forget the tech for a second. Here is the whole product as a story.

**Meet Priya.** She is doing a CloudLabs hands-on lab called "Deploy a
Containerized App to Azure Container Apps" (this is real lab `70001` in our demo).
She is on **step 8**: pushing her Docker image to Azure Container Registry. She
hits an error: `unauthorized: authentication required`. She is stuck.

**Without Polaris:** she Googles, gets generic answers that do not match this
lab's exact steps, wastes 20 minutes, or files a ticket that just says "step 8 not
working" and waits an hour for support to ask her what she means.

**With Polaris:** she clicks the chat in the lab and types "I get unauthorized when
I push to ACR." Polaris:
1. Knows it is *this* lab (not the other 999), because the chat is bound to this
   lab's agent.
2. Looks inside *this lab's own guide*, finds the step 8 section, and even spots
   the guide's own "common gotcha" note: your `az acr login` token expired,
   re-run it.
3. Answers her in plain language with the exact fix and says "from challenge-1.md,
   step 8" so she can trust it is from the real guide, not made up.

**Now the clever part - what if Polaris does not know?** Say Priya asks "can I get
a discount on my Azure bill?" That is not in the lab guide. Polaris does NOT guess
(guessing is how chatbots embarrass you). Instead it says "I am not sure, I have
raised this to the lab team," and files a *clean, complete* ticket: which lab,
which step, her exact question, what it tried, and a one-line summary. Support
opens a ticket that is already 80 percent sorted.

**And the bird nobody expects (our differentiator).** Every question Priya and her
99 classmates ask is quietly recorded against the step they were on. After the
event, the lab owner gets a report: "18 of 100 learners got stuck on step 8's ACR
login. Suggested fix: add a callout about token expiry before the push command."
The lab literally gets better for the next batch of learners. Run it again, it
improves again.

**That is Polaris in one breath:** it answers from the real guide, escalates
cleanly when unsure, and turns every confused question into a concrete way to
improve the lab. Same engine works for all ~1000 labs.

> Use this Priya story when you explain Polaris to anyone non-technical. It lands
> faster than any architecture diagram.

---

## 2. Why we are building it (the problem and the business case)

### The problem
- Learners get stuck mid-lab. Today they wait, raise a vague ticket ("it's not
  working"), or abandon the lab.
- Support staff burn time reconstructing context for every ticket from scratch.
- Lab owners (TOs - Technical Owners) have no clean signal about *where* their
  labs confuse people.
- Generic chatbots fail on technical/Azure content and on labs whose steps depend
  on one specific guide. A plain "GPT in an iframe" hallucinates lab steps.

### Why it matters to Spektra (the business case)
- **CloudLabs** is Spektra's hands-on lab platform: 500+ ISVs/enterprises/
  universities, multi-cloud (Azure, AWS, GCP, Oracle), ~1000 labs.
- Azure Lab Services retires 28 June 2027 and Microsoft recommends CloudLabs as
  the alternative. That is a tailwind: more labs, more learners, more support load.
- Polaris attacks support cost (deflect + pre-triage tickets) AND content quality
  (the improvement digest) at once. It scales because it is downstream of the
  existing lab-production pipeline (Cosmos builds labs, Atlas maps cost, Polaris
  guides the learner).

### Who the users are
- **Learner**: asks Polaris inside the lab. Wants an answer or fast help.
- **TO / lab owner**: builds an agent for their lab, receives the improvement
  digest. Wants fewer tickets and a better lab.
- **Support team**: receives pre-triaged escalations instead of vague tickets.

### Why now / why it needs modern AI
- Grounded retrieval + a confidence gate + LLM clustering of confusions are only
  possible with current models. This is a clean "wouldn't exist without modern AI"
  story, which maps to the hackathon's theme score.

---

## 3. The three pillars (never cut these)

These three ARE the pitch. Everything else is enhancement.

1. **Grounded answer** - retrieval from the lab's own guide first, with citations.
2. **Structured escalation** - a pre-triaged ticket when confidence is low, not a
   guess.
3. **Lab-improvement digest** - the "second bird"; every question becomes
   step-level evidence of confusion that tells the author what to rewrite.

Enhancements (cut from the bottom if behind): multi-lingual -> Socratic toggle ->
premium side-panel -> resolved-escalation-to-FAQ loop.

> Framing note for the jury: call the digest a "continuous lab-improvement signal",
> NOT "RLHF". RLHF implies weight updates; what we have is human-readable,
> step-level evidence of real confusion. Arguably more useful, and a sharp judge
> will catch the wrong term.

---

## 4. The hackathon scorecard (how this maps to points)

SpektraX '26. Submission due Sun 05 July 23:59 IST. Judged on:
- Innovation 25% - the second bird (self-improving labs) is the "I would not have
  thought of that" hook.
- Business Impact & Feasibility 25% - real support-cost + content-quality value
  on a real platform with a real tailwind.
- Technical Execution 20% - grounded RAG, the two-signal confidence gate, cost
  levers, "agents are config not deployments". Survives a sharp question.
- Alignment with Theme 10% - genAI-core by design.
- Presentation 10% - the live "ODL ID to agent in 60s" demo opener.
- Team Composition 10% - three people each owning a surface.

Hard rules we satisfy: meaningful genAI use; a runnable/clickable artefact (live
demo is mandatory); synthetic data only; Spektra accounts only; AI disclosure is
mandatory (write it as we go).

---

## 5. System architecture (two surfaces, one engine)

```
   BUILDER PORTAL (internal)                 LEARNER WIDGET (in-lab)
   enter ODL ID + optional FAQ               ask a question (+ current step)
        |  POST /v1/agents                        |  POST /v1/agents/{id}/chat (SSE)
        v                                          v
   ============================  POLARIS ENGINE  ============================
   one stateless FastAPI app; per-lab context loaded by agent_id at request time
   - control plane: build/manage agents      - data plane: grounded chat
   - ingest: resolve guide -> chunk -> embed  - retrieve -> confidence gate ->
     -> upsert into pgvector                    answer (stream) OR escalation
   - async event bus -> question_log          - semantic cache, prompt cache,
   - digest job -> lab-improvement insights     model routing, token accounting
   =========================================================================
        |                          |                         |
        v                          v                         v
   Postgres + pgvector       Azure OpenAI (Foundry)     ODL API (mock today)
   (configs, vectors,        chat + embeddings          resolve guide by ODL ID
    logs, escalations,
    cache, digests)
```

### The load-bearing idea: "agents are config, not deployments"

This is the smartest decision in the whole product, so it is worth really
understanding. Both the layman version and the technical version below.

**Layman version (the Netflix analogy).** Netflix has hundreds of millions of
users, each with their own profile: watchlist, preferences, viewing history.
Netflix does NOT run a separate copy of Netflix for every user. It runs *one*
service that, when you log in, loads *your* profile by your ID and behaves like
"your" Netflix. Polaris works the same way. We do not build a separate chatbot for
each lab. We run *one* engine that, when a question comes in, loads *that lab's*
profile (its guide, its FAQ, its settings) by an `agent_id` and behaves like
"that lab's" expert. An "agent" is not a running program; it is just a row of
configuration that the one engine reads on demand.

Another way to picture it: imagine a single brilliant help-desk person sitting
with 1000 labelled folders. When a call comes in for lab 70001, they grab folder
70001, read it, and answer as the expert for that lab. They do not need 1000
clones of themselves sitting around. The "agent" is the folder, not the person.

**Why this is different from "one agent deployed per lab" (the naive way).** The
obvious approach is: every lab gets its own deployed chatbot service. That sounds
fine for 5 labs and falls apart at 1000.

| Dimension | One agent deployed per lab | Agents as config (what we do) |
|---|---|---|
| Cost when a lab is idle | Each service still runs / reserves resources = paying for 1000 mostly-idle things | An idle lab is just rows at rest = ~zero cost |
| Spinning up a new lab | Provision + deploy a new service (slow, ops-heavy) | Insert a config row + embed the guide (seconds) |
| Updating the engine logic | Redeploy 1000 services | Deploy 1 engine; all labs get it instantly |
| Scaling under load | Scale each lab's service separately | Scale the one engine horizontally; any replica serves any lab |
| Memory / compute footprint | 1000 processes | 1 process (a few replicas), data in Postgres |
| Isolation (no context bleed) | Physical, but expensive | Enforced by `agent_id` on every query, by construction |
| A bug / crash | Risk per service to manage | One codebase to fix |

**Technical version.** We run one **stateless** engine (FastAPI on Azure Container
Apps, scale-to-zero). It holds no per-lab state in memory. All state - agent
config, the embedded guide chunks, logs, escalations, cache - lives in Postgres +
pgvector. Every query is filtered by `agent_id`, so one lab can never see
another's content (isolation by construction, not by hoping). Because compute is
stateless and state is external, we can scale replicas freely, restart safely, and
let idle labs cost nothing. This is the single best answer to "how does this scale
and what does it cost?": agents are rows, not deployments.

> Wow line for the pitch: "We do not deploy 1000 bots. We run one engine that
> becomes any lab on demand, the way Netflix is one service that becomes your
> Netflix when you log in. That is why 1000 labs costs almost nothing at rest."

### Two request lifecycles (memorize these two flows)

**Build flow (control plane):**
1. Portal calls `POST /v1/agents {odl_id, faq?, settings}`.
2. `routers/agents.py` creates the agent row in `building` status via the agent
   store, returns `202 {agent_id, build_id, status}` immediately, and kicks off
   the build in a background task.
3. `services/ingest.py` resolves the guide (ODL API), chunks it, batch-embeds the
   chunks, and upserts them into pgvector. Idempotent by per-file content hash.
4. On success the agent is finalized to `ready` (lab name, chunk count, sources,
   and the stable system prefix are persisted). On failure it is marked `failed`.
5. Portal polls `GET /v1/agents/{id}` until `ready`.

**Chat flow (data plane):**
1. Widget calls `POST /v1/agents/{id}/chat {session_id, message, step?, locale?}`.
2. Embed the question.
3. **Semantic cache check**: if a near-identical prior question exists for this
   agent, serve the cached answer instantly (cheapest path).
4. **Retrieve**: vector search within `agent_id`, step-boosted, FAQ-weighted.
5. **Two-signal confidence gate** (the spine): decide answer vs escalate.
6. **Stream** the grounded answer token by token over SSE, then a final event
   with citations + confidence; OR emit a structured escalation payload.
7. Fire the question log (and cache write) onto the async event bus so logging
   never sits on the answer path.

---

## 6. Backend deep dive (module by module)

Location: `/backend/app`. Stack: Python + FastAPI, asyncpg + pgvector, Azure
OpenAI (Foundry) via a provider shim, Pydantic for contracts. Runs locally in
Docker; designed for Azure Container Apps.

> Mental model of the folders:
> - `routers/` = the HTTP surface (control plane + data plane).
> - `services/` = the brains (ingest, retrieval, gate, llm, eventbus, digest,
>   agent_store).
> - `db.py` + `schema.sql` = all persistence.
> - `config.py` = all tunables.
> - `schemas.py` = the API wire contracts.

### 6.1 `config.py` - all settings in one typed object
Loaded from environment / `.env`. Every module reads `get_settings()` instead of
`os.environ`, so config cannot drift and a provider swap stays config, not code.
Key knobs: Azure endpoint/keys, deployment names (`chat_deployment` =
gpt-5-mini, `embedding_deployment` = text-embedding-3-small, 1536 dims),
`retrieval_top_k`, the gate thresholds (`answer_threshold` 0.55,
`escalation_threshold` 0.35), and the two-signal gate flags I added
(`gate_self_check`, `gate_retrieval_weight`).

### 6.2 `services/llm.py` - the provider shim (the only file that talks to Azure)
This is the locked decision "provider shim keeps a future model swap cheap."
Nothing else in the engine imports the OpenAI SDK. Methods:
- `embed(texts)` - batch embeddings (returns 1536-dim vectors).
- `stream_chat(system, user, large=?)` - streams answer tokens; captures token
  usage including `cached_tokens` (proves prompt caching is working).
- `complete_json(system, user)` - non-streaming JSON completion, used by the
  digest clustering.
- `verify_grounding(context, question)` - **my addition**: the gate's Signal B; a
  strict JSON self-check on whether the context actually answers the question.
- **Offline mode**: if `POLARIS_FAKE_LLM=1`, all methods return deterministic
  fakes, so the whole engine runs with no Azure seat (great for dev and a safe
  fallback if the demo Wi-Fi dies).

### 6.3 `schema.sql` - the data model (one Postgres + pgvector store)
Every table is scoped by `agent_id` for tenant isolation (one engine, one set of
tables, filtered by id). Tables:
- `agents` - the agent config: `agent_id, odl_id, status, settings (jsonb:
  socratic/language), source_hashes, config_version, lab_name, chunk_count,
  sources, system_prefix, timestamps`. (The last four columns are from my
  persist-agents PR.)
- `chunks` - the retrieval corpus: `agent_id, odl_id, source (guide|faq),
  source_file, step, order, content, content_hash, embedding vector(1536)`. HNSW
  cosine index on the embedding; btree on `(agent_id, step)`.
- `qa_cache` - the semantic answer cache: question + `q_embedding` + answer +
  citations + hits. HNSW index.
- `question_log` - every question: confidence, escalated, tokens in/out, model,
  step, session. Powers FinOps and the digest.
- `escalations` - the structured ticket payload.
- `digests` - the per-lab improvement digest output for the Insights page.

### 6.4 `db.py` - the async access layer
Owns the asyncpg pool and every query. Important property: **the pool is
optional**. If Postgres is unreachable, `is_ready()` returns False and the engine
degrades gracefully (build still resolves + chunks; chat escalates on empty
retrieval). When Postgres is present, `schema.sql` is applied on startup
(idempotent) and everything runs for real. Functions: agent CRUD
(`create_agent`, `finalize_agent`, `get_agent`, `list_agents`,
`update_agent_settings`, `get_system_prefix`, `delete_agent`, `get_source_hashes`),
chunk upsert + vector search (`replace_chunks`, `search_chunks`), the semantic
cache (`cache_lookup`, `cache_store`), `log_question`, `store_escalation`, and the
digest queries.

### 6.5 `services/agent_store.py` - where agent config lives (my persist-agents PR)
A thin abstraction over agent config that is **DB-authoritative when Postgres is
ready and falls back to in-memory when offline**. Before this, agents lived only
in an in-memory dict, so they vanished on restart and were not shared across
replicas - which contradicted "agents are config." Now an agent survives restart.
It also warm-caches the large per-lab system prefix so the chat hot path does not
round-trip the DB for it. The control plane (`routers/agents.py`) calls
`create` -> `finalize`/`set_status`; chat calls `get` + `get_prefix`.

### 6.6 `services/chunker.py` - structure-aware chunking
Splits the combined guide text by markdown heading, infers the **lab step** from
headings like "## Step 7" or "# Challenge 1" and carries it forward, and packs
sections to a ~550-800 word budget with small overlap. Every chunk knows its
`source_file`, `order`, and `step`, so answers can cite and the digest can
aggregate by step. The `<!-- source: ... -->` markers come from the shared
`resolve_guide()` so each chunk is attributed to the right file.

### 6.7 `services/ingest.py` - the build pipeline
`build_agent(agent_id, odl_id, faq)` does: resolve the guide -> build the stable
per-lab system prefix (lab name + description + capped overview, used for prompt
caching) -> chunk the guide (+ FAQ if provided) -> compute per-file content hashes
-> **idempotency check** (if hashes match the last build, skip re-embedding) ->
batch-embed -> upsert chunks into pgvector. It owns only the chunk corpus now; the
agent row lifecycle is owned by the control plane (post my PR).

### 6.8 `services/retrieve.py` - retrieval + the confidence gate (THE SPINE)
This is the most important and most jury-probed part.

- `retrieve(agent_id, q_embedding, step)` - cosine ANN within the agent,
  over-fetch then re-rank in Python with the grounding priority: FAQ weighted
  above guide, and a boost for chunks on the learner's current step.
- `confidence_from(hits)` - **Signal A**: retrieval coverage = blend of the best
  hit's similarity and how many solid chunks corroborate it.
- `decide(...)` - **my addition**: blends Signal A with **Signal B** (the model
  self-check from `llm.verify_grounding`) and routes three ways. Crucially, a
  model verdict of "cannot answer" forces a human escalation regardless of how
  good retrieval looked. This catches "retrieved something similar that does not
  actually answer the question", which pure similarity misses.
- `route(confidence)` - the original two-band routing, still used when the
  self-check is off or there are no hits.

**The three outcomes of the gate:**
1. `answer` - good confidence: answer with the cheap model + citations.
2. `escalate_model` - marginal: escalate the model tier once (more capable model)
   before giving up.
3. `escalate_human` - low confidence OR model says it cannot answer: emit the
   structured escalation payload instead of guessing.

Why two signals: it is the defensible answer to "how do you avoid confident-wrong
answers?" One coverage number can be fooled; an independent model check on the
actual retrieved text closes the gap.

### 6.9 Prompt caching + model routing + token accounting (Girish, build step 4)
- **Prompt-prefix caching**: each lab has a stable system prefix built at ingest
  time and sent as the system message, identical across the cohort, so Azure
  serves it from prompt cache (`cached_tokens > 0` proves it). The dynamic
  retrieved chunks + question go last.
- **Model routing**: cheap model by default; escalate the tier once on marginal
  coverage; structured human escalation when low. (Both tiers point at gpt-5-mini
  today because the seat has one chat model; the routing is real and ready for a
  second model.)
- **Token accounting**: `tokens_in/out/cached` captured via streaming usage and
  written to `question_log` + surfaced in the chat final event = real per-agent
  FinOps.

### 6.10 `services/eventbus.py` - async logging (Service Bus stand-in)
`chat -> emit(event) -> asyncio.Queue -> background worker -> Postgres`. `emit()`
is non-blocking (`put_nowait`); under backpressure it drops and counts rather than
stalling an answer. Events are plain dataclasses (`LogEvent`, `CacheEvent`), so
swapping in real Azure Service Bus later is a localized change. Started/stopped in
the app lifespan. This keeps logging and cache writes off the answer critical
path. Note: cache writes are now eventually-consistent, so rapid-fire duplicates
may miss the cache until the worker drains (negligible across a real cohort).

### 6.11 `services/digest.py` - the lab-improvement digest (the second bird)
`build_digest(agent_id, odl_id)` aggregates `question_log` into metrics (total
questions, escalation rate, cache-hit rate, tokens, per-step volume) and asks the
model (JSON mode) to **cluster the step-tagged questions into root confusions,
each with one concrete suggested guide fix**, plus a headline. Stored in the
`digests` table and served at `GET /v1/agents/{id}/insights`. Also runnable as a
standalone scheduled job (`python -m app.services.digest`), which is the stand-in
for the production scheduled ACA Job over the Batch API.

### 6.12 The two API planes (`routers/agents.py`, `routers/chat.py`)
All under `/v1` (set by `api_prefix`).

Control plane (portal -> engine):
- `POST /v1/agents` - build an agent `{odl_id, faq?, settings}` -> `202`.
- `GET /v1/agents` - list agents.
- `GET /v1/agents/{id}` - config + build status (portal polls this).
- `PATCH /v1/agents/{id}` - update settings (bumps config_version).
- `POST /v1/agents/{id}/reindex` - force a rebuild on guide drift (idempotent).
- `GET /v1/agents/{id}/insights` - the lab-improvement digest.
- `DELETE /v1/agents/{id}`.

Data plane (widget -> engine):
- `POST /v1/agents/{id}/chat` - SSE stream: `token` events then a `final` event
  with citations, confidence, and an escalation payload if it escalated.
- `POST /v1/agents/{id}/escalations` - create a structured ticket.
- `POST /v1/agents/{id}/feedback` - thumbs up/down (a friction signal).

### 6.13 Offline / fake mode (important for a safe demo)
`POLARIS_FAKE_LLM=1 uvicorn app.main:app` runs the whole engine with deterministic
fake embeddings + stub answers and no Azure seat. Combined with the graceful
no-DB degradation, you can boot and click through the API with zero external
dependencies. This is your insurance if the demo network or Azure misbehaves.

---

## 6B. Each backend part in plain English (with examples)

Section 6 is the technical version. Here is the same machine explained the way you
would explain it to a smart friend who does not code. Each part: what it is, a
plain analogy, and a concrete example from Priya's lab.

- **The engine (`main.py`)** - the one worker that handles everything. *Analogy:*
  the single help-desk person with 1000 folders. *Example:* Priya's question and a
  different learner's question on a different lab both hit the same engine; it just
  grabs the right folder (agent) for each.

- **Config (`config.py`)** - one place that holds all the dials (which model, how
  strict the confidence gate is, etc). *Analogy:* the settings menu for the whole
  app. *Example:* if escalations feel too trigger-happy, we change one number here,
  not ten files.

- **Provider shim (`llm.py`)** - the only part that actually talks to the AI model.
  *Analogy:* a universal power adapter; the rest of the device does not care which
  country's socket is behind it. *Example:* today it plugs into Azure OpenAI; if we
  switched to Anthropic, only this one file changes. It also has a "fake mode" so
  the whole app runs with no AI at all for testing.

- **The guide resolver + chunker (`ingest.py`, `chunker.py`)** - takes a lab's
  long guide and chops it into bite-sized, labelled pieces the AI can search.
  *Analogy:* turning a textbook into a deck of index cards, each tagged with its
  chapter and page. *Example:* lab 70001's guide becomes ~19 cards; the card about
  "step 8, push to ACR" is tagged step 8, so it is easy to find later.

- **Embeddings + pgvector (the search brain)** - every card is turned into a list
  of numbers ("embedding") that captures its meaning, so we can find cards by
  *meaning*, not just keywords. *Analogy:* giving every index card GPS coordinates
  in "meaning space" so similar ideas sit near each other. *Example:* Priya types
  "unauthorized when I push" and we instantly find the "ACR login token expired"
  card even though she did not use those exact words.

- **Retrieval + the confidence gate (`retrieve.py`) - THE SPINE** - finds the best
  cards and then *decides whether they are actually good enough to answer*.
  *Analogy:* a careful librarian who, before answering, checks "do I really have a
  source for this, or am I about to wing it?" If unsure, they refer you to a
  specialist instead of guessing. *Example:* for the ACR question it has a strong,
  on-topic card, so it answers. For "Azure discount?" it has nothing relevant, so
  it refuses and escalates.

- **The two-signal gate (the part I added)** - the confidence check uses *two*
  independent opinions before answering: (1) "how well do the found cards match?"
  and (2) "does the AI itself, looking at those cards, say they truly answer this
  question?" Only if both agree do we answer. *Analogy:* a bank needing two
  signatures for a big cheque. *Example:* sometimes a card looks similar but does
  not actually answer; signal 2 catches that and we escalate instead of giving a
  confident-but-wrong reply.

- **Streaming answer (SSE)** - the answer appears word by word instead of after a
  long pause. *Analogy:* watching someone type a reply live vs waiting for a sealed
  letter. *Example:* Priya sees the fix start appearing in ~1 second, which feels
  fast and alive.

- **Structured escalation** - when Polaris is unsure, it does not dump the learner
  to a blank form; it files a complete ticket automatically. *Analogy:* a good
  receptionist who takes down every detail before transferring your call, so you
  do not repeat yourself. *Example:* the ticket already has lab name, step 8, the
  exact question, what Polaris tried, and a one-line summary.

- **Semantic cache (`qa_cache`)** - remembers answers to questions already asked,
  so the 40th person asking the same thing gets an instant, free answer.
  *Analogy:* an FAQ that writes itself. *Example:* once one learner asks the ACR
  question, the rest of the cohort get the cached answer without paying for the AI
  again.

- **Prompt-prefix caching** - the unchanging part of each lab's instructions is
  sent in a fixed way so the AI provider charges less for repeating it. *Analogy:*
  a stamp/letterhead the printer already has loaded, so it only prints the new
  lines. *Example:* lab 70001's standing instructions are billed cheaply across
  all 100 learners.

- **Model routing (cheap vs strong model)** - easy questions use the cheap fast
  model; only the hard, borderline ones get bumped to a stronger model. *Analogy:*
  a triage nurse; most cases the nurse handles, only the tricky ones go to the
  senior doctor. *Example:* Priya's clear question is handled by the cheap model;
  cost stays low.

- **Async event bus (`eventbus.py`)** - the "write this question to the log" work
  happens *after* the answer is sent, so logging never slows the learner down.
  *Analogy:* a waiter takes your order to the kitchen and rings it into the till
  later, not while you are still ordering. *Example:* Priya's answer streams
  instantly; the record of her question is saved a moment later in the background.

- **Agent store (the part I added)** - remembers each lab's agent in the database
  so it survives restarts and works across multiple engine copies. *Analogy:*
  saving your game to disk instead of only in memory; turn it off and on, your
  progress is still there. *Example:* if we redeploy the engine mid-event, every
  lab's agent is still there, no rebuild needed.

- **The digest / second bird (`digest.py`)** - reads all the logged questions for
  a lab and uses AI to group them into "here are the 3 things people kept getting
  confused about, and here is how to fix the guide." *Analogy:* a teacher reading
  the whole class's questions after an exam and realizing "everyone missed Q7, I
  taught it badly." *Example:* the lab owner learns step 8's ACR auth confuses
  people and gets a concrete suggested edit.

- **Graceful degradation + fake mode** - if the database or the AI is unavailable,
  the engine still runs in a reduced way instead of crashing. *Analogy:* a car
  that limps to the garage instead of stopping dead. *Example:* if demo Wi-Fi
  dies, we flip on fake mode and still click through the whole flow.

---

## 6C. The smart bits (developer wow factors)

If a developer or a sharp judge looks under the hood, these are the things that
should make them nod. Use these to show the work is engineered, not hacked.

1. **Agents are config, not deployments.** One stateless engine becomes any of
   1000 labs by loading a row. Cheap at rest, instant new labs, isolation by
   `agent_id`. (Full explanation + table in section 5.) This is the headline.

2. **The two-signal confidence gate.** Most "AI support" demos will confidently
   hallucinate. Ours needs *two independent yeses* (retrieval coverage AND a model
   self-check) before it answers, and a "no" from the model forces an escalation
   even when retrieval looked fine. Being *structurally* hard to be confidently
   wrong is rare in a hackathon build.

3. **The escalation is a feature, not a fallback.** "I do not know" is turned into
   a pre-triaged, fully-populated support ticket. Failure becomes a useful
   product moment, not an apology.

4. **The second bird (self-improving labs).** The same question logs that power
   support also feed an AI digest that tells the lab author exactly what to fix,
   per step. One mechanism, two payoffs (deflect tickets AND improve content). The
   "I would not have thought of that" factor.

5. **Step-tagged chunking is what makes the second bird possible.** Because every
   chunk and every logged question carries the lab step, we can say "step 8
   confuses people," not just "people are confused." A small, deliberate data-model
   choice that unlocks the whole differentiator.

6. **Semantic cache that doubles as a friction signal.** A repeat question is both
   the cheapest answer (served from cache) AND the strongest signal of confusion
   (still logged, so the digest counts it). One feature, two uses.

7. **Idempotent, content-hash builds.** Rebuild a lab after the guide changed and
   we only re-embed the files that actually changed. This directly addresses
   CloudLabs' real "lab drift" problem and saves tokens. Mature, not naive.

8. **Prompt-prefix caching, and we can prove it.** The stable per-lab prefix is
   sent so Azure serves it from cache, and we surface `cached_tokens` in the
   response so we can *show* the discount is real, not claimed.

9. **Async logging with backpressure.** Logging and cache writes ride an in-process
   queue (a Service Bus stand-in) so they never slow an answer; under overload it
   drops-and-counts instead of stalling. Production-shaped, swap-ready.

10. **Graceful degradation + offline fake mode.** The engine runs with no database
    and even with no AI seat. That is both good engineering and demo insurance.

11. **Provider shim = one swap point.** Every model call goes through one file, so
    switching providers is config, not a rewrite. We are not married to a vendor.

12. **pgvector, one store for everything.** Vectors AND relational data (configs,
    logs, escalations, cache) in a single Postgres. No separate vector DB to run
    or pay for, and a clean answer to "why not Azure AI Search" (it bills even
    when idle; we do not).

13. **Production-shaped, demo-pragmatic.** We designed the full Azure stack (Service
    Bus, APIM, Content Safety, ACA Jobs) and built robust in-process equivalents
    for the demo. We can speak to the scaled version without depending on it on
    stage. Judges reward "survives a sharp question."

> Pick your top 3 for the live demo: agents-as-config, the two-signal gate, and
> the second bird. They cover scale, trust, and innovation in one breath.

---

## 7. Cost levers (the FinOps story, in order of impact)
1. **Semantic cache** across a cohort - deflects a large share of LLM calls
   (most "step 7" questions are the same question).
2. **Prompt caching** of the per-lab prefix - discounts the repeated context.
3. **Model tiering** - small model answers most; large only on escalation.
4. **Step-scoped retrieval** - smaller, sharper context = fewer tokens.
5. **Scale-to-zero compute + idle vectors at rest** - idle labs cost ~nothing.
6. **Content-hash idempotent builds** - embed only what changed on drift.
7. **Batch API for the digest** - it is not latency-sensitive.

This is why "1000 agents" is cheap: agents are config, the cache + caching +
tiering cut tokens, and idle labs are just rows.

---

## 8. The shared ODL contract + the mock API (how a lab becomes grounding)
- A lab is identified by an **ODL ID** (bare integer, e.g. real `68217`, synthetic
  `70001`/`70002`).
- `/shared/contracts/odl.py` + `odl.ts` define the contract (kept in Py/TS sync):
  ODL ID -> name/description/`masterdoc_url`; the masterdoc is an array listing
  ordered `.md` guide files.
- `/shared/contracts/odl_resolver.py::resolve_guide()` walks ODL ID -> record ->
  masterdoc -> each `.md` file in order -> a single combined, chunk-ready text.
  The backend imports this so there is one source of truth.
- `/mocks/odl-api` is a FastAPI app that reproduces the real ODL API so we can
  develop without production access. Hybrid seed: one real public CloudLabs sample
  (`68217`, live GitHub masterdoc) + two fully offline synthetic labs
  (`70001` ACA deploy lab, `70002` RAG lab). When the real ODL API is available,
  point `ODL_API_BASE` at it with no contract change.

---

## 9. The portal (Person 2's surface, brief)
`/portal` is Next.js 15 + TypeScript + Tailwind v4. Surfaces: Home (editorial +
quick-launch), Builder (enter ODL ID + FAQ -> Build -> endpoint + agentID),
Agents (data table), Insights (the digest). It currently proxies the mock ODL API
and mocks the build call; pointing `POLARIS_API_BASE` at the real backend wires it
to the live engine with no UI change. Brand: the Polaris compass-rose mark + an
animated north-star hero.

---

## 10. Status: what is done vs not

Done (on `main`):
- Full engine skeleton, two-plane API, SSE chat.
- pgvector data layer: chunk upsert, vector search, semantic cache, escalation
  persistence, question logging.
- Live Azure Foundry seat (gpt-5-mini + text-embedding-3-small).
- Prompt caching + model routing + token accounting.
- Async logging (event bus) + the lab-improvement digest + insights endpoint.
- Two-signal confidence gate (my PR #7).
- Persistent agent config in Postgres (my PR #8).
- Mock ODL API + shared resolver. Builder portal UI.

Not done / open:
- **Per-launch session tokens** (data-plane auth so the widget cannot be reused
  across labs). The last real backend security gap.
- **Eval harness** to tune the gate thresholds with data (the "we measured our
  escalation precision" jury answer).
- **Azure deployment** (Container Apps) - we develop on local Docker pgvector to
  protect the MSDN credit; provision at demo time.
- **Portal <-> live backend** wiring (Insights page to the real digest endpoint).
- Web search grounding tier (designed, not built).
- Real ticketing/email connector (we default to formatted payload + email).
- Resolved-escalation-to-FAQ loop (stretch).

---

## 11. Key design decisions and the defensible "why" (decision log)
- **Azure OpenAI (Foundry), not Anthropic** - we are Azure-native; a provider
  shim keeps a swap cheap.
- **pgvector, not Azure AI Search** - AI Search bills a fixed monthly cost even
  idle; with hundreds of mostly-idle labs that breaks cost-optimization. pgvector
  gives vectors + relational data in one store. Clean answer to "why not AI
  Search."
- **One stateless engine, agents as config** - 1000 labs as rows, not 1000
  deployments. Cheap to scale, no context bleed (isolation by `agent_id`).
- **Two-signal confidence gate** - structurally hard to be confidently wrong.
- **Async logging via an event bus** - logging never slows an answer; swap-ready
  for Service Bus.
- **Production-shaped, demo-pragmatic** - we designed the full Azure stack
  (Service Bus, APIM, Content Safety, ACA Jobs) and documented it, but for the
  demo we use in-process equivalents so the live demo is robust. We can speak to
  the production version without depending on it.
- **No em-dashes** - house writing rule.

---

## 12. Jury Q&A prep (likely sharp questions, crisp answers)
- *"Isn't this just GPT in an iframe?"* The moat is the per-lab grounding pipeline
  + the confidence-gated escalation + the feedback loop that improves 1000 labs.
  The model is one component; the system is the product.
- *"How do you avoid hallucinated/confident-wrong answers?"* Strict grounding
  priority (guide first), citations, and a two-signal confidence gate (retrieval
  coverage + an independent model self-check). Below threshold we escalate with a
  structured ticket instead of guessing.
- *"How does this scale to 1000 labs / what does it cost?"* Agents are config, not
  deployments: one stateless engine, per-lab context by id, idle labs are rows.
  Cost levers: semantic cache, prompt caching, model tiering, scale-to-zero.
- *"Why pgvector not a vector DB / AI Search?"* One store for vectors + relational
  data; no fixed idle cost; right-sized for our scale.
- *"What about prompt injection? Guides are external content."* Retrieved text is
  wrapped in a delimited context block the system prompt declares untrusted; the
  model is told never to follow instructions inside it. Production adds Content
  Safety / Prompt Shields.
- *"Is the feedback loop RLHF?"* No - it is human-readable, step-level evidence of
  confusion with suggested fixes. No weight updates. Arguably more actionable.
- *"What is the second bird worth?"* Every cohort makes the lab better for the
  next cohort. Across 1000 labs, a self-improving content library.

---

## 13. Glossary
- **ODL ID** - unique lab identifier (bare integer).
- **TO (Technical Owner)** - the lab owner; receives the improvement digest.
- **masterdoc** - the JSON that lists a lab guide's ordered `.md` files.
- **agent / agent_id** - one Polaris instance bound to one lab; isolation key.
- **grounding** - answering from retrieved source text, not model memory.
- **confidence gate** - the decision to answer vs escalate.
- **escalation payload** - the structured pre-triaged support ticket.
- **digest / second bird** - the per-lab improvement report (the differentiator).
- **pgvector** - the Postgres extension for vector similarity search.
- **HNSW** - the approximate-nearest-neighbour index used for vector search.
- **SSE** - Server-Sent Events; how the chat streams tokens.
- **Cosmos / Atlas / Polaris** - the AI family: Cosmos builds labs, Atlas maps
  cost, Polaris guides the learner.

---

## 14. Where to look in the code (fast map)
- Boot + lifespan + health: `backend/app/main.py`
- Settings: `backend/app/config.py`
- API contracts: `backend/app/schemas.py`
- Control plane: `backend/app/routers/agents.py`
- Chat (SSE): `backend/app/routers/chat.py`
- Model calls: `backend/app/services/llm.py`
- Build pipeline: `backend/app/services/ingest.py`
- Chunking: `backend/app/services/chunker.py`
- Retrieval + gate: `backend/app/services/retrieve.py`
- Agent store: `backend/app/services/agent_store.py`
- Async logging: `backend/app/services/eventbus.py`
- Digest: `backend/app/services/digest.py`
- Persistence + schema: `backend/app/db.py`, `backend/app/schema.sql`
- Shared ODL contract + resolver: `shared/contracts/`
- Mock ODL API: `mocks/odl-api/`
- Engine design (the canonical spec): `docs/engine-design.md`
- Project source of truth: `context.md`

---

## 15. Decisions awaiting you (team lead)
1. **Next backend priority**: per-launch session tokens (security) vs eval harness
   (defensibility) vs Azure deploy (demo-readiness). Recommendation: run the full
   stack end-to-end first, then session tokens, then eval harness.
2. **Demo environment**: local Docker (safe, free) vs deployed Azure (impressive,
   costs credit + network risk). Recommendation: rehearse on local, decide on
   Azure deploy only if it is rock-solid by ~Day 12.
3. **Ticketing**: keep payload + email for the demo, or wire a real test instance.
   Recommendation: payload + email; mention the real connector as a roadmap item.
4. **Ownership going forward**: you and Girish both touched the backend. Decide who
   owns which modules so we do not collide again (this is what caused the parallel
   backends). Suggestion: Girish owns ingest/llm/eventbus/digest; you own the gate
   + agent_store + API contracts; both review each other's PRs.
```
