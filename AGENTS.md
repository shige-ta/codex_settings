# AGENTS.md

> **Motto:** *Small, clear, safe steps — always grounded in real docs.*  
> **目的:** 巨大リポを「速く・点で読む」。優先度・再現性・安全性を担保し、**Python=uv / Node=pnpm / Chrome拡張(MV3)** を標準化。Windows/PowerShell 前提。

---

## Principles（原則）
- 変更は**小さく/明確/可逆**に保つ（small, clear, safe, reversible）。
- 賢さより**堅実さ**、複雑さより**単純さ**（clarity over cleverness; simplicity over complexity）。
- **不要な依存は増やさない**。必要なときだけ追加し、可能なら削除する。

---

## Knowledge & Libraries（MCP/hooks）
- コーディング前に **`context7` (MCP server)** を使って**最新ドキュメントを取得**する。  
- 外部ライブラリは **`resolve_library_id` → `get_library_docs`** の順で API を確認。  
- 不確実なら**一旦停止して確認を取る**（pause & request clarification）。

> これらのルールは uv/pnpm ベースの「再現性の高い実行」とセットで運用する。

---

## 成果物（Outcome）
- **情報の優先度**に沿って読む：**Lock/CI → 仕様 → ルール → 本書(AGENTS.md) → README → コード → 履歴**。
- **Python は `uv`、Node は `pnpm` で完全再現**（速い/汚染ゼロ/ロック厳守）。
- **静的検査・型・テスト**をワンコマンドで回し、**秘匿情報の混入**を検知。
- **並列・バッチ**で広く薄く当て、**200行単位**で深掘り。
- **Chrome 拡張（MV3）**は Vite + React + TS + **@crxjs/vite-plugin** で即走れる。
- **サマリ自動化**（関数一覧 → md 要約）で「読まない前提」を徹底。

---

## 情報の優先度（Information Hierarchy）
0. **Lock/CI**（`uv.lock`, `.pre-commit-config.yaml`, `pyproject.toml`, `pnpm-lock.yaml`, CI 定義）  
1. **プロダクト仕様 / チケット**（PRD, Issue tracker）  
2. **プロジェクト規約 / Docs**（CONTRIBUTING, ADR, ARCH）  
3. **AGENTS.md（本書）**  
4. **README.md（ユーザ向け導入）**  
5. **コードコメント / Docstring / JSDoc**  
6. **コミット履歴（必要時のみ局所）**  

---

## Workflow（ワークフロー）
1) **Plan**: 主要変更前に短い計画を共有。小さなレビュー可能な差分を好む。  
2) **Read**: 変更前に**関連ファイルを洗い出し読み切る**。  
3) **Verify**: 外部API/前提を**ドキュメントで再確認**。編集後は**構文/インデント**再チェック。  
4) **Implement**: スコープは**タイト**に。**単機能/疎結合/小さなファイル**で書く。  
5) **Test & Docs**: **テストとドキュメントを同時に**更新。ビジネスロジックのアサーションを最新に。  
6) **Reflect**: 根本原因を直し、回帰防止のための**仕組み**を検討。

**Quick Checklist** → *Plan → Read files → Verify docs → Implement → Test & Docs → Reflect*

---

## クイックスタート（3ステップ原則）
1) **候補抽出**：`rg` と `fd` で対象を点で洗い出す  
2) **薄く当てる**：関数宣言・Docstring/JSDoc・ヘッダ200行だけ読む  
3) **検証する**：`uv run` / `pnpm` でテスト・静的検査を即実行

```powershell
# 1) 候補抽出
fd -e py -E .venv -E node_modules src tests | tee pyfiles.txt
fd -e ts -e tsx -e js -e jsx -E node_modules -E dist -E build src web extension | tee jsfiles.txt

# 2) 構造の鳥瞰（関数/クラス・公開API）
rg -n -S -e '^(def|class)\s' -g '!node_modules' -g '!.venv' src tests
rg -n -S -e '^(export\s+)?(function|class)\s' -g '!node_modules' src web extension

# 3) 即時検証
uv run ruff check .; uv run pytest -q; uv run mypy src
pnpm -w -r lint && pnpm -w -r typecheck && pnpm -w -r test
```

---

## Code Style & Limits（コード規約）
- **Files ≤ 300 LOC**。ファイルは**単一目的**に分割。  
- **ファイル冒頭にヘッダ（5行）必須**：*Purpose / Inputs / Outputs / Side-effects / Owner*。  
- **コメントは多めに**。背景・前提・トレードオフ・非自明ロジックを明示。  
- **設定は `config/` に集中**。コード/テストでの**マジックナンバー禁止**。依存注入時は**デフォルトをconfigから**。  
- **Simplicity**: 要求**通り**に実装。余計な機能は入れない（no extra features）。  
- **Docstring/JSDoc 必須**（Public API）。副作用/例外も1行で要約。  
- **整形/品質**: Python=ruff-format, TS/JS=Prettier+ESLint、型チェックは `mypy`/`tsc --noEmit`。

---

## Collaboration & Accountability（連携/責任）
- **曖昧/セキュリティ/UX・API契約に関わる変更**は**必ずエスカレーション**。  
- プラン/コード/修正に**不安がある時は申告**（confidence < 80% でヘルプ要請）。  
- スコアリング：**壊れる変更 -4pt**、**成功 +1pt**、**正直な不確実性の申告 0pt**。  
  → スピードより**正確さを重視**（wrong change のコストは “小勝ち” より重い）。

---

## Python (uv) – Reproducible & Fast
### インストール
```powershell
winget install Astral.UV
# または
irm https://astral.sh/uv/install.ps1 | iex
```
### 環境作成と同期
```powershell
uv venv .venv
uv sync --frozen             # uv.lock があれば lock に忠実
.\.venv\Scripts\python.exe -V

# lock がない初回
uv pip install -r requirements.txt
uv lock
uv sync
```
### 実行/開発ツール
```powershell
uv run python -m pytest -q
uv run ruff check .
uv run mypy src/
uv add --dev ruff mypy pytest pytest-cov pre-commit
pre-commit install
uv run pre-commit run -a
```
**Tips**: `UV_LINK_MODE=copy`、`uv cache dir` を CI キャッシュ・AV 除外に。

---

## Node (pnpm) – Fast, Reproducible, Workspace-first
### 準備
```powershell
corepack enable
corepack prepare pnpm@latest --activate
pnpm -v
```
### インストール/実行
```powershell
pnpm -w install --frozen-lockfile
pnpm -w -r lint
pnpm -w -r typecheck
pnpm -w -r test --if-present
pnpm -w -r build --if-present
```
**設計**: `.npmrc` に `node-linker=isolated`。`pnpmfile.cjs` で暫定ピン留め。lock差分はPRレビュー必須。

---

## Chrome Extension（MV3） – Vite + React + TS + @crxjs/vite-plugin
### ひな形
```powershell
pnpm dlx create-vite@latest . --template react-ts
pnpm add -D @crxjs/vite-plugin vite tsx zod eslint prettier typescript @types/chrome
pnpm add react react-dom
```
### `vite.config.ts`
```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { crx } from '@crxjs/vite-plugin'
import manifest from './src/manifest'
export default defineConfig({ plugins: [react(), crx({ manifest })], build: { sourcemap: true, outDir: 'dist' } })
```
### `src/manifest.ts`
```ts
import { ManifestV3Export } from '@crxjs/vite-plugin'
const manifest: ManifestV3Export = {
  manifest_version: 3,
  name: 'My Extension',
  version: '0.1.0',
  action: { default_popup: 'index.html', default_title: 'MyExt' },
  options_page: 'options.html',
  background: { service_worker: 'src/background.ts' },
  permissions: ['storage', 'activeTab', 'scripting'],
  host_permissions: ['https://*/*', 'http://*/*'],
  content_scripts: [{ matches: ['https://*/*', 'http://*/*'], js: ['src/content.ts'] }]
}
export default manifest
```
### スクリプト例
```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "zip": "7z a -tzip dist.zip .\\dist\\*",
    "lint": "eslint . --ext .ts,.tsx --max-warnings=0",
    "format": "prettier -w .",
    "typecheck": "tsc --noEmit"
  }
}
```
**審査Tips**: host_permissions最小化、秘匿情報を埋め込まない、CSP を理解（eval不可/外部script制限）。

---

## コア・コマンド（Core Commands）
### A. リポ鳥瞰・入口
```powershell
fd -H -g 'pyproject.toml' -g 'uv.lock' -g '.pre-commit-config.yaml' -g '.ruff*' -g 'mypy*' -g 'pytest.ini' `
      -g 'pnpm-lock.yaml' -g 'package.json' -g 'pnpm-workspace.yaml'

rg -n -S -e 'ADR|Architecture|Design|API Spec|OpenAPI|CONTRIBUTING|CODEOWNERS' docs .
rg -n -S -e '^if __name__ == "__main__"' -g '!node_modules' -g '!.venv' src
rg -n -S -e 'chrome\.runtime|manifest_version|service_worker' extension src web
```
### B. 構造（関数/クラス/Docstring/JSDoc）
```powershell
rg -n -S -e '^class [A-Z][A-Za-z0-9_]+' -e '^def [a-zA-Z_]\w+\(' src
rg -n -P '^\s*def\s+\w+\(.*\):\s*(?!\n\s+""")' src   # Docstring 欠落
rg -n -S -e '^(export\s+)?(function|class)\s' -e '/\*\*' src web extension
```
### C. 危険文字列・秘匿情報スキャン
```powershell
rg -n -S -e 'AKIA[0-9A-Z]{16}' `
          -e '-----BEGIN (RSA|EC) PRIVATE KEY-----' `
          -e 'password\s*=' -g '!node_modules' -g '!.venv' -g '!dist' -g '!build' .
```
### D. 変更点の最小読解
```powershell
git diff --name-only origin/main...HEAD | tee changed.txt
Get-Content changed.txt | ForEach-Object {
  $f = $_
  if ($f -match '\.py$') { rg -n --max-columns 200 --context 3 -e 'def |class ' $f }
  elseif ($f -match '\.(ts|tsx|js|jsx)$') { rg -n --max-columns 200 --context 3 -e 'function |class |export ' $f }
}
```

---

## RAG Hooks & サマリ自動化
```powershell
rg -n -S '^(def|class)\s' src > summaries\py-symbols.txt
rg -n -S '^(export\s+)?(function|class)\s' extension src web > summaries\ts-symbols.txt
# 上位 N ファイルの先頭200行 + 公開関数近傍を抽出→ LLM で 1-3行/関数に短文化
```

---

## Troubleshooting（Windows/uv/pnpm）
- 長いパス: `git config core.longpaths true`  
- 改行混在: `.gitattributes` に `* text=auto eol=lf`、`.editorconfig` で統一  
- UTF-8 健診:
  ```powershell
  uv run python - <<<'print("✓ UTF-8")'
  ```
- AV 干渉: `.venv`, `uv cache dir`, `node_modules/.pnpm` を除外  
- pnpm shim不整合: `corepack prepare pnpm@x.y.z --activate`  
- Proxy/SSL: `UV_HTTP_TIMEOUT`, `REQUESTS_CA_BUNDLE`, `NODE_EXTRA_CA_CERTS`

---

## 付録 A: CI（pre-commit + Node）
`.pre-commit-config.yaml`
```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.11.2
    hooks:
      - id: mypy
        additional_dependencies: [types-requests]
  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v4.0.0-alpha.8
    hooks:
      - id: prettier
        additional_dependencies: [prettier]
```
**Node CI（GitHub Actions例）**
```yaml
name: node-ci
on:
  pull_request:
    paths: ["**/*.ts", "**/*.tsx", "package.json", "pnpm-lock.yaml", "pnpm-workspace.yaml"]
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "lts/*"
          cache: "pnpm"
      - run: corepack enable
      - run: corepack prepare pnpm@latest --activate
      - run: pnpm -v
      - run: pnpm -w install --frozen-lockfile
      - run: pnpm -w -r lint
      - run: pnpm -w -r typecheck
      - run: pnpm -w -r test --if-present
      - run: pnpm -w -r build --if-present
```

---

## 付録 B: `pyproject.toml` 抜粋
```toml
[tool.ruff]
line-length = 120
target-version = "py311"
lint.select = ["E","F","I","UP","B","SIM","PYI","DOC"]
lint.ignore = ["E501"]

[tool.mypy]
python_version = "3.11"
ignore_missing_imports = true
warn_unused_ignores = true

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q"
```

---

## 付録 C: `.npmrc` / `pnpmfile.cjs`
`.npmrc`
```
node-linker=isolated
strict-peer-dependencies=false
```
`pnpmfile.cjs`
```js
module.exports = {
  hooks: {
    readPackage(pkg) {
      // 例: 一時的なピン留め/置換をここで集中管理
      return pkg
    }
  }
}
```

---

## 付録 D: PR テンプレ（`PULL_REQUEST_TEMPLATE.md`）
```md
## 目的 / 背景
-

## 変更点（点で書く）
-

## 検証
- [ ] context7 で仕様/ライブラリを取得・確認
- [ ] uv sync --frozen / Windows
- [ ] uv run ruff / mypy / pytest
- [ ] pnpm -w install --frozen-lockfile
- [ ] pnpm -w -r lint / typecheck / test / build
- [ ] 秘匿情報スキャン（rg プリセット実行）

## 影響範囲 / ロールバック
-

## 追加メモ
-
```

---

## I/O Fast Path & Permissions（ファイル入出力を**短縮**し、権限問題で詰まらないために）

> 目的: **読む量を最小化**し、**書き込みは安全に一発**で終える。権限プロンプトや遅延の要因を**構造で潰す**。

### 0) 原則（Policy）
- **Least-Privilege**: 管理者権限は原則使わない。必要時は *作業の最小単位* に限定して昇格（UAC）。
- **Write-Redirect**: 書き込みは既定で **`%LOCALAPPDATA%\workspace\<repo>`** へ。保護ディレクトリ直書き禁止。
- **Atomic Writes**: すべての生成物は `*.tmp` に書いてから **原子的 rename**（PowerShell: `Move-Item -Force`）。
- **Allowlist I/O**: 触るパスは **明示リスト**で制御（`io-allowlist.txt`）。pre-commit で逸脱を**Fail**。

### 1) クローン/チェックアウトを軽量化（読む対象自体を減らす）
```powershell
# 最小ダウンロード（履歴/大容量を除外）
git clone --filter=blob:none --sparse <repo-url>
cd <repo>; git sparse-checkout set src docs extension

# LFS/大容量は必要時のみ
git lfs install
git lfs fetch --include="docs/**" --exclude="*"
```

### 2) 読み込みの高速化（Read Fast Path）
```powershell
# サイズ制限・除外で rg を最速に
rg -n -S --max-filesize 1M -g '!node_modules' -g '!.venv' -g '!dist' -g '!build' -e '^(def|class)\s' src

# 大容量は先頭200行 or 末尾200行だけ確認
Get-Content -TotalCount 200 .\path\large.py
Get-Content .\path\large.log -Tail 200
```

**Indexing**
```powershell
# ctags で記号索引 → fzf でジャンプ（読み飛ばしを増やす）
choco install universal-ctags -y
ctags -R -f .tags src
rg -n -S 'mySymbol' | fzf
```

### 3) 書き込みの安全化（Safe Write Path）
```powershell
# 出力先はユーザ領域に固定（速度/権限/AV衝突の回避）
$OUT = "$env:LOCALAPPDATA\workspace\myproj"
New-Item -ItemType Directory -Force $OUT | Out-Null

# 原子的書き込み
$TMP = Join-Path $OUT "AGENTS.md.tmp"
"content" | Out-File -FilePath $TMP -Encoding UTF8 -NoNewline
Move-Item $TMP (Join-Path $OUT "AGENTS.md") -Force
```

**バックアップ/ロールバック**
```powershell
Copy-Item "$OUT\AGENTS.md" "$OUT\_backup\AGENTS.md.$(Get-Date -Format yyyyMMdd-HHmmss)"
```

### 4) 権限で詰まるとき（Windows）
- **長いパス**: `git config --global core.longpaths true`
- **AV（Defender等）で I/O 遅延**: `.%LOCALAPPDATA%\workspace`, `.venv`, `uv cache dir`, `node_modules/.pnpm` を**除外**。
- **OneDrive/同期FS**: 大量 I/O は**ローカル固定**（同期配下に置かない）。
- **UNC/共有**: ネットワーク遅延はローカル staging に書いてから**まとめて移動**。

### 5) キャッシュを**事前温め**（Pre-warm）
```powershell
# Python
uv cache dir
uv sync --frozen

# Node
corepack enable; corepack prepare pnpm@latest --activate
pnpm -w install --frozen-lockfile
```

### 6) Chrome 拡張（MV3）の I/O
- **`chrome.storage.local`** を基本にし、**PII/秘匿は保存しない**。
- **`downloads`** API は **`optional_permissions`** で**実行時要求**し、ユーザ同意後に実行。
- **`host_permissions`** は**最小権限**。`file:///*` を必要とする場合は**開発時のみ**（`Allow access to file URLs`）。

**manifest 例（抜粋）**
```ts
const manifest = {
  optional_permissions: ["downloads", "storage"],
  host_permissions: ["https://*/*"],
  // ※ file URL は開発時のみ。公開版では避ける。
}
```

### 7) pre-commit に I/O ガードを追加
`.pre-commit-config.yaml` へ以下ルールを追加：
```yaml
- repo: local
  hooks:
    - id: io-allowlist-guard
      name: Block writes outside allowlist
      entry: pwsh -NoProfile -Command ./scripts/io-guard.ps1
      language: system
      stages: [commit, push]
```
`scripts/io-guard.ps1`（雛形）:
```powershell
$allow = Get-Content io-allowlist.txt
$changed = git diff --name-only --cached
$violations = $changed | Where-Object { $p = $_; -not ($allow | ForEach-Object { $p -like $_ }) }
if ($violations) {
  Write-Error "I/O guard: out-of-allowlist changes detected:`n$($violations -join "`n")"
  exit 1
}
```

### 8) エラー時の**指数バックオフ + 再試行**（共通）
```powershell
function Invoke-WithRetry {
  param([scriptblock]$Action, [int]$Max=5)
  for ($i=0; $i -lt $Max; $i++) {
    try { & $Action; return } catch {
      Start-Sleep -Seconds ([math]::Pow(2, $i))
    }
  }
  throw "Max retries exceeded"
}
```

**TL;DR**: *読む対象を物理的に減らす（sparse/サイズ制限/索引）→ ローカル書き込みに統一 → 原子的に書く → 事前にキャッシュを温める → 権限とAVで詰まる要因を構造で潰す。*

