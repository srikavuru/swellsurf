# SwellSurf

**A surf forecast that books its own trip — up to the point where a human has to say yes.**

Every six hours SwellSurf pulls the wave and wind forecast for your break. When the numbers
line up — big enough, offshore, and holding for several days running — it works out the
travel window, prices the flights, and drops a single message in your group chat with a
button on it. Nothing else happens until somebody taps that button.

Two [n8n](https://n8n.io) workflows. No scrapers, no paid APIs, and no key at all for the
forecast half.

---

## The problem it actually solves

Good swells are visible three to five days out, and the people who score them are the ones
who commit early. The failure mode isn't ignorance — it's friction. By the time somebody
checks the forecast, screenshots it, posts it, argues about dates, and opens six tabs to
find out what flights cost, the cheap seats are gone and the group chat has moved on.

SwellSurf collapses that into one message. The forecast has already been read, the window
has already been chosen, the fares have already been looked up. What lands in the chat
isn't "hey, is anyone seeing this?" — it's a decision, pre-made, with a yes/no attached.

## What it looks like

This is real output from a live run against Uluwatu:

```
Uluwatu is about to fire.

Surf     up to 8-12ft (2x overhead)
Wind     10kt SE, offshore
Window   5 days straight, 2026-08-07 to 2026-08-12

2026-08-07   8-10ft   2.2m @ 16s SW   13kt ESE
2026-08-08   8-10ft   2.9m @ 16s SSW  14kt ESE
2026-08-09   8-10ft   1.8m @ 15s SW   15kt SE
2026-08-10   8-12ft   2.9m @ 17s SSW  10kt SE
2026-08-11   8-12ft   3.2m @ 16s SSW  11kt SE

Flights PER to DPS: from AUD 655 return, VIRGIN AUSTRALIA, 1 stop

Say yes and I will send the booking links round.

                  [ 🤙 I'm in ]    [ 😔 Can't make it ]
```

Tap **I'm in** and the second workflow wakes up, emails everyone the booking links with the
dates pre-filled, and confirms in the chat. Tap **Can't make it**, or ignore it for twelve
hours, and it goes quiet — and won't raise that same swell again.

## How it works

```
  ┌─ workflow 1 · detect and propose ─────────────────────────┐
  │  every 6h → Surfline wave + wind                          │
  │             → is it 6ft+ and offshore 3 days running?     │
  │             → have I already proposed this swell?         │
  │             → price flights (Amadeus)                     │
  │             → post proposal, SUSPEND until someone taps   │
  └───────────────────────────┬───────────────────────────────┘
                              │  approved
  ┌─ workflow 2 · book it ────▼───────────────────────────────┐
  │  build booking links → DRY_RUN? → email + confirm in chat │
  └───────────────────────────────────────────────────────────┘
```

### Why two workflows

The interesting problem in a build like this isn't the forecast maths — it's **where the
human goes**.

Chain detection straight into action in a single workflow and the human step has nowhere
to live. It becomes an `if` node, and then, once you're testing and impatient, a node that
just returns `true`. The gate quietly stops being a gate while still looking like one on
the canvas.

Splitting it makes the gate structural. Workflow 2 has no schedule and no trigger of its
own; its only entry point is workflow 1 handing it an approved trip. If nobody approves,
it doesn't run — not because a condition evaluated false, but because nothing invoked it.

The approval itself is n8n's `sendAndWait`, which genuinely suspends the execution and
parks it until someone clicks a URL button in Telegram. There is no code path that fakes
it.

### How it decides a swell is worth flying for

Three rules, all of which exist because the naive version gets them wrong:

**Consecutive days, not just good days.** A forecast with surf on Monday, Wednesday and
Friday is not a swell you book a flight for. `Evaluate swell` finds the longest *unbroken*
run of firing days and measures that against your threshold.

**Local days, not UTC days.** Surfline returns UTC timestamps. Bucket them without
shifting to the spot's timezone and a Friday-evening swell in Bali gets filed under
Thursday, quietly shifting your travel window by a day. The offset comes from the API
response itself, so changing the spot moves the timezone with it.

**Peak conditions from firing intervals only.** Take the maximum over every interval and
you can end up announcing "offshore and clean" using the wind reading from the biggest
*onshore* hour of the week. Only intervals that pass both tests are eligible to be the
headline.

Each swell also gets a stable event key — spot plus first firing day — recorded before
anything costs money. A three-day swell satisfies the trigger condition on roughly twelve
consecutive six-hourly runs; without that key you would send twelve identical proposals
and pay for twelve fare searches for one trip.

## What it deliberately doesn't do

It does not book anything. It doesn't hold a card, drive a checkout, or place a call. The
end state is booking links in your inbox with the dates filled in, and you take it from
there.

That's a deliberate limit rather than an unfinished edge. Everything upstream of the
approval button is reversible — reading a forecast, checking a fare, sending a message —
which is what makes it safe to leave running on a schedule for months. The moment you add
something that spends money without a human in the loop, you own a very different kind of
risk, and you should want a lot more error handling than a weekend project has.

If you do extend it, workflow 2's false branch is the only place in either workflow
permitted to spend money or contact anyone, and `DRY_RUN` sits in front of it.

## Requirements

| | |
|---|---|
| **n8n** | Cloud, or self-hosted **with a publicly reachable webhook URL** — see the note below |
| **Telegram bot** | free, via [@BotFather](https://t.me/BotFather) |
| **Gmail** | free, n8n's built-in OAuth |
| **Amadeus** | optional; free self-service tier for live fares |

Everything except n8n itself is free. The forecast needs no account at all.

> **The webhook URL matters.** `sendAndWait` puts URL buttons in the Telegram message that
> point back at your n8n instance. On localhost with no tunnel those buttons are dead when
> tapped from a phone. n8n Cloud handles this for you; self-hosted, set `WEBHOOK_URL` to
> your public address or run something like `ngrok http 5678` and point it there.

## Setup

1. **Workflows → Import from File** → `swellsurf-2-book.workflow.json` first. Save it and
   copy its ID out of the URL.
2. Import `swellsurf-1-detect.workflow.json`. Open **Hand off to booking** and paste that
   ID in place of `PASTE_EXECUTOR_WORKFLOW_ID`.
3. Fill in the **Config** node (workflow 1) and the **Safety** node (workflow 2).
4. Attach credentials — see below.
5. Run workflow 2 on its manual trigger with `DRY_RUN` still on. Read the output.
6. Run workflow 1 manually. If the spot is firing you'll get a real proposal with live
   buttons; tap one and check the execution resumes down the branch you expect.
7. Activate workflow 1 and let the schedule fire twice. The second run must log
   `alreadyProposed: true`.

Steps 1–6 work with no Amadeus account — the fare search fails soft and the proposal goes
out without prices. That's the fastest route to seeing a button land in your chat.

## Config

Everything tunable lives in one Set node per workflow. No threshold is buried in code.

| Field | Meaning |
|---|---|
| `spotId` | Surfline spot — it's in the surfline.com URL. Default is Uluwatu. |
| `minSurfFt` / `minFiringDays` | 6ft or better, on 3 **consecutive** days |
| `tzOffsetHours` | Fallback only; the real offset comes from the API response |
| `originIata` / `destIata` | `PER` → `DPS` |
| `stayNear`, `adults`, `currency` | Used for the booking links and the fare search |
| `telegramChatId` | Your group. Ask [@get_id_bot](https://t.me/get_id_bot) — group IDs are negative. |
| `amadeusHost` | `test.api.amadeus.com` — cached sandbox data; switch to `api.amadeus.com` when you're happy |

Workflow 2's **Safety** node holds `DRY_RUN` (on by default) and `notifyEmail`.

Thresholds are tuned for Uluwatu, which gets far more consistent swell than most breaks.
Somewhere fickle may need two days or a lower bar, or it will simply never fire and you'll
assume it's broken.

## Credentials

None of these live in the JSON. That's deliberate: re-exporting or pushing these files can
never leak a token.

| Node | Credential | Notes |
|---|---|---|
| Ask the crew / Tell the group | **Telegram** | bot token via @BotFather |
| Amadeus token | **Custom Auth** (generic) | JSON: `{"body":{"client_id":"…","client_secret":"…"}}` |
| Email the booking links | **Gmail OAuth2** | |

Amadeus' self-service tier gives you a real flight-offers API — no per-run charge, and
nothing that silently breaks when someone reshuffles a page layout.

## Things worth knowing

**Surfline sits behind Cloudflare.** The `kbyg` endpoint is genuinely public and keyless,
but it returns 403 to a default API-client User-Agent. Both HTTP nodes send a browser UA.
This is very likely why other builds of this idea proxy the call through `r.jina.ai` — you
don't need to, and you shouldn't, since it adds a rate limit and a dependency to your only
data source.

**Dedupe only works in production.** `Claim this swell` uses n8n static data, which is only
persisted on production runs — workflow active, fired by its real trigger. Manual test
executions always look like a fresh swell. That's n8n, not the code.

**A fare outage doesn't kill the proposal.** `Search flights` is set to continue on error,
so the message still goes out, just without prices. Losing a proposal to a flaky third
party would be a worse failure than showing no fares.

**The event key claims before it spends.** If the fare search dies after the claim, you
lose that one proposal rather than risking a double charge and a spammed group chat. That
trade is deliberate; flip it if your priorities differ.

## Credit

The idea — surf forecast triggers the whole trip — comes from
[aaronparton2-sketch/swell-event](https://github.com/aaronparton2-sketch/swell-event) (MIT),
which is worth a look for the original take.

This is an independent rebuild rather than a fork: different forecast logic, a real
approval gate, per-swell dedupe, an API instead of scrapers, and no secrets in the files.
