# TSMS — Final Year Presentation
## Part 1: Full End-to-End Demo Script (14 minutes)

> **Goal:** Demonstrate **every** completed piece of functionality across all three roles (Student, Agent, Admin), all five AI subsystems, real-time WebSocket features, the community module, analytics, and the local-first / cloud-fallback architecture — without leaving anything out.

---

## 0. Pre-Demo Setup Checklist (do BEFORE you go on)

Run these the morning of the demo and leave them running:

```bash
# Terminal 1 — Ollama (so RAG uses real embeddings, not lexical fallback)
ollama serve
ollama list   # confirm llama3.1:8b and nomic-embed-text are pulled

# Terminal 2 — Redis
redis-server

# Terminal 3 — Postgres (or Docker)
brew services start postgresql@16

# Terminal 4 — App (ASGI so WebSockets work)
source .venv/bin/activate
python manage.py migrate
python _seed_demo.py            # users, tickets, KB articles, embeddings
python _check_kb.py             # confirm embeddings exist
daphne -b 0.0.0.0 -p 8000 tsms.asgi:application
```

**Browser windows to pre-open in separate profiles / windows** (so you don't waste demo time logging in):

| Window | Profile / Browser | Logged in as | Tab(s) open |
|---|---|---|---|
| **A** | Chrome (Student) | `student@example.com` | `/tickets/my/` |
| **B** | Chrome Incognito (Agent) | `agent@example.com` | `/tickets/queue/` and `/agent/` |
| **C** | Firefox (Admin) | `admin@example.com` | `/ops/dashboard/` and `/analytics/` |
| **D** | Chrome Incognito 2 (Live cloud) | logged out | `https://tud-tsms.onrender.com` |

Have a **fresh local PDF receipt** of the architecture diagram on a side monitor in case the projector shows the small UI.

> **Backup:** if Ollama dies mid-demo, set `USE_FAKE_EMBEDDINGS=1` in `.env` and restart Daphne — RAG will fall back to lexical search and the demo continues. Mention this on camera, it's a feature, not a bug.

---

## Time Budget (14:00 total)

| # | Section | Window | Time | Cumulative |
|---|---|---|---|---|
| 1 | Landing + auth + roles | A/B/C | 0:45 | 0:45 |
| 2 | Student — AI Wizard → ticket creation | A | 1:45 | 2:30 |
| 3 | Student — RAG Assistant (streaming, KB citations) | A | 1:30 | 4:00 |
| 4 | Student — Community Q&A (ask, vote, accept, escalate) | A | 1:30 | 5:30 |
| 5 | Agent — Queue, claim, skill routing, recommended steps | B | 2:00 | 7:30 |
| 6 | Real-time chat (Student ↔ Agent) + presence + notifications | A+B | 1:30 | 9:00 |
| 7 | Agent — Profile, skills, availability, agent analytics | B | 1:00 | 10:00 |
| 8 | Admin — Ops dashboard, focus mode, broadcast, triage panel | C | 1:30 | 11:30 |
| 9 | Admin — Analytics dashboard (live KPI WebSocket feed) | C | 1:00 | 12:30 |
| 10 | Admin — KB CRUD + embedding regeneration + categories/SLA | C | 0:45 | 13:15 |
| 11 | Local-first vs cloud fallback (live Render demo) | D | 0:30 | 13:45 |
| 12 | Wrap: feature flags, audit log, health endpoint | C | 0:15 | 14:00 |

---

## 1 · Landing, Auth & Roles  *(0:45)*

**Window C (Admin):** Open `https://tud-tsms.onrender.com` (or `localhost:8000`).

**Say:**
> "TSMS is a multi-role support platform built on Django 6 ASGI, Channels, PostgreSQL with pgvector, Redis, and either local Ollama or Groq for the LLM. Three roles — Student, Agent, Admin — each with role-gated views enforced by custom decorators."

- Show the **landing page**.
- Switch to Window A — show the **Student dashboard** at `/tickets/my/`.
- Switch to Window B — show the **Agent queue** at `/tickets/queue/`.
- Switch to Window C — show the **Ops dashboard** at `/ops/dashboard/`.

> "Same code base, three completely different UIs, gated by `@role_required` decorators in `accounts/decorators.py`."

---

## 2 · Student: AI Wizard → Ticket Creation  *(1:45)*

**Window A.** Navigate to `/assistant/wizard/`.

**Say:**
> "Instead of dumping a vague ticket on an agent, students go through a multi-step AI Wizard — `assistant/wizard_views.py` — that uses the LLM to refine the description and surface relevant KB articles before submission."

1. Click **Start Wizard**.
2. Type a deliberately vague problem: **"I can't get into my email"**.
3. Click **Next** — the LLM returns clarifying questions. Answer one (e.g., "Outlook on the web, password not accepted").
4. Step through to the **Diagnose** view — show the AI-suggested category, priority, and the KB articles it pulled (Office 365 setup, password reset).
5. On the **Result** page, point at the **Escalate to Ticket** button.
6. Click **Escalate** — it pre-fills `/tickets/new/` with the refined description, suggested priority, suggested category.
7. Attach a file (drag any small PDF). Submit.
8. Land on `/tickets/<id>/`. Point out:
   - **AI Triage banner** — category + priority + confidence score (`ai_triage/services.py`).
   - **Recommended Agents (Top 3)** — names + match scores from `community/recommender.py` (40% skill / 25% workload / 20% resolution rate / 15% AHT).
   - **Similar historical tickets** panel.
   - **SLA countdown timer**.

> "One submission, four AI subsystems already running: wizard refinement, triage classifier, agent matching, and KB retrieval."

---

## 3 · Student: AI Real-Time Chat — The Integrated KB → Community → Escalate Pipeline *(2:00)*

> **This is the most important section of the demo. It is where every piece of the platform — RAG, the KB, pgvector, the community module, the ticket system, and the action-confirmation framework — is shown working as one integrated flow.**

**Window A.** Open `/assistant/` (chat page).

**Say (the integration narrative — say this out loud, slowly):**
> "This isn't just a chatbot. The assistant is wired into three other subsystems and chooses between them based on confidence. Every user message walks the same three-step pipeline in `assistant/views.py`:
>
> **Step 1 — Knowledge Base RAG.** The query is embedded locally with `nomic-embed-text`, pgvector returns the top-k cosine-similar chunks from the `KBArticle` table, and `llama3.1:8b` generates a grounded answer with a confidence score and snippet citations. The streamed response comes back over Server-Sent Events.
>
> **Step 2 — Community fallback.** If KB confidence is below 0.8, the assistant calls `community.search.find_resolved_answer` to look for a previously *accepted* answer to a semantically similar community question. If one exists, the assistant inlines it — author, vote score, link. If no resolved answer exists, it falls back further to `search_similar_questions` and shows the top three similar open questions.
>
> **Step 3 — Dual escalation.** If the community has nothing relevant either, the assistant offers the user **two pre-built action requests** in one message: *Ask the Community* (creates a community question) **or** *Create a Support Ticket* (escalates to an agent). Both are queued as `AssistantActionRequest` rows pending one-click confirmation.
>
> So the same chat surface gracefully degrades: pgvector RAG → community search → human escalation. And every write action — ticket creation, community post, ticket assignment, status change — funnels through the same confirm/cancel framework, so the user is always in control."

### Demo the three branches in order — pick questions that hit each fallback

**Branch A — High-confidence KB hit (clean RAG path):**
1. New conversation → ask: **"How do I set up Office 365 on my phone?"**
2. Watch the response stream token-by-token.
3. Point at the **KB confidence score** and the **snippet pills** below the answer — those come straight from `kb_snippets` in the message payload.
4. Ask a follow-up: **"What about app passwords?"** — context is preserved across turns.

**Branch B — Low-confidence KB → community resolved-answer fallback:**
1. New conversation → ask something the KB doesn't fully cover but a community user has answered, e.g. **"Why is my student email rejecting attachments over 25MB?"**
2. The KB answer streams in first (low confidence).
3. Below it, a **second assistant message** appears: *"I found a community answer that might help"* — with the question title, accepted-answer preview, author name, vote score, and a link to `/community/<id>/`.
4. Click through to the community question to prove the link works.

**Branch C — Nothing found → dual-escalation offer:**
1. New conversation → ask something genuinely novel: **"How do I export my historical timetable as iCal for the past three years?"**
2. KB answer is low-confidence, no resolved community answer, no similar questions.
3. The assistant posts the **escalation card** with **two buttons side by side**: *Ask the Community* and *Create a Support Ticket*.
4. Click **Ask the Community** → confirmation screen → confirm → switch to `/community/` and show the question is now there with the user as author.
5. Re-trigger the flow with another novel query and this time click **Create a Support Ticket** → confirm → land on the new ticket detail page with the original question, KB confidence, and KB snippets all baked into the description for the agent.

**Branch D — Direct command routing (the action layer):**
1. Type **"show my open tickets"** → assistant returns a structured `tickets_list` card pulled from `assistant/tools.py::list_my_open_tickets`.
2. Type **"create ticket: VPN drops every 10 minutes"** → assistant returns a pending action; click **Confirm** to create.
3. Type **"status of ticket #5"** → assistant returns ticket status inline.
4. As an agent (Window B briefly): **"assign ticket #5"** → pending action → confirm → ticket reassigned.

**Wrap the section with:**
> "One chat box, four behaviours: RAG retrieval, community lookup, dual-channel escalation, and structured tool calls — all stitched together by an intent router and a single action-confirmation framework. And if Ollama goes down on the Render deployment, `USE_FAKE_EMBEDDINGS=1` swaps the embedding step for PostgreSQL trigram search and Groq takes over inference — without a single line of view-layer code changing."

---

## 4 · Student: Community Q&A  *(1:30)*

**Window A.** Navigate to `/community/`.

**Say:**
> "Not every question deserves a ticket. The community module (`community/`) lets students help each other — Stack-Overflow style — with reputation, badges, and one-click escalation if no answer arrives."

1. Show the **question list** with vote counts and accepted-answer badges.
2. Click **Ask** → post: **"Best way to reset my student portal password?"**
3. Open the question detail. Switch to a second tab as the same user (or quickly to Window B as the agent) and **post an answer**.
4. Back in Window A, **upvote** the answer (live counter update).
5. Click **Accept Answer** — reputation awarded (`community/reputation.py`), badge appears.
6. Click **Escalate to Ticket** on a different unanswered question — show it converts into a ticket pre-populated with the question body.
7. Click **Leaderboard** (`/community/leaderboard/`) — top contributors by reputation.
8. Click **My Activity** — personal contribution history.

---

## 5 · Agent: Queue, Claim, Skill Routing, Recommended Steps  *(2:00)*

**Window B.** Navigate to `/tickets/queue/`.

**Say:**
> "Agents land on a unified queue with priority filters, skill-tag routing, search, and a live WebSocket-fed offer system."

1. Show the queue — point out priority colour bars, SLA timers, skill tags on each row.
2. Apply a **priority filter** (High) and a **search**.
3. Open one of the **AI-recommended offers** at the top — these come from skill-tag matching against the agent's profile.
4. Click **Accept Offer** — ticket transitions to the agent (`/tickets/offers/<id>/accept/`).
5. Open a different unclaimed ticket → click **Claim** → status flips to In Progress, the audit log row appears.
6. On the ticket detail page, point out:
   - **AI-Recommended Resolution Steps** (generated by `assistant/diagnosis.py`).
   - **Internal Note** field (only visible to agents) → add a quick note.
   - **Reassign** dropdown — assign to another agent.
   - **Set Priority / Set Category / Set Status** controls.
7. Drop the status to **Resolved** — show the SLA timer freezing and a CSAT request being sent to the customer (`analytics/submit_csat`).

---

## 6 · Real-time Chat + Presence + Notifications  *(1:30)*

**Side-by-side windows A (Student) and B (Agent), same ticket.**

**Say:**
> "Real-time chat over Django Channels — `chat/consumers.py::TicketChatConsumer` on the path `/ws/tickets/<id>/`. Presence is tracked in the channel layer."

1. In Window B, open the ticket. In Window A, open the same ticket.
2. Point at the **green presence dot** showing the other party is online.
3. Type a message in B — it appears instantly in A.
4. Reply in A — appears instantly in B.
5. Show a **typing indicator** if implemented (otherwise skip).
6. In Window A, navigate away from the ticket — show that the **notification badge** updates in the top nav (ticket update notification consumer).
7. Switch back — the notification clears.

> "Two WebSocket channels in production: ticket chat + analytics live feed. Both backed by Redis as the channel layer."

---

## 7 · Agent: Profile, Skills, Availability, Agent Analytics  *(1:00)*

**Window B.** Navigate to `/agent/profile/`.

1. Show the **skill tags** the agent has registered (e.g., `email`, `vpn`, `office365`).
2. Toggle **Available / Away** — explain this changes which offers route to them.
3. Add a new skill tag → save.
4. Navigate to `/analytics/agent/` → show **per-agent KPIs**:
   - Resolution rate
   - Average handle time
   - CSAT trend chart
   - Tickets resolved per day chart

> "These charts are wired to `/analytics/api/agent-kpis/` and re-render on the WebSocket analytics feed."

---

## 8 · Admin: Ops Dashboard, Focus Mode, Broadcast, Triage Panel  *(1:30)*

**Window C.** `/ops/dashboard/`.

**Say:**
> "The ops dashboard is the operational nerve centre — `ops/views.py::ops_dashboard` — a single-page view of the entire support floor in real time."

1. Show the **agent roster** with status indicators (Online / Away / Offline).
2. Show **queue depth**, **tickets in progress**, **breached SLAs**, **live triage feed**.
3. Click an agent → enter **Focus Mode** — shows that agent's current ticket, last action, average handle time today.
4. Click **Broadcast** → send an announcement: **"Reminder: Friday standup at 4pm"** → switch to Window B for half a second to show it appearing in the agent's notification bar.
5. Open the **Triage Panel** — show pending tickets with AI-suggested category + priority + confidence. Approve one suggestion with one click.

---

## 9 · Admin: Analytics Dashboard (Live WebSocket Feed)  *(1:00)*

**Window C.** `/analytics/`.

1. Show the top-row KPIs — **open tickets**, **resolved today**, **avg first-response time**, **avg resolution time**, **CSAT score**.
2. Show **ticket volume trend** chart (last 7 days).
3. Show **resolution funnel** (created → claimed → resolved → closed).
4. Show **category heatmap** (which categories spike on which days).
5. Switch to Window A briefly, submit one new ticket — switch back to Window C and show the **counter increment live** without refreshing (`AnalyticsDashboardConsumer` on `/ws/analytics/`).

---

## 10 · Admin: KB CRUD + Embedding Regen + Categories + SLA  *(0:45)*

**Window C.** Django admin at `/admin/` (or the KB management view if you have one).

1. Open a **KBArticle** — show title, body, category, embedding column.
2. Edit one — change a sentence — save → embedding regenerates via signal (`kb/signals.py`).
3. Run `python _check_kb.py` in the side terminal — show all articles have non-null vectors.
4. Navigate to **Categories** in admin — add a new category.
5. Navigate to **SLA Policies** — show the per-priority SLA hours that drive the countdown timers seen on every ticket.

---

## 11 · Local-First vs Cloud Fallback  *(0:30)*

**Window D.** `https://tud-tsms.onrender.com`.

**Say:**
> "Same code, deployed to Render's free tier where Ollama can't run. `USE_FAKE_EMBEDDINGS=1`, so embedding generation is skipped, retrieval falls back to PostgreSQL trigram search, and Groq's `llama-3.1-8b-instant` handles inference."

1. Log in as `student@example.com`.
2. Open `/assistant/` → ask the **same Office 365 question** as in section 3.
3. Response streams back via Groq + lexical retrieval.

> "Zero application code changes between environments — only env vars. Designed for graceful degradation, not failure."

---

## 12 · Wrap-up (15 seconds)

**Window C.**

1. `curl https://tud-tsms.onrender.com/health` (or open in tab) → JSON 200 OK from `core/views.py::healthz`.
2. Open `core/flags.py` in the editor on the side monitor → point at the feature-flag dictionary.
3. Open the **audit log** in admin — show one ticket's full state-transition history.

**Closing line:**
> "That's 12 Django apps, 24 models, 80-plus endpoints, 5 AI subsystems, 2 real-time channels, 530-plus tests, all running locally with Ollama and on Render with Groq — same code, two execution profiles."

---

## Demo Recovery Cheat-Sheet (if something breaks live)

| Symptom | One-line fix |
|---|---|
| RAG response hangs | `pkill -f ollama && ollama serve &` then retry |
| WebSocket not connecting | Confirm Daphne (not `runserver`) is running; check `REDIS_URL` |
| Triage banner missing | `python manage.py shell -c "from ai_triage.services import classify_ticket; classify_ticket(<id>)"` |
| Empty queue | `python _seed_demo.py` |
| Embeddings empty | `python manage.py ingest_kb` then `python _check_kb.py` |
| Cloud demo cold | Hit `/health` 30 sec before you switch to Window D |

---

## What This Demo Covers (so you can confidently say "nothing left out")

**Roles & Auth (3.1–3.3 of README)**
- [x] Student dashboard, register, login, profile, apply-as-agent
- [x] Agent queue, claim, profile, skills, availability
- [x] Admin ops dashboard, focus mode, broadcast, triage approval

**Ticket Lifecycle**
- [x] Create, attach files, list, my-tickets, detail, claim, assign, reassign
- [x] Set priority / category / status, customer close, offer accept/skip
- [x] Internal notes, attachments download, audit log, SLA timers

**AI Subsystems (4.1–4.5)**
- [x] AI Wizard (multi-step refinement)
- [x] RAG Assistant (streaming SSE, citations, conversations)
- [x] AI Triage Classifier (category + priority + confidence)
- [x] AI FAQ Generator (mention + show one generated entry)
- [x] Intelligent Agent Matching (top-3 with scores)

**Real-time (Channels)**
- [x] Ticket chat with presence
- [x] Live analytics dashboard feed
- [x] Notification updates

**Community**
- [x] Ask, answer, vote, accept, escalate-to-ticket, leaderboard, my-activity, reputation, badges

**Analytics**
- [x] Admin dashboard + agent dashboard + CSAT submission + live KPI WebSocket

**Knowledge Base**
- [x] CRUD, embedding regeneration, lexical fallback, ingest command

**Platform**
- [x] Role decorators, feature flags, audit log, health endpoint, Docker, ASGI/Daphne
- [x] Local Ollama + cloud Groq demonstrated side-by-side
