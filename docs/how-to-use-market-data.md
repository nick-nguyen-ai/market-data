# How to use market data

A self-contained brief for an internal coding agent. The job is to build an internal GitHub Pages site that displays the public daily **markets** and **AI** JSON feeds, and a CI workflow that copies new files as soon as they appear.

This document is based only on the public files in [nick-nguyen-ai/market-data](https://github.com/nick-nguyen-ai/market-data) (README, `index.json`, every `days/<date>/*.json` file from 16 October 2026 through 9 November 2026, and the commit history). Do not invent fields. Ignore unknown fields if they appear later. The feed is general market and AI news only: it does not include a personal share list.

**Disclaimer to show on every page:** "Information only, not financial advice. Prices are delayed and unofficial."

---

## 1. Purpose and overview

The public repository publishes two JSON documents each calendar day (Sydney date):

- **markets.json** — a morning market briefing (every day, including Saturday and Sunday), plus an afternoon close wrap on most weekdays.
- **ai.json** — a daily AI news report (almost every day; see the known gap below).

The internal site should:

1. Mirror those JSON files into this internal repository (recommended: a `data/` folder).
2. Render a home page, a Markets page and an AI page that follow the layout in [§7 UI reference](#7-ui-reference).
3. Rebuild and deploy GitHub Pages whenever the mirrored data changes.

Do **not** fetch the JSON in the visitor's browser as the only path. Corporate networks often block `raw.githubusercontent.com` or `api.github.com`. Fetch in CI, commit the files, and serve them from the internal Pages site.

---

## 2. Repository layout and URLs

Public repo: `https://github.com/nick-nguyen-ai/market-data`
Default branch: `main`

| Path | What it is |
| --- | --- |
| `latest/markets.json` | The newest markets file. Always a copy of `days/<latest-markets-date>/markets.json`. |
| `latest/ai.json` | The newest AI file. Always a copy of `days/<latest-ai-date>/ai.json`. |
| `days/YYYY-MM-DD/markets.json` | Markets briefing for that Sydney date. |
| `days/YYYY-MM-DD/ai.json` | AI report for that Sydney date (when present). |
| `index.json` | Catalogue of available dates and which files each date has, plus pointers to the newest date for each feed. |
| `docs/how-to-use-market-data.md` | This guide (field dictionary, schedule, sample CI, UI reference). |
| `docs/screenshots/` | Captures that accompany the UI reference. |
| `README.md` | Short description. States that raw URLs send `Access-Control-Allow-Origin: *`. |

Raw file URLs (these are the ones CI should fetch):

```
https://raw.githubusercontent.com/nick-nguyen-ai/market-data/main/latest/markets.json
https://raw.githubusercontent.com/nick-nguyen-ai/market-data/main/latest/ai.json
https://raw.githubusercontent.com/nick-nguyen-ai/market-data/main/index.json
https://raw.githubusercontent.com/nick-nguyen-ai/market-data/main/days/2026-11-09/markets.json
https://raw.githubusercontent.com/nick-nguyen-ai/market-data/main/days/2026-11-09/ai.json
```

Observed HTTP headers on the raw URLs (checked 9 November 2026):

- `Access-Control-Allow-Origin: *` (browser fetches are allowed when the network can reach the host).
- `Content-Type: text/plain; charset=utf-8` — not `application/json`. Parse the body as JSON anyway.
- `ETag` is present. You can send `If-None-Match` on later requests.
- `Cache-Control: max-age=300` (five minutes at the CDN).

The GitHub API (`GET https://api.github.com/repos/nick-nguyen-ai/market-data/commits/main`) also sends `Access-Control-Allow-Origin: *` and `Cache-Control: public, max-age=60`. Unauthenticated calls are limited to **60 requests per hour** per IP. A GitHub token (even with no special scopes) raises that limit substantially; use one on CI.

`index.json` shape:

```json
{
  "schema_version": 1,
  "latest": { "markets": "2026-11-09", "ai": "2026-11-09" },
  "days": [
    { "date": "2026-11-09", "files": ["ai.json", "markets.json"] }
  ]
}
```

`days` is newest-first. `latest.markets` and `latest.ai` are independent date strings; they are usually the same date but must be read separately. `files` is almost always `["ai.json", "markets.json"]`. One observed exception: **2026-10-29 has `markets.json` only** (`ai.json` is absent). Always look at `files` before fetching `ai.json` for a historical date.

The catalogue currently covers 16 October 2026 through 9 November 2026 (25 market days, 24 AI days).

---

## 3. Update schedule and how to detect new data

### 3.1 When files usually change

Times below are converted from commit timestamps on `main` (stored in UTC) into **Australia/Sydney**. All of the observed history falls in daylight saving, so the local offset is **AEDT, UTC+11**.

Typical pattern, from 26 October 2026 through 9 November 2026:

| Update | Days | Observed Sydney time | Typical UTC |
| --- | --- | --- | --- |
| Morning (markets + AI) | Every day, including Saturday and Sunday | **08:36–08:47 AEDT** | 21:36–21:47 UTC the previous calendar day |
| Close (markets; AI usually unchanged) | Weekdays only | **16:27–16:38 AEDT** | 05:27–05:38 UTC the same calendar day |

Exceptions worth coding for:

- Friday 30 October 2026 close landed at **18:44 AEDT** (07:44 UTC) — more than two hours later than usual. A poll window that only covers 16:15–17:15 Sydney time would have missed it.
- The first commits, on Sunday 25 October 2026 at 16:56–16:58 AEDT, were the initial load, not a normal close run.
- After the 9 November 2026 close (16:32 AEDT), three further commits landed at 21:46–21:49 AEDT (`Remove per-share data` and two `Update data 2026-11-09` rewrites). That was a one-off content cleanup, not a third daily edition. The same date's files were overwritten again.
- **One date's files can be updated twice** (morning, then close). Occasional extra rewrites also happen. Do not assume “already fetched today's date” means you are done.

Commit messages for ordinary updates are:

```
Update data YYYY-MM-DD
```

The date in the message is the Sydney briefing date, which may be the next calendar date relative to the UTC timestamp (morning updates are sent around 21:40 UTC, which is already the next morning in Sydney).

Weekend days have a morning update only. `markets.close` is `null` on those days.

### 3.2 How to detect a change cheaply

**Do not rely on `index.json` dates alone.** A close update overwrites `days/<date>/markets.json` and `latest/markets.json` but usually leaves `index.json` byte-identical (the date and file list are already correct). Comparing only `index.latest.markets === "2026-11-09"` will miss the close wrap.

Recommended, in order:

1. **Latest commit SHA (best).**
   `GET https://api.github.com/repos/nick-nguyen-ai/market-data/commits/main`
   Read `sha`. Store it (for example in `data/.upstream-sha`). If it is unchanged, exit the workflow immediately. One request, no file download.

2. **ETag on `latest/markets.json` and `latest/ai.json`.**
   `HEAD` or `GET` with `If-None-Match`. A close update changes the markets ETag even when `index.json` does not.

3. **Content hash of `latest/markets.json` and `latest/ai.json`.**
   Useful as a belt-and-braces check after a download. Do not compare only `generated_at_utc` across historical days: after a bulk rewrite, every markets file in the archive currently shares the same `generated_at_utc` (`2026-11-09T10:49:06Z`). That field is “when this file's current bytes were produced”, not “when the briefing was first published”.

You cannot subscribe to push events from a repository you do not own, so **poll on a schedule**. Cross-repo `repository_dispatch` / `workflow_run` triggers are not available here.

### 3.3 Suggested poll cadence

Poll **every 15 minutes, 24 hours a day**. Most runs will see an unchanged SHA and finish in a few seconds. That cadence covers the usual morning and close windows, the 18:44 outlier, one-off rewrites, and the twice-yearly change in Sydney offset.

GitHub Actions `cron` is **UTC**. If you also want denser polling around the usual windows, use both of the following conversions because Sydney daylight saving ends in early April and starts in early October:

| Sydney window | AEDT (UTC+11, roughly October–April) | AEST (UTC+10, roughly April–October) |
| --- | --- | --- |
| 08:30–09:30 | `21:30–22:30 UTC` previous day | `22:30–23:30 UTC` previous day |
| 16:15–17:15 | `05:15–06:15 UTC` same day | `06:15–07:15 UTC` same day |

In 2026, daylight saving ends on 5 April and starts on 4 October. Do not hard-code a single UTC window for the whole year. The every-15-minutes poll is the robust default; the table is only if you need to add extra runs.

---

## 4. Sample GitHub Actions workflow

The workflow below polls the public repo, downloads JSON only when the SHA changed, validates it, and commits into `data/` on the internal repo. It then uploads a Pages artifact. Adjust the deploy job to match your GitHub Enterprise Pages setup (classic `peaceiris/actions-gh-pages`, or `actions/deploy-pages` with `environment: github-pages`).

Put this at `.github/workflows/refresh-market-data.yml`.

```yaml
name: Refresh market data

on:
  schedule:
    # Every 15 minutes, UTC. Most runs no-op after the SHA check.
    - cron: "*/15 * * * *"
  workflow_dispatch:

permissions:
  contents: write
  pages: write
  id-token: write

concurrency:
  group: refresh-market-data
  cancel-in-progress: false

env:
  UPSTREAM: nick-nguyen-ai/market-data
  RAW: https://raw.githubusercontent.com/nick-nguyen-ai/market-data/main

jobs:
  refresh:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    outputs:
      changed: ${{ steps.check.outputs.changed }}
    steps:
      - uses: actions/checkout@v4

      - name: Check latest upstream commit
        id: check
        env:
          # A token raises API rate limits (unauthenticated is 60 requests/hour/IP).
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          set -euo pipefail
          sha=$(curl -fsSL \
            -H "Accept: application/vnd.github+json" \
            -H "X-GitHub-Api-Version: 2022-11-28" \
            -H "Authorization: Bearer ${GH_TOKEN}" \
            "https://api.github.com/repos/${UPSTREAM}/commits/main" \
            | python3 -c "import json,sys; print(json.load(sys.stdin)['sha'])")
          echo "sha=${sha}" >> "$GITHUB_OUTPUT"
          mkdir -p data
          if [ -f data/.upstream-sha ] && [ "$(cat data/.upstream-sha)" = "$sha" ]; then
            echo "No upstream change."
            echo "changed=false" >> "$GITHUB_OUTPUT"
          else
            echo "New upstream commit ${sha}"
            echo "changed=true" >> "$GITHUB_OUTPUT"
          fi

      - name: Fetch JSON
        if: steps.check.outputs.changed == 'true'
        run: |
          set -euo pipefail
          mkdir -p data/latest
          curl -fsSL "$RAW/index.json" -o data/index.json
          curl -fsSL "$RAW/latest/markets.json" -o data/latest/markets.json
          curl -fsSL "$RAW/latest/ai.json" -o data/latest/ai.json

          python3 - <<'PY'
          import json, pathlib, urllib.request, os
          raw = os.environ["RAW"]
          root = pathlib.Path("data")
          index = json.loads((root / "index.json").read_text())
          # Keep a local archive of every date the catalogue lists.
          for day in index["days"]:
              date = day["date"]
              dest = root / "days" / date
              dest.mkdir(parents=True, exist_ok=True)
              for name in day["files"]:
                  target = dest / name
                  if target.exists():
                      continue  # already mirrored; latest/ is the one that may have been overwritten
                  url = f"{raw}/days/{date}/{name}"
                  print("fetch", url)
                  with urllib.request.urlopen(url) as r:
                      target.write_bytes(r.read())
          # Always refresh the current latest dates, even if the dated file already existed
          # (a close update overwrites the same path).
          for kind, date in index["latest"].items():
              if not date:
                  continue
              src = root / "latest" / f"{kind}.json"
              dest = root / "days" / date / f"{kind}.json"
              dest.parent.mkdir(parents=True, exist_ok=True)
              dest.write_bytes(src.read_bytes())
          PY
          printf '%s\n' "${{ steps.check.outputs.sha }}" > data/.upstream-sha

      - name: Validate JSON
        if: steps.check.outputs.changed == 'true'
        run: python3 scripts/validate_feed.py data

      - name: Commit mirrored data
        if: steps.check.outputs.changed == 'true'
        run: |
          set -euo pipefail
          git config user.name "market-data-bot"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add data
          if git diff --cached --quiet; then
            echo "Fetched upstream but local data/ is already current."
            exit 0
          fi
          date=$(python3 -c "import json; print(json.load(open('data/index.json'))['latest']['markets'] or '')")
          git commit -m "Refresh market data ${date}"
          git pull --rebase origin HEAD || true
          git push

      - name: Build internal site
        if: steps.check.outputs.changed == 'true'
        run: |
          # Replace with your static site build (Jekyll, Next export, plain HTML, etc.).
          # The build must read JSON from data/latest/ and data/days/, not from the public internet.
          mkdir -p _site
          cp -R data _site/data
          # ...plus your rendered HTML/CSS/JS...

      - name: Upload Pages artifact
        if: steps.check.outputs.changed == 'true'
        uses: actions/upload-pages-artifact@v3
        with:
          path: _site

  deploy:
    needs: refresh
    if: needs.refresh.outputs.changed == 'true'
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.dep.outputs.page_url }}
    steps:
      - id: dep
        uses: actions/deploy-pages@v4
```

A minimal `scripts/validate_feed.py`:

```python
#!/usr/bin/env python3
import json, sys
from pathlib import Path

root = Path(sys.argv[1])
errors = []

def load(path):
    try:
        return json.loads(path.read_text(encoding="utf-8"))
    except Exception as e:
        errors.append(f"{path}: not valid JSON ({e})")
        return None

index = load(root / "index.json")
if index:
    for key in ("schema_version", "latest", "days"):
        if key not in index:
            errors.append(f"index.json missing {key}")

for kind, required in {
    "markets": ["schema_version", "type", "date", "generated_at_utc", "title",
                "edition", "morning", "close", "disclaimer"],
    "ai": ["schema_version", "type", "date", "generated_at_utc", "title",
           "top_stories", "sections"],
}.items():
    path = root / "latest" / f"{kind}.json"
    data = load(path)
    if not data:
        continue
    if data.get("type") != kind:
        errors.append(f"{path}: type {data.get('type')!r} != {kind!r}")
    for key in required:
        if key not in data:
            errors.append(f"{path}: missing {key}")
    if kind == "markets" and data.get("morning") is None:
        errors.append(f"{path}: morning must be an object")
    if kind == "ai":
        if not isinstance(data.get("top_stories"), list):
            errors.append(f"{path}: top_stories must be an array")
        if not isinstance(data.get("sections"), list):
            errors.append(f"{path}: sections must be an array")

if errors:
    print("Validation failed:")
    print("\n".join(errors))
    sys.exit(1)
print("OK")
```

### 4.1 GitHub Enterprise notes

- **Allow-list / proxy.** Runners may need outbound access to `github.com`, `api.github.com` and `raw.githubusercontent.com`. If those hosts are blocked, have a network team allow them for this workflow, or fetch through an approved proxy (`HTTPS_PROXY`). Do not work around a block by scraping a logged-in browser session.
- **Prefer CI fetch over client-side fetch.** Even though the raw host currently sends `Access-Control-Allow-Origin: *`, a staff browser on the corporate network may still be unable to reach it. The Pages site should read JSON from same-origin paths such as `/data/latest/markets.json`.
- **Rate limits.** Unauthenticated `api.github.com` is 60 requests/hour/IP. Shared enterprise runner IPs can exhaust that. Always send `Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}` (or a PAT with public-repo read) on API calls. Raw file downloads do not use that API budget.
- **Do not fork-sync as the only mechanism** unless your Enterprise can fork a public GitHub.com repo. A scheduled poll is the portable approach.
- **Pin what you serve.** Once a day's file is mirrored, keep it. A later rewrite of the same date should overwrite `data/days/<date>/` and `data/latest/`, then rebuild Pages.
- **Secrets.** No API key is required to read this public repo. Do not embed tokens in the Pages JavaScript.

---

## 5. Edition model (weekday vs weekend, morning vs close)

### 5.1 `markets.json`

Top-level `edition` is `"weekday"` or `"weekend"`.

Both editions always have a `morning` object. `close` is either a full object (same shape as `morning`) or `null`.

| Situation | `edition` | `morning` | `close` |
| --- | --- | --- | --- |
| Weekday, after the close update | `"weekday"` | present (pre-open briefing) | object (afternoon wrap) |
| Weekday, after the morning update only | `"weekday"` | present | `null` until the close file arrives |
| Saturday or Sunday | `"weekend"` | present | `null` |
| 16 October 2026 (weekday, historical) | `"weekday"` | present | `null` (no close was ever published for that date) |

The UI should treat `close === null` as a morning-only page (no Morning/Close tabs). After `close` becomes an object, show the tabs. On a weekday *after 16:30 Sydney time*, default the selected tab to Close if you are showing “today”; otherwise default to Morning. Older weekdays with a close object should still default to Close when opened.

`morning.cutoff_local` has been `YYYY-MM-DDT07:30:00+11:00` on every file in the archive (07:30 Sydney, AEDT in this period). `close.cutoff_local` has been `YYYY-MM-DDT16:30:00+11:00`. When the offset is `+10:00`, label the clock time **AEST**; when it is `+11:00`, label it **AEDT**. Do not hard-code the abbreviation.

`session_note` is a free-text label, not an enum. Observed values:

- Weekday morning: `Weekday ASX edition · before the 10:00 open`
- Weekday close: `Weekday ASX edition · after the close`
- Weekend mornings vary, for example `Weekend edition · week in review`, `Weekend edition · what's new since Saturday, plus the week ahead`, `Weekend edition · the week in review, with the week ahead`, and a longer Sunday note that prices are Friday's closes.

Drive tab structure from `edition` + whether `close` is an object. Display `session_note` (or the derived “Data as of HH:MM TZ” line) as secondary text.

`title` is a human heading. Current style is `Markets: Monday 9 November`. Earlier archive titles used `ASX morning briefing: Friday 16 October` and `Weekend global briefing: Saturday 17 October`. Render the string as given.

### 5.2 `ai.json`

There is no morning/close split and no `edition` field. One file per date. It is normally published with the morning markets update and left unchanged at the close. A close commit may rewrite `ai.json` only if the AI content itself changed (the publisher preserves `generated_at_utc` when the rest of the file is unchanged).

### 5.3 What is in each markets edition

Both `morning` and `close` have the same keys: `cutoff_local`, `session_note`, `summary`, `prices`, `sections`, `rates`, `calendar`.

Differences observed across the archive:

| Topic | Morning | Close |
| --- | --- | --- |
| `sections` | Always 4, in this order: `market-drivers`, `australia-asx`, `rates-expectations`, `global-risk` | Always 6, in this order: `asx-close`, `asia-close`, `rates-fx`, `commodities`, `tonight`, `still-open` |
| A section may have zero cards | Yes (once: 18 October, `australia-asx` was empty). Show a quiet “No new verified items in this edition.” | Cards are present on every close section so far (1–2 cards each) |
| `prices.quotes` | Always 16 instruments (no Asian cash indices, no copper) | Always 21 instruments (the morning 16 plus `nikkei`, `hang_seng`, `shanghai`, `kospi`, `copper_fut`) |
| Week-over-week changes on quotes | Present on **weekend** mornings only (`display_week_change` and related fields, typically 11–12 of the 16 quotes) | Never observed |
| `rates.fed` | Always present as a key; status may be `"unavailable"` | Often **omitted** (present on only 3 of 16 close editions) |
| `rates.rba` | Always present; status may be `"unavailable"` | Always present and `"available"` in the archive |
| `calendar` | 3–20 rows (weekends and Mondays tend to be a week-ahead list) | 3–9 rows, usually the evening/next session |
| `summary` | Always exactly 3 strings | Always exactly 3 strings |

---

## 6. Data dictionary

Conventions used in the tables:

- **Always** means the key was present in every object of that kind in the archive.
- **Optional** means the key was sometimes absent. Treat absence like “do not render”.
- **Nullable** means the key is present but the value may be `null` (notably `markets.close`, and some quote fields when `status` is `"unavailable"`).
- Types are JSON types. Strings are UTF-8. Some numeric fields are integers in one file and floats in another; parse as numbers.
- Percentage changes stored as numbers are in **percentage points**, not decimals: `0.44` means 0.44%, not 44%. Prefer the pre-formatted `display_*` strings in the UI.
- Basis points are stored in `change_bp` / `week_change_bp` (so `-9` means −9 bp).

Unknown keys may be added in a later `schema_version` or even under `schema_version: 1`. **Ignore keys you do not recognise.** Do not fail the build.

### 6.1 `index.json`

| Field | Type | Presence | Meaning | Example |
| --- | --- | --- | --- | --- |
| `schema_version` | number (int) | always | Catalogue schema. Currently `1`. | `1` |
| `latest` | object | always | Newest date for each feed. |  |
| `latest.markets` | string (`YYYY-MM-DD`) | always | Date of `latest/markets.json`. | `"2026-11-09"` |
| `latest.ai` | string (`YYYY-MM-DD`) | always | Date of `latest/ai.json`. | `"2026-11-09"` |
| `days` | array of objects | always | Catalogue, newest first. |  |
| `days[].date` | string (`YYYY-MM-DD`) | always | Sydney date of that folder. | `"2026-11-08"` |
| `days[].files` | array of strings | always | File names present. Values observed: `"markets.json"`, `"ai.json"`. Length 1 or 2. | `["ai.json", "markets.json"]` |

### 6.2 `markets.json` — top level

| Field | Type | Presence | Meaning | Example |
| --- | --- | --- | --- | --- |
| `schema_version` | number (int) | always | File schema. Currently `1`. | `1` |
| `type` | string | always | Always `"markets"`. | `"markets"` |
| `date` | string (`YYYY-MM-DD`) | always | Sydney date of the briefing. | `"2026-11-09"` |
| `generated_at_utc` | string | always | UTC timestamp of this file version, `YYYY-MM-DDTHH:MM:SSZ`. See the caveat in §3.2. | `"2026-11-09T10:49:06Z"` |
| `title` | string | always | Page heading. | `"Markets: Monday 9 November"` |
| `edition` | string | always | `"weekday"` or `"weekend"`. | `"weekday"` |
| `morning` | object | always | Morning (or weekend) briefing. See §6.3. |  |
| `close` | object or `null` | always (key is always present) | Afternoon wrap, or `null`. | `null` on weekends |
| `disclaimer` | string | always | Legal line. Identical on every file: `"Information only, not financial advice. Prices are delayed and unofficial."` |  |

### 6.3 Edition object (`morning` / `close`)

| Field | Type | Presence | Meaning | Example |
| --- | --- | --- | --- | --- |
| `cutoff_local` | string (ISO 8601 with offset) | always | Data cutoff in Sydney local time. | `"2026-11-09T07:30:00+11:00"` |
| `session_note` | string | always | One-line description of the session. Free text. | `"Weekday ASX edition · before the 10:00 open"` |
| `summary` | array of 3 strings | always | The “If you read nothing else” bullets. May contain Markdown links `[label](https://...)`. | See latest file |
| `prices` | object | always | Snapshot metadata + quotes. §6.4. |  |
| `sections` | array of section objects | always | Story groups. §6.5. |  |
| `rates` | object | always | Policy-rate pricing and forecasts. §6.7. May be `{}` in principle; in the archive it always has at least `rba`. |  |
| `calendar` | array of event objects | always | Forward diary. May be empty in principle; in the archive it always has at least 3 rows. §6.8. |  |

A section may include optional `note` (string) — a caveat shown under the section title (used most often on close `asia-close`, when Asian cash markets are still open at the 16:30 Sydney cutoff).

### 6.4 `prices`

| Field | Type | Presence | Meaning | Example |
| --- | --- | --- | --- | --- |
| `retrieved_at_utc` | string | always | When the snapshot was pulled. **Format is not strict ISO-8601**: it uses a space, not `T`, and often has no seconds, with a trailing `Z`. | `"2026-11-09 05:32Z"` |
| `cutoff_utc` | string (ISO 8601, `Z`) | always | UTC equivalent of the edition cutoff. Morning is typically 20:30 UTC the previous evening (07:30 AEDT). Close is typically 05:30 UTC (16:30 AEDT). | `"2026-11-08T20:30:00Z"` |
| `quotes` | array of quote objects | always | 16 (morning) or 21 (close) instruments, in a stable order. §6.4.1. |  |

#### 6.4.1 Quote object

Instruments observed, in order:

**Morning (16):** `asx200`, `sp500`, `audusd`, `au10y`, `brent_spot`, `vix`, `nasdaq`, `au2y`, `au_curve`, `us10y`, `us2y`, `brent_fut`, `gold_fut`, `rba_cash`, `asx200_fut`, `iron_ore`.

**Close adds (5):** `nikkei`, `hang_seng`, `shanghai`, `kospi`, `copper_fut`.

New `id` values may appear later. Render unknown ids in the expanded tile group rather than dropping them.

**Always present**, including when the quote is unavailable:

| Field | Type | Meaning | Observed values / units | Example |
| --- | --- | --- | --- | --- |
| `id` | string | Stable machine id. | See list above. | `"asx200"` |
| `display_name` | string | Label for the tile. | | `"S&P/ASX 200"` |
| `tier` | string | `"primary"` (shown in the main tile grid) or `"expanded"` (behind “All N metrics”). Morning primary: `asx200`, `sp500`, `audusd`, `au10y`, `brent_spot`, `vix`. Close primary adds `nikkei`, `hang_seng`. | `"primary"` |
| `instrument_type` | string | Kind of instrument. | `equity_index`, `fx`, `government_bond`, `commodity`, `volatility_index`, `spread`, `policy_rate`, `equity_index_future` | `"equity_index"` |
| `quote_kind` | string | How the number was taken. | `index_close`, `spot`, `yield`, `spread`, `futures_last` | `"index_close"` |
| `unit` | string | Unit of `value` / `display_value`. | `pts`, `USD`, `%`, `USD/bbl`, `USD/oz`, `USD/t`, `USD/lb` (close, copper), `bp` | `"pts"` |
| `currency` | string | ISO currency, or `""` for VIX. | `AUD`, `USD`, `JPY`, `HKD`, `CNY`, `KRW`, `""` | `"AUD"` |
| `decimals` | number (int) | Suggested decimal places. | 0, 1, 2, 3, 4 | `1` |
| `change_unit` | string | Unit of the daily change. | `pct`, `bp`, `pts` (VIX uses `pts`) | `"pct"` |
| `expected_session_date` | string (`YYYY-MM-DD`) | Session the snapshot was aiming for. | | `"2026-11-09"` |
| `observation_timezone` | string (IANA) | Time zone of the underlying market. Morning quotes are `Australia/Sydney` or `America/New_York`. Close also uses `Asia/Tokyo`, `Asia/Hong_Kong`, `Asia/Shanghai`, `Asia/Seoul`. | `"Australia/Sydney"` |
| `status` | string | `"ok"` or `"unavailable"`. | | `"ok"` |
| `freshness_state` | string | Quality flag for the tile. | `latest_official_close`, `live_or_delayed`, `scheduled_lag`, `stale`, `unavailable` | `"latest_official_close"` |
| `provider` | string or `null` | Data provider, often with a ticker in parentheses. `null` when unavailable. | `"Yahoo Finance (^AXJO)"` |
| `provider_url` | string or `null` | Link to the provider page. `null` when unavailable. | `"https://finance.yahoo.com/quote/%5EAXJO/"` |
| `licence_note` | string or `null` | Attribution / licence sentence. `null` when unavailable. Show in a “Data sources” footer, not on the tile. | |
| `official` | boolean or `null` | Whether this is an official series. `null` when unavailable. | `true` for RBA/FRED official series, `false` for delayed Yahoo quotes. |

**Present when `status` is `"ok"`; omitted (or unused) when `"unavailable"`:**

| Field | Type | Meaning | Example |
| --- | --- | --- | --- |
| `value` | number | Raw last value. High precision; do not display this directly. | `9020.7998` |
| `display_value` | string | Ready-to-show level. Yields already include `%`. The reference UI prefixes `US$` when `unit` is `USD/bbl`, `USD/oz`, `USD/t` or `USD/lb`. | `"9,020.8"`, `"5.08%"`, `"0.6900"` |
| `display_change` | string | Ready-to-show daily change, already signed, with `%`, `bp` or `pts`. Use typographic minus `−` as provided. | `"−0.5%"`, `"+3 bp"`, `"−2.54 pts"` |
| `change_abs` | number | Absolute change in the instrument's unit. Present for `change_unit` `pct` or `pts`, not for `bp`. | `-44.3` |
| `change_pct` | number | Percent change (0.44 = 0.44%). Present alongside `change_abs`. | `-0.5` |
| `change_bp` | number | Change in basis points. Present when `change_unit` is `"bp"`. | `-9` |
| `market_session_date` | string (`YYYY-MM-DD`) | Session the value actually belongs to (may be earlier than `expected_session_date` when lagged). | `"2026-11-06"` |
| `observation_timestamp_utc` | string (ISO 8601, `Z`) | Instant of the observation. | `"2026-11-06T05:55:00Z"` |
| `as_of_sydney` | string | Human “as of” line in Sydney time, already labelled AEDT/AEST. Optional on a few older morning quotes; prefer it when present. | `"6 Nov 2026, 16:55 AEDT"` |
| `comparison_date` | string (`YYYY-MM-DD`) | Session the change is measured against. | `"2026-11-05"` |
| `comparison_value` | number | Level on `comparison_date`. | `9065.1` |
| `source_label` | string | Short quality label. Observed: `"delayed, unofficial"`, `"official"`. | `"delayed, unofficial"` |
| `source_vintage` | string | Free-text retrieval stamp. Sometimes notes a historical backfill. | `"retrieved 2026-11-09 05:32Z"` |
| `note` | string | Optional caveat under the tile (lagged official series, instrument differences, etc.). | `"RBA interpolated 10-year yield; published with a lag of about two business days."` |
| `cross_checks` | array of objects | Optional independent confirmations. Each has `provider` (string), `value` (number), `session_date` (`YYYY-MM-DD`). Observed on some S&P 500 and Nasdaq quotes. | |

**Weekend morning only** (omitted on weekdays and on all close quotes):

| Field | Type | Meaning | Example |
| --- | --- | --- | --- |
| `display_week_change` | string | Ready-to-show weekly change. | `"+1.3% wk"`, `"−9 bp wk"` |
| `week_comparison_date` | string (`YYYY-MM-DD`) | Date the weekly change is measured against. | `"2026-10-30"` |
| `week_comparison_value` | number | Level on that date. | `8900.1` |
| `week_change_pct` | number | Weekly percent change, when the change is in percent. | `1.3` |
| `week_change_bp` | number | Weekly change in basis points, when the change is in bp. | `-13` |
| `week_change_abs` | number | Rare; weekly absolute change. Only a handful of quotes. | `5.15` |

**When `status` is `"unavailable"`:**

| Field | Type | Meaning | Example |
| --- | --- | --- | --- |
| `unavailable_reason` | string | Why there is no number. Show this on the dashed tile. | `"No free, permitted feed for ASX 24 SPI 200 futures (licensed exchange data)."` |

`asx200_fut` is unavailable in every file in the archive (licensed futures). `iron_ore` and `copper_fut` are sometimes unavailable (fetch or licence limits, or a display/value rounding mismatch). Render those tiles; do not hide the slot.

### 6.5 Markets sections and cards

Each section:

| Field | Type | Presence | Meaning |
| --- | --- | --- | --- |
| `id` | string | always | Section id (see the lists in §5.3). There is no `name` field in the JSON. Use the recommended headings in §7. |
| `cards` | array | always | Story cards. May be empty. |
| `note` | string | optional | Section-level caveat. |

Each card:

| Field | Type | Presence | Meaning |
| --- | --- | --- | --- |
| `id` | string | always | Anchor id, unique within the day (not necessarily globally). Example: `"asx200-close"`. |
| `headline` | string | always | Card title. Plain text. |
| `takeaway` | string | always | One-sentence dek, shown in the collapsed card. |
| `tags` | array of strings | always | 1–4 short topic labels. |
| `what_happened` | string | always | Body. **Markdown**: paragraphs, `**bold**`, and `[label](url)` links. |
| `why_it_matters` | string | always | Body, same Markdown rules. |
| `sources` | array of source objects | always | 1–9 named sources. §6.9. |
| `key_figure` | string | optional | Highlight figure, shown as a small pill on the collapsed card. Example: `"8,845.7 · +0.44% (19 Oct close)"`. |
| `new_since_last_issue` | string | optional | What changed since the previous briefing. |
| `competing_views` | array | optional | 1–3 attributed views. Each: `who` (string), `view` (string), `source` (source object). |
| `what_to_watch` | string | optional | Forward-looking sentence. May contain Markdown links. |
| `market_reaction` | string | optional | Morning cards only, in the archive. May contain Markdown links. |
| `priced_in` | string | optional | Morning cards only, rare. What is already in market prices. |
| `comparison` | object | optional | Side-by-side of disagreeing figures. §6.6. |
| `chart` | object | optional | Small bar chart. Almost always on the close ASX-close card (sector moves); rare on morning cards. §6.6. |

Suggested collapsed-card chrome (as in the reference UI): tags, headline, takeaway, `key_figure` if present, up to three `sources[].label` values with `+N` if there are more, a “Different figures · N sources” pill if `comparison` exists, and a “Competing views” pill if `competing_views` exists.

### 6.6 Comparison and chart objects

`comparison`:

| Field | Type | Presence | Meaning | Observed values |
| --- | --- | --- | --- | --- |
| `title` | string | always | Question the table answers. | `"Which Brent price?"` |
| `why_different` | string | always | Explanation under the table. |  |
| `resolution` | string | always | Observed: `"explained"`, `"unresolved"`. If a new value appears, still show `why_different` and label it from the raw value. |  |
| `rows` | array (2–4) | always | One row per source. |  |
| `rows[].source` | string | always | Name of the source. | `"Yahoo Finance BZ=F"` |
| `rows[].url` | string | always | Link. |  |
| `rows[].figure` | string | always | The reported number, already formatted. | `"US$98.32/bbl"` |
| `rows[].measure` | string | always | What the figure is. | `"Brent front-month futures, daily close"` |
| `rows[].timestamp` | string | always | When, free text. | `"6 Nov 2026 session"` |
| `rows[].method` | string | always | How it was produced. | `"Exchange futures close (delayed, unofficial)"` |
| `rows[].status` | string | always | Observed: `"primary measured"`, `"independently reported"`, `"estimate"`, `"official claim"`, `"unverified"`. |  |

`chart`:

| Field | Type | Presence | Meaning |
| --- | --- | --- | --- |
| `type` | string | always | Observed: `"bar"` only. |
| `label` | string | always | Chart title. |
| `labels` | array of strings | always | Category axis. Length matches `values`. |
| `values` | array of numbers | always | Bar values (ints or floats). |
| `unit` | string | always | e.g. `"%"`, `"m bbl/day"`. |
| `caption` | string | always | Sentence under the chart. |
| `source` | string | always | Attribution text. |
| `source_url` | string | optional | Link for the attribution. |

### 6.7 `rates`

Keys observed: `rba` (almost always) and `fed` (always on morning; often missing on close). Each value is an object:

| Field | Type | Presence | Meaning |
| --- | --- | --- | --- |
| `status` | string | always when the object exists | `"available"` or `"unavailable"`. |
| `summary` | string | almost always | The priced view, or `"Market pricing unavailable"`. Missing on one early weekend RBA row; when `status` is `"unavailable"` a summary is still often present. |
| `as_of` | string | optional | Human timestamp, mixed formats (`"6 Nov 2026, 2:26pm AEDT (RBA Rate Watch)"`, `"Reported 16 Oct 2026"`). |
| `source` | source object | optional | Usually present when status is `"available"`. Sometimes omitted even then. |
| `forecasts` | array | optional | Economist views. May be present even when market pricing is `"unavailable"` (e.g. 9 November morning RBA). May be an empty array. |

Each forecast: `who` (string), `view` (string), `source` (source object).

Reference UI labels: “RBA: what markets price” and “Fed: what markets price”. If `status !== "available"`, show “Market pricing unavailable” (and the `summary` if you want the longer reason), but still list `forecasts` when present, headed “Economist forecasts (opinion, attributed)”.

### 6.8 Calendar event

| Field | Type | Presence | Meaning |
| --- | --- | --- | --- |
| `when` | string | always | **Human timetable, not ISO.** Examples: `"Mon 9 Nov, 9:30am AEDT"`, `"Tue 10 Nov (time not confirmed)"`, `"Late Oct"`, `"Tue 20 Oct, ~8:30am"`. Do not parse this as a date. |
| `event` | string | always | Description. |
| `country` | string | always | Short code or label. Observed: `AU`, `US`, `EU`, `CN`, `UK`, `JP`, `NZ`, `DE`, `KR`, `CA`, `FR`, `IL`, `BR`, `TR`, `RU`, `INT`, `Global`, `Middle East`, and combined labels `AU/US`, `JP/CN`, `AU/UK`. |
| `importance` | string | always | `"high"`, `"medium"`, or `"low"`. The reference UI marks only `"high"` (a small “▲ High”). |
| `source` | source object | almost always | Omitted on a very small number of rows. |
| `prior` | string | optional | Previous print, already formatted. |
| `consensus` | string | optional | Consensus forecast, already formatted. |
| `consensus_source` | string | optional (present when `consensus` is) | Who compiled the consensus. |
| `actual` | string | rare (optional) | Printed result when already released. Seen once in the archive. |

### 6.9 Source object (markets)

Used on cards, competing views, rates, forecasts, and most calendar rows.

| Field | Type | Presence | Meaning | Observed values |
| --- | --- | --- | --- | --- |
| `label` | string | always | Display name. | `"ABC"`, `"Reuters via CNA"` |
| `url` | string | always | `https://…` link. Open in a new tab (`rel="noopener"`). |  |
| `claim_type` | string | usual on cards; often missing on older rate/calendar sources | Kind of claim. Observed: `reporting`, `measurement`, `official_statement`, `forecast`, `opinion`. |  |
| `date` | string | usual, but not always | Publication date, **mixed formats**: `"2026-10-19"`, `"19 Oct 2026"`, `"Nov 2026"`. |  |
| `perspective` | string | rare | Editorial perspective, when the publisher flagged one. Examples: `"wire"`. Optional; show only if you have a design for it. |  |

### 6.10 `ai.json` — top level

| Field | Type | Presence | Meaning | Example |
| --- | --- | --- | --- | --- |
| `schema_version` | number (int) | always | Currently `1`. | `1` |
| `type` | string | always | Always `"ai"`. | `"ai"` |
| `date` | string (`YYYY-MM-DD`) | always | Sydney date. | `"2026-11-09"` |
| `generated_at_utc` | string | always | UTC timestamp, `YYYY-MM-DDTHH:MM:SSZ`. | `"2026-11-08T21:36:47Z"` |
| `title` | string | always | Page heading. | `"Daily AI report · Monday 9 Nov 2026"` |
| `top_stories` | array of 3 objects | always | Pointers to the three lead cards. |  |
| `sections` | array of 7 objects | always | Always these seven, in this order. |  |

There is **no `disclaimer` field** in `ai.json`. Still show the same disclaimer on the AI page (from the markets wording, or the short form “Information only, not financial advice.”).

`top_stories[]`:

| Field | Type | Presence | Meaning |
| --- | --- | --- | --- |
| `section_id` | string | always | Matches a `sections[].id`. Observed: `coding`, `evals`, `agenteng`, `models`, `industry`, `papers` (not every section is used as a lead every day). |
| `card_id` | string | always | Matches a `sections[].cards[].id` in that section. Every pointer in the archive resolves. |
| `headline` | string | always | Copy of that card's headline, for convenience. Prefer looking up the full card for the body. |

`sections[]`:

| Field | Type | Presence | Meaning | Observed values |
| --- | --- | --- | --- | --- |
| `id` | string | always | Machine id. | `coding`, `evals`, `agenteng`, `models`, `industry`, `papers`, `repos` (this order, every day) |
| `name` | string | always | Heading. | `Coding Agents`, `AI Evals, Safety & Governance`, `Agent Engineering & Frameworks`, `Models & Major Releases`, `AI Industry, Policy & Banking`, `Papers Worth Reading`, `Repos Worth Trying` |
| `cards` | array | always | 3–10 cards (typically 3–5). |  |

`sections[].cards[]`:

| Field | Type | Presence | Meaning | Notes |
| --- | --- | --- | --- | --- |
| `id` | string | always | Anchor id. |  |
| `headline` | string | always | Title. |  |
| `takeaway` | string | always | One-line dek. |  |
| `tags` | array of strings | always | 1–3 labels. |  |
| `sources` | array of **strings** | always | Short source names only (`"Changelog"`, `"Docs"`, `"Blog"`, `"X"`). **Not** objects — unlike markets sources. |  |
| `verdict` | string | always | Exactly one of `"Worth trying"`, `"Worth reading"`, `"Monitor only"`. |  |
| `what_changed` | string | always | Body. |  |
| `why_it_matters` | string | always | Body. |  |
| `caveats` | string | always | May be `""`. Hide the row when empty. |  |
| `links` | array of `{label, url}` | always | 1–6 outbound links. |  |
| `how_to_try` | string | optional | Practical next step. Present on about one in six cards, mostly in `coding`. |  |

---

## 7. UI reference

Match the look of the screenshots in `screenshots/` (same folder as this file). Branding in those images has been replaced with a generic “Daily briefing” wordmark; your internal site should use your organisation's own header. The layout, typography, colour, cards, tiles, tabs and chips are what to copy.

All typography, colour and width values below are taken from the reference CSS, not guessed.

### 7.1 Screenshot index

Desktop width is 1280px; mobile width is 390px. “Full” is a full-page capture; “above-fold” is the first screen.

**Home page (latest briefings)**

![Home page at 1280 pixels, above the fold](screenshots/home-1280-above-fold.png)

![Home page at 1280 pixels, full page](screenshots/home-1280-full.png)

![Home page at 390 pixels, above the fold](screenshots/home-390-above-fold.png)

![Home page at 390 pixels, full page](screenshots/home-390-full.png)

**Weekday Markets, Monday 9 November 2026 — Morning tab**

![Weekday Markets morning tab at 1280 pixels, above the fold: badges, Morning and Close tabs, chip nav, three-bullet summary](screenshots/markets-2026-11-09-morning-1280-above-fold.png)

![Weekday Markets morning tab at 1280 pixels, full page](screenshots/markets-2026-11-09-morning-1280-full.png)

![Primary Markets at a glance tiles, weekday morning](screenshots/markets-2026-11-09-morning-1280-tiles.png)

![Collapsed story cards in What Moved and Why, weekday morning](screenshots/markets-2026-11-09-morning-1280-cards.png)

![What comes next calendar, weekday morning](screenshots/markets-2026-11-09-morning-1280-calendar.png)

![Weekday Markets morning tab at 390 pixels, above the fold](screenshots/markets-2026-11-09-morning-390-above-fold.png)

![Weekday Markets morning tab at 390 pixels, full page](screenshots/markets-2026-11-09-morning-390-full.png)

**Weekday Markets, Monday 9 November 2026 — Close tab**

![Weekday Markets close tab at 1280 pixels, above the fold](screenshots/markets-2026-11-09-close-1280-above-fold.png)

![Weekday Markets close tab at 1280 pixels, full page](screenshots/markets-2026-11-09-close-1280-full.png)

![Primary Markets at a glance tiles, weekday close (includes Asia indices)](screenshots/markets-2026-11-09-close-1280-tiles.png)

![ASX close story cards](screenshots/markets-2026-11-09-close-1280-cards.png)

![Tonight and overnight calendar, weekday close](screenshots/markets-2026-11-09-close-1280-calendar.png)

![Weekday Markets close tab at 390 pixels, above the fold](screenshots/markets-2026-11-09-close-390-above-fold.png)

![Weekday Markets close tab at 390 pixels, full page](screenshots/markets-2026-11-09-close-390-full.png)

**Weekend Markets, Sunday 8 November 2026** (no Close tab; left section sidebar on desktop)

![Weekend Markets at 1280 pixels, above the fold, with left section nav](screenshots/markets-2026-11-08-weekend-1280-above-fold.png)

![Weekend Markets at 1280 pixels, full page](screenshots/markets-2026-11-08-weekend-1280-full.png)

![Weekend Markets at a glance tiles; some quotes may show a muted weekly change](screenshots/markets-2026-11-08-weekend-1280-tiles.png)

![Weekend Markets story cards](screenshots/markets-2026-11-08-weekend-1280-cards.png)

![Weekend What comes next calendar](screenshots/markets-2026-11-08-weekend-1280-calendar.png)

![Weekend Markets at 390 pixels, above the fold](screenshots/markets-2026-11-08-weekend-390-above-fold.png)

![Weekend Markets at 390 pixels, full page](screenshots/markets-2026-11-08-weekend-390-full.png)

**AI report, Monday 9 November 2026**

![AI report at 1280 pixels, above the fold: left section nav and Top stories lead panel](screenshots/ai-2026-11-09-1280-above-fold.png)

![AI report at 1280 pixels, full page](screenshots/ai-2026-11-09-1280-full.png)

![Top stories panel: one lead story on the left, two supporting stories stacked on the right](screenshots/ai-2026-11-09-1280-top-stories.png)

![A section of expandable AI cards with verdict pills](screenshots/ai-2026-11-09-1280-cards.png)

![AI report at 390 pixels, above the fold](screenshots/ai-2026-11-09-390-above-fold.png)

![AI report at 390 pixels, full page](screenshots/ai-2026-11-09-390-full.png)

### 7.2 Colour palette

CSS custom properties (light, which is what the screenshots use):

| Token | Hex | Typical use |
| --- | --- | --- |
| `--ink` | `#0f1724` | Primary text, tab bar baseline, theme colour |
| `--paper` | `#f3f2ee` | Page background |
| `--teal` / `--accent` / `--link` | `#2f7a75` | Links, active chips, section accents, tab and topic underlines |
| `--teal-bright` | `#66b8ac` | Dark-mode accent (and some hover states) |
| `--bg-elevated` | `#ffffff` | Cards, tiles, chips, tab surfaces |
| `--text-muted` | `#5c6570` | Meta, session line, tile names, footnotes |
| `--border` | `#e0ddd6` | Hairlines, card and tile borders |
| `--accent-soft` | `#e4f1ef` | Active chip fill, key-figure pill, “new since last issue” band |
| `--link-hover` | `#1f5a52` | Link hover |
| `--badge-weekend` | `#5a4a3a` | Weekend badge |
| `--badge-ai` | `#3d5a80` | AI badge (also the “Worth reading” verdict, light) |
| `--badge-unconfirmed` | `#595959` | Reserved badge colour |
| `--rule-strong` | same as `--text` / `--ink` | 2px rules under the home masthead and section titles |
| `--shadow-sm` | `0 1px 2px color-mix(in srgb, var(--ink) 6%, transparent)` | Cards, active nav items |
| `--shadow-md` | `0 1px 2px` ink 5% plus `0 8px 24px -12px` ink 18% | Elevated cards and the Top stories panel |

Supporting (not tokens, but used in the CSS):

| Use | Hex |
| --- | --- |
| Lagged flag / “Different figures” pill text | `#8a6a2a` (border `#b08a3e`, pill fill is a mix of `#f5e7c8`) |
| Stale flag | `#a14b3a` |
| Chart value up | `#1a7a4c` |
| Chart value down | `#a33b3b` |
| Verdict “Worth trying” | background `#1b6b3a`, text `#fff` |
| Verdict “Worth reading” | background `#3d5a80`, text `#fff` |
| Verdict “Monitor only” | background `#5c5c5c`, text `#fff` |

Dark-mode alternatives exist (`--bg` becomes `#0f1724`, `--text` becomes `#f3f2ee`, `--border` `#2a3344`, `--accent` `#66b8ac`, verdicts lighten). Implement dark mode if you want; the screenshots are light.

The page background also uses a faint radial wash of `--accent-soft` at the top.

### 7.3 Typography

Self-hosted files:

- **Source Serif 4** (400, 400 italic, 600, 600 italic) — body, page titles, section titles, card headlines, Morning/Close tab labels, the home hero.
- **IBM Plex Mono** (400, 500) — session lines, tile values, tags, calendar times, section numbers, related chrome.

Fallback stacks from CSS:

- Serif: `"Source Serif 4", "Iowan Old Style", "Palatino Linotype", Palatino, Georgia, serif`
- Sans: `-apple-system, BlinkMacSystemFont, "Segoe UI", "Helvetica Neue", Helvetica, Arial, sans-serif` (system; not self-hosted)
- Mono: `"IBM Plex Mono", ui-monospace, "Cascadia Code", "SF Mono", Menlo, monospace`

The CSS sets some titles to `font-weight: 700`, but the self-hosted Source Serif 4 files only include 400 and 600, so the browser may synthesise 700. Shipping 400 and 600 is enough to match the reference closely.

Key sizes (root is 16px; body is 17px / line-height 1.6 / weight 400). From a 1600px viewport upward the reference scales `html` to `106.25%` and `body` to `18px`; the screenshots were taken at 1280px, so they show the 17px body.

| Element | Family | Size | Weight | Notes |
| --- | --- | --- | --- | --- |
| Body | serif | 17px | 400 |  |
| Home hero (`.landing__hero`) | serif | `clamp(2.4rem, 1.6rem + 4vw, 4.25rem)` | 700 | line-height 1.02, letter-spacing `-0.03em`. The screenshots use the generic line “Today's briefings”; use a heading that fits your organisation. |
| Home subline | serif | `clamp(1.05rem, 1rem + 0.4vw, 1.25rem)` | 400 | max-width `36rem` |
| Home card title | serif | ~1.15–1.25rem | 600 |  |
| Markets page title (`.issue-title`) | serif | 1.65rem (1.45rem below 640px) | 700 | letter-spacing `-0.015em`, line-height 1.25 |
| AI page title | serif | `clamp(1.6rem, 1.1rem + 2.2vw, 2.4rem)` | 600 | line-height 1.15, letter-spacing `-0.022em` |
| Morning/Close tab | serif | 1.05rem | 600 |  |
| Summary bullets | serif | 1.05rem (0.95rem below 640px) | 400 | line-height 1.45; marker in mono, accent colour |
| Section title (e.g. “Markets at a glance”, AI section names) | serif | 1.35rem | 600 | line-height 1.25, letter-spacing `-0.015em`, **not** uppercase; 2px solid `--rule-strong` top border; margin `2.5rem 0 1rem`, padding-top `0.7rem` |
| AI section number | mono | 0.75rem | 600 | accent colour, letter-spacing `0.08em` |
| Topic nav link | sans | 0.86rem | 600 | muted until active |
| Tile name | sans | 0.72rem | 600 | muted, ellipsis |
| Tile value | mono | 1.2rem | 500 | tabular-nums |
| Tile change | mono | 0.8rem | 500 | weekly change is muted, weight 400 |
| Tile / calendar micro text | sans or mono | 0.68–0.72rem | 400 | muted |
| Card headline (Markets) | serif | 1.05rem | 600 | line-height 1.3 |
| Card takeaway (Markets) | serif | 0.92rem | 400 |  |
| Key figure pill | mono | 0.8rem | 400 | `--accent-soft` background |
| Tags | mono | 0.62rem | 400 | uppercase, accent colour, `·` separators |
| Chip / nav item | sans | 0.8rem | 600 |  |
| Badge (`MARKETS`, `WEEKDAY`) | sans | 0.7rem | 600 | uppercase, 1px currentColor border, 2px radius |
| Footer disclaimer | sans, italic | 0.75rem | 400 | muted |
| Top stories lead headline | serif | `clamp(1.3rem, 1rem + 1.4vw, 1.6rem)` | 600 | line-height 1.22, letter-spacing `-0.02em` |
| Top stories supporting headline | serif | 1.12rem | 600 | line-height 1.3 |
| Top stories lead standfirst | serif | 1rem | 400 |  |
| Top stories supporting standfirst | serif | 0.93rem | 400 | line-height 1.55, clamped to 4 lines when the panel is wide |
| AI card headline | sans | 1.16rem | 600 | line-height 1.32 |
| AI takeaway | sans | 0.95rem | 400 | line-height 1.55 |
| AI verdict pill | sans | 0.68rem | 700 | 999px radius, white text |

Tap targets are at least `--tap-min: 44px`.

### 7.4 Layout widths and spacing

| Token / rule | Value | Where it applies |
| --- | --- | --- |
| `--shell` | `76rem` | `.site-header__bar`, `.site-shell`, `.site-footer__inner` — one centred max-width for header, content and footer |
| `--reading` | `46rem` | Markets and AI reading columns |
| `--page-pad` | `max(1rem, env(safe-area-inset-left))`; `1.25rem` from 1024px | Horizontal page padding |
| Home article (`body:has(.landing) .site-main`) | `max-width: 64rem` | Home only |
| Tabbed weekday Markets | reading column `max-width: var(--reading)` (`46rem`), centred; the left sidebar is hidden | Morning/Close tabs |
| Weekend Markets (from 1024px) | `grid-template-columns: 13.5rem minmax(0, var(--reading)); justify-content: center; column-gap: clamp(2rem, 4vw, 3.5rem)` | Left section nav + 46rem article, centred as one unit |
| AI desktop (from 900px) | inner grid `13.5rem minmax(0, var(--reading))`, `justify-content: center`, same column-gap as weekend; the outer shell is a single column | Left “Sections” list + article |
| Card / tile / Top stories radius | `--radius: 10px` | 1px `--border`; small controls use `--radius-sm: 6px` |
| Sticky header offset | `--header-offset: 3.25rem` | Chip bars stick just under the header |
| Vertical rhythm | `--space: 1.125rem` | Footer padding, some margins |
| Tile grid | 2 columns; **3 columns from 720px**; gap `0.5rem` |  |
| Home cards | one column; **two equal cards from 800px** (`repeat(auto-fit, minmax(20rem, 1fr))`), gap `1.25rem` |  |
| Rates grid | 1 column; **2 columns from 720px** |  |
| Comparison table | `min-width: 34rem` with horizontal scroll on desktop; **stacks into labelled blocks at `max-width: 640px`** |  |
| Chip radius | `999px` |  |

Header: sticky, blurred (`backdrop-filter: blur(10px)`), 1px bottom border. The bar is a three-column grid (`auto minmax(0, 1fr) auto` on a phone; `minmax(0, 1fr) auto minmax(0, 1fr)` from 1024px) so the wordmark sits on the left, topic links (Markets, AI) plus an optional Archive link sit in the middle from 1024px (or in a row next to the wordmark on a phone), and the right cell can stay empty. The active topic is body-coloured text with a 2px accent underline (`::after`, inset slightly from the padding). Do not render a sign-in or share control on the internal site.

Footer: 1px top border, disclaimer, no product motto required.

### 7.5 Home page structure and JSON mapping

Use `index.json` (or the two `latest/*.json` files) to build the home page. Two cards, one per topic:

| UI | JSON |
| --- | --- |
| Masthead date | `date` from the newest file (or `index.json` latest dates), formatted as a long Sydney date, plus a fixed “Published 08:25 Sydney” line if you want to match the reference |
| Hero and subline | Your own heading; subline of the form “The few things that matter today, in Markets and AI.” |
| Card topic label | “Markets” / “AI” |
| Markets status chip | “After the close” when `latest/markets.json` has a non-null `close`; otherwise “Morning briefing” |
| Card title | `title`, linking to the topic page |
| Card date | `date` |
| Markets top items (numbered 01–03) | First three bullets of `close.summary` if `close` is present, otherwise `morning.summary`. Strip Markdown to plain text and truncate long bullets. |
| AI top items (numbered 01–03) | The three `top_stories[].headline` values, each linking to the card on the AI page (`#` + `card_id`) |
| Card call to action | “Read the Markets briefing” / “Read the AI briefing” |
| Archive link | Optional link to your archive of past issues |

Do not add a sign-up band or a per-share panel on the home page. That data is not in the feed.

### 7.6 Markets page structure and JSON mapping

Recommended section order and headings (the JSON has ids only; these are the headings used in the reference UI):

**Every edition, first:**

| UI heading | JSON | Notes |
| --- | --- | --- |
| (header) `MARKETS` badge + `WEEKDAY`/`WEEKEND` badge | `type`, `edition` |  |
| Page title | `title` |  |
| Date line | `date` + `edition` | e.g. “Monday 9 November 2026 · Weekday edition” |
| “Data as of HH:MM AEDT/AEST” | `cutoff_local` | Derive the abbreviation from the numeric offset |
| Morning / Close tabs | presence of `close` | Omit the tab bar when `close` is `null` |
| Chip / sidebar nav | built from the headings below, with a count pill | Counts: 3 for the summary, number of quotes for tiles, `cards.length` for story sections, `calendar.length` for the diary |

**Then the body of the active edition** (`morning` or `close`):

| UI heading | JSON | Count in the chip |
| --- | --- | --- |
| If you read nothing else | `summary` (3 bullets). Render Markdown links. | 3 |
| Markets at a glance | `prices.quotes` where `tier === "primary"`, plus a disclosure “All N metrics” for `tier === "expanded"`. Sub-note: “Each figure shows its own session. Changes are versus the prior session. Yields in basis points.” | N = all quotes, including unavailable |
| What Moved & Why | section `market-drivers` | `cards.length` |
| Australia & ASX | `australia-asx` |  |
| Rates & Expectations | `rates-expectations`, and render the `rates` block at the top of this section |  |
| Global Macro & Risk | `global-risk` |  |
| What comes next | `calendar` (morning / weekend) | `calendar.length` |
| ASX close | `asx-close` (close tab only) |  |
| Asia close | `asia-close` (close tab only). Show `section.note` if present — Asian markets are often still open at 16:30 Sydney. |  |
| Rates & FX | `rates-fx` plus the `rates` block |  |
| Commodities | `commodities` |  |
| Tonight | `tonight` |  |
| Still open | `still-open` |  |
| Tonight & overnight | `calendar` (close tab only) |  |

Do not invent a per-share list. That data is not in the feed.

Under the calendar, a collapsed “Data sources, licences and what was left out” disclosure lists each `status === "ok"` quote's `display_name`, `provider` (linked with `provider_url`), `quote_kind`, `market_session_date` and `licence_note`. End the page with the disclaimer.

**Tiles.** For `status === "ok"`: `display_name`, `display_value` (with a `US$` prefix for the commodity units listed in §6.4.1), `display_change`, and on weekends `display_week_change` in muted text. Micro line: `as_of_sydney` (or the session date) · first clause of `provider`. Flags from `freshness_state`:

- `live_or_delayed` → “live/intraday” (accent border)
- `latest_official_close` → “close”
- `scheduled_lag` → “lagged” (amber)
- `stale` → “stale” (red)
- plus “delayed, unofficial” when `official === false`

For `status !== "ok"`: dashed tile, italic “Unavailable”, and `unavailable_reason`. The reference UI does **not** colour daily changes green/red on tiles.

**Cards.** Collapsed `<details>` with a circular `+`/`−` control in accent colour. Expanded body, in this order when the field exists: “New since last issue”, “What happened”, “Why it matters”, “Market reaction”, “What's priced in”, chart, comparison table, competing views, “What to watch”, “Sources” (linked label, optional `claim_type` chip, optional `date`). Render Markdown in the prose fields.

### 7.7 AI page structure and JSON mapping

| UI | JSON |
| --- | --- |
| `AI` badge + `title` | `type`, `title` |
| Date | `date` |
| Left nav / chips (section `name` + card count; an “All” item) | `sections` |
| Top stories (numbered 01–03, category, headline as a link, `why_it_matters`, “Read analysis” jump link) | `top_stories` resolved to the full card |
| Verdict legend | the three `verdict` values, with short meanings: “a tool or workflow you can test now” / “important context or research” / “good to know, no action needed yet” |
| Section heading (`name` + count) | `sections[]` |
| Card: verdict pill, headline, takeaway, “Read details” | `cards[]`. Do not repeat the category on the card — the section heading already names it. Expanded: What changed, Why it matters, Caveats (if non-empty), How to try (if present), links, string sources |

The Top stories block is one panel (1px border, 10px radius, light shadow). On a wide reading column (container query `min-width: 42rem`, typical at 1280px) the first item is a lead story on the left (`minmax(0, 1.5fr)`, teal 3px inset bar at the top) and the other two stack on the right (`minmax(0, 1fr)`). Between 34rem and 42rem the lead spans the full width and the other two sit in a two-column row. On a phone the three stories stack. Headlines inside the panel are links to the card further down the page. From 900px the section list becomes a sticky left column; on a phone, section chips are a horizontal scroller under the header.

### 7.8 Mobile behaviour (390px)

- Tiles drop to two columns (the 3-column grid only starts at 720px).
- Home cards stack to one column (the two-column home grid starts at 800px).
- Horizontal chip nav under the tabs (weekday) or under the header (weekend / AI), with scroll, no visible scrollbar (`scrollbar-width: none`).
- Page title shrinks to 1.45rem on Markets below 640px; summary tightens.
- Comparison tables un-stack into blocks, each cell prefixed with its column label.
- Keep 44px minimum tap height on tabs, chips, card summaries and “All N metrics”.
- Avoid typographic overflow: long URLs already `overflow-wrap: anywhere` in the reference CSS; keep that.

---

## 8. Rendering guidance

- **Timezone.** Briefing `date` is a Sydney calendar date. Show clock times in Australia/Sydney. Convert any `…Z` timestamp before display. For `cutoff_local`, use the embedded offset to choose AEDT or AEST. `as_of_sydney` is already a display string — print it as is.
- **Markdown.** `summary`, `what_happened`, `why_it_matters`, `market_reaction`, `priced_in` and `what_to_watch` may contain `[label](url)` and `**bold**`. Render safely (no raw HTML from the feed).
- **Attribution.** Every claim that has a `source` or `sources` entry should remain attributable. Link the label to `url`. Do not drop sources in the expanded card.
- **Optional fields.** If a key is missing, skip that UI element. Do not show empty “Caveats” or “How to try” rows. Do not invent consensus numbers.
- **Unavailable data.** Keep the tile or rates card on screen, in the dashed / italic treatment, with the reason when you have it. Forecasts can still appear when market pricing is unavailable.
- **Defensive parsing.** Ignore unknown keys and unknown quote ids (show the latter in the expanded grid). Accept both int and float for numeric fields. Accept mixed date-string formats. Accept `retrieved_at_utc` with a space instead of `T`.
- **Do not use `index.json` as a freshness check** for today's close wrap. Use the commit SHA or the hash/ETag of `latest/markets.json`.
- **Serve mirrored files.** The Pages app should load `/data/latest/markets.json` (or baked-in HTML) from the same origin.
- **Disclaimer** on Markets and AI, in the footer and at the end of the Markets body.
- **Archive.** `index.days` is enough to build a date list. For a date with only `markets.json`, hide or disable the AI tab.
- **Schema changes.** `schema_version` is `1` today. If it increases, read the new README and extend this mapping; until then, new keys under version 1 should be ignored rather than treated as errors.

---

## 9. Validation checklist and acceptance criteria

### CI / feed mirroring

- [ ] A scheduled workflow runs at least every 15 minutes and can also be run by hand.
- [ ] The workflow compares the latest `main` SHA (or the ETag of `latest/markets.json` and `latest/ai.json`) and **exits early** when nothing changed.
- [ ] Detection does **not** depend only on `index.latest.*.date`.
- [ ] When the SHA changes, the workflow downloads `index.json`, `latest/markets.json` and `latest/ai.json`, refreshes `data/days/<date>/` for the latest dates, and fills in any missing historical days from `index.days`.
- [ ] Downloaded files parse as JSON. Required top-level keys in §6 are present. `type` is `markets` or `ai` as appropriate.
- [ ] A day listed without `ai.json` (e.g. 2026-10-29) does not fail the job.
- [ ] Files are committed only when content changed, then Pages rebuilds from the mirrored `data/` folder.
- [ ] No secret or token is written into the Pages artefact.

### Markets page

- [ ] Today's morning briefing renders from `morning`, including the three-bullet summary, primary tiles, four story sections, rates and calendar.
- [ ] When `close` is an object, Morning/Close tabs work without a reload; Close shows the six close sections, the 21-quote snapshot, and “Tonight & overnight”.
- [ ] When `close` is `null` (weekends, or a weekday before the close update), the tab bar is omitted and the weekend/morning layout is used.
- [ ] Week-over-week change strings appear on weekend tiles when `display_week_change` is present, and not on weekday tiles.
- [ ] Unavailable quotes (notably SPI futures) show as dashed “Unavailable” tiles with the reason, not as blanks.
- [ ] Lagged / close / live / unofficial flags match `freshness_state` and `official`.
- [ ] Card Markdown links work. Comparison tables and competing views render when present.
- [ ] Calendar shows `when`, `event`, country, high-importance mark, actual/consensus/prior when present, and a source link.
- [ ] The disclaimer is visible.
- [ ] Layout matches the screenshots at 1280px and 390px: 46rem centred article on tabbed weekdays, 13.5rem sidebar plus 46rem article centred as one unit on weekend desktop, 64rem home article, 2-column tiles on a phone, 3-column tiles from 720px, and the AI Top stories lead-plus-stack layout at 1280px.

### AI page

- [ ] Seven sections render in the stated order, with names from the file.
- [ ] Three top stories resolve to real cards and jump to them.
- [ ] Verdict pills use the three official strings and the colours in §7.2.
- [ ] `how_to_try` shows only when present; empty `caveats` are hidden.
- [ ] `sources` are treated as strings; `links` are treated as `{label, url}`.
- [ ] The same disclaimer appears in the footer.
- [ ] A missing `ai.json` for a date is handled cleanly.

### Quality bar

- [ ] No client-only dependency on `raw.githubusercontent.com` for first paint.
- [ ] Unknown JSON fields do not crash the build or the page.
- [ ] Opening an older `days/<date>/` file (weekday with close, weekday without, weekend, AI) all work.
- [ ] After a morning run, the site shows the new day; after a close run the same day, the Close tab appears or updates without manual intervention (the next successful CI poll is enough).

---

## 10. Things that are easy to get wrong

1. **Close updates do not change `index.json`.** Poll the commit SHA or the markets file itself.
2. **`retrieved_at_utc` is not always ISO-8601.** Use a loose parser.
3. **`calendar.when` is not a timestamp.** Display it verbatim.
4. **AI `sources` are strings; markets `sources` are objects.**
5. **`change_pct` is already in percent** (`-0.5` is −0.5%). Use `display_change` in the UI.
6. **`asx200_fut` is always unavailable** in this public feed. That is expected, not a fetch bug.
7. **`generated_at_utc` can be identical across many historical markets files** after a bulk rewrite. Do not sort or de-duplicate days by that field.
8. **Sydney offset changes in April and October.** Prefer polling every 15 minutes over a single UTC cron window.

That is the whole contract: mirror the two JSON feeds, detect both the morning and close writes, render the pages from the fields above, and keep the disclaimer in view.
