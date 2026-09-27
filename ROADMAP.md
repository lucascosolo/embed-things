# Roadmap — "Things" (working title)

Status: draft for approval · Last updated: 2026-09-27
Companion documents: [PRODUCT_SPEC.md](PRODUCT_SPEC.md), [ARCHITECTURE.md](ARCHITECTURE.md)

---

## 1. How this roadmap works

Each milestone has a **scope**, an **exit gate** (evidence needed to proceed), and a **budget**. Work on a milestone starts only after the previous gate passes. Gates are judged on data, and a missed gate means iterate, re-scope, or stop, never "build more features to fix the numbers."

Durations assume one developer working with Claude Code. They're planning estimates, not commitments.

```
M-1 Spikes ──gate──▶ M0 Closed alpha ──gate──▶ M1 Public beta ──gate──▶ M2 Library ──gate──▶ M3 Developer preview ──gate──▶ M4 Platform
 (~1 wk)              (~3–4 wk build,           (~4–6 wk)               (~6 wk)              (~6–8 wk)                     (open)
                       4–6 wk measuring)
```

## 2. Milestones

### M-1 — De-risking spikes (≈ 1 week)
Purpose: prove the three things that could invalidate the architecture before building the product.

| Spike | Deliverable | Pass criterion |
|---|---|---|
| **S1. Sandbox** | Content Worker + prelude + the first 25 hostile Things + Playwright suite | All hostile Things fail at their goal in Chromium, WebKit, and Firefox. Manual check on real iOS Safari and the in-app browsers of iMessage, WhatsApp, Instagram, Discord, and Telegram. Knob fragment and postMessage bridge work in all of them. |
| **S2. AI edit quality and cost** | Five seed Things, the 50-task eval harness, runs on the candidate models | At least one model reaches ≥ 85 % run-clean and a median rating ≥ 4. The measured $/edit is recorded. **Model choice goes to the approver.** |
| **S3. Mobile editor feel** | Clickable prototype of the knobs panel + Ask AI on a phone, tried by 5 non-coders | 4 of 5 complete a knob remix unaided in < 60 s. Their confusion points are written down. |

**Gate to M0:** S1 and S2 pass. S3 findings are folded into the M0 editor design. If S1 fails in a major in-app browser, re-plan the sharing model before building anything else.

**Budget:** eval runs ≈ $30–60 of API usage; infrastructure $5.

### M0 — Closed alpha: the smallest testable loop (≈ 3–4 weeks build, 4–6 weeks measurement)
**Scope (exactly PRODUCT_SPEC §6.1, nothing more):**
1. Player page with OG unfurls, poster cards, the watchdog, and Remix/Share/Restart.
2. Embed player (`/e/:id`, click-to-play) and copy-embed.
3. Mobile-first editor: knobs, Ask AI (Edit), Auto-fix, idea chips, undo/redo, and a desktop code view.
4. Anonymous publish with Turnstile, a device session, and a manage link.
5. Lineage: the parent chain, the children list, and "what changed."
6. About 12 seed Things and the home page with a curated shelf.
7. Delete (tombstone), report, publish-time review, an admin queue behind Access, and kill switches.
8. Rate limits and the global AI spend cap.
9. Analytics Engine events for V1–V9, and a simple internal metrics page (a SQL query over Analytics Engine; no dashboard product).
10. The hostile suite and the AI eval in CI.

**Distribution:** invite-only. About 30 seed users (friends, creative-coder acquaintances) who each put Things into real group chats. Recipients need no invite, since links work for anyone, but publishing is gated behind an alpha code to control volume.

**Exit gate (PRODUCT_SPEC §10):** the minimum sample (1,000 recipient opens and 150 published remixes) is reached, plus:
- **Pass:** V1 ≥ 60 %, V2 ≥ 8 %, V3 ≥ 35 %, V4 ≥ 50 %, V6 ≥ 0.5, V7 ≤ 2 s, V8 ≤ $0.15, V9 ≤ 2 %.
- **Iterate:** V1 is met but V2 or V3 misses. Up to 2 editor iteration cycles of about 2 weeks each, then re-measure.
- **Stop / rethink:** V1 < 40 % or V2 < 3 % after the iterations (see §5).
- **Qualitative:** at least 10 user interviews (senders and recipients) about why they did or didn't remix.

**Budget:** infrastructure $5/mo; AI hard cap $25/day, expected ~$100–250 in total.

### M1 — Public beta: harden and open the doors (≈ 4–6 weeks)
Only starts if the M0 gate passes.

**Scope:**
- Remove the alpha publish code. Launch publicly (creative-coding communities, a Show HN-style post, seeded group chats).
- Legal plumbing (ToS with a remix licence, privacy notice, DMCA, 13+ requirement), reviewed by counsel.
- Safety at scale: better admin tooling (batch actions, a reporter trust signal), Safe Browsing / domain-reputation monitoring for both domains, and an abuse contact.
- **oEmbed** endpoint (cheap, and it widens where embeds work).
- **Real previews for popular Things:** a headless render (Cloudflare Browser Rendering or equivalent) producing an animated preview for Things above a view threshold only. Still no user-supplied pixels.
- "Download as .html" (the portability promise) with the sandbox policy documented in a README header.
- Seed pack expansion driven by M0 data (which seeds generated the deepest chains).
- **Only if M0 interviews ask for it:** optional magic-link accounts to claim Things across devices (attaching to `creator_key`).
- Editor improvements chosen from M0 funnel drop-off data, not guesses.

**Exit gate:**
- The loop holds at public scale: V1–V4 stay within 20 % of the M0 values with ≥ 10 k recipient opens.
- V6 ≥ 0.8 over a 30-day cohort, **or** a clear, organic, growing weekly count of publishing devices (≥ 20 % week-over-week for 4 weeks).
- 4-week retention of creators (published ≥ 1 Thing in week 4 after their first publish) ≥ 15 %.
- V8 ≤ $0.15 and V9 ≤ 2 %, sustained.
- There are no unresolved critical security findings and no domain blocklisting incident left unresolved for more than 48 h.

**Budget:** infrastructure ~$15/mo; AI cap $150/day (≈ $2 k/mo at the model chosen in S2); legal review is a one-off cost.

### M2 — Public library: from links to a place (≈ 6 weeks)
Only starts if the M1 gate passes.

**Scope:**
- Opt-in "list publicly" at publish. Listed Things go through a stricter review and get a **content rating** (G / PG / not-listable).
- Search (D1 FTS5 over title, knob text, and change notes) and a browse page (new, most-remixed this week, deepest chains). No personalised recommendations.
- Remix-tree visualisation (the whole family tree, not just parent and children).
- **Experiment:** a blank-canvas start ("describe it → AI picks the nearest seed and applies your idea as the first edit"), A/B tested against seed-only on chain depth (V5) and remix completion (V3). It ships only if it doesn't reduce human remixing downstream.
- Lightweight creator identity (a handle on accounts, if accounts exist) for credit in the library.

**Exit gate:** ≥ 25 % of weekly opens start from the library, not links, **and** the library doesn't lower V4 or V5; the moderation workload per 1,000 listed Things stays sustainable for the team size; the rating accuracy audit (a sample of 200) is ≥ 95 % correct.

### M3 — Developer preview: become infrastructure (≈ 6–8 weeks)
Only starts if the M2 gate passes and **at least 3 external products have asked to integrate** (logged requests, not assumed demand).

**Scope:**
- The `thing/1` format spec, stable and public, with a published conformance suite derived from the hostile tests.
- An open-source prelude and player so partners can self-host playback under the same security policy.
- A public read API (`/v1/things/:id`, `/lineage`, `/remixes`, `/search`) with API keys, quotas, rating filters, and attribution requirements.
- A status page, abuse and takedown webhooks for partners, and API terms.

**Exit gate:** ≥ 3 live third-party integrations with real traffic; API-sourced opens ≥ 10 % of the total; takedown propagation to partners within 1 h.

### M4 — Platform: picker, SDKs, business model (open-ended)
- A drop-in picker web component (search + preview + insert), then React Native and iOS/Android wrappers around a policy-hardened webview.
- Monetisation experiments in order of plausibility (PRODUCT_SPEC §12): partner API tiers, Creator Pro, sponsored seeds.
- Evaluate native (non-webview) rendering only if partners require it.

## 3. What gets built when (feature → milestone)

| Feature | M-1 | M0 | M1 | M2 | M3 | M4 |
|---|---|---|---|---|---|---|
| Sandbox runtime + hostile tests | ● | ● | ● | ● | conformance suite | |
| Player, unfurls, poster cards | | ● | animated previews | | | |
| Embed iframe | | ● | oEmbed | | partner embeds | picker |
| Knobs + Ask AI + undo + auto-fix | proto | ● | refine | blank-start A/B | | |
| Anonymous publish + manage link | | ● | | | | |
| Accounts | | | only if asked | handles | | |
| Lineage (parent/children/diff) | | ● | | full tree viz | API | |
| Reports, review, admin, kill switches | | ● | scaled | ratings | partner webhooks | |
| Public library, search | | | | ● | API | picker |
| Public API, open format | | | | | ● | |
| SDKs, monetisation | | | | | | ● |

## 4. Development usage estimates (Claude and cloud)

These are rough planning numbers so the approver can size the commitment. They'll be replaced with actuals after M0.

**Claude Code (building the product):**
| Phase | Work | Rough Claude Code usage |
|---|---|---|
| M-1 | 3 spikes, hostile suite, eval harness | ~8–12 focused sessions |
| M0 | Two Workers, player, editor, lineage, safety, seeds, tests | ~25–40 focused sessions |
| M1 | Hardening, oEmbed, previews, legal pages | ~15–25 sessions |

Most of the token volume in a session is cached context re-reads. The codebase at M0 is expected to be about 8–12 k lines of TypeScript, including tests and seeds, which keeps per-session context small.

**Claude API (the product's AI features):** see ARCHITECTURE §11. In summary: M-1 eval ≈ $30–60; M0 alpha ≈ $100–250 total (capped at $25/day); M1 beta ≈ $2 k/month at the Sonnet-5-class price point (or ≈ $5 k at the Opus-5-class price), capped at $150/day.

**Cloud:** Cloudflare Workers Paid at $5/month covers M-1 through M1 at the projected volumes. Two domain registrations cost about $20–60 a year. Nothing else is needed.

## 5. Stop and pivot criteria

The product should **stop or change direction** rather than add features if any of the following holds after the allowed iterations:
- **Recipients don't play** (V1 < 40 %). The primitive itself doesn't grab people. Rethink the format, for example watch-first Things or a lighter interaction.
- **Players don't remix** (V2 < 3 %) even after two editor iterations. Remix isn't the hook. Consider a "make-your-own from a template" product without lineage, or a creator-tool pivot aimed at tinkerers only.
- **Remixes don't travel** (V4 < 25 %, V6 < 0.2). There's no viral loop. The product might work as a tool, but not as a media primitive, so the GIPHY-style infrastructure vision is off.
- **Unit economics fail** (V8 > $0.40 with no path down). AI dependence is too high. Push knobs harder, cap AI, or move AI to a paid tier.
- **Abuse overwhelms** (V9 > 5 % or repeated domain blocklisting). Require accounts for publishing, or restrict to invite-only communities.

## 6. Critical review record (completed before implementation)

This section records the pre-implementation review of all three documents: what was challenged, what was cut, and what was changed. See the subsections below.
