# Codex CLI: Rapid Codebase Scanning (Windows)

---
scope: repository root
for: AI agents and maintainers
not_for: end-user documentation
last_updated: 2025-09-13
maintainers: [@shige-ta]
---

Purpose: Load large repos fast by focusing only on relevant lines.
Effective output limit in this environment: ~10KB and up to ~256 lines per command. Prefer small, precise reads over whole‑file dumps.

Note: README.md is the user-facing quick guide. This AGENTS.md gives operators practical rules, collaboration patterns, and advanced tips for Windows/PowerShell.

## Table of Contents
- Front Matter
- Requirements
- Workflow
- Core Commands
- Chunk Reading Patterns
- PowerShell Quoting Cheatsheet
- Detect Encoding / Mojibake
- Expo/React Native Quick Filters
- Examples
- Tips
- Troubleshooting
- Batch / Parallel Processing
- Timeouts / Resume / Logging
- Positive Guidance Principles
- Information Hierarchy & Sources of Truth
- Roles & Collaboration
- Style & Token Budgets
- Naming & Structure
- RAG Hooks & Summaries
- Code Documentation for Agents
- Templates
- 日本語クイックガイド（要点）

## Requirements
- PowerShell 5.1 or PowerShell 7+ (recommended).
- ripgrep (`rg`) for fast search.
  - Install: `winget install BurntSushi.ripgrep` or `choco install ripgrep`
  - Fallback: `Select-String` (slower) if `rg` is unavailable.

## Workflow
- Scope files with ignore rules.
- Narrow via `rg -n` and multiple `-e` patterns.
- Read small chunks only (head/tail or windows around hits).
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

## PowerShell Quoting Cheatsheet
- Prefer single quotes for regex: `'ApiClient'`, `'useEffect\('`, `'password\b'`.
- Use double quotes only if variable expansion is needed.
- For paths that might start with `-`, use `-LiteralPath`.
  ```powershell
  Get-Content -LiteralPath $path -TotalCount 80 | Out-Null
  ```
- Keep outputs concise:
  ```powershell
  rg -n -C 2 -m 200 --max-columns 200 --color=never -e 'ApiClient' -g '!node_modules'
  ```

## Detect Encoding / Mojibake
- Non‑ASCII scan (PCRE2):
  ```powershell
  rg -n -P '[^\x09\x0A\x0D\x20-\x7E]' -g '!node_modules' -g '!android'
  ```
- Suspicious sequence (helps spot import breaks):
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

## Examples
- Find an import/syntax break in a screen file:
  ```powershell
  rg -n -S 'Switch,\s+const' -g '!node_modules' -g '!android'
  ```
- Inspect the top of a candidate file:
  ```powershell
  Get-Content -LiteralPath 'app\(tabs)\index.tsx' -TotalCount 120
  ```
- Search for duplicate function definitions:
  ```powershell
  rg -n '^export const saveAllCampaigns' utils
  ```

## Tips
- Keep outputs under ~200–250 lines to avoid truncation.
- Always exclude generated/large directories.
- Prefer many small, focused reads over one large dump.
- If Windows env‑var syntax breaks, run commands directly instead of npm scripts.

## Troubleshooting
- `pwsh` not found → use `powershell` or install PowerShell 7.
- `rg` says “No files were searched” → a `-g` pattern likely excluded everything. Try `-uu` or relax filters. Inspect with `rg --debug`.
- Regex “interpreted” by PowerShell (e.g., `password\b` error) → quote with single quotes: `-e 'password\b'`.
- Mixed encodings/line endings → normalize to UTF‑8; prefer LF in git; allow CRLF on checkout.

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
- Timeouts: prefer native `--timeout`; otherwise wrap jobs with a watchdog timer.
- Resume: keep `ok.txt` and `fail.txt`; rerun the diff via `-notin (Get-Content ok.txt)`.
- Logging: minimize color/noise for diffing.
  ```powershell
  $stamp = Get-Date -Format yyyyMMdd-HHmmss
  some-command --color=never 2>&1 | Tee-Object -FilePath "logs\run-$stamp.log"
  ```

## Positive Guidance Principles
- Prefer “Do X” over “Don’t Y”. State desired behaviors, not prohibitions.
- Lead with outcome and next step; defer background to links.
- Show copy‑ready commands; avoid ambiguous placeholders.
- Keep each instruction actionable within 3 steps.
- Favor deterministic flags and predictable output size.

## Information Hierarchy & Sources of Truth
Order of reference when resolving conflicts:
1) Source of truth (product specs/issue tracker)
2) Project rules/docs (this repo)
3) AGENTS.md (this file)
4) README.md (quick how‑to)
5) Code comments
6) Commit history

## Roles & Collaboration
- Role playbooks (≤5 lines each):
  - `jr-dev`: scan → minimal diff → patch proposal
  - `refactor`: clarify boundaries → rename safely → side‑effect checks
  - `review`: rules/safety/diff size/regression risk
- Branch naming: `agent/<role>/<topic>` (e.g., `agent/jr-dev/readme-toc`).
- Locking: create `ops/locks/<path>.lock` for long edits.
- Commit template: `type(scope): summary (#issue) [agent:@name]`.

## Style & Token Budgets
- Section ≤6 lines; line ≤120 chars; code block ≤10 lines.
- Bullets over prose; link deep explanations via anchors.
- Prefer minimal variables; avoid env‑specific branches inline.
- Start sections with “Outcome: …” when helpful (e.g., “Outcome: list 3 files + 5 hits”).

## Naming & Structure
- Directories: `docs/`, `summaries/`, `prompts/`, `workflows/`, `scripts/`, `ops/locks/`.
- Naming: folders `kebab-case`; TS types `PascalCase`; vars `camelCase`; constants `SCREAMING_SNAKE_CASE`.
- One responsibility per file; keep under ~500 LOC; add 5‑line module overview at top.

## RAG Hooks & Summaries
- Summaries directory:
  - `summaries/overview.md` (~200 words)
  - `summaries/routes.md`, `summaries/api.md`, `summaries/storage.md`
- Retrieval examples:
  ```powershell
  rg -n -S -e '^# (Rules|Playbooks|Decisions)' AGENTS.md
  rg -n -S -e 'Outcome:' summaries/*.md
  ```

## Code Documentation for Agents
- Top‑of‑file: 5‑line overview with purpose, inputs, outputs.
- Public functions: 1‑line doc comment with side‑effects and errors.
- High‑risk spots: add a short “Decision Record” with rationale + 1 link.

## Templates
- PR description:
  - Purpose → Scope → Non‑Goals → Validation → Risks → Rollback
- Change plan:
  - Context → Impacted areas → Phased rollout → Exit criteria → Metrics

## 日本語クイックガイド（要点）
- 目的: 出力上限（約10KB/最大256行）を意識し、必要な行だけを抽出。
- 指示法: 否定ではなく肯定的な行動指示で統一（Doベース）。
- 基本フロー: 生成物を除外 → `rg -n` で複数キーワード絞り込み → `Get-Content` で必要範囲のみ読む。
- マルチエージェント: 役割別プレイブック、`agent/<role>/<topic>` ブランチ、`ops/locks/` ロック。
- RAG/サマリー: `summaries/` に階層サマリーを配置し、まず `overview.md` から参照。
- 命名/構造: ディレクトリは `kebab-case`、型は `PascalCase`、1ファイル1責務（~500行）。
- トークン節約: 1セクション6行以内、コードは10行以内、コピー可能な最小コマンドを提示。

