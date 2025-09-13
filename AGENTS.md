# Codex CLI: Rapid Codebase Scanning (Windows)

Purpose: Load large repos fast by focusing only on relevant lines.
Hard limit: each command output is ~10KB/256 lines, so avoid whole‑file dumps.

## Workflow
- List files: use `rg --files` with ignores to scope the repo.
- Search targets: `rg -n` with multiple `-e` patterns to narrow to the lines you need.
- Read chunks only: use `Get-Content -TotalCount` (head) / `-Tail` (tail), or read a window around a known line.
- Iterate by theme (routing, API, storage, tests) instead of reading everything.

## Core Commands
- File list (exclude heavy dirs):
  - `rg --files -g "!node_modules" -g "!android" -g "!.git" -g "!dist" -g "!build"`
- By extension:
  - `rg --files -g "*.{ts,tsx,js,jsx}" -g "!node_modules" -g "!android"`
- Multi-keyword search (case/regex literal):
  - `rg -n -S -e "useEffect\(" -e "export default function" -e "ApiClient" -g "!node_modules" -g "!android"`
- TODO/FIXME:
  - `rg -n -e "TODO|FIXME" -g "!node_modules" -g "!android"`
- Type/exports overview:
  - `rg -n -e "^export (interface|type|class) " -e "^export default function" -g "!node_modules" -g "!android"`
- Largest files (avoid expensive reads):
  - `Get-ChildItem -Recurse -File | Sort-Object Length -Descending | Select-Object -First 50 Length,FullName`

## Chunk Reading Patterns
- Head 200 lines: `Get-Content "path\\to\\file.tsx" -TotalCount 200`
- Tail 200 lines: `Get-Content "path\\to\\file.tsx" -Tail 200`
- Around line N (~200 lines window):
  - `Get-Content "file.tsx" -TotalCount (N+100) | Select-Object -Last 200`

## Detect Encoding/Mojibake Hints
- Non-ASCII scan (PCRE2): `rg -n -P "[^\x09\x0A\x0D\x20-\x7E]" -g "!node_modules" -g "!android"`
- Specific odd sequence: `rg -n -S "Switch,\s+const" -g "!node_modules" -g "!android"`

## Expo/React Native Quick Filters
- Routes/screens: `rg -n -e "useRouter\(|Stack|Tabs|_layout\\.tsx" app -g "!node_modules"`
- Storage/API usage: `rg -n -e "AsyncStorage" -e "ApiClient" -g "!node_modules" -g "!android"`

## Example (poi_manage2)
- Find the import/syntax break: `rg -n -S "Switch,\s+const" -g "!node_modules" -g "!android"`
- Inspect the file head: `Get-Content "app\\(tabs)\\index.tsx" -TotalCount 120`
- Search for duplicate functions: `rg -n "^export const saveAllCampaigns" utils`

## Tips
- Keep outputs under 200–250 lines to avoid truncation.
- Always exclude generated/large dirs.
- Prefer many small, focused reads over one large dump.
- If Windows env-var syntax breaks in scripts, run commands directly instead of npm scripts.

## 複数ファイル読み込みの効率化（推奨ルール）
- 処理の並列化: 複数ファイルを一括処理する場合は `--concurrency` 等の並列オプションを活用し、同時に複数タスクを実行する。
- バッチ分割: 大量ファイルは最大100件/バッチを目安に分割し、必要に応じて更に細かくする。
- 対象の絞り込み: 拡張子やフォルダを明示的に限定してスコープを狭める（例: `*.{ts,tsx,js,jsx}` や特定ディレクトリのみ）。
- 中断/再開: 長時間処理に備えて `--timeout` や `--resume` を利用し、途中で落ちても再開できるようにする。
- ログ活用: 進捗と失敗ログを自動保存（タイムスタンプ付）し、次回以降のスキップ/再試行に再利用する。

### 例: 対象抽出 → 100件ごとのバッチ処理
```
# 対象をリストアップ（生成物は除外）
rg --files -g "*.{ts,tsx,js,jsx}" -g "!node_modules" -g "!android" -g "!dist" -g "!build" > files.txt

# 1バッチ目（先頭100件）
Get-Content files.txt | Select-Object -First 100 |
  ForEach-Object { $_ }

# 2バッチ目（次の100件）
Get-Content files.txt | Select-Object -Skip 100 -First 100 |
  ForEach-Object { $_ }
```

### 例: 並列実行の書き方（環境に応じて選択）
- PowerShell 7+（推奨）
```
Get-Content files.txt | Select-Object -First 100 |
  ForEach-Object -Parallel {
    # ここにファイル単位の処理を書く（例: 先頭だけ読む等）
    $path = $_
    Get-Content -TotalCount 80 -- $path | Out-Null
  } -ThrottleLimit 6  # ← 同時実行数（--concurrency に相当）
```

- Windows PowerShell 5.1（`-Parallel`非対応のためジョブで代替）
```
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

### タイムアウト/再開/ログの基本方針
- タイムアウト: 長い処理にはコマンド側の `--timeout` や、ジョブ実行に対する監視タイマーを設ける。
- 再開: 成功/失敗のファイル一覧（例: `ok.txt`/`fail.txt`）を都度更新し、`--resume` や `-notin (Get-Content ok.txt)` で差分実行。
- ログ: `Tee-Object` やリダイレクトで `logs\\run-$(Get-Date -f yyyyMMdd-HHmmss).log` に標準出力/エラーを保存。
