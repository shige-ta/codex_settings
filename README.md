# Codex CLI: Rapid Codebase Scanning (Windows)

A practical workflow for loading and scanning large codebases fast on Windows by focusing only on relevant lines. Optimized for PowerShell + ripgrep and mindful of typical terminal output limits (~10KB / ~256 lines per command).

## Table of Contents
- Why This Exists
- Quick Start
- Core Workflow
- Core Commands
- Chunk Reading Patterns
- Troubleshooting
- For Agents
- 日本語クイックガイド（要点）
- Contributing

## Why This Exists
- Terminal limits: Avoid slow, noisy whole-file dumps that get truncated.
- Focused scans: Find only the lines you need, then read small chunks.
- Windows-first: Examples use PowerShell-safe quoting and paths.

## Quick Start
- Requirements: PowerShell 5.1+ (or 7+), ripgrep `rg` recommended.
  - Install: `winget install BurntSushi.ripgrep` or `choco install ripgrep`
- Scope the repo:
  ```powershell
  rg --files -g '!node_modules' -g '!.git' -g '!dist' -g '!build'
  ```
- Search multiple targets:
  ```powershell
  rg -n -S -e 'useEffect\(' -e 'export default function' -e 'ApiClient' -g '!node_modules' -g '!android'
  ```
- Read only chunks:
  ```powershell
  Get-Content -LiteralPath 'path\to\file.tsx' -TotalCount 200   # head
  Get-Content -LiteralPath 'path\to\file.tsx' -Tail 200         # tail
  # Around line N (~200 lines window)
  Get-Content -LiteralPath 'file.tsx' -TotalCount (N+100) | Select-Object -Last 200
  ```

## Core Workflow
- List files: Prefer `rg --files` with ignores to reduce noise.
- Narrow with searches: Use `rg -n` and multiple `-e` patterns (single quotes).
- Read small windows: Head/tail or windows around matched lines.
- Iterate by theme: Routing, API, storage, tests—avoid reading everything.

## Core Commands
- File list (exclude heavy dirs):
  ```powershell
  rg --files -g '!node_modules' -g '!android' -g '!.git' -g '!dist' -g '!build'
  ```
- By extension:
  ```powershell
  rg --files -g '*.{ts,tsx,js,jsx}' -g '!node_modules' -g '!android'
  ```
- TODO / FIXME:
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
  Get-Content -LiteralPath 'path\to\file.tsx' -TotalCount 200
  ```
- Tail 200 lines:
  ```powershell
  Get-Content -LiteralPath 'path\to\file.tsx' -Tail 200
  ```
- Around line N (~200 lines window):
  ```powershell
  Get-Content -LiteralPath 'file.tsx' -TotalCount (N+100) | Select-Object -Last 200
  ```
- Keep output readable:
  ```powershell
  rg -n -C 2 -m 200 --max-columns 200 --color=never -e 'ApiClient' -g '!node_modules'
  ```

## Troubleshooting
- rg: “No files were searched” → `-g` filters likely excluded everything. Try `-uu` or relax patterns; use `rg --debug` to inspect.
- Regex interpreted by PowerShell → Use single quotes, e.g. `-e 'password\b'`.
- pwsh not found → Use `powershell` or install PowerShell 7.

## For Agents
See AGENTS.md for operator guidance. Highlights:
- Positive guidance: Write instructions as “Do X” (not “Don’t Y”).
- Roles & collaboration: Branch `agent/<role>/<topic>`, lock via `ops/locks/`, commit `type(scope): summary [agent:@name]`.
- Style & token budgets: ≤6 lines/section, ≤120 chars/line, code ≤10 lines; provide copy‑ready commands.
- RAG hooks & summaries: Use `summaries/` (overview/routes/api/storage) for lightweight retrieval.
- Naming & structure: `kebab-case` dirs, `PascalCase` types, single‑responsibility files (~500 LOC) with a 5‑line overview.

## 日本語クイックガイド（要点）
- 目的: 出力上限（約10KB/最大256行）を意識し、必要な行だけを抽出。
- 基本フロー: 生成物を除外 → `rg -n` で複数キーワード絞り込み → `Get-Content` で必要範囲のみ読む。
- 引用/エスケープ: PowerShell は基本 `'...'`。`rg` の正規表現は `useEffect\(` のようにエスケープ。パスは `-LiteralPath`。
- 出力制御: `-C`（前後文脈）、`-m`（件数上限）、`--max-columns`、`--color=never` を併用。
- 並列/バッチ: PS7 は `-Parallel`、5.1 はジョブで代替。対象は `files.txt` に分割。
- ログ/再開: `ok.txt`/`fail.txt` を更新し差分再実行。ログは `logs\run-<timestamp>.log` に保存。

## Contributing
Small fixes are welcome. Keep examples concise and Windows‑friendly; prefer patterns that stay under typical output limits.

