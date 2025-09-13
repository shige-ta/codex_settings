# Codex CLI: Rapid Codebase Scanning (Windows)

A practical workflow for loading and scanning large codebases fast on Windows by focusing only on relevant lines. It is optimized for PowerShell and ripgrep and designed to keep every command under typical terminal output limits (~10KB / ~256 lines).

## Why This Exists
- **Terminal limits:** Avoid slow, noisy whole‑file dumps that get truncated.
- **Focused scans:** Find only the lines you need, then read small chunks around them.
- **Windows-first:** All examples use PowerShell-friendly syntax.

## Quick Start
- **Requirements:** PowerShell 5.1+ (or PowerShell 7+), optional `rg` (ripgrep) for fast search.
- **Scope the repo:**
  ```powershell
  rg --files -g "!node_modules" -g "!.git" -g "!dist" -g "!build"
  ```
- **Search by multiple targets:**
  ```powershell
  rg -n -S -e "useEffect\(" -e "export default function" -e "ApiClient" -g "!node_modules" -g "!android"
  ```
- **Read only chunks:**
  ```powershell
  Get-Content path\to\file.tsx -TotalCount 200   # head
  Get-Content path\to\file.tsx -Tail 200         # tail
  # Around line N (~200 lines window)
  Get-Content file.tsx -TotalCount (N+100) | Select-Object -Last 200
  ```

## Core Workflow
- **List files:** Prefer `rg --files` with ignores to reduce noise.
- **Narrow with searches:** Use `rg -n` and multiple `-e` patterns.
- **Read small windows:** Head/tail or windows around matched lines.
- **Iterate by theme:** Routing, API, storage, tests—never read everything.

## Core Commands
- **File list (exclude heavy dirs):**
  ```powershell
  rg --files -g "!node_modules" -g "!android" -g "!.git" -g "!dist" -g "!build"
  ```
- **By extension:**
  ```powershell
  rg --files -g "*.{ts,tsx,js,jsx}" -g "!node_modules" -g "!android"
  ```
- **TODO / FIXME:**
  ```powershell
  rg -n -e "TODO|FIXME" -g "!node_modules" -g "!android"
  ```
- **Type/exports overview:**
  ```powershell
  rg -n -e "^export (interface|type|class) " -e "^export default function" -g "!node_modules" -g "!android"
  ```
- **Largest files (avoid expensive reads):**
  ```powershell
  Get-ChildItem -Recurse -File | Sort-Object Length -Descending | Select-Object -First 50 Length,FullName
  ```

## Detect Encoding / Mojibake Hints
- **Non‑ASCII scan (PCRE2):**
  ```powershell
  rg -n -P "[^\x09\x0A\x0D\x20-\x7E]" -g "!node_modules" -g "!android"
  ```
- **Specific odd sequence:**
  ```powershell
  rg -n -S "Switch,\s+const" -g "!node_modules" -g "!android"
  ```

## Expo / React Native Quick Filters
- **Routes / screens:**
  ```powershell
  rg -n -e "useRouter\(|Stack|Tabs|_layout\\.tsx" app -g "!node_modules"
  ```
- **Storage / API usage:**
  ```powershell
  rg -n -e "AsyncStorage" -e "ApiClient" -g "!node_modules" -g "!android"
  ```

## Example Snippets
- **Find a suspicious import/syntax break:**
  ```powershell
  rg -n -S "Switch,\s+const" -g "!node_modules" -g "!android"
  ```
- **Inspect a file head:**
  ```powershell
  Get-Content "app\(tabs)\index.tsx" -TotalCount 120
  ```
- **Search duplicate function definitions:**
  ```powershell
  rg -n "^export const saveAllCampaigns" utils
  ```

## Tips
- Keep outputs under ~200–250 lines to avoid truncation.
- Always exclude generated/large directories.
- Prefer many small, focused reads over one large dump.
- If Windows env‑var syntax breaks in scripts, run commands directly instead of npm scripts.

## Batch / Parallel Processing (PowerShell)
- **List targets (exclude generated artifacts):**
  ```powershell
  rg --files -g "*.{ts,tsx,js,jsx}" -g "!node_modules" -g "!android" -g "!dist" -g "!build" > files.txt
  ```
- **Batch 1 (first 100 files):**
  ```powershell
  Get-Content files.txt | Select-Object -First 100 | ForEach-Object { $_ }
  ```
- **Batch 2 (next 100 files):**
  ```powershell
  Get-Content files.txt | Select-Object -Skip 100 -First 100 | ForEach-Object { $_ }
  ```
- **Parallel (PowerShell 7+):**
  ```powershell
  Get-Content files.txt | Select-Object -First 100 |
    ForEach-Object -Parallel {
      $path = $_
      Get-Content -TotalCount 80 -- $path | Out-Null
    } -ThrottleLimit 6
  ```
- **Jobs alternative (Windows PowerShell 5.1):**
  ```powershell
  $throttle = 6
  $jobs = @()
  Get-Content files.txt | Select-Object -First 100 | ForEach-Object {
    while ($jobs.Count -ge $throttle) {
      $done = Wait-Job -Any $jobs
      $jobs = $jobs | Where-Object { $_.Id -ne $done.Id }
      Receive-Job $done | Out-Null
      Remove-Job $done
    }
    $jobs += Start-Job -ScriptBlock {
      param($p)
      Get-Content -TotalCount 80 -- $p | Out-Null
    } -ArgumentList $_
  }
  Wait-Job $jobs | Receive-Job | Out-Null; Remove-Job $jobs
  ```

## Timeouts / Resume / Logging
- **Timeouts:** Prefer tool-native `--timeout` flags or wrap jobs with a monitoring timer.
- **Resume:** Maintain `ok.txt` and `fail.txt`; rerun on the diff using `--resume` or `-notin (Get-Content ok.txt)` patterns.
- **Logging:** Save stdout/stderr to timestamped logs, e.g.:
  ```powershell
  $stamp = Get-Date -Format yyyyMMdd-HHmmss
  some-command 2>&1 | Tee-Object -FilePath "logs\run-$stamp.log"
  ```

---

## 日本語クイックガイド (要点)
- **目的:** 大規模リポジトリを高速に把握するため、必要な行だけを素早く抽出します。
- **基本方針:** 生成物を除外してファイル一覧→複数キーワードで絞り込み→必要範囲だけ読む→テーマ単位で反復。
- **並列化:** PowerShell 7+ は `ForEach-Object -Parallel`、5.1 はジョブで代替し、`-ThrottleLimit` 等で同時実行数を調整。
- **バッチ処理:** 対象を `files.txt` に出力し、100件単位などに分割して処理。
- **再開可能性:** 成功/失敗リスト（`ok.txt`/`fail.txt`）を更新し差分再実行。ログは `logs\run-<timestamp>.log` に保存。

> これらのコマンド断片はサンプルです。実案件では拡張子・除外パターン・閾値（行数/同時数）をリポジトリ規模に応じて調整してください。

## FAQ
- **ripgrep が無い場合?** `Get-ChildItem` と `Select-String` で代替できますが、速度は低下します。
- **macOS/Linux でも使える?** ほぼ同じ概念で動作しますが、コマンド・パス区切り・エンコーディングが異なります。

## Contributing
Small fixes and improvements to commands and wording are welcome. Keep examples concise and Windows-friendly.

