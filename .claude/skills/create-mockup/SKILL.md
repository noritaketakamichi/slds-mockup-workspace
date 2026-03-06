---
name: create-mockup
description: >
  SLDS準拠のHTMLモックアップを生成する。
  以下の場面でこのスキルを使用する:
  (1) ユーザーがSalesforce画面のスクリーンショットを提供し、モックアップ作成を依頼した時
  (2) 「この画面を作って」「スクショからモックアップ」のように画面再現を求められた時
  (3) 既存モックアップをスクリーンショットに合わせて修正する時
  (4) 「一覧画面」「リスト」「テーブル画面」の作成を依頼された時
  (5) 既存のリストビューモックアップに新しいカラムやボタンを追加する時
  トリガーワード例: 「スクショ」「スクリーンショット」「画面を再現」「モックアップ作成」「この画面を作って」「一覧」「リスト」「リストビュー」「テーブル画面」
---

# Create Mockup

SLDS準拠のHTMLモックアップを生成するワークフロー。レコード詳細画面とリストビュー画面の両方に対応。

## リファレンス

### レコード詳細画面
- **テンプレート**: `.claude/templates/record-detail-page.html`
- **パーツ**: `.claude/templates/parts/` 配下
- **実装例**: `mockups/user-detail.html`, `mockups/invoice-detail.html`

### リストビュー画面
- **テンプレート**: `.claude/templates/list-view-page.html`
- **実装例**: `mockups/invoice_list/invoice-list.html`（請求書一覧）

---

## ワークフロー

### Step 1: 画面タイプ判定

提供された画像や要件から画面タイプを判定する:
- **レコード詳細画面**: Highlights Panel、Path、2カラムレイアウトがある → 「レコード詳細画面」セクションへ
- **リストビュー画面**: ツールバー + データテーブルがある → 「リストビュー画面」セクションへ

### Step 2: テンプレート参照

画面タイプに応じたテンプレートとリファレンス実装を読み込む。

### Step 3: HTML生成

以下のルールを厳守:
- **アイコン**: 絵文字は絶対に使わない。すべてインラインSVG。Material Iconsのpathデータ活用可
- **ボタングループ**: `gap: 0` + `margin-left: -1px` で隙間なく詰める。最左だけ左角丸、最右だけ右角丸
- **グローバルヘッダー**: `#032D60` 紺色バー、SVGクラウドロゴ、検索バー
- **JavaScript**: `MockupRouter` ユーティリティ、`data-tab` タブ切り替え、`toggleSection(id)` 折りたたみ

### Step 4: 出力と確認

1. `mockups/{機能名}/` または `mockups/` に保存
2. 元画像との差異がないか確認し、差異があればリストアップして報告

---

## レコード詳細画面

### 画面構造（上から順に）

1. **グローバルヘッダー**: `#032D60` 紺色バー
2. **グローバルナビゲーション**: `slds-context-bar`
3. **レコードタブバー**: `slds-context-bar__item_tab`
4. **Highlights Panel**: `slds-page-header_record-home` — オブジェクトアイコン + レコード名 + ハイライト項目 + アクションボタン
5. **Path**: `slds-path` — ステータス遷移ステップバー（ある場合のみ）
6. **レコード本体**: 左カラム(~65%) 関連リスト + 右カラム(~35%) 詳細パネル

### テンプレート参照先
- `.claude/templates/record-detail-page.html`
- `.claude/templates/parts/highlights-panel.html`
- `.claude/templates/parts/path.html`
- `.claude/templates/parts/record-body.html`
- `.claude/templates/parts/related-list.html`

---

## リストビュー画面

### 画面構造（上から順に）

#### 1. グローバルヘッダー
- `#032D60` 紺色バー（高さ40px）
- Salesforceクラウドロゴ（SVG） + 検索バー + ユーティリティアイコン + アバター

#### 2. グローバルナビゲーション (`slds-context-bar`)
- App Launcher（ワッフルアイコン）+ アプリ名
- ナビタブ一覧（現在のオブジェクトに `slds-is-active`）

#### 3. リストビューツールバー
2段構成:

**上段 (row1)**:
- 左: オブジェクトアイコン（32px角丸四角、色付き背景 + 白SVG） + オブジェクトラベル + ビュー名 + ▼ドロップダウン + ピンアイコン
- 右: アクションボタン群（`slds-button_neutral` のみ、隙間なしグループ）

**下段 (row2)**:
- フィルター情報テキスト（レコード数、ソート基準、検索条件、更新時間）

#### 4. データテーブル
- `slds-table_cell-buffer slds-table_bordered slds-table_fixed-layout`
- 横スクロール対応: `overflow-x: auto` + `min-width`
- 先頭列: チェックボックス（全選択対応）
- 2列目: # (行番号)
- 各カラムヘッダー: ソート用▼矢印付き
- 最終列: アクションボタン（▼ドロップダウン）

### アクションボタン群のルール

- **全て `slds-button_neutral`** を使用（`slds-button_brand` は使わない）
- ボタングループ: `gap: 0` + `margin-left: -1px` で隙間なく詰める
- 最左だけ左角丸 `border-radius: 4px 0 0 4px`
- 最右だけ右角丸 `border-radius: 0 4px 4px 0`
- フォントサイズ: `0.75rem`、高さ: `30px`

### カラム型パターン

#### テキスト列
```html
<td><div class="slds-truncate">テキスト</div></td>
```

#### リンク列
```html
<td><div class="slds-truncate"><a href="#">リンクテキスト</a></div></td>
```

#### Boolean列（チェックボックスアイコン）
```html
<!-- true: 緑チェック -->
<td class="col-bool"><div class="bool-check bool-check--true"><svg viewBox="0 0 24 24"><path d="M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm-9 14l-5-5 1.41-1.41L10 14.17l7.59-7.59L19 8l-9 9z"/></svg></div></td>

<!-- false: グレー空チェック -->
<td class="col-bool"><div class="bool-check bool-check--false"><svg viewBox="0 0 24 24"><path d="M19 5v14H5V5h14m0-2H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2z"/></svg></div></td>
```

#### ステータスバッジ列
```html
<td><div class="slds-truncate"><span class="status-badge status-badge--draft">下書き</span></div></td>
```
バリエーション: `--draft`(グレー), `--active`(青), `--success`(緑), `--error`(赤)

#### 金額列（右寄せ）
```html
<td><div class="slds-truncate amount-col">&yen;14,000</div></td>
```

#### カラムヘッダー（ソート対応）
```html
<th scope="col" style="width: 130px;">
  <a class="slds-th__action slds-text-link_reset" href="#">
    <div class="slds-grid slds-grid_vertical-align-center slds-has-flexi-truncate">
      <span class="slds-truncate" title="カラム名">カラム名</span>
      <svg viewBox="0 0 24 24" style="width:10px; height:10px; fill:#706e6b; margin-left:0.25rem;"><path d="M7 10l5 5 5-5z"/></svg>
    </div>
  </a>
</th>
```

---

## チェックリスト（共通）

- [ ] SLDSのCDNが読み込まれているか
- [ ] 絵文字が一切使われていないか（全SVGか）
- [ ] ボタングループが隙間なく詰まっているか
- [ ] `MockupRouter` が組み込まれているか

### レコード詳細画面の追加チェック
- [ ] タブ切り替え・セクション折りたたみが動作するか
- [ ] `buttonHandlers` が組み込まれているか

### リストビューの追加チェック
- [ ] アクションボタンが全て `slds-button_neutral` か
- [ ] カラムヘッダーにソート用▼があるか
- [ ] 全選択チェックボックスが機能するか
