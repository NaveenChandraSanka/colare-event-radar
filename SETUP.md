# Setup — One-time steps to go live

You're 6 steps away from a daily 7am PT alert in `#colare-events-radar`.

> **Why Firecrawl?** Luma's listing and detail pages are JS-rendered — a plain fetch returns an empty shell, which historically caused the routine to hallucinate event titles, hosts, and locations. Firecrawl renders JS server-side and returns clean markdown, so the routine works from real page content. The upgraded `routine_prompt.md` hard-aborts if Firecrawl isn't connected rather than silently falling back to a non-rendering fetcher.

---

## 0. Wire up Firecrawl MCP (required — do this first)

1. Get a Firecrawl API key:
   - Sign in at [firecrawl.dev](https://firecrawl.dev) → **Dashboard** → **API Keys** → **Create**
   - The free tier (500 scrapes/mo) is enough for a daily 7am run with ~60 detail-page fetches
2. The repo already includes `.claude/settings.json` which tells the Routine to spin up the Firecrawl MCP server via `npx firecrawl-mcp`. It reads the API key from a `FIRECRAWL_API_KEY` env var.
3. In the Routine settings at [claude.ai/code/routines](https://claude.ai/code/routines), under **Environment variables**, add:
   - Name: `FIRECRAWL_API_KEY`
   - Value: your Firecrawl key
4. Confirm the MCP server appears in the connector list when the Routine starts (visible in the run log on first execution).

If the routine ever runs without Firecrawl connected, it now aborts immediately rather than posting a fabricated digest — so a missing/expired key surfaces as silence, not noise.

---

## 1. Rotate the Notion token (security)

The token shared in chat (`ntn_...`) is in this conversation transcript. After confirming the DB schema looks right at [notion.so/.../351426c71b2680e99b5ec973f7d2f392](https://www.notion.so/351426c71b2680e99b5ec973f7d2f392):

1. Go to [notion.so/profile/integrations](https://notion.so/profile/integrations)
2. Find the integration that issued this token → **Revoke**
3. Generate a new token if the Routine ends up needing direct API access (see step 4 — usually it won't)

Also: delete the `[TEST] Smoke-test row` from the Event Radar DB when you're ready (currently row id `351426c7-1b26-8171-86bd-e2e9ba71ceec`).

---

## 2. Grant the Claude Notion connector write access to the DB

The cloud-fired Routine uses the Anthropic-managed Notion connector — different from the `ntn_` token. It needs to be added to the DB explicitly:

1. Open **Event Radar NAVEEN** in Notion
2. Click `···` (top-right) → **Connections** → **Add connections**
3. Pick the Claude connector → Confirm

Once granted, the cloud Routine can read/write the DB during its 7am run.

---

## 3. Push the local repo to GitHub

From `/Users/naveensanka/Downloads/colare-event-radar`:

```bash
cd /Users/naveensanka/Downloads/colare-event-radar
git init
git add .
git commit -m "Initial: Luma Event Radar config + routine prompt"
gh repo create colare-event-radar --private --source=. --push
```

(Requires `gh auth login` first if not already authed.)

---

## 4. Register the Routine

Go to [claude.ai/code/routines](https://claude.ai/code/routines) → **New routine**:

| Field | Value |
|---|---|
| Name | Colare Event Radar |
| Schedule | `0 7 * * *` (7am daily, your local TZ) |
| Repository | `<your-gh-handle>/colare-event-radar` |
| Branch | `main` |
| Connectors | ✅ Slack (channel access to `#colare-events-radar`) · ✅ Notion (DB access from step 2) · ✅ Firecrawl MCP (auto-spawned from `.claude/settings.json`, needs `FIRECRAWL_API_KEY` env var from step 0) |
| Prompt | Paste contents of `routine_prompt.md` from the repo |

Click **Save**.

---

## 5. Run once manually before enabling the schedule

On the Routine's page, click **Run now**. Watch the run log:

- ✅ Should clone the repo, scrape `luma.com/sf`, filter, score
- ✅ Should query the DB and find only the smoke-test row (so all real events are "new" on first run)
- ✅ Should insert new events into Notion + post a digest to Slack

If the digest looks good, enable the daily schedule. If it's noisy, tune `config/icp_keywords.yaml` / `config/icp_host_orgs.yaml`, push, and the next run picks it up.

---

## Day 1–7 tuning checklist

- [ ] Delete the smoke-test row from the Event Radar DB
- [ ] Watch the first 3 mornings — note false positives (events that scored but aren't ICP) and false negatives (events you saw separately and the radar missed)
- [ ] For each false positive: identify which keyword/host triggered it. Tighten or remove that entry.
- [ ] For each false negative: add the missing keyword to `icp_keywords.yaml` or the missing host to `icp_host_orgs.yaml`.
- [ ] When SF Tech Week, Deep Tech Week, or any other major hardtech week is active in SF: uncomment the relevant URL in `config/luma_sources.yaml` so the radar pulls those calendars too.

Goal: by week 2, the digest has zero "what is this doing here?" entries.
