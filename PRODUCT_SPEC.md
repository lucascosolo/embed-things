# Product Specification — "Things" (working title)

Status: draft for approval · Last updated: 2026-09-27
Companion documents: [ARCHITECTURE.md](ARCHITECTURE.md), [ROADMAP.md](ROADMAP.md)

---

## 1. One-line definition

A **Thing** is a tiny interactive creation (a toy, a game, a card, a gadget) that opens instantly from a link, can be played with in any browser without an account, and can be remixed in under a minute, with each remix remembering where it came from.

Things should be to interactive content what GIFs are to animation: small, fast, safe to open, easy to send, and cheap to make a new version of.

## 2. Product thesis

The whole product is a bet on one loop:

> **Receive → Play → Remix → Send**

The thesis has three parts. Each can be tested, and the MVP exists only to test them:

1. **People will play.** Someone who gets a Thing link in a group chat will open it and interact with it, at rates much closer to opening a GIF than to installing an app.
2. **People will remix.** A meaningful minority of those players will change it (a knob, a line of text, an AI-assisted tweak), because it is fast, fun, and personal ("make it say Sam's name", "make it rain tacos").
3. **Remixes travel.** Remixed Things get sent onward and produce more players and remixers, so ideas move through chains of people instead of stopping at the first creator.

If any of the three fails, a public library, API, and SDKs have nothing to be built on. So the MVP leaves all of that out.

## 3. Target users

### 3.1 MVP primary user: "the sender"
Someone aged roughly 16–35 who already sends GIFs, memes, stickers, and links in group chats (iMessage, WhatsApp, Discord, Telegram, Slack). They don't write code. They'll spend 30–90 seconds personalising something if it makes a friend laugh. Their phone is the main device: **assume a mobile browser opened from a chat app's in-app browser.**

What they want: a reaction, a birthday greeting, an inside joke, a tiny challenge ("beat my score"), a decision ("spin the wheel for dinner").

### 3.2 MVP secondary user: "the tinkerer"
Creative coders, AI hobbyists, and people who already build toys with AI tools. They'll make most of the first interesting Things and the deep remixes, and they'll be the early seeders. They want direct control, which means a code view and precise knobs.

**Honest expectation:** the first several hundred active creators will be tinkerers. The thesis is only confirmed if *senders* (non-coders who received a link) remix at a meaningful rate. The metrics in §10 are split by entry path so tinker-heavy data can't hide a failure with senders.

### 3.3 Later users (not MVP)
Third-party apps that want an interactive "sticker/GIF" slot (chat apps, social tools, Notion-like editors, community platforms), educators, and brands. They depend on a library, content ratings, an API, and an SDK. See ROADMAP M3–M4.

## 4. Core concepts

| Concept | Definition |
|---|---|
| **Thing** | An immutable, published interactive creation: one small self-contained HTML document (≤ 64 KB) plus a set of knob values. It has a permanent short link. |
| **Knob** | A declared, typed parameter of a Thing (number, colour, text, toggle, choice, emoji) that anyone can change with a simple control, without touching code. Knobs are the main "simple tool." |
| **Remix** | A new Thing made by changing an existing one. It always records its parent. |
| **Ancestry / lineage** | The chain of parents back to the root, plus the tree of children. Always visible, and never rewritten. |
| **Seed** | A root Thing authored by the team as a starting point (about 12 at launch). |
| **Player** | The page or embed that runs a Thing safely, with attribution and a Remix button. |
| **Editor** | The mobile-first remix surface: knobs, "Ask AI," undo, and publish. Desktop adds a code view. |

A Thing never changes after publishing. "Editing" always means making a new Thing. That keeps links trustworthy (what you sent is what they see), makes caching nearly free, and makes ancestry a simple tree.

## 5. The core loop in detail

```
      ┌────────────────────────────────────────────────────────┐
      │                                                        │
  [Link in chat] → [Player: plays instantly] → [Remix] → [Editor: knobs / Ask AI]
                                                                │
                           [Share sheet / copy link] ← [Publish (no account)]
```

The loop's design targets are these:

- A shared link shows the **first interactive frame within 2 s at p75** on a mid-range Android phone over 4G.
- A **knob-only remix takes under 60 s** median from tapping Remix to having a shareable link.
- **An AI-assisted change returns within 20 s at p75**, and the player sees progress while it runs.
- **Zero accounts, installs, or sign-in walls** anywhere in the loop.

## 6. MVP scope

The MVP is the smallest product that can run the loop end to end with real people from real group chats.

### 6.1 In scope

1. **Player page** (`/t/{id}`): runs the Thing in a locked-down sandbox. It shows the title, optional creator nickname, a "remixed from …" line, a remix count, and Remix / Share / Restart buttons. Open Graph and Twitter card metadata make links unfurl in chat apps.
2. **Embed player** (`/e/{id}`): the same player with minimal chrome and click-to-play, plus a "copy embed code" button that gives a single `<iframe>` snippet. It costs almost nothing because it reuses the player. There's no oEmbed, no SDK, and no parameters beyond what's listed in ARCHITECTURE §9.
3. **Editor** (`/t/{id}/remix`):
   - **Knobs panel** with a live preview. This is the primary tool.
   - **Ask AI**: a single text box ("make the cat purple and add a score"). The AI edits the current Thing, summarises the change in one line, and adds new tunable values as knobs. The user can Keep or Undo each change.
   - **Remix idea chips**: three suggested changes per Thing, generated once and cached.
   - **Undo/redo** step history within the session.
   - **Code view**: read and edit on desktop, read-only on mobile. It's for tinkerers and for transparency.
   - **Auto-fix**: if a change makes the Thing throw an error, the user gets a one-tap "fix it" retry that sends the error to the AI.
4. **Publish without an account**: optional title and nickname, an invisible bot check, and back comes a short link, the native share sheet, copy link, and copy embed.
5. **Ancestry**: every Thing shows its parent chain (the last 3 ancestors, expandable) and a list of direct remixes. Each step shows **what changed** (knob diffs, plus AI change summaries, plus a "code edited" flag).
6. **Seeds**: about 12 hand-made root Things on the home page, chosen to cover the target use cases (see §7.3).
7. **Delete your own Thing**: a creator-only manage link, also remembered in a cookie. Deleted Things become tombstones so the chain stays intact.
8. **Safety basics**: publish-time automated review, a report button on every Thing, a small admin queue, tombstoning, and global kill switches for publishing and for AI.
9. **Instrumentation**: anonymous product events for the metrics in §10.

### 6.2 Explicitly out of scope for the MVP (see §11 for the full non-goals list)
There are no accounts, profiles, feeds, likes, comments, or follows. There's no public search or browsable library beyond the curated seed shelf. There's no image, audio, or file upload, no network access or persistent storage inside Things, no multiplayer, and no blank-page "prompt → finished app" generation. There are no native apps, public API, oEmbed, picker, SDK, or monetisation.

## 7. UX flows

### 7.1 Flow A: open a shared link (the most important flow)
1. The user taps the link in a chat and it opens in the in-app browser.
2. The player shell renders immediately with the title and a poster card. The Thing starts running automatically, because the user explicitly navigated to it.
3. The user interacts (tap, drag, type).
4. Persistent but unobtrusive actions are available: **Remix** (the primary button), **Share**, and **↺ Restart**. Under the Thing it says "by Sam · remixed from *Taco Rain* · 14 remixes."
5. No cookie wall and no sign-up prompt. A single short consent banner appears only where legally required.

**Failure states:** if the Thing crashes or stops responding, the user sees "This Thing stopped responding. Restart / Report." If the Thing was removed, a tombstone links to its parent if one is available.

### 7.2 Flow B: remix
1. The user taps **Remix** and the editor opens in place with the same Thing already running. No account, no dialog.
2. The **Knobs** tab is open by default: sliders, colour swatches, text fields, and toggles with the declared labels. Every change updates the preview instantly.
3. Optionally, the user taps an idea chip or types into **Ask AI**. A progress state shows ("Changing… adding a score counter"). When the result arrives, it shows a one-line summary with **Keep** / **Undo**. Any new tunables the AI added appear as knobs.
4. If the preview throws an error, a banner appears: "Something broke. [Fix it] [Undo]."
5. The user taps **Publish**. A sheet asks for an optional title (pre-filled) and an optional nickname (remembered). A line of copy says: *"Published Things are public to anyone with the link and appear in the remix tree of the Thing you remixed."*
6. On success the native share sheet opens (Web Share API), with copy-link and copy-embed as fallbacks. The new Thing's page shows "remixed from …".

### 7.3 Flow C: start from a seed
The home page is a grid of about 12 seeds, plus a short "what is this?" line and a hand-curated "made by people" shelf (not algorithmic). Tapping a seed opens it in the player, and **Remix** works exactly as in Flow B. The launch seeds are chosen to cover the target uses:

- **Greeting / message:** an animated birthday card whose text and colours are knobs.
- **Decision:** a spin-the-wheel with editable options.
- **Challenge:** a reaction-time or tap-speed game that shows your score as text you can send.
- **Toy / fidget:** a physics emoji pit, fireworks on tap, and a pet-the-creature clicker.
- **Oracle / joke:** a magic 8-ball with editable answers, and a "which X are you" quiz.
- **Instrument:** a four-pad soundboard (synthesised sounds only).
- **Mini-game:** single-button jump and a pong-like game.

### 7.4 Flow D: explore the lineage
From any Thing, the user can tap "remixed from X" to go to the parent, or "14 remixes" to see its direct children (title, nickname, what changed). The children list only shows Things that are not held, removed, or deleted.

### 7.5 Flow E: embed
The user taps **Share → Embed** to copy `<iframe src="https://<player-domain>/e/{id}" …>`. The embed shows the poster with ▶, runs on tap, and always shows attribution plus a "Remix" link that opens the full editor in a new tab.

### 7.6 Flow F: report and delete
**Report** lives in the ⋯ menu. The user picks a reason (spam, hateful, sexual, harassment, deceptive or phishing, other) with optional text. **Delete** appears only to the creator (via cookie or manage link) and asks for confirmation. The Thing becomes a tombstone "removed by its creator."

## 8. Role of AI: accelerator, not author

AI's job is to shorten the distance between a human's idea and a working change, while keeping the human in charge of *what* changes and *when*. The MVP makes this concrete with seven rules:

1. **AI edits; it does not originate.** Every AI call starts from an existing Thing, either a seed or someone's creation, and applies one requested change. The MVP has no blank-canvas "generate an app" path. (This is a deliberate hypothesis. ROADMAP M2 re-tests it.)
2. **Every AI change is a visible, reversible step.** It shows a one-line summary and offers Keep / Undo. Nothing is published without an explicit human Publish.
3. **AI turns its work into human controls.** When the AI introduces a tunable value (speed, colour, text, count), it has to expose it as a knob, so the next person can tune it by hand without AI.
4. **Humans get ideas, not answers.** Idea chips suggest *small* changes the person can take, change, or ignore.
5. **Provenance is honest.** Each Thing records how it was made (knob changes, AI-assisted changes, hand code edits) and shows AI change summaries in the lineage. It doesn't publish the user's raw prompt text.
6. **Bounded usage.** Each device gets a daily AI-edit allowance and there's a global daily spend cap. When AI is unavailable, knobs and code still work. AI is an accelerator, not a dependency.
7. **AI has no powers beyond text.** It has no tools, no network, and no access to other users' data. It reads one Thing and writes one Thing, which then runs in the same sandbox as any other.

## 9. Differentiation

| Alternative | What it proves | Why Things is different |
|---|---|---|
| GIFs / stickers (GIPHY, Tenor) | Tiny media spreads through chat when it's instant and searchable | Things are interactive and remixable, and personalising one takes seconds |
| Scratch | Remix culture with visible ancestry works and scales | Things live in chat, are mobile-first, are for adults as well as kids, and have AI assistance |
| CodePen / p5 editor / JSFiddle | Tiny code artefacts get shared | Those are tools for developers who already code, need an account to save, and aren't built for phones or chat |
| AI app generators (prompt-to-app tools) | People want to make software by describing it | Those generate *finished*, heavy, one-off apps with no lineage and no portable safe format. Things are tiny, knob-first, human-steered, and designed to be embedded safely |
| Social video remix (duets, templates) | Remix is a native social behaviour | Things are interactive rather than watch-only, and they're an open, embeddable format rather than locked inside one app |

The **defensible asset** isn't the editor. It's the *format and runtime* (a safe, portable, tiny interactive unit with a stable embed contract) plus the *remix graph* (a growing corpus of Things and their lineage). The MVP builds both foundations without building the ecosystem on top.

## 10. Validation criteria

All metrics are measured per cohort and **split by entry path**: people who first arrived via a shared link ("recipients") vs. via the home page or seeds ("direct"). The thesis is judged on recipients.

The thresholds below are **starting hypotheses**, to be recalibrated after the first 2 weeks of alpha data. They're not based on external benchmarks.

| # | Metric | Definition | M0 pass threshold |
|---|---|---|---|
| V1 | **Play rate** | Recipient opens with ≥ 1 interaction within 30 s | ≥ 60 % |
| V2 | **Remix start rate** | Recipient opens that tap Remix | ≥ 8 % |
| V3 | **Remix completion** | Remix sessions that publish | ≥ 35 % |
| V4 | **Onward share** | Published Things opened by ≥ 1 other device within 72 h | ≥ 50 % |
| V5 | **Chain depth** | Root Things (seeds excluded) reaching depth ≥ 3 with ≥ 2 distinct creators | ≥ 10 % |
| V6 | **Viral coefficient (K)** | New recipient devices generated per publishing device over 14 days | ≥ 0.5 (signal); ≥ 1.0 (strong) |
| V7 | **Time to first frame** | p75, player page, mobile | ≤ 2 s |
| V8 | **AI unit cost** | AI spend ÷ published Things | ≤ $0.15 |
| V9 | **Abuse rate** | Published Things removed for policy reasons | ≤ 2 % |

**Minimum sample before judging:** 1,000 recipient opens and 150 published remixes. Below that, the result counts as inconclusive, not a failure.

**Interpretation rules:**
- **Pass:** V1–V4 are met and V6 ≥ 0.5 → proceed to M1.
- **Partial:** V1 is met but V2 or V3 misses → iterate on the editor (at most 2 cycles of about 2 weeks) before deciding.
- **Fail:** V1 < 40 % *or* V2 < 3 % after 2 iteration cycles → stop and rethink the primitive, not the features. See ROADMAP §5.

## 11. Explicit non-goals (MVP)

Each item is excluded because it doesn't help validate the loop, and it adds cost, risk, or both.

- **Accounts, profiles, follows, feeds, likes, comments, DMs.** Social-network features come later; the loop runs on links. Accounts would add friction exactly where the thesis needs none.
- **A public searchable library, trending, or recommendations.** A library turns unlisted content into platform-endorsed content and needs moderation at that scale plus content ratings. It's ROADMAP M2, gated on the loop working.
- **Public API, oEmbed, picker, SDKs, partner integrations.** These are ROADMAP M3–M4. The MVP only keeps their *contracts* stable (URL shapes, format versioning); see ARCHITECTURE §10.
- **User uploads (images, audio, fonts, files).** They bring CSAM/NCII and copyright exposure, moderation costs, and storage growth. Things use code-drawn visuals, emoji, system fonts, and synthesised sound.
- **Network access, persistent storage, or external libraries inside Things.** These are the core of the safety model (ARCHITECTURE §5). No CDN imports, no fetch, no localStorage.
- **Multiplayer or real-time features.** They need network access and servers.
- **Blank-canvas "prompt → finished Thing."** This is a deliberate human-agency and cost choice, re-tested in M2.
- **Editing a published Thing in place, or versioning.** Things are immutable. A change is a remix.
- **Native apps, browser extensions, keyboard apps.**
- **Monetisation of any kind.**
- **Real-time collaboration, teams, private or password-protected Things.**
- **Creator analytics dashboards.** Creators see only a remix count.
- **Localisation beyond English, beyond making sure Things can render any Unicode text.**
- **Crypto, NFTs, tokens.**

## 12. Monetisation possibilities (post-validation; none in the MVP)

These are listed so the architecture doesn't block them, not as commitments. Ordered from most to least plausible:

1. **Platform/API licensing (the GIPHY-style model).** A free API with attribution requirements for small integrators, and paid tiers or revenue share for high-volume platforms that want the picker, ratings, and SLAs.
2. **Creator Pro.** Higher AI allowances, larger size caps, curated asset packs (fonts, sprite sets, sound kits), download/export, private or unlisted-forever Things, and custom embed styling. Low price, high volume.
3. **Branded / sponsored Things.** Interactive branded stickers and seeds that people remix (GIPHY's historical revenue path). This needs the library and ratings first.
4. **Education.** Classroom workspaces where students remix teacher seeds, with safe defaults. Needs accounts and organisations.
5. **Embeds for marketing/interactive content** (quizzes, configurators) on business sites. Higher price, with a risk of pulling the product away from its consumer focus.

The key economic fact from the MVP design is that **everything except AI costs almost nothing per user** (ARCHITECTURE §11). Monetisation mainly has to cover AI usage, which is why AI allowances are the natural Pro lever.

## 13. Risks and open questions

| Risk | Why it matters | MVP mitigation |
|---|---|---|
| **Remixing is too hard for senders on phones** | Kills thesis part 2 | Knobs are the primary tool, AI is optional, the editor is mobile-first, and remixes are tested with real users in the first alpha week |
| **In-app browsers break the experience** (iOS/Android chat webviews) | Most opens happen there | Mandatory test matrix in the sandbox spike (ROADMAP M-1): iMessage, WhatsApp, Instagram, Discord, Telegram webviews |
| **The AI makes broken or low-quality edits** | Frustration, cost | Pre-launch eval with a pass bar; auto-fix; undo; small scoped edits |
| **AI costs grow faster than value** | Burn | Knob remixes are free; per-device limits; global daily cap; model choice driven by eval; cost per published Thing tracked (V8) |
| **Abuse: phishing-style Things, hate, harassment, sexual content drawn in code** | Legal, trust, and chat-app link blocking | No network or forms means no data exfiltration; publish-time automated review; reports; tombstones; kill switches; link-only distribution (no public feed) limits reach |
| **Chat apps flag or block the domain** | Distribution dies | Separate user-content domain; quick takedown; abuse rate kept low; Safe Browsing monitoring in M1 |
| **Cold start: nothing worth remixing** | Loop never starts | Strong seed set; founders and tinkerers seed real group chats; curated shelf |
| **"Unlisted" Things are exposed through the lineage** | Privacy surprise | Explicit copy at publish; no search; users can delete |
| **Legal: copyright in remixes, minors, GDPR** | Liability | ToS grants a remix licence; 13+ age requirement; minimal personal data; DMCA process; legal review before public beta (M1) |

**Open questions for the approver:**
1. **AI model choice.** The default is the most capable model, but it costs about 2.5× the mid-tier model per edit (ARCHITECTURE §11). The proposal is to decide from the M-1 eval results.
2. **Anonymous publishing in the public beta.** Keep it (maximum reach) or require a lightweight magic-link claim for publishing (less abuse)? The proposal is anonymous in M0, re-evaluated at the M1 gate using V9.
3. **Working name and domains.** "Things" is a placeholder. Two registrable domains are needed: one for the product and one for user content.

## Appendix A. Pre-implementation review record

See ROADMAP.md §6 for the critical review of all three documents, which lists what was challenged, what was cut, and what was changed.
