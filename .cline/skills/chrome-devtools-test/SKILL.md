---
name: chrome-devtools-test
description: >
  SAP CAP + SAPUI5 アプリ実装後に chrome-devtools MCP
  サーバ（@modelcontextprotocol/server-puppeteer）を
  使ってブラウザテストを実行する。要件書駆動の汎用版——特定のチャート ID やエンドポイントにハードコードせず、 $metadata とDOM
  自動探索で任意ダッシュボードをテストする。 起動確認・コンソールエラー検出・OData
  疎通確認・チャート描画確認・スクリーンショット保存（発表資料用）の手順を提供する。
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(npx *)
  - Bash(cds *)
  - Bash(npm *)
  - Bash(lsof *)
  - mcp__chrome-devtools__puppeteer_navigate
  - mcp__chrome-devtools__puppeteer_screenshot
  - mcp__chrome-devtools__puppeteer_evaluate
  - mcp__chrome-devtools__puppeteer_click
  - mcp__chrome-devtools__puppeteer_fill
  - mcp__chrome-devtools__puppeteer_hover
  - mcp__chrome-devtools__puppeteer_select
  - mcp__cds-mcp__*
  - mcp__ui5-mcp-server__*
disabled: true
---

# /chrome-devtools-test — SAP UI5 アプリ ブラウザテストスキル（汎用版）

Cline に事前設定済みの `chrome-devtools`（`@modelcontextprotocol/server-puppeteer`）を使って
`cds watch` 起動中の SAP CAP + SAPUI5 アプリをブラウザテストする。

このスキルは要件書で定義した**任意テーマ**のダッシュボードに対応する（特定のチャート ID や
OData パスにハードコードしない）。スクリーンショットは発表資料として使える。

**使用するツール名（MCP サーバ名 `chrome-devtools` + Puppeteer ツール名）:**

| 操作 | ツール名 |
|------|---------|
| URL ナビゲーション | `mcp__chrome-devtools__puppeteer_navigate` |
| スクリーンショット | `mcp__chrome-devtools__puppeteer_screenshot` |
| JavaScript 実行 | `mcp__chrome-devtools__puppeteer_evaluate` |
| CSS セレクタクリック | `mcp__chrome-devtools__puppeteer_click` |
| フォーム入力 | `mcp__chrome-devtools__puppeteer_fill` |
| ホバー | `mcp__chrome-devtools__puppeteer_hover` |

Arguments passed: `$ARGUMENTS`

---

## 前提条件の確認

### 1. MCP サーバ設定の確認

`chrome-devtools` は Cline の UI から事前に登録済みであることを前提とする。
設定は Cline のグローバル設定ファイル `cline_mcp_settings.json` に保存されており、
相当する内容は以下のとおり（このプロジェクトに `.mcp.json` は置かない）：

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-puppeteer"],
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

Cline パネル右上の **MCP Servers** アイコン →「Configure MCP Servers」から確認・編集できる。
macOS での設定ファイルの実パス：
`~/Library/Application Support/Code/User/globalStorage/saoudrizwan.claude-dev/settings/cline_mcp_settings.json`

初回は Chromium のインストールが必要な場合がある：

```bash
npx puppeteer browsers install chrome
```

### 2. CAP サーバの起動確認

```bash
lsof -i :4004 | grep LISTEN
```

起動していない場合はプロジェクトディレクトリで：

```bash
npx cds deploy --to sqlite:db.sqlite  # 未実施なら先に
npm run watch &
```

---

## テスト変数の決定

Arguments から取得するか、自動探索で決定する。

| 変数 | Arguments 例 | 自動探索方法 |
|------|-------------|------------|
| `BASE_URL` | `url=http://localhost:4004` | デフォルト `http://localhost:4004` |
| `APP_PATH` | `app=store-dashboard` | `app/` 配下のディレクトリ名を `Read` で確認 |
| `ODATA_BASE` | `odata=/odata/v4/dashboard` | `srv/*.cds` の `@(path:)` アノテーションを `Read` で確認 |

Arguments が渡された場合はそれを優先する。  
渡されていない場合はプロジェクト内の以下を読んで自動決定する：

```bash
# app/ 配下のアプリディレクトリ名を確認
ls app/

# service の OData パスを確認
grep -r '@(path:' srv/
```

---

## テスト実行手順

### Step 0: アプリ URL の構築

決定した変数から `APP_URL` を構築する。

パターン1（Cline 標準）: `{BASE_URL}/{APP_PATH}/webapp/index.html`  
パターン2（CAP 自動マウント）: `{BASE_URL}/{APP_PATH}/index.html`

どちらが有効か確認するため、まず `$metadata` でサービスが動いているか確認：

```
mcp__chrome-devtools__puppeteer_navigate を呼び出す。
引数: { "url": "{ODATA_BASE}/$metadata" }
```

XML が返れば OData サービスは正常。

---

### Step 1: ダッシュボード画面へナビゲーション

```
mcp__chrome-devtools__puppeteer_navigate を呼び出す。
引数: { "url": "{APP_URL}" }
```

確認：ナビゲーションが成功すること（エラーなし）。

---

### Step 2: UI5 初期化待機 + 初期スクリーンショット

UI5 の非同期初期化が完了するまで待ってからスクリーンショットを撮影する。

```
mcp__chrome-devtools__puppeteer_evaluate を呼び出す。
引数: {
  "script": "new Promise(resolve => { const check = setInterval(() => { if (typeof sap !== 'undefined' && sap.ui && sap.ui.getCore().isInitialized()) { clearInterval(check); resolve('ready'); } }, 200); setTimeout(() => { clearInterval(check); resolve('timeout'); }, 15000); })"
}
```

`'ready'` が返ったら（`'timeout'` の場合は後述のデバッグ参照）：

```
mcp__chrome-devtools__puppeteer_screenshot を呼び出す。
引数: { "name": "01_initial_view" }
```

---

### Step 3: コンソールエラーの捕捉とページタイトル確認

```
mcp__chrome-devtools__puppeteer_evaluate を呼び出す。
引数: {
  "script": "window.__capturedErrors = []; const _orig = console.error.bind(console); console.error = (...a) => { window.__capturedErrors.push(a.join(' ')); _orig(...a); }; JSON.stringify({ title: document.title, url: location.href })"
}
```

5 秒後にエラーを確認：

```
mcp__chrome-devtools__puppeteer_evaluate を呼び出す。
引数: {
  "script": "JSON.stringify(window.__capturedErrors || [])"
}
```

確認ポイント：
- sap.ui ロード失敗エラーがないこと
- 404 Not Found がないこと
- 未定義コントロールエラーがないこと

---

### Step 4: OData エンドポイント動的探索と疎通確認

`$metadata` からエンティティセット一覧を自動取得して全エンドポイントを確認する。

```
mcp__chrome-devtools__puppeteer_evaluate を呼び出す。
引数: {
  "script": "fetch('{ODATA_BASE}/$metadata').then(r => r.text()).then(xml => { const matches = [...xml.matchAll(/EntitySet Name=\"([^\"]+)\"/g)]; return JSON.stringify(matches.map(m => m[1])); })"
}
```

取得したエンティティセット一覧を使って全エンドポイントを確認：

```
mcp__chrome-devtools__puppeteer_evaluate を呼び出す。
引数: {
  "script": "const sets = <<Step4で取得したリスト>>; Promise.all(sets.map(s => fetch('{ODATA_BASE}/' + s + '?$top=1').then(r => ({ name: s, status: r.status })))).then(r => JSON.stringify(r))"
}
```

全エンドポイントが `status: 200` であることを確認する。

---

### Step 5: データ読み込み待機 + チャートエリアスクリーンショット

OData からデータが読み込まれ UI に反映されるまで待機する。

```
mcp__chrome-devtools__puppeteer_evaluate を呼び出す。
引数: {
  "script": "new Promise(resolve => setTimeout(resolve, 3000)).then(() => 'waited')"
}
```

チャートエリアのスクリーンショット：

```
mcp__chrome-devtools__puppeteer_screenshot を呼び出す。
引数: { "name": "02_charts_loaded" }
```

---

### Step 6: VizFrame チャート描画の汎用確認

チャート ID をハードコードせず、`.sapVizFrame` セレクタで全 VizFrame を探して SVG の有無を確認する。

```
mcp__chrome-devtools__puppeteer_evaluate を呼び出す。
引数: {
  "script": "const frames = Array.from(document.querySelectorAll('[class*=\"sapVizFrame\"]')); const results = frames.map(f => { const id = f.id || f.getAttribute('data-sap-ui') || 'unknown'; const hasSvg = !!f.querySelector('svg'); return { id, hasSvg }; }); JSON.stringify({ total: frames.length, results })"
}
```

- `total > 0` であること（要件書でチャートを 1 つ以上宣言した場合）
- 全チャートで `hasSvg: true` であること
- `hasSvg: false` のチャートがある場合は該当 id を記録し、Step 11 の自己修正ループへ

---

### Step 7: ヒートマップ確認（存在する場合のみ）

要件書にヒートマップが含まれる場合：

```
mcp__chrome-devtools__puppeteer_evaluate を呼び出す。
引数: {
  "script": "const els = document.querySelectorAll('[id*=\"heatmapContainer\"]'); let tableFound = false, cellCount = 0; els.forEach(el => { if (el.querySelector('table')) { tableFound = true; cellCount = el.querySelectorAll('td').length; } }); JSON.stringify({ tableFound, cellCount })"
}
```

- `tableFound: true`、`cellCount > 0` であること

---

### Step 8: マップ確認（マップありの場合のみ）

要件書にマップが含まれる場合は「マップ」ボタンをクリックして確認する。

```
mcp__chrome-devtools__puppeteer_evaluate を呼び出す。
引数: {
  "script": "const btns = Array.from(document.querySelectorAll('.sapMBtnInner bdi')); const mapBtn = btns.find(b => b.textContent.includes('マップ')); mapBtn ? mapBtn.closest('button').id : 'not found'"
}
```

取得した id でクリック、3 秒待機後に Leaflet 確認：

```
mcp__chrome-devtools__puppeteer_evaluate を呼び出す。
引数: {
  "script": "new Promise(resolve => setTimeout(resolve, 3000)).then(() => { const containers = document.querySelectorAll('.leaflet-container'); let tileCount = 0, markerCount = 0; containers.forEach(c => { tileCount += c.querySelectorAll('.leaflet-tile-loaded').length; markerCount += c.querySelectorAll('.leaflet-interactive').length; }); return JSON.stringify({ hasLeaflet: containers.length > 0, tileCount, markerCount }); })"
}
```

```
mcp__chrome-devtools__puppeteer_screenshot を呼び出す。
引数: { "name": "03_map_view" }
```

---

### Step 9: 更新ボタン動作確認

```
mcp__chrome-devtools__puppeteer_evaluate を呼び出す。
引数: {
  "script": "const btn = document.querySelector('[data-sap-ui-icon=\"sap-icon://refresh\"], [icon=\"sap-icon://refresh\"]'); const parent = btn && btn.closest('button'); if (parent) parent.click(); parent ? 'refresh clicked' : 'button not found'"
}
```

---

### Step 10: KPI カード確認

```
mcp__chrome-devtools__puppeteer_evaluate を呼び出す。
引数: {
  "script": "const cards = document.querySelectorAll('.dashKpiCard'); const values = Array.from(cards).map(c => ({ text: c.querySelector('.dashKpiNum') ? c.querySelector('.dashKpiNum').textContent : '-' })); JSON.stringify({ count: cards.length, values })"
}
```

- `count > 0`（要件書で KPI カードを宣言した場合）
- KPI 値が `"0"` や `"-"` でない（データが読み込まれている）こと

---

### Step 11: 最終スクリーンショット（発表資料用）

全テスト完了後の最終状態を撮影する。

```
mcp__chrome-devtools__puppeteer_screenshot を呼び出す。
引数: { "name": "04_final_dashboard" }
```

> スクリーンショットは発表時にグループの成果物として使える。

---

### Step 12: テスト結果レポート出力

以下の形式でまとめて出力する：

```
## テスト結果レポート
実行日時: <timestamp>
対象 URL: {APP_URL}
OData: {ODATA_BASE}

| # | テスト項目 | 結果 | 備考 |
|---|-----------|------|------|
| 1 | ページナビゲーション | PASS/FAIL | |
| 2 | UI5 初期化 | PASS/FAIL | |
| 3 | コンソールエラー | PASS/FAIL | エラー件数: N |
| 4 | OData エンドポイント疎通 | PASS/FAIL | エンティティ数: N / 失敗: 一覧 |
| 5 | VizFrame チャート描画 | PASS/FAIL | 総数: N / 未描画: ID 一覧 |
| 6 | ヒートマップ描画 | PASS/FAIL/N/A | セル数: N |
| 7 | マップ描画 | PASS/FAIL/N/A | マーカー数: N |
| 8 | 更新ボタン動作 | PASS/FAIL | |
| 9 | KPI カード表示 | PASS/FAIL | カード数: N |

総合: PASS/FAIL (N/9 passed)
スクリーンショット: 01_initial_view, 02_charts_loaded[, 03_map_view], 04_final_dashboard
```

---

### Step 13: 失敗時の自己修正ループ

FAIL 項目がある場合は以下の手順で自己修正する（最大 3 回ループ）。

| 症状 | 調査・修正 |
|------|-----------|
| `UI5 初期化 timeout` | `typeof sap` が `'object'` か確認。CDN 疎通問題の可能性 |
| `OData 404` | `cds watch` ログでサービスパスを確認。`srv/*.cds` の `@(path:)` を確認 |
| チャート `hasSvg: false` | controller の `setProperty` でデータが空か確認。OData レスポンスを評価 |
| ヒートマップ空 | `_renderHeatmap` の呼び出しとデータが届いているか確認 |
| マップ表示されない | `typeof L` が `'function'` か確認（Leaflet CDN ブロックの可能性） |
| KPI が 0 / `-` | `bindList().requestContexts()` の結果が空か確認。`$top` 件数を増やす |

**修正後はステップ 2 から再実行する**（最大 3 回試みて改善しなければエラー内容を報告する）。

> **MCP ルール（実装の前提）**: 自己修正でコードを直す際は Cline に設定済みの MCP サーバを使う。
> - CDS/CSV/service の修正（OData 404・no such table 等）→ **`cds-mcp`（`@cap-js/mcp-server`）**
> - view/controller/manifest/sap.viz の修正（`hasSvg: false`・uid 誤り等）→ **`ui5-mcp-server`（`@ui5/mcp-server`）**
> - ブラウザ検証（本スキル）→ **`chrome-devtools`（`@modelcontextprotocol/server-puppeteer`）**
>
> 固定足場・部品レシピは `/utility-dashboard` スキルの逐語コピーを正とし、MCP は API 検証に使う。

---

## 失敗時のデバッグ手順

| 症状 | 調査 |
|------|------|
| ページが表示されない | `lsof -i :4004` で cds watch 起動確認 |
| `puppeteer_navigate` がタイムアウト | `npx puppeteer browsers install chrome` を実行 |
| セレクタが見つからない | evaluate で `document.body.innerHTML` の先頭部分を確認 |
| スクリーンショットが空白 | UI5 初期化待機時間を伸ばす（10000ms → 15000ms） |

---

## 他アプリへの適用

Arguments にカスタム URL を渡すことで他の SAPUI5 アプリにも適用できる：

```
/chrome-devtools-test url=http://localhost:4004 app=my-app-name odata=/odata/v4/dashboard
```
