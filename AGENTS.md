# Codex CLI: Rapid Codebase Scanning (Windows)

Purpose: Load large repos fast by focusing only on relevant lines.
Effective output limit in this environment: about 10KB / up to ~256 lines per command. Prefer small, precise reads over whole‑file dumps.

Note: For a user‑facing summary of the workflow, see `README.md`. This document focuses on operator tips, pitfalls, and advanced usage on Windows/PowerShell.

## Table of Contents
- Requirements
- Workflow
- Core Commands
- Chunk Reading Patterns
- PowerShell Quoting Cheatsheet
- Detect Encoding / Mojibake
- Expo/React Native Quick Filters
- Example
- Tips
- Troubleshooting
- Batch / Parallel Processing
- Timeouts / Resume / Logging
- 日本語クイックガイド（要点）

## Requirements
- PowerShell 5.1 or PowerShell 7+ (recommended).
- ripgrep (`rg`) installed for fast search.
  - winget: `winget install BurntSushi.ripgrep`
  - Chocolatey: `choco install ripgrep`
  - If `rg` is unavailable, use `Select-String` as a slower fallback.

## Workflow
- List files with ignore rules to scope the repo.
- Narrow with `rg -n` using multiple `-e` patterns.
- Read small chunks only: head/tail or a window around known lines.
- Iterate by theme (routing, API, storage, tests) instead of reading everything.

## Core Commands
- File list (exclude heavy dirs):
  ```powershell
  rg --files -g '!node_modules' -g '!android' -g '!.git' -g '!dist' -g '!build'
  ```
- By extension:
  ```powershell
  rg --files -g '*.{ts,tsx,js,jsx}' -g '!node_modules' -g '!android'
  ```
- Multi-keyword search (case/regex literal):
  ```powershell
  rg -n -S -e 'useEffect\(' -e 'export default function' -e 'ApiClient' -g '!node_modules' -g '!android'
  ```
- TODO/FIXME:
  ```powershell
  rg -n -e 'TODO|FIXME' -g '!node_modules' -g '!android'
  ```
- Type/exports overview:
  ```powershell
  rg -n -S -e '^export (interface|type|class) ' -e '^export default function' -g '!node_modules' -g '!android'
  ```
- Largest files (avoid expensive reads):
  ```powershell
  Get-ChildItem -Recurse -File | Sort-Object Length -Descending | Select-Object -First 50 Length,FullName
  ```

## Chunk Reading Patterns
- Head 200 lines:
  ```powershell
  Get-Content 'path\to\file.tsx' -TotalCount 200
  ```
- Tail 200 lines:
  ```powershell
  Get-Content 'path\to\file.tsx' -Tail 200
  ```
- Around line N (~200 lines window):
  ```powershell
  Get-Content 'file.tsx' -TotalCount (N+100) | Select-Object -Last 200
  ```

## PowerShell Quoting Cheatsheet
- Prefer single quotes around regex strings: `'ApiClient'`, `'useEffect\('`, `'password\b'`.
- Use double quotes only if variable expansion is needed.
- For paths that might start with `-`, use `-LiteralPath` instead of relying on `--`.
  ```powershell
  Get-Content -LiteralPath $path -TotalCount 80 | Out-Null
  ```
- For external commands like `rg`, single quotes keep PCRE escapes intact on Windows.
- Keep outputs concise with context/limits:
  ```powershell
  rg -n -C 2 -m 200 --max-columns 200 --color=never -e 'ApiClient' -g '!node_modules'
  ```

## Detect Encoding / Mojibake
- Non‑ASCII scan (PCRE2):
  ```powershell
  rg -n -P '[^\x09\x0A\x0D\x20-\x7E]' -g '!node_modules' -g '!android'
  ```
- Specific odd sequence (useful for import/syntax breaks in RN projects):
  ```powershell
  rg -n -S 'Switch,\s+const' -g '!node_modules' -g '!android'
  ```

## Expo/React Native Quick Filters
- Routes/screens:
  ```powershell
  rg -n -e 'useRouter\(|Stack|Tabs|_layout\.tsx' app -g '!node_modules'
  ```
- Storage/API usage:
  ```powershell
  rg -n -e 'AsyncStorage' -e 'ApiClient' -g '!node_modules' -g '!android'
  ```

## Example
- Find the import/syntax break in a screen file:
  ```powershell
  rg -n -S 'Switch,\s+const' -g '!node_modules' -g '!android'
  ```
- Inspect the top of a candidate file:
  ```powershell
  Get-Content 'app\(tabs)\index.tsx' -TotalCount 120
  ```
- Search for duplicate function definitions in utilities:
  ```powershell
  rg -n '^export const saveAllCampaigns' utils
  ```

## Tips
- Keep outputs under ~200–250 lines to avoid truncation.
- Always exclude generated/large directories.
- Prefer many small, focused reads over one large dump.
- If Windows env‑var syntax breaks in scripts, run commands directly instead of npm scripts.

## Troubleshooting
- `pwsh` not found: use `powershell` or install PowerShell 7.
- `rg` returns “No files were searched”: a `-g` pattern likely excluded everything. Try `-uu` or remove over‑strict `-g` filters. Use `rg --debug` to see why.
- Regex looks “interpreted” by PowerShell (e.g., `password\b` error): quote with single quotes: `-e 'password\b'`.
- Encoding noise/mixed line endings: normalize files to UTF‑8 and prefer LF in git; let Git handle CRLF on checkout as needed.

## Batch / Parallel Processing
- List targets (exclude generated artifacts):
  ```powershell
  rg --files -g '*.{ts,tsx,js,jsx}' -g '!node_modules' -g '!android' -g '!dist' -g '!build' > files.txt
  ```
- Batch 1 (first 100 files):
  ```powershell
  Get-Content files.txt | Select-Object -First 100 | ForEach-Object { $_ }
  ```
- Batch 2 (next 100 files):
  ```powershell
  Get-Content files.txt | Select-Object -Skip 100 -First 100 | ForEach-Object { $_ }
  ```
- Parallel (PowerShell 7+):
  ```powershell
  Get-Content files.txt | Select-Object -First 100 |
    ForEach-Object -Parallel {
      $path = $_
      Get-Content -LiteralPath $path -TotalCount 80 | Out-Null
    } -ThrottleLimit 6
  ```
- Jobs alternative (Windows PowerShell 5.1):
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
      Get-Content -LiteralPath $p -TotalCount 80 | Out-Null
    } -ArgumentList $_
  }
  Wait-Job $jobs | Receive-Job | Out-Null; Remove-Job $jobs
  ```

## Timeouts / Resume / Logging
- Timeouts: prefer native `--timeout` where available; otherwise wrap jobs with a watchdog timer.
- Resume: maintain `ok.txt` and `fail.txt`; rerun on the diff via `-notin (Get-Content ok.txt)` patterns.
- Logging: save stdout/stderr with minimal color/noise for diffing.
  ```powershell
  $stamp = Get-Date -Format yyyyMMdd-HHmmss
  some-command --color=never 2>&1 | Tee-Object -FilePath "logs\run-$stamp.log"
  ```

## 日本語クイックガイド（要点）
- 目的: 大規模リポジトリを高速に把握するため、必要な行だけを抽出。
- 基本フロー: 生成物を除外→`rg -n` 複数キーワードで絞り込み→`Get-Content` で必要範囲だけ読む→テーマ単位で反復。
- 引用/エスケープ: PowerShell は基本 `'...'` を使用。パスは `-LiteralPath` を推奨。`rg` は `'useEffect\('` のようにエスケープ。
- 並列化: PowerShell 7+ は `ForEach-Object -Parallel`、5.1 はジョブで代替し `-ThrottleLimit` で同時数を制御。
- バッチ処理: 対象を `files.txt` に出力し、100件単位で処理。
- 再開/ログ: `ok.txt`/`fail.txt` を更新し差分再実行。ログは `logs\run-<timestamp>.log` に保存。

