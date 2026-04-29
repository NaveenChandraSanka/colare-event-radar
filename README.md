# Colare Event Radar

Daily Claude Code Routine that scrapes Luma for SF Bay Area deeptech/hardtech events in the next 14 days, scores them against Colare's ICP, and posts a ranked top-5 digest to `#colare-events-radar`.

## How it works

```
7am PT daily → Routine clones this repo → Firecrawl scrapes luma.com/tech + luma.com/sf
            → Filter (geo + window) → Score (keywords + host allowlist)
            → Dedup vs Notion DB → Insert new events to Notion → Post top 5 to Slack
```

Routines have no persistent filesystem, so dedup state lives in the Notion DB `Colare Event Radar`. The routine clones this repo fresh every run — edit the YAML configs here to tune the radar without touching the routine prompt.

## Tuning surface

All filtering/scoring is config-driven. Edit, commit, push — next morning's run picks it up.

| File | What it does | Score weight |
|---|---|---|
| `config/icp_keywords.yaml` | Hardtech/deeptech vocab matched against title + description | +2 per hit |
| `config/talent_keywords.yaml` | Hiring/talent themes matched against title + description | +2 per hit |
| `config/icp_host_orgs.yaml` | ICP host allowlist matched against event host field | +5 if matched |
| `config/sf_locations.yaml` | Bay Area location whitelist (geo filter, not a score) | filter only |

`routine_prompt.md` is the prompt the cloud routine executes each day. Avoid editing unless you're changing the pipeline shape — config tuning should happen in YAML.

## Adding a new ICP host org

Edit `config/icp_host_orgs.yaml`, add the org name (case-insensitive substring match — "Anduril" catches "Anduril Industries"), commit, push. Effective on the next morning's run.

## Adding new keywords

Edit `config/icp_keywords.yaml` or `config/talent_keywords.yaml`. Same flow. Watch for false positives in the first few digests after a new keyword lands — overly generic terms ("AI", "tech") will pollute the radar.

## Notion DB schema

DB name: **Colare Event Radar**

| Field | Type | Notes |
|---|---|---|
| Name | Title | Event title |
| Slug | Text | Luma slug — dedup key (must be unique) |
| URL | URL | Full luma.com/{slug} |
| Event Date | Date | When the event happens |
| Location | Text | Venue + city |
| Host | Text | Hosting org/person |
| Score | Number | Sum of ICP signals matched |
| Signals | Multi-select | `hardtech`, `talent`, `host-allowlist` |
| Alerted Date | Date | When the routine first surfaced it |
| RSVP Count | Number | If Luma exposes it |

## Routine setup (one-time)

1. Push this repo to GitHub (private is fine).
2. Visit [claude.ai/code/routines](https://claude.ai/code/routines) → New routine.
3. Schedule: `0 7 * * *` (7am PT daily).
4. Repo: select this one.
5. Connectors: enable **Slack** and **Notion**. Firecrawl is invoked as a skill, no separate connector needed.
6. Prompt: paste contents of `routine_prompt.md`.
7. Click "Run now" once to verify end-to-end before enabling the daily schedule.

## Manual ad-hoc run

From any Claude Code session with this repo cloned:

> "Run the Colare Event Radar — read routine_prompt.md and execute it against today's date."

Useful when you want a Wednesday 3pm pull or want to test config changes before the next morning's cron fires.

## Tuning loop

Day 1–3: scan the digest each morning. For false positives ("not relevant"), check which signal triggered the score and tighten that keyword/host. For false negatives (events you saw separately and the radar missed), add the missing keyword or host.

Goal: by week 2, every event in the digest is one Naveen or Esther would actually consider attending.
