# Claude CodeでSLDSモックアップを効率的に作るための事前準備

素晴らしい質問です。ポイントは**Claude Codeが毎回参照する「コンテキスト」を事前に仕込んでおく**ことです。大きく3つのレイヤーで準備すると効果的です。

---

## 1. `CLAUDE.md` に SLDS のルールを書き込む（最重要）

Claude Codeはプロジェクトルートの `CLAUDE.md` を毎回自動で読みます。ここにSLDSの設計ルールを書いておくのが最もインパクトが大きいです。

```markdown
# CLAUDE.md（例）

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

## コンポーネント命名規則

- LWCコンポーネント名: camelCase（例: accountDetailCard）
- ファイル構成: componentName/componentName.html, .js, .css

## よく使うパターン

- 一覧画面 → slds-card + slds-table
- 詳細画面 → slds-card + slds-form (stacked)
- モーダル → slds-modal + slds-backdrop
```

これだけで `「アカウント一覧画面のモックアップを作って」` という短いプロンプトでもSLDS準拠のHTMLが出てきます。

---

## 2. テンプレートファイルを置いておく

プロジェクト内に参照用のテンプレートを用意し、`CLAUDE.md` から参照させます。

```
project-root/
├── CLAUDE.md
├── .claude/
│   └── templates/
│       ├── mockup-base.html      ← SLDS CDN読み込み済みのベースHTML
│       ├── list-page.html        ← 一覧画面テンプレート
│       ├── detail-page.html      ← 詳細画面テンプレート
│       ├── modal-pattern.html    ← モーダルパターン
│       └── design-tokens.md      ← プロジェクト固有の色・spacing定義
```

**`mockup-base.html` の例:**

```html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>{{PAGE_TITLE}}</title>
    <link
      rel="stylesheet"
      href="https://unpkg.com/@salesforce-ux/design-system/assets/styles/salesforce-lightning-design-system.min.css"
    />
    <style>
      body {
        font-family: "Salesforce Sans", Arial, sans-serif;
      }
      .app-container {
        max-width: 1280px;
        margin: 0 auto;
        padding: 1rem;
      }
    </style>
  </head>
  <body>
    <div class="slds-scope">
      <div class="app-container">
        <!-- CONTENT HERE -->
      </div>
    </div>
  </body>
</html>
```

`CLAUDE.md` に以下を追記:

```markdown
## テンプレート

- モックアップ作成時は `.claude/templates/mockup-base.html` をベースにすること
- 一覧系は `.claude/templates/list-page.html` を参考にすること
```

---

## 3. プロジェクト固有の用語・オブジェクト定義を渡す

Salesforce SaaS開発では**ドメイン知識**が重要です。これも `CLAUDE.md` かファイルに書いておきます。

```markdown
## ビジネスドメイン

- 本プロダクトは「物流管理SaaS」である
- 主要オブジェクト:
  - 配送依頼 (DeliveryRequest\_\_c): 顧客からの配送依頼
  - 配送計画 (DeliveryPlan\_\_c): 配送ルートの計画
  - ドライバー (Driver\_\_c): 配送担当者
- 画面の日本語ラベルは上記オブジェクト名を使うこと
```

---

## 実際の使用イメージ

上記を準備しておくと、こんな短いプロンプトで済みます:

| プロンプト                     | 出力                              |
| ------------------------------ | --------------------------------- |
| `配送依頼の一覧画面`           | SLDS table + card のHTML          |
| `ドライバー割当モーダル`       | SLDS modal パターンのHTML         |
| `配送計画の詳細画面、地図付き` | stacked form + 地図プレースホルダ |

---

## 補足Tips

**コンポーネントカタログのURLを貼る**: 特定のSLDSコンポーネントの仕様が複雑な場合、`CLAUDE.md` にURLを書いておくと、Claude Codeが `web_fetch` で参照できます。

```markdown
## 参照URL

- Data Table: https://www.lightningdesignsystem.com/components/data-tables/
- Modal: https://www.lightningdesignsystem.com/components/modals/
```

**出力先を指定する**: モックアップの保存先も決めておくと便利です。

```markdown
## 出力ルール

- モックアップHTMLは `mockups/` ディレクトリに保存すること
- ファイル名: `{機能名}-{画面種別}.html`（例: delivery-list.html）
```

---

要するに、**「CLAUDE.md にSLDSルール + テンプレートファイル + ドメイン知識」の3点セット**を用意しておくことで、短いプロンプトでも一貫性のあるモックアップが生成されるようになります。一度セットアップすればチーム全体で恩恵を受けられるので、投資対効果はかなり高いです。
