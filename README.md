# Theatre Reminder

Daily n8n automation that checks the Subotica National Theatre's repertoire page
(https://www.suteatar.org/lat/repertoar) for the play **"Sabirni centar"** in
August/September/October 2026, and pings a Telegram bot so tickets can be
bought before they sell out.

## How it works

The repertoire page is fully server-rendered — every month back to 2018 is
already in the HTML; the month tabs are just a client-side filter. So the
workflow does a plain HTTP GET and parses the returned HTML directly — no
headless browser needed.

Each performance is a list item like:

```html
<li class="work-item 2026-07 ticket-sold col-lg-6">
  ...
  <h4 class="menu-title font-alt" data-id="97" data-title="Sabirni centar">
    <a href="/lat/predstave/sabirni-centar">Dušan Kovačević: Sabirni centar (1)</a>
  </h4>
</li>
```

The year-month is in the `work-item` class, the play is matched exactly via
`data-title`, and a sold-out show carries the `ticket-sold` class + "Rasprodato"
label. The parsing logic (in the `Parse Repertoire` Code node) was verified
against a real snapshot of the page: it correctly found all 10 known
"Sabirni centar" listings (March–June 2026) with correct sold-out status, and
correctly found zero for August/September/October (currently empty, as expected).

## Workflow logic

1. **Daily 19:00 Trigger** — Schedule Trigger, cron `0 19 * * *`, timezone
   `Europe/Belgrade` (set in workflow Settings — adjust if this isn't your
   timezone).
2. **Config** — the *only* node you should need to edit later. Holds:
   - `targetMonths`: `["2026-08","2026-09","2026-10"]` — JSON array of `YYYY-MM` strings
     to check. Update this once these months fill up, or for a future season.
   - `playTitle`: `"Sabirni centar"`
   - `playUrl`: link included in Telegram messages
3. **Fetch Repertoire Page** — HTTP GET with a browser User-Agent header
   (the site may reject requests without one).
4. **Parse Repertoire** — Code node that regex-parses the HTML, filters to
   `targetMonths`, and checks for an exact `playTitle` match.
5. **Any listings in Aug-Oct?** — if all target months are still completely
   empty, the workflow stops silently (no Telegram message — nothing to
   report).
6. **Sabirni centar found?** — branches to one of two Telegram messages:
   - **Found**: lists every matching date/time and whether it's sold out.
     If *all* found dates are already sold out, the message says so instead
     of "book now", so you're not misled into rushing for nothing.
   - **Not found yet**: "repertoire updated, but still no Sabirni centar" —
     confirms the check ran and something changed, without you having to
     look yourself.

**No dedup/state tracking is used** — the workflow re-evaluates fresh every
run. This means both the "found" and "not found yet" messages repeat every
day the underlying condition still holds. That's intentional (so you keep
getting reminded until you've bought a ticket or the play is gone), but it
does mean that once other plays get added to Aug/Sep/Oct, you'll get a daily
"still not found" message until "Sabirni centar" itself shows up.

## Setup

### 1. Create the Telegram bot

1. In Telegram, message **@BotFather** → `/newbot` → follow the prompts →
   copy the bot token it gives you.
2. Send any message (e.g. `/start`) to your new bot from your own account —
   bots can't message you first.
3. In a browser, open:
   `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates`
   and find your `"chat":{"id": ...}` — that number is your chat ID.

### 2. Import the workflows into n8n

1. In n8n, **Import from File** → select `n8n/theatre-reminder.workflow.json`.
2. Also import `n8n/error-alert.workflow.json` (a separate small workflow that
   sends you a Telegram alert if the main workflow ever fails — e.g. if the
   site goes down or changes its HTML structure).
3. In both workflows, open each **Telegram** node and create/select a
   Telegram credential using your bot token. The `chatId` field is already
   set to `534842390`.
4. In the main workflow's **Settings**, set **Error Workflow** to
   "Theatre Reminder - Error Alert" so failures actually trigger the alert.
5. Confirm the **Daily 19:00 Trigger** node's timezone matches yours
   (defaults to `Europe/Belgrade`).
6. Activate both workflows.

### 3. Test it

- Temporarily set `Config.targetMonths` to `["2026-03"]` (known to contain
  five sold-out "Sabirni centar" showings) and manually execute the workflow
  to confirm the "found, sold out" Telegram message arrives correctly.
- Set it back to `["2026-08","2026-09","2026-10"]` and execute again — you should get
  the "not found yet" message (current real state as of writing).
- Temporarily break the URL in **Fetch Repertoire Page** to confirm the
  error-alert workflow fires.

## Files

- `n8n/theatre-reminder.workflow.json` — main daily-check workflow
- `n8n/error-alert.workflow.json` — failure notifier workflow
