# SwellSurf

Watches a surf forecast. When the spot is going to fire for several days straight, it
prices the trip, drops one proposal in the group chat, and waits for someone to actually
tap yes before it does anything else.

Two n8n workflows, no scrapers, no API keys for the forecast half.

```
  ┌─ workflow 1 ──────────────────────────────────────────────┐
  │  every 6h → Surfline wave + wind                          │
  │             → is it 6ft+ and offshore 3 days running?     │
  │             → have I already proposed this swell?         │
  │             → price flights (Amadeus)                     │
  │             → post proposal, SUSPEND until someone taps   │
  └───────────────────────────┬───────────────────────────────┘
                              │  approved
  ┌─ workflow 2 ──────────────▼───────────────────────────────┐
  │  build booking links → DRY_RUN? → email + confirm in chat │
  └───────────────────────────────────────────────────────────┘
```

## Why two workflows

The interesting problem in a build like this isn't the forecast maths, it's **where the
human goes**. If you chain detection straight into action in one workflow, the human step
has nowhere to live and quietly degrades into a node that returns `true`. Splitting it
means the gate is structural: workflow 2 has no schedule and no trigger of its own, so if
nobody approves, it simply never runs.

The approval itself is n8n's `sendAndWait`, which genuinely suspends the execution until
someone clicks. It can't be faked by editing a Code node.

## Import

1. **Workflows → Import from File** → `swellsurf-2-book.workflow.json` first. Save it and
   copy its ID out of the URL.
2. Import `swellsurf-1-detect.workflow.json`. Open **Hand off to booking** and paste that
   ID in place of `PASTE_EXECUTOR_WORKFLOW_ID`.
3. Fill in the **Config** node (workflow 1) and the **Safety** node (workflow 2).
4. Attach credentials — see below.
5. Run workflow 2 on its manual trigger first, with `DRY_RUN` on. Then workflow 1.

## Config

Everything tunable is in one Set node. No threshold is hidden in a Code node.

| Field | Meaning |
|---|---|
| `spotId` | Surfline spot — it's in the surfline.com URL. Default is Uluwatu. |
| `minSurfFt` / `minFiringDays` | 6ft or better, on 3 **consecutive** days |
| `tzOffsetHours` | Fallback only; the real offset comes from the API response |
| `originIata` / `destIata` | `PER` → `DPS` |
| `stayNear`, `adults`, `currency` | Used for the booking links and the fare search |
| `telegramChatId` | Your group. Ask `@get_id_bot`. |
| `amadeusHost` | `test.api.amadeus.com` — cached sandbox data, switch to `api.amadeus.com` when you're happy |

Workflow 2's **Safety** node holds `DRY_RUN` (on by default) and `notifyEmail`.

## Credentials

None of these live in the JSON — that's deliberate, so re-exporting or pushing the file
can never leak a token.

| Node | Credential | Notes |
|---|---|---|
| Ask the crew / Tell the group | **Telegram** | bot token via @BotFather |
| Amadeus token | **Custom Auth** (generic) | JSON: `{"body":{"client_id":"…","client_secret":"…"}}` |
| Email the booking links | **Gmail OAuth2** | |

Amadeus' self-service tier is free and gives you a real flight-offers API — no per-run
charge and nothing that breaks when a page layout changes.

## Things worth knowing

**Surfline is behind Cloudflare.** The `kbyg` endpoint is public and keyless, but it 403s
a default API-client User-Agent. Both HTTP nodes send a browser UA. This is very likely
why other builds of this idea proxy the call through `r.jina.ai` — you don't need to.

**Dedupe only works in production.** `Claim this swell` uses n8n static data, which is
only persisted on production runs (workflow active, fired by its real trigger). Manual
test executions always look like a new swell. That's n8n, not the code.

**Why dedupe matters at all.** A three-day swell satisfies the condition on roughly twelve
consecutive six-hourly runs. Without an event key you'd send twelve identical proposals
and pay for twelve fare searches for one trip.

**Consecutive days, not just three good days.** Monday, Wednesday and Friday is not a
swell you fly for. `Evaluate swell` looks for the longest unbroken run.

**Peak conditions come from firing intervals only.** Take the max over every interval and
you can end up announcing "offshore and clean" using the wind from the biggest *onshore*
hour of the week.

**A fare outage doesn't kill the proposal.** `Search flights` continues on error; the
message just goes out without prices.

## Extending it

Workflow 2's false branch is the only place in either workflow permitted to spend money or
contact anyone. Add side effects there and nowhere else, and keep them behind `DRY_RUN`
until you've watched the logged output a few times.

If you add anything genuinely irreversible, put another `sendAndWait` immediately before
it. Cheap operations gate expensive ones; reversible ones gate irreversible ones.

## Credit

The idea — surf forecast triggers the whole trip — is from
[aaronparton2-sketch/swell-event](https://github.com/aaronparton2-sketch/swell-event) (MIT).
This is an independent rebuild, not a fork: different forecast logic, different flight
source, real approval gate, and dedupe.
