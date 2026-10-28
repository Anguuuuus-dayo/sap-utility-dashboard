---
name: utility-dashboard
description: >
  要件書 `requirements.md` から、任意テーマの SAP CAP + SAPUI5 ダッシュボードを一から生成する。ハンズオンワークショップ
  Phase 2 用 B 版スキル—— 業務内容は要件書が決め、技術的品質はスキルのレシピが保証する。
disabled: true
---

> ## 🚨 最重要・参照ルール（必読・例外なし）
>
> このスキルを起動したら **最初に `.cline/skills/utility-dashboard/SKILL.md` を
> `start_line` / `end_line` でページングして全行読み込むこと。**
>
> 1. **固定足場・レシピ構文は逐語コピー**すること。
>    コード・XML・JSON・CSS は一字一句このファイルの通りに出力し、
>    推測や省略で変えてはならない。
> 2. **業務内容（schema 中身・CSV 行・カード/チャート選択）は要件書 `requirements.md` から生成する。**
>    「発電所」「電力会社」「WorkOrder」等の固定語を要件書外のテーマに流用しない。
> 3. **配管（OData パス・manifest dataSource・FeedItem uid）は推測で変えない。**
>    §1 で導出した実装プランで決まる部品だけ差し込み、それ以外はレシピ通り。
> 4. 切り詰め（`...[truncated N chars]...`）が出たら欠落範囲を再度読み込む。
>    補完・他ファイル流用は厳禁。
> 5. **実装には Cline に設定済みの MCP サーバを使う**（§0-1 参照）。
>    CAP/CDS 層は `cds-mcp`、SAPUI5/sap.viz 層は `ui5-mcp-server`、
>    ブラウザ検証は `chrome-devtools` を前提とする。

---

# Skill: utility-dashboard（B 版）
## ワークショップ Phase 2 — 要件書 × レシピで本物を作る

---

### Phase 1 vs Phase 2 の違い（ファシリテーター向けメモ）

| | Phase 1（自由プロンプト） | Phase 2（このスキル） |
|---|---|---|
| 指示 | 「〇〇のダッシュボードを作って」 | `/utility-dashboard`（要件書から自動） |
| テーマ | LLM が勝手に決める | **グループが自由に選ぶ** |
| データ | LLM が想像した仮データ | 要件書のルールから現実感ある CSV を生成 |
| UI | HTML テーブルや崩れたチャート | SAPUI5 + sap.viz、部品レシピで品質保証 |
| 動作 | 高確率で起動不能 | CAP + OData V4 を完全に設定済みで確実に動く |

**このスキルが証明すること**: AI は「何を作るか」ではなく「どう指示するか」で結果が劇的に変わる。

---

## §0 ツール確認（MCP と前提ソフトウェア）

### 0-1. MCP サーバを利用する（実装ルール・必須）

> **このスキルでの実装は、Cline に事前設定済みの以下 3 つの MCP サーバを使うことを前提とする。**
> （MCP サーバは Cline の UI から登録済みで、設定は `cline_mcp_settings.json` に保存されている。
> このプロジェクトに `.mcp.json` は置かない。）
> 推測でコードを書く前に、必ず対応する MCP サーバのツールで公式情報・テンプレートを確認すること。

| Cline 上のサーバ名 | パッケージ | 用途（どの工程で使うか） |
|---|---|---|
| `cds-mcp` | `@cap-js/mcp-server` | **CAP/CDS 層**。schema.cds（§4）・service.cds（§6）・CSV 配備（§8）の生成と検証。CDS モデルの確認、`cds` コマンド補助、CAP ドキュメント参照。 |
| `ui5-mcp-server` | `@ui5/mcp-server` | **SAPUI5 層**。manifest.json・view・controller・sap.viz（§7）の生成と検証。UI5/sap.viz の API リファレンス・ガイドライン参照、アプリ雛形生成・実行。 |
| `chrome-devtools` | `@modelcontextprotocol/server-puppeteer` | **ブラウザ検証**。`/chrome-devtools-test`（§8 以降）で起動・OData 疎通・チャート描画・スクショ確認。 |

**使い方の原則:**

1. 各サーバが公開する**ツール名は起動時に `tools/list` で確認して使う**（バージョンで変わり得るため逐語で決め打ちしない）。
   - 例: `ui5-mcp-server` の API リファレンス取得ツールで `sap.viz.ui5.controls.VizFrame` / `FeedItem` の `uid`・`vizType` を確認する。
   - 例: `cds-mcp` のドキュメント／モデル検証ツールで `@readonly` projection・Association 展開・CSV 命名を確認する。
2. **固定足場・部品レシピ（§3・§7）はこのスキルの逐語コピーを正とする。**
   MCP で得た情報は「レシピが最新 API と矛盾していないかの検証」と「要件書の可変部の埋め方の確認」に使い、レシピ本体を勝手に書き換えない。
3. MCP で得た情報とこのスキルのレシピが食い違う場合は、**動作確認済みのレシピを優先**し、相違点はユーザーに一言報告する。
4. いずれかの MCP サーバが利用不可の場合のみ、このスキルに埋め込まれたコード（動作確認済み）をそのまま使ってよい。その旨をユーザーに報告する。

### 0-2. ソフトウェア前提を確認する

```bash
node -v    # v20 以上（下記の Node 24 注意も参照）
npm -v
cds -v     # 不足時: npm install -g @sap/cds-dk
```

> ## 🚨 Node.js × SQLite ドライバの重大注意（必読）
>
> **`better-sqlite3` はネイティブアドオンで Node.js メジャーバージョンと強く結びついている。
> 古い `better-sqlite3`（v11〜v13 系）は Node.js v24 以降のGCと非互換で、
> `$metadata` 応答直後などに次のクラッシュを起こす:**
>
> ```
> node[xxxxx]: void node::RemoveEnvironmentCleanupHook(...) at ../src/api/hooks.cc:142
> Assertion failed: (env) != nullptr
> ```
>
> これはアプリのバグではなく **依存ライブラリとランタイムの非互換**。回避策:
>
> 1. `package.json` に `better-sqlite3` を **直接書かない**。
>    `@cap-js/sqlite@^2` が Node 24 対応の `better-sqlite3`（v12 系）を内蔵するので
>    SQLite ドライバはそちら経由で自動解決させる。
> 2. `@sap/cds@^9` / `@cap-js/sqlite@^2` / `@sap/cds-dk@^9` を使う。
> 3. クラッシュが出たら `rm -rf node_modules package-lock.json && npm install` で再インストール。

---

## §1 要件書 `requirements.md` の読み込みと実装プラン導出

> **前提**: このスキルは、先行する `/dashboard-requirements` が生成した
> 業務物語中心の要件書 `requirements.md`（プロジェクト直下）を入力とする。
> グループが構造化表を手で埋めることはしない——構造化はこのスキルが内部で行う。

### 1-1. 要件書を全文読む

プロジェクト直下の `requirements.md` を**全文**読み込む。
特に「今回実装するスコープ」節（作る KPI／チャート／地図の宣言）を正とする。

- `requirements.md` が無い場合は、先に `/dashboard-requirements` でドラフト（`draft.md`）から
  要件書を生成するようユーザーに案内し、ここで停止する。

### 1-2. 実装プランを内部導出する（生成前にユーザー提示）

要件書の業務物語とスコープ節から、以下の**実装プラン**を導出し、
生成に入る前に箇条書きでユーザーに提示して合意を取る（受け入れ確認を兼ねる）。
緯度経度・コード等の具体値はグループに書かせず、**物語から現実感を持ってこのスキルが付与する**。

1. **Subject 一覧** — 要件書に登場する拠点/設備/店舗等。各 Subject に
   短い英数字コード（例 `S001`）を採番し、地図ありなら現実感ある緯度・経度を付与する。
2. **指標 Metric 一覧** — KPI カードやチャートに使う数値。コード・表示名・単位・目標値・
   「高い方が良いか」を物語から決める。
3. **画面レイアウト** — スコープ節に対応づく部品を上から順に並べる
   （`KPIカード行` / `donut` / `line` / `bar` / `stacked_bar` / `heatmap` / `map` から選択。§7 の手札）。
4. **モックデータ生成ルール** — 期間（月次）・値レンジ・傾向・拠点ごとの特性。
   物語の雰囲気（増加傾向・季節性・拠点規模差など）を反映する。

> 実装プランは §2 のマッピング規則の入力になる。スコープ節に無い部品は作らない。
> 地図は要件書が明示的に求める場合のみ有効化する。

---

## §2 実装プラン → 実装マッピング規則

§1 で導出した実装プランの各要素を以下のように変換する。

| 実装プランの要素 | 生成される成果物 |
|---|---|
| テーマ | `package.json` の `name` / `description` |
| Subject 一覧の各行 | `db/schema.cds` の Subject entity の行 / `db/data/<ns>-Subject.csv` の行 |
| Subject の緯度・経度あり（地図スコープ時） | マップ機能を有効化（§7 マップ部品レシピを使う） |
| 指標 Metric 一覧の各行 | `db/schema.cds` の MonthlyMetric entity のフィールド / CSV の列 |
| レイアウト「KPIカード行」 | `view/Dashboard.view.xml` の KPI 行ブロック |
| レイアウト「donut」 | donut 部品レシピ（§7）のパネル |
| レイアウト「line」 | line 部品レシピ（§7）のパネル |
| レイアウト「bar」 | bar 部品レシピ（§7）のパネル |
| レイアウト「stacked_bar」 | stacked_bar 部品レシピ（§7）のパネル |
| レイアウト「heatmap」 | heatmap 部品レシピ（§7）の HTML テーブル |
| レイアウト「map」 | Leaflet マップ部品レシピ（§7）+ mapPanel |
| モックデータ生成ルール | `db/data/*.csv` の具体的な行の値 |

> Subject コード・緯度経度はグループ記入ではなく、**§1 でこのスキルが物語から現実感を持って生成する**。

**名前一致ルール**（これを守らないと OData が壊れる）:
- CDS の entity 名 ↔ CSV ファイル名（`<namespace>-<EntityName>.csv`）
- CDS のフィールド名 ↔ CSV のカラム名
- service.cds の projection 名 ↔ controller の `bindList("/...")` パス名
- manifest.json の `dataSource.uri` ↔ service の `@(path:)` アノテーション

---

## §3 固定足場（逐語コピー）

### 3-1. プロジェクト初期化

```bash
cds init . --add nodejs
```

`package.json`, `db/`, `srv/`, `app/` が生成されたことを確認してから次へ進む。

### 3-2. `package.json`（上書き）

> **`better-sqlite3` は直接書かない**（§0-2 参照）。

要件書のテーマに合わせて `name` と `description` のみ変更し、それ以外は逐語コピー。

```json
{
  "name": "<<要件書のテーマを短い英数字で>>",
  "version": "1.0.0",
  "description": "<<要件書のテーマの日本語説明>>",
  "license": "UNLICENSED",
  "private": true,
  "dependencies": {
    "@sap/cds": "^9",
    "express": "^4"
  },
  "devDependencies": {
    "@cap-js/sqlite": "^2",
    "@sap/cds-dk": "^9"
  },
  "scripts": {
    "start": "cds-serve",
    "watch": "cds watch"
  },
  "cds": {
    "requires": {
      "db": {
        "kind": "sqlite",
        "credentials": { "database": "db.sqlite" }
      }
    }
  },
  "allowScripts": {
    "better-sqlite3": true
  }
}
```

### 3-3. `app/<app-id>/webapp/index.html`

`<app-id>` は要件書のテーマから `<英数字-dashboard>` 形式で決める
（例: `store-dashboard`、`factory-dashboard`）。
`data-sap-ui-resourceroots` の名前空間も同様に決める（例: `my.company.storedashboard`）。

> **Leaflet を使う場合（レイアウト表に `map` がある場合）** は `<link>` と `<script>` も含める。
> **マップなしの場合** は Leaflet 関連の行を省略する。

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title><<要件書のテーマ>></title>
  <!-- マップありの場合のみ以下 2 行を含める -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" crossorigin=""/>
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js" crossorigin=""></script>
  <!-- ここまで -->
  <script id="sap-ui-bootstrap"
    src="https://ui5.sap.com/resources/sap-ui-core.js"
    data-sap-ui-theme="sap_horizon"
    data-sap-ui-resourceroots='{"<<名前空間>>": "./"}'
    data-sap-ui-compatVersion="edge"
    data-sap-ui-async="true"
    data-sap-ui-frameOptions="allow"
    data-sap-ui-xx-waitForTheme="init">
  </script>
  <script>
    sap.ui.getCore().attachInit(function() {
      sap.ui.require(["sap/ui/core/ComponentContainer"], function(ComponentContainer) {
        new ComponentContainer({
          height: "100%",
          name: "<<名前空間>>",
          settings: { id: "dashboard" },
          manifest: true
        }).placeAt("body");
      });
    });
  </script>
  <style>html, body { margin: 0; padding: 0; height: 100%; overflow: hidden; }</style>
</head>
<body class="sapUiBody" id="body" style="height:100vh;"></body>
</html>
```

### 3-4. `app/<app-id>/webapp/Component.js`

名前空間のみ実装プランに合わせて変更する。

```js
sap.ui.define(["sap/ui/core/UIComponent", "sap/ui/Device"], function(UIComponent, Device) {
  "use strict";
  return UIComponent.extend("<<名前空間>>.Component", {
    metadata: { manifest: "json" },
    init() {
      UIComponent.prototype.init.apply(this, arguments);
      this.getRouter().initialize();
    }
  });
});
```

### 3-5. `app/<app-id>/webapp/view/App.view.xml`（逐語コピー）

```xml
<mvc:View xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m" displayBlock="true">
  <App id="app"/>
</mvc:View>
```

### 3-6. `app/<app-id>/webapp/i18n/i18n.properties`

```properties
appTitle=<<要件書のテーマ>>
```

---

## §4 CDS スキーマ・レシピ

> **MCP ルール（§0-1）**: この節（§4）と §6・§8 の CAP/CDS 生成は `cds-mcp` を使う。
> CDS 構文・`@readonly` projection・Association 展開・CSV 命名規則を `cds-mcp` のツールで確認してから出力する。

### 4-1. namespace と共通 import

namespace は `<会社略称>.dashboard` 形式で統一する（例: `my.company.dashboard`）。  
`<<NS>>` はここで決めた namespace で以下を置換すること。

```cds
namespace <<NS>>;

using { cuid, managed, sap.common.CodeList } from '@sap/cds/common';
```

### 4-2. Subject（対象）entity

§1 で導出した Subject 一覧から生成。

- 緯度・経度列がある（マップあり）場合は `latitude` / `longitude` フィールドを含める
- ない（マップなし）場合は含めない

```cds
// カテゴリ用コードリスト（実装プランのカテゴリ値から）
entity Category : CodeList { key code : String(20); }

// Subject（拠点/店舗/設備 等）
entity Subject : cuid, managed {
  subjectId   : String(20) not null;
  name        : String(100) not null;
  category    : Association to Category;
  // マップありの場合のみ:
  latitude    : Decimal(9, 6);
  longitude   : Decimal(9, 6);
}
```

### 4-3. MonthlyMetric（月次指標）entity

§1 で導出した指標 Metric 一覧から生成。  
指標は横持ち（1 行 = 1 Subject × 1 Month の全指標値）にする。

```cds
entity MonthlyMetric : cuid {
  yearMonth   : String(7)  not null;   // 例: "2024-01"
  subject     : Association to Subject;
  // 以下は実装プランの指標コードに合わせて列を追加
  <<M001_code>> : Decimal(15, 2);
  <<M002_code>> : Decimal(15, 2);
  // ...
}
```

### 4-4. 追加 entity（stacked_bar / donut 用カテゴリが必要な場合）

stacked_bar でカテゴリ別の積み上げをする場合、カテゴリ軸の entity を追加する。

```cds
// 例: 商品カテゴリ別月次データ
entity MonthlyCategoryMetric : cuid {
  yearMonth   : String(7) not null;
  subject     : Association to Subject;
  category    : Association to Category;
  <<M001_code>> : Decimal(15, 2);
}
```

---

## §5 CSV データ・レシピ

### 5-1. ファイル命名規則

```
db/data/<namespace>-<EntityName>.csv
```

例: `my.company.dashboard-Subject.csv`、`my.company.dashboard-MonthlyMetric.csv`

### 5-2. カラム規則

| フィールド種別 | CSV カラム名 |
|---|---|
| 通常フィールド | フィールド名そのまま |
| Association（外部キー） | `<assoc名>_<key名>`（例: `subject_ID`、`category_code`） |
| cuid の ID | `ID` |

### 5-3. CodeList CSV（Category 等）

```csv
code,name
<<実装プランの各カテゴリ値>>,<<カテゴリ表示名>>
```

### 5-4. Subject CSV

ID は `s<6桁連番>-0000-0000-0000-<12桁連番>` 形式で採番。

```csv
ID,subjectId,name,category_code[,latitude,longitude]
s000001-0000-0000-0000-000000000001,<<コード>>,<<表示名>>,<<カテゴリcode>>[,<<緯度>>,<<経度>>]
...
```

### 5-5. MonthlyMetric CSV

**生成ルール（§1 のモックデータ生成ルールに従って現実感ある非ゼロ値を生成する）**:

- 行数 = 期間月数 × Subject 数
- 値は実装プランの値レンジ・傾向・拠点特性を反映する
- ゼロ行を作らない（ゼロだとチャートが空白になる）

```csv
ID,yearMonth,subject_ID,<<M001_code>>,<<M002_code>>[,...]
mm000001-0000-0000-0000-000000000001,2024-01,s000001-...,<<値>>,<<値>>
...
```

---

## §6 サービス・レシピ

**方針: `@readonly` projection のみ。カスタム OData 関数は使わない。**  
集計はコントローラ側（§7-7）で行う。これにより service.cds と service.js の名前不一致事故を排除する。

### 6-1. `srv/dashboard-service.cds`

namespace・entity 名・パスは実装プランに合わせて変更。  
projection は **全フィールド + Association の name を展開**する。

```cds
using <<NS>> as db from '../db/schema';

service DashboardService @(path: '/odata/v4/dashboard') {

  @readonly entity Subjects as projection on db.Subject {
    *, category.name as categoryName,
  }

  @readonly entity MonthlyMetrics as projection on db.MonthlyMetric {
    *, subject.name as subjectName, subject.subjectId as subjectId,
       subject.category.name as categoryName,
  }

  // stacked_bar 用カテゴリ指標がある場合:
  @readonly entity MonthlyCategoryMetrics as projection on db.MonthlyCategoryMetric {
    *, subject.name as subjectName, subject.subjectId as subjectId,
       category.name as categoryName, category.code as categoryCode,
  }
}
```

### 6-2. `srv/dashboard-service.js`（省略可）

`@readonly` projection のみなら JavaScript ファイルは不要。  
デフォルトの CDS ハンドラが OData を自動提供する。  
JavaScript が必要な場合（例: カスタムロジック）は空の実装から始める:

```js
const cds = require('@sap/cds');
module.exports = class DashboardService extends cds.ApplicationService {};
```

---

## §7 UI レシピ（固定フレーム＋可変中身）

> このセクションのコードブロックは品質内蔵の「部品レシピ」。
> **sap.viz の vizType / FeedItem uid は逐語コピーすること**（Phase 1 で最も間違いやすい箇所）。
>
> **部品は「手札（パレット）」**: §7-4〜§7-10 は選択可能な部品カタログ
> （`KPIカード行` / `donut` / `line` / `bar` / `stacked_bar` / `heatmap` / `map`）。
> §1 で導出した実装プランのレイアウトが要求する部品**だけ**を選んで差し込む。
> 要件書が既存の手札に無い図表を求めた場合は、**この節に新しい部品レシピを 1 つ追加**してから使う
> （既存レシピ本体は逐語コピーのまま変更しない）。
>
> **MCP ルール（§0-1）**: この節（§7）の manifest.json・view・controller・sap.viz 生成は `ui5-mcp-server` を使う。
> `vizType` / `FeedItem` の `uid` / manifest の dataSource 設定を `ui5-mcp-server` の API リファレンス・ガイドラインで検証してから出力する。
> レシピと食い違う場合は動作確認済みのレシピを優先し、相違点を報告する。

### 7-1. `app/<app-id>/webapp/css/style.css`（固定・逐語コピー）

```css
.dashHeaderSub    { font-size: 0.75rem; color: #666; font-weight: 600; letter-spacing: 0.05em; }
.dashHeaderTitle  { font-size: 0.9rem;  font-weight: bold; color: #1a1a1a; }
.dashHeaderDivider { font-size: 0.75rem; color: #bbb; }
.dashLastUpdated  { font-size: 0.75rem; color: #666; }
.dashLegendItem   { font-size: 0.72rem; margin-right: 0.25rem; }
.dashMapPanel     { min-width: 0; }
.dashMapDiv {
  width: 100%;
  height: calc(100vh - 11.5rem);
  min-height: 300px;
}
.dashChartsScroll { min-width: 0; box-sizing: border-box; }
.dashKpiRow {
  background: #f0f4f9;
  border-bottom: 2px solid #d4dce8;
  padding: 0.75rem 1rem;
}
.dashKpiCard {
  background: #ffffff;
  border: 1px solid #dce4ee;
  border-radius: 8px;
  padding: 1rem 1.5rem;
  margin: 0.3rem;
  min-width: 150px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  flex: 1;
}
.dashKpiNum     { font-size: 2.2rem !important; font-weight: 700 !important; line-height: 1 !important; color: #1a1a1a !important; }
.dashKpiNumUnit { font-size: 1rem !important; font-weight: 500 !important; color: #555 !important; padding-bottom: 0.3rem; }
.dashKpiLabel   { font-size: 0.8rem; color: #444; margin-top: 0.4rem; text-align: center; font-weight: 500; }
.dashKpiSub     { font-size: 0.68rem; color: #888; text-align: center; }
```

### 7-2. `app/<app-id>/webapp/manifest.json`

`<<APP_ID>>` / `<<名前空間>>` / `<<タイトル>>` のみ実装プランに合わせて変更。  
それ以外は逐語コピー（特に `odataVersion`, `synchronizationMode`, `operationMode`）。

```json
{
  "_version": "1.65.0",
  "sap.app": {
    "id": "<<名前空間>>",
    "type": "application",
    "title": "<<要件書のテーマ>>",
    "applicationVersion": { "version": "1.0.0" },
    "dataSources": {
      "DashboardService": {
        "uri": "/odata/v4/dashboard/",
        "type": "OData",
        "settings": { "odataVersion": "4.0" }
      }
    }
  },
  "sap.ui": {
    "technology": "UI5",
    "icons": { "icon": "sap-icon://it-system" },
    "supportedThemes": ["sap_horizon", "sap_horizon_dark"]
  },
  "sap.ui5": {
    "rootView": {
      "viewName": "<<名前空間>>.view.App",
      "type": "XML",
      "async": true,
      "id": "app"
    },
    "dependencies": {
      "minUI5Version": "1.120.0",
      "libs": { "sap.ui.core": {}, "sap.m": {}, "sap.f": {}, "sap.viz": {}, "sap.ui.layout": {} }
    },
    "resources": {
      "css": [{ "uri": "css/style.css" }]
    },
    "models": {
      "": {
        "dataSource": "DashboardService",
        "preload": true,
        "settings": { "synchronizationMode": "None", "operationMode": "Server", "autoExpandSelect": true }
      },
      "i18n": {
        "type": "sap.ui.model.resource.ResourceModel",
        "settings": { "bundleName": "<<名前空間>>.i18n.i18n" }
      }
    },
    "routing": {
      "config": {
        "routerClass": "sap.m.routing.Router",
        "viewType": "XML",
        "viewPath": "<<名前空間>>.view",
        "controlId": "app",
        "controlAggregation": "pages",
        "async": true
      },
      "routes": [{ "pattern": "", "name": "RouteDashboard", "target": "TargetDashboard" }],
      "targets": { "TargetDashboard": { "viewId": "dashboard", "viewName": "Dashboard" } }
    }
  }
}
```

### 7-3. `view/Dashboard.view.xml` — ページ外枠（固定・逐語コピー）

実装プランのレイアウトに従って中身のブロックを差し込む。  
`<<TITLE>>` / `<<名前空間>>` のみ変更。  
マップが **ない** 場合は `mapPanel` ブロックと `btnToggleMap` を省略する。

```xml
<mvc:View
  controllerName="<<名前空間>>.controller.Dashboard"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns="sap.m"
  xmlns:core="sap.ui.core"
  xmlns:l="sap.ui.layout"
  xmlns:viz="sap.viz.ui5.controls"
  xmlns:feeds="sap.viz.ui5.data"
  xmlns:vizFeeds="sap.viz.ui5.controls.common.feeds"
  xmlns:html="http://www.w3.org/1999/xhtml"
  displayBlock="true">

  <Page id="mainPage" showHeader="true" enableScrolling="false">
    <customHeader>
      <Bar class="dashHeaderBar">
        <contentLeft>
          <HBox alignItems="Center">
            <core:Icon src="sap-icon://it-system" size="1.3rem" color="#0070F2" class="sapUiSmallMarginEnd"/>
            <Text text="<<TITLE>>" class="dashHeaderTitle"/>
          </HBox>
        </contentLeft>
        <contentRight>
          <Text text="{view>/lastUpdated}" class="dashLastUpdated sapUiSmallMarginEnd"/>
          <Button icon="sap-icon://refresh" press=".onRefresh" tooltip="更新"/>
          <!-- マップありの場合のみ: -->
          <Button id="btnToggleMap" icon="sap-icon://map" text="マップ" type="Ghost"
                  press=".onToggleMapPanel" tooltip="マップを表示/非表示"/>
        </contentRight>
      </Bar>
    </customHeader>

    <content>
      <FlexBox id="mainLayout" direction="Row" width="100%">

        <!-- マップありの場合のみ: mapPanel ブロック -->
        <Panel id="mapPanel" class="sapUiNoContentPadding dashMapPanel" visible="{view>/mapVisible}">
          <layoutData><FlexItemData growFactor="45" shrinkFactor="0" baseSize="0px"/></layoutData>
          <headerToolbar>
            <OverflowToolbar>
              <Title text="<<マップタイトル>>" level="H4" class="sapUiTinyMarginBegin"/>
            </OverflowToolbar>
          </headerToolbar>
          <content>
            <html:div id="subjectMap" class="dashMapDiv"/>
          </content>
        </Panel>
        <!-- ここまで -->

        <!-- チャートエリア: マップなしは growFactor="100" に変更 -->
        <ScrollContainer id="chartsScroll" vertical="true" class="dashChartsScroll">
          <layoutData><FlexItemData growFactor="55" shrinkFactor="0" baseSize="0px"/></layoutData>

          <!-- ↓ ここに実装プランのレイアウトを順番に展開する ↓ -->

        </ScrollContainer>
      </FlexBox>
    </content>
  </Page>
</mvc:View>
```

### 7-4. KPI カード行レシピ（実装プランに KPIカード行 がある場合）

KPI カード 1 枚のテンプレート。指標コードごとに繰り返す。  
`border-top` の色は指標ごとに変える（4 色例: `#188918` / `#0070F2` / `#E9730C` / `#6A1B9A`）。

```xml
<FlexBox direction="Row" justifyContent="SpaceAround" wrap="Wrap"
         class="sapUiTinyMarginBeginEnd sapUiTinyMarginTop dashKpiRow">

  <!-- 指標 1 枚分のカード（指標の数だけ繰り返す） -->
  <VBox class="dashKpiCard" style="border-top:4px solid #188918;" alignItems="Center">
    <layoutData><FlexItemData growFactor="1" baseSize="0px"/></layoutData>
    <HBox alignItems="Baseline" justifyContent="Center">
      <Text text="{view>/kpi/<<M001_code>>}" class="dashKpiNum"/>
      <Text text=" <<単位>>" class="dashKpiNumUnit"/>
    </HBox>
    <Text text="<<指標表示名>>" class="dashKpiLabel"/>
    <Text text="目標 {view>/kpi/<<M001_code>>Target}<<単位>>" class="dashKpiSub"/>
  </VBox>

</FlexBox>
```

### 7-5. donut レシピ

`uid="size"` と `uid="color"` を使う。`vizType` は必ず `"donut"` にする（`"pie"` は Fiori 非推奨）。

```xml
<Panel headerText="<<donutタイトル>>" height="18rem">
  <viz:VizFrame id="chartDonut" vizType="donut" width="100%" height="15rem"
    uiConfig="{applicationSet:'fiori'}"
    vizProperties='{"title":{"visible":false},"legend":{"visible":true},"plotArea":{"dataLabel":{"visible":true,"type":"percentage"},"colorPalette":["#1EB7B2","#5D3FD3","#0070F2","#188918","#F0AB00","#E9730C"]}}'>
    <viz:dataset>
      <feeds:FlattenedDataset data="{view>/<<donut用データパス>>}">
        <feeds:dimensions>
          <feeds:DimensionDefinition name="<<カテゴリ軸名>>" value="{<<categoryField>>}"/>
        </feeds:dimensions>
        <feeds:measures>
          <feeds:MeasureDefinition name="<<値軸名>>" value="{<<valueField>>}"/>
        </feeds:measures>
      </feeds:FlattenedDataset>
    </viz:dataset>
    <viz:feeds>
      <vizFeeds:FeedItem uid="size"  type="Measure"   values="<<値軸名>>"/>
      <vizFeeds:FeedItem uid="color" type="Dimension" values="<<カテゴリ軸名>>"/>
    </viz:feeds>
  </viz:VizFrame>
</Panel>
```

### 7-6. line レシピ

`uid="categoryAxis"` / `uid="color"` / `uid="valueAxis"` を使う。

```xml
<Panel headerText="<<lineタイトル>>" height="18rem">
  <viz:VizFrame id="chartLine" vizType="line" width="100%" height="15rem"
    uiConfig="{applicationSet:'fiori'}"
    vizProperties='{"title":{"visible":false},"plotArea":{"dataLabel":{"visible":false}},"valueAxis":{"title":{"visible":true,"text":"<<単位>>"}},"categoryAxis":{"title":{"visible":false}}}'>
    <viz:dataset>
      <feeds:FlattenedDataset data="{view>/<<line用データパス>>}">
        <feeds:dimensions>
          <feeds:DimensionDefinition name="月"        value="{yearMonth}"/>
          <feeds:DimensionDefinition name="<<系列名>>" value="{<<seriesField>>}"/>
        </feeds:dimensions>
        <feeds:measures>
          <feeds:MeasureDefinition name="<<値軸名>>" value="{<<valueField>>}"/>
        </feeds:measures>
      </feeds:FlattenedDataset>
    </viz:dataset>
    <viz:feeds>
      <vizFeeds:FeedItem uid="categoryAxis" type="Dimension" values="月"/>
      <vizFeeds:FeedItem uid="color"        type="Dimension" values="<<系列名>>"/>
      <vizFeeds:FeedItem uid="valueAxis"    type="Measure"   values="<<値軸名>>"/>
    </viz:feeds>
  </viz:VizFrame>
</Panel>
```

### 7-7. bar レシピ

`vizType="bar"` は縦棒（SAPUI5 の命名に注意）。`uid="categoryAxis"` / `uid="valueAxis"` を使う。

```xml
<Panel headerText="<<barタイトル>>" height="18rem">
  <viz:VizFrame id="chartBar" vizType="bar" width="100%" height="15rem"
    uiConfig="{applicationSet:'fiori'}"
    vizProperties='{"title":{"visible":false},"valueAxis":{"title":{"visible":true,"text":"<<単位>>"}}}'>
    <viz:dataset>
      <feeds:FlattenedDataset data="{view>/<<bar用データパス>>}">
        <feeds:dimensions>
          <feeds:DimensionDefinition name="<<カテゴリ軸名>>" value="{<<categoryField>>}"/>
        </feeds:dimensions>
        <feeds:measures>
          <feeds:MeasureDefinition name="<<値軸名>>" value="{<<valueField>>}"/>
        </feeds:measures>
      </feeds:FlattenedDataset>
    </viz:dataset>
    <viz:feeds>
      <vizFeeds:FeedItem uid="categoryAxis" type="Dimension" values="<<カテゴリ軸名>>"/>
      <vizFeeds:FeedItem uid="valueAxis"    type="Measure"   values="<<値軸名>>"/>
    </viz:feeds>
  </viz:VizFrame>
</Panel>
```

### 7-8. stacked_bar レシピ

`vizType="stacked_bar"`。`uid="categoryAxis"` / `uid="color"` / `uid="valueAxis"` を使う。

```xml
<Panel headerText="<<stacked_barタイトル>>" height="18rem">
  <viz:VizFrame id="chartStackedBar" vizType="stacked_bar" width="100%" height="15rem"
    uiConfig="{applicationSet:'fiori'}"
    vizProperties='{"title":{"visible":false},"legend":{"visible":true},"plotArea":{"colorPalette":["#0070F2","#F0AB00","#188918","#E9730C"]}}'>
    <viz:dataset>
      <feeds:FlattenedDataset data="{view>/<<stacked用データパス>>}">
        <feeds:dimensions>
          <feeds:DimensionDefinition name="月"        value="{yearMonth}"/>
          <feeds:DimensionDefinition name="<<積み上げ軸名>>" value="{<<stackField>>}"/>
        </feeds:dimensions>
        <feeds:measures>
          <feeds:MeasureDefinition name="<<値軸名>>" value="{<<valueField>>}"/>
        </feeds:measures>
      </feeds:FlattenedDataset>
    </viz:dataset>
    <viz:feeds>
      <vizFeeds:FeedItem uid="categoryAxis" type="Dimension" values="月"/>
      <vizFeeds:FeedItem uid="color"        type="Dimension" values="<<積み上げ軸名>>"/>
      <vizFeeds:FeedItem uid="valueAxis"    type="Measure"   values="<<値軸名>>"/>
    </viz:feeds>
  </viz:VizFrame>
</Panel>
```

### 7-9. heatmap レシピ（HTML テーブル）

VizFrame を使わず `html:div` にコントローラから HTML テーブルを差し込む方式。  
div の id は `heatmapContainer` で固定（コントローラレシピと対応）。

```xml
<Panel headerText="<<heatmapタイトル>>" height="18rem">
  <content>
    <html:div id="heatmapContainer"
      style="width:100%;height:15rem;overflow:auto;padding:0.5rem;box-sizing:border-box;"/>
  </content>
</Panel>
```

### 7-10. マップ レシピ（Leaflet）

マップありの場合、§7-3 の `mapPanel` ブロックを含め、コントローラで Leaflet を初期化する。  
Subject CSV に緯度・経度が含まれていることが前提。

`subjectMap` div の id はコントローラと対応させる（変更しない）。

### 7-11. l:Grid でチャートを並べる

レイアウト表の「列数」に応じて `defaultSpan` を変える。  
2 列（デフォルト）: `XL6 L6 M12 S12` / 1 列（全幅）: `XL12 L12 M12 S12`

```xml
<l:Grid id="chartsGrid" defaultSpan="XL6 L6 M12 S12" width="100%">
  <!-- 実装プランのレイアウトの順にチャートパネルを並べる -->
</l:Grid>
```

---

## §7-12. `controller/Dashboard.controller.js` — データロード・レシピ

**方針**: `bindList().requestContexts()` で OData から取得し、JS 側で集計する。  
カスタム OData 関数は使わない。

以下はテンプレート。`<<NS_LOWER>>` / `<<METRIC_FIELD>>` 等は実装プランに合わせる。

```js
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/json/JSONModel",
  "sap/m/MessageToast"
], function(Controller, JSONModel, MessageToast) {
  "use strict";

  return Controller.extend("<<名前空間>>.controller.Dashboard", {

    onInit() {
      const oVM = new JSONModel({
        lastUpdated: "",
        mapVisible: false,   // マップなしの場合は削除
        kpi: {},
        // チャートデータは後で setProperty で設定
      });
      this.getView().setModel(oVM, "view");
      this._loadAllData();
    },

    onAfterRendering() {
      const oScroll = this.byId("chartsScroll");
      if (oScroll) oScroll.setHeight((window.innerHeight - 104) + "px");
    },

    _getODataModel() { return this.getOwnerComponent().getModel(); },

    async _loadAllData() {
      await Promise.all([
        this._loadSubjects(),
        this._loadMonthlyMetrics(),
      ]);
      this.getView().getModel("view").setProperty(
        "/lastUpdated", "最終更新: " + new Date().toLocaleString("ja-JP")
      );
    },

    // --- Subjects（マップ用）---
    async _loadSubjects() {
      try {
        const aCtxs = await this._getODataModel().bindList("/Subjects").requestContexts(0, 200);
        this._subjects = aCtxs.map(c => c.getObject());
        // マップあり: this._initMap(); 等を呼ぶ
      } catch(e) { console.error("Subjects load error", e); }
    },

    // --- MonthlyMetrics ---
    async _loadMonthlyMetrics() {
      try {
        const aCtxs = await this._getODataModel()
          .bindList("/MonthlyMetrics", null, null, null, { $orderby: "yearMonth,subjectId" })
          .requestContexts(0, 2000);
        const aData = aCtxs.map(c => c.getObject());
        const vm    = this.getView().getModel("view");

        // KPI: 最新月の平均
        const months  = [...new Set(aData.map(d => d.yearMonth))].sort();
        const latest  = months[months.length - 1];
        const latestD = aData.filter(d => d.yearMonth === latest);
        const avg = field => latestD.reduce((s, r) => s + (r[field] || 0), 0) /
                             (latestD.length || 1);
        const kpi = {};
        // 実装プランの指標コードごとに kpi オブジェクトを設定
        // 例: kpi.SALES = Math.round(avg("SALES") * 10) / 10;
        vm.setProperty("/kpi", kpi);

        // チャート用データを vm にセット
        // donut 用（最新月のカテゴリ別合計）
        // vm.setProperty("/donutData", ...);

        // line 用（全月 × 全 Subject）
        vm.setProperty("/lineData", aData);

        // bar 用（Subject 別合計、上位 5）
        const subjectTotals = {};
        aData.forEach(d => {
          if (!subjectTotals[d.subjectId]) {
            subjectTotals[d.subjectId] = { subjectId: d.subjectId, subjectName: d.subjectName, total: 0 };
          }
          // 例: subjectTotals[d.subjectId].total += d.SALES || 0;
        });
        vm.setProperty("/barData",
          Object.values(subjectTotals).sort((a, b) => b.total - a.total).slice(0, 5)
        );

        // heatmap 用
        this._renderHeatmap(aData, months);

      } catch(e) { console.error("MonthlyMetrics load error", e); }
    },

    // --- ヒートマップ描画（HTML テーブル）---
    _renderHeatmap(aData, months) {
      const oDom = document.getElementById(this.createId("heatmapContainer"));
      if (!oDom || !aData.length) return;
      const subjects = [...new Set(aData.map(d => d.subjectName))];
      const metricField = "<<heatmapで使う指標コード>>";
      const toColor = v => {
        if (v == null) return "#eee";
        // 実装プランの目標値に合わせて閾値を調整
        if (v >= 90) return "#188918";
        if (v >= 80) return "#1EB7B2";
        if (v >= 70) return "#0070F2";
        if (v >= 60) return "#F0AB00";
        if (v >= 50) return "#E9730C";
        return "#BB0000";
      };
      let html = `<table style="border-collapse:collapse;font-size:0.68rem;width:100%;">`;
      html += `<tr><th style="padding:2px 4px;text-align:left;"></th>`;
      months.forEach(m => {
        html += `<th style="padding:2px 4px;text-align:center;white-space:nowrap;">${m}</th>`;
      });
      html += `</tr>`;
      subjects.forEach(sub => {
        html += `<tr><td style="padding:2px 6px;white-space:nowrap;font-weight:bold;">${sub}</td>`;
        months.forEach(month => {
          const entry = aData.find(d => d.subjectName === sub && d.yearMonth === month);
          const val   = entry ? entry[metricField] : null;
          const bg    = toColor(val);
          const fg    = val != null && val >= 70 ? "#fff" : "#333";
          html += `<td style="padding:3px 5px;text-align:center;background:${bg};color:${fg};border:1px solid #fff;">` +
                  `${val != null ? Number(val).toFixed(1) : "-"}</td>`;
        });
        html += `</tr>`;
      });
      html += `</table>`;
      oDom.innerHTML = html;
    },

    // --- マップ（マップあり時のみ）---
    _initMap() {
      const oDom = this.byId("subjectMap").getDomRef();
      if (!oDom || typeof L === "undefined") return;
      this._map = L.map(oDom, { zoomControl: true }).setView([36.5, 137.5], 5);
      L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
        attribution: "© OpenStreetMap contributors"
      }).addTo(this._map);
      if (this._subjects) this._renderMapPins(this._subjects);
    },

    _renderMapPins(subjects) {
      if (!this._map) return;
      const group = L.layerGroup().addTo(this._map);
      subjects.forEach(s => {
        if (!s.latitude || !s.longitude) return;
        L.circleMarker([s.latitude, s.longitude], {
          radius: 10, fillColor: "#0070F2", color: "#fff", weight: 1.5, opacity: 1, fillOpacity: 0.85
        }).bindPopup(`<b>${s.name}</b><br>カテゴリ: ${s.categoryName || "-"}`).addTo(group);
      });
    },

    onToggleMapPanel() {
      const vm   = this.getView().getModel("view");
      const bOn  = !vm.getProperty("/mapVisible");
      vm.setProperty("/mapVisible", bOn);
      this.byId("btnToggleMap").setType(bOn ? "Emphasized" : "Ghost");
      if (bOn) setTimeout(() => {
        if (!this._map) this._initMap();
        else this._map.invalidateSize();
      }, 250);
    },

    onRefresh() {
      this._loadAllData();
      if (this._map) this._map.invalidateSize();
      MessageToast.show("データを更新しました");
    }
  });
});
```

---

## §8 起動確認

```bash
npm install

# CSV データを SQLite に投入（必須。省くと "no such table" でキャッシュ失敗）
npx cds deploy --to sqlite:db.sqlite

npm run watch
```

以下がすべて 200 で応答することを確認する:

| URL | 期待 |
|---|---|
| `http://localhost:4004` | CAP サービストップ |
| `http://localhost:4004/odata/v4/dashboard/Subjects` | 対象一覧 JSON |
| `http://localhost:4004/odata/v4/dashboard/MonthlyMetrics` | 月次指標 JSON |
| `http://localhost:4004/<app-id>/index.html` | ダッシュボード画面 |

確認できたら `/chrome-devtools-test` でブラウザテスト（スクショ撮影）を実行する。

> **🚨 `Assertion failed: (env) != nullptr` が出たら**（Node 24 × better-sqlite3 非互換）:
> ```bash
> rm -rf node_modules package-lock.json && npm install
> rm -f db.sqlite && npx cds deploy --to sqlite:db.sqlite
> npm run watch
> ```

---

## §9 ハードルール＆自己検証チェックリスト

実装完了前に以下をすべて確認する。1 つでも × なら修正してから次へ進む。

- [ ] `package.json` に `better-sqlite3` の **直依存が無い**
- [ ] `cds deploy` が成功し、全 CSV テーブルが作成された（`no such table` エラー なし）
- [ ] `cds watch` でサーバが起動し `$metadata` が返る（連続アクセスで落ちない）
- [ ] 要件書スコープの全 Subject がダッシュボードに反映されている
- [ ] 要件書スコープで宣言した全 KPI カードが表示されている
- [ ] 要件書スコープで宣言した全チャートが描画されている（宣言していないチャートは無い）
- [ ] 要件書スコープに `map` がない場合、マップボタン・Leaflet が存在しない
- [ ] schema ↔ CSV ↔ service ↔ UI の名前（entity 名・フィールド名）が一致している
- [ ] OData の各エンドポイントが 200 を返している
- [ ] データが非空（空配列でチャートが白紙にならない）
- [ ] CAP/CDS 生成に `cds-mcp`、SAPUI5/sap.viz 生成に `ui5-mcp-server` を使った（利用不可の場合はその旨を報告した）

チェック完了後 → `/chrome-devtools-test` でブラウザ起動・OData 疎通・チャート描画を確認する。

