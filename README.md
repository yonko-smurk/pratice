# TSMS — Final Year Presentation
## Part 2: Code-Review Script (7 minutes)

> **Goal:** Walk the examiner through the *engineering* of the system — code structure, the algorithms that matter, the storage backend, the architecture, and the deployment / testing / coverage story — without slipping into another demo. Every claim below is grounded in an exact file path and (where it matters) a line number, so you can pull the file up live if pressed.

---

## Time Budget (7:00 total)

| # | Section | Time | Cumulative |
|---|---|---|---|
| 1 | Code structure & key components | 1:00 | 1:00 |
| 2 | Frameworks, storage backend, architecture | 1:15 | 2:15 |
| 3 | Critical algorithm 1 — RAG + confidence gating | 0:45 | 3:00 |
| 4 | Critical algorithm 2 — agent ranking (5-signal scorer) | 0:45 | 3:45 |
| 5 | Critical algorithm 3 — ticket-offer routing & expiry | 0:30 | 4:15 |
| 6 | Real-time stack — Channels, signals, presence | 0:45 | 5:00 |
| 7 | Testing — 535 tests, pytest-django, Playwright E2E | 0:45 | 5:45 |
| 8 | CI/CD, Docker, Render deployment, feature flags | 0:45 | 6:30 |
| 9 | Code quality, Ruff, coverage 78%, wrap | 0:30 | 7:00 |

> **Window setup for this section:** VS Code in full-screen on the projector, `cmd-P` ready. Have these files pre-pinned in tabs in this order so you can switch instantly:
> 1. `ai_faq/services.py` (RAG + confidence)
> 2. `community/recommender.py` (agent scorer + ops engine)
> 3. `tickets/routing.py` (offer routing)
> 4. `chat/consumers.py` (WebSocket)
> 5. `pyproject.toml` (Ruff + coverage config)
> 6. `.github/workflows/ci.yml` (CI pipeline)
> 7. `htmlcov/index.html` open in browser (coverage report)

---

## 1 · Code Structure & Key Components  *(1:00)*

**Open VS Code Explorer panel.**

**Say:**
> "TSMS is a 12-app Django monolith. Each app owns one bounded context, and the file convention is the same throughout: `views.py` is HTTP only, `services.py` holds business logic and the LLM calls, `models.py` is the schema plus minimal helpers, and signals fire side-effects so the views stay thin."

Walk the explorer once, point at each app:

| App | Purpose | Headline file |
|---|---|---|
| `accounts` | Custom `User`, three roles, role decorators | `accounts/decorators.py` |
| `tickets` | Ticket lifecycle, SLA, audit log, offer model | `tickets/routing.py` |
| `agent` | Agent profiles, skills, presence middleware | `agent/middleware.py` |
| `assistant` | AI Wizard + RAG chat + intent router | `assistant/router.py` |
| `ai_triage` | LLM classification (category + priority + skills) | `ai_triage/services.py` |
| `ai_faq` | RAG pipeline + SSE streaming | `ai_faq/services.py` |
| `kb` | Knowledge-base articles + pgvector chunks | `kb/models.py` |
| `chat` | Channels consumer for ticket chat + presence | `chat/consumers.py` |
| `community` | Q&A, reputation, recommender engine | `community/recommender.py` |
| `analytics` | Live KPI dashboard + WebSocket consumer | `analytics/consumers.py` |
| `ops` | Admin ops dashboard + AI recommendations panel | `ops/views.py` |
| `core` | Feature flags, health endpoint | `core/flags.py` |

**Say:**
> "The design decision behind splitting into 12 apps rather than one giant app is testability — each app has its own `tests/` folder and can be tested in isolation. And because all cross-app dependencies go through service functions (not direct ORM queries from another app's view), I could swap the LLM provider from Ollama to Groq with a single env var change."

---

## 2 · Frameworks, Storage Backend, Architecture  *(1:15)*

### Frameworks

**Say:**
> "The web framework is **Django 6** on Python 3.12. I went with Django over Flask or FastAPI because the project needed three things out of the box: a real ORM with migrations, a battle-tested admin, and a permissions system. Three roles, role-gated views, role-aware querysets — Django gives me that for free."
>
> "For real-time I added **Channels 4** running on **Daphne** as the ASGI server. WebSockets aren't a bolt-on — the `tickets/<id>/` chat is a `TicketChatConsumer` and the analytics dashboard is an `AnalyticsDashboardConsumer`, both backed by **Redis** as the channel layer."
>
> "The frontend is server-rendered Django templates styled with **Tailwind v3**, plus **Alpine.js** for the small reactive bits — no SPA framework. That keeps the bundle tiny and means a JSON response and an HTML response come from the same view function."

### Storage backend

> "Storage is **PostgreSQL 16** with the **pgvector** extension enabled in `kb/migrations/0002_vector_extension.py`. I chose Postgres over a separate vector DB like Pinecone or Chroma because pgvector lets me keep the embeddings inside the relational schema — every `KbChunk` row has its parent article as a foreign key *and* a 768-dimensional embedding vector on the same row, so a single query gives me both the cosine-ranked chunks and the article metadata."

Open `kb/models.py` line ~25 and point at:
```python
class KbChunk(models.Model):
    article   = models.ForeignKey(KbArticle, related_name="chunks", on_delete=models.CASCADE)
    chunk_index = models.PositiveIntegerField()
    text      = models.TextField()
    embedding = VectorField(dimensions=768)
```
> "768 dimensions because that's what `nomic-embed-text` produces. Redis sits beside Postgres as the Channels message broker and the session cache."

### Architecture

> "The macro shape is a **monolithic ASGI app, three roles, two real-time channels**. There's no microservice split because the system is single-tenant — splitting it would add network hops without buying anything. But the *internal* boundaries are strict: every write that crosses an app boundary goes through a service function, and every cross-cutting concern — audit logging, notifications, reputation recalculation — is wired up via Django signals so the calling code never has to remember to fire them."

Trace one request out loud:
> "When a student types into the assistant chat: `POST /assistant/send-stream/` hits `assistant/views.py::send_message_stream`, which first runs the intent router (`detect_intent`) for cheap structured queries — and for everything else delegates to `ai_faq/services.py::stream_answer_with_context`. That generator embeds the query through `embed_text()` (Ollama's `/api/embed` endpoint), runs a pgvector cosine query, computes a confidence score, calls the LLM, and yields **Server-Sent Events** chunk by chunk wrapped in a `StreamingHttpResponse(content_type='text/event-stream')`. The non-streaming sibling endpoint `POST /ai/answer/` (in `ai_faq/views.py`) uses the same retrieval pipeline via `answer_with_context` but returns a single JSON blob — that's what the legacy HTMX widgets call. Same retrieval, two response shapes."

---

## 3 · Critical Algorithm 1 — RAG + Confidence Gating  *(0:45)*

**Open `ai_faq/services.py`.**

**Say:**
> "This is the core retrieval function. Two things make it non-trivial: the cosine query and the confidence formula."

Show the retrieval (around line 192):
```python
qvec = embed_text(query)
qs = (
    KbChunk.objects
    .annotate(distance=CosineDistance("embedding", qvec))
    .order_by("distance")[:top_k]
)
sims = [1.0 - r.distance for r in qs]
```
> "pgvector exposes `CosineDistance` as a Django ORM annotation, so I can sort by semantic similarity in the database — no Python-side ranking, no fetching all chunks into memory."

Show the confidence calculation (around line 336):
```python
if sims:
    top3 = sims[:3] if len(sims) >= 3 else sims
    conf = 0.5 * max(sims) + 0.5 * (sum(top3) / len(top3))
else:
    conf = 0.0
```
> "The confidence isn't just `max(similarity)`. It's a 50/50 blend of the **best** match and the **average of the top three**. Why? Because a single high-scoring chunk in a sea of low ones is unreliable — it could be a one-off keyword collision. A high *average* across the top-three means several semantically related chunks all agree, which is the signal I actually trust. Below 0.6 the assistant escalates to community fallback and then to a ticket — the integrated pipeline I show in Part 1."

> "And if Ollama isn't reachable, `embed_text` returns a zero vector and `_search_chunks` falls back to lexical `icontains` over `KbChunk.text`. Same view, same response shape, same SSE stream — the swap is invisible to the frontend. That's how the Render deployment works without a GPU."

---

## 4 · Critical Algorithm 2 — Agent Ranking  *(0:45)*

**Open `community/recommender.py` line ~46.**

**Say:**
> "This is `recommend_agents` — the function that powers both the 'Top 3 Recommended Agents' card on every ticket detail page and the AI Recommendations panel on the ops dashboard. It produces a 100-point score per agent across **five weighted signals**:"

Show the structure:
```python
def recommend_agents(ticket, limit=5):
    candidates = []
    for agent in eligible_agents:
        skill_score    = _score_skill_match(agent, ticket)        # 0–30
        category_score = _score_category_expertise(agent, ticket) # 0–25
        perf_score     = _score_performance(agent)                # 0–20
        comm_score     = _score_community_expertise(agent)        # 0–15
        avail_score    = _score_availability(agent)               # 0–10 (can go negative)
        total = skill_score + category_score + perf_score + comm_score + avail_score
        candidates.append({
            "agent": agent, "score": total,
            "subscores": {...}, "reasons": [...],
        })
    return sorted(candidates, key=lambda c: -c["score"])[:limit]
```

> "Three things to call out. **First**, the weights are not arbitrary — skill match dominates because the worst outcome is sending a printer ticket to a network specialist; community expertise gets only 15 points because a great forum contributor isn't necessarily fast at tickets. **Second**, availability can go *negative* — if an agent is in focus mode or already at max capacity, the function actively penalises them rather than just zeroing them out. **Third**, every score carries a `reasons` list — strings like `'Matched 4/5 skill keywords'` or `'Avg handle time 12 min'` — and those propagate all the way to the UI so the admin can see *why* the recommender suggested this agent. That transparency was a deliberate design choice; black-box recommenders are a hard sell in a support-floor context."

> "The category-expertise sub-score uses `log2(resolved_in_category)` rather than raw count, because an agent who's resolved 100 email tickets isn't ten times better than one who's resolved 10 — the marginal expertise tapers off."

---

## 5 · Critical Algorithm 3 — Ticket-Offer Routing & Expiry  *(0:30)*

**Open `tickets/routing.py` around line 113.**

**Say:**
> "Recommendation is one thing, *delivery* is another. When a new ticket is created, a `post_save` signal fires `offer_next_agent`, which is the function that takes a ranked agent list and turns it into a real time-boxed offer."

Show the core:
```python
# 1. Mark stale offers as expired
TicketOffer.objects.filter(
    ticket=ticket, status=PENDING, expires_at__lt=now,
).update(status=EXPIRED, responded_at=now)

# 2. Rank candidates, excluding agents already offered this ticket
already_offered = set(TicketOffer.objects.filter(ticket=ticket).values_list("agent_id", flat=True))
ranked = _rank_agents(ticket, exclude_agent_ids=already_offered)

# 3. Create a new offer for the top-ranked agent with a 45-second TTL
TicketOffer.objects.create(
    ticket=ticket, agent=ranked[0]["agent"],
    expires_at=now + timedelta(seconds=45),
    round_number=TicketOffer.objects.filter(ticket=ticket).count() + 1,
)
```

> "Three subtleties. The **`round_number`** lets me reconstruct the offer history per ticket — useful for analytics and for debugging routing fairness. **`exclude_agent_ids`** stops the same agent being offered the same ticket twice. And the **45-second expiry** means a busy agent who never opens their dashboard can't block a ticket — it auto-cycles to the next-best agent. That last one is the difference between a routing system that *works* and one that silently jams."

> "`_rank_agents` itself is a leaner version of the 100-point scorer — it computes `(skill_score * 10) - load` because for *routing* (as opposed to recommendation) the only signals that matter in the moment are 'do they know the topic' and 'are they free right now'."

---

## 6 · Real-Time Stack — Channels, Signals, Presence  *(0:45)*

**Open `chat/consumers.py`.**

**Say:**
> "There are two WebSocket channels in the system. Both inherit from `AsyncWebsocketConsumer`, both join a Redis-backed channel group, and both use `@database_sync_to_async` to bridge to the synchronous Django ORM."

Show the presence trick (line ~14):
```python
class TicketChatConsumer(AsyncWebsocketConsumer):
    _room_users: dict[str, set[int]] = {}  # class-level, per-process

    async def connect(self):
        ticket_id = self.scope["url_route"]["kwargs"]["ticket_id"]
        self.group = f"ticket_{ticket_id}"
        await self.channel_layer.group_add(self.group, self.channel_name)
        self._room_users.setdefault(self.group, set()).add(self.scope["user"].id)
        await self.channel_layer.group_send(self.group, {
            "type": "presence_update",
            "online_user_ids": list(self._room_users[self.group]),
        })
```

> "Presence is a class-level dict keyed by group name, mutated on `connect` and `disconnect`. When either fires, every other socket in the group receives a `presence_update` event and the green dot in the UI flips. No polling, no heartbeat — the WebSocket lifecycle *is* the heartbeat."

Switch to `analytics/signals.py`:
> "The other half of real-time is signals. Whenever a `Ticket` is saved, a `post_save` handler in `analytics/signals.py` does two things: writes a row to `TicketEventLog` for the audit trail, and broadcasts a JSON payload to the `analytics_dashboard` group so every admin watching `/analytics/` sees the KPI tick up *in the same render frame*. That's the demo moment in Part 1 where the counter increments without a refresh."

> "Custom middleware in `agent/middleware.py` updates `last_seen_at` on every authenticated request — that's how the ops dashboard knows which agents are 'active' versus 'idle'."

---

## 7 · Testing — 535 Tests, pytest-django, Playwright  *(0:45)*

**Open the side terminal:** `pytest --collect-only -q 2>&1 | tail -3`

> **535 tests collected, 8 deselected** (the deselected ones are the Playwright E2E suite, gated behind the `e2e` marker so they don't run on every commit).

**Say:**
> "Testing splits into three layers."

> **Layer 1 — unit tests** in every app's `tests/` folder. Pure functions, mocked LLM calls. The `_score_*` functions in the recommender, the confidence formula, the intent router regexes — all unit-tested in isolation. `assistant/tests/test_router.py` alone covers every branch of `detect_intent`."

> **Layer 2 — integration tests** that hit the real database, real signals, real channel layer. Example: `tickets/tests/test_routing.py` creates a real ticket, asserts that `TicketOffer` rows materialise with the correct `round_number`, and asserts that calling `accept` on one offer marks the others as `SUPERSEDED`. The `broadcast_signals` pytest marker keeps the analytics signals connected for the cases where I actually want to assert the side-effect."

> **Layer 3 — end-to-end** with **Playwright** in `e2e/`. The conftest spins up Daphne in a subprocess on a random port, opens a real Chromium browser, logs in as a student, fills the wizard, submits a ticket, switches to an agent context, accepts the offer, sends a chat message, and asserts the student sees it. That's a real WebSocket round-trip across two browser contexts — the closest thing to an actual demo, run automatically."

Show `pytest.ini`:
```ini
DJANGO_SETTINGS_MODULE = tsms.settings
asyncio_mode = auto
addopts = -m "not e2e"
markers =
    e2e: Playwright end-to-end tests
    real_channel_layer: opt-in to live Redis channel layer
    broadcast_signals: keep analytics signals connected for this test
```

> "The two custom markers exist because Channels and signal-driven side-effects make tests slow and flaky if they're on by default. Opt-in markers mean the fast suite stays fast — the full 535-test run completes in under a minute on my laptop."

---

## 8 · CI/CD, Docker, Render, Feature Flags  *(0:45)*

### CI

**Open `.github/workflows/ci.yml`.**

> "CI runs on **GitHub Actions** on every push and every PR. Two stages."

> **Stage 1 — Ruff lint** (~30 seconds): catches unused imports, undefined names, common bugs from the `bugbear` ruleset, and import ordering.

> **Stage 2 — pytest with coverage** (~3 minutes): spins up real Postgres-16-with-pgvector and Redis-7 service containers, sets `USE_FAKE_EMBEDDINGS=1` and `AI_FAQ_OFFLINE=1` so the LLM calls are stubbed (CI doesn't have a GPU and shouldn't pay Groq tokens), runs the full 535-test suite, uploads `coverage.xml` and an HTML report as a build artifact."

> "If lint or tests fail, the build is red and I can't merge."

### Docker

**Open `Dockerfile`.**

> "**Two-stage build**. Stage 1 is a fat builder image that compiles the native deps — `psycopg2`, `pgvector` C extension, `Pillow`. Stage 2 copies just the wheel cache into a slim Python 3.12 image and runs as a non-root user. Final image is around 200 MB."

**Open `docker-compose.yml`.**

> "Compose orchestrates four services for local dev: `pgvector/pg16`, `redis:7-alpine`, the Django app, and a one-shot seed container. There's an `extra_host` entry mapping `host.docker.internal` so the container can reach the Ollama daemon running on the host Mac."

**Open `docker-entrypoint.sh`:**

> "Entry script waits for Postgres TCP, runs `migrate`, runs `collectstatic`, optionally re-seeds if `SEED_ON_START=1`, then exec's Daphne. Idempotent — restart the container as many times as you like."

### Render deployment

> "Production is on **Render**. Three managed pieces: Render web service running `daphne -b 0.0.0.0 -p $PORT tsms.asgi:application`, Render-managed PostgreSQL with the pgvector extension installed, and **Upstash Redis** over TLS for the channel layer. The LLM calls switch from local Ollama to **Groq's `llama-3.1-8b-instant`** by setting `LLM_PROVIDER=groq` and `GROQ_API_KEY` — no application code change."

### Feature flags

**Open `core/flags.py`.**

> "The graceful-degradation story is anchored by a tiny feature-flag module. `is_ai_auto_assign()`, `is_ai_recommend_mode()`, `get_ai_assignment_mode()` — three env-driven booleans that let me dial AI behaviour from `off` to `recommend` to `auto` without redeploying. Combined with `USE_FAKE_EMBEDDINGS=1` and `AI_FAQ_OFFLINE=1`, the same code base runs in four distinct modes — full local AI, recommend-only, embeddings-disabled lexical, and fully offline — all gated by env vars."

---

## 9 · Code Quality, Coverage 78%, Wrap  *(0:30)*

**Open `pyproject.toml`.**

> "Code quality is enforced by **Ruff** with the `E, F, W, I, B, UP` rule families enabled — that covers PEP-8 errors, pyflakes, import ordering, and the `bugbear` set of common Python footguns. `E501` (line length) is intentionally relaxed because LLM prompt strings are easier to read on one line."

**Open `htmlcov/index.html` in the browser.**

> "Coverage is measured by `coverage.py` with **branch coverage** turned on — meaning a line that contains an `if/else` only counts as fully covered if both branches were executed. The headline number is **78 %** across roughly 5 000 statements, with the high-value modules — `accounts`, `tickets/routing`, `ai_faq/services`, `community/recommender` — well above 90 %. The lowest-covered files are the LLM streaming helpers, where the missing lines are real-network error paths that I deliberately don't exercise in CI."

**Closing line:**
> "So to wrap: 12 Django apps, 5 000 lines of Python at 78 % branch coverage across 535 tests, three AI subsystems wired together by a shared confidence-gating pipeline, two WebSocket channels backed by Redis, deployed identically by Docker locally and by Render in the cloud, with the same code base happily switching between local Ollama and cloud Groq via a single env var. That's the engineering story."

---

## Quick-Reference Cheat Sheet

| Topic | One-liner | Source of truth |
|---|---|---|
| Web framework | Django 6 + DRF | `requirements.txt` |
| Async stack | Channels 4 + Daphne | `tsms/asgi.py` |
| Database | PostgreSQL 16 + pgvector (768-dim) | `kb/migrations/0002_vector_extension.py` |
| Cache / broker | Redis 7 (channel layer + sessions) | `tsms/settings.py` |
| LLM (local) | Ollama `llama3.1:8b` + `nomic-embed-text` | `ai_faq/services.py::_call_llm` |
| LLM (cloud) | Groq `llama-3.1-8b-instant` | env `LLM_PROVIDER=groq` |
| Confidence formula | `0.5 × max + 0.5 × avg(top-3)` | `ai_faq/services.py:~336` |
| Agent scorer | 30/25/20/15/10 weights, max 100 | `community/recommender.py:46` |
| Offer TTL | 45 seconds, round-tracked | `tickets/routing.py:113` |
| WebSockets | `/ws/tickets/<id>/`, `/ws/analytics/` | `chat/consumers.py`, `analytics/consumers.py` |
| Tests | 535 collected; 8 E2E (Playwright) | `pytest --collect-only` |
| CI | GitHub Actions: Ruff → pytest → coverage | `.github/workflows/ci.yml` |
| Lint | Ruff (E,F,W,I,B,UP), Python 3.12 | `pyproject.toml` |
| Coverage | **78 %** branch coverage, ~5 000 stmts | `htmlcov/index.html` |
| Production | Render + Upstash Redis + Groq | README §Deployment |
| Feature flags | env-driven, four runtime modes | `core/flags.py` |

---

## Likely Examiner Follow-Ups (Pre-Loaded Answers)

**Q: "Why pgvector and not Pinecone or Chroma?"**
> "Single source of truth. With pgvector my embeddings live in the same Postgres row as the article metadata, so one ORM query gives me both — no two-system consistency problem, no extra service to deploy. Pinecone would have been faster at very large scale, but at 11 articles and ~30 chunks the cosine query returns in single-digit milliseconds and I get transactional guarantees for free."

**Q: "Why Channels and not just polling?"**
> "Latency and battery. Polling at 1 Hz gives you a one-second lag on chat and burns the agent's CPU sitting on the queue page. Channels gives me sub-100 ms message delivery, presence updates that fire on disconnect, and the analytics dashboard pushes only when something *actually* changes — so an idle dashboard does zero work."

**Q: "Why a monolith?"**
> "Operational simplicity for a single-tenant system. Splitting `tickets`, `chat`, `assistant`, `kb` into separate services would add four deployment targets and three network hops on the critical path of a single chat message — for zero scaling benefit at this user count. The internal boundaries are still strict — every cross-app call goes through a service function — so if the system ever needs to scale out, the seams already exist."

**Q: "What happens if Ollama dies in production?"**
> "Two layers of fallback. First, `LLM_PROVIDER=groq` swaps inference to Groq Cloud automatically. Second, `USE_FAKE_EMBEDDINGS=1` swaps the retrieval step to PostgreSQL `icontains` lexical search. The view layer doesn't know about either swap — both are gated in `ai_faq/services.py` behind `if settings.USE_FAKE_EMBEDDINGS` and `if settings.LLM_PROVIDER == 'groq'`."

**Q: "Why 78 % and not 100 %?"**
> "The missing 22 % is mostly two things: real-network LLM error branches that would require either a live Ollama instance or extensive `responses` mocking to exercise, and template rendering branches in admin views that the Playwright suite covers but `coverage.py` only sees if I lower the bar to template-line coverage. Branch coverage at 78 % means every meaningful business-logic branch in services and routing is exercised — I optimised for *meaningful* coverage rather than line-count theatre."

**Q: "What's the single most complex piece of code in here?"**
> "`community/recommender.py::recommend_agents`. It's 200 lines, five sub-scorers, two database query passes, and produces both a numeric ranking *and* a UI-ready reasons array. It feeds the ticket detail page, the AI Recommendations panel on the ops dashboard, *and* the offer-routing function — three consumers, one source of truth. If anything in the system would survive a rewrite to microservices, that's the function I'd extract first."
