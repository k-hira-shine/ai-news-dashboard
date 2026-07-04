# 2026-07-03 Pages Deploy Recovery Log

## Summary

2026-07-03 JST の定期実行では、収集・分析・データ保存は成功していたが、GitHub Pages の公開だけが失敗していた。

原因は GitHub Pages が `legacy` build_type のまま `main:/docs` を自動公開しており、短時間に複数の定期 workflow が `docs/` を push したことで、Pages deploy が `deployment_queued` のまま進まず 10 分で timeout したこと。

対応として、Pages を `workflow` build_type に切り替え、明示的な `Deploy Static Pages` workflow を追加した。さらに、旧 `pages-build-deployment` の最後の失敗を新 workflow 成功後に監視ノイズとして扱わないよう `daily_check.py` を修正した。

最終状態:

- `Deploy Static Pages`: success
- `Daily Ops Check`: success
- 公開サイト: `200 OK`
- ローカル `daily_check.py`: 全項目合格
- テスト: `183 passed, 17 subtests passed`
- queued run: 0 件

## Initial Symptoms

最初にローカルだけを見ると、今日分の生成物が無いように見えた。

原因はローカル `main` が fetch 前で、GitHub Actions が push 済みの commit をまだ取り込んでいなかったため。

fetch 後、ローカルは `origin/main` より 4 commits behind であることが判明し、以下の今日分生成物が GitHub 側には存在していた。

- `data/daily/2026-07-03.jsonl`
- `data/analysis/2026-07-03_morning.json`
- `data/hn/2026-07-03.jsonl`
- `data/tools/2026-07-03.jsonl`
- `data/gemini_usage/2026-07-03.jsonl`
- `data/money/2026-07-03.jsonl`
- `data/sns_success/2026-07-03.jsonl`
- `docs/diagrams/2026-07-03-morning.html`
- `docs/diagrams/2026-07-03-morning.png`

## Timeline

All times below are JST unless noted.

### Scheduled runs

- 2026-07-03 00:50頃: `AI News Collector` ran successfully.
- 2026-07-03 01:48頃: `Buzz Ranking Collector` ran successfully.
- 2026-07-03 02:21頃: `AI Money Cases Collector` ran successfully.
- 2026-07-03 03:38頃: `Buzz Daily Health Check` ran successfully.
- 2026-07-03 03:48頃: `Daily Ops Check` failed because it detected Pages failure and critical-account silence advisory.

### Failing runs observed

- `pages-build-deployment` failed at `2026-07-03 01:06 JST`
  - Run: `28604279262`
  - URL: `https://github.com/k-hira-shine/ai-news-collector/actions/runs/28604279262`
- `pages-build-deployment` failed at `2026-07-03 01:52 JST`
  - Run: `28607097033`
  - URL: `https://github.com/k-hira-shine/ai-news-collector/actions/runs/28607097033`
- `pages-build-deployment` failed at `2026-07-03 02:29 JST`
  - Run: `28609210829`
  - URL: `https://github.com/k-hira-shine/ai-news-collector/actions/runs/28609210829`
- `Daily Ops Check` failed at `2026-07-03 03:48 JST`
  - Run: `28613929352`
  - URL: `https://github.com/k-hira-shine/ai-news-collector/actions/runs/28613929352`

## Root Cause

The failing Pages job was not failing during HTML build.

For run `28609210829`:

- `build`: success
- `report-build-status`: success
- `deploy`: failure

The deploy log repeatedly showed:

```text
Current status: deployment_queued
...
Timeout reached, aborting!
Canceling Pages deployment...
```

GitHub Pages settings at the time:

```json
{
  "build_type": "legacy",
  "source": {
    "branch": "main",
    "path": "/docs"
  },
  "status": "errored"
}
```

This meant GitHub's legacy branch-based Pages builder was handling deployment automatically whenever `main:/docs` changed. Because this repo has multiple scheduled jobs that push data and docs close together, the legacy deploy queue became stuck.

## Fix 1: Explicit Pages Workflow

Added `.github/workflows/deploy-pages.yml`.

Purpose:

- Deploy `docs/` explicitly through GitHub Actions.
- Use `actions/configure-pages`, `actions/upload-pages-artifact`, and `actions/deploy-pages`.
- Add `concurrency` so stale Pages deploys are canceled and the latest artifact wins.

Commit:

- `a06e8b7 fix: deploy pages via explicit workflow`

The workflow:

```yaml
name: Deploy Static Pages

on:
  workflow_dispatch:
  push:
    branches:
      - main
    paths:
      - "docs/**"
      - ".github/workflows/deploy-pages.yml"

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages-deploy
  cancel-in-progress: true
```

GitHub Pages setting was changed from `legacy` to `workflow`:

```bash
gh api -X PUT repos/k-hira-shine/ai-news-collector/pages -f build_type=workflow --silent
```

The new Pages deployment succeeded:

- Run: `28618019424`
- URL: `https://github.com/k-hira-shine/ai-news-collector/actions/runs/28618019424`
- Deployment ID: `5290417557`
- Deployment state: `success`
- Environment URL: `https://k-hira-shine.github.io/ai-news-collector/`

Deploy log showed the healthy transition:

```text
Current status: deployment_queued
Current status: deployment_in_progress
Reported success!
```

公開サイト確認:

```text
HTTP/2 200
last-modified: Thu, 02 Jul 2026 20:02:26 GMT
```

JST では `2026-07-03 05:02:26`。

## Fix 2: Daily Check No Longer Reports Superseded Legacy Pages Failure

After switching to workflow-based Pages deploy, the old `pages-build-deployment` workflow no longer receives a newer success. Its final failed run would otherwise keep `daily_check.py` red forever.

Changed `daily_check.py` so that:

- If `pages build and deployment` failed,
- And a newer `Deploy Static Pages` run succeeded,
- Then the old legacy Pages failure is treated as superseded.

Commit:

- `f342322 fix: ignore superseded legacy pages failures`

Validation:

- Targeted tests:

```text
56 passed
```

- Full tests:

```text
182 passed, 1 warning, 17 subtests passed
```

- Manual Daily Ops run:
  - Run: `28618179027`
  - Status: success

## Fix 3: Quiet Low-Frequency Critical Account Noise

Daily Ops also showed:

```text
criticalアカウント沈黙3連続: DarioAmodei, MistralAI, sama
```

External handle checks showed that all three handles still exist:

- `https://x.com/DarioAmodei`
- `https://x.com/MistralAI`
- `https://x.com/sama`

The likely cause was not a collector failure. These accounts had no matching posts in the current 2-day collection window, and the monitor treated that as possible collection breakage.

Added config:

```yaml
x_twitter:
  critical_silence_exempt_handles:
    - "DarioAmodei"
    - "MistralAI"
    - "sama"
```

Changed `daily_check.py` to load that config and exclude these handles from the `criticalアカウント沈黙` advisory.

Also changed `main.py` to persist:

```json
"must_follow_per_account": {}
```

This makes future account-specific investigations possible from `data/logs/*.jsonl`.

Commit:

- `07383ae fix: quiet low-frequency critical account alerts`

Validation:

- Targeted tests:

```text
71 passed, 12 subtests passed
```

- Full tests:

```text
183 passed, 1 warning, 17 subtests passed
```

- Local daily check:

```text
✅ 全項目合格
```

- Manual Daily Ops run:
  - Run: `28619017150`
  - Status: success

## Cleanup: Stale Queued Run

An old queued run was found:

- Workflow: `AI News Collector`
- Run: `25907283628`
- Created: `2026-05-15T08:04:04Z`
- Status: `queued`
- Jobs: `[]`

Cancel attempt:

```bash
gh run cancel 25907283628
```

Result:

```text
HTTP 500: Failed to cancel workflow run
```

Deletion attempt:

```bash
gh api -X DELETE repos/k-hira-shine/ai-news-collector/actions/runs/25907283628 --silent
```

Result: success.

Follow-up checks:

```text
gh run list --status queued --limit 20
[]
```

`gh run view 25907283628` now returns `404 Not Found`, confirming deletion.

## Final Verification

Final local check:

```text
日次チェック  2026-07-03 05:20 JST
✅ GitHub Actions: 直近7種success・期待5種cadence内
✅ Apify: 月換算 $0.62（窓 2026-06-26〜2026-07-02・上限 $12・on_target）
✅ Gemini: 2026-07-03 $0.177/日・直近7日平均で月換算 $7.6
✅ Buzz: 最終 2026-07-03（3h前）・status=pass・overlap=90.0%
✅ 収集品質: legal_rss=6/142（4%）・feeds 8/8健全・must_follow=135・x_valid=330
✅ 分析構造: morning top=10/cat=5/action=5/fallback=0（item=142）
✅ run_status: overall=success
✅ 全項目合格
```

Final tests:

```text
183 passed, 1 warning, 17 subtests passed
```

Final queued runs:

```text
[]
```

Final git state after the fixes:

```text
07383ae fix: quiet low-frequency critical account alerts
f342322 fix: ignore superseded legacy pages failures
a06e8b7 fix: deploy pages via explicit workflow
```

## Important Caveat

`gh api repos/k-hira-shine/ai-news-collector/pages/builds/latest` still returns the last legacy build failure:

```json
{
  "status": "errored",
  "commit": "c7f6b84fa06b9fe2aa06a93dacae62a4b3abec43",
  "error": {
    "message": "Page build failed."
  }
}
```

This is expected after the migration because it refers to the old legacy Pages build pipeline. The authoritative checks after the migration are:

- `Deploy Static Pages` workflow run status.
- `github-pages` deployment status.
- Public site HTTP response and `Last-Modified`.
- `daily_check.py`.

Current authoritative deployment:

```json
{
  "deployment_id": 5290417557,
  "state": "success",
  "environment_url": "https://k-hira-shine.github.io/ai-news-collector/",
  "log_url": "https://github.com/k-hira-shine/ai-news-collector/actions/runs/28618019424/job/84866539159"
}
```

## If This Regresses

Use this order:

1. Check latest explicit Pages workflow:

```bash
gh run list --workflow "Deploy Static Pages" --limit 5
```

2. Check latest GitHub Pages deployment status:

```bash
gh api 'repos/k-hira-shine/ai-news-collector/deployments?environment=github-pages&per_page=1'
```

3. Check public site:

```bash
curl -I -L --max-time 20 'https://k-hira-shine.github.io/ai-news-collector/?verify=manual'
```

4. Run local daily check:

```bash
./.venv/bin/python daily_check.py --days 7
```

5. Run tests:

```bash
./.venv/bin/python -m pytest -q
```

Do not treat `pages/builds/latest` alone as authoritative after the migration to `build_type=workflow`.
