---
name: microsoft-clarity-diagnose-friction
description: >-
  Find where users are struggling on a site using Microsoft Clarity's frustration
  signals — rage clicks, dead clicks, excessive scroll, quickback clicks, script
  errors and error clicks — sliced by device, browser, OS or URL. Use when asked
  "where are users getting stuck", "what's broken on mobile", or to triage a UX
  or checkout complaint.
api: Microsoft Clarity DataExport API
operations:
  - getProjectLiveInsights
generated: '2026-08-13'
method: generated
source: >-
  openapi/microsoft-clarity-dataexport-api-openapi.yml and
  https://learn.microsoft.com/en-us/clarity/setup-and-installation/clarity-data-export-api
---

# Diagnose friction with Clarity

Clarity's differentiator is that it counts *frustration*, not just traffic. Six
of the metrics `getProjectLiveInsights` returns are friction signals:

| Metric | What it means |
| --- | --- |
| `Rage Click Count` | Repeated rapid clicks on the same spot — the user expected something to happen |
| `Dead Click Count` | Clicks on something that isn't interactive |
| `Excessive Scroll` | Scrolling far past where the content ends or the answer should be |
| `Quickback Click` | User landed, bounced straight back — wrong page |
| `Script Error Count` | JavaScript threw |
| `Error Click Count` | A click that produced an error |

## Spend your one call well

The quota is **10 requests per project per day**, with no remaining-count
header. This diagnosis should cost you exactly one call, because every metric
comes back on every call — you only choose the slicing.

Pick the dimensions from the shape of the complaint:

- "Broken on mobile" -> `dimension1=Device`
- "Only some users" -> `dimension1=Browser&dimension2=OS`
- "Which pages?" -> `dimension1=URL`
- "Since the campaign launched" -> `dimension1=Source&dimension2=Medium`

```
GET https://www.clarity.ms/export-data/api/v1/project-live-insights?numOfDays=3&dimension1=Device&dimension2=Browser
Authorization: Bearer <YOUR_API_TOKEN>
```

Use `numOfDays=3` for diagnosis — the widest window the API offers — unless you
are checking whether a fix landed, in which case use `1`.

## Interpreting it

1. Pull the friction metric groups out of the response array by `metricName`.
2. Pull the `Traffic` group too, from the same payload. Raw friction counts are
   meaningless without a denominator: 400 rage clicks on Chrome is not worse
   than 40 on Safari if Chrome carries ten times the sessions.
3. Coerce the count fields — they arrive as JSON **strings**.
4. Normalise per session, then rank. The finding is a *rate*, not a count.
5. `Script Error Count` and `Error Click Count` concentrated on one
   browser/OS pair is an engineering bug. `Rage Click Count` and
   `Dead Click Count` concentrated on one URL is a design problem. Say which
   one you found.

## Then go look at it

The API returns aggregates only — it will tell you *where* friction is, never
*what* the user saw. To watch it happen you need session recordings, and those
are **not exposed by this API**. Two routes:

- The first-party MCP server's `list-session-recordings` tool, which can filter
  recordings by URL, device, browser, OS, country and city. Run it locally:
  `npx @microsoft/clarity-mcp-server --clarity_api_token=<token>`. Note it draws
  on the *same* 10-call daily budget.
- The Clarity dashboard, filtering to the segment you identified.

## Caveats to state in your answer

- **Bots inflate everything.** `Traffic` reports `totalBotSessionCount`
  alongside `totalSessionCount`. Subtract before you compute rates, and say
  whether you did.
- **Three days is the whole window.** You cannot compare against last month.
  Any "this is up/down" claim needs a baseline you stored yourself.
- **1,000-row cap, no pagination.** A `URL`-sliced call on a large site is
  almost certainly truncated to the top rows. Do not present a truncated list as
  exhaustive.
- **UTC.** Time-of-day reasoning must account for it.

## Errors

`400` bad parameters, `401` bad token, `403` token not authorized, `429`
`Exceeded daily limit`. There is no `Retry-After` and no error body — branch on
the status. Never retry a 4xx in a loop; each attempt costs one of ten daily
calls.
