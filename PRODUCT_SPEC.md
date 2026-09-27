# Product Specification — "Things" (working title)

Status: reviewed draft, awaiting approval · Last updated: 2026-09-27
Companion documents: [ARCHITECTURE.md](ARCHITECTURE.md), [ROADMAP.md](ROADMAP.md). The pre-implementation review record is in ROADMAP §6.

---

## 1. One-line definition

A **Thing** is a tiny interactive creation (a toy, game, card, or gadget). It opens instantly from a link, and anyone can play with it in any browser without an account. Remixing one takes under a minute, and every remix remembers where it came from.

The aim is for Things to do for interactive content what GIFs did for animation: small, fast, safe to open, easy to send, and cheap to make a new version of.

## 2. Product thesis

The whole product is a bet on one loop:

> **Receive → Play → Remix → Send**

The thesis has three parts. Each can be tested, and the MVP exists only to test them:

1. **People will play.** Someone who gets a Thing link in a group chat opens it and interacts with it, at rates closer to opening a GIF than to installing an app.
2. **People will remix.** A meaningful minority of players change it (a knob, a line of text, an AI-assisted tweak), because doing so is fast, fun, and personal ("make it say Sam's name", "make it rain tacos").
3. **Remixes travel.** Remixed Things get sent onward and produce new players and remixers, so ideas move through chains of people.

If any part fails, there's nothing for a public library, API, or SDK to stand on. For that reason the MVP leaves all of them out.

## 3. Target users

### 3.1 MVP primary user: "the sender"
This person is roughly 16–35 and already sends GIFs, memes, stickers, and links in group chats (iMessage, WhatsApp, Discord, Telegram, Slack). They don't code. They'll spend 30–90 seconds personalising something if it gets a laugh. They mostly use a phone. **Design for a mobile browser opened from a chat app's in-app browser.**

They want a reaction, a birthday greeting, an inside joke, a small challenge ("beat my score"), or a decision ("spin the wheel for dinner").

### 3.2 MVP secondary user: "the tinkerer"
Creative coders and AI hobbyists. They'll make most of the early interesting Things, do most of the deep remixing, and do most of the seeding. They want direct control, so the MVP gives them a desktop source editor and a Blank seed.

**Honest expectation:** tinkerers will dominate early creation. The thesis only counts as confirmed if *recipients* (people who arrived through someone else's shared link) play and remix. §10 measures recipients separately so tinkerer activity can't hide a failure.

### 3.3 Later users (not MVP)
Later users are third-party apps that want an interactive "sticker/GIF" slot, plus educators and brands. They need a library, content ratings, an API, and an SDK. See ROADMAP M2–M4.

## 4. Core concepts

| Concept | Definition |
|---|---|
| **Thing** | An immutable, published interactive creation made of two parts. One is a small self-contained HTML document (≤ 32 KB), called its *source*. The other is a set of knob values. It gets a permanent short link. |
| **Knob** | A declared, typed parameter of a Thing (number, colour, text, toggle, choice, emoji). Anyone can change a knob with a simple control, without touching code. Knobs are the primary "simple tool." |
| **Remix** | A new Thing made by changing an existing one. It always records its parent. |
| **Lineage** | The chain of parents back to a seed, plus the tree of children. It's recorded permanently and never rewritten. |
| **Seed** | A root Thing authored by the team as a starting point. The launch set is about 12, including one *Blank* seed. |
| **Player** | The page that runs a Thing safely, with attribution and a Remix button. |
| **Editor** | The mobile-first remix surface: knobs, "Ask AI," undo, and publish. Desktop also gets a plain source editor. |

A Thing never changes after it's published, and "editing" always makes a new Thing. This has three benefits:

- Links stay trustworthy: what you sent is what they see.
- Caching is nearly free.
- Lineage is a simple tree.

## 5. The core loop in detail

```
  [Link in chat] → [Player: plays instantly] → [Remix] → [Editor: knobs / Ask AI]
         ▲                                                         │
         └──────────── [Share sheet / copy link] ← [Publish (no account)]
```

The loop's design targets:

- A shared link reaches its **first interactive frame within 2 s at p75** on a mid-range Android phone over 4G.
- A **knob-only remix takes under 60 s** median, from tapping Remix to having a shareable link.
- **An AI-assisted change returns within 20 s at p75**, and the user sees a clear progress state while waiting.
- There are **zero accounts, installs, or sign-in walls** anywhere in the loop.

## 6. MVP scope (M0)

This is the one authoritative scope list. ROADMAP M0 references it and doesn't restate it differently.

### 6.1 In scope

1. **Player page** (`/t/{id}`):
   - Runs the Thing in a locked-down sandbox.
   - Shows the title and optional nickname.
   - Shows a compact lineage line ("remixed from *Taco Rain* ← *Rain* ← 🌱 seed", each item tappable), the remix count, and an "AI-assisted" label if any AI edits were made.
   - Has Remix, Share, Restart, and Report buttons.
   - Has Open Graph metadata: a per-Thing `og:title`, plus a preview image inherited from the Thing's seed.
2. **Editor** (`/t/{id}/remix`):
   - A **Knobs panel** with a live preview. This is the primary tool.
   - **Ask AI:** one text box ("make the cat purple and add a score"). The AI applies that one change and summarises it in a line. Any new tunable values it adds are exposed as knobs. The user can Keep or Undo.
   - **Undo/redo** history within the session.
   - An **error banner** if the preview throws: "Something broke. [Undo] [Ask AI to fix]". The fix button just pre-fills Ask AI with the error.
   - A **source editor** on desktop only: a plain text area with Run. It's for tinkerers and for transparency.
3. **Publish without an account.** The user can add an optional title and nickname, then passes an invisible bot check. They get the native share sheet or copy-link. Shared links carry a referral token for measurement. **Anyone can publish, including recipients.**
4. **Seeds:** about 12 hand-made root Things on the home page, including a Blank seed (see §7.3).
5. **Self-delete:** the creator can delete from the same browser (cookie-based). A deleted Thing becomes a tombstone, so lineage stays intact.
6. **Safety basics:**
   - The sandbox.
   - A report button, which alerts the team.
   - Daily human review of every publish.
   - Removal by the team (tombstone plus cache purge).
   - Global kill switches for publishing and for AI.
7. **Cost controls:** daily AI limits per browser and per network, a cap on input size, and a global daily AI spend cap enforced before each call.
8. **Measurement:** anonymous first-party events for the metrics in §10.

### 6.2 Deliberately deferred from M0

The critical review cut these from M0. Each is either unneeded to validate the loop, or costly or risky for the value it adds now. The table gives each one's target milestone.

| Deferred item | Why | Target |
|---|---|---|
| Web embed (`<iframe>` player for other sites) and oEmbed | Part of the long-term vision, but it doesn't test the chat loop. It also needs the sandbox's framing rules relaxed (ARCHITECTURE §5.2). | M1 |
| Per-Thing preview images / animated previews | Rendering them server-side is costly and complex (fonts, emoji, headless browsers). Seed images plus per-Thing titles are enough for unfurls. | M1 |
| List of a Thing's remixes and "what changed" diffs | They're nice for exploring lineage but don't affect V1–V4. The data is recorded from day one. | M1 |
| AI idea chips, streaming AI output, one-tap auto-fix | Each adds calls, cost, or plumbing. Ask AI with a pre-filled prompt covers the need. | M1 (if data shows people stall) |
| Code editor with syntax highlighting | A plain text area is enough for tinkerers in the alpha. | M1 |
| Automated publish-time AI review, admin web UI | At alpha volume, daily human review plus scripts is more accurate and simpler. | M1 |
| Manage links / cross-browser delete | They put a credential in a URL. Cookie delete plus "email us" is enough. | M1, with accounts |
| Download as .html | A portability promise with no validation value yet. | M1 |

### 6.3 Out of scope for the MVP

The full list of non-goals is in §11. In short, the MVP has none of the following:

- **Social and discovery:** accounts, profiles, feeds, likes, comments, follows, and public search or browsing beyond the seed grid.
- **Media inside Things:** uploads, network access, persistent storage, and multiplayer.
- **Blank-page "prompt → finished app" generation.**
- **Platform and business:** native apps, a public API, a picker, SDKs, and monetisation.

## 7. UX flows

### 7.1 Flow A: open a shared link (the most important flow)
1. The user taps the link in a chat, and it opens in the in-app browser.
2. The player shell renders immediately with the title. The Thing starts at once, because the user chose to open it.
3. The user plays (tap, drag, type). Sound starts only after a tap inside the Thing.
4. Around the Thing are **Remix** (primary), **Share**, **↺ Restart**, and **⋯ → Report**. Below it: "by Sam · remixed from *Taco Rain* · 14 remixes · AI-assisted".
5. There's no sign-up prompt. The only overlay is a minimal notice, and only where the law requires it (see ARCHITECTURE §8.3).

**Failure states:**
- The Thing crashes or stops sending heartbeats: "This Thing stopped responding. [Restart] [Report]".
- The Thing was removed: a tombstone page with a link to its parent, if available.

### 7.2 Flow B: remix
1. The user taps **Remix**. The editor opens in place with the same Thing running, with no dialog.
2. The **Knobs** panel is open by default: sliders, colour swatches, text fields, and toggles. Every change updates the preview immediately.
3. Optionally, the user types into **Ask AI**. A progress state shows while it works. The result shows the AI's one-line summary with **Keep / Undo**, and any new knobs appear in the panel.
4. If the preview throws an error, the banner shows **Undo** and **Ask AI to fix**.
5. The user taps **Publish**. A sheet offers an optional title (pre-filled) and an optional nickname, which is remembered. It includes this line: *"Published Things are public to anyone with the link and are credited in the lineage of the Thing you remixed."*
6. On success the share sheet opens (Web Share API), with copy-link as a fallback. The new Thing's page shows "remixed from …".

### 7.3 Flow C: start from a seed
The home page is a grid of seeds with one line explaining what this is. Tapping a seed opens it in the player, and **Remix** works exactly as in Flow B. The launch seeds cover the target uses:

- **Greeting:** an animated card with knobs for the message and colours.
- **Decision:** spin-the-wheel with editable options.
- **Challenge:** a reaction-time game that shows your score.
- **Toys:** an emoji physics pit, fireworks on tap, and pet-the-creature.
- **Oracle / joke:** a magic 8-ball with editable answers, and a "which X are you" quiz.
- **Instrument:** a four-pad synth soundboard.
- **Mini-games:** a one-button jumper and a pong-like game.
- **Blank:** an empty canvas with a message knob. Tinkerers start here, either with the source editor or by building step by step with Ask AI. Lineage stays honest either way.

### 7.4 Flow D: follow the lineage
Tapping any item in the lineage line opens that ancestor's player page. The list of children arrives in M1.

### 7.5 Flow E: report and delete
- **Report:** found in the ⋯ menu. The user picks a reason (spam, hateful, sexual, harassment, deceptive or phishing, other) and can add text. The team is alerted. Reports don't hide anything automatically in M0, which prevents griefing.
- **Delete:** shown only in the browser that published the Thing. It asks for confirmation, then replaces the Thing with a "removed by its creator" tombstone.

## 8. Role of AI: accelerator, not author

AI's job is to shorten the distance between a human's idea and a working change. The human stays in charge of *what* changes and *when*. In the MVP that means six rules:

1. **AI edits; it doesn't originate.** Every AI call starts from an existing Thing (a seed, the Blank seed, or someone's creation) and applies one requested change. There's no "generate a whole app" button. Building from Blank is possible, but it's step-by-step and the human directs each step.
2. **Every AI change is a visible, reversible step.** Each shows a one-line summary with Keep / Undo. Nothing gets published without an explicit Publish.
3. **AI turns its work into human controls.** When the AI introduces a tunable value (speed, colour, text, count), it must expose it as a knob. The next person can then tune it by hand, without AI.
4. **Provenance is honest.** Each Thing records how it was made: counts of knob, AI, and hand edits, plus the AI's change summaries. M0 shows an "AI-assisted" label; the per-step summaries show in M1. The user's raw prompts are never published.
5. **Usage is bounded, and AI is optional.** There are daily allowances and a global spend cap. When AI is unavailable, knobs and the source editor still work.
6. **AI has no powers beyond text.** It has no tools, no network, and no access to other users' data. It reads one Thing and writes one draft, which runs in the same sandbox as everything else.

## 9. Differentiation

| Alternative | What it proves | How Things differs |
|---|---|---|
| GIFs / stickers (GIPHY, Tenor) | Tiny media spreads through chat when it's instant and searchable | Things are interactive and remixable, and personalising one takes seconds |
| Scratch | Remix culture with visible lineage works at scale | Things live in chat, work on mobile first, are for all ages, and have AI assistance |
| CodePen / p5 editor / JSFiddle | People share tiny code artefacts | Those are account-first tools for people who already code. They aren't built for phones or chat. |
| AI app generators (prompt-to-app tools) | People want to make software by describing it | They produce *finished*, heavy, one-off apps with no lineage and no safe portable format. Things are tiny, knob-first, and human-steered, and are designed to be embedded safely. |
| Social video remix (duets, templates) | Remixing is a native social behaviour | Things are interactive rather than watch-only, and they're an open, embeddable format rather than locked inside one app |

The defensible asset isn't the editor. It's two other things:

- **The format and runtime:** a safe, portable, tiny interactive unit with stable link and embed contracts.
- **The remix graph:** a growing body of Things and their lineage.

M0 builds the foundations of both and nothing more.

## 10. Validation criteria

### 10.1 Definitions
These are made precise so they can actually be measured without accounts (ARCHITECTURE §8.3):

- **Viewer:** an anonymous first-party browser identifier set on first view. Chat apps' in-app browsers each keep separate cookies, so one person can show up as several viewers. That inflates viewer counts and K, which is why V4–V6 are treated as directional.
- **Share ref:** a random token created whenever someone uses Share or copy-link, and added to the link (`?r=`).
- **Recipient view:** a player view with a share ref where the viewer is neither the sharer nor the Thing's creator.
- **Interaction:** the first pointer or key event inside the Thing, reported by the runtime.

### 10.2 Metrics
Thresholds are **starting hypotheses**, not benchmarks. They get recalibrated once, after two weeks of alpha data, and the change is recorded in ROADMAP §6.

| # | Metric | Definition | Role | M0 threshold |
|---|---|---|---|---|
| V1 | **Play rate** | Unique recipient viewers who interact within 30 s ÷ unique recipient viewers | Primary | ≥ 60 % |
| V2 | **Remix start** | Unique recipient viewers who open the editor ÷ unique recipient viewers | Primary | ≥ 8 % |
| V3 | **Remix completion** | Editor sessions that publish ÷ editor sessions (reported for recipients and for all) | Secondary | ≥ 35 % |
| V4 | **Onward travel** | Published Things with ≥ 1 recipient view within 72 h ÷ published Things | Secondary | ≥ 50 % |
| V5 | **Chain depth** | First-generation remixes of seeds whose descendants reach two further generations, involving ≥ 2 distinct creators ÷ first-generation remixes | Directional | report (hope ≥ 10 %) |
| V6 | **Viral coefficient (K)** | New recipient viewers attributed through share refs to Things published by a viewer, per publishing viewer, over 14 days | Directional | report (≥ 0.5 is a signal) |
| V7 | **Time to first frame** | p75 from navigation start to the Thing's `ready`, on mobile | Guardrail | ≤ 2 s |
| V8 | **AI cost per published Thing** | Total AI spend ÷ published Things, *including* abandoned sessions | Guardrail | ≤ $0.15 |
| V9 | **Abuse rate** | Published Things removed for policy reasons | Guardrail | ≤ 2 % |

**Sample floor before judging:** at least 2,000 unique recipient viewers, reached through at least 20 distinct sharers. At the thresholds, that's about 160 remix starts and 55 published recipient remixes. It needs about 50–100 seed users (ROADMAP M0). Below the floor, the result is *inconclusive*, not a failure. Qualitative input comes from at least 10 interviews with senders and recipients.

### 10.3 Decision table (the only one; ROADMAP references it)

| Outcome | Condition | Action |
|---|---|---|
| **Pass** | V1–V4 all meet thresholds, and guardrails V7–V9 are met | Proceed to M1 |
| **Iterate: editor** | V1 ≥ 60 %, but V2 or V3 below threshold | Up to 2 cycles (about 2 weeks each) of editor and seed changes, then re-measure |
| **Iterate: content** | 40 % ≤ V1 < 60 % | 1 cycle of seed, player, and first-frame work, then re-measure |
| **Iterate: travel** | V1–V3 pass, and 25 % ≤ V4 < 50 % | 1 cycle on sharing (unfurl copy, share prompts, seed choice), then re-measure |
| **Guardrail miss** | V7, V8, or V9 miss | Fix before M1. This doesn't judge the thesis. |
| **Stop / rethink** | After the allowed cycles: V1 < 40 %, **or** V2 < 3 %, **or** V4 < 25 % | Rethink the primitive rather than adding features (ROADMAP §5) |

V5 and V6 inform the M1 plan and seed choices. With this measurement method they aren't reliable enough to gate on.

## 11. Explicit non-goals (MVP)

Each is excluded because it doesn't help validate the loop, and it adds cost, risk, or both.

- **Accounts, profiles, follows, feeds, likes, comments, DMs.** The loop runs on links. Accounts would add friction exactly where the thesis needs none.
- **Public library, search, trending, recommendations.** A library turns link-only content into endorsed content, which needs moderation and content ratings at scale. It's gated to M2.
- **Public API, oEmbed, embed widget, picker, SDKs, partner integrations.** These are M1–M4. M0 only keeps the *contracts* stable (ARCHITECTURE §10).
- **User uploads** (images, audio, fonts, files). They'd bring CSAM/NCII and copyright exposure, moderation cost, and storage growth. Things use code-drawn visuals, emoji, system fonts, and synthesised sound.
- **Network access, persistent storage, or external libraries inside Things.** These are core to the safety model (ARCHITECTURE §5).
- **Multiplayer or real-time.** These need network access and servers.
- **One-shot "prompt → finished Thing" generation.** This is a deliberate human-agency and cost choice, re-tested as an experiment in M2.
- **Editing published Things in place, or versioning.** A change is always a remix.
- **Native apps, browser extensions, keyboard apps.**
- **Monetisation.**
- **Collaboration, teams, private or password-protected Things.**
- **Creator analytics.** Creators see only the remix count.
- **Localisation beyond English UI**, beyond Things rendering any Unicode text.
- **Crypto, NFTs, tokens.**

## 12. Monetisation possibilities (post-validation; none in the MVP)

These are listed so the architecture doesn't block them, not as commitments. They're ordered by plausibility.

1. **Platform/API licensing (GIPHY-style):** a free tier with attribution requirements, and paid tiers or revenue share for high-volume platforms that want the picker, ratings, and SLAs.
2. **Creator Pro:** higher AI allowances, larger size caps, curated asset packs (fonts, sprites, sound kits), export, and custom embed styling.
3. **Branded / sponsored Things:** remixable brand seeds and stickers. This needs the library and ratings first.
4. **Education:** classroom spaces where students remix teacher seeds. This needs accounts and organisations.
5. **Business embeds:** interactive marketing content. Higher price, with a risk of pulling the product away from its consumer focus.

The key economic fact from the design is that **everything except AI costs almost nothing per user** (ARCHITECTURE §11). Monetisation mainly needs to cover AI, which makes AI allowances the natural Pro lever.

## 13. Risks and open questions

| Risk | Why it matters | Mitigation |
|---|---|---|
| **Remixing is too hard for senders on phones** | Kills thesis part 2 | Knobs come first and AI is optional. The editor is mobile-first. Five non-coders test a prototype in M-1 (ROADMAP S3). |
| **In-app browsers break the experience** | Most opens happen there | Mandatory real-device test matrix in M-1: iMessage, WhatsApp, Instagram, Discord, and Telegram webviews |
| **AI edits are broken or poor** | Frustration and cost | A pre-launch eval with a pass bar (ROADMAP S2), small scoped edits, undo, and Ask-AI-to-fix |
| **AI costs outgrow value** | Burn | Knob remixes are free. There are per-browser and per-network allowances, a global spend cap reserved before each call, and an input size cap. The model is chosen by eval and V8. V8 is tracked including abandoned sessions. |
| **Abuse: offensive or deceptive content drawn in code, phishing-style social engineering** | Legal, trust, and chat apps blocking links | Link-only distribution (no public feed) limits reach. Daily human review of all publishes, reports, fast removal, and kill switches. Things can't make network requests, submit forms, or navigate the page. |
| **Residual data leak via WebRTC** | A hostile Thing could learn a viewer's IP address or leak text typed into it | Documented honestly (ARCHITECTURE §5.4). Mitigated but not eliminated. Things never have access to any data beyond their own inputs. |
| **Chat apps flag the domain** | Distribution dies | User content lives on a separate registrable domain, removal is quick, and abuse is kept low. Reputation monitoring starts in M1. |
| **Cold start: nothing worth remixing** | The loop never starts | Strong seeds plus the Blank seed. Seed users post into real group chats. |
| **Lineage exposes link-only Things** | Privacy surprise | Explicit wording at publish, no search, and self-delete |
| **Measurement is inflated or blocked** (webview cookie jars, consent rules) | Wrong decisions | Recipients are defined via share refs, V4–V6 are treated as directional, and a legal read on the viewer cookie is done before the alpha (ARCHITECTURE §8.3) |
| **Legal:** copyright in remixes, minors, privacy | Liability | ToS with a remix licence, a 13+ requirement, minimal data, and a DMCA process. Counsel reviews before the public beta. |

**Decisions needed from the approver:**
1. **Edit model.** The rule proposed everywhere is *the cheapest model that passes the M-1 quality eval and keeps V8 ≤ $0.15*. By the cost model (ARCHITECTURE §11), only Haiku- or Sonnet-class models fit V8. Opus-class models fit only if the approver raises V8 to about $0.25–0.35.
2. **Anonymous publishing** stays in M0 and M1 and is re-evaluated at the M1 gate using V9.
3. **Working name and two domains:** one for the product and one, separately registered, for user content.
