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
2. **People will remix.** A meaningful minority of players change it (a colour, a name, a message, a speed), because doing so is fast, fun, and personal ("make it say Sam's name", "make it rain tacos").
3. **Remixes travel.** Remixed Things get sent onward and produce new players and remixers, so ideas move through chains of people.

If any part fails, there's nothing for a public library, API, or SDK to stand on. For that reason the MVP leaves all of them out.

## 3. Target users

### 3.1 MVP primary user: "the sender"
This person is roughly 16–35 and already sends GIFs, memes, stickers, and links in group chats (iMessage, WhatsApp, Discord, Telegram, Slack). They don't code. They'll spend 30–90 seconds personalising something if it gets a laugh. They mostly use a phone. **Design for a mobile browser opened from a chat app's in-app browser.**

They want a reaction, a birthday greeting, an inside joke, a small challenge ("beat my score"), or a decision ("spin the wheel for dinner").

### 3.2 MVP secondary user: "the tinkerer"
Creative coders and hobbyists. They'll make most of the early interesting Things, do most of the deep remixing, and do most of the seeding. They want direct control, so the MVP gives them a desktop source editor and a Blank seed.

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
| **Editor** | The mobile-first remix surface: knobs, undo, and publish. Desktop also gets a plain source editor. |

A Thing never changes after it's published, and "editing" always makes a new Thing. This has three benefits:

- Links stay trustworthy: what you sent is what they see.
- Caching is nearly free.
- Lineage is a simple tree.

## 5. The core loop in detail

```
  [Link in chat] → [Player: plays instantly] → [Remix] → [Editor: knobs / source]
         ▲                                                         │
         └──────────── [Share sheet / copy link] ← [Publish (no account)]
```

The loop's design targets:

- A shared link reaches its **first interactive frame within 2 s at p75** on a mid-range Android phone over 4G.
- A **knob-only remix takes under 60 s** median, from tapping Remix to having a shareable link.
- There are **zero accounts, installs, or sign-in walls** anywhere in the loop.

## 6. MVP scope (M0)

This is the one authoritative scope list. ROADMAP M0 references it and doesn't restate it differently.

### 6.1 In scope

1. **Player page** (`/t/{id}`):
   - Runs the Thing in a locked-down sandbox.
   - Shows the title and optional nickname.
   - Shows a compact lineage line ("remixed from *Taco Rain* ← *Rain* ← 🌱 seed", each item tappable), and the remix count.
   - Has Remix, Share, Restart, and Report buttons.
   - Has Open Graph metadata: a per-Thing `og:title`, plus a preview image inherited from the Thing's seed.
2. **Editor** (`/t/{id}/remix`):
   - A **Knobs panel** with a live preview. This is the primary tool.
   - **Undo/redo** history within the session.
   - An **error banner** if the preview throws: "Something broke. [Undo]".
   - A **source editor** on desktop only: a plain text area with Run. It's for tinkerers and for transparency.
3. **Publish without an account.** The user can add an optional title and nickname, then passes an invisible bot check. They get the native share sheet or copy-link. Shared links carry a referral token for measurement. **Anyone can publish, including recipients.**
4. **Seeds:** about 12 hand-made root Things on the home page, including a Blank seed (see §7.3).
5. **Self-delete:** the creator can delete from the same browser (cookie-based). A deleted Thing becomes a tombstone, so lineage stays intact.
6. **Safety basics:**
   - The sandbox.
   - A report button, which alerts the team.
   - Daily human review of every publish.
   - Removal by the team (tombstone plus cache purge).
   - A global kill switch for publishing.
7. **Abuse controls:** daily publish and report limits per browser and per network, and a bot check on the first publish.
8. **Measurement:** anonymous first-party events for the metrics in §10.

### 6.2 Deliberately deferred from M0

The critical review cut these from M0. Each is either unneeded to validate the loop, or costly or risky for the value it adds now. The table gives each one's target milestone.

| Deferred item | Why | Target |
|---|---|---|
| Web embed (`<iframe>` player for other sites) and oEmbed | Part of the long-term vision, but it doesn't test the chat loop. It also needs the sandbox's framing rules relaxed (ARCHITECTURE §5.2). | M1 |
| Per-Thing preview images / animated previews | Rendering them server-side is costly and complex (fonts, emoji, headless browsers). Seed images plus per-Thing titles are enough for unfurls. | M1 |
| List of a Thing's remixes and "what changed" diffs | They're nice for exploring lineage but don't affect V1–V4. The data is recorded from day one. | M1 |
| AI-assisted editing of any kind | Not needed to test the loop: knobs are the remix tool for senders, and the source editor covers tinkerers. It would add the only significant variable cost, a third-party dependency, and a new abuse surface (§8). | Optional experiment from M1, only if V8 and interviews show demand (ROADMAP M1) |
| Code editor with syntax highlighting | A plain text area is enough for tinkerers in the alpha. | M1 |
| Automated publish-time review, admin web UI | At alpha volume, daily human review plus scripts is more accurate and simpler. | M1 |
| Manage links / cross-browser delete | They put a credential in a URL. Cookie delete plus "email us" is enough. | M1, with accounts |
| Download as .html | A portability promise with no validation value yet. | M1 |

### 6.3 Out of scope for the MVP

The full list of non-goals is in §11. In short, the MVP has none of the following:

- **Social and discovery:** accounts, profiles, feeds, likes, comments, follows, and public search or browsing beyond the seed grid.
- **Media inside Things:** uploads, network access, persistent storage, and multiplayer.
- **AI-assisted creation or editing.**
- **Platform and business:** native apps, a public API, a picker, SDKs, and monetisation.

## 7. UX flows

### 7.1 Flow A: open a shared link (the most important flow)
1. The user taps the link in a chat, and it opens in the in-app browser.
2. The player shell renders immediately with the title. The Thing starts at once, because the user chose to open it.
3. The user plays (tap, drag, type). Sound starts only after a tap inside the Thing.
4. Around the Thing are **Remix** (primary), **Share**, **↺ Restart**, and **⋯ → Report**. Below it: "by Sam · remixed from *Taco Rain* · 14 remixes".
5. There's no sign-up prompt. The only overlay is a minimal notice, and only where the law requires it (see ARCHITECTURE §8.3).

**Failure states:**
- The Thing crashes or stops sending heartbeats: "This Thing stopped responding. [Restart] [Report]".
- The Thing was removed: a tombstone page with a link to its parent, if available.

### 7.2 Flow B: remix
1. The user taps **Remix**. The editor opens in place with the same Thing running, with no dialog.
2. The **Knobs** panel is open by default: sliders, colour swatches, text fields, and toggles. Every change updates the preview immediately.
3. On desktop, tinkerers can open the **source editor**, change the code, and tap Run. If the preview throws an error, the banner offers **Undo**.
4. The user taps **Publish**. A sheet offers an optional title (pre-filled) and an optional nickname, which is remembered. It includes this line: *"Published Things are public to anyone with the link and are credited in the lineage of the Thing you remixed."*
5. On success the share sheet opens (Web Share API), with copy-link as a fallback. The new Thing's page shows "remixed from …".

### 7.3 Flow C: start from a seed
The home page is a grid of seeds with one line explaining what this is. Tapping a seed opens it in the player, and **Remix** works exactly as in Flow B. The launch seeds cover the target uses:

- **Greeting:** an animated card with knobs for the message and colours.
- **Decision:** spin-the-wheel with editable options.
- **Challenge:** a reaction-time game that shows your score.
- **Toys:** an emoji physics pit, fireworks on tap, and pet-the-creature.
- **Oracle / joke:** a magic 8-ball with editable answers, and a "which X are you" quiz.
- **Instrument:** a four-pad synth soundboard.
- **Mini-games:** a one-button jumper and a pong-like game.
- **Blank:** an empty canvas with a message knob. Tinkerers start here with the source editor, which is how brand-new kinds of Things enter the lineage.

### 7.4 Flow D: follow the lineage
Tapping any item in the lineage line opens that ancestor's player page. The list of children arrives in M1.

### 7.5 Flow E: report and delete
- **Report:** found in the ⋯ menu. The user picks a reason (spam, hateful, sexual, harassment, deceptive or phishing, other) and can add text. The team is alerted. Reports don't hide anything automatically in M0, which prevents griefing.
- **Delete:** shown only in the browser that published the Thing. It asks for confirmation, then replaces the Thing with a "removed by its creator" tombstone.

## 8. Human agency, and why the MVP has no AI

Every change to a Thing in the MVP is made by a person, either by turning a knob or by editing source. There is no AI in the product.

The loop doesn't need AI to be tested:

- **Senders remix with knobs.** Changing a name, a message, a colour, the options on a wheel, or the emoji that falls is exactly the kind of personalisation the thesis bets on, and knobs deliver it instantly with no cost and no failure modes.
- **Seed design carries the weight.** Seeds are designed so their most remixable qualities are exposed as knobs (text, choices, emoji, colours, speeds). That is where the team's effort goes instead of AI.
- **Tinkerers extend the format by hand.** New kinds of Things come from people writing source, starting from the Blank seed or any existing Thing.

Leaving AI out removes the product's only significant variable cost and its only third-party runtime dependency. It also removes a whole abuse and failure surface: cost-exhaustion attacks, prompt injection, broken generated code, and model evaluation.

**When AI would be reconsidered.** Only if the alpha shows that people want changes knobs can't make. The signals are V8 (beyond-knob demand, §10.2) and interviews. If it comes back, it's as an optional M1 experiment that must beat knobs-only on V3 and V5 (ROADMAP M1), under constraints that keep humans in charge:

1. AI would only edit an existing Thing, one requested change at a time. It would never generate a finished Thing from nothing.
2. Every AI change would be a visible, undoable step, and nothing would publish without an explicit human Publish.
3. Any new tunable value the AI introduced would have to become a knob, so the next person could adjust it by hand.
4. Provenance would be recorded and shown honestly.
5. It would have no tools, network, or cross-user data, and its output would run in the same sandbox as everything else.

## 9. Differentiation

| Alternative | What it proves | How Things differs |
|---|---|---|
| GIFs / stickers (GIPHY, Tenor) | Tiny media spreads through chat when it's instant and searchable | Things are interactive and remixable, and personalising one takes seconds |
| Scratch | Remix culture with visible lineage works at scale | Things live in chat, work on mobile first, and are for all ages |
| CodePen / p5 editor / JSFiddle | People share tiny code artefacts | Those are account-first tools for people who already code. They aren't built for phones or chat. |
| AI app generators (prompt-to-app tools) | People want to make software by describing it | They produce *finished*, heavy, one-off apps with no lineage and no safe portable format. Things are tiny, human-made, and knob-first, with visible lineage, and are designed to be embedded safely. |
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
| V8 | **Beyond-knob demand** | Share of editor sessions that open the source editor, plus how often interviewees describe changes knobs couldn't make | Directional | report (informs whether AI or richer tools are worth an M1 experiment) |
| V9 | **Abuse rate** | Published Things removed for policy reasons | Guardrail | ≤ 2 % |

**Sample floor before judging:** at least 2,000 unique recipient viewers, reached through at least 20 distinct sharers. At the thresholds, that's about 160 remix starts and 55 published recipient remixes. It needs about 50–100 seed users (ROADMAP M0). Below the floor, the result is *inconclusive*, not a failure. Qualitative input comes from at least 10 interviews with senders and recipients.

### 10.3 Decision table (the only one; ROADMAP references it)

| Outcome | Condition | Action |
|---|---|---|
| **Pass** | V1–V4 all meet thresholds, and guardrails V7 and V9 are met | Proceed to M1 |
| **Iterate: editor** | V1 ≥ 60 %, but V2 or V3 below threshold | Up to 2 cycles (about 2 weeks each) of editor and seed changes, then re-measure |
| **Iterate: content** | 40 % ≤ V1 < 60 % | 1 cycle of seed, player, and first-frame work, then re-measure |
| **Iterate: travel** | V1–V3 pass, and 25 % ≤ V4 < 50 % | 1 cycle on sharing (unfurl copy, share prompts, seed choice), then re-measure |
| **Guardrail miss** | V7 or V9 misses | Fix before M1. This doesn't judge the thesis. |
| **Stop / rethink** | After the allowed cycles: V1 < 40 %, **or** V2 < 3 %, **or** V4 < 25 % | Rethink the primitive rather than adding features (ROADMAP §5) |

V5, V6, and V8 inform the M1 plan, seed choices, and whether to run an AI experiment. With this measurement method they aren't reliable enough to gate on.

## 11. Explicit non-goals (MVP)

Each is excluded because it doesn't help validate the loop, and it adds cost, risk, or both.

- **Accounts, profiles, follows, feeds, likes, comments, DMs.** The loop runs on links. Accounts would add friction exactly where the thesis needs none.
- **Public library, search, trending, recommendations.** A library turns link-only content into endorsed content, which needs moderation and content ratings at scale. It's gated to M2.
- **Public API, oEmbed, embed widget, picker, SDKs, partner integrations.** These are M1–M4. M0 only keeps the *contracts* stable (ARCHITECTURE §10).
- **User uploads** (images, audio, fonts, files). They'd bring CSAM/NCII and copyright exposure, moderation cost, and storage growth. Things use code-drawn visuals, emoji, system fonts, and synthesised sound.
- **Network access, persistent storage, or external libraries inside Things.** These are core to the safety model (ARCHITECTURE §5).
- **Multiplayer or real-time.** These need network access and servers.
- **AI-assisted creation or editing.** Not needed to test the loop (§8). At most an optional, gated M1 experiment.
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
2. **Creator Pro:** larger size caps, curated asset packs (fonts, sprites, sound kits), export, and custom embed styling.
3. **Branded / sponsored Things:** remixable brand seeds and stickers. This needs the library and ratings first.
4. **Education:** classroom spaces where students remix teacher seeds. This needs accounts and organisations.
5. **Business embeds:** interactive marketing content. Higher price, with a risk of pulling the product away from its consumer focus.

The key economic fact from the design is that **viewing, remixing, and publishing cost almost nothing per user** (ARCHITECTURE §11), so there's no pressure to monetise before the loop is proven. If an AI experiment ever ships, its usage would be the first thing worth charging for.

## 13. Risks and open questions

| Risk | Why it matters | Mitigation |
|---|---|---|
| **Remixing is too hard for senders on phones** | Kills thesis part 2 | Knobs are the whole mobile remix tool, and seeds are designed around them. The editor is mobile-first. Five non-coders test a prototype in M-1 (ROADMAP S2). |
| **In-app browsers break the experience** | Most opens happen there | Mandatory real-device test matrix in M-1: iMessage, WhatsApp, Instagram, Discord, and Telegram webviews |
| **Knobs are too shallow, so remixes feel samey** | Weak V2/V5; chains don't evolve | Seeds expose rich knobs (text, choices, emoji, several numeric controls). The source editor and Blank seed let tinkerers create new kinds of Things. V8 and interviews measure unmet demand, and an AI or richer-tools experiment is the M1 response if needed. |
| **Abuse: offensive or deceptive content drawn in code, phishing-style social engineering** | Legal, trust, and chat apps blocking links | Link-only distribution (no public feed) limits reach. Daily human review of all publishes, reports, fast removal, and a publish kill switch. Things can't make network requests, submit forms, or navigate the page. |
| **Residual data leak via WebRTC** | A hostile Thing could learn a viewer's IP address or leak text typed into it | Documented honestly (ARCHITECTURE §5.4). Mitigated but not eliminated. Things never have access to any data beyond their own inputs. |
| **Chat apps flag the domain** | Distribution dies | User content lives on a separate registrable domain, removal is quick, and abuse is kept low. Reputation monitoring starts in M1. |
| **Cold start: nothing worth remixing** | The loop never starts | Strong seeds plus the Blank seed. Seed users post into real group chats. |
| **Lineage exposes link-only Things** | Privacy surprise | Explicit wording at publish, no search, and self-delete |
| **Measurement is inflated or blocked** (webview cookie jars, consent rules) | Wrong decisions | Recipients are defined via share refs, V4–V6 are treated as directional, and a legal read on the viewer cookie is done before the alpha (ARCHITECTURE §8.3) |
| **Legal:** copyright in remixes, minors, privacy | Liability | ToS with a remix licence, a 13+ requirement, minimal data, and a DMCA process. Counsel reviews before the public beta. |

**Decisions needed from the approver:**
1. **Anonymous publishing** stays in M0 and M1 and is re-evaluated at the M1 gate using V9.
2. **Working name and two domains:** one for the product and one, separately registered, for user content.
