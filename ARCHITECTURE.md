# Architecture — "Things" (working title)

Status: draft for approval · Last updated: 2026-09-27
Companion documents: [PRODUCT_SPEC.md](PRODUCT_SPEC.md), [ROADMAP.md](ROADMAP.md)

---

## 1. Principles

1. **The sandbox is the security boundary.** We never rely on inspecting, linting, or AI-reviewing code to keep users safe. Every Thing is treated as hostile. Code review only matters for *content policy*, not for *containment*.
2. **Immutable content, mutable pointers: none.** Published Things and their code blobs never change. Everything is cacheable forever at the edge.
3. **The cheapest request is the one that never reaches the origin.** The player shell is static and Thing code is served content-addressed with immutable caching. The only dynamic hot-path work is a small metadata read.
4. **AI is an optional, bounded, stateless service.** No AI call is on the critical path of viewing, and the loop still works (knobs, code) if AI is turned off.
5. **Build for one region of complexity.** One platform (Cloudflare), one language (TypeScript), one repo, one database. Nothing is introduced before a milestone needs it.
6. **Keep future contracts stable, and build nothing for them yet.** URL shapes, the format version, and the embed contract are fixed from day one because third parties will depend on them later. The API/SDK surfaces themselves are not built.

## 2. System overview

```
                           ┌───────────────────────────────────────────────┐
  Browser / chat webview   │  PLAYER DOMAIN  (e.g. things.example)         │
  ───────────────────────▶ │  • Static assets: player shell, editor (lazy) │
                           │  • Worker "app":                              │
                           │      GET  /t/:id        (HTML + OG meta)      │
                           │      GET  /e/:id        (embed player)        │
                           │      GET  /og/:id.png   (poster card)         │
                           │      /api/v1/*          (JSON)                │
                           └──────┬───────────────┬───────────────┬────────┘
                                  │               │               │
                          ┌───────▼──────┐ ┌──────▼─────┐  ┌──────▼──────────┐
                          │ D1 (SQLite)  │ │ R2 blobs   │  │ Anthropic API   │
                          │ things,      │ │ code/{sha} │  │ (server-side    │
                          │ reports,     │ │ drafts/{sha}│ │  key, via Worker│
                          │ ai_budget    │ └──────▲─────┘  │  only)          │
                          └──────────────┘        │        └─────────────────┘
                                                  │
  <iframe sandbox>         ┌──────────────────────┴────────────────────────┐
  ───────────────────────▶ │  CONTENT DOMAIN (separate registrable domain, │
                           │  e.g. thingcontent.example)                   │
                           │  Worker "content": GET /r/:sha  → prelude +   │
                           │  Thing HTML, strict CSP, sandbox, immutable   │
                           └───────────────────────────────────────────────┘

  Side services: Turnstile (bot check), Workers Rate Limiting binding,
  Workers Analytics Engine (product events), Cloudflare Access (admin page).
```

The two Workers live in one repository and deploy together. The player domain owns the UI, the API, and metadata. The content domain does exactly one thing: it serves Thing code under the most restrictive policy the browser supports.

## 3. The Thing format (`thing/0`)

### 3.1 Shape
A Thing's **source** is one UTF-8 HTML document, at most **64 KB**. Typical AI-authored Things are 4–15 KB. The source contains a manifest:

```html
<!doctype html>
<script type="application/thing+json">
{
  "format": "thing/0",
  "title": "Taco Rain",
  "aspect": "1:1",
  "knobs": [
    { "id": "speed",   "type": "number", "label": "Fall speed", "min": 0.5, "max": 10, "step": 0.5, "default": 3 },
    { "id": "emoji",   "type": "emoji",  "label": "What falls", "default": "🌮" },
    { "id": "message", "type": "text",   "label": "Message", "maxLength": 80, "default": "It's raining tacos" },
    { "id": "bg",      "type": "color",  "label": "Sky", "default": "#1b1b3a" },
    { "id": "sound",   "type": "toggle", "label": "Sound", "default": true },
    { "id": "mode",    "type": "choice", "label": "Mode", "options": ["gentle", "storm"], "default": "gentle" }
  ]
}
</script>
<style>/* … */</style>
<canvas id="c"></canvas>
<script>
  const k = thing.knobs;              // resolved values (defaults overridden by this Thing's values)
  thing.on("knobs", (next) => { … }); // optional live update
  // … ordinary JS using DOM, Canvas 2D, SVG, CSS, Web Audio, pointer/keyboard events
</script>
```

- **Knob types in `thing/0`:** `number`, `color`, `text` (≤ 200 chars), `toggle`, `choice` (≤ 12 options), `emoji`. There are at most 16 knobs per Thing.
- **Aspect:** `1:1` (default), `4:5`, `9:16`, `16:9`. The player letterboxes to fit.
- **Allowed capabilities (the runtime contract):** DOM, CSS, SVG, Canvas 2D, Web Audio (starts after a user gesture), pointer, touch, and keyboard events, `requestAnimationFrame`, timers, and `crypto.getRandomValues`. WebGL may happen to work, but it isn't part of the contract and isn't guaranteed.
- **Not available:** network (fetch, XHR, WebSocket, EventSource, beacons, remote images/fonts/scripts), storage (cookies, localStorage, IndexedDB, Cache), navigation of the parent or top window, popups, forms submission, camera, microphone, geolocation, sensors, clipboard, fullscreen, payment, workers loaded from URLs, and nested frames.

### 3.2 A published Thing = source blob + knob values + metadata
Knob **values** are stored separately from the source. A knob-only remix therefore reuses the parent's source blob (same hash) and only stores new values. This makes the most common remix free in storage and AI terms, and it makes "what changed" an exact structured diff.

### 3.3 Versioning
`format` is required. The runtime refuses unknown major formats with a clear placeholder. `thing/0` is explicitly **unstable** during the MVP; it becomes `thing/1` (stable, publicly specified) at ROADMAP M3. A migration job can rewrite `thing/0` Things into new blobs, because published Things refer to blobs by hash and metadata is separate.

### 3.4 Why HTML and not a custom DSL or engine
| Option | Verdict |
|---|---|
| **Single-file HTML + JS in a hard sandbox** (chosen) | Maximum expressiveness. LLMs are fluent in it. Zero custom runtime to build. Portable (a Thing is a file that runs anywhere under the same policy). Safety comes from the browser sandbox, which is the most heavily attacked and patched isolation layer that exists. |
| Declarative DSL / JSON scene graph | Safer by construction, but it needs an engine, an editor, and an LLM fluency layer, and it caps expressiveness at whatever we build. Months of work before any validation. Rejected for the MVP. It may return as an *optional* higher-level authoring layer that compiles to HTML. |
| JS in a WASM interpreter (e.g. QuickJS) with a custom draw API | Strong isolation, and it could run in native SDKs without a webview. But it has no DOM, needs a custom rendering API, and gives slower, more limited Things. Recorded as a future option for native SDK rendering (M4+), not the MVP. |
| Fixed templates with parameters only | Trivial and safe, but no real remixing beyond knobs, and it can't test whether AI-assisted changes matter. Rejected. |

## 4. Rendering pipeline

### 4.1 Published Things
1. `GET /t/:id` on the player domain returns a small HTML shell (inline critical CSS, ~20–40 KB JS gzipped, cached at the edge). The metadata JSON is inlined into the HTML so there's no extra round-trip. The Worker reads it from D1 via the edge cache, and it's immutable per id except for `status` and `remix_count` (short TTL; see §6.3).
2. The shell creates:
   ```html
   <iframe
     src="https://thingcontent.example/r/{sha256}#k={base64url(knob values JSON)}"
     sandbox="allow-scripts"
     allow="autoplay"
     referrerpolicy="no-referrer"
     loading="eager"
     title="{Thing title}">
   </iframe>
   ```
3. The content Worker serves `/r/{sha256}`: the **prelude** (a ~3 KB script, versioned) followed by the source blob from R2. Headers are described in §5.2. The response is `Cache-Control: public, max-age=31536000, immutable`; the knob values travel in the URL fragment, so one cached response serves every knob variant.
4. The prelude parses the fragment and the manifest, resolves knob values against the declared types (clamping, truncating, dropping unknown keys), exposes `window.thing`, neutralises the APIs listed in §5.4, installs error, heartbeat, and resize reporting, and then lets the Thing's scripts run.

### 4.2 Drafts in the editor
The same render path is used for previews, so there's only one pipeline to secure:
- **Knob changes** only change the fragment. The prelude fires `thing.on("knobs")` if the Thing handles live updates; otherwise the iframe reloads, which takes milliseconds from cache.
- **Code changes** (AI or hand edits) are uploaded as `POST /api/v1/drafts` (the body is the source). The server validates the size and manifest, stores it at `drafts/{sha256}` in R2 (a lifecycle rule expires it after 7 days), and the preview iframe points at `/r/{sha256}`. The content Worker resolves `code/` first and then `drafts/`.
- Draft uploads require a valid session (Turnstile-issued, §8.1) and are rate-limited.

### 4.3 Host ↔ Thing bridge (postMessage)
It's deliberately tiny, and it's the only channel. The host validates `event.source === iframe.contentWindow` (the origin is opaque, `"null"`) and schema-checks every message.

| Direction | Message | Purpose |
|---|---|---|
| Thing → host | `ready` | First script ran |
| Thing → host | `heartbeat` (every 1 s) | Watchdog |
| Thing → host | `error {message, line}` | Crash banner, auto-fix input |
| Host → Thing | `visibility {visible}` | Pause when off-screen |
| Host → Thing | `prefs {muted, reducedMotion}` | Accessibility and preferences |

Things **cannot** ask the host to navigate, share, resize the page, open URLs, or read anything. Remix, share, and attribution UI lives only in the host chrome, outside the Thing's pixels.

### 4.4 Watchdog and resource limits
- If there's no heartbeat for 3 s, the host shows "stopped responding," and Restart reloads the iframe. On browsers with site isolation (desktop Chrome and Firefox, Android Chrome), the separate content domain puts the Thing in its own process, so the host stays responsive. **On iOS WebKit there's no out-of-process iframe**, so an infinite loop can freeze the whole tab. The mitigations are the embed's click-to-play mode, the publish-time check (§7.3), and fast takedown. This is an accepted residual risk.
- The 64 KB source cap bounds download size. Memory is not boundable from JS; browsers kill runaway frames.

## 5. Security and sandboxing

### 5.1 Threat model
| Actor | Goal | Primary control |
|---|---|---|
| Malicious Thing author | Steal viewer data or credentials; track viewers; phish; attack the host page; mine crypto; crash devices | Sandbox + CSP + separate origin + no network (§5.2) |
| Malicious Thing author | Offensive or illegal visible content | Publish-time review, reports, takedown (§7) |
| Spammer / bot | Mass publishing, AI cost abuse | Turnstile, rate limits, budget caps (§8) |
| Prompt injection via a Thing's source | Make the AI produce something harmful | The AI has no tools, secrets, or cross-user data; output runs in the same sandbox and goes through publish review (§6.4) |
| Third-party embedder | Frame raw Things without attribution | `frame-ancestors` restricts the content domain to the player domain (§5.2) |
| Attacker targeting the content domain directly | Host phishing pages at top level | CSP `sandbox` header applies at top level; `Sec-Fetch-Dest` check; nothing on the domain has any authority |

### 5.2 Content-domain response headers (`/r/:sha`)
```
Content-Type: text/html; charset=utf-8
Content-Security-Policy:
  sandbox allow-scripts;
  default-src 'none';
  script-src 'unsafe-inline';
  style-src 'unsafe-inline';
  img-src data: blob:;
  media-src data: blob:;
  font-src data:;
  connect-src 'none';
  frame-src 'none'; child-src 'none'; worker-src 'none';
  form-action 'none';
  base-uri 'none';
  navigate-to 'none';            (ignored where unsupported; harmless)
  frame-ancestors https://things.example
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), usb=(), serial=(),
  bluetooth=(), hid=(), clipboard-read=(), clipboard-write=(), fullscreen=(), display-capture=(),
  accelerometer=(), gyroscope=(), magnetometer=(), publickey-credentials-get=()
Cross-Origin-Resource-Policy: same-site
Cross-Origin-Opener-Policy: same-origin
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
Cache-Control: public, max-age=31536000, immutable
```
- The iframe `sandbox="allow-scripts"` attribute plus the CSP `sandbox` directive give an **opaque origin** (no cookies, no storage, no same-origin access to anything) even if someone loads the URL at top level. The sandbox never grants `allow-same-origin`, `allow-top-navigation*`, `allow-popups`, `allow-forms`, `allow-modals`, `allow-downloads`, or `allow-pointer-lock`.
- **Separate registrable domain** (not a subdomain): no shared cookies, no same-site privileges, and a clean reputation boundary so a bad Thing can't get the product domain blocklisted.
- **`Sec-Fetch-Dest`:** requests with `Sec-Fetch-Dest: document` (top-level navigation) get a 403 with a pointer to the player page. Requests without the header (older browsers) still get the CSP sandbox.
- The content domain has **no cookies, no API, no user data, and no secrets**. Compromising it gains nothing.

### 5.3 Player-domain protections
Strict CSP on the player domain with no `unsafe-inline` scripts (nonce-based), `frame-src https://thingcontent.example`, and `frame-ancestors *` only on `/e/:id` (embeds) versus `'self'` elsewhere. Things never get player-domain privileges.

### 5.4 Known gaps and residual risks (reviewed deliberately)
| Gap | Detail | Decision |
|---|---|---|
| **WebRTC bypasses `connect-src`** | Browsers don't consistently govern `RTCPeerConnection` with CSP, so a Thing could reach a STUN/TURN server and reveal the viewer's IP (tracking), or leak a small amount of data typed into it. | The prelude deletes or overrides `RTCPeerConnection`, `webkitRTCPeerConnection`, and `RTCDataChannel` before user code runs. `frame-src`/`child-src 'none'` stops the "fresh about:blank frame" trick of recovering pristine constructors via a new iframe. The publish-time static signal flags `RTC`, `iceServers`, and `stun:`. **Residual risk accepted for MVP:** there's no credential exposure, because Things never see anything but their own input. Revisit when the CSP `webrtc` directive ships broadly. |
| DNS prefetch / preconnect hints | `<link rel=dns-prefetch>` can leak a DNS lookup in some browsers. | Low impact (no payload). The prelude strips `link[rel]` hints; static flag. |
| CPU/battery abuse | An infinite loop or heavy rendering. | §4.4. Click-to-play in embeds. Pausing when off-screen (`visibility`). |
| Visual deception | A Thing draws a fake "Google login." Without network or forms it can't send anything anywhere, and it can't navigate. The risk is social: persuading people to act *outside* the Thing ("text your code to …"). | Content policy, review, reports. Host chrome always frames Things clearly as user content ("Made by a user · Report"). |
| Browser zero-days | Sandbox escapes. | Out of our control. Keep the content domain devoid of value; takedowns are fast. |

### 5.5 Security verification (a gating deliverable, not an afterthought)
A **hostile Things test suite** (Playwright, run in CI against Chromium, WebKit, and Firefox) of at least 25 adversarial Things. Each one must *fail* at its goal: fetch/XHR/WebSocket/beacon/image-pixel exfiltration, form submission, top navigation, `window.open`, parent DOM access, cookie and storage access, `document.domain` tricks, nested iframe creation, WebRTC after neutering, service worker registration, redirect via meta refresh, top-level load of `/r/:sha`, oversized source, malformed manifest, knob-value injection (XSS via text knob into the host chrome), and postMessage spoofing. **It's run manually on real iOS Safari and in the in-app browsers** (iMessage, WhatsApp, Instagram, Discord, Telegram) in the M-1 spike and before each milestone gate.

## 6. Data and storage model

### 6.1 D1 schema (SQLite)
```sql
CREATE TABLE things (
  id            TEXT PRIMARY KEY,          -- 8-char base62, random (unguessable)
  source_sha    TEXT NOT NULL,             -- sha256 of source blob in R2 code/
  knob_values   TEXT NOT NULL,             -- JSON, validated against manifest
  title         TEXT NOT NULL,             -- ≤ 80 chars
  nickname      TEXT,                      -- ≤ 32 chars, optional, unverified
  parent_id     TEXT REFERENCES things(id),
  root_id       TEXT NOT NULL,             -- self for roots
  depth         INTEGER NOT NULL,          -- 0 for roots
  change_kind   TEXT NOT NULL,             -- 'root' | 'knobs' | 'code'  (code ⇒ source changed)
  change_notes  TEXT,                      -- JSON array of AI one-line summaries (≤ 8), public
  edit_counts   TEXT NOT NULL,             -- JSON {"knob":n,"ai":n,"code":n}
  aspect        TEXT NOT NULL,
  status        TEXT NOT NULL DEFAULT 'live', -- live | held | removed | deleted
  rating        TEXT,                      -- reserved for M2 content ratings; NULL in M0
  is_seed       INTEGER NOT NULL DEFAULT 0,
  remix_count   INTEGER NOT NULL DEFAULT 0, -- direct live children
  creator_key   TEXT NOT NULL,             -- sha256(device session id); enables delete & rate limits
  created_at    TEXT NOT NULL              -- ISO-8601 UTC
);
CREATE INDEX things_parent ON things(parent_id, created_at);
CREATE INDEX things_creator ON things(creator_key, created_at);

CREATE TABLE reports (
  id TEXT PRIMARY KEY, thing_id TEXT NOT NULL, reason TEXT NOT NULL,
  detail TEXT, reporter_key TEXT NOT NULL, created_at TEXT NOT NULL,
  resolved_at TEXT, resolution TEXT
);

CREATE TABLE ai_spend_daily (          -- global cost circuit breaker
  day TEXT PRIMARY KEY, usd_micros INTEGER NOT NULL DEFAULT 0, calls INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE idea_cache (              -- remix-idea chips, generated once per source
  source_sha TEXT PRIMARY KEY, ideas TEXT NOT NULL, created_at TEXT NOT NULL
);
```
There are no user tables and no stored IPs. Views, plays, and funnel events go to **Workers Analytics Engine**, not D1, so D1 writes stay proportional to *publishes*, not *views*.

### 6.2 R2 layout
- `code/{sha256}`: published sources. Immutable and deduplicated across all Things sharing a source (every knob-only remix).
- `drafts/{sha256}`: editor drafts, expiring via a 7-day lifecycle rule. Publishing copies the draft to `code/` if it's absent.

Sources go in R2 rather than D1 because D1 caps a database at 10 GB and costs about 50× more per GB. Metadata stays in D1 because it's relational (lineage queries).

### 6.3 Caching
| Resource | Cache policy |
|---|---|
| `/r/:sha` (content) | Immutable, 1 year, edge and browser |
| Player shell / static assets | Hashed filenames, immutable |
| `/t/:id` HTML (with inlined metadata) | Edge 60 s + `stale-while-revalidate`; purged on status change (removal must take effect quickly) |
| `/og/:id.png` | Immutable per id (the poster shows only immutable fields) |
| `/api/v1/things/:id/remixes` | Edge 60 s |

### 6.4 Lineage queries
- **Ancestors:** a `WITH RECURSIVE` walk up `parent_id`, limited to 50, returning id, title, nickname, change_kind, change_notes, and status. The UI shows 3 and expands on demand.
- **Children:** `WHERE parent_id = ? AND status = 'live' ORDER BY created_at DESC LIMIT 50` with a cursor.
- **Integrity:** rows are never hard-deleted. `deleted` and `removed` become tombstones: the lineage keeps the node (title replaced with "Removed" and no source served), and children keep working because they have their own source blobs.
- **"What changed"** is computed on read: the knob diff comes from comparing `knob_values` against the parent's (it's the same manifest when `change_kind = 'knobs'`), plus `change_notes` and `edit_counts`.

## 7. Trust and safety pipeline (MVP)

1. **Publish-time automated review:** a single low-cost model call (Haiku-class) reads the title, nickname, knob text values, and the source's string literals and visible text. It classifies content against the policy (hate, harassment, sexual content, self-harm, violent threats, deceptive/phishing text, spam, personal data of others) and returns `allow | hold`. Held Things are published with `status = held`, so the creator sees them and shares fail softly ("This Thing is being reviewed") until an admin decides. **Knob-only remixes of an already-reviewed source only review the changed text**, which is cheaper.
2. **Static signals** (regex, not a security control): WebRTC identifiers, `link[rel=…]` hints, very long string literals, obfuscation patterns (`eval`, `Function(`, heavy base64). These raise hold priority.
3. **Reports:** a button on every Thing. Three independent reports put a Thing into `held` automatically, pending review.
4. **Admin queue:** one page behind Cloudflare Access listing held and reported Things with a sandboxed preview, with Remove / Restore actions. Removal purges the edge cache for `/t/:id` and the R2 source if no live Thing references it.
5. **Kill switches:** environment flags `PUBLISH_ENABLED`, `AI_ENABLED`, `DRAFTS_ENABLED`, changeable without a deploy.
6. **Legal plumbing before public beta (M1):** ToS with a remix licence, 13+ age requirement, a DMCA agent and process, a privacy notice, and NCMEC reporting procedures documented (low likelihood without uploads, but required).

## 8. Identity, abuse control, and rate limits

### 8.1 Sessions without accounts
The first mutating action (draft, AI, publish, report) needs a **Turnstile** token. The server then issues an HttpOnly, Secure, SameSite=Lax cookie holding a random 128-bit device session id, signed and valid for 90 days. `creator_key = sha256(session id)`. The session id is also shown once as a **manage link** at publish (`/m/{id}?k=…`), so creators can delete Things from another browser. No emails, passwords, or IP storage.

### 8.2 Limits (initial; tuned in alpha)
| Action | Per device session | Per IP (sliding) | Global |
|---|---|---|---|
| AI edits | 20 / day | 60 / day | Daily USD cap (alpha $25, beta $150), then AI disabled with a message |
| Draft uploads | 200 / day | 600 / day | — |
| Publishes | 30 / day | 100 / day | `PUBLISH_ENABLED` |
| Reports | 20 / day | 60 / day | — |

These use the Workers Rate Limiting binding for sliding windows and the `ai_spend_daily` table for the global cap. The table is updated after each AI call with actual token usage priced at list rates.

## 9. Sharing and embed model

- **Canonical link:** `https://things.example/t/{id}`. The id is 8 base62 characters (about 2×10¹⁴ space, randomly generated, with collision retry).
- **Unfurls:** `og:title`, `og:description` ("An interactive Thing, tap to play · remixed from …"), `og:image` = `/og/{id}.png`, `twitter:card=summary_large_image`.
- **Poster cards are server-generated from immutable metadata** (title, nickname, emoji knob or a palette colour taken from the knobs, the "▶ Tap to play" mark), rendered as SVG and converted to PNG in the Worker (resvg-wasm), and cached forever. **They contain no user-controlled pixels**, only text we render, so unfurls in chat apps can't be used to push images. Real screenshots or animated previews of the Thing itself are deferred to M1, and only for Things passing a popularity threshold, to keep headless-browser costs proportional to value.
- **Embed contract (stable from M0):** `https://things.example/e/{id}` with an optional `?autoplay=0|1` (default 0, meaning click-to-play). It always shows attribution plus Remix, and it can't be styled away. `frame-ancestors *`. This URL and its parameters are a public contract; later additions must stay backward-compatible.
- **Share action:** Web Share API where available, with copy link and copy embed as fallbacks.

## 10. AI integration

### 10.1 Boundary
- **Where it runs:** only in the player-domain Worker, via the Anthropic TypeScript SDK and a server-held API key. Browsers never talk to the model provider directly.
- **Inputs:** the current draft source (≤ 64 KB, typically ≤ 15 KB), the knob manifest, the user's instruction (≤ 300 chars), and optionally the last runtime error.
- **Outputs:** structured edits (a list of search/replace hunks against the source, plus a `summary` of ≤ 100 chars and an updated manifest). A full rewrite is allowed only when the source is under 4 KB. The Worker applies the hunks, re-validates the size and manifest, stores a draft, and returns the new sha and summary.
- **No tools, no browsing, no memory across users.** Each call is stateless. Nothing a model call returns has authority beyond becoming a draft that runs in the sandbox.
- **Streaming:** responses stream to the Worker, and the Worker relays a coarse progress state to the client (Server-Sent Events) for perceived latency.

### 10.2 Calls
| Call | Trigger | Model tier | Est. tokens (in / out) |
|---|---|---|---|
| **Edit** | Ask AI or an idea chip | Chosen by the M-1 eval (candidates: Opus 5, Opus 5.5, Sonnet 5, Haiku 4.5) | ~7 k (2 k cached system prompt) / ~1.5 k + thinking |
| **Auto-fix** | The user taps "Fix it" after an error | Same as Edit | ~7 k / ~1 k |
| **Idea chips** | First Remix open of a source (cached per sha) | Haiku 4.5 | ~5 k / ~150 |
| **Publish review** | Each publish (text-only delta for knob remixes) | Haiku 4.5 | ~4 k / ~50 |

**System prompt duties (Edit):** make only the requested change; keep Things within `thing/0` capabilities; never add network, storage, or external resources; expose new tunables as knobs; keep the source small; return a one-line honest summary; refuse policy-violating requests with a short explanation. The system prompt is byte-stable so prompt caching applies.

### 10.3 Prompt injection stance
A Thing's source is untrusted text that ends up in the model's context. An injected instruction can only influence the *new draft*, which (a) is previewed by the person who asked for it, (b) runs under the same sandbox, and (c) goes through publish review. The model sees no secrets and no other users' data. The residual risk ("remixing X quietly adds offensive text") is handled like any other content risk.

### 10.4 Quality eval (the gate before launch)
Build a fixed set of 50 edit tasks over the seed Things (e.g. "add a score", "make it two-player on one screen", "change the theme to space") with automatic checks (the result parses, stays under the size cap, runs 5 s in headless Chromium without errors, and the manifest is valid) plus a blind human rating of instruction-following (1–5). The pass bar is **≥ 85 % run without errors and a median rating ≥ 4**. Every candidate model is run to measure cost per successful edit. The cheapest model that passes becomes the default. The choice goes to the approver with the numbers.

## 11. Operating cost model

Prices are list prices as of 2026-09: Cloudflare Workers Paid ($5/mo, 10 M requests and 30 M CPU-ms included), R2 ($0.015/GB-mo, zero egress, 1 M Class A / 10 M Class B ops free), D1 (25 B row reads and 50 M row writes included), and Claude per MTok (input/output): Haiku 4.5 $1/$5, Sonnet 5 $2/$10, Opus 5.5 $4/$20, Opus 5 $5/$25.

### 11.1 Per-unit AI cost (estimate; the M-1 eval replaces this with measurements)
The assumptions are 5 k uncached input + 2 k cached input + 1.5 k output + ~1 k thinking per edit.

| Model | ≈ $/edit | ≈ $/published AI remix (avg 2.5 edits + review + chips) |
|---|---|---|
| Haiku 4.5 | $0.018 | $0.05 |
| Sonnet 5 | $0.035 | $0.10 |
| Opus 5.5 | $0.07 | $0.19 |
| Opus 5 | $0.09 | $0.23 |

Knob-only remixes cost about $0.002 (the review delta only). **If half of all remixes are knob-only, the blended cost per published Thing roughly halves.** The V8 target (≤ $0.15) is met by Sonnet 5 or Haiku 4.5 as the edit model; Opus-class models need a high knob-only share or tighter per-session limits.

### 11.2 Monthly scenarios
| Scenario | Opens/mo | Published/mo | AI edits/mo | Infra | AI (Sonnet 5) | AI (Opus 5) |
|---|---|---|---|---|---|---|
| Closed alpha (M0) | 10 k | 800 | 2 k | $5 | ~$80 | ~$200 |
| Public beta (M1) | 1 M | 20 k | 50 k | ~$5–15 | ~$1,900 | ~$4,800 |
| Growth (M2) | 20 M | 300 k | 600 k | ~$40–80 | ~$23 k | ~$58 k |

Infrastructure stays roughly flat because viewing is served almost entirely from cache: about 3 Worker requests per open, with static assets free, R2 egress free, and D1 writes only on publish. **AI is more than 95 % of the variable cost at every scale.** The levers, in order, are: knob-first UX (free remixes), per-device allowances, prompt caching, the cheaper model when the eval allows, a size cap that keeps inputs small, and eventually a Pro tier.

## 12. Tech stack and repository layout

| Layer | Choice | Why |
|---|---|---|
| Hosting | Cloudflare Workers + static assets, D1, R2, Turnstile, Rate Limiting, Analytics Engine, Access | Near-zero idle cost, zero egress, edge cache, and all the bot/abuse/admin primitives we need from one vendor. The lock-in risk is acceptable: the data is SQLite plus plain HTML blobs. |
| Language | TypeScript (strict) | One language across the Workers, UI, prelude, and tests |
| Server routing | Hono | Small, Workers-native |
| Player shell | Vanilla TS | Kept to a ≤ 40 KB gz budget; the first frame matters most |
| Editor | Preact + CodeMirror 6 (lazy-loaded, desktop code view) | Small; the editor loads only on Remix |
| Model API | `@anthropic-ai/sdk` | Official SDK |
| Tests | Vitest (unit), Playwright (Chromium/WebKit/Firefox: e2e and the hostile suite), plus the AI eval harness | Security and quality gates are automated |

```
/apps/player      Worker: pages, API, OG images, static assets (shell + editor)
/apps/content     Worker: /r/:sha only
/packages/format  thing/0 manifest schema, knob validation, prelude source (shared)
/packages/seeds   seed Things (source + default knobs)
/tests/hostile    adversarial Things + Playwright suite
/tests/ai-eval    edit-task set + harness
```

## 13. Path to the public library, API, and SDKs (designed for, not built)

| Future capability | What M0 already does to keep it cheap later | When |
|---|---|---|
| **Public library and search** | The `status` and `rating` columns exist; the moderation pipeline and admin tooling exist; the metadata is small and indexable (D1 FTS5 on title and knob text is enough to start) | M2 |
| **oEmbed + richer embeds** | The embed URL contract is stable; posters exist | M1 |
| **Public read API** (`GET /v1/things/:id`, `/lineage`, `/remixes`, later `/search`) | The internal `/api/v1` is already shaped as the public contract (JSON, cursor pagination, no internal fields); adding API keys, quotas, and attribution rules is the only new work | M3 |
| **Open format and player** (`thing/1` spec + open-source prelude/player so third parties can self-host playback under the same policy) | `format` versioning, a single prelude package, and the hostile test suite (which becomes the conformance suite) | M3 |
| **Picker SDK** (web component, then React Native) | Depends on search and ratings; the embed player is the rendering primitive | M4 |
| **Native rendering without webviews** | Evaluate the WASM-interpreter option (§3.4) only if partners demand it | M4+ |
| **Accounts** (claim your Things, cross-device) | `creator_key` can be attached to an account later without migrating Things | M1–M2, only if users ask |

Third-party requirements to be ready by M3, and deliberately *not* built before: API keys and quotas, content ratings (G/PG/…), attribution rules, SLA and status page, abuse contact, and partner takedown webhooks.
