# FEATURE: Events page + home events section, data sourced from Matrix polls

DOC PURPOSE: implementation spec. Read top to bottom. All facts are explicit.
ASSUMPTIONS: repo = rustporto.github.io (Jekyll, GitHub Pages, plugin
jekyll-multiple-languages-plugin 1.8.0, Node 22 in CI, Tailwind 4).

---

## SUMMARY

- Event data source: ONE Matrix room (call it EVENTS_ROOM).
- Each event = ONE Matrix poll posted in EVENTS_ROOM.
- RSVP = poll responses. Count = latest response per sender.
- Site reads Matrix at BUILD TIME via a Node script in GitHub Actions.
- Script output: `_data/events.json`.
- Jekyll renders that data via ONE shared include used by 2 consumers:
  - `events.html` page at `/events/` (all events)
  - home page `index.html` (next 3 upcoming events)
- No client-side JS. No token in browser. No runtime API calls from users.

---

## DATA FLOW

1. Human posts poll in EVENTS_ROOM (Element).
2. GitHub Actions build job runs `node .github/scripts/fetch-matrix-events.mjs`.
3. Script calls Matrix Client-Server API (CS API) with bot access token.
4. Script writes `_data/events.json`.
5. `jekyll build` renders `_includes/events.html` from `site.data.events`.
6. Deploy.

Freshness = last deploy. Daily cron added to keep it current.

---

## HARD CONSTRAINTS (MUST / MUST NOT)

- EVENTS_ROOM MUST be **unencrypted**. If encrypted, poll events arrive as
  `m.room.encrypted`, the bot cannot decrypt, RSVP reading FAILS.
- Bot account MUST be a member of EVENTS_ROOM (script also attempts join).
- Bot access token MUST be an Actions secret, never committed.
- Poll answers MUST be entered in the fixed order: Yes, Maybe, No (top to bottom).
  Mapping is BY POSITION (index), not by text/language.
- Poll SHOULD be a disclosed poll (Element default) so responses are visible.
- Poll question SHOULD follow the POLL QUESTION format in MATRIX ROOM
  CONVENTION (below). It is best-effort; non-conforming questions MUST NOT
  drop the event.
- Event title/location text is NOT translated. Same text shows in EN and PT.
- `_data/` is not processed by jekyll-multiple-languages-plugin; it is shared
  across all languages.
- Do NOT rely on a stable Matrix poll spec. Polls are NOT in the stable spec
  (MSC3381, rides on MSC1767 Extensible Events). Handle both stable and unstable
  event type names and content keys.

---

## MATRIX ROOM CONVENTION

EVENTS_ROOM:
- public, joinable
- unencrypted
- bot joined

Per event, post ONE poll:

POLL QUESTION format (best-effort parse):
```
<Title> · <YYYY-MM-DD HH:MM> · <Location>
```
- Field separator: `·` (U+00B7 MIDDLE DOT).
- Field 0 = Title. REQUIRED (falls back to whole question if split yields nothing).
- Date/time field: OPTIONAL. The first `·` field after the title that contains
  the date regex match below.
- Location field: OPTIONAL. The first `·` field after the title that is NOT the
  date field. Consequence: `<Title> · <Location>` (date omitted) still yields
  the location; `<Title> · <date>` yields an empty location.
- If no `YYYY-MM-DD` date found anywhere in question: start = null.
- Regex used for date: `(\d{4}-\d{2}-\d{2})(?:[ T](\d{2}:\d{2}))?`
- If only date (no time): time defaults to `00:00`.

POLL ANSWERS (order fixed):
1. Yes    -> bucket `yes`
2. Maybe  -> bucket `maybe`
3. No     -> bucket `no`

RSVP counting rule:
- Fetch all response events for the poll.
- Group by `sender`.
- Keep only the response with max `origin_server_ts` per sender.
- Map that response's first selection id -> index in poll answers -> bucket.
- Count per bucket. Empty selection = abstain (no bucket).

---

## FILES: NEW

| path | purpose |
|---|---|
| `.github/scripts/fetch-matrix-events.mjs` | Matrix reader; writes `_data/events.json` |
| `_data/events.json` | committed placeholder; CI overwrites at build |
| `_includes/events.html` | shared events renderer |
| `events.html` | Events page at `/events/` |
| `docs/matrix-events.md` | this document |

## FILES: MODIFIED

| path | change |
|---|---|
| `.github/workflows/jekyll.yml` | add `schedule` trigger + fetch step before Jekyll build |
| `index.html` | add home events section incl. `events.html limit=3 upcoming_only=true` |
| `_includes/header.html` | add nav link `{% tl events %}` |
| `_i18n/en.yml` | add `titles.events`, `nav.events`, new `events:` block |
| `_i18n/pt.yml` | same keys as en.yml |
| `_config.yml` | add `docs` to `exclude` (this doc must not be published) |

---

## FILE: .github/scripts/fetch-matrix-events.mjs

```js
// Fetches event polls from a Matrix room and writes _data/events.json.
import { writeFile, mkdir } from "node:fs/promises";

const HS    = (process.env.MATRIX_HOMESERVER || "").replace(/\/$/, "");
const TOKEN = process.env.MATRIX_ACCESS_TOKEN;
const ROOM  = process.env.MATRIX_EVENTS_ROOM_ID;
const OUT   = "_data/events.json";

const POLL_START = ["m.poll.start", "org.matrix.msc3381.poll.start"];
const POLL_RESP  = ["m.poll.response", "org.matrix.msc3381.poll.response"];
const ANSWERS    = ["yes", "maybe", "no"]; // poll answer order -> buckets

if (!HS || !TOKEN || !ROOM) {
  console.error("MATRIX_HOMESERVER, MATRIX_ACCESS_TOKEN, MATRIX_EVENTS_ROOM_ID required");
  process.exit(1);
}

async function api(path, init = {}) {
  const res = await fetch(`${HS}/_matrix/client/v3${path}`, {
    ...init,
    headers: { Authorization: `Bearer ${TOKEN}`, "Content-Type": "application/json" },
  });
  if (!res.ok) throw new Error(`${init.method || "GET"} ${path} -> ${res.status} ${await res.text()}`);
  return res.json();
}

const join = (id) =>
  api(`/join/${encodeURIComponent(id)}`, { method: "POST", body: "{}" }).catch(() => {});

// Collect poll-start events, newest first, following the pagination token.
async function fetchPolls(roomId) {
  const filter = encodeURIComponent(JSON.stringify({ types: POLL_START }));
  let from, polls = [];
  for (let page = 0; page < 10; page++) {
    const q = `/rooms/${encodeURIComponent(roomId)}/messages?dir=b&limit=100&filter=${filter}` +
              (from ? `&from=${encodeURIComponent(from)}` : "");
    const { chunk, end } = await api(q);
    polls.push(...chunk.filter(e => POLL_START.includes(e.type) && !e.unsigned?.redacted_because));
    // Spec: an empty chunk does NOT imply no more events; only `end` absence does.
    if (!end) break;
    from = end;
  }
  return polls;
}

// Best-effort: pull an ISO-ish date, title and location out of the question.
// Location = first `·` field after the title that is not the date field, so a
// question without a date field still yields its location.
function parseQuestion(q) {
  const m = q.match(/(\d{4}-\d{2}-\d{2})(?:[ T](\d{2}:\d{2}))?/);
  const parts = q.split("·").map(s => s.trim()).filter(Boolean);
  const dateIdx = m ? parts.findIndex((p, i) => i > 0 && p.includes(m[1])) : -1;
  const locIdx = parts.findIndex((p, i) => i > 0 && i !== dateIdx);
  return {
    start: m ? `${m[1]}T${m[2] || "00:00"}:00` : null,
    title: parts[0] || q.trim(),
    location: locIdx === -1 ? "" : parts[locIdx],
  };
}

async function tally(roomId, pollId, order) {
  const base = `/rooms/${encodeURIComponent(roomId)}/relations/${encodeURIComponent(pollId)}/m.reference`;
  // The server picks its own default page size; paginate on next_batch or RSVPs
  // silently undercount on polls larger than that default.
  const responses = [];
  let from;
  for (let page = 0; page < 10; page++) {
    const { chunk, next_batch } = await api(`${base}?limit=100${from ? `&from=${encodeURIComponent(from)}` : ""}`);
    responses.push(...chunk);
    if (!next_batch) break;
    from = next_batch;
  }
  const latest = new Map();
  for (const r of responses) {
    if (!POLL_RESP.includes(r.type)) continue;
    const sel = r.content["m.selections"] || r.content["org.matrix.msc3381.poll.response"]?.answers || [];
    const prev = latest.get(r.sender);
    if (!prev || r.origin_server_ts > prev.ts) latest.set(r.sender, { ts: r.origin_server_ts, sel });
  }
  const rsvp = { yes: 0, maybe: 0, no: 0 };
  for (const { sel } of latest.values()) {
    const i = order.indexOf(sel[0]);
    if (i >= 0 && i < ANSWERS.length) rsvp[ANSWERS[i]]++;
  }
  return rsvp;
}

// /rooms/{roomId}/... requires a room ID (spec); only /join accepts #alias.
const roomId = (await join(ROOM))?.room_id ?? ROOM;
const now = Date.now();
const events = [];
for (const poll of await fetchPolls(roomId)) {
  const block = poll.content["m.poll"] || poll.content["org.matrix.msc3381.poll.start"] || {};
  const question = block.question?.["m.text"]?.[0]?.body || block.question?.["org.matrix.msc1767.text"] || "";
  const { start, title, location } = parseQuestion(question);
  const order = (block.answers || []).map(a => a["m.id"] ?? a.id);
  events.push({
    id: poll.event_id, title, location, start,
    url: `https://matrix.to/#/${ROOM}`, // raw ROOM: matrix.to accepts aliases
    start_display: start ? start.replace("T", " ").slice(0, 16) : "",
    created: poll.origin_server_ts,
    upcoming: !start || new Date(start).getTime() >= now,
    rsvp: await tally(roomId, poll.event_id, order),
  });
}
// Dated events first (ascending), undated last; newest poll wins ties.
const rank = (e) => (e.start ? 0 : 1);
events.sort((a, b) => rank(a) - rank(b) || (a.start || "").localeCompare(b.start || "") || b.created - a.created);

await mkdir("_data", { recursive: true });
await writeFile(OUT, JSON.stringify(events, null, 2));
console.log(`Wrote ${events.length} events to ${OUT}`);
```

SCRIPT NOTES:
- Node 18+ built-in `fetch`. No dependencies.
- Env inputs: MATRIX_HOMESERVER, MATRIX_ACCESS_TOKEN, MATRIX_EVENTS_ROOM_ID.
- Room ID vs alias: `MATRIX_EVENTS_ROOM_ID` may be `!id:server` or
  `#alias:server`; the `/join` response's `room_id` is what gets used for the
  `/rooms/...` calls (fall back to ROOM if join fails, e.g. already a member
  with no write access).
- Scans up to 1000 poll-start events (10 pages x 100). The `types` filter means
  pages contain only poll-start events, so chat does not consume the cap.
  Raise the page cap if the room grows.
- `filter` = room event filter `{types:[poll-start-types]}`; excludes chat.
- Reads BOTH stable and unstable poll shapes:
  - start type: `m.poll.start` | `org.matrix.msc3381.poll.start`
  - start content block: `m.poll` | `org.matrix.msc3381.poll.start`
  - question text: `question.m.text[0].body` (stable) | `question["org.matrix.msc1767.text"]` (unstable)
  - answer id: `a["m.id"]` (stable) | `a.id` (unstable)
  - response selections: `m.selections` (stable) | `["org.matrix.msc3381.poll.response"].answers` (unstable)
- `upcoming` computed at build time from `start` (or true if start is null).

---

## FILE: _data/events.json

Committed placeholder content: `[]`.
CI overwrites it in the runner workspace. Not committed back.
Local `jekyll build` uses the placeholder unless script run locally.

Generated element schema (JSON array):
```json
[
  {
    "id": "$abc123",
    "title": "Rust Meetup Porto #12",
    "location": "Porto Innovation Hub",
    "start": "2026-11-12T18:30:00",
    "url": "https://matrix.to/#/!roomid:matrix.org",
    "start_display": "2026-11-12 18:30",
    "created": 1762000000000,
    "upcoming": true,
    "rsvp": { "yes": 12, "maybe": 5, "no": 3 }
  }
]
```

FIELD TYPES:
- id: string (Matrix event_id)
- title: string
- location: string (may be "")
- start: string ISO `YYYY-MM-DDTHH:MM:SS` or null
- url: string (matrix.to link to EVENTS_ROOM)
- start_display: string `YYYY-MM-DD HH:MM` or ""
- created: number (ms epoch, poll origin_server_ts)
- upcoming: boolean
- rsvp: object {yes:number, maybe:number, no:number}; always present, all zeros if the poll has no responses

---

## FILE: _includes/events.html

```liquid
{% comment %} Params: limit (n), upcoming_only (bool) {% endcomment %}
{% assign limit = include.limit | plus: 0 %}
{% assign events = site.data.events %}
{% if include.upcoming_only %}{% assign events = events | where: "upcoming", true %}{% endif %}
{% comment %} Order comes from the script (dated first, undated last, newest-first
ties). Do NOT sort here: Liquid's sort is not stable and would discard the
created tie-break, and it places nulls differently from the script. {% endcomment %}
<div class="space-y-6">
  {% for event in events %}
    {% if limit > 0 and forloop.index > limit %}{% break %}{% endif %}
    <article class="p-6 bg-white rounded-lg shadow-sm">
      <h3 class="text-xl font-bold text-gray-900">{{ event.title }}</h3>
      {% if event.start_display %}<p class="text-sm text-gray-500 mt-1">{{ event.start_display }}</p>{% endif %}
      {% if event.location %}<p class="text-gray-600 mt-1">{{ event.location }}</p>{% endif %}
      {% if event.rsvp %}
        <p class="mt-3 text-sm text-gray-700">{% t events.rsvp %}:
          {% t events.yes %} {{ event.rsvp.yes }} ·
          {% t events.maybe %} {{ event.rsvp.maybe }} ·
          {% t events.no %} {{ event.rsvp.no }}</p>
      {% endif %}
      <a href="{{ event.url }}" class="inline-block mt-3 text-blue-600 hover:underline font-medium">{% t events.join %}</a>
    </article>
  {% endfor %}
</div>
{% if events.size == 0 %}<p class="text-gray-600">{% t events.none %}</p>{% endif %}
```

PARAMS:
- `limit=N`: render at most N events. Default 0 = all.
- `upcoming_only=true`: filter `upcoming == true`.

---

## FILE: events.html (repository root)

```html
---
layout: default
title: titles.events
namespace: events
permalink: /events/
---
<div class="min-h-screen bg-linear-to-b from-white to-gray-50">
  <section class="py-20 bg-gray-50">
    <div class="max-w-3xl mx-auto px-6">
      <div class="text-center mb-12">
        <h1 class="text-4xl font-bold text-gray-900 mb-4">{% t events.heading %}</h1>
      </div>
      {% include events.html %}
    </div>
  </section>
</div>
```

- `namespace: events` REQUIRED so `{% tl events %}` resolves in header.
- `title: titles.events` consumed by `_layouts/default.html` `<title>`.

---

## EDIT: index.html

Insert a new block (place before the Frameworks block; styling matches existing
sections):

```html
<div class="mb-12">
  <h2 class="text-2xl font-bold text-gray-900 mb-4">{% t events.upcoming %}</h2>
  {% include events.html limit=3 upcoming_only=true %}
</div>
```

---

## EDIT: _includes/header.html

Add inside the existing desktop nav container (currently holds Home and Contact):

```html
<a href="{% tl events %}" class="text-gray-600 hover:text-blue-600 font-medium">{% t nav.events %}</a>
```

---

## EDIT: _i18n/en.yml

`titles` and `nav` keys already exist as maps; ADD one key to each. Add a NEW
top-level `events:` map.

Additions (indent the `events:` keys to match the existing `titles:` / `nav:`
maps — do NOT re-declare those parent keys, or YAML duplicate-key resolution
will silently drop the existing entries):
```yaml
# add inside the existing `titles:` map
  events: "Events"
# add inside the existing `nav:` map
  events: "Events"
# add as a NEW top-level map
events:
  heading: "Events"
  upcoming: "Upcoming events"
  none: "No upcoming events right now — check back soon!"
  rsvp: "RSVP"
  yes: "Yes"
  maybe: "Maybe"
  no: "No"
  join: "Join in Matrix"
```

## EDIT: _i18n/pt.yml

Additions (same structure as en.yml):
```yaml
# add inside the existing `titles:` map
  events: "Eventos"
# add inside the existing `nav:` map
  events: "Eventos"
# add as a NEW top-level map
events:
  heading: "Eventos"
  upcoming: "Próximos eventos"
  none: "Sem eventos agendados de momento — volta em breve!"
  rsvp: "Presenças"
  yes: "Sim"
  maybe: "Talvez"
  no: "Não"
  join: "Entrar no Matrix"
```

---

## EDIT: .github/workflows/jekyll.yml

CHANGE trigger block. Add `schedule`; keep existing `push` and `workflow_dispatch`.

```yaml
on:
  push:
    branches: ["main"]
  schedule:
    - cron: "0 6 * * *"
  workflow_dispatch:
```

ADD one step in the `build` job, BEFORE the `Build with Jekyll` step. Keep it
`continue-on-error`: a bad token, deleted room or Matrix outage must degrade to
the committed `_data/events.json`, never block the deploy (the script only
writes the file at the end, so a mid-run failure leaves the checked-out copy
untouched):

```yaml
      - name: Fetch Matrix events
        continue-on-error: true
        env:
          MATRIX_HOMESERVER: ${{ vars.MATRIX_HOMESERVER }}
          MATRIX_EVENTS_ROOM_ID: ${{ vars.MATRIX_EVENTS_ROOM_ID }}
          MATRIX_ACCESS_TOKEN: ${{ secrets.MATRIX_ACCESS_TOKEN }}
        run: node .github/scripts/fetch-matrix-events.mjs
```

---

## EDIT: _config.yml

Add `docs` to `exclude`. Without it, `docs/matrix-events.md` (no front matter)
is treated as a static file and published verbatim to `_site/docs/`:

```yaml
exclude: ["README.md", "input.css", "docs"]
```

---

## CONFIG: GITHUB ACTIONS

Location: repo Settings -> Secrets and variables -> Actions.

| kind | name | value |
|---|---|---|
| Variable | `MATRIX_HOMESERVER` | `https://matrix.org` |
| Variable | `MATRIX_EVENTS_ROOM_ID` | `!xxxxxx:matrix.org` or `#alias:server` (alias is resolved via `/join`) |
| Secret | `MATRIX_ACCESS_TOKEN` | bot access token |

BOT SETUP (one-time):
1. Create bot user, e.g. `@rustporto-bot:matrix.org`.
2. Join bot to EVENTS_ROOM.
3. Get token:
```bash
curl -sX POST \
  -d '{"type":"m.login.password","identifier":{"type":"m.id.user","user":"rustporto-bot"},"password":"***"}' \
  https://matrix.org/_matrix/client/v3/login
```
4. Copy `access_token` into `MATRIX_ACCESS_TOKEN`.

---

## MATRIX CS API REFERENCE (endpoints used)

Base: `${MATRIX_HOMESERVER}/_matrix/client/v3`
Auth: header `Authorization: Bearer <token>`.

| method | path | purpose |
|---|---|---|
| POST | `/join/{roomIdOrAlias}` | bot joins room (idempotent enough; ignore errors). Response `room_id` is the ID used by all later calls |
| GET | `/rooms/{roomId}/messages?dir=b&limit=100&filter={types:[...]}` | list poll-start events; `end` = next page token |
| GET | `/rooms/{roomId}/relations/{eventId}/m.reference?limit=100` | list poll responses for a poll; `next_batch` = next page token |

Room ID vs alias: only `/join` accepts `#alias:server`; `/rooms/{roomId}/...`
takes a **room ID** (spec type: "Room ID"). The script therefore resolves ROOM
via the `/join` response and uses that `room_id` for `/messages` and
`/relations`, falling back to ROOM itself if join fails. The raw ROOM string is
kept only for the matrix.to link (aliases render fine there).

Pagination terminators differ per endpoint: `/messages` stops when `end` is
absent (an empty chunk does NOT mean the end); `/relations` stops when
`next_batch` is absent.

---

## IMPLEMENTATION ORDER

1. Create `.github/scripts/fetch-matrix-events.mjs` (verbatim from this doc).
2. Create `_data/events.json` with `[]`.
3. Create `_includes/events.html`.
4. Create `events.html`.
5. Edit `index.html` (add section).
6. Edit `_includes/header.html` (nav link).
7. Edit `_i18n/en.yml` and `_i18n/pt.yml`.
8. Edit `_config.yml` (add `docs` to `exclude`).
9. Set Actions variables + secret (§CONFIG). Do this BEFORE step 10: the fetch
   step needs them on its first run (it is `continue-on-error`, so a miss only
   degrades to the committed placeholder rather than blocking the deploy).
10. Edit `.github/workflows/jekyll.yml` (trigger + step).
11. Verify (§VERIFICATION).

---

## LIMITATIONS / RISKS

1. Polls are NOT in the stable Matrix spec (MSC3381 + MSC1767). Clients currently
   send unstable types `org.matrix.msc3381.*`. Script tolerates both. Spec churn
   risk is real. Stable alternative for RSVP: emoji reactions (`m.reaction` /
   `m.annotation`) — swap `POLL_*` handling for relations query on
   `/relations/{eventId}/m.annotation/m.reaction`.
2. EVENTS_ROOM must be unencrypted. Encrypted => unreadable.
3. Event title/location not translated (single text for EN+PT).
4. Freshness = last deploy. GitHub disables scheduled workflows after ~60 days
   of repo inactivity; re-enable via push or `workflow_dispatch`.
5. Date display is raw `YYYY-MM-DD HH:MM`, no timezone conversion, no locale
   month names. Intentional. Improve later if needed.
6. All events share one `url` (matrix.to link to EVENTS_ROOM).
7. Scan cap = 1000 poll-start events; older polls not listed until cap raised.
8. A Matrix fetch failure (bad token, deleted room, outage) does NOT block the
   deploy: the fetch step is `continue-on-error`, so the site rebuilds from the
   last committed `_data/events.json` at the cost of silently stale events.
   Watch for the step's warning in the workflow run.

---

## VERIFICATION

Local script run:
```bash
MATRIX_HOMESERVER=https://matrix.org \
MATRIX_EVENTS_ROOM_ID='!xxxx:matrix.org' \
MATRIX_ACCESS_TOKEN='...' \
node .github/scripts/fetch-matrix-events.mjs
```
Then inspect `_data/events.json`.

Local site build:
```bash
bundle exec jekyll build
(cd _site && python3 -m http.server)
```
Check: `/events/`, `/pt/events/`, `/`, `/pt/`.
Check the plan doc is NOT published: `_site/docs/` must not exist
(`test ! -e _site/docs/matrix-events.md`).

CI: push to `main` or run workflow manually; confirm deployed site updates.

---

## CHECKLIST

NEW:
- [ ] `.github/scripts/fetch-matrix-events.mjs`
- [ ] `_data/events.json` (`[]`)
- [ ] `_includes/events.html`
- [ ] `events.html`
- [ ] `docs/matrix-events.md`

MODIFIED:
- [ ] `.github/workflows/jekyll.yml`
- [ ] `index.html`
- [ ] `_includes/header.html`
- [ ] `_i18n/en.yml`
- [ ] `_i18n/pt.yml`
- [ ] `_config.yml` (`docs` excluded)
