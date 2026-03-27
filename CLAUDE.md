# CLAUDE.md — スキル分析・育成計画 レポートジェネレーター

## プロジェクト概要

チームリーダーがフォームに入力したスキル分析・育成計画データを、
ブラウザ上でHTMLレポートとして出力・ダウンロードするツール。

**サーバー不要・API不要・インストール不要。HTMLファイル1つで完結。**

---

## 技術方針（最重要）

```
構成: HTMLファイル 1つ（index.html）
言語: HTML / CSS / Vanilla JavaScript（フレームワーク不使用）
依存: 外部CDN・npm・Python・Node.js すべて不使用
動作: ブラウザで直接開くだけ（file:// プロトコル対応）
```

### やってはいけないこと
- `fetch()` や XMLHttpRequest によるサーバー通信の追加
- `import` / `require` などのモジュール構文の使用
- React / Vue / Alpine.js などフレームワークの導入
- `localStorage` / `sessionStorage` の使用（機密データ保護のため）
- CDN からの外部スクリプト・スタイルシートの読み込み
- APIキーをコードに埋め込む行為

---

## ファイル構成

```
skill-report-app/
├── CLAUDE.md          # このファイル（Claude Codeへの指示書）
├── SPEC.md            # 機能仕様書
├── ARCHITECTURE.md    # 設計詳細
├── REQUIREMENT.md     # 要件定義書
└── index.html         # アプリ本体（すべてここに集約）
```

`index.html` の内部構造:

```
index.html
├── <style>          # アプリUI用CSS（フォーム画面）
├── <!-- Step 1 -->  # 基本情報フォーム
├── <!-- Step 2 -->  # 予定案件テーブル
├── <!-- Step 3 -->  # 育成方針フォーム
├── <!-- Step 4 -->  # メンバー育成目標テーブル
├── <!-- Preview --> # レポートプレビュー（iframe）
└── <script>         # 全ロジック（状態管理・レポート生成・DL）
```

---

## 開発の進め方

### 起動確認
```bash
# ブラウザで直接開くだけ
open index.html          # macOS
start index.html         # Windows
xdg-open index.html      # Linux
```

### 変更・デバッグ
- ブラウザの開発者ツール（F12）でコンソールエラーを確認
- 変更後はブラウザをリロード（Ctrl+R / Cmd+R）するだけ

---

## コーディング規約

### JavaScript
- `const` / `let` を使用（`var` 禁止）
- 関数は先頭に集約して定義する
- DOM操作は `document.getElementById()` を基本とする
- エラーは `console.error()` に出力し、ユーザーにはUI上で通知
- 型変換は明示的に行う（例: `parseInt(lv) || 0`）

### CSS
- カラーパレットは定数として冒頭にコメントでまとめる
- クラス名は `-` 区切りのケバブケース
- インラインスタイルはレポートHTML生成部分のみに限定
- アプリUI側はすべて `<style>` タグで管理

### HTML生成（レポート出力部分）
- テンプレートリテラル（バッククォート）を使用
- XSS対策として `escapeHTML()` 関数で必ずサニタイズしてから埋め込む
- 生成するHTMLもスタンドアロン完結（外部参照なし）

### セキュリティ
```javascript
// 必ずこの関数を通してHTMLに値を埋め込む
function escapeHTML(str) {
  return String(str)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;');
}
```

---

## 状態管理の方針

グローバル変数で管理する（シンプルさを優先）:

```javascript
let currentStep = 0;      // 現在表示中のステップ (0-3)
let projIds    = [];       // 案件行のID配列
let memberIds  = [];       // メンバー行のID配列
let reportHTML = '';       // 生成済みレポートHTML文字列
```

ステップ間のデータはDOMから直接読み取る（二重管理しない）。

---

## テスト方針

自動テストは導入しない（オーバーエンジニアリング防止）。
以下の手動確認チェックリストで品質を担保する。

### 機能チェック
- [ ] Step1〜4 のフォーム入力・ステップ遷移が正常に動作する
- [ ] 必須項目（年度・部門名・リーダー名）が空のとき Step2 に進めない
- [ ] 案件行・メンバー行の追加・削除が正常に動作する
- [ ] 「レポートを生成」ボタンで iframe にプレビューが表示される
- [ ] 「HTMLをダウンロード」でファイルが保存される
- [ ] ダウンロードしたHTMLをブラウザで開くと正しく表示される

### 表示チェック
- [ ] Chrome 最新版で表示崩れなし
- [ ] Edge 最新版で表示崩れなし
- [ ] 案件0件・メンバー0件でも生成できる（エラーにならない）
- [ ] 記号・特殊文字（<>"'&）を入力してもHTMLが壊れない

---

## よくある実装ミスと対処

| ミス | 正しい対処 |
|------|-----------|
| `innerHTML` に未サニタイズの値を埋め込む | `escapeHTML()` を必ず通す |
| テンプレートリテラルの閉じ忘れ | ネストに注意、インデントを揃える |
| 行削除後に古いIDが残る | `delRow()` で `arr.splice()` も必ず実行 |
| iframe の高さが足りない | `onload` で `scrollHeight` を取得して再設定 |
| 日本語ファイル名のDL失敗 | `encodeURIComponent` は不要（Blob URL使用のため） |
