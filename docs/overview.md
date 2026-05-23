# Task Board アプリ 仕様書

## 目次

1. [アプリ概要](#1-アプリ概要)
2. [技術スタック](#2-技術スタック)
3. [ディレクトリ構成](#3-ディレクトリ構成)
4. [機能仕様](#4-機能仕様)
5. [データ設計](#5-データ設計)
6. [コンポーネント設計](#6-コンポーネント設計)
7. [スタイル設計](#7-スタイル設計)
8. [ローカルストレージ仕様](#8-ローカルストレージ仕様)
9. [デプロイ構成](#9-デプロイ構成)
10. [開発環境のセットアップ](#10-開発環境のセットアップ)
11. [改修・機能追加ガイド](#11-改修機能追加ガイド)

---

## 1. アプリ概要

シンプルなタスク管理ボードアプリ。テキスト入力でタスクを追加・管理でき、ブラウザのローカルストレージにデータを保存するため、ページをリロードしてもタスクが消えない。

- **公開URL**: `https://katsu84mt.github.io/task-board/`
- **リポジトリ**: `https://github.com/katsu84mt/task-board`

---

## 2. 技術スタック

| 種別 | 技術 | バージョン |
|---|---|---|
| UIフレームワーク | React | ^19.2.6 |
| ビルドツール | Vite | ^8.0.12 |
| 言語 | JavaScript (JSX) | — |
| スタイリング | CSS (通常CSSファイル) | — |
| データ永続化 | ブラウザ localStorage | — |
| CI/CD | GitHub Actions | — |
| ホスティング | GitHub Pages | — |

外部UIライブラリ・状態管理ライブラリは使用していない。React 標準の `useState` / `useEffect` のみで実装している。

---

## 3. ディレクトリ構成

```
task-board/
├── .github/
│   └── workflows/
│       └── deploy.yml        # GitHub Actions デプロイ設定
├── docs/
│   └── overview.md           # 本仕様書
├── public/
│   ├── favicon.svg           # ファビコン
│   └── icons.svg             # SVGアイコン定義
├── src/
│   ├── assets/               # 静的アセット（画像など）
│   ├── App.jsx               # メインコンポーネント（全ロジック）
│   ├── App.css               # App コンポーネント用スタイル
│   ├── index.css             # グローバルスタイル
│   └── main.jsx              # エントリーポイント
├── CLAUDE.md                 # Claude Code 向けプロジェクト設定
├── index.html                # HTMLエントリーポイント
├── package.json              # 依存関係・スクリプト定義
├── eslint.config.js          # ESLint設定
└── vite.config.js            # Vite設定
```

---

## 4. 機能仕様

### 4.1 タスク追加

- 画面上部のテキスト入力欄にタスク名を入力し、「追加」ボタンまたは **Enter キー** で追加できる
- 入力が空白のみの場合は追加されない（`input.trim()` で検証）
- 追加後、入力欄は空になる
- タスクはリストの末尾に追加される

### 4.2 完了・未完了の切り替え

- 各タスクの左側にあるチェックボックスをクリックで完了・未完了を切り替えられる
- 何度でもトグルできる

### 4.3 完了済みタスクの視覚的区別

完了済みタスク（`done: true`）には以下のスタイルが適用される：

| プロパティ | 未完了 | 完了済み |
|---|---|---|
| 背景色 | `#fafafa` | `#f3f4f6`（グレー） |
| 透明度 | 1.0 | 0.6 |
| テキスト色 | `#374151` | `#9ca3af`（グレー） |
| テキスト装飾 | なし | `line-through`（打ち消し線） |

### 4.4 タスク削除

- 各タスクの右端にある「✕」ボタンをクリックでタスクを削除できる
- 確認ダイアログはなく即時削除される
- ✕ボタンはホバー時に赤色に変化する

### 4.5 件数カウンター

- リストの右下に `{完了数} / {全体数} 件完了` の形式で表示される
- タスクの追加・削除・完了切り替えにリアルタイムで連動する

### 4.6 空リスト表示

- タスクが0件のとき、リストの代わりに「タスクがありません」のメッセージを表示する

---

## 5. データ設計

### タスクオブジェクト

```js
{
  id:   number,   // ユニークID（既存タスクの最大ID + 1 で採番）
  text: string,   // タスクのテキスト
  done: boolean,  // 完了フラグ
}
```

### 状態（State）

| 変数名 | 型 | 初期値 | 説明 |
|---|---|---|---|
| `tasks` | `Task[]` | localStorageから復元、なければ `[]` | タスク一覧 |
| `input` | `string` | `""` | テキスト入力欄の現在値 |

### ID採番ルール

```js
const newId = tasks.length > 0 ? Math.max(...tasks.map((t) => t.id)) + 1 : 1
```

既存タスクの最大IDに +1 することで、削除後も重複しないIDを保証する。

---

## 6. コンポーネント設計

現在はコンポーネント分割を行わず、`App.jsx` 1ファイルにすべてのロジックとUIを実装している。

### App コンポーネント

**ファイル**: `src/App.jsx`

#### ハンドラ関数

| 関数名 | 処理内容 |
|---|---|
| `addTask()` | 入力値を検証してタスクを追加し、入力欄をクリアする |
| `toggleTask(id)` | 指定IDのタスクの `done` フラグを反転する |
| `deleteTask(id)` | 指定IDのタスクを配列から除外する |
| `handleKeyDown(e)` | Enter キー押下時に `addTask()` を呼び出す |

#### 副作用（useEffect）

```js
useEffect(() => {
  localStorage.setItem('tasks', JSON.stringify(tasks))
}, [tasks])
```

`tasks` が更新されるたびに localStorage へ同期保存する。

#### JSX構造

```
<div.board>
  <h1.board-title>
  <div.input-row>
    <input.task-input>
    <button.add-btn>
  <p.empty>  ← タスクが0件のときのみ表示
  <ul.task-list>
    <li.task-item [.done]>
      <input[checkbox].task-checkbox>
      <span.task-text>
      <button.delete-btn>
  <div.stats>
```

---

## 7. スタイル設計

### ファイル構成

| ファイル | 役割 |
|---|---|
| `src/index.css` | body・#root など全体に適用するリセット・ベーススタイル |
| `src/App.css` | ボード・タスクアイテムなどアプリ固有のスタイル |

### 主要なCSSクラス一覧

| クラス名 | 要素 | 説明 |
|---|---|---|
| `.board` | `div` | カード全体のコンテナ（最大幅560px、中央寄せ、白背景） |
| `.board-title` | `h1` | アプリタイトル |
| `.input-row` | `div` | 入力欄と追加ボタンの横並びコンテナ |
| `.task-input` | `input` | テキスト入力欄（フォーカス時にボーダーが紫に変化） |
| `.add-btn` | `button` | 追加ボタン（紫背景、ホバーで濃くなる） |
| `.task-list` | `ul` | タスク一覧（縦並び、gap 10px） |
| `.task-item` | `li` | タスク1件のコンテナ |
| `.task-item.done` | `li` | 完了済みタスク（グレー背景、opacity 0.6） |
| `.task-checkbox` | `input[checkbox]` | チェックボックス（accent-color: 紫） |
| `.task-text` | `span` | タスクのテキスト |
| `.delete-btn` | `button` | 削除ボタン（ホバーで赤くなる） |
| `.empty` | `p` | タスク0件時のメッセージ |
| `.stats` | `div` | 右下の完了カウンター |

### カラーパレット

| 用途 | カラーコード |
|---|---|
| アクセントカラー（ボタン・チェック） | `#4f46e5`（インディゴ） |
| アクセントカラー（ホバー） | `#4338ca` |
| 削除ボタンホバー | `#ef4444`（赤） |
| 背景（ページ全体） | `#f0f2f5`（薄いグレー） |
| 背景（カード） | `#ffffff` |
| 背景（完了タスク） | `#f3f4f6` |
| テキスト（通常） | `#374151` |
| テキスト（完了・補足） | `#9ca3af` |

---

## 8. ローカルストレージ仕様

| キー | 値の型 | 説明 |
|---|---|---|
| `tasks` | `string`（JSON） | タスク配列を JSON.stringify したもの |

### 読み込み（初期化時）

```js
useState(() => {
  try {
    return JSON.parse(localStorage.getItem('tasks')) ?? []
  } catch {
    return []
  }
})
```

- `localStorage` に値がない場合は `[]` を返す
- JSON のパースに失敗した場合も `[]` にフォールバックする（破損データへの耐性）

### 書き込み（更新時）

- `tasks` state が変化するたびに `useEffect` で自動的に保存される
- 手動での保存操作は不要

---

## 9. デプロイ構成

### Vite ビルド設定

**ファイル**: `vite.config.js`

```js
base: '/task-board/'
```

GitHub Pages のサブパス（`/task-board/`）に合わせて `base` を設定している。この設定により、ビルド成果物のアセットパスが `/task-board/assets/...` になる。

### GitHub Actions ワークフロー

**ファイル**: `.github/workflows/deploy.yml`

| トリガー | `main` ブランチへの push |
|---|---|
| ビルド環境 | Ubuntu latest、Node.js 20 |
| ビルドコマンド | `npm ci && npm run build` |
| デプロイ先 | GitHub Pages（`dist/` ディレクトリ） |

**ジョブ構成**:

1. **build ジョブ**: コードをチェックアウト → `npm ci` → `npm run build` → `dist/` をアーティファクトとしてアップロード
2. **deploy ジョブ**: build ジョブ完了後、アーティファクトを GitHub Pages へデプロイ

同時実行制御（`concurrency`）により、同一ブランチへの連続 push 時は前のデプロイをキャンセルして最新のもののみ実行する。

---

## 10. 開発環境のセットアップ

### 前提条件

- Node.js 20 以上
- npm 9 以上

### 手順

```bash
# リポジトリのクローン
git clone https://github.com/katsu84mt/task-board.git
cd task-board

# 依存関係のインストール
npm install

# 開発サーバーの起動
npm run dev
# → http://localhost:5173 でアクセス可能

# プロダクションビルド
npm run build

# ビルド結果のプレビュー
npm run preview
```

---

## 11. 改修・機能追加ガイド

### タスクにフィールドを追加する場合

`addTask()` 関数内のタスクオブジェクト生成部分を変更する。

```js
// 例: 優先度フィールドを追加
setTasks([...tasks, { id: newId, text, done: false, priority: 'normal' }])
```

ローカルストレージは自動で新しいフィールドを保持するが、既存の保存データには新フィールドが存在しないため、参照時に `task.priority ?? 'normal'` のようなデフォルト値の考慮が必要。

### フィルタリング（全件・未完了・完了済み）を追加する場合

1. `useState` でフィルター状態を追加: `const [filter, setFilter] = useState('all')`
2. 表示するタスクをフィルター: `const visibleTasks = tasks.filter(...)` を定義
3. JSXの `tasks.map(...)` を `visibleTasks.map(...)` に変更
4. フィルター切り替えボタンを `.input-row` の下に追加

### タスクの編集機能を追加する場合

1. 編集中のタスクIDを管理する state を追加: `const [editingId, setEditingId] = useState(null)`
2. `.task-text` をクリックしたとき `editingId` をセットし、`<input>` に差し替える
3. `updateTask(id, newText)` 関数を追加して `tasks` を更新

### コンポーネントを分割する場合

現在は `App.jsx` にすべてが集約されているが、規模が大きくなった場合の分割例：

```
src/
├── App.jsx               # ルートコンポーネント（stateとハンドラのみ）
├── components/
│   ├── TaskInput.jsx     # 入力欄と追加ボタン
│   ├── TaskList.jsx      # タスク一覧
│   ├── TaskItem.jsx      # タスク1件
│   └── TaskStats.jsx     # 件数カウンター
```

### スタイルを変更する場合

- **色の変更**: `src/App.css` 内の該当カラーコードを直接変更する
- **レイアウトの変更**: `.board` の `max-width` や `padding` を変更する
- **アニメーションの追加**: `.task-item` に `transition` プロパティを追加する
