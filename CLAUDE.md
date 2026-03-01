# SLDS モックアップ生成プロジェクト

## デザインシステム

- UIモックアップは必ず Salesforce Lightning Design System (SLDS) に準拠すること
- CSSクラスは `slds-` プレフィックスを使用（例: `slds-button`, `slds-card`）
- 公式リファレンス: https://www.lightningdesignsystem.com/
- カラーはSLDSデザイントークンを使用すること（例: `--slds-g-color-brand-base-50`）

## モックアップ生成ルール

- HTMLモックアップを作る際は、SLDSのCDNを読み込むこと:
  `<link rel="stylesheet" href="https://unpkg.com/@salesforce-ux/design-system/assets/styles/salesforce-lightning-design-system.min.css">`
- レイアウトは `slds-grid` / `slds-col` を使うこと
- アイコンは SLDS Utility Icons を使うこと
- モバイル対応は `slds-size_1-of-1 slds-medium-size_1-of-2` 等のグリッドで行う

## ★ Salesforce標準レコードページの構造（最重要）

モックアップは実際のSalesforceの画面構造を正確に再現すること。上から順に以下の構造:

### 1. グローバルナビゲーション (`slds-context-bar`)
- 紺色の横バー。左にアプリランチャー（9ドットアイコン）+ アプリ名
- 右にナビゲーションタブ（オブジェクト名が並ぶ）
- テンプレート: `.claude/templates/parts/global-nav.html`

### 2. レコードタブバー (`slds-context-bar__item_tab`)
- 現在開いているレコードのタブ（例: "IV-589706 | 請求書"）
- 編集中は `*` プレフィックス、閉じるボタン付き

### 3. レコードヘッダー / Highlights Panel (`slds-page-header slds-page-header_record-home`)
- 上段: オブジェクトアイコン（色付き丸 + SVGアイコン）+ オブジェクトラベル + レコード名
- 中段: ハイライト項目（キー項目を横一列に表示）`slds-page-header__detail-row` > `slds-page-header__detail-block`
- 右上: アクションボタン群
- テンプレート: `.claude/templates/parts/highlights-panel.html`

### 4. Path コンポーネント (`slds-path`) ※ステータス遷移がある場合
- 横長のステップバー。完了=チェック付き、現在=青塗り、未到達=グレー
- テンプレート: `.claude/templates/parts/path.html`

### 5. レコードページ本体 (2カラムレイアウト)
- **左カラム (~65%)**: 関連リスト（Related Lists）を縦に積む
  - 各関連リスト: アイコン + タイトル(件数) + リフレッシュボタン + テーブル
- **右カラム (~35%)**: 詳細パネル
  - タブ: 概要 | 活動 | Chatter
  - 概要タブ: フィールド名 + 値の縦並びリスト
- テンプレート: `.claude/templates/parts/record-body.html`

### 6. 関連リストの構造
- ヘッダー: 色付きアイコン + タイトル(件数) + ソート情報 + リフレッシュボタン
- テーブル: ソート可能なカラムヘッダー（ドロップダウン矢印付き）
- テンプレート: `.claude/templates/parts/related-list.html`

## コンポーネント命名規則

- LWCコンポーネント名: camelCase（例: accountDetailCard）
- ファイル構成: componentName/componentName.html, .js, .css

## テンプレート

- **レコード詳細画面**: `.claude/templates/record-detail-page.html` をベースにすること（★推奨）
- パーツテンプレート: `.claude/templates/parts/` 内の各パーツを参考にすること
- 一覧系: `.claude/templates/list-page.html`
- モーダル: `.claude/templates/modal-pattern.html`
- ベースHTML: `.claude/templates/mockup-base.html`
- デザイントークン: `.claude/templates/design-tokens.md`

## 参照URL

- Global Navigation: https://www.lightningdesignsystem.com/components/global-navigation/
- Path: https://www.lightningdesignsystem.com/components/path/
- Page Header (Record Home): https://www.lightningdesignsystem.com/components/page-headers/
- Data Table: https://www.lightningdesignsystem.com/components/data-tables/
- Modal: https://www.lightningdesignsystem.com/components/modals/
- Card: https://www.lightningdesignsystem.com/components/cards/
- Form Element: https://www.lightningdesignsystem.com/components/form-element/
- Button: https://www.lightningdesignsystem.com/components/buttons/
- Tabs: https://www.lightningdesignsystem.com/components/tabs/

## アイコンルール（★重要）

- **絵文字(emoji)は絶対に使わないこと** — すべてのアイコンはインラインSVGで描画する
- 編集ペンシルアイコン: `<svg viewBox="0 0 24 24"><path d="M3 17.25V21h3.75L17.81 9.94l-3.75-3.75L3 17.25zM20.71 7.04a1 1 0 000-1.41l-2.34-2.34a1 1 0 00-1.41 0l-1.83 1.83 3.75 3.75 1.83-1.83z"/></svg>`
- ユーザーアイコン: `<svg viewBox="0 0 24 24"><path d="M12 12c2.21 0 4-1.79 4-4s-1.79-4-4-4-4 1.79-4 4 1.79 4 4 4zm0 2c-2.67 0-8 1.34-8 4v2h16v-2c0-2.66-5.33-4-8-4z"/></svg>`
- 歯車アイコン: `<svg viewBox="0 0 24 24"><path d="M19.14 12.94c.04-.3.06-.61.06-.94...（略）"/></svg>`
- ▼ドロップダウン: `<svg viewBox="0 0 24 24"><path d="M7 10l5 5 5-5z"/></svg>`
- その他のアイコンも同様にSVGで描画。Material Icons の path データを利用してよい

## ボタングループのパターン

Salesforceのアクションボタン群（コピー, 編集, オーダー作成 等）は隙間なく詰めて表示する:
```css
.highlights-panel__actions { display: flex; gap: 0; }
.highlights-panel__actions .slds-button_neutral,
.highlights-panel__actions .slds-button_outline-brand {
  border-radius: 0;
  margin-left: -1px;  /* ボーダー重ね */
}
/* 最初のボタンだけ左角丸、最後のボタンだけ右角丸 */
```
- 人型アイコンボタンはグループの左側に `margin-right` で少し離す
- ▼ドロップダウンボタンはグループ最右端に付ける

## グローバルヘッダーの構成

- 最上部に `#032D60` の紺色バー（高さ40px）
- 左: Salesforceクラウドロゴ（SVG）
- 中央: 検索バー（半透明白背景）
- 右: "Partners" バッジ（`#0070d2` 背景） → ユーティリティアイコン群（SVG） → ユーザーアバター
- 全ユーティリティアイコンは `fill: rgba(255,255,255,0.8)` の白SVG

## JavaScript 構造ガイドライン

画面遷移を後から模擬できるよう、以下の構造を初期から組み込むこと:

1. **ボタンハンドラはオブジェクトで管理**: 各ボタンに `id` を付与し、`buttonHandlers` オブジェクトにまとめる
2. **MockupRouter ユーティリティ**: `navigateTo(url)`, `showModal(id)`, `hideModal(id)` を用意
3. **タブ切り替え**: `data-tab` 属性 + イベント委譲パターンで実装
4. **セクション折りたたみ**: `toggleSection(id)` 関数 + `is-hidden` / `is-collapsed` クラスで制御

## 既存モックアップ（参考）

- `mockups/user-detail.html` — ユーザーレコード詳細画面（テスト太郎）
- `mockups/invoice-detail.html` — 請求書レコード詳細画面
- `mockups/delivery-plan-detail.html` — 配送計画詳細画面

新しいモックアップを作る際は既存ファイルの構造を参考にすること。

## 出力ルール

- モックアップHTMLは指定ディレクトリ、または `mockups/` ディレクトリに保存すること
- ファイル名: `{機能名}-{画面種別}.html`（例: delivery-list.html）
