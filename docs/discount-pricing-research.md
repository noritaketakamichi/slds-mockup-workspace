# 受注管理における割引・プライシング設計リサーチ

> 塾向け基幹システムの商品受注管理機能において、複数割引の適用（初月無料・兄弟割引・社員割引等）を実現するためのUI/UX設計リファレンス

**調査日**: 2026-03-03

---

## 目次

1. [ユースケース整理](#1-ユースケース整理)
2. [SaaS課金プラットフォームの割引モデル](#2-saas課金プラットフォームの割引モデル)
3. [ECサブスクリプションサービスの事例](#3-ecサブスクリプションサービスの事例)
4. [塾業界の割引制度の実態](#4-塾業界の割引制度の実態)
5. [UI/UXデザインパターン（管理画面）](#5-uiuxデザインパターン管理画面)
6. [UI/UXデザインパターン（顧客向け）](#6-uiuxデザインパターン顧客向け)
7. [割引適用ロジックの設計パターン](#7-割引適用ロジックの設計パターン)
8. [設計への提言](#8-設計への提言)
9. [参考URL一覧](#9-参考url一覧)

---

## 1. ユースケース整理

今回対象とするユースケース:

| # | ケース | 例 |
|---|--------|-----|
| 1 | **初月無料** | キャンペーンで初月の授業料を0円にする |
| 2 | **継続割引** | 次月以降も兄弟割引（5%OFF）が毎月適用される |
| 3 | **条件付き割引** | 社員・母子家庭など特定属性のユーザーにのみ適用 |
| 4 | **複数割引の重複適用** | 兄弟割引 + 社員割引 + 初月無料が同時に適用される |

---

## 2. SaaS課金プラットフォームの割引モデル

### 2.1 Stripe Billing

**無料トライアル / 初月無料**
- `trial_period_days` または `trial_end` パラメータでサブスクリプションに最大730日のトライアルを設定可能
- トライアル中は$0の請求書が生成され「Free trial」行が表示される
- 決済手段の収集を遅延可能（`payment_method_collection=if_required`）
- トライアル終了時の挙動を設定可能（キャンセル or 一時停止）

**複数割引のスタッキング**
- 2025年より、単一の`coupon`パラメータに代わり `discounts` 配列（最大20件）をサポート
- 各割引はサブスクリプション内の全アイテムに適用される
- 制限: 同一クーポンの重複不可、同一クーポン由来のプロモーションコードとの併用不可

**条件付き割引（Script Coupons）**
- 2025年導入: カスタムロジック（コード）で割引額を動的に計算する「Script Coupons」
- 例: 年払いのみ10%OFF、$1000上限の20%OFF、10席以上なら$100OFF（それ以外$30OFF）
- メタデータベースでの高価値顧客ターゲティングも可能

> **参考**: [Stripe Trials](https://docs.stripe.com/billing/subscriptions/trials) / [Stripe Coupons](https://docs.stripe.com/billing/subscriptions/coupons) / [Script Coupons](https://docs.stripe.com/billing/subscriptions/script-coupons)

---

### 2.2 Chargebee

**無料トライアル**
- プランごとにトライアル日数/月数を設定可能
- 有料のディスカウントトライアル（初月500円等）はネイティブ未対応で、プロモーション用アドオンでのワークアラウンドが必要

**複数割引のスタッキング**
- Settings > Billing LogIQ で複数クーポン機能を有効化
- **適用順**: 定額クーポン → パーセンテージベースの手動割引
- サブスクリプションあたり最大10件の手動割引（ラインアイテム + 請求書レベル合計）
- クーポンと手動割引は同時に適用可能

**条件付き割引**
- 顧客セグメント: 「新規顧客」クーポン（過去の有効な請求書がない） vs. 「既存顧客」クーポン
- ドリルダウン適用: 特定プラン、特定アドオン、特定価格帯、特定通貨に限定可能
- ラインアイテムレベルでの割引適用可

> **参考**: [Chargebee Coupons](https://www.chargebee.com/docs/billing/2.0/product-catalog/coupons) / [Multiple Coupons](https://www.chargebee.com/docs/billing/2.0/kb/product-catalog/applying-multiple-coupons-and-manual-discounts-to-a-subscription)

---

### 2.3 Recurly

**複数割引のスタッキング（最も柔軟な設定）**
- 「Multiple Coupons per Account」機能: 顧客は複数クーポンを利用でき、全てが各請求書に適用される
- **適用順設定**:
  - 「パーセンテージ先」 or 「定額先」を選択可能
- **パーセンテージ割引モード**:
  - **Full Line Item Amount**: 各パーセンテージ割引がライン全額に対して順次適用
  - **Compound**: 前の割引適用後の残額に対して次の割引を適用

**計算例: $100の商品に10%(A) + 50%(B) のクーポン**

| モード | 割引A | 割引B | 合計割引額 |
|--------|-------|-------|-----------|
| Full Line Item | $10 (10% of $100) | $50 (50% of $100) | **$60** |
| Compound | $10 (10% of $100) | $45 (50% of $90) | **$55** |

> **参考**: [Recurly Multiple Coupons](https://docs.recurly.com/docs/multiple-coupons-per-account) / [How Coupons Applied](https://support.recurly.com/hc/en-us/articles/360023026532)

---

### 2.4 Zuora（最も高機能）

**無料トライアル**
- 同一プロダクトレートプラン内に「100%割引チャージ（1期間のみ）」を設定
- Contract Effective Date（トライアル開始）とService Activation Date（有料課金開始）の2つのトリガー日

**複数割引のスタッキング**
- **2つのモード**:
  - **Sequential（デフォルト）**: 各割引が前の割引適用後の残額に対して適用
  - **Stacked**: 全割引パーセンテージを合算し、元の金額に一括適用
- **Discount Classes**: 優先順位付けシステム。上位クラスの割引が先に適用。同一クラス内ではスタック割引を合算→非スタック割引を順次適用
- **Nested Discount Rows（CPQ X）**: 見積UIでプロダクト行の下に割引行をネストして表示

**計算例: $100に対して5%, 10%, 15%のスタック割引**
→ 合計30% = $30の割引

> **参考**: [Zuora Discount Charge Models](https://knowledgecenter.zuora.com/Zuora_Billing/Build_products_and_prices/Basic_concepts_and_terms/B_Charge_Models/B_Discount_Charge_Models) / [Enhanced Discount Use Cases](https://knowledgecenter.zuora.com/Zuora_Billing/Build_products_and_prices/Use_cases_of_charge_models/D_Enhanced_discount_use_cases)

---

## 3. ECサブスクリプション・プロモーションプラットフォームの事例

### 3.1 Kibo Commerce（レイヤーベースのスタッキングモデル）

最も明示的な管理UIを持つプラットフォームの一つ:

- **Stacking Layers**: 各割引に番号付き「レイヤー」を割り当て。レイヤーごとに注文金額を再計算してから次のレイヤーの割引を適用
- **グリッドベース管理**: 割引一覧に「Stacking Layer」「Stackable」カラムがあり、トグルで表示/非表示を切り替え可能
- **高度なフィルタ**: 「Stackable」（Yes/No）と「Stacking Layer」（数値）でフィルタリング可能
- **トグルパターン**: 個別の割引ごとに「Stackable」トグルとレイヤードロップダウンセレクターを配置

> **参考**: [Kibo Commerce Stackable Discounts](https://docs.kibocommerce.com/help/stackable-discounts)

### 3.2 commercetools（ランク付きスタッキングモード）

Product DiscountとCart Discountを明確に区別:

- **Product Discounts**: 最も高ランクの割引のみ適用（排他的）
- **Cart Discounts**: 全てのアクティブな割引がランク順に適用（デフォルトでスタッキング）
- **StopAfterThisDiscount**: 特定の割引以降のスタッキングを停止するモード
- **プロジェクトレベル設定**: 「Stacking」モード（全割引適用）or「BestDeal」モード（最もお得な割引のみ適用）を選択可能
- **制限**: ディスカウントコードあたり最大10件、コードなし最大100件のアクティブCart Discount

> **参考**: [commercetools Pricing and Discounts](https://docs.commercetools.com/api/pricing-and-discounts-overview)

### 3.3 WooCommerce（クーポン制限モデル）

- **「個別使用のみ」チェックボックス**: 他クーポンとの併用を禁止
- **Smart Couponsプラグイン**: 「使用可能なクーポン」「使用不可のクーポン」のホワイトリスト/ブラックリストを設定
- **Advanced Couponsプラグイン**: カート条件ベースの細かいスタッキングルール

> **参考**: [WooCommerce Smart Coupons](https://woocommerce.com/document/smart-coupons/how-to-stack-or-combine-coupons/) / [Advanced Coupons](https://advancedcouponsplugin.com/allow-stack-coupons-to-be-applied/)

### 3.4 Shopify

**割引のスタッキング**
- **割引クラス**: 商品割引、注文割引、送料割引の3クラス
- クラス間の組み合わせは可能（商品 + 注文 + 送料を同時適用可）
- **クラス内の制限**:
  - ラインアイテムあたり商品割引は1つのみ
  - 注文あたり送料割引は1つのみ
  - 最大5つの商品/注文割引 + 1つの送料割引

**管理画面UI**
- 割引一覧に「Combinations」カラムがあり、各割引がどのクラスと組み合わせ可能かを表示
- 割引作成時に、組み合わせ可能なクラスを明示的にチェックボックスで選択
- フィルタ: ステータス、方法、タイプ、組み合わせ、開始日、使用回数

> **参考**: [Shopify Combining Discounts](https://help.shopify.com/en/manual/discounts/discount-combinations) / [UX for Discounts](https://shopify.dev/docs/apps/build/discounts/ux-for-discounts)

### 3.5 Amazon Subscribe & Save

**割引のスタッキング**
- 基本割引: 個別商品5%OFF、月次配送で5アイテム以上バンドルすると最大15%OFF
- **クーポンスタッキング**: デジタルクーポンはS&S割引と重複適用（%クーポン先→S&S割引後）
- **初回限定クーポン**: 初回配送のみに適用、定期分には適用されない
- **Prime家族**: おむつ・ベビーフードは5+サブスクで最大20%OFF

> **参考**: [Amazon Subscribe & Save](https://www.amazon.com/b?ie=UTF8&node=15283820011)

### 3.6 Voucherify（プロモーションエンジン）

**最も多機能な割引ルールエンジン**
- 1回のAPIリクエストで最大30の割引オブジェクトを検証・適用可能
- クーポン、ギフトカード、プロモーションティア、ロイヤリティ報酬、紹介コードを組み合わせ可能
- **条件ルール**: 顧客属性/セグメント、購入履歴、注文/カートコンテキスト、商品/カテゴリ、地域/チャネル、デバイス、時間帯、速度/予算上限、カスタムメタデータ、決済方法、SKUターゲティング

> **参考**: [Voucherify Stacking Rules](https://support.voucherify.io/article/604-stacking-rules) / [UX Best Practices](https://www.voucherify.io/blog/coupon-promotions-ui-ux-best-practices-inspirations)

---

## 4. 塾業界の割引制度の実態

### 4.1 ITTO個別指導学院（最も参考になる事例）

| 割引種別 | 条件 | 特典内容 |
|---------|------|---------|
| **兄弟姉妹割引** | 2名以上同時在籍 | 2人目以降: 入会金免除、年会費無料、授業料5%OFF |
| **母子家庭割引** | 母子家庭であること | 入会金50%OFF、授業料5%OFF |
| **転塾特典** | 他塾からの転入 | 初年度年会費全額免除 |
| **紹介割引** | 在籍生からの紹介 | 入会金50%OFF |

**重要なポイント**: 複数の割引が理論的にスタック可能。例えば、母子家庭で兄弟がおり、他塾から転入する場合、兄弟割引 + 母子家庭割引 + 転塾特典が同時に適用される可能性がある。各割引のターゲットが異なる（入会金・年会費・授業料）点が設計上のポイント。

> **参考**: [ITTO各種割引](https://www.itto.jp/discount/) / [みやび個別指導学院](https://www.miyabi-kobetsu.jp/discount/)

### 4.2 東京個別指導学院（Benesse系列）

- 兄弟紹介: 3ヶ月目以降に特別割引
- 施設費免除: 2人目以降の兄弟が同時在籍時
- 条件: 一方の兄弟が累計13ヶ月以上在籍 or 12ヶ月一括支払いプラン

> **参考**: [東京個別指導学院 紹介制度](https://www.kobetsu.co.jp/referral/)

### 4.3 公文（Kumon）

- **兄弟割引・紹介制度なし**
- 教科ごとの固定月額制

### 4.4 塾管理システム

| システム名 | 特徴 |
|-----------|------|
| **SCHPASS（スクパス）** | 2,476+教室で導入、19機能、ワンクリック一括請求書生成 |
| **Comiru** | 請求管理、保護者連絡、請求書作成・追跡 |
| **Jukugo** | 生徒1人あたり月額100円〜、頻繁なアップデート |

> **参考**: [塾システム比較](https://juku-system-navi.com/review-juku-system/)

---

## 5. UI/UXデザインパターン（管理画面）

### 5.1 Salesforce CPQ の Price Waterfall パターン

Salesforce CPQは、見積ラインに対して複数の割引を「ウォーターフォール」（滝）のように段階的に適用する:

```
定価 (List Price)
  └→ スケジュール割引 (Volume/Schedule Discount)
      └→ 契約割引 (Contracted Price)
          └→ 追加割引 (Additional Discount) ← 営業担当が手動入力
              └→ パートナー割引 (Partner Discount)
                  └→ 代理店割引 (Distributor Discount)
                      └→ 最終価格 (Net Price)
```

**UIの特徴**:
- Quote Line Editor で各割引フィールドがカラムとして表示される
- 各段階の金額が一目で確認できる透明性の高い設計
- `AdditionalDiscountUnit` フィールドで許可する割引タイプ（%、金額、単価、合計）を制限可能

> **参考**: [Salesforce CPQ Price Waterfall](https://help.salesforce.com/s/articleView?id=000380701) / [Nebula Consulting解説](https://nebulaconsulting.co.uk/insights/salesforce-cpq-the-price-waterfall/)

### 5.2 各プラットフォームの管理画面パターン比較

| パターン | 採用プラットフォーム | 説明 |
|---------|-------------------|------|
| **レイヤーベーススタッキング** | Kibo Commerce | 番号付きレイヤーで適用順を管理、レイヤー間で再計算 |
| **組み合わせ可否カラム** | Shopify | 一覧テーブルに各割引がどのクラスと組み合わせ可能か表示 |
| **ネスト割引行** | Zuora CPQ X | 見積UIで商品行の下に割引行をインライン表示 |
| **一元管理ダッシュボード** | Stripe, Chargebee | Products > Coupons ページで作成/編集/削除 |
| **フィルタ/ソート** | Shopify, Kibo Commerce | ステータス、種類、組み合わせ、日付、使用回数でフィルタ |
| **手動割引オーバーレイ** | Chargebee | クーポン割引の上にさらに最大10件の手動割引を追加可 |
| **スタッキングトグル** | Zuora, Kibo Commerce | 割引ごとに「この割引はスタック可能か？」のフラグ |
| **ホワイトリスト/ブラックリスト** | WooCommerce | 「併用可」「併用不可」の割引リストを明示的に指定 |
| **StopAfterThisDiscount** | commercetools | 特定の割引以降のスタッキングを停止するモード |
| **2層アーキテクチャ** | Stripe | 内部Coupon + 顧客向けPromotion Codeの分離管理 |

### 5.3 推奨: 塾向け管理画面の割引設定UI

```
┌─────────────────────────────────────────────────────┐
│ 割引マスタ管理                                        │
├─────────────────────────────────────────────────────┤
│ [+ 新規割引を追加]                                    │
│                                                     │
│ 名前          │ 種別   │ 値     │ 対象      │ 条件     │ スタック │
│ ─────────────┼────────┼────────┼──────────┼─────────┼────────│
│ 初月無料      │ 定額   │ 100%   │ 授業料   │ 初月のみ  │ ✓      │
│ 兄弟割引      │ %     │ 5%    │ 授業料   │ 兄弟在籍  │ ✓      │
│ 社員割引      │ %     │ 10%   │ 授業料   │ 社員     │ ✓      │
│ 母子家庭割引   │ %     │ 5%    │ 授業料   │ 母子家庭  │ ✓      │
│ 入会金免除     │ 定額   │ 全額   │ 入会金   │ 兄弟在籍  │ ✓      │
│ 転塾特典      │ 定額   │ 全額   │ 年会費   │ 他塾転入  │ ✓      │
└─────────────────────────────────────────────────────┘
```

---

## 6. UI/UXデザインパターン（顧客向け）

### 6.1 割引表示のベストプラクティス（UXリサーチに基づく）

#### Baymard Institute の4つの落とし穴（18%以上のECサイトが該当）
1. 割引情報が他の要素に埋もれている
2. 割引情報が価格から遠い位置にある
3. 同じプロモーションを複数箇所に重複表示している
4. 割引率や割引額を強調していない

#### 100の法則（Jonah Berger）
- **$100未満の商品**: パーセンテージ表示が効果的（例: ¥2,000の商品 → 「25%OFF」）
- **$100以上の商品**: 金額表示が効果的（例: ¥100,000の商品 → 「¥10,000 OFF」）

> **参考**: [Baymard - Price Discounts](https://baymard.com/blog/product-page-price-discounts)

### 6.2 注文サマリーでの割引表示パターン

```
┌─────────────────────────────────────────┐
│ ご注文内容                               │
├─────────────────────────────────────────┤
│ 中学2年 数学（週2回）                     │
│   通常授業料          ¥24,000           │
│   ✓ 初月無料         -¥24,000  ← 緑色   │
│   ✓ 兄弟割引(5%)     -¥1,200   ← 緑色   │  ※初月は0円なので
│   ✓ 社員割引(10%)    -¥2,400   ← 緑色   │   実質適用されない
│ ─────────────────────────────────────── │
│ 入会金                                   │
│   通常               ¥22,000            │
│   ✓ 兄弟入会金免除   -¥22,000  ← 緑色    │
│ ─────────────────────────────────────── │
│ 小計                  ¥24,000            │
│ 割引合計             -¥24,000   ← 赤/緑  │
│ ─────────────────────────────────────── │
│ ご請求額（初月）        ¥0               │
│ ★ ¥24,000 お得になりました！              │
│                                          │
│ ※次月以降: ¥20,400/月（兄弟割引+社員割引適用）│
└─────────────────────────────────────────┘
```

### 6.3 主要な表示パターン

| パターン | 採用サービス | 説明 |
|---------|------------|------|
| **割引別行表示** | Shopify, Voucherify, Amazon | 適用された各割引を小計〜合計の間に個別行で表示 |
| **ラインアイテム割引** | Shopify, Chargebee | 各商品行に適用された割引と割引額を表示 |
| **節約バッジ** | Amazon S&S, Voucherify | 「¥X お得」「X%OFF」を目立つ位置に表示 |
| **自動適用プロモーション** | Voucherify, Shopify | 条件を満たすと割引が自動的に表示される（コード不要） |
| **コンフリクトエラー** | Shopify | 互いに排他的な割引を適用しようとした際に明確なメッセージ |

> **参考**: [Voucherify UX Kit](https://support.voucherify.io/article/551-coupons-promotions-ui-kit) / [NN/G Communicating Discounts](https://www.nngroup.com/articles/communicating-discounts/)

---

## 7. 割引適用ロジックの設計パターン

### 7.1 適用順序モデルの比較

| パターン | 採用先 | 説明 | 適合シーン |
|---------|--------|------|-----------|
| **Sequential（逐次適用）** | Zuora (default), Recurly Compound | 各割引が前の割引後の残額に適用 | 厳密な金額管理が必要な場合 |
| **Stacked（加算適用）** | Zuora (opt-in), Recurly Full Line Item | 全割引を元の金額に対して適用 | シンプルで顧客に分かりやすい |
| **Priority Classes** | Zuora | 割引をグループに分け優先順位付け | 大規模で多種の割引がある場合 |
| **Type-based ordering** | Chargebee, Recurly | 定額割引→%割引（or逆）の固定順 | 設定をシンプルにしたい場合 |
| **Script/Custom Logic** | Stripe Script Coupons | コードで条件ロジックを自由記述 | 特殊な条件が多い場合 |
| **Rules Engine** | Voucherify | 10種類以上の条件タイプを組み合わせ | エンタープライズ級のプロモーション |

### 7.2 塾向けシステムの推奨設計

塾の割引は**対象（ターゲット）が異なる**ため、Zuoraのモデルに近い設計が最適:

```
割引のデータモデル:

Discount {
  id: UUID
  name: string              // "兄弟割引"
  type: PERCENTAGE | FIXED  // パーセンテージ or 定額
  value: number             // 5 (= 5%) or 22000 (= ¥22,000)
  target: TUITION | ENROLLMENT_FEE | ANNUAL_FEE  // 適用対象
  duration: FIRST_MONTH | ONGOING | ONE_TIME      // 適用期間
  stackable: boolean        // 他割引との重複適用可否
  priority: number          // 適用優先順位
  conditions: Condition[]   // 適用条件
}

Condition {
  type: SIBLING | EMPLOYEE | SINGLE_PARENT | TRANSFER | REFERRAL | CAMPAIGN
  params: object            // 条件パラメータ
}
```

**適用フロー**:
1. 顧客の属性・条件をチェック
2. 適格な割引を全て抽出
3. 対象（target）ごとにグルーピング
4. 同一対象内でpriority順にソート
5. stackable=true の割引はStacked方式で合算
6. stackable=false の割引は最も優先度の高いものだけ適用
7. 合計割引額が元の金額を超えないようキャップ

---

## 8. 設計への提言

### 8.1 データモデル設計のポイント

1. **割引をファーストクラスエンティティとして扱う**（Zuoraパターン）
   - 割引は商品やプランとは独立したマスタデータとして管理
   - 各割引に対象（授業料・入会金・年会費）、期間、条件、スタック可否を持たせる

2. **適用順序を明示的・設定可能にする**（Recurlyパターン）
   - 「%割引先 or 定額先」「加算 or 逐次」の設定を管理画面で選択可能に

3. **割引クラス/優先順位を導入する**（Zuora Discount Classesパターン）
   - キャンペーン割引（最優先）→ 属性割引 → 手動割引 のように階層化

### 8.2 管理画面UIのポイント

1. **割引マスタ画面**: 全割引の一覧・作成・編集。Shopifyの「Combinations」カラムを参考に、各割引がどの割引と組み合わせ可能かを明示
2. **受注/請求画面**: Salesforce CPQのPrice Waterfallを参考に、各段階の金額を透明に表示
3. **シミュレーション機能**: 割引適用前後の金額をリアルタイムでプレビュー

### 8.3 顧客向けUIのポイント

1. **割引は別行で表示**: 各割引を個別に表示し、何がどれだけ割引されているか一目瞭然に
2. **自動適用 + 節約額の強調**: 条件を満たす割引は自動適用し、「¥X お得」を緑色で表示
3. **次月以降の料金予告**: 初月無料の場合、次月以降の請求額を必ず表示（信頼性向上）
4. **100の法則**: 月額¥10,000未満は%表示、¥10,000以上は金額表示が効果的

### 8.4 特に注意すべき設計判断

| 判断項目 | 推奨 | 理由 |
|---------|------|------|
| 割引の適用方式 | **Stacked（加算）** | 塾の保護者にとって分かりやすい |
| 初月無料の実装 | **100%OFF割引（1期間）** | Zuoraのモデル。割引として統一管理可能 |
| 兄弟割引の管理単位 | **家族（世帯）単位** | 生徒個人ではなく家族に紐づけ |
| 割引の組み合わせ上限 | **対象ごとに合計100%まで** | 過剰割引を防止 |
| 条件チェックのタイミング | **受注時 + 月次課金時** | 条件が変わった場合（兄弟退塾等）に対応 |

---

## 9. 参考URL一覧

### SaaS課金プラットフォーム
- [Stripe Billing - Trials](https://docs.stripe.com/billing/subscriptions/trials)
- [Stripe Billing - Coupons](https://docs.stripe.com/billing/subscriptions/coupons)
- [Stripe - Script Coupons](https://docs.stripe.com/billing/subscriptions/script-coupons)
- [Chargebee - Coupons](https://www.chargebee.com/docs/billing/2.0/product-catalog/coupons)
- [Chargebee - Multiple Coupons](https://www.chargebee.com/docs/billing/2.0/kb/product-catalog/applying-multiple-coupons-and-manual-discounts-to-a-subscription)
- [Chargebee - Discount Management](https://www.chargebee.com/subscription-management/discount-management-system/)
- [Recurly - Multiple Coupons](https://docs.recurly.com/docs/multiple-coupons-per-account)
- [Recurly - How Coupons Applied](https://support.recurly.com/hc/en-us/articles/360023026532)
- [Zuora - Discount Charge Models](https://knowledgecenter.zuora.com/Zuora_Billing/Build_products_and_prices/Basic_concepts_and_terms/B_Charge_Models/B_Discount_Charge_Models)
- [Zuora - Enhanced Discount Use Cases](https://knowledgecenter.zuora.com/Zuora_Billing/Build_products_and_prices/Use_cases_of_charge_models/D_Enhanced_discount_use_cases)
- [Zuora - CPQ X Nested Discounts](https://knowledgecenter.zuora.com/Zuora_CPQ/CPQ_X/4_Advanced_CPQ_X_Functionalities/Enable_and_use_Nested_Discount_Rows_in_CPQ_X)

### ECサブスクリプション・プロモーション
- [Shopify - Combining Discounts](https://help.shopify.com/en/manual/discounts/discount-combinations)
- [Shopify - UX for Discounts](https://shopify.dev/docs/apps/build/discounts/ux-for-discounts)
- [Shopify - Build Discounts UI](https://shopify.dev/docs/apps/build/discounts/build-ui-with-remix)
- [Amazon Subscribe & Save](https://www.amazon.com/b?ie=UTF8&node=15283820011)
- [Voucherify - Stacking Rules](https://support.voucherify.io/article/604-stacking-rules)
- [Voucherify - UX Best Practices](https://www.voucherify.io/blog/coupon-promotions-ui-ux-best-practices-inspirations)
- [Voucherify - UX Kit](https://support.voucherify.io/article/551-coupons-promotions-ui-kit)
- [Voucherify - Figma UI Kit](https://www.figma.com/community/file/1100356622702326488/ecommerce-coupons-promotions-ui-kit)
- [Kibo Commerce - Stackable Discounts](https://docs.kibocommerce.com/help/stackable-discounts)
- [commercetools - Pricing and Discounts](https://docs.commercetools.com/api/pricing-and-discounts-overview)
- [WooCommerce Smart Coupons](https://woocommerce.com/document/smart-coupons/how-to-stack-or-combine-coupons/)
- [Advanced Coupons - Stacking](https://advancedcouponsplugin.com/allow-stack-coupons-to-be-applied/)

### Salesforce CPQ
- [Salesforce CPQ - Price Waterfall](https://help.salesforce.com/s/articleView?id=000380701)
- [Salesforce CPQ - Discounting Tools (Trailhead)](https://trailhead.salesforce.com/content/learn/modules/discounting-tools-in-salesforce-cpq)
- [Salesforce CPQ - Manual Discounts (Trailhead)](https://trailhead.salesforce.com/content/learn/modules/discounting-tools-in-salesforce-cpq/apply-manual-discounts)
- [Salesforce CPQ - Discount Schedules (Trailhead)](https://trailhead.salesforce.com/content/learn/modules/discounting-tools-in-salesforce-cpq/configure-quantities-for-discount-schedules)
- [CPQ Price Waterfall 解説 (Nebula)](https://nebulaconsulting.co.uk/insights/salesforce-cpq-the-price-waterfall/)
- [CPQ Price Waterfall 解説 (Milo Massimo)](https://milomassimo.com/Salesforce-CPQ-Price-Waterfall.html)
- [CPQ Price Waterfall 解説 (Oreate AI)](https://www.oreateai.com/blog/unpacking-the-price-waterfall-how-salesforce-cpq-navigates-discounts/cf1a72b0ab0a1f9e42954d296983a278)

### UXリサーチ
- [Baymard - Product Page Price Discounts](https://baymard.com/blog/product-page-price-discounts)
- [NN/G - Communicating Ecommerce Discounts](https://www.nngroup.com/articles/communicating-discounts/)
- [Sufio - Line Item Discounts for Better Clarity](https://sufio.com/news/line-item-discounts/)
- [Invoice Simple - How to Show Discount](https://www.invoicesimple.com/blog/show-discount-on-invoice)

### 日本市場
- [日本のSaaS UI/UX要件](https://nihonium.io/japanese-uiux-design-key-requirements-for-saas/)
- [日本市場向けSaaS料金モデル](https://nihonium.io/pricing-models-saas-japan-success/)
- [日本向けSaaS料金ローカライズ](https://nihonium.io/localizing-saas-pricing-for-japan/)

### 塾業界
- [ITTO個別指導学院 - 各種割引](https://www.itto.jp/discount/)
- [みやび個別指導学院 - 各種割引](https://www.miyabi-kobetsu.jp/discount/)
- [東京個別指導学院 - 紹介制度](https://www.kobetsu.co.jp/referral/)
- [公文 - 会費](https://www.kumon.ne.jp/kaihi/index.html)
- [塾システム比較](https://juku-system-navi.com/review-juku-system/)
