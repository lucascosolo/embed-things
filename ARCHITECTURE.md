# Architecture — "Things" (working title)

Status: reviewed draft, awaiting approval · Last updated: 2026-09-27
Companion documents: [PRODUCT_SPEC.md](PRODUCT_SPEC.md), [ROADMAP.md](ROADMAP.md). The pre-implementation review record is in ROADMAP §6.

---

## 1. Principles

1. **The sandbox is the security boundary.** We never rely on inspecting, linting, or AI-reviewing code to keep users safe. Every Thing is treated as hostile. Review exists to enforce *content policy*, not for *containment*.
2. **Content is immutable, and only moderation state changes.** A Thing's source and knob values never change after publishing. The only mutable fields are `status` (live/removed/deleted) and the denormalised `remix_count`. Sources can therefore be cached aggressively, with explicit purges on takedown.
3. **The cheapest request never reaches the database.** The player shell is static, and Thing sources are content-addressed and edge-cached. Hot-path dynamic work is limited to one small metadata read and event writes.
4. **AI is an optional, bounded, stateless service.** No AI call sits on the viewing path, and the loop still works (knobs, source editor) with AI switched off.
5. **Keep M0 minimal.** One vendor (Cloudflare), one language (TypeScript), one Worker, and one data store (D1). Other services are added only when a milestone needs them.
6. **Keep future contracts stable, but don't build them early.** Link shapes and the format version are fixed from day one because third parties will depend on them later. The API, embed, and SDK surfaces are not built yet.

## 2. System overview (M0)

```
                     ┌─────────────────────────────────────────────────────────────┐
                     │ ONE Cloudflare Worker, two hostnames (branch on Host header)│
                     │                                                             │
  Browser / chat ──▶ │ PRODUCT HOST  things.example                                │
  webview            │   static: player shell, editor bundle (lazy), seed images   │
                     │   GET  /                 home (seed grid)                   │
                     │   GET  /t/:id            player HTML + inlined metadata + OG│
                     │   GET  /t/:id/remix      editor                             │
                     │   POST /api/v1/ai/edit   AI edit (Turnstile-verified session)│
                     │   POST /api/v1/things    publish                            │
                     │   POST /api/v1/events    measurement events (batched)       │
                     │   POST /api/v1/reports, DELETE /api/v1/things/:id           │
                     │                                                             │
  <iframe sandbox>──▶│ CONTENT HOST  thingcontent.example  (separate registrable   │
                     │   domain; no cookies, no API, no secrets exposed)           │
                     │   GET  /r/:sha           prelude + published source         │
                     │   GET  /loader           prelude + draft loader (editor)    │
                     └───────────────┬───────────────────────────────┬─────────────┘
                                     │                               │
                            ┌────────▼─────────┐            ┌────────▼────────┐
                            │ D1 (SQLite)      │            │ Anthropic API   │
                            │ things, blobs,   │            │ (server-held key│
                            │ sessions, events,│            │  via Worker only)│
                            │ counters, flags  │            └─────────────────┘
                            └──────────────────┘
  Also used: Turnstile (bot check), Rate Limiting binding (bursts), cache + zone purge API.
```

One Worker on two hostnames means one deploy and one config. Isolation comes from the **separate registrable domain** and the browser sandbox, not from process separation on the server. The content-host code path never reads cookies, never calls the API, and never returns anything except Thing HTML.

## 3. The Thing format (`thing/0`)

### 3.1 Shape
A Thing's **source** is one UTF-8 HTML document of at most **32 KB**. AI-authored toys are typically 4–15 KB. The cap keeps loads fast, keeps AI inputs bounded (about 10 k tokens or fewer), and keeps storage trivial. The source embeds a manifest:

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
  const k = thing.knobs;                 // resolved values
  thing.on("knobs", (next) => { … });    // optional: live updates in the editor
  // … ordinary JS: DOM, Canvas 2D, SVG, CSS, Web Audio, pointer/keyboard events
</script>
```

- **Knob types:** `number`, `color`, `text` (≤ 200 chars), `toggle`, `choice` (≤ 12 options), `emoji`. At most 16 knobs.
- **Aspect ratios:** `1:1` (default), `4:5`, `9:16`, `16:9`. The player letterboxes.
- **Runtime contract (what a Thing may rely on):** DOM, CSS, SVG, Canvas 2D, Web Audio (starts after a tap inside the Thing), pointer, touch, and keyboard events, `requestAnimationFrame`, timers, `eval`/`Function`, and `crypto.getRandomValues`. WebGL may happen to work but isn't guaranteed.
- **Not available:**
  - network requests of any kind;
  - storage;
  - navigating the parent or top window;
  - popups and form submission;
  - camera, microphone, geolocation, and sensors;
  - clipboard and fullscreen;
  - external scripts, images, and fonts;
  - workers.

  The WebRTC caveat is covered in §5.4.

### 3.2 A published Thing = source blob + knob values + metadata
Knob **values** are stored apart from the source. A knob-only remix reuses its parent's source blob (same hash) and stores only new values. The most common remix therefore costs nothing in AI or storage, and "what changed" is an exact structured diff. The side effect: **one source blob can be shared by many Things** (this matters for takedowns, §7).

### 3.3 Versioning
`format` is required, and unknown major versions render as a placeholder. `thing/0` is explicitly **unstable** during M0–M2 and becomes the stable, published `thing/1` at M3. Things reference blobs by hash and keep metadata separately, so a migration can rewrite old sources into new blobs.

### 3.4 Why HTML rather than a custom DSL or engine
| Option | Verdict |
|---|---|
| **Single-file HTML + JS in a hard browser sandbox** (chosen) | Maximum expressiveness. LLMs are fluent in it, and there's no custom engine to build. A Thing is a file that runs anywhere under the same policy. Safety rests on the browser sandbox, the most heavily attacked and patched isolation layer available. |
| Declarative DSL / JSON scene graph | Safer by construction, but it needs an engine, an editor, and teaching the LLM a new language. That's months of work before any validation, and expressiveness is capped. Rejected for now; possible later as an optional authoring layer that compiles to HTML. |
| JS in a WASM interpreter (e.g. QuickJS) with a custom draw API | Strong isolation, and it could render in native SDKs without a webview. But there's no DOM, the API is custom, and Things would be more limited. Recorded as an M4+ option for native rendering. |
| Fixed templates with parameters only | Safe and trivial, but it can't test whether AI-assisted changes matter. Rejected. |

## 4. Rendering

### 4.1 Published Things (player)
1. `GET /t/:id` returns a small HTML shell (≤ 40 KB gzipped JS budget) with the Thing's metadata inlined, so there's no second round-trip. The HTML is edge-cached for 60 s and purged on removal.
2. The shell creates:
   ```html
   <iframe src="https://thingcontent.example/r/{sha}#k={base64url(knob values)}"
           sandbox="allow-scripts" referrerpolicy="no-referrer" title="{title}"></iframe>
   ```
3. The content host serves `/r/:sha` as the **prelude** (a small versioned script) plus the source blob from D1.
   - The `Sec-Fetch-Dest` check runs *before* any cache lookup (§5.2).
   - The response is edge-cached for a long time through the Worker's cache and gets `Cache-Control: public, max-age=86400` for browsers.
   - Knob values travel in the fragment, which never reaches the server, so one cached response serves every knob variant.
   - The fragment is used **only for the initial load**.
4. The prelude runs before any author code and does the following:
   - reads the fragment and the manifest, validates and clamps the knob values, and exposes `window.thing`;
   - applies the neutralisations in §5.4;
   - installs error reporting, a heartbeat, and first-interaction reporting;
   - posts `ready {sha}`.
5. The host verifies that `ready.sha` matches the expected sha. If it doesn't match, or if a second `ready` arrives, the host tears the frame down. This catches a Thing that navigates its own frame to other content.

### 4.2 Drafts (editor) without uploads
Drafts are **never stored or served by URL**:
1. The preview iframe loads the fixed `https://thingcontent.example/loader` (prelude plus a loader script), with the same headers and sandbox as `/r/:sha`.
2. The host posts `{load: {source, knobs, nonce}}`. The loader accepts it only from `window.parent` and only once. It then writes the source into its own document with `document.open()`/`document.write()`. The document keeps its CSP and sandbox, and the prelude has already run.
3. The prelude posts `ready {nonce}`, and the host checks the nonce the same way as the sha in §4.1.
4. **Knob changes** are sent as a `knobs` message. If the Thing registered `thing.on("knobs")`, it updates live. Otherwise the host reloads the loader and re-sends the source with the new knob values. The URL never changes, so no browser history entries pile up.
5. **Code changes** (AI or hand edits) reload the loader with the new source.

> **Verification item (M-1, S1):** confirm in Chromium, WebKit, Firefox, and the target in-app browsers that the CSP and sandbox survive `document.open()`/`write()`. **Fallback if they don't:** store the draft under an unguessable, `no-store`, 24-hour URL on the content host, keyed by the session, and exclude it from the cache.

### 4.3 Host ↔ Thing bridge (postMessage)
The bridge is deliberately tiny, and it's the only channel between host and Thing. The host checks that `event.source === iframe.contentWindow`, since the origin is opaque (`"null"`), and schema-validates every message.

| Direction | Message | Purpose |
|---|---|---|
| Thing → host | `ready {sha \| nonce}` | First-frame timing (V7); integrity check |
| Thing → host | `interact` (once) | First pointer or key event, from a capture-phase listener the prelude installs before author code runs (V1) |
| Thing → host | `heartbeat` (every 1 s) | Watchdog |
| Thing → host | `error {message, line}` | Error banner; pre-filled "Ask AI to fix" |
| Host → Thing | `knobs {values}` | Live knob updates (editor) |
| Host → Thing | `visibility {visible}` | Pause when hidden |
| Host → Thing | `prefs {muted, reducedMotion}` | Accessibility |
| Host → loader | `load {source, knobs, nonce}` | Draft preview (editor only) |

Author code runs in the same JS realm as the prelude, so it **can forge** `interact`, `heartbeat`, and `error`. That means V1 and V7 could in principle be gamed by authors. This is accepted for the alpha: the incentive is low, and the metrics are aggregated across many Things. Things can't ask the host to navigate, share, resize, or read anything. All remix, share, and attribution UI lives in the host, outside the Thing's pixels.

### 4.4 Watchdog and resource limits
If no heartbeat arrives for 3 s, the host shows "stopped responding" with Restart (reload the iframe) and Report.

**Honest limitation:** on iOS WebKit, and on typical mid-range Android devices, cross-site iframes usually share the page's renderer process and main thread. A busy loop can then freeze the whole tab, and the host's own UI can't draw. Only desktop Chrome and Firefox reliably isolate the frame. This is a residual risk on mobile, handled by review, reports, and fast removal. The 32 KB cap limits download size. Memory can't be bounded from JS, but browsers kill runaway frames.

## 5. Security and sandboxing

### 5.1 Threat model
| Actor | Goal | Primary control |
|---|---|---|
| Malicious Thing author | Steal viewer data or credentials, attack the host page, redirect viewers, mine crypto | Opaque-origin sandbox + CSP + separate domain + no network (§5.2); host-side `frame-src` limits frame navigation; sha check (§4.1) |
| Malicious Thing author | Track viewers or leak what they type | Mostly blocked; WebRTC residual (§5.4) |
| Malicious Thing author | Offensive, illegal, or deceptive visible content | Daily human review, reports, removal (§7) |
| Spammer / bot | Mass publishing, AI cost abuse | Turnstile, D1 daily counters, spend cap reserved before each call (§8) |
| Prompt injection via a Thing's source | Make the AI produce something harmful | The AI has no tools, secrets, or cross-user data; its output is a draft that runs in the same sandbox and is covered by review (§10.3) |
| Attacker loading the content host at top level | Host phishing pages on our domain | `Sec-Fetch-Dest: document` gets a 403 before the cache; CSP `sandbox` applies even at top level; the domain holds nothing of value |

### 5.2 Content-host response headers (`/r/:sha` and `/loader`)
```
Content-Type: text/html; charset=utf-8
Content-Security-Policy:
  sandbox allow-scripts;
  default-src 'none';
  script-src 'unsafe-inline' 'unsafe-eval';
  style-src 'unsafe-inline';
  img-src data: blob:;  media-src data: blob:;  font-src data:;
  connect-src 'none';
  frame-src 'none'; child-src 'none'; worker-src 'none';
  form-action 'none';
  base-uri 'none';
  frame-ancestors https://things.example
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), usb=(), serial=(),
  bluetooth=(), hid=(), clipboard-read=(), clipboard-write=(), fullscreen=(), display-capture=(),
  accelerometer=(), gyroscope=(), magnetometer=()          (defence in depth; mostly Chromium)
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
Vary: Sec-Fetch-Dest
```

Why each part is there:

- **Opaque origin everywhere.** The iframe's `sandbox="allow-scripts"` plus the CSP `sandbox` directive give an opaque origin: no cookies, no storage, and no same-origin access. That holds even if the URL is loaded directly. The sandbox never grants `allow-same-origin`, `allow-top-navigation*`, `allow-popups`, `allow-forms`, `allow-modals`, `allow-downloads`, or `allow-pointer-lock`.
- **`'unsafe-eval'` is intentional.** Inline script is already allowed, so blocking `eval` would add no security and would break some AI-written code. `script-src` still blocks remote, `data:`, and `blob:` scripts, and `worker-src 'none'` blocks workers.
- **Separate registrable domain** (not a subdomain). There are no shared cookies or same-site privileges, and there's a reputation boundary: a bad Thing can't get the product domain blocklisted.
- **`frame-ancestors https://things.example`.** This limits framing to our own player in M0. It is checked against *every* ancestor, so **the M1 embed feature must relax it to `*`**. After that, anyone could frame raw Things without attribution. That's acceptable because attribution isn't a security property (see ROADMAP M1).
- **Frame self-navigation.** A Thing can still navigate its own frame (`location = …`, meta refresh). What limits the destinations is the **product host's CSP `frame-src https://thingcontent.example`**, which blocks external, `data:`, and `javascript:` destinations. The sha/nonce check (§4.1) catches navigation to other content-host documents. (The `navigate-to` directive was removed from the CSP spec and is not used.)
- **`Sec-Fetch-Dest`.** The Worker checks it *before* consulting the cache, and requests with `document` get a 403 and a link to the player. Browsers that don't send it (pre-2023) still get the CSP sandbox.

### 5.3 Product-host protections
- Strict nonce-based CSP with no inline scripts.
- `frame-src https://thingcontent.example` and `frame-ancestors 'self'` (M0 has no embeds).
- Knob text shown in host UI is always rendered as text, never as HTML.
- Session cookies are `HttpOnly; Secure; SameSite=Lax`.

### 5.4 Known gaps and residual risks
| Gap | Detail | Decision |
|---|---|---|
| **WebRTC bypasses `connect-src`** | No shipping browser lets CSP block `RTCPeerConnection`. The prelude deletes the constructors, but a Thing can get pristine ones back from a new `about:blank` or `srcdoc` iframe, which involves no fetch, so `frame-src` doesn't stop it. That iframe shares the Thing's opaque origin, so its constructors are reachable. A hostile Thing can therefore contact a STUN/TURN server, which reveals the viewer's IP address, and can leak text typed into it. | **"No network" is best-effort, and we say so.** Mitigations: the prelude neutralises the constructors on its own window and on `HTMLIFrameElement` creation paths (`document.createElement`, `innerHTML` observers), which raises the bar; daily review flags `RTC`/`stun:`/`iceServers`; hostile tests document which bypasses remain. There's no credential or cross-site data exposure, because Things never have anything except their own inputs. Revisit when a CSP `webrtc` directive ships. |
| DNS prefetch hints | `<link rel=dns-prefetch>` can leak a DNS lookup in some browsers | Low impact. The prelude removes `link[rel]` hints; also flagged in review. |
| CPU and battery abuse | Busy loops, heavy rendering | §4.4; pause when hidden |
| Visual deception | A Thing can *draw* a fake login, but can't send the data or navigate anywhere. The risk is persuasion to act outside the Thing. | Content policy, review, reports. The host frames every Thing as user content ("Made by a user · Report"). |
| Stale cached content after takedown | Browser caches keep a copy for up to 1 day | Purge the edge copy; accept the browser residue (§7) |
| Browser zero-days | Sandbox escape | Out of our control. The content domain holds nothing of value. |

### 5.5 Security verification (a gating deliverable)
A **hostile Things suite** of 25 or more adversarial Things runs under Playwright in CI on Chromium and WebKit, with Firefox run manually each milestone. Each Thing must *fail* at its goal, unless the test is marked as a documented residual (WebRTC via a fresh frame).

The attempts covered:

- **Exfiltration:** fetch, XHR, WebSocket, EventSource, sendBeacon, and image-pixel requests.
- **Forms:** form submission.
- **Navigation and popups:**
  - top navigation;
  - `window.open`;
  - frame navigation to an external URL, to `data:`, and to `javascript:`;
  - frame navigation to another `/r/:sha` or to `/loader`, which the sha check must catch.
- **Isolation:**
  - parent DOM access;
  - cookies, localStorage, and IndexedDB;
  - `document.domain`;
  - service worker registration.
- **Nested frames:** WebRTC via the prelude-neutered constructors, via `about:blank`, and via `srcdoc` frames.
- **Direct loads:** a top-level load of `/r/:sha`, including after it's cached.
- **Malformed input:** oversized source, a malformed manifest, and knob text containing HTML (the host must not render it).
- **Bridge abuse:** a postMessage from a non-child window, and a loader receiving a second `load`.

The suite is also run **manually on real devices**: iOS Safari and the in-app browsers of iMessage, WhatsApp, Instagram, Discord, and Telegram. That happens in M-1 and before every milestone gate.

## 6. Data model (D1 only in M0)

### 6.1 Schema
```sql
CREATE TABLE blobs (                 -- content-addressed Thing sources
  sha         TEXT PRIMARY KEY,      -- sha256 hex of source
  source      TEXT NOT NULL,         -- ≤ 32 KB
  blocked     INTEGER NOT NULL DEFAULT 0,  -- 1 ⇒ never served (policy-violating source)
  created_at  TEXT NOT NULL          -- ISO-8601 UTC
);

CREATE TABLE things (
  id            TEXT PRIMARY KEY,    -- 8-char base62, random
  source_sha    TEXT NOT NULL REFERENCES blobs(sha),
  knob_values   TEXT NOT NULL,       -- JSON, validated against the manifest
  title         TEXT NOT NULL,       -- ≤ 80 chars
  nickname      TEXT,                -- ≤ 32 chars, optional, unverified
  parent_id     TEXT REFERENCES things(id),
  seed_id       TEXT NOT NULL,       -- the seed at the root of this lineage (self for seeds)
  depth         INTEGER NOT NULL,    -- 0 for seeds
  change_kind   TEXT NOT NULL,       -- 'seed' | 'knobs' | 'source'
  change_notes  TEXT,                -- JSON array of AI one-line summaries (≤ 8); shown in M1
  edit_counts   TEXT NOT NULL,       -- JSON {"knob":n,"ai":n,"hand":n}
  aspect        TEXT NOT NULL,
  status        TEXT NOT NULL DEFAULT 'live',  -- live | removed | deleted
  rating        TEXT,                -- reserved for M2 content ratings; NULL until then
  remix_count   INTEGER NOT NULL DEFAULT 0,    -- direct live children
  creator_key   TEXT NOT NULL,       -- sha256(viewer id) of the publisher
  created_at    TEXT NOT NULL
);
CREATE INDEX things_parent  ON things(parent_id, created_at);
CREATE INDEX things_creator ON things(creator_key, created_at);
CREATE INDEX things_source  ON things(source_sha);

CREATE TABLE sessions (              -- Turnstile-verified viewers allowed to mutate
  viewer_key TEXT PRIMARY KEY, verified_at TEXT NOT NULL
);

CREATE TABLE share_refs (
  ref TEXT PRIMARY KEY, thing_id TEXT NOT NULL, sharer_key TEXT NOT NULL, created_at TEXT NOT NULL
);

CREATE TABLE events (                -- measurement; pruned after 180 days
  id INTEGER PRIMARY KEY, at TEXT NOT NULL, viewer_key TEXT NOT NULL,
  kind TEXT NOT NULL,                -- view | ready | interact | editor_open | ai_edit | publish | share | error
  thing_id TEXT, ref TEXT, value_ms INTEGER, detail TEXT
);
CREATE INDEX events_kind_at ON events(kind, at);

CREATE TABLE counters (              -- daily limits: key = '<action>:<scope>:<id>:<YYYY-MM-DD>'
  key TEXT PRIMARY KEY, n INTEGER NOT NULL
);

CREATE TABLE ai_spend_daily (        -- global cost circuit breaker
  day TEXT PRIMARY KEY, reserved_usd_micros INTEGER NOT NULL DEFAULT 0,
  settled_usd_micros INTEGER NOT NULL DEFAULT 0, calls INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE reports (
  id TEXT PRIMARY KEY, thing_id TEXT NOT NULL, reason TEXT NOT NULL, detail TEXT,
  reporter_key TEXT NOT NULL, created_at TEXT NOT NULL, resolved_at TEXT, resolution TEXT
);

CREATE TABLE flags (                 -- kill switches, changeable without a deploy
  name TEXT PRIMARY KEY, value TEXT NOT NULL   -- publish_enabled, ai_enabled, ai_daily_cap_usd
);
```
There are no user tables, emails, or stored IP addresses. Network-scoped counters use a daily-salted hash of the IP /24 (IPv4) or /48 (IPv6), and those rows are deleted after 2 days.

### 6.2 Why D1 only
At M0 volumes (hundreds to low thousands of Things; sources ≤ 32 KB), a single SQLite database keeps sources, lineage, counters, and events together. It's queryable with plain SQL, and there's one binding. The limits are known:

- D1 caps a database at 10 GB.
- Events grow with views.

**Move sources to R2 and events to an analytics store at M1 or M2** if storage passes about 2 GB or event writes pass about 20 M a month. Both moves are mechanical because sources are content-addressed and events are append-only.

### 6.3 Caching
| Resource | Policy |
|---|---|
| `/r/:sha` | Worker cache API + edge, long TTL; browser `max-age=86400`; purged on takedown via the zone purge API. The cache API's own delete is per-data-centre only, so it isn't enough. |
| `/loader`, static assets | Hashed or immutable |
| `/t/:id` HTML | Edge 60 s + `stale-while-revalidate`; purged on status change |
| Seed OG images | Static, generated at build time |

### 6.4 Lineage
- **Ancestors:** a `WITH RECURSIVE` walk up `parent_id`, capped at 50 and computed at render. It's cheap, since pages are cached for 60 s. The player shows the nearest 2 plus the seed.
- **Children (M1 UI):** `WHERE parent_id = ? AND status = 'live' ORDER BY created_at DESC` with a cursor.
- **Integrity:** rows are never hard-deleted. `deleted` and `removed` Things become tombstones that keep their place in the lineage. Their children keep working unless they share a *blocked* source (§7).

## 7. Trust and safety (M0)

1. **Daily human review of every publish.** A script lists the previous day's new Things with their title, nickname, text knob values, source string literals, and a flag for suspicious patterns (`RTC`, `stun:`, `link rel`, obfuscation). Each one gets a sandboxed preview link. The expected alpha volume is about 10–40 a day. Automated model-based review arrives in M1, when volume makes human review impractical.
2. **Reports** go into the `reports` table and send an alert (email or webhook) to the team. There's no automatic hiding, which prevents griefing.
3. **Removal** (CLI script):
   - Set `status='removed'`.
   - If the *source itself* violates policy, set `blobs.blocked=1` and tombstone every Thing sharing that sha.
   - Purge `/t/:id` and `/r/:sha` via the zone purge API.
   - Decrement the parent's `remix_count`.

   A cleanly removed Thing disappears from edge caches within seconds. Copies in browser caches expire within 1 day.
4. **Self-delete:** the publishing browser can delete (matched by `creator_key`). The steps are the same, except the blob is blocked only if no other live Thing uses it.
5. **Kill switches:** `publish_enabled` and `ai_enabled` in the `flags` table are read on each mutating request. They're changeable in seconds with one SQL statement, and no deploy is needed.
6. **Legal groundwork, before the public beta (M1):**
   - ToS with a remix licence;
   - 13+ age requirement;
   - DMCA agent;
   - privacy notice;
   - documented NCMEC reporting procedure (unlikely to be needed without uploads, but required).

## 8. Identity, limits, and measurement

### 8.1 Viewer identity without accounts
- On first view, the product host sets a random 128-bit **viewer id** in a signed, `HttpOnly; Secure; SameSite=Lax` cookie that lasts 1 year. `viewer_key = sha256(viewer id)` is what gets stored.
- The first mutating action (AI edit, publish, report, delete) requires a **Turnstile** token. That marks the `viewer_key` as verified in `sessions`.
- There are no emails, passwords, or manage links. Deletion only works from the browser that published, or by emailing the team.

### 8.2 Limits
Daily limits use D1 `counters`, updated with an upsert on every mutating call. (The Workers Rate Limiting binding only supports 10 s and 60 s windows, so it's used only for burst protection: 10 requests in 10 s per viewer on the AI and publish endpoints.)

| Action | Per viewer / day | Per network (/24 or /48) / day | Global |
|---|---|---|---|
| AI edits | 20 | 60 | Daily USD cap from `flags` (alpha $25, beta $150), then AI is disabled with a message |
| Publishes | 30 | 100 | `publish_enabled` |
| Reports | 20 | 60 | — |

**Reserve-then-settle spend cap.** Before each AI call, the Worker runs one atomic statement:

```sql
UPDATE ai_spend_daily SET reserved_usd_micros = reserved_usd_micros + :max_cost
WHERE day = :today AND reserved_usd_micros + :max_cost <= :cap RETURNING …
```

`:max_cost` is the worst case: the input-token estimate plus `max_tokens`, at list price. If no row comes back, AI is off for the day. After the call, the actual cost is settled and the unused reservation released. Concurrent calls can't overshoot the cap. An alert fires at 50 % of the cap.

**Input cap.** A source must be ≤ 32 KB. Requests estimated at more than 12 k input tokens are rejected, and so are instructions longer than 300 characters.

**Accepted reality.** Turnstile-solving services plus residential proxies can get past the per-viewer and per-network limits. The global cap is the real backstop, and it bounds the worst case at the daily cap. The trade-off: under attack, AI goes offline for real users until the next day. Knobs keep working.

### 8.3 Measurement
- **Share refs.** Share and copy-link create a `share_refs` row and append `?r={ref}` to the link.
- **Events.** The player and editor send batched events (`view`, `ready` with ms, `interact`, `editor_open`, `ai_edit`, `publish`, `share`, `error`) to `/api/v1/events`, which writes them to the D1 `events` table.
- **Analysis.** Metrics are computed with saved SQL queries (`wrangler d1 execute`, or an exported file analysed in DuckDB). M0 has no dashboard.
- **Recipient view.** A view with a ref where `viewer_key` ≠ the ref's `sharer_key` and ≠ the Thing's `creator_key`. See PRODUCT_SPEC §10.1 for the caveats.
- **Consent check (before the alpha).** Get a one-page legal read on whether the viewer-id cookie, used for security, rate limiting, and first-party aggregate product measurement, needs consent in the EU and UK. **Fallback if it does:**
  - set the cookie only on the first mutating action (strictly necessary for abuse prevention);
  - measure pure viewers per ref using a daily-salted, non-stored hash;
  - accept that V1 and V6 will then be coarser.

## 9. Sharing model

- **Canonical link:** `https://things.example/t/{id}`. The id is 8 random base62 characters (about 2×10¹⁴ possible values), with a retry on collision. Share refs are appended as `?r=…` and don't change the canonical URL (`<link rel=canonical>`).
- **Unfurls:**
  - `og:title` is the Thing's title.
  - `og:description` is "An interactive Thing: tap to play · remixed from …".
  - `og:image` is the **seed's pre-rendered image**, captured at build time from the seed with Playwright.
  - `twitter:card=summary_large_image`.

  No user-controlled pixels reach chat previews. Per-Thing and animated previews are M1, and only for popular Things.
- **Sharing:** the Web Share API where available, with copy-link as the fallback.
- **Embed (M1, contract reserved now):** `https://things.example/e/{id}?autoplay=0|1`. It will always show attribution and a Remix button, and it requires relaxing the content host's `frame-ancestors` (§5.2). The path is reserved now so it's never used for anything else.

## 10. AI integration

### 10.1 Boundary
- **Where:** only in the Worker, through the Anthropic TypeScript SDK with a server-held key. Browsers never contact the model provider.
- **Inputs:** the current source (≤ 32 KB), the user's instruction (≤ 300 chars), and optionally the last runtime error (pre-filled by "Ask AI to fix").
- **Output:** one structured result:
  - a list of search/replace hunks against the source (a full rewrite is allowed if the source is under 4 KB);
  - a `summary` of ≤ 100 chars;
  - the updated manifest.

  The Worker applies the hunks, re-validates the size and manifest, and returns the new source and summary to the editor. The editor then previews it through the loader. Nothing is stored until publish.
- **Stateless and tool-free.** There are no tools, no browsing, and no cross-user context. The response is a plain request/response with an indeterminate progress UI on the client (no streaming in M0).

### 10.2 Calls in M0
| Call | Trigger | Model | Est. tokens (in / out) |
|---|---|---|---|
| **Edit** | Ask AI (including "Ask AI to fix") | Chosen per §10.4 from Haiku 4.5, Sonnet 5, Opus 5.5, Opus 5 | ~7 k (~2 k cacheable system prompt) / ~1.5 k + ~1 k thinking |

That's the only call. Idea chips and model-based publish review are M1.

**System prompt duties:**
- Make only the requested change.
- Stay within the `thing/0` capabilities, and never add network, storage, or external resources.
- Expose new tunable values as knobs.
- Keep the source small.
- Return an honest one-line summary.
- Decline policy-violating requests briefly.

The prompt is byte-stable so prompt caching can apply. S2 checks that it meets the model's minimum cacheable length. At alpha traffic, many calls will miss the cache's TTL, and the cost model assumes that.

### 10.3 Prompt injection stance
A Thing's source is untrusted text inside the model's context. An injected instruction can only shape the *new draft*, and that draft:

- is previewed by the person who asked for it;
- runs in the same sandbox;
- is subject to review once published.

The model sees no secrets and no other users' data. The remaining risk ("remixing X quietly adds offensive text") is handled like any other content risk.

### 10.4 Quality eval and model choice
- **Task set:** 50 fixed edit tasks over the seeds, for example "add a score", "two players on one screen", "space theme", and "fix this error".
- **Automatic checks:** the hunks apply, the size cap holds, the manifest is valid, and the result runs for 5 s in headless Chromium **under the real CSP** with no errors.
- **Human check:** a blind 1–5 rating of instruction-following.
- **Pass bar:** ≥ 85 % run clean and a median rating ≥ 4.
- **Selection rule (the same in every document):** *the cheapest model that passes the eval and keeps projected V8 ≤ $0.15*. The approver sees the measured numbers for every candidate and makes the final call.
- **When it runs:** manually, before launch and before any model or system-prompt change. It isn't in CI, because it costs money on every run.

## 11. Operating cost model

List prices as of 2026-09:

- **Cloudflare:** Workers Paid is $5/mo and includes 10 M requests and 30 M CPU-ms. D1 includes 25 B row reads, 50 M row writes, and 5 GB. Turnstile is free.
- **Claude, per million tokens (input/output):** Haiku 4.5 $1/$5; Sonnet 5 $2/$10; Opus 5.5 $4/$20; Opus 5 $5/$25. Cache reads are about 10 % of the input price.

### 11.1 AI cost per edit and per published Thing (estimates; S2 replaces them with measurements)
**Assumptions:**
- Each edit uses 5 k uncached input, 2 k cached input, 1.5 k output, and 1 k thinking tokens.
- AI remix sessions average 2.5 edits, including fixes.
- 35 % of editor sessions publish (V3), and **abandoned sessions still cost money**.
- Half of all remixes are knob-only and cost $0.

| Model | $/edit | $ per published AI remix (2.5 edits ÷ 0.35) | **$ per published Thing, blended with 50 % knob-only (V8)** |
|---|---|---|---|
| Haiku 4.5 | $0.018 | $0.13 | **$0.06** |
| Sonnet 5 | $0.035 | $0.25 | **$0.13** |
| Opus 5.5 | $0.071 | $0.51 | **$0.25** |
| Opus 5 | $0.089 | $0.63 | **$0.32** |

Cache-write premiums and cache misses at low traffic add roughly 5–10 %. **Only Haiku- and Sonnet-class models meet V8 ≤ $0.15.** An Opus-class model needs V8 raised to about $0.25–0.35, or the edit allowance cut. This is the approver's decision.

### 11.2 Monthly scenarios
| Scenario | Opens/mo | Published/mo | AI edits/mo | Infra | AI @ Haiku 4.5 | AI @ Sonnet 5 | AI @ Opus 5.5 |
|---|---|---|---|---|---|---|---|
| Closed alpha (M0) | 10 k | 800 | 2 k | $5 | ~$40 | ~$75 | ~$150 |
| Public beta (M1) | 1 M | 20 k | 50 k | ~$5–15 | ~$900 | ~$1.8 k | ~$3.6 k |
| Growth (M2) | 20 M | 300 k | 600 k | ~$50–120 (R2 + analytics store by then) | ~$11 k | ~$21 k | ~$43 k |

Infrastructure stays nearly flat. Viewing is served mostly from cache, at roughly 3 Worker requests and 3 event writes per open, and static assets are free. **AI is more than 90 % of variable cost at every scale.** The cost levers, in order of impact:

1. Knob-first UX (free remixes).
2. Per-viewer allowances.
3. The model the eval allows.
4. The size cap (bounded inputs).
5. Prompt caching.
6. Eventually, a Pro tier.

The daily caps bound worst-case spend at $25/day in the alpha and $150/day in the beta.

## 12. Tech stack and repository layout

| Layer | Choice | Why |
|---|---|---|
| Hosting | Cloudflare: one Worker with static assets, D1, Turnstile, Rate Limiting binding (bursts only), zone purge API | Near-zero idle cost, edge cache, and bot protection from one vendor. The lock-in risk is acceptable: the data is SQLite plus plain HTML. |
| Language | TypeScript (strict) | One language for the Worker, UI, prelude, and tests |
| Routing | Hono | Small and Workers-native |
| Player shell | Vanilla TS | ≤ 40 KB gzipped budget, because the first frame matters most |
| Editor | Preact; a plain `<textarea>` for desktop source editing | Small, and loaded only when the user taps Remix |
| Model API | `@anthropic-ai/sdk` | Official SDK |
| Tests | Vitest (unit); Playwright (e2e + hostile suite on Chromium and WebKit in CI); AI eval harness (manual) | Security gates are automated; the cost-bearing eval isn't |

```
/src/worker        Hono app: product-host routes, content-host routes, API, limits, AI
/src/player        player shell (vanilla TS)
/src/editor        editor (Preact)
/src/format        thing/0 schema, knob validation, prelude + loader source
/seeds             seed Things + build-time OG image capture
/migrations        D1 schema
/scripts           review list, remove, metrics SQL
/tests/hostile     adversarial Things + Playwright suite
/tests/ai-eval     edit-task set + harness
```

## 13. Path to the library, API, and SDKs (designed for, not built)

| Future capability | What M0 already provides | Milestone |
|---|---|---|
| **Embed widget + oEmbed** | Reserved `/e/:id` path; attribution lives in the host UI. Requires `frame-ancestors *` on the content host. | M1 |
| **Per-Thing and animated previews** | Headless render (e.g. Cloudflare Browser Rendering), only for Things above a popularity threshold | M1 |
| **Automated review + admin UI** | The review script's logic becomes the prompt and rubric; `reports` and `status` already exist | M1 |
| **Public library and search** | `status` and `rating` columns; moderation workflow; small indexable metadata (D1 FTS5 over title and knob text is enough to start) | M2 |
| **Public read API** (`/v1/things/:id`, `/lineage`, `/remixes`, later `/search`) | Internal `/api/v1` already uses public-shaped JSON (cursor pagination, no internal fields). New work would be API keys, quotas, and attribution terms. | M3 |
| **Open format and player** (`thing/1` spec, open-source prelude/player for self-hosted playback) | `format` versioning, a single prelude package, and the hostile suite, which becomes the conformance suite | M3 |
| **Picker SDK** (web component, then React Native) | The embed player is the rendering primitive; depends on search and ratings | M4 |
| **Native rendering without webviews** | Evaluate the WASM-interpreter option (§3.4) only if partners require it | M4+ |
| **Accounts** | `creator_key` can attach to an account later without migrating Things | M1–M2, only if users ask |

The third-party prerequisites are needed by M3 and deliberately not built earlier: API keys and quotas, content ratings, attribution rules, an SLA and status page, an abuse contact, and partner takedown webhooks.
