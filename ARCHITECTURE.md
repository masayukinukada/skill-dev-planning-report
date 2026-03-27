# ARCHITECTURE.md — アーキテクチャ設計書

## 1. システム全体像

```
┌──────────────────────────────────────────────────┐
│               ユーザーのPC（ブラウザのみ）           │
│                                                  │
│   ┌───────────────────────────────────────────┐  │
│   │           index.html                      │  │
│   │                                           │  │
│   │  ┌─────────────┐    ┌──────────────────┐  │  │
│   │  │  フォームUI  │ → │  レポート生成JS  │  │  │
│   │  │  (HTML/CSS)  │    │  (Vanilla JS)    │  │  │
│   │  └─────────────┘    └────────┬─────────┘  │  │
│   │                              │             │  │
│   │                    ┌─────────▼──────────┐  │  │
│   │                    │  HTMLレポート文字列 │  │  │
│   │                    │  (テンプレートリテ  │  │  │
│   │                    │   ラルで構築)       │  │  │
│   │                    └─────────┬──────────┘  │  │
│   │                              │             │  │
│   │               ┌──────────────▼──────────┐  │  │
│   │               │  iframe プレビュー       │  │  │
│   │               └──────────────┬──────────┘  │  │
│   │                              │             │  │
│   │               ┌──────────────▼──────────┐  │  │
│   │               │  Blob URL ダウンロード   │  │  │
│   │               └─────────────────────────┘  │  │
│   └───────────────────────────────────────────┘  │
│                                                  │
│   外部通信: なし（完全オフライン動作）             │
└──────────────────────────────────────────────────┘
```

---

## 2. index.html の内部構造

```
index.html
│
├── <head>
│   ├── meta charset / viewport
│   └── <style> ─── アプリUI用CSS（フォーム画面全体）
│
├── <body>
│   ├── .app-header    ─── タイトルバー
│   ├── .step-nav      ─── ステップタブ（1〜4）
│   │
│   ├── #step0         ─── Step 1: 基本情報フォーム
│   ├── #step1         ─── Step 2: 予定案件テーブル
│   ├── #step2         ─── Step 3: 育成方針フォーム
│   ├── #step3         ─── Step 4: メンバー育成目標テーブル
│   │
│   └── #preview-wrap  ─── レポートプレビュー（iframe）
│
└── <script>
    ├── 状態変数宣言
    ├── goStep(n)       ─── ステップ切り替え
    ├── next(n)         ─── バリデーション付きステップ進行
    ├── addProj()       ─── 案件行追加
    ├── addMember()     ─── メンバー行追加
    ├── delRow()        ─── 行削除
    ├── collect()       ─── 全フォームデータ収集
    ├── escapeHTML()    ─── XSSサニタイズ
    ├── priorityBadge() ─── 重要度バッジHTML生成
    ├── lvBar()         ─── レベルドットHTML生成
    ├── initials()      ─── アバターイニシャル生成
    ├── avatarColor()   ─── アバター色決定
    ├── generate()      ─── レポートHTML構築 + iframe表示
    └── download()      ─── Blob URLダウンロード
```

---

## 3. データフロー

```
[DOM入力値]
    │
    │ collect() が読み取る
    ▼
[JavaScriptオブジェクト]
{
  year, team, leader, period, date, theme, scope,
  focus, target, vision, gap, policy,
  projects: [ {name, overview, timing, kind, skills, priority}, ... ],
  members:  [ {name, skill, currentLv, targetLv, timing, measure, note}, ... ]
}
    │
    │ generate() がテンプレートリテラルで展開
    │ escapeHTML() で全値をサニタイズ
    ▼
[reportHTML 文字列]（完全なHTMLドキュメント）
    │
    ├─── iframe.srcdoc に代入 → プレビュー表示
    │
    └─── Blob URL 経由でダウンロード
```

---

## 4. 状態管理

グローバル変数のみで管理（フレームワーク不使用）:

```javascript
let currentStep = 0;   // 表示中ステップ番号 (0-3)
let projIds    = [];    // 案件行のタイムスタンプID配列
let memberIds  = [];    // メンバー行のタイムスタンプID配列
let reportHTML = '';    // 最後に生成したHTMLレポート文字列
```

### 行IDの生成方針
動的に追加される行は `Date.now()` をIDとして使用する。
これにより同一セッション内での衝突を回避する。

```javascript
function addProj() {
  const id = Date.now();   // 例: 1717123456789
  projIds.push(id);
  // 各入力要素のidに付与: pnm1717123456789, pov1717123456789 ...
}
```

---

## 5. レポートHTML生成の設計

### 5-1. 構築方式
JavaScriptのテンプレートリテラルで `<!DOCTYPE html>` から `</html>` まで
すべてを文字列として組み立てる。

```javascript
reportHTML = `<!DOCTYPE html>
<html lang="ja">
<head>
  <style>/* レポート用CSS（インライン完結）*/</style>
</head>
<body>
  <div class="report">
    ${coverSection(d)}
    ${projectsSection(d)}
    ${policySection(d)}
    ${membersSection(d)}
    ${levelDefSection()}
    ${footer(d)}
  </div>
</body>
</html>`;
```

### 5-2. セクション分割（推奨リファクタリング）
現在は `generate()` に一括実装。
コードが長くなった場合は以下のように関数分割する:

| 関数名 | 役割 |
|--------|------|
| `buildCover(d)` | 表紙HTML生成 |
| `buildProjects(d)` | 予定案件テーブルHTML生成 |
| `buildPolicy(d)` | 育成方針カードHTML生成 |
| `buildMembers(d)` | メンバーカードHTML生成 |
| `buildLevelDef()` | スキルレベル定義テーブルHTML生成 |

### 5-3. XSSサニタイズ
ユーザー入力はすべて `escapeHTML()` を通してから埋め込む。

```javascript
function escapeHTML(str) {
  return String(str)
    .replace(/&/g,  '&amp;')
    .replace(/</g,  '&lt;')
    .replace(/>/g,  '&gt;')
    .replace(/"/g,  '&quot;')
    .replace(/'/g,  '&#039;');
}

// 使用例
`<td>${escapeHTML(proj.name)}</td>`
```

---

## 6. CSSアーキテクチャ

### アプリUI用CSS（`<style>` タグ内）
フォーム画面のスタイルを定義。クラスベースで管理。

カラーパレット（会社テーマカラー準拠）:
- メインカラー: `#02993B`（グリーン）
- アクセントカラー: `#F39346`（オレンジ）
- 背景色: `#FFFFFF`
- 文字色: `#212121`

```
.app            ─── 最外ラッパー（最大幅860px）
.app-header     ─── タイトルバー
.step-nav       ─── ステップタブコンテナ
.step-btn       ─── ステップタブ（active / done 修飾クラスあり）
.card           ─── 各ステップのコンテンツカード
.field-grid     ─── 2カラムグリッドレイアウト
.field          ─── ラベル+入力のセット
.tbl-wrap       ─── テーブルの横スクロールラッパー
.add-btn        ─── 行追加ボタン（緑系）
.del-btn        ─── 行削除ボタン（赤系）
.btn-pri        ─── 主要アクションボタン（ネイビー）
.btn-sec        ─── 補助アクションボタン（グレー）
.btn-gen        ─── レポート生成ボタン（ティール）
```

### レポート用CSS（生成HTML内インライン）
レポートHTMLが単体で表示できるよう、すべてのスタイルを
生成するHTML文字列の `<style>` タグ内に含める。

---

## 7. ブラウザ互換性

| ブラウザ | バージョン | 対応状況 |
|---------|-----------|---------|
| Chrome | 90以上 | ✅ 主要ターゲット |
| Edge | 90以上 | ✅ |
| Firefox | 88以上 | ✅ |
| Safari | 14以上 | ✅ |

使用する主要API:
- `document.getElementById` / `createElement` / `appendChild`
- テンプレートリテラル（ES6）
- `URL.createObjectURL` / `Blob`（ダウンロード）
- `iframe.srcdoc`（プレビュー）
- `Date.now()`（行ID生成）
- `Array.prototype.map / filter / splice`

IE11は対象外。

---

## 8. パフォーマンス特性

| 操作 | 所要時間の目安 |
|------|--------------|
| ファイルを開く | 即時（ネットワーク通信なし） |
| ステップ切り替え | 即時（DOM show/hide のみ） |
| レポート生成 | < 100ms（文字列組み立てのみ） |
| ダウンロード | < 200ms（Blob生成のみ） |

メンバー50名・案件20件の大規模データでも体感上の遅延は生じない。
