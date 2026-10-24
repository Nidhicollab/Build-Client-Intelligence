# Client Intelligence Web App Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A public Next.js web app on Vercel where any consultant enters a company URL, walks a six-step wizard, and gets a live, sourced, role-tailored client hub (podcast + slide-style sections) researched by ten Claude agents, plus on-demand update runs that refresh the hub and email newsletter subscribers.

**Architecture:** Next.js App Router renders the wizard, hub, library and owner page. Short Claude calls (company profile, scope summary) use the Messages API with server-side web search and fetch inside Vercel functions. Long research runs as a Claude Managed Agents multiagent session (one Opus 5.5 coordinator, ten Sonnet 5.5 specialists); the app starts the session, polls it from the progress page, and ingests the JSON files the coordinator writes to `/mnt/session/outputs/`. Postgres stores hubs, records and subscribers; blob storage holds uploads and audio.

**Tech Stack:** Node 24 LTS, Next.js (App Router, TypeScript, latest stable at install), Tailwind CSS, Vitest + Testing Library, Playwright (one smoke test), Drizzle ORM + Postgres (Neon on Vercel, Docker locally), Vercel Blob, `@anthropic-ai/sdk`, zod, Mermaid, Recharts, `pdf-parse`, `mammoth`, Resend, OpenAI speech API.

**Spec:** `docs/superpowers/specs/2026-10-23-client-intelligence-web-app-design.md` (read it before any task). The source skill lives at `~/Documents/Client Intelligence ClaudeCode Project/client-intelligence/plugins/client-industry-intel/skills/client-industry-intel/` (called SKILL below).

## Global Constraints

- No login anywhere except the owner page, which requires `OWNER_KEY` (env var) via `?key=` once, then an httpOnly cookie.
- Models: `claude-sonnet-5-5` for research specialists and update runs; `claude-opus-5-5` for the coordinator (verification and synthesis), company profile and scope summary. Never other model IDs. Opus 5.5 effort set explicitly (`high` for coordinator, `medium` for profile).
- Web tools: `web_search_20260209` and `web_fetch_20260209` on the Messages API; `agent_toolset_20260401` with `web_search`/`web_fetch` enabled for Managed Agents.
- Evidence standard (spec 7.4) is in every research prompt verbatim.
- SOW/proposal text never appears in: a web search query, research agent prompts (they get the scope summary only), the podcast script sent to speech, or any email. Only the scope summary leaves the coordinator.
- Never send any email address in a request header, URL or payload to a research source.
- Copy: plain words, help panel on every wizard step with "Why we ask" and, where useful, a "Sharp" and a "Vague" example (mockup `design-system-v1.html`).
- Design tokens exactly as spec section 12; WCAG AA on every text pair; works at 375 px width with no horizontal scroll; keyboard-usable wizard.
- Every number or claim shown in the hub links to its source record; missing data renders "not publicly available" plus the proxy.
- No section shows more than about 60 words before "Go deeper".
- Secrets only in env vars: `ANTHROPIC_API_KEY`, `DATABASE_URL`, `BLOB_READ_WRITE_TOKEN`, `RESEND_API_KEY`, `EMAIL_FROM`, `OPENAI_API_KEY`, `OWNER_KEY`, `APP_URL`, `CI_AGENT_IDS` (JSON written by Task 7). `.env.example` lists them all; `.env*` except `.env.example` is gitignored.
- Update runs never run on a schedule. No cron config.

## Review Focus

1. **Messy URLs** (`stripe.com`, `http://www.stripe.com/jobs`, trailing spaces, a non-company URL like a news article): normalize to the company origin; reject non-http(s) and IPs with a plain message. Test in Task 5.
2. **Agent output that is malformed or partial** (missing file, invalid JSON, one agent failed): ingest stores what validates, marks the affected sections `incomplete` with the reason, never crashes the hub. Test in Task 8.
3. **Leaving and returning during research** (reload, closed tab, a second device): progress page resumes from the run's stored state; ingest runs exactly once even if two tabs poll. Test in Task 8.
4. **Private company or subsidiary**: no investor section (renders "Investor view: public companies only"), subsidiary shows its filing parent. Test in Task 10.
5. **SOW leakage**: the scope summary, speech script and email bodies contain none of the SOW's names, figures or contract terms. Test with `tests/fixtures/dummy-sow.md` in Tasks 6, 11 and 14.

---

## File Structure

```
app/
  layout.tsx, globals.css                 tokens, fonts, theme
  page.tsx                                home: library of hubs
  how-it-works/page.tsx
  new/page.tsx                            wizard host (steps 1–3)
  hubs/[hubId]/page.tsx                   hub (steps 4–6 live here)
  hubs/[hubId]/research/page.tsx          progress screen
  owner/page.tsx                          owner page
  unsubscribe/[token]/page.tsx
  api/profile/route.ts                    step 1 → profile + watchlist
  api/hubs/route.ts                       create hub after step 3, start run
  api/upload/route.ts                     SOW/proposal upload → text + scope summary
  api/runs/[runId]/route.ts               poll: status, progress, ingest when done
  api/hubs/[hubId]/roles/route.ts         add a role
  api/hubs/[hubId]/subscribe/route.ts
  api/owner/update/route.ts               start an update run
  api/owner/rebuild/route.ts
  api/podcast/[hubRoleId]/route.ts        synthesize audio, resumable
components/ui/                            Button, Pill, Card, HelpPanel, Field, RoleCard
components/wizard/                        WizardShell, StepCompany, StepConfirm, StepAboutYou
components/hub/                           HubShell, SectionNav, Slide, GoDeeper, SourceLink, PodcastPlayer, WhatsNew
components/visuals/                       Kpis, Table, Cards, Quadrant, Timeline, Heatmap, Mermaid, Chart
lib/schemas.ts                            zod schemas (single source of truth for shapes)
lib/url.ts                                normalizeCompanyUrl
lib/db/schema.ts, lib/db/index.ts, lib/db/repo.ts
lib/claude/client.ts, profile.ts, scope.ts
lib/agents/prompts/*.md                   coordinator + ten specialists + update agent
lib/agents/setup.ts                       one-time create of environment and agents
lib/agents/runs.ts                        startBuildRun, startRoleRun, startUpdateRun, pollRun
lib/agents/ingest.ts                      validate + store outputs
lib/roles/<slug>.md                       copied from SKILL roles/
lib/roles/index.ts                        ROLES table, getRole
config/role-sources.ts                    spec 7.5 lists
lib/podcast/tts.ts, lib/email/newsletter.ts, lib/owner.ts, lib/cost.ts
public/samples/sample-sow.md              copy of client-intelligence tests/fixtures/dummy-sow.md
tests/…                                   mirrors lib/ and components/
Dockerfile, docker-compose.yml, .env.example, README.md
```

---

### Task 1: Toolchain, scaffold and design tokens

**Files:**
- Create: Next.js scaffold, `app/globals.css`, `app/layout.tsx`, `vitest.config.ts`, `tests/tokens.test.ts`, `.env.example`, `.gitignore` (keep `.superpowers/`)

**Interfaces:**
- Produces: CSS custom properties named exactly as in the mockup (`--surface`, `--panel`, `--chrome`, `--ink`, `--ink-2`, `--line`, `--line-soft`, `--primary`, `--on-primary`, `--selected-bg`, `--selected-ink`, `--attention`, `--attention-bg`, `--ok`, `--ok-bg`, `--tip-bg`, `--tip-edge`, `--display`, `--body`) plus `--m-*` brand colors; Tailwind theme maps to them. `npm test` runs Vitest.

- [ ] **Step 1:** Node is not installed. Ask the owner to install Node 24 LTS from nodejs.org (macOS installer), then confirm `node -v` prints `v24.x` and `npm -v` works.
- [ ] **Step 2:** In `~/Documents/Build-Client-Intelligence`, run `npx create-next-app@latest . --ts --tailwind --app --eslint --src-dir=false --import-alias "@/*" --use-npm` (keep existing README, LICENSE, docs, .gitignore lines). Add Vitest, Testing Library, jsdom.
- [ ] **Step 3: Write the failing test** `tests/tokens.test.ts`: read `app/globals.css`; assert it contains `--m-blue:#4442e3`, `--m-coral:#ff584e`, `--m-yellow:#ffb60c`, `--attention:#b8322a`, a `prefers-color-scheme: dark` block guarded by `:root:not([data-theme="light"])`, and a `:root[data-theme="dark"]` block.
- [ ] **Step 4:** Run `npm test -- tokens` → FAIL.
- [ ] **Step 5:** Port the `:root`, dark-mode blocks and base `body` rule from `.superpowers/mockups/design-system-v1.html` into `app/globals.css`; load Manrope (next/font/google, weights 400–700) and Clash Display (Fontshare stylesheet link in `layout.tsx`); top bar in `layout.tsx` with the three-dot brand mark and nav: New hub, Newsletters (links to `/#newsletters`), How it works.
- [ ] **Step 6:** Run `npm test` → PASS; `npm run build` succeeds.
- [ ] **Step 7:** Write `.env.example` with every var from Global Constraints. Commit: `Scaffold Next.js app with Method design tokens`.

### Task 2: Shared schemas

**Files:**
- Create: `lib/schemas.ts`, `tests/schemas.test.ts`, `tests/fixtures/` (`profile.json`, `records.json`, `signals.json`, `hub.json` — small valid examples)

**Interfaces:**
- Produces (zod schemas and inferred types, exported with these names):
  - `CompanyProfile`: `{ name, legalEntity, ownership: "public"|"private"|"subsidiary", parent: string|null, ticker: string|null, industry, geography, segments: string[], products: string[], careersUrl: string|null, jobBoardUrl: string|null, facts: Fact[], watchlist: string[] }`; `Fact = { key, value, status: "confirmed"|"check", sourceTitle: string|null, sourceUrl: string|null }`.
  - `EvidenceRecord`: SKILL's record schema with `id` (`r001`…), `section` enum = `company|financials|investor|industry|funding|customers|competitors|speedc|competitor_signals|voice_of_customer|role_knowledge|client_document`, `evidence_type` enum = `filing|earnings|regulator|company|website|research|news|sentiment|social|practitioner|client_document`, `confidential: boolean`.
  - `Signals`: SKILL Step 5 agent-7 shape plus `entity: string` and `source: "careers"|"job_board"|"linkedin"` on each `notable_roles` item and each `groups` item.
  - `Visual` (discriminated on `kind`): `kpis` `{items:{label,value,delta?,trend?:"up"|"down"|"flat",recordIds}[]}`; `table` `{columns:string[], rows:Cell[][]}` with `Cell = {text, level?:"high"|"medium"|"low", recordIds?}`; `cards` `{cards:{title,subtitle?,lines:string[],tags?:string[]}[]}`; `quadrant` `{xLabel,yLabel,quadrants:[string,string,string,string],points:{label,x,y}[]}`; `timeline` `{lanes:{label,items:{text,tone:"tailwind"|"headwind"|"neutral"}[]}[]}`; `heatmap` `{rows:string[],cols:string[],cells:{row,col,text,level}[]}`; `mermaid` `{code}`; `chart` `{type:"line"|"bar",xKey,series:{key,label}[],data:Record<string,number|string>[],unit}`.
  - `Slide`: `{ id, group:"start"|"for_you"|"full"|"proof", title, headline, tldr:[string,string,string], visual: Visual|null, recordIds:string[], status:"ok"|"incomplete", note?:string }`.
  - `HubDoc`: `{ brief:{findings:{text,recordIds}[] (5), implications:string[] (3), kpis: Visual&{kind:"kpis"}}, roleSlides: Slide[] (4), sections: Slide[], pointOfView: Slide[] }`.
  - `WhatsNew`: `{ checked: string, roleNotes: Partial<Record<RoleSlug,string>>, items:{kind:"news"|"hiring"|"post"|"market"|"competitor"|"creator", entity, text, url, date, paywalled?:boolean}[], topItem: string|null }`.
  - `parseOrIssues<T>(schema, data): {ok:true,value:T}|{ok:false,issues:string[]}`.

- [ ] **Step 1: Write the failing tests:** each fixture parses; a record with `section:"stock"` fails; a `Slide` with a 2-item `tldr` fails; `parseOrIssues` returns readable issue paths.
- [ ] **Step 2:** Run → FAIL. **Step 3:** Implement. **Step 4:** Run → PASS. **Step 5:** Commit `Add shared zod schemas`.

### Task 3: Database and repository

**Files:**
- Create: `lib/db/schema.ts`, `lib/db/index.ts`, `lib/db/repo.ts`, `drizzle.config.ts`, `docker-compose.yml` (Postgres 17 service only for now), `tests/db/repo.test.ts`

**Interfaces:**
- Tables: `hubs(id uuid, slug, companyUrl, profile jsonb, watchlist text[], task text, focus text, scopeSummary text|null, hasSow bool, createdAt, updatedAt)`; `hub_roles(id, hubId, role, hubDoc jsonb|null, podcastScript text|null, podcastAudioUrl text|null, podcastStatus "none"|"pending"|"ready"|"failed", podcastError text|null)`; `runs(id, hubId, kind "build"|"role"|"update", role|null, sessionId, status "running"|"ingesting"|"done"|"failed", progress jsonb, error text|null, costUsd numeric|null, usage jsonb|null, startedAt, finishedAt)`; `records(hubId, id, data jsonb)` PK (hubId,id); `signals(hubId, entity, data jsonb, checked)`; `whats_new(hubId, runId, data jsonb, createdAt)`; `subscribers(id, hubId, email, role, unsubToken, createdAt)` unique (hubId,email).
- Repo functions (all async): `createHub(input)`, `getHub(id)`, `listHubs()`, `addHubRole(hubId, role)`, `saveHubDoc(hubRoleId, doc)`, `createRun(input)`, `getRun(id)`, `claimRunForIngest(id): Promise<boolean>` (atomic `UPDATE … SET status='ingesting' WHERE id=$1 AND status='running'`), `finishRun(id, patch)`, `upsertRecords(hubId, records)`, `upsertSignals(hubId, signals[])`, `addWhatsNew(hubId, runId, data)`, `latestWhatsNew(hubId)`, `subscribe(hubId, email, role)`, `unsubscribe(token)`, `subscribersFor(hubId)`.

- [ ] **Step 1: Failing tests** (against the docker-compose Postgres, `DATABASE_URL` from `.env.test`): create then get a hub; `claimRunForIngest` returns true once and false on the second call; `subscribe` twice with the same email keeps one row; `unsubscribe(token)` removes it.
- [ ] **Step 2:** Docker is not installed: ask the owner to install Docker Desktop, then `docker compose up -d db`. Run tests → FAIL.
- [ ] **Step 3:** Implement with Drizzle; `npm run db:push` script. **Step 4:** PASS. **Step 5:** Commit `Add database schema and repository`.

### Task 4: UI kit and wizard shell

**Files:**
- Create: `components/ui/{Button,Pill,Card,HelpPanel,Field,RoleCard}.tsx`, `components/wizard/WizardShell.tsx`, `lib/roles/index.ts`, `lib/roles/*.md` (copy SKILL `roles/*.md`), `tests/components/wizard.test.tsx`

**Interfaces:**
- `ROLES: {slug, label, outputs:[string,string,string,string], blurb}[]` in spec section 2 order; `getRole(slug)`; `readRoleFile(slug): Promise<string>`.
- `<WizardShell step={1..6} company={string|null} help={ReactNode}>` renders the black progress band with the six step labels (Company, Confirm, About you, Research, Your hub, Newsletter), `aria-current="step"` on the current one, form left, help right.
- `<Button variant="primary"|"quiet" size?>`, `<Pill tone="ok"|"check"|"role"|"new">`, `<HelpPanel title examples?={{sharp?:string, vague?:string}}>`, `<RoleCard role selected onSelect>` (a radio in a radiogroup).

- [ ] **Step 1: Failing tests:** WizardShell marks step 3 current and steps 1–2 done; RoleCards are keyboard-selectable with arrow keys and expose `role="radio"`; HelpPanel renders "Sharp"/"Vague" labels only when given.
- [ ] **Steps 2–4:** FAIL → implement matching `design-system-v1.html` → PASS.
- [ ] **Step 5:** Commit `Add UI kit and wizard shell`.

### Task 5: Steps 1–2: company URL, profile, confirm

**Files:**
- Create: `lib/url.ts`, `lib/claude/client.ts`, `lib/claude/profile.ts`, `app/api/profile/route.ts`, `components/wizard/StepCompany.tsx`, `components/wizard/StepConfirm.tsx`, `app/new/page.tsx`, tests `tests/url.test.ts`, `tests/claude/profile.test.ts`

**Interfaces:**
- `normalizeCompanyUrl(input: string): {ok:true, url:string /* https origin */}|{ok:false, message:string}`.
- `buildCompanyProfile(url: string, deps?:{client}): Promise<CompanyProfile>`: one `messages.create` on `claude-opus-5-5`, effort `medium`, tools `web_search_20260209` + `web_fetch_20260209`, structured output `output_config.format` from `CompanyProfile`'s JSON schema, `fallbacks:"default"` with beta `server-side-fallback-2026-07-01`. Prompt = SKILL Steps 1–2 plus spec 6 step 2 and 7.3 (watchlist). Handles `pause_turn` by continuing.
- `POST /api/profile {url}` → `{profile}` or `{error}`; `export const maxDuration = 300`.
- Wizard state in the client (`useReducer`), persisted to `sessionStorage` (try/catch) so a reload keeps answers.

- [ ] **Step 1: Failing tests:** `normalizeCompanyUrl` → `"stripe.com"`→`https://stripe.com`; `" http://www.stripe.com/jobs "`→`https://www.stripe.com`; `"ftp://x.com"` and `"http://10.0.0.1"` → not ok with a message; `"not a url"` → not ok. `buildCompanyProfile` with a fake client returning fixture JSON yields a valid `CompanyProfile`; when the fake returns invalid JSON it throws `ProfileError` with a user-facing message.
- [ ] **Steps 2–4:** FAIL → implement → PASS.
- [ ] **Step 5:** StepCompany: URL field, help "Why we ask" (spec 6), and a fallback "Enter the company name instead" path when the profile call reports the site blocked reading. StepConfirm: fact list with `Confirmed`/`Check this` pills and source, "N to check" notice, edit-in-place for any fact, parent/subsidiary question when `ownership==="subsidiary"` and parent unclear, watchlist chips with "Add a term".
- [ ] **Step 6:** Manually run `npm run dev`, enter `https://stripe.com`, confirm the facts screen renders. Commit `Add company intake and confirm steps`.

### Task 6: Step 3: about you, documents, scope summary

**Files:**
- Create: `components/wizard/StepAboutYou.tsx`, `app/api/upload/route.ts`, `lib/claude/scope.ts`, `public/samples/sample-sow.md`, tests `tests/claude/scope.test.ts`

**Interfaces:**
- `extractText(file: {name, bytes}): Promise<string>` (PDF via pdf-parse, .docx via mammoth, .md/.txt as is; else throws `UnsupportedFile`).
- `summarizeScope(docText: string, deps?): Promise<{scopeSummary: string}>`: Opus 5.5, no web tools, prompt = SKILL Step 4 confidentiality rules: kind of work and business areas only, no staff names, no figures, no contract terms.
- `POST /api/upload` (multipart, max 10 MB) → `{fileId, scopeSummary}`; stores the raw text in blob storage, private.

- [ ] **Step 1: Failing test:** run `summarizeScope` on `public/samples/sample-sow.md` with a recorded fake response, and (separately, `RUN_LIVE=1`) live; assert the summary contains none of `Alex Rivera`, `Sam Lee`, `Priya Nair`, `1,200,000`, `$1.2`, `20%`, `40 crews`.
- [ ] **Steps 2–4:** FAIL → implement → PASS.
- [ ] **Step 5:** StepAboutYou: RoleCards with each role's four outputs; task (2–3 lines) and focus fields with Sharp/Vague help; drop zone for PDF/Word plus "Use a sample SOW"; "Contains sample client material" note when a document is attached; "Start research" shows "About 20–40 min · you can leave". Commit `Add about-you step and scope summary`.

### Task 7: Research agents and prompts

**Files:**
- Create: `lib/agents/prompts/{coordinator,company,investor,industry,customers,competitors,speedc,client-signals,competitor-signals,voice-of-customer,role-knowledge,update}.md`, `config/role-sources.ts`, `lib/agents/setup.ts`, `scripts/setup-agents.ts`, `tests/agents/prompts.test.ts`

**Interfaces:**
- `ROLE_SOURCES: Record<RoleSlug, {name, kind:"newsletter"|"podcast"|"youtube"|"blog"|"substack", url?: string}[]>` exactly the spec 7.5 table.
- `setupAgents(): Promise<{environmentId, coordinatorId, specialistIds: Record<string,string>, updateAgentId}>`: creates one cloud environment and the agents; prints JSON for `CI_AGENT_IDS`. Run once via `npx tsx scripts/setup-agents.ts`; never in the request path. Specialists: `claude-sonnet-5-5`, toolset with only `web_search`, `web_fetch`, `read`, `write` enabled. Coordinator: `claude-opus-5-5`, effort `high`, `multiagent: {type:"coordinator", agents:[...specialistIds]}`, toolset with `web_search`, `web_fetch`, `read`, `write`, `bash`. Permission policy `always_allow` for these read/web/write tools (runs are unattended).
- Environment networking: `unrestricted` is required for open-web research. Before running setup, show the owner the docs' warning (whole sandbox gets full egress) and get a yes; keep no secrets or files other than the run's inputs in the sandbox.
- Prompts: specialists 1–6 = SKILL Step 5 assignments verbatim + shared rules verbatim + spec 7.4; agent 7/8 = spec 7.2; 9 = spec 7.1 row 9; 10 = spec 7.5 with `ROLE_SOURCES` injected. Coordinator = SKILL Steps 5–7 (spawn all specialists in parallel, skip investor for non-public), spec 7.4 verification, then write outputs (Task 8 file contract), using the role file's outputs and podcast segment and SKILL `outputs/podcast.md` script rules for two hosts. The coordinator also writes `pointOfView` slides (spec 8.1) from the six transformation frameworks: copy each framework's definition and its "What the briefing supplies" paragraph from the Framework Guide (Google Doc `1cRi-vkNXB3iFRfVVwzrIeIPufj6hzb8nigOgNdQGAYo`, read once and saved as `lib/agents/prompts/frameworks.md`), and order them by the engagement focus.

- [ ] **Step 1: Failing tests:** every prompt file exists; each specialist prompt contains the strings `not publicly available`, `company claim`, and the email rule sentence; `coordinator.md` names all ten specialists and every output file in Task 8's contract; `frameworks.md` names all six transformation frameworks; `role-knowledge.md` renders the PM list including `Lenny Rachitsky`, `Aakash Gupta`, `Claire Vo`.
- [ ] **Steps 2–4:** FAIL → write prompts and setup → PASS.
- [ ] **Step 5:** With owner approval of networking, run setup once; put the printed JSON in `.env.local` as `CI_AGENT_IDS`. Commit `Add research agents and prompts` (no IDs committed).

### Task 8: Starting, polling and ingesting a build run

**Files:**
- Create: `lib/agents/runs.ts`, `lib/agents/ingest.ts`, `lib/cost.ts`, `app/api/hubs/route.ts`, `app/api/runs/[runId]/route.ts`, `app/hubs/[hubId]/research/page.tsx`, tests `tests/agents/ingest.test.ts`, `tests/agents/runs.test.ts`

**Interfaces:**
- Output file contract (coordinator writes to `/mnt/session/outputs/`): `records.json` (EvidenceRecord[]), `signals.json` (Signals[], client first), `hub.json` (HubDoc), `podcast-script.md`, `progress.json` (`{agents:{name,status:"queued"|"running"|"done"|"failed",note?}[]}`, rewritten as work proceeds), `errors.json` (optional `{agent, message}[]`).
- `startBuildRun(hubId): Promise<{runId}>`: uploads a `run-input.json` (profile, watchlist, role, task, focus, scopeSummary, ROLE_SOURCES for the role) as a session file resource; creates the session; sends `user.message` "Start the build described in /mnt/session/inputs/run-input.json".
- `pollRun(runId): Promise<{status, progress, hubId}>`: retrieves the session; reads `progress.json` if present; when the session is idle and not `requires_action`, calls `claimRunForIngest` and, if it wins, `ingestRun`. Returns stored state for any later poll.
- `ingestRun(runId)`: download files (retry the listing up to 3 times, 2 s apart); validate each with `parseOrIssues`; store valid parts; any invalid or missing part marks the matching slides `status:"incomplete"` with `note` = the reason; sets `podcastStatus:"pending"` if a script exists; records `usage` and `costUsd` via `estimateCost(usage)` (Sonnet $2/$10, Opus $4/$20 per MTok, web search $10 per 1,000, session $0.08/hour, or `usage.list_cost` when present).
- `POST /api/hubs` → creates hub + role row + run → `{hubId, runId}`; client navigates to `/hubs/{hubId}/research?run={runId}`.
- Research page polls `GET /api/runs/{runId}` every 10 s; shows each agent's status, elapsed time, "You can close this tab; your hub link is …" with copy button; redirects to the hub when done; on failure shows the error and "Try again".

- [ ] **Step 1: Failing tests:** `ingestRun` with fixture files stores records and hub doc; with `hub.json` missing, ingest succeeds, marks role slides incomplete with note "Hub layout was not produced", and the run is `done`; with invalid `records.json`, issues are saved in `runs.error` and slides citing missing records are `incomplete`. `pollRun` called twice concurrently on an idle session ingests once (fake client + real DB).
- [ ] **Steps 2–4:** FAIL → implement → PASS.
- [ ] **Step 5:** Commit `Add build runs with polling and ingest`.

### Task 9: Visual primitives

**Files:**
- Create: `components/visuals/{Kpis,Table,Cards,Quadrant,Timeline,Heatmap,Mermaid,Chart,Visual}.tsx`, `tests/components/visuals.test.tsx`

**Interfaces:**
- `<Visual visual={Visual} records={Map<string,EvidenceRecord>}>` dispatches on `kind`. Level chips: high = attention colors, medium = yellow tint, low = ok colors (always with the word, never color alone). Mermaid renders client-side with theme variables from the tokens and re-renders on theme change; wide diagrams scroll horizontally inside their card. Chart uses Recharts with the brand series order blue, coral, yellow, dark grey; y-axis shows `unit`.
- Every value with `recordIds` renders a `<SourceLink>` (superscript number opening the record's source in a new tab, title attr = source title and date).

- [ ] **Step 1: Failing tests:** a table cell with `level:"high"` shows text "High"; a KPI with `recordIds:["r003"]` renders a link to that record's `source_url`; Mermaid with invalid code renders a readable fallback message, not a crash; each kind renders at 375 px without overflowing its container (check `scrollWidth <= clientWidth` on the card except Mermaid's scroll area).
- [ ] **Steps 2–4:** FAIL → implement → PASS. **Step 5:** Commit `Add slide visual primitives`.

### Task 10: The hub page

**Files:**
- Create: `app/hubs/[hubId]/page.tsx`, `components/hub/{HubShell,SectionNav,Slide,GoDeeper,SourceLink,WhatsNew}.tsx`, `tests/components/hub.test.tsx`

**Interfaces:**
- URL: `/hubs/{hubId}?role={slug}&s={slideId}`; default role = the hub's first role.
- `<Slide slide records>`: title, one-sentence headline, 3-line TL;DR, visual, "Go deeper" (`<details>`) listing each record with claim, metric, source link, date, confidence, evidence type, and a "Company claim" or "Sentiment" label where applicable. Incomplete slides show the note and still render whatever validated.
- Nav groups and order exactly as spec 8.1; role switcher lists built roles and "Add a role" (Task 12). Header: company, "Updated <date>", role pill, "N new since last update" (count from latest `whats_new`), "Contains sample client material" badge when `hasSow`.
- Proof group: Sources (all records, filterable by section and evidence type) and Data gaps (records with claim "not publicly available" or an `open_question`).

- [ ] **Step 1: Failing tests:** with fixture data, nav shows the four groups in order and the role's four output titles under "For you"; a private-company profile shows "Investor view: public companies only" in place of the investor slide; a subsidiary shows "Filing parent: <parent>" in Company snapshot; the badge appears only when `hasSow`; the Hiring and talent slide shows careers-page, job-board and LinkedIn counts as separate labeled numbers and never a combined total; a slide whose TL;DR + headline exceed 60 words fails a dev-time `console.warn` assertion (guardrail for prompts).
- [ ] **Steps 2–4:** FAIL → implement → PASS. **Step 5:** Commit `Add hub page`.

### Task 11: Podcast

**Files:**
- Create: `lib/podcast/tts.ts`, `app/api/podcast/[hubRoleId]/route.ts`, `components/hub/PodcastPlayer.tsx`, `tests/podcast/tts.test.ts`

**Interfaces:**
- `parseScript(md: string): {speaker:"A"|"B", text:string}[]` (lines starting `HOST A:`/`HOST B:`; ignores `#` titles and `[bracketed]` text); `estimateMinutes(turns)` at 150 wpm; `chapters(md)` from `##` headings or the six SKILL parts.
- `synthesize(hubRoleId)`: OpenAI speech API, two voices from env `TTS_VOICE_A`/`TTS_VOICE_B`, model from `TTS_MODEL`; processes turns in order, saving each MP3 chunk to blob and progress in `hub_roles`, so a timed-out call resumes from the next turn; when all chunks exist, concatenates to one MP3 and sets `podcastStatus:"ready"`. `maxDuration = 300`. The hub page triggers it once when status is `pending` and keeps polling.
- Player: black card, play/pause, scrubber, chapters, "Read the script"; on `failed` shows the script and `podcastError`.

- [ ] **Step 1: Failing tests:** `parseScript` on a sample yields alternating turns and drops bracketed text; `estimateMinutes` of 2,200 words ≈ 14.7; before sending to speech, a guard `assertNoSowTerms(script, sowText)` throws if any capitalized name, number with a currency or %, or term from the SOW appears (test with `sample-sow.md`).
- [ ] **Steps 2–4:** FAIL → implement → PASS. **Step 5:** Commit `Add two-host podcast audio`.

### Task 12: Add a role to a hub

**Files:**
- Create: `app/api/hubs/[hubId]/roles/route.ts`, `startRoleRun` in `lib/agents/runs.ts`, `tests/agents/role-run.test.ts`

**Interfaces:**
- `startRoleRun(hubId, role)`: session input includes stored `records.json` and `signals.json` plus the role; coordinator prompt section "Role run": run only that role's extra research asks (specialists as needed), then write `records.json` (new records only), `hub.json` (that role's `roleSlides` and brief implications; shared sections unchanged), `podcast-script.md`.
- Ingest merges new records and writes the new `hub_roles.hubDoc` with shared `sections` copied from the first role.

- [ ] **Step 1: Failing test:** ingesting a role run adds a second `hub_roles` row, keeps existing records, and the new role's doc has its own four `roleSlides`.
- [ ] **Steps 2–4:** FAIL → implement → PASS. **Step 5:** Commit `Add role runs`.

### Task 13: Owner page and update runs

**Files:**
- Create: `lib/owner.ts`, `app/owner/page.tsx`, `app/api/owner/update/route.ts`, `app/api/owner/rebuild/route.ts`, `startUpdateRun` in `lib/agents/runs.ts`, `lib/agents/prompts/update.md` (from Task 7), `tests/owner.test.ts`, `tests/agents/update-run.test.ts`

**Interfaces:**
- `requireOwner(req): boolean` compares the cookie or `?key=` to `OWNER_KEY` in constant time; a valid `?key=` sets an httpOnly, secure, same-site cookie and redirects without the key.
- `startUpdateRun(hubId)`: update agent (Sonnet 5.5, `self` roster) researches only the last 24 hours per spec 10, for the client and competitors, using the watchlist and the hub's roles' `ROLE_SOURCES`; writes `whats-new.json` (WhatsNew) and `signals.json`. Ingest stores both, bumps `hubs.updatedAt`, then calls `sendNewsletters(hubId, runId)` (Task 14).
- `rebuild` starts a new build run for the hub's first role, then role runs for the others.
- Owner page: table of hubs with last update, buttons "Run today's update" and "Rebuild hub", and runs with status, duration and cost.

- [ ] **Step 1: Failing tests:** `/owner` without key returns 401 page; with key sets cookie; `POST /api/owner/update` without owner returns 401; ingesting a fixture `whats-new.json` makes the hub's "N new since last update" equal the item count and keeps items older than 24 h out (filter by `date`).
- [ ] **Steps 2–4:** FAIL → implement → PASS. **Step 5:** Commit `Add owner page and update runs`.

### Task 14: Newsletter

**Files:**
- Create: `app/api/hubs/[hubId]/subscribe/route.ts`, `lib/email/newsletter.ts`, `app/unsubscribe/[token]/page.tsx`, hub step 6 panel in `components/hub/Subscribe.tsx`, `tests/email/newsletter.test.ts`

**Interfaces:**
- `renderNewsletter({hub, whatsNew, role, unsubUrl}): {subject, html, text}`: subject `"<Company> daily: <topItem>"`; sections Top item, News, People's posts, New roles, What it means for you (one or two lines per role from the update run's role notes); empty update → text "Nothing new on <Company> in the last 24 hours."; every item has its link; footer unsubscribe link.
- `sendNewsletters(hubId, runId)`: one Resend email per subscriber; failures logged per subscriber, never stop the others.
- Subscribe panel (step 6 of the wizard, shown at the end of a new hub and in the hub's "What's new"): email + role, help text "Updates are sent when a new update runs. In this prototype the owner runs them."

- [ ] **Step 1: Failing tests:** subject format; empty update renders the one-line text; rendered html/text contains none of the sample SOW's terms even when the hub has `hasSow` (render from a fixture hub whose `scopeSummary` and records include confidential items: confidential records are excluded); unsubscribe page with a bad token shows "This link has expired" and with a good one removes the row.
- [ ] **Steps 2–4:** FAIL → implement → PASS. **Step 5:** Commit `Add newsletter`.

### Task 15: Home library and How it works

**Files:**
- Modify: `app/page.tsx`; Create: `app/how-it-works/page.tsx`, `tests/components/home.test.tsx`

**Interfaces:**
- Home: hero ("Paste a company's website. Get a hub built for your role."), primary "Start a new hub", library grid (company, roles built as pills, last updated, link), `#newsletters` section explaining how to subscribe from any hub.
- How it works: the six steps with one sentence each, the ten agents, the evidence standard in plain words, the role output table, and what the prototype doesn't do (no login, sample documents only, updates on demand).

- [ ] **Step 1: Failing test:** with two fixture hubs, the library lists both with their role pills; with none, it shows an empty state pointing to "Start a new hub".
- [ ] **Steps 2–4:** FAIL → implement → PASS. **Step 5:** Commit `Add home library and how-it-works page`.

### Task 16: Docker, README and Vercel

**Files:**
- Create: `Dockerfile` (multi-stage, Next.js standalone output), extend `docker-compose.yml` (`app` + `db`), `README.md`, Playwright `tests/e2e/smoke.spec.ts`

**Interfaces:**
- `docker compose up` serves the app at `http://localhost:3000` against the compose Postgres.
- README: what it is, local run (Node and Docker), env vars, one-time agent setup, deploy to Vercel (Neon + Blob from the Vercel Marketplace, env vars, `npm run db:push`), owner page, cost notes, prototype limits.

- [ ] **Step 1:** Playwright smoke: home loads; "Start a new hub" opens step 1; entering `stripe.com` shows the normalized URL; no horizontal scroll at 375 px.
- [ ] **Step 2:** `docker compose up --build` → smoke passes against it.
- [ ] **Step 3:** Ask the owner to create the Vercel project from the GitHub repo and add the env vars; deploy; run the smoke against the Vercel URL.
- [ ] **Step 4:** Commit `Add Docker, README and smoke test`; push.

### Task 17: End-to-end demo run

No product code. Costs real money: confirm with the owner first.

- [ ] **Step 1:** On the Vercel URL, build a hub for `https://stripe.com` as Product manager with no SOW. Check: confirm step shows confirmed/check facts with sources and a watchlist; research finishes; the hub shows all four groups; every slide either renders or says why it is incomplete; five random source links resolve; podcast plays and is 13–17 minutes; role knowledge cites Lenny, Aakash Gupta or Claire Vo content where relevant; no slide over ~60 words before "Go deeper".
- [ ] **Step 2:** Build a hub for `https://www.duke-energy.com` as Engagement lead with the sample SOW. Check the badge appears, the investor view is present (public), and the podcast script and scope summary contain no SOW terms.
- [ ] **Step 3:** Add the Engineering lead role to the Stripe hub; check new role slides and podcast.
- [ ] **Step 4:** Subscribe the owner's address to the Stripe hub; from the owner page run one update; check the email arrives with the five sections, only last-24-hour items, links, and an unsubscribe link that works.
- [ ] **Step 5:** Report each check to the owner, failures first, with the owner page's cost per run.
