# ハンズオンワークショップ — グループ別ダッシュボード生成

**対象**: 大手電力会社 IT 部門 若手向けグループワーク＆ハンズオン  
**テーマ**: Vibe Coding 体験 → スキル駆動 AI 開発の違いを体感する

---

## 概要

Claude Code のスキル（ガードレール）を使うと、AI が正確なレシピに従って SAP CAP + SAPUI5 のダッシュボードを生成できることを体験する。
まずガードレールなしの「素の Vibe Coding」で失敗を体験し、次にスキルを効かせた状態で同じお題を再実装して違いを感じ取る。

---

## 技術スタック

| カテゴリ | 技術 |
|---|---|
| バックエンド | SAP CAP (`@sap/cds` v9) + SQLite (`@cap-js/sqlite` v2) |
| フロントエンド | SAPUI5 (`sap.viz` チャート) |
| AI ツール | Claude Code (Cline) + スキル (`.cline/skills/`) |
| MCP サーバ | `@cap-js/mcp-server` / `@ui5/mcp-server` / `@modelcontextprotocol/server-puppeteer` |
| 実行環境 | Node.js v20 以上 |

---

## ワークショップの流れ

### Phase 1 — 素の Vibe Coding（ガードレールなし・体験）

1. グループでどんなダッシュボードが欲しいかを話し合う
2. `draft.md` にアイデアの種を書き出す
3. `/dashboard-requirements` スキルで `requirements.md`（要件定義書）を生成する
4. 生成された `requirements.md` を参照させて、**スキルなしのまま**実装を頼む

> 例: 「`requirements.md` を読んで、SAP CAP + SAPUI5 でダッシュボードを実装してください」

何が起こるか体験する：

- データが LLM の想像で構成される
- `sap.viz` の vizType / FeedItem uid が誤る
- OData パス・`manifest.json` の設定漏れで起動できない
- 高確率でエラーが出て「動かない」を体験

---

### Phase 2 — スキル × MCP（ガードレールあり・本番）

Phase 1 で作成した **`requirements.md`** をそのまま使う。今度はスキルを効かせて確実に生成する。

```
/utility-dashboard  →  /chrome-devtools-test
```

| ステップ | コマンド | 内容 |
|---|---|---|
| 1 | `/utility-dashboard` | `requirements.md` → CAP + UI5 ファイル一式を生成 |
| 2 | `/chrome-devtools-test` | ブラウザ起動・OData 疎通・チャート描画確認 + スクショ |

---

## 前提セットアップ（当日マシン・事前確認必須）

### 1. Node.js / CDS

```bash
node -v    # v20 以上推奨（v24 も可）
cds -v     # @sap/cds-dk がなければ: npm install -g @sap/cds-dk
```

### 2. Chromium（chrome-devtools MCP 用）

```bash
npx puppeteer browsers install chrome
```

> MCP が動いていないと `/chrome-devtools-test` のブラウザテスト工程が詰まる。当日前に疎通確認すること。

### 3. MCP サーバ（Cline に事前設定済み）

スキルは以下 3 つの MCP サーバを実装ルールの前提としている。  
Cline の UI から登録済みで、設定は `cline_mcp_settings.json` に保存されている（本プロジェクトに `.mcp.json` は置かない）。

| Cline 上のサーバ名 | パッケージ | 用途 |
|---|---|---|
| `cds-mcp` | `@cap-js/mcp-server` | CAP/CDS 層（schema / service / CSV 配備）の生成・検証 |
| `ui5-mcp-server` | `@ui5/mcp-server` | SAPUI5 層（manifest / view / controller / sap.viz）の生成・検証 |
| `chrome-devtools` | `@modelcontextprotocol/server-puppeteer` | ブラウザ起動・OData 疎通・チャート描画・スクショ |

### 4. 作業コピーの準備

グループごとに `handson/` を複製（または空ディレクトリで）作業する。

---

## グループへの案内（当日の流れ）

1. `handson/` を AI ツール（Claude Code / Cline）で開く
2. `templates/draft-template.md` を `draft.md` にコピーして穴埋めする（テーマ・対象・見たいこと・拠点・地図要否）
3. `/dashboard-requirements` と入力 → `requirements.md`（業務要件書）が生成される。内容を確認する
4. `/utility-dashboard` と入力 → Claude が実装プランを提示し、ファイル一式を生成する
5. 生成後にアプリを起動する

```bash
npm install
npx cds deploy --to sqlite:db.sqlite
npm run watch
```

6. `/chrome-devtools-test` でブラウザ確認 + スクリーンショット（発表資料に使う）

---

## ディレクトリ構成

```
handson/
├── .cline/skills/
│   ├── dashboard-requirements/SKILL.md  ← ドラフト→要件書 増幅スキル
│   ├── utility-dashboard/SKILL.md       ← メイン生成スキル（CAP + UI5）
│   └── chrome-devtools-test/SKILL.md    ← ブラウザテストスキル
├── templates/
│   ├── draft-template.md                ← draft.md の記入用テンプレート
│   └── requirements-template.md         ← 要件書の参考サンプル（電力会社向け詳細版）
├── draft.md            ← グループが記入するアイデアの種（穴埋め）
├── requirements.md     ← /dashboard-requirements が生成する業務要件書
├── package.json
└── README.md
```

> MCP 設定は Cline の UI 側（`cline_mcp_settings.json`）に保持。このディレクトリには含まない。

---

## よくあるトラブル

| 症状 | 対処 |
|------|------|
| `Assertion failed: (env) != nullptr` でサーバが落ちる | `package.json` に `better-sqlite3` 直依存がないか確認。`@cap-js/sqlite@^2` 経由のみにする |
| `no such table` で CSV が入らない | `npx cds deploy --to sqlite:db.sqlite` を先に実行する |
| チャートが真っ白 | OData エンドポイントが 200 か確認。`/chrome-devtools-test` のステップ 4 を参照 |
| マップが表示されない | `requirements.md` のスコープで `map` が選ばれているか確認（`draft.md` §5 が「はい」か） |
| MCP サーバに接続できない | Cline の MCP 設定画面でサーバ一覧を確認し、再接続する |
