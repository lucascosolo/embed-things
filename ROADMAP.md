# Roadmap — "Things" (working title)

Status: reviewed draft, awaiting approval · Last updated: 2026-09-27
Companion documents: [PRODUCT_SPEC.md](PRODUCT_SPEC.md), [ARCHITECTURE.md](ARCHITECTURE.md)

---

## 1. How this roadmap works

Each milestone has a **scope**, an **exit gate** (the evidence needed to move on), and a **budget**. A milestone starts only after the previous gate passes. Gates are judged on data. Missing a gate means iterating, re-scoping, or stopping. It never means building more features to fix the numbers.

Durations assume one developer working with Claude Code. They are planning estimates, not commitments.

```
M-1 Spikes ─gate─▶ M0 Closed alpha ─gate─▶ M1 Public beta ─gate─▶ M2 Library ─gate─▶ M3 Developer preview ─gate─▶ M4 Platform
 (~1 wk)           (~2–3 wk build,         (~4–6 wk)              (~6 wk)            (~6–8 wk)                    (open)
                    4–8 wk measuring)
```

## 2. Milestones

### M-1: De-risking spikes (≈ 1 week)
This milestone tests the three assumptions that could sink the architecture, before any product is built.

| Spike | Deliverable | Pass criterion |
|---|---|---|
| **S1. Sandbox** | Content host, prelude, draft loader, the first 25 hostile Things, and the Playwright suite | Every hostile Thing fails at its goal in Chromium and WebKit (and in Firefox, run manually), except documented residuals. The CSP and sandbox survive `document.write` in the loader (ARCHITECTURE §4.2), or the fallback is chosen. Knob fragments and the postMessage bridge work on real iOS Safari and in the iMessage, WhatsApp, Instagram, Discord, and Telegram in-app browsers. The WebRTC residual is measured and written down. |
| **S2. AI edit quality and cost** | Five seeds, the 50-task eval harness, and runs on Haiku 4.5, Sonnet 5, Opus 5.5, and Opus 5 | At least one model reaches ≥ 85 % run-clean with a median rating ≥ 4. Measured $/edit is recorded, and the system prompt's cacheability is confirmed. **The approver picks the model** using the rule in ARCHITECTURE §10.4. |
| **S3. Mobile editor feel** | A clickable phone prototype of Knobs + Ask AI, tested with 5 non-coders | 4 of 5 finish a knob remix unaided in under 60 s. Every point of confusion is recorded. |

**Gate to M0:** S1 and S2 pass, and the S3 findings are folded into the editor design. If S1 fails in a major in-app browser, the sharing model is re-planned before anything else is built. In parallel, get the legal read on the viewer cookie (ARCHITECTURE §8.3).

**Budget:** about $30–60 of API usage for the eval, plus $5 of infrastructure.

### M0: Closed alpha, the smallest testable loop (≈ 2–3 weeks to build, 4–8 weeks to measure)
**Scope:** exactly PRODUCT_SPEC §6.1, which is the only authoritative list. In summary:

- player with lineage line and OG metadata;
- mobile-first editor (knobs, Ask AI, undo, error banner, desktop source text area);
- anonymous publish with share refs;
- about 12 seeds, including Blank;
- self-delete;
- sandbox, reports, daily human review, removal script, and kill switches;
- D1 daily limits and the reserve-then-settle spend cap;
- D1 event logging with saved SQL queries.

Two engineering deliverables are part of M0 but aren't user features: the hostile suite running in CI on Chromium and WebKit, and the review, removal, and metrics scripts.

**Distribution:** 50–100 invited seed users (friends, creative coders, a few group-chat-heavy non-coders). Each is asked to post Things into real group chats. **Publishing is open to everyone who arrives by link.** The thesis depends on recipients remixing, so there's no publish code. Volume is controlled by the rate limits, the spend cap, the kill switches, and the fact that the home page isn't publicised.

**Exit gate:** apply the decision table in PRODUCT_SPEC §10.3, once the sample floor in §10.2 is reached (≥ 2,000 unique recipient viewers from ≥ 20 distinct sharers), plus at least 10 interviews. Thresholds may be recalibrated once, after two weeks, and the change is logged in §6.3.

**Budget:** $5/month of infrastructure. AI spend is capped at $25/day; expected total is about $50–150.

### M1: Public beta, harden and open up (≈ 4–6 weeks)
M1 starts only if the M0 gate passes.

**Scope**, taken from the M0 deferrals in PRODUCT_SPEC §6.2 and prioritised by M0 funnel data:

1. **Legal and safety at scale:**
   - ToS with a remix licence, privacy notice, DMCA process, and 13+ age requirement, reviewed by counsel;
   - automated publish-time review (a Haiku-class classifier built from the rubric of the daily review script), plus a small admin page behind Cloudflare Access;
   - domain-reputation monitoring for both domains, and an abuse contact.
2. **Embed widget + oEmbed.** This relaxes the content host's `frame-ancestors` to `*` and uses click-to-play by default.
3. **Lineage UI:** a list of each Thing's remixes and "what changed" per step (knob diff, AI summaries).
4. **Per-Thing previews:** headless-rendered, including animated previews for Things above a view threshold only.
5. **Editor upgrades, only where M0 data shows people stalling:** idea chips, one-tap fix, streaming progress, and a code editor with syntax highlighting.
6. **Download as .html**, with the sandbox policy documented in the file header.
7. **Seed expansion**, guided by which seeds produced the most onward travel (V4) and the deepest chains (V5) in M0.
8. **Optional magic-link accounts** to claim Things across devices, **only if M0 interviews ask for them.**
9. **Storage moves (R2 for sources, an analytics store for events)**, only when the thresholds in ARCHITECTURE §6.2 are reached.

**Exit gate:**
- The loop holds at public scale: V1–V4 stay within 20 % of the M0 values across ≥ 10 k unique recipient viewers.
- Either V6 ≥ 0.8 over a 30-day cohort, **or** weekly publishing viewers grow organically by ≥ 20 % week over week for 4 weeks.
- Creator 4-week retention (publishing again in week 4 after a first publish) is ≥ 15 %.
- V8 and V9 are met on a sustained basis.
- No critical security findings are open.
- No domain blocklisting incident has gone unresolved for more than 48 h.

**Budget:** about $15/month of infrastructure. AI is capped at $150/day (a worst case of about $4.5 k/month); expected spend is about $0.9–1.8 k/month at the Haiku/Sonnet price point. Legal review is a one-off cost.

### M2: Public library, from links to a place (≈ 6 weeks)
M2 starts only if the M1 gate passes.

**Scope:**
- **Opt-in public listing.** Listed Things get stricter review and a **content rating** (G / PG / not listable).
- **Search and browse.** Search over titles, knob text, and change notes (D1 FTS5 or its successor store). Browse by new, most remixed this week, and deepest chains. No personalised recommendations.
- **Full remix-tree visualisation.**
- **Experiment: a near-blank start.** "Describe it" makes the AI pick the closest seed and apply the idea as the first edit. It's A/B tested against seed-only starts on V3 and V5, and ships only if it doesn't reduce human remixing later in the chain.
- **Creator handles** for library credit, if accounts exist.

**Exit gate:**
- ≥ 25 % of weekly opens start in the library rather than from links, **without** lowering V4 or V5.
- The moderation workload per 1,000 listed Things is sustainable for the team.
- A rating accuracy audit on a sample of 200 Things is ≥ 95 % correct.

### M3: Developer preview, becoming infrastructure (≈ 6–8 weeks)
M3 starts only if the M2 gate passes **and at least 3 external products have asked to integrate**. Those requests must be logged, not assumed.

**Scope:**
- The public **`thing/1` spec**, plus a conformance suite built from the hostile tests.
- An **open-source prelude and player**, so partners can self-host playback under the same policy.
- A **public read API** (`/v1/things/:id`, `/lineage`, `/remixes`, `/search`) with keys, quotas, rating filters, and attribution terms.
- A **status page, partner takedown webhooks, and API terms.**

**Exit gate:**
- ≥ 3 live third-party integrations with real traffic.
- API-sourced opens are ≥ 10 % of all opens.
- Takedowns propagate to partners within 1 h.

### M4: Platform, with picker, SDKs, and business model (open-ended)
- A drop-in picker web component, then React Native and iOS/Android wrappers around a policy-hardened webview.
- Monetisation experiments in order of plausibility (PRODUCT_SPEC §12).
- Native, non-webview rendering, evaluated only if partners require it.

## 3. What gets built when

| Feature | M-1 | M0 | M1 | M2 | M3 | M4 |
|---|---|---|---|---|---|---|
| Sandbox runtime + hostile suite | ● | ● (CI) | ● | ● | conformance suite | |
| Player, lineage line, OG with seed images | | ● | per-Thing / animated previews | | | |
| Knobs + Ask AI + undo | prototype | ● | chips, fix, streaming, code editor | near-blank A/B | | |
| Anonymous publish + share refs | | ● | | | | |
| Remix list + "what changed" | | data only | ● | full tree | API | |
| Embed + oEmbed | | reserved path | ● | | partner embeds | picker |
| Review and safety | | human daily + scripts | automated + admin UI | ratings | partner webhooks | |
| Accounts | | | only if asked | handles | | |
| Public library, search | | | | ● | API | picker |
| Public API, open format | | | | | ● | |
| SDKs, monetisation | | | | | | ● |

## 4. Usage estimates: Claude and cloud

These are rough planning numbers so the approver can size the commitment. They'll be replaced with actuals after M0.

**Claude Code (building the product):**
| Phase | Work | Rough effort |
|---|---|---|
| M-1 | 3 spikes, hostile suite, eval harness, prototype | ~6–10 focused sessions |
| M0 | Worker, player, editor, publish, lineage line, limits, events, scripts, 12 seeds, tests | ~15–25 focused sessions |
| M1 | Legal pages, automated review, embed/oEmbed, previews, lineage UI, editor upgrades | ~15–25 sessions |

The M0 codebase should come to about 4–7 k lines of TypeScript, including tests and seeds. That keeps the per-session context small, and most session tokens are cached re-reads.

**Claude API (the product's AI features):** see ARCHITECTURE §11.

| Milestone | Expected spend | Cap |
|---|---|---|
| M-1 eval | ≈ $30–60 one-off | n/a |
| M0 alpha | ≈ $50–150 in total | $25/day |
| M1 beta | ≈ $0.9–1.8 k/month (Haiku/Sonnet price point) | $150/day |

**Cloud:** Cloudflare Workers Paid at $5/month covers M-1 and M0. M1 is expected at about $5–15/month. Two domain registrations cost about $20–60 a year. Nothing else is needed before M2.

## 5. Stop and pivot criteria

These mirror the "Stop / rethink" row of PRODUCT_SPEC §10.3 and add what each result would mean.

- **Recipients don't play** (V1 < 40 % after one content iteration). The primitive itself doesn't grab people. Rethink the format, for example watch-first Things or lighter interaction.
- **Players don't remix** (V2 < 3 % after two editor iterations). Remixing isn't the hook. Consider make-your-own-from-a-template without lineage, or a creator tool aimed at tinkerers only.
- **Remixes don't travel** (V4 < 25 % after one sharing iteration). There's no viral loop. It may work as a tool, but not as a media primitive, and the GIPHY-style infrastructure vision is off.
- **The economics fail** (V8 > $0.40 with no path down). AI dependence is too high. Push knobs, cap AI, or move AI to a paid tier.
- **Abuse overwhelms** (V9 > 5 %, or repeated domain blocklisting). Require accounts for publishing, or restrict to invite-only communities.

## 6. Critical review record (completed before implementation)

The first drafts of all three documents got an adversarial review. Two passes covered them: an independent reviewer instructed to challenge scope, security claims, consistency, assumptions, and cost, and the author's own review. Every finding and its resolution is listed here. The documents above already reflect the resolutions.

### 6.1 Blocking defects found and fixed
| # | Finding | Resolution |
|---|---|---|
| R1 | **Embeds could never have worked.** The content host's `frame-ancestors https://things.example` is checked against *every* ancestor, so any third-party page embedding the player would have been refused. | The embed moved to M1. When it ships, the content host relaxes to `frame-ancestors *`, accepting that raw Things can be framed without attribution. The threat-model row claiming attribution protection was removed. |
| R2 | **Recipients couldn't publish.** The alpha gated publishing behind an invite code, but the thesis is judged on recipients' remixes, so V3, V4, and V6 were structurally unmeasurable. | Publishing is open to anyone who arrives by link. Volume is controlled with limits, the spend cap, and kill switches. |
| R3 | **V1 (play rate) was unmeasurable.** Interactions inside a cross-origin sandboxed frame are invisible to the host, and the bridge had no interaction message. | The prelude installs capture-phase listeners before author code runs and posts `interact`. Authors can forge it; that's accepted for the alpha. |
| R4 | **V5 was defined over an empty set** ("non-seed roots", which couldn't exist). | V5 is redefined over first-generation remixes of seeds and made directional. |
| R5 | **The WebRTC mitigation was overstated.** A fresh `about:blank`/`srcdoc` iframe needs no fetch, so `frame-src 'none'` doesn't stop it, and it yields pristine `RTCPeerConnection` constructors. | "No network" is now described as best-effort. The IP/typed-text leak is a documented residual, covered by hostile tests and flagged in review. The spec's "no data exfiltration" claim was corrected. |
| R6 | **Daily rate limits were infeasible.** The Workers Rate Limiting binding supports only 10 s and 60 s windows. | Daily limits moved to D1 counters. The binding is used only for bursts. |
| R7 | **"Device" and "recipient" were undefined without accounts.** Cookies were only set on mutation, in-app browsers keep separate cookie jars, and a creator opening their own link inflated K. | Added a viewer id on first view, share refs, and a precise recipient definition. V4–V6 are directional. A legal read on the cookie happens before the alpha, with a fallback defined. |

### 6.2 Major changes: cuts, simplifications, corrections
| # | Finding | Resolution |
|---|---|---|
| R8 | Analytics Engine can't do the joins V2–V6 need, and the metrics page was scope creep | Events go to a D1 table, analysed with saved SQL. No dashboard in M0. |
| R9 | The AI cost model ignored abandoned sessions (65 % of editor sessions don't publish) and fix calls | V8 is recomputed per published Thing, including abandonment. Result: only Haiku/Sonnet-class models meet V8 ≤ $0.15. |
| R10 | The model-choice rule was inconsistent ("most capable" vs "cheapest passing"), and the 2.5× figure was wrong for Opus 5.5 | One rule everywhere: the cheapest model that passes the eval and keeps V8 ≤ $0.15, with the approver deciding. |
| R11 | Pass/iterate/stop rules differed between SPEC and ROADMAP, and some outcomes had gaps | One decision table (SPEC §10.3), referenced from here, with guardrails split from thesis metrics. |
| R12 | ROADMAP's M0 list differed from SPEC §6.1, and this review section was an empty stub | SPEC §6.1 is the single source of truth. This section is now filled in. |
| R13 | The M0 build estimate (3–4 weeks, 8–12 k lines of code) was unrealistic for the listed scope | Scope cut (R14). The estimate is now 2–3 weeks and 4–7 k lines of code. |
| R14 | **M0 scope creep.** Items that don't validate V1–V4. | **Cut to M1:** embed player, per-Thing posters, remix list, "what changed" diffs, idea chips, one-tap auto-fix, streaming, CodeMirror, curated shelf, admin UI, automated review with a "held" state, manage links, download. **Kept:** player, knobs, Ask AI, undo, publish, seeds, limits, spend cap, reports, kill switches, hostile suite, self-delete. |
| R15 | Per-Thing OG posters via resvg-wasm weren't feasible (no colour-emoji font fits within the Worker size limit) | Each seed's image is pre-rendered at build time. Remixes inherit it, and the title goes in `og:title`. |
| R16 | Draft uploads created free, unreviewed, publicly cached HTML hosting, and a published Thing could navigate its frame to a draft | Uploads removed. The editor previews through a fixed loader fed by postMessage. The host verifies the sha/nonce in `ready` and kills mismatched frames. |
| R17 | `navigate-to` was cited as a control, but it was removed from the CSP spec | Removed. The product host's `frame-src` is documented as the real control, with hostile tests for self-navigation. |
| R18 | "Held" semantics depended on a cookie the creator's in-app browser lacks. Auto-hold after 3 reports invited griefing, and a Haiku review of string literals misses canvas-drawn content. | No held state in M0. Everything publishes live, with daily human review of all publishes and report alerts. Automated review comes in M1. |
| R19 | Cost-exhaustion attacks: Turnstile farms, 64 KB inputs, spend recorded only after calls | Added an input cap (32 KB source, ≤ 12 k tokens), reserve-then-settle spending, per-/24 limits, and an alert at 50 %. The daily cap is acknowledged as the real backstop. |
| R20 | The claim that "Android Chrome has site isolation" is mostly false on mid-range devices | Hangs are a residual risk on all mobile platforms. The watchdog is kept trivial. |
| R21 | Takedown gaps: `/r/:sha` stayed cached for a year, `/og` was "immutable forever", and knob remixes share their parent's blob | Zone purge on removal, browser max-age cut to 1 day, and a `blobs.blocked` flag that tombstones every Thing sharing a violating source. Seed-only OG images leave nothing per-Thing to purge. |
| R22 | The manage link put the device's master credential in a URL | Cut. Deletion works from the publishing browser, or by email. |
| R23 | The sample floor gave only about 28 recipient remixes (V3 CI ±18 points) from about 30 clustered seeders | The floor is now ≥ 2,000 unique recipient viewers from ≥ 20 distinct sharers, with 50–100 seed users. V1 and V2 are primary. V4–V6 are directional. |
| R24 | *(author)* Kill switches as environment variables need a deploy to change | Moved to a D1 `flags` table that takes effect immediately. |
| R25 | *(author)* The Cloudflare cache API's `delete` only affects one data centre | Takedowns use the zone purge API. |

### 6.3 Minor corrections
- Removed COOP and CORP from the content host headers: COOP does nothing in a frame, and CORP risks breaking framing under COEP. Permissions-Policy is kept, labelled as defence in depth.
- Dropped `allow="autoplay"`. Sound starts after a tap inside the Thing, which matches the runtime contract.
- Allowed `'unsafe-eval'`. It adds no capability beyond inline script, and blocking it would break AI-written code. The eval harness runs under the real CSP.
- Knob values travel in the fragment for the initial load only. Live changes go through postMessage, so browser history doesn't fill up with entries.
- The `Sec-Fetch-Dest` check runs before the cache lookup, and responses send `Vary: Sec-Fetch-Dest`.
- One Worker on two hostnames replaces two Workers. D1 holds blobs in M0, and R2 is added when the thresholds are hit.
- Size cap reduced from 64 KB to 32 KB to bound AI inputs.
- Fixed number drift: the M1 cap of $150/day means about $4.5 k/month worst case, with expected spend separate. "Immutable content" is now worded precisely.
- Added a Blank seed, which gives tinkerers a near-blank start with honest lineage and costs no extra scope.

### 6.4 Challenged and deliberately kept
- **Single-file HTML/JS as the format** (vs. a DSL or WASM). It's the only option that is expressive, fluent for LLMs, and buildable in weeks. Safety comes from the browser sandbox, not from the format.
- **Anonymous publishing.** Friction is the thesis's main enemy. Abuse is bounded by link-only distribution, review, limits, and kill switches. It gets re-evaluated at the M1 gate using V9.
- **Ask AI in M0.** Knobs alone can't test the "AI as accelerator" part of the vision, and the cost is bounded by caps.
- **No one-shot blank-canvas generation.** Recipients arrive holding a Thing, so the thesis doesn't need it. The Blank seed covers tinkerers, and M2 tests a near-blank start.
- **Cloudflare as the single vendor.** Lowest idle cost, zero egress, and bot protection included. The data stays portable (SQLite plus HTML).
- **About 12 seeds.** This is content, not code. Seeds are the cold-start mechanism and the source of every lineage.

### 6.5 Resulting scope check
- **Internally consistent.** There's one scope list (SPEC §6.1), one decision table (SPEC §10.3), one model-selection rule (ARCH §10.4), and one set of cost numbers (ARCH §11). ROADMAP references them rather than restating them.
- **Technically feasible.** Every browser behaviour the design relies on has a named verification in S1. The remaining uncertainty (whether CSP survives `document.write`) has a defined fallback.
- **Intentionally small.** M0 is one Worker, one database, one AI call type, two UI surfaces (player and editor), about 12 seeds, and scripts in place of admin tools.
