# Client Intelligence web app: design

Date: 2026-10-23
Status: approved by owner 2026-10-23
Source skill: [Nidhicollab/client-intelligence](https://github.com/Nidhicollab/client-intelligence) (`plugins/client-industry-intel`), v1 and v2

## 1. Intent

Consultants starting work with a new client need to get up to speed on the client's business, market, industry, competitors, stock, hiring and news, and then keep up as things change. The `client-industry-intel` Claude Code skill does this today, but only for people who run Claude Code. Version 1 produced a 20-page briefing that was thorough but hard to read. Version 2 produced a podcast and four role-tailored visuals but dropped much of v1's depth.

This web app makes the skill usable by any consultant with only a browser, and combines the two versions: v1's full research depth and breadth, delivered in v2's role-tailored, visual, short-form outputs. The hub stays live: each update run refreshes it with the latest news, hiring, stock and competitor moves.

It is a prototype and proof of concept for the owner's firm (Method). It is for demos with sample client documents, not production use.

## 2. Users

Consultants at the firm, on different client projects, in one of five roles:

| # | Role | Slug |
|---|---|---|
| 1 | Product manager | `product-manager` |
| 2 | Engineering lead or architect | `engineering-lead` |
| 3 | UX or product designer | `ux-designer` |
| 4 | Engagement lead or partner | `engagement-lead` |
| 5 | Business or data analyst | `data-analyst` |

Assume they have never used Claude Code and won't read documentation. The app must walk them through each step with help text.

## 3. Goals, non-goals, anti-goals

**Goals**
- Anyone can enter any company's website URL and get a fact-checked, sourced hub tailored to their role.
- Research is both deep (v1's six frameworks and evidence standard) and wide (more sources, competitor signals, role knowledge).
- Outputs are easy to take in: a 15-minute podcast first, then slide-style visual sections. No long documents.
- Anyone can sign up for newsletters on several companies.
- The hub stays current: an update run refreshes it and emails subscribers.

**Non-goals for this prototype**
- Login, accounts, permissions, multi-tenant separation.
- Real confidential client data. Only sample SOWs and proposals are used.
- A scheduled daily newsletter. Updates run only when the owner triggers them (section 10).
- A Miro board export.
- Kubernetes.
- Paid market-data feeds. Stock and market data comes from public web sources.

**Anti-goals (failure even if it works)**
- A wall of text. If a section reads like the v1 PDF split into pages, it has failed.
- Unsourced or guessed numbers. Every claim traces to a source; gaps say "not publicly available".
- A wizard that leaves people unsure what to type or why.
- Getting around paywalls or logins to read content.

## 4. Decisions made by the owner

| Topic | Decision |
|---|---|
| Delivery | A web app, written as normal code, opened in Cursor |
| Location | `~/Documents/Build-Client-Intelligence`, pushed to `github.com/Nidhicollab/Build-Client-Intelligence` |
| Hosting | Vercel |
| Access | No login. Anyone with the link can use it |
| Model | Claude, on the owner's Anthropic API key. Owner accepts the cost |
| Client data | Prototype only, sample SOWs and proposals |
| Flow layout | Step-by-step wizard (mockup option A): one step per screen, progress bar, help panel |
| Visual design | Method's brand colors, with expert product-design practice (section 12) |
| Intake | Matches v2 exactly (section 6) |
| Outputs | v1's full report sections plus all of v2's outputs, organized by role (section 8) |
| Output style | Slide-style layouts: boxes, shapes, color and lines used well, less is more. Not only flowcharts |
| Research | v1 depth and breadth plus the additions in section 7 |
| Hub | Live, refreshed by each update run |
| Newsletter | In the prototype, not scheduled. Runs one day's update only when the owner says |
| Containers | Docker yes (for local runs and portability), Kubernetes no |

## 5. Constraints

- **Vercel request limits.** A full hub build takes 20 to 40 minutes, far longer than a Vercel function may run. Research must run outside the request (section 11).
- **Cost.** Every build and update spends the owner's Anthropic credit. Estimate: about $10 to $20 per full build with ten agents; well under $1 per update run. Measure real cost on the first runs and show it in an admin view.
- **Public content only.** LinkedIn posts and LinkedIn job posts are used only as they appear in web search results or on public pages. No logged-in scraping. No paywall bypass.
- **Evidence standard.** v1 and v2's rules apply everywhere (section 7.4).
- **Sample documents.** SOW and proposal content never goes into a web search query, a newsletter, or text sent to a third-party speech service (same rule as v2).

## 6. The wizard (intake matches v2)

Layout: top bar in Method black; a black progress band with the company name and six steps; a light work area with the form on the left and a help panel on the right. On phones the help panel moves below the form.

Every step has: a plain heading, one line saying why the step exists, the form, and a help panel with "why we ask" plus a sharp and a vague example answer where it helps.

| Step | What the user does | What the app does |
|---|---|---|
| 1. Company | Pastes the company's website URL | Reads the site's key pages: about, products, investor relations, newsroom, leadership, careers |
| 2. Confirm | Reviews a fact list and corrects anything wrong | Works out name, legal entity, public/private/subsidiary (and filing parent), industry, geography, segments, main products, official careers page and the job board behind it (Greenhouse, Lever, Workday). Checks each against an independent source. Marks each "Confirmed" (with source) or "Check this". Also shows the keyword watchlist (7.3), which the user can add to. If parent vs subsidiary is unclear, asks which one the engagement is with |
| 3. About you | Picks a role (cards that preview each role's outputs), writes 2 or 3 lines on what they've been asked to build or change, the project or idea focus, and optionally uploads a sample proposal and SOW (PDF or Word), or clicks "Use a sample SOW" (ships with the app: the skill repo's `tests/fixtures/dummy-sow.md`) | Reads the documents and writes a scope summary with no names, figures or contract terms |
| 4. Research | Waits, or leaves and comes back via the hub link | Runs the research (section 7). Shows live progress per agent, with an honest time estimate |
| 5. Your hub | Reads the hub | Section 8 |
| 6. Newsletter | Enters an email address to subscribe | Section 10 |

If the site blocks automated reading, step 1 asks for the company name and continues. If a role isn't listed, the user picks the closest one; the role cards should make this easy.

Not in intake (v1 questions that do not come back): scope, depth, optional modules. Every run is full depth. The investor view runs for public companies only.

Podcast voice is not a question: the app always uses a two-host speech service (section 9).

## 7. Research

### 7.1 Agents

Ten research agents run in parallel. Agents 1 to 7 are v2's agents, which carry v1's assignments in full. 8 to 10 are new.

| # | Agent | Covers | Framework |
|---|---|---|---|
| 1 | Company and financials | v1/v2 assignment: business model, revenue by segment, latest annual and quarterly results, guidance, capex, stated strategy, leadership changes, 90 days of material news, 6 to 10 industry KPIs with benchmarks | Industry KPI library |
| 2 | Investor and market (public only) | v1/v2 assignment: price vs sector index (1 year, YTD), valuation vs peers, rating mix and price-target range, recurring earnings-call questions, short interest, institutional and activist moves | Bull and bear cases |
| 3 | Industry and funding | v1/v2 assignment: 10 to 15 years of industry evolution, current state, market size with who estimated it, innovations, business model shifts; VC and PE funding over 4 to 8 quarters, sector rank, notable rounds, most active investors | Industry evolution, funding radar |
| 4 | Customers and segments | v1/v2 assignment: buckets, need, revenue share, growth, buyer/user/influencer, switching costs, emerging segments, internal users when relevant | Buying roles |
| 5 | Competitors | v1/v2 assignment: 3 to 5 of each type with evidence, what they do differently, recent moves, traction, threat | Incumbent / Adjacent / Insurgent |
| 6 | SPEED-C and regulation | v1/v2 assignment: 2 to 4 forces per driver, tailwind or headwind, time horizon, effect, pending legislation and rulings | SPEED-C |
| 7 | Client signals | See 7.2, for the client | Hiring as a spending signal |
| 8 | Competitor signals and investment | See 7.2, for each competitor from agent 5's list (or from a quick pre-scan if agent 5 hasn't returned): funding rounds, acquisitions, patents, product launches, R&D statements, hiring by team and skill, where they are putting money | Where competitors invest |
| 9 | Voice of the customer | App store ratings (score, count, date checked), review sites, Reddit and community threads, support complaint themes. All labeled sentiment | Sentiment |
| 10 | Role knowledge | See 7.5 | Practitioner view |

Each agent also gets its role's extra research asks, taken from the skill's role files (`plugins/client-industry-intel/skills/client-industry-intel/roles/<slug>.md`, section "Research asks").

Agents 1 to 6, 8, 9 and 10 return evidence records (7.4). Agent 7 returns v2's Signals object, extended in 7.2.

### 7.2 Signals (client and competitors)

For the client (agent 7) and for each competitor (agent 8):

- **Social accounts.** The company's and its leaders' public posts on LinkedIn, X, YouTube and the company blog, as they appear in web search or public pages. Keep business signals only: hiring pushes, launches, team announcements, strategy. Name the person and title as the source. Do not build profiles of individuals.
- **News TL;DRs.** RSS feeds (company newsroom, a Google News RSS feed for the company name, the industry's main trade publications) and industry press. One line per item with its link. Paywalled items are headline-only and marked.
- **Job posts.** The official careers page, the job board it links to (for example Greenhouse), and LinkedIn job posts as they appear in public search. Group by team, location and seniority; count the skills and tools named; list notable roles; give a total. This changes v2's "official careers page only" rule at the owner's request: LinkedIn jobs are now included, labeled by source, and counts from different sources are never added together.
- **Where money goes.** From the above, infer where the company is growing and spending, and label it as inference.

Keep v2's Signals JSON shape as the base (`checked`, `hiring`, `posts`, `news`, `feeds`) and add `entity` (client or competitor name) and a `source` field on each hiring item.

### 7.3 Keyword watchlist

Built at step 2 from the confirmed profile: company and product names, executives, top competitors, regulators, and the industry's key terms. Shown to the user, who can add terms. Every research agent and every update run uses it in its searches.

### 7.4 Evidence standard (from v1 and v2, unchanged)

- Source priority, highest first: regulatory filings and regulators; earnings materials and call transcripts; company releases and website; reputable research and industry bodies; major business press; trade press. Blogs, forums, social posts and creator content count as sentiment or practitioner opinion and are labeled that way.
- A claim found only on the company's website is a company claim unless an independent source confirms it.
- Every metric has a value, unit, reporting period and reporting entity.
- Reported facts, third-party estimates and interpretation are kept separate.
- Missing data says "not publicly available" and names the closest proxy. Never guess.
- Prefer the last 12 months; mark anything older.
- Never put the user's or anyone's email address in a request header, URL or payload.
- Record shape: v2's record schema, with `section` extended to include `competitor_signals`, `voice_of_customer` and `role_knowledge`, and `evidence_type` extended with `social` and `practitioner`.

Verification (after research, from v1 and v2): re-fetch the 10 most important records, starting with numbers used in the brief and podcast; show conflicting values side by side, never averaged; downgrade stale or single-source claims; label company-only claims; show SOW vs public evidence gaps as client questions; list coverage gaps as open questions.

### 7.5 Role knowledge (agent 10)

The agent searches the public content of the role's practitioner sources (posts, newsletters, podcast episode pages and transcripts, YouTube video pages and transcripts) for what applies to this client, industry and task. It returns 5 to 15 records, each with the creator, the piece, the date, the idea, and how it applies here. It reads only public content. For paywalled posts it uses the public preview and says so. It does not try to read whole archives on every run; it searches them.

Source lists live in one editable config file, so the owner can change them without code changes:

| Role | Sources |
|---|---|
| Product manager (owner's list) | Lenny Rachitsky (Lenny's Newsletter, Lenny's Podcast, all public content), Aakash Gupta (Product Growth), Claire Vo (How I AI, ChatPRD), plus creators on AI product management, AI product builders and forward-deployed PMs across Substack, YouTube, podcasts and newsletters |
| Product manager (my additions) | Marty Cagan (SVPG), Shreyas Doshi |
| Engineering lead (my picks) | Gergely Orosz (The Pragmatic Engineer), Simon Willison, Latent Space (swyx and Alessio), Will Larson (Irrational Exuberance), Martin Fowler |
| UX designer (my picks) | Nielsen Norman Group, Figma Config talks, Lenny's design episodes, Jared Spool (Center Centre), Julie Zhuo |
| Engagement lead (my picks) | Stratechery (Ben Thompson), HBR, McKinsey Insights, Not Boring (Packy McCormick), Ethan Mollick (One Useful Thing) |
| Data analyst (my picks) | Benn Stancil (benn.substack), Locally Optimistic, Seattle Data Guy, Ethan Mollick, Chip Huyen |

The owner will review the lists marked "my picks".

## 8. The hub

### 8.1 Structure

One page per company and role, with a left section menu (a bottom sheet on phones). Header: company name, "Updated <date time>", the role view, "Switch role", and a "new since last update" count.

| Group | Sections | From |
|---|---|---|
| Start here | The brief (5 things to know, 3 implications, 4 headline KPIs); Podcast (15 min); What's new (latest update run) | v1 executive summary, v2 podcast, new |
| For you | The role's four outputs, as defined in the skill's role files (`## Outputs`, items 2 to 5) | v2 |
| Full picture | Company snapshot; Financials and KPIs; Investors and stock; Industry landscape; Funding and innovation; Customers and segments; Competitors; Where competitors invest; Hiring and talent; Voice of the customer; SPEED-C forces; Role knowledge; Implications and questions; Point of view (drafts) | v1 sections 2 to 10, plus new |
| Proof | Sources; Data gaps | v1 section 11, v2 Sources |

Switching role reorders "For you" and the brief's implications. It does not hide any section. A new role on an existing hub needs that role's four outputs and its own podcast (the role segment differs); the shared research is reused, and role-specific research asks run only for the new role.

**Implications and questions** keeps v1's contents: testable hypotheses, opportunities and risks, 5 to 8 stakeholder interview questions, questions to validate with the client, and "what would change our view".

**Point of view (drafts)** gives a first pass at the six transformation frameworks from the Framework Guide (Waves of Change, capability map, transformation touchstones, Kotter's Accelerate, Business Portfolio Map, Lean Startup), each marked as a draft for the consultant to challenge, with the evidence it used. The engagement focus picks which open first (Framework Guide table: digital transformation, then Waves, capability map and touchstones; new product, then Lean Startup and Portfolio Map).

### 8.2 Every section is a slide

Each section is laid out like a well-made slide, not a document page:

- A headline that states the "so what" in one sentence.
- A 3-line TL;DR under it.
- One main visual that fits the content: KPI tiles, a 2x2 or quadrant, a comparison table with level chips, a timeline, cards, a tree, a flow, a heatmap, a chart. Pick the form from the content, not one form for everything.
- "Go deeper" expands the full evidence list for that section, each item with its source link, date, confidence and evidence type.
- Less is more: no section shows more than about 60 words before "Go deeper".
- Every number or claim on the slide links to its source.
- Charts and diagrams are drawn from data, not images, and read correctly in light and dark themes.

Role outputs keep v2's visual types (mermaid flows, tables with level chips, cards, charts) as their starting point, restyled into the slide format.

### 8.3 Confidential badge

If a sample SOW was used, the hub shows "Contains sample client material" (the badge exists so the flow can be demoed as it would work with real documents).

### 8.4 Hubs and the library

Each pass through the wizard creates one hub, with its own stable, shareable link. "Switch role" adds that role's outputs and podcast to the same hub. The home page lists every hub (company, roles built, last updated), since there are no accounts.

### 8.5 Owner page

A page protected by an owner key (an environment variable; my call) holds the actions only the owner takes: "Run today's update" and "Rebuild hub" for any hub, and a cost view showing tokens, searches and dollars per run. Everything else needs no key.

## 9. Podcast

Follows the skill's `outputs/podcast.md` running order and style rules: about 2,200 words, two hosts (`HOST A`, `HOST B`), cold open, company in two minutes, industry and competitors, the role segment (from the role file's `## Podcast segment`), risks and signals, what to do this week. Written for the ear, sources named in speech, estimates said aloud.

Audio comes from a two-voice text-to-speech service on the owner's key. Because the script leaves the app for the speech service, SOW names, figures and contract terms are kept out of it, as v2 requires. The player shows chapters and the script. If audio fails, the hub shows the script and says why.

## 10. Update runs and the newsletter

**Update run.** Researches only the last 24 hours, for the client and its competitors: news and RSS TL;DRs, social posts, new job posts (careers page, Greenhouse, LinkedIn), stock moves for public companies, and creator content from the role-knowledge lists, all against the keyword watchlist. It writes the results into the hub: "What's new", the Signals data, the stock view, hiring numbers and competitor moves. Deeper sections change only on a full rebuild ("Rebuild hub").

**In this prototype, update runs are not scheduled.** They run only when the owner triggers one from the owner page (8.5). The demo runs one day's update.

**Newsletter.** Anyone can subscribe to a hub with an email address and a role, and can subscribe to several hubs. After an update run, each subscriber gets an email with v2's format: subject "<Company> daily: <the most important item>"; sections Top item, News, People's posts, New roles, What it means for you (written for the subscriber's role). If nothing is new, one line says so. Every email has an unsubscribe link. No SOW content ever goes into an email.

## 11. Architecture (builder may adjust with reason)

| Part | Choice |
|---|---|
| Web app | Next.js (App Router, TypeScript) on Vercel |
| Research runtime | Claude Managed Agents: Anthropic runs the long agent sessions, so Vercel only starts them and reads results. Ten agents as a multiagent session. Webhooks or polling tell the app when a run finishes |
| Models | Claude Sonnet 5.5 for the research agents and update runs; Claude Opus 5.5 for verification, the brief, the role outputs, the point-of-view drafts and the podcast script |
| Storage | Postgres for hubs, runs, records, subscribers; blob storage for documents and audio |
| Email | A transactional email service sending from the app's own address |
| Text to speech | A two-voice speech service on the owner's key |
| Containers | A Dockerfile and docker-compose that run the whole app locally with one command; no Kubernetes |
| Secrets | Environment variables only, never in the repo |

Before relying on Managed Agents, the builder confirms it supports the run length, web search and fetch, multiagent sessions and getting structured results back. If it doesn't, the fallback is a background job service that works with Vercel.

## 12. Design system

From method.com's own stylesheets, approved by the owner in `design-system-v1.html`:

- **Palette.** Black `#000000`; blue `#4442E3` with tint `#D9DCFA`; coral `#FF584E` with tint `#F9DEDA`; yellow `#FFB60C` with tint `#FDF1D0`; greys `#D9D9D9`, `#EDEDED`, `#646464`.
- **Roles for color.** Blue: primary actions, current step, selection. Coral: "check this" and attention (dots and backgrounds only, never text on white). Yellow: help. Added for contrast: coral-dark `#B8322A` for attention text, a quiet green for "confirmed".
- **Surfaces.** Black chrome (top bar, progress band, podcast card), light work surface. Dark mode switches to Method black throughout.
- **Type.** Clash Display for headings, Manrope 500 for body. No Druk Wide.
- **Shape.** Pill buttons and chips; 12 to 20 px card radii.
- **Accessibility.** WCAG AA contrast on every text pair, visible focus states, keyboard-usable wizard, works at phone width with no sideways scroll.

Mockups for reference: `.superpowers/mockups/flow-layout.html`, `design-system-v1.html`, `hub-v1.html`.

## 13. Left to the builder

- Database schema, API routes, component structure, chart and diagram libraries.
- Which email and speech services, within the constraints above.
- How progress is streamed to the research screen.
- Exact JSON shapes beyond what section 7 fixes.
- Test approach.

## 14. My calls (owner to check)

- Black chrome with a light work surface, rather than an all-black app.
- Ten agents, with 8 to 10 new.
- Role-knowledge source lists marked "my picks".
- Point-of-view drafts of the six transformation frameworks.
- A public home-page library of all hubs.
- Deeper sections refresh only on a full rebuild; update runs refresh signals, news, stock and hiring.
- Two-host speech service for every podcast, no voice question.
- Sonnet 5.5 for research, Opus 5.5 for synthesis.
- Managed Agents as the research runtime.
- An owner page behind an owner key for update runs, rebuilds and costs.
- One hub per wizard run; newsletter subscriptions attach to a hub; one podcast per role.
