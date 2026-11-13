# PropDoc daily activity report — recipe

Covers **yesterday, midnight-to-midnight America/New_York**: outbound/inbound calls and texts on the PropDoc Quo number(s), plus emails sent/received on `propdochomes@gmail.com`. Break outbound calls and texts down by teammate. Emails are a shared-inbox total only (see Gaps).

Read-only. Never send messages, place calls, or modify contacts, labels, or drafts.

Connectors: `user-Quo` and `user-Gmail--propdoc`. Discover schemas with GetDynamicTools before calling. User IDs and the inbox live in `team.json`; refresh that file if `list-users` or `list-inboxes` changes.

## 1. Date window

Compute yesterday’s bounds in `America/New_York` (EST is UTC−5, EDT is UTC−4):

```python
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo
tz = ZoneInfo("America/New_York")
today = datetime.now(tz).replace(hour=0, minute=0, second=0, microsecond=0)
start, end = today - timedelta(days=1), today
# start.isoformat() / end.isoformat() e.g. 2026-11-11T00:00:00-05:00
```

Use these ISO strings as `createdAt` GTE (`start`) and LT (`end`). Always also pass `timezone: "America/New_York"` to `get-analytics`.

Gmail search dates (account-local / historically Pacific — see sanity check below):

- sent/received on the report day: `after:YYYY/MM/DD before:YYYY/MM/DD` where `after` is the report date and `before` is the next day (e.g. `after:2026/11/11 before:2026/11/12`).

## 2. Roster and inboxes (2 tool calls)

1. `user-Quo` `list-users` `{ "maxResults": 50 }`. Paginate with `pageToken` if present. Map each `id` (US…) to `name`. `list-users` has timed out; if it does, use cached `team.json` plus the names on `list-inboxes`.
2. `user-Quo` `list-inboxes` `{}`. As of 2026-11-12 there is one inbox: `PNlh3M0zWH` / `+14132885606` (“Primary”), assigned to all four members.

Write any new users or numbers back to `team.json`. Do not invent people who are not in the roster (no Matt/Alice as of this date).

Current IDs:

| id | name | shortName | role |
|----|------|-----------|------|
| USLxOHKdvR | Antonio Cucciniello | Antonio | owner |
| USSVprv1rS | Kevin Shippee | Kevin | admin |
| USrINbBvIg | Gabriel R | Gabe | member |
| USMiqzMpNA | Ruby Anna Cepria | Ruby | member |

## 3. Quo counts — use `get-analytics` (about 5 calls)

Do **not** use `fetch-messages`, `fetch-call-transcripts`, or `fetch-missed-calls` for daily totals. Those fan out many API requests, must run one inbox at a time, and transcripts only cover recorded/transcribed calls. Analytics returns grouped counts directly. `get-analytics` has no pagination; `limit` defaults to 50, max 200 (enough for this 4-person, 1-inbox workspace). A `createdAt` GTE+LT filter is required.

Shared filter on every query:

```json
"filters": [
  { "field": "createdAt", "operator": "GTE", "values": ["<start>"] },
  { "field": "createdAt", "operator": "LT",  "values": ["<end>"] }
],
"timezone": "America/New_York",
"limit": 200
```

Run these (parallel is fine; serialize if you see timeouts):

1. **Workspace call totals by direction** — this is the official call total, do not sum per-user rows for it.
   - `entityType: "calls"`, `groupBy: [{ "field": "direction" }]`, `metrics: [{ "field": "calls", "aggregation": "COUNT_DISTINCT" }]`
   - Plus the shared date filter. A missing direction means 0 (e.g. only `outgoing | 11` ⇒ 0 inbound).

2. **Workspace text totals by direction**
   - Same as (1) but `entityType: "messages"`, metric field `messages`.

3. **Outbound calls per person**
   - `entityType: "calls"`, `groupBy: [{ "field": "user" }]`, `metrics: [{ "field": "calls", "aggregation": "COUNT_DISTINCT" }]`
   - Add filter `{ "field": "direction", "operator": "EQ", "values": ["outgoing"] }`.
   - Optional extra: `groupBy: [{ "field": "actorType" }, { "field": "user" }]` so you can separate teammate-placed calls (`actorType` = `user`) from an AI agent.
   - Optional: group by `outgoingOutcome` (no user) for connected / no-answer / busy / canceled / failed.

4. **Outbound texts per person**
   - `entityType: "messages"`, `groupBy: [{ "field": "user" }]`, metric `messages` COUNT_DISTINCT
   - Filter `direction` EQ `outgoing` plus the date window.
   - Incoming texts have no sender user ID; do not apply a `user` group to inbound.

5. **Inbound call outcomes (missed / voicemail / answered)**
   - `entityType: "calls"`, `groupBy: [{ "field": "incomingOutcome" }]`, metric `calls` COUNT_DISTINCT
   - Filter `direction` EQ `incoming` plus the date window.
   - Outcomes: `answered-by-user`, `answered-by-agent`, `agent-abandoned`, `voicemail`, `missed`, `abandoned`, `forwarded`.
   - If inbound > 0 and you want who answered: group by `user` and also filter `dialStatus` EQ `answered` and `incomingOutcome` EQ `answered-by-user`. That credits only the teammate who answered.

**Important:** shared-inbox calls can emit one analytics row per associated user, including people who never rang. Never sum per-user call counts to get a workspace total — always use query (1). After the per-user queries, compare `sum(per-user outbound calls)` to the workspace outbound total. If they match, the per-person table is a clean partition (this happened on 2026-11-11: 8+3=11). If they diverge, keep the workspace total from (1) and treat per-user numbers as “associated with,” not “placed by,” unless `actorType`+`user` still partitions cleanly.

Users missing from a per-user result have **0** for that metric. “Picked nothing up” = 0 outbound calls and 0 outbound texts (emails cannot be attributed).

## 4. Gmail counts — `user-Gmail--propdoc` (2–4 calls)

1. Optional confirm: `get_me` — expect `propdochomes@gmail.com`.
2. Optional, once in a while: `list_send_as`. As of 2026-11-12 there is only the primary address, with empty `displayName` and empty `signature`. No aliases.
3. Sent: `search_emails` `{ "query": "in:sent after:YYYY/MM/DD before:YYYY/MM/DD", "maxResults": 25 }`. Follow `nextPageToken`. Count messages (not threads).
4. Received: `search_emails` `{ "query": "-in:sent after:YYYY/MM/DD before:YYYY/MM/DD", "maxResults": 25 }` (or `in:inbox after:… before:…` if you only want the inbox). Follow `nextPageToken`.

If a dated search returns 0, sanity-check by listing newest mail without a date (`query: "in:sent"` and `query: "in:inbox"`, `maxResults: 25`) and converting each `date` (UTC, often a `Z` suffix) to America/New_York. Gmail’s `after:`/`before:` operators are date-only and can disagree with ET midnight at the edges. On 2026-11-11 both the dated search (including spam/trash) and the newest-first lists agreed: nothing that day. Last sent mail was 2026-11-06 11:52 AM ET; last human inbox reply was 2026-11-04.

**Attribution:** not possible with this mailbox. Sent mail is always `From: Prop Doc <propdochomes@gmail.com>`. There are no send-as aliases and no default signature. Sample bodies have no personal sign-off. Report a single shared-inbox sent/received total, not a per-person email column.

## 5. Assemble the report

- **Totals:** outbound calls, inbound calls (plus missed/voicemail/answered split), outbound texts, inbound texts, emails sent, emails received.
- **Per person:** outbound calls, outbound texts. Use Quo’s actual names (Gabriel R, Ruby Anna Cepria, Kevin Shippee, Antonio Cucciniello); short names in `team.json` are only for display.
- **Picked nothing up:** roster members with 0 outbound calls and 0 outbound texts.
- Convert any displayed timestamps to America/New_York and label the zone.

## Gaps and caveats

- Only Quo workspace numbers are visible. As of 2026-11-12 that is one line, +1 (413) 288-5606. Calls or texts on personal phones are not in this report.
- `outgoingOutcome: connected` means the far end was answered, not that a human talked or a deal progressed.
- `fetch-call-transcripts` is not a complete call log; do not use it for volume.
- Emails cannot be attributed to Gabe / Ruby / Kevin / Antonio.
- Google notification mail (security alerts, device setup) will show up as received if it landed yesterday; call that out if you include it in the received count.
- `get-analytics` rows with no activity are omitted; always join against the full roster.
- `list-users` may time out; `team.json` plus `list-inboxes` is the fallback.

## Worked example — Wed Nov 11, 2026 (ET)

Window: `2026-11-11T00:00:00-05:00` ≤ t < `2026-11-12T00:00:00-05:00`.

| | Outbound calls | Inbound calls | Outbound texts | Inbound texts | Emails sent | Emails received |
|--|--:|--:|--:|--:|--:|--:|
| **Workspace** | 11 (all connected) | 0 | 2 | 0 | 0 | 0 |

| Person | Outbound calls | Outbound texts |
|--------|--:|--:|
| Ruby Anna Cepria | 8 | 1 |
| Gabriel R | 3 | 1 |
| Kevin Shippee | 0 | 0 |
| Antonio Cucciniello | 0 | 0 |

Picked nothing up: Kevin Shippee, Antonio Cucciniello. All 11 outbound calls had `actorType=user` and `outgoingOutcome=connected`. Per-user outbound calls (8+3) and texts (1+1) matched the workspace totals.
