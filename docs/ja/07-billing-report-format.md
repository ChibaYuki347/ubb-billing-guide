# Appendix: Copilot Usage Billing CSV フォーマット詳解と AIC コスト計算

> **本章について**: [jacwu/ghfaq — `docs/billing-report-format.md`](https://github.com/jacwu/ghfaq/blob/main/docs/billing-report-format.md) (CC-BY 等、原リポジトリのライセンスに準拠) を補足章として日本語化したものです。
> 表現上の都合により節構成・例の値は原文のまま保持しています。

本章では、Copilot Billing Preview にアップロードする **Copilot usage-based billing (使用量ベース課金)** レポートの **CSV カラムスキーマ** を、各列の意味が分かるように例を交えて説明します。

---

## Column definitions (カラム定義)

| # | Column | Type | 説明 |
|---|--------|------|-----|
| 1 | `date` | ISO date (`YYYY-MM-DD`) | 利用が発生した日付。 |
| 2 | `username` | string | リクエストを発生させた GitHub ユーザー名。 |
| 3 | `product` | string | 製品名。通常は `copilot` または `spark`。 |
| 4 | `sku` | string | 課金対象アクションの SKU 識別子。例: `copilot_premium_request`、`coding_agent_premium_request`、`copilot_ai_credit`、`coding_agent_ai_credit`、`spark_ai_credit`。 |
| 5 | `model` | string | 実際に呼び出されたモデル名。例: `Claude Sonnet 4.6`、`Claude Opus 4.6`、`GPT-5.4`、`Auto: GPT-5.4`。`Auto: XXX` は Auto モデルが XXX にルーティングしたことを意味します。 |
| 6 | `quantity` | number | この行の **課金単位数** (billable unit count)。`unit_type = requests` のときは消費した PRU 数 (premium request 数) に等しく、それ以外のとき (例: `ai-credits`) は消費した AI Credits 数に等しい。 |
| 7 | `unit_type` | string | 課金単位タイプ。`requests` はリクエスト単位 (PRU) 課金、それ以外の値 (例: `ai-credits`) は AI Credits 単位の課金を意味します。 |
| 8 | `applied_cost_per_quantity` | number | 単位あたりの単価 (USD)。CSV では `0.04` が一般的で、これは **1 PRU = $0.04** を意味します。 |
| 9 | `gross_amount` | number (USD) | **Gross (グロス)**: ディスカウント前の定価相当額。`quantity × applied_cost_per_quantity` で計算されます。 |
| 10 | `discount_amount` | number (USD) | ディスカウント額。多くの場合は **月次の無料枠 (free quota)** による相殺額です。 |
| 11 | `net_amount` | number (USD) | **Net (ネット)**: 実際に請求される額。`gross_amount − discount_amount` で計算されます。 |
| 12 | `exceeds_quota` | boolean (`True` / `False`) | この行が月次無料枠を超過したかどうか。`True` の場合は通常 `discount = 0`、`net = gross` となります。 |
| 13 | `total_monthly_quota` | number | このユーザーの月次無料枠 (PRU の上限)。Business は通常 `300`、Enterprise は通常 `1000`、枠がない場合は `0`。 |
| 14 | `organization` | string | ユーザーが所属する Organization のスラグ。個人アカウントや Organization に紐付かない利用では空。 |
| 15 | `cost_center_name` | string / 空 | 任意の cost center ラベル。Organization 内で請求を分類するために使用します。 |
| 16 | `aic_quantity` | number | 同じ利用に対する AI Credits の消費量。 |
| 17 | `aic_gross_amount` | number (USD) | AI Credits ビューでの Gross 額。 |

---

## Key formulas (主要な計算式)

```
net_amount   = gross_amount - discount_amount
gross_amount = quantity × applied_cost_per_quantity   (PRU 行の場合)
```

| 概念 | 意味 |
|---|---|
| Gross | 定価 (ディスカウント前) |
| Discount | 相殺額 (通常は無料枠による) |
| Net | 実際の支払い額 (ディスカウント後) |

PRU ビューと AIC ビューは **並行して** 存在します。各行は両方の数値セットを保持しており、PRU 側は `quantity / gross / discount / net` を、AIC 側は `aic_quantity / aic_gross_amount` (+ 推定値の `aic_net_amount`) を用いることで、**「もう一方の課金方式ではいくらになるか?」** の比較を可能にしています。

---

## Examples (例)

### Example 1: 無料枠内 (`net = 0`)

```
date=2026-04-01, username=octocat, model=Claude Opus 4.6
quantity=15, applied_cost_per_quantity=0.04
gross_amount=0.60, discount_amount=0.60, net_amount=0
exceeds_quota=False, total_monthly_quota=300
aic_quantity=688.72, aic_gross_amount=6.89
```

**読み方**: Claude Opus 4.6 を 15 回呼び出し、定価 $0.60 ですが、まだ月次無料枠の範囲内なので **実支払いは $0** です。同じ利用を AIC 課金で換算すると、約 688 クレジット (≒ $6.89) を消費することになります。

### Example 2: 無料枠超過 (`net > 0`)

```
date=2026-04-02, username=hubot, model=Claude Opus 4.6
quantity=23.3, applied_cost_per_quantity=0.04
gross_amount=0.932, discount_amount=0, net_amount=0.932
exceeds_quota=True, total_monthly_quota=300
aic_quantity=7.92, aic_gross_amount=0.0792
```

**読み方**: Claude Opus 4.6 を 23.3 回呼び出し、定価 $0.93。月次無料枠を超過しているため (`exceeds_quota=True`)、ディスカウントは無く、**実支払いは $0.93** です。

---

## Overview page data explanation (Overview ページのデータ解説)

Overview ページでは **2 つの課金ビュー** を左右に並べて表示します。左が **現行の PRU 課金**、右が **AIC / usage-based billing** です。**License cost (シート料)** は両ビューで共通で、主な違いは **included credits (含まれるクレジット) による相殺の仕方** です。

### PRU billing view (PRU 課金ビュー)

| UI 値 | 意味 | 計算 |
|---|---|---|
| **Consumed (PRUs)** | PRU ビューでの全利用のディスカウント前コスト。 | `sum(gross_amount)` |
| **Discount (included PRUs)** | 各ユーザーの included PRU 枠で相殺された額。 | `sum(discount_amount)` |
| **Overages** | included PRUs 適用後に残った超過分のコスト。 | `sum(net_amount)`、または `Consumed (PRUs) - Discount (included PRUs)` |
| **License cost** | 月次の Copilot シート料金。 | Business / Enterprise のシート数と月単価から算出。 |
| **Total (license + overages)** | 現行 PRU 課金における合計請求額。 | `License cost + Overages` |

**PRU のポイント**: included PRUs は **ユーザーごとに独立** して適用されます。あるユーザーの未使用 PRU が、枠を超過した別のユーザーへ **自動的に振り替えられることはありません**。

### AIC billing view (AIC 課金ビュー)

| UI 値 | 意味 | 計算 |
|---|---|---|
| **Consumed (AICs)** | AIC ビューでの全利用のディスカウント前コスト。 | `sum(aic_gross_amount)` |
| **Discount (included AICs)** | included AIC プールで相殺された額。 | `sum(aic_gross_amount) - sum(aic_net_amount)` |
| **Additional usage** | included AIC プールを使い切った後に残った超過分のコスト。 | `sum(aic_net_amount)`、または `Consumed (AICs) - Discount (included AICs)` |
| **License cost** | 月次の Copilot シート料金。 | PRU 側と同じく、Business / Enterprise のシート数と月単価から算出。 |
| **Total (license + additional usage)** | AIC 課金における合計請求額。 | `License cost + Additional usage` |

**AIC のポイント**: included AICs は **アカウント / Organization プール単位で共有** されます。全ユーザーが同じプールから引き出し、プールを使い切ったあとの利用が **Additional usage** になります。

---

## Mapping to the Users view UI columns (Users ビューの UI カラム対応)

| UI カラム | CSV フィールド | 集計 |
|---|---|---|
| User | `username` | グループキー |
| PRUs | `quantity` (PRU 行のみ) | 合計 |
| AICs | `aic_quantity` | 合計 |
| Models used | `model` | ユニーク数 |
| PRU Net Cost | `net_amount` | 合計 |
| AIC Net Cost | `aic_net_amount` (推定値) | 合計 |
| Difference | `net_amount − aic_net_amount` | ユーザー単位の差分 |

---

## How `aic_net_amount` is computed (`aic_net_amount` の算出方法)

> CSV 自体には `aic_quantity` と `aic_gross_amount` **のみ** が含まれ、`aic_net_amount` は **含まれていません**。アプリは included credits プールに基づき、**行ごとに推定** します。

### Step 1: included credits プールの算出

| レポート範囲 | プールルール | 月次クレジット |
|---|---|---|
| **Organization** (複数ユーザー / Org あり) | Organization 全体で **共有プール** を 1 つ持つ | `Business シート数 × 3000 + Enterprise シート数 × 7000` |

シート数はデフォルトで CSV から推定されます (`total_monthly_quota = 300` → Business、`= 1000` → Enterprise)。

### Step 2: CSV の出現順にプールから差し引く

各行について、次のように計算します。

```
covered        = min(aic_quantity, remaining_pool)
uncovered      = aic_quantity − covered
aic_net_amount = aic_gross_amount × (uncovered / aic_quantity)
remaining_pool -= covered
```

**読み方**: その行の credits のうち、**無料プールでカバーされなかった割合だけが課金される** ということです。

### One-line summary (1 行サマリ)

> `aic_net_amount = aic_gross_amount × (その行の AIC のうち無料プールでカバーされなかった割合)`。
> レポート全体は CSV 行を上から **ストリーミング処理** し、**単一の included credits プール** から逐次差し引いて算出されます。

---

| ナビゲーション | |
| --- | --- |
| ← 前へ | [6. Disclaimers](06-disclaimers.md) |
| 🏠 トップ | [README](README.md) |
