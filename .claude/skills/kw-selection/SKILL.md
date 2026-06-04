---
name: kw-selection
description: 遺品整理・生前整理など低ボリューム地域特化ジャンルのSEOキーワード選定を行うスキル。月間検索数50以上を基準に、低競合フロントKW（仏具買取・着物買取・骨董品買取など）から本命KW（遺品整理・生前整理）への動線設計を含む。「次の記事のKWを決めたい」「キーワード選定して」「どのKWを攻めるべきか」などの指示があれば必ずこのスキルを使うこと。
---

# KW選定エージェント

**目的**: 次に書く記事のキーワードを決定する

**所要時間**: 約15分

---

## 実行フロー

### Step 1: 記事作成ログの確認

```
1. スプレッドシート「記事作成ログ」タブを開く
2. ステータス列を確認
3. 「未着手」の一番上の行を取得
```

**未着手行がある場合**:
→ そのKWで Phase 2（リサーチエージェント）に進む（Ahrefsリサーチ不要）

**未着手行がない場合**:
→ 以下の手順でKW選定を実行

---

### Step 2: 候補KW発掘（Keyword Planner）

**Google Ads Keyword Planner（Pythonスクリプト経由）**:

```bash
cd "/Users/hiroyuki/Documents/Obsidian Vault/google-ads-mcp" && \
GOOGLE_ADS_DEVELOPER_TOKEN=DszQLfwo2qQlucd1WtAqgg .venv/bin/python3 -c "
from google.ads.googleads.client import GoogleAdsClient
import google.auth

credentials, _ = google.auth.default(scopes=['https://www.googleapis.com/auth/adwords'])
client = GoogleAdsClient(credentials=credentials, developer_token='DszQLfwo2qQlucd1WtAqgg', version='v24')

customer_id = '7153235607'
kw_idea_service = client.get_service('KeywordPlanIdeaService')
request = client.get_type('GenerateKeywordIdeasRequest')
request.customer_id = customer_id
request.language = 'languageConstants/1005'   # 日本語
request.geo_target_constants.append('geoTargetConstants/20636')  # 日本

# ジャンル起点のシードKW（適宜変更）
seed_keywords = [
    '仏具 買取', '着物 買取', '骨董品 買取',
    '仏壇 処分', '遺品 買取', '家具 処分',
    '家電 処分', '不用品 買取', '生前整理 買取', '神棚 処分'
]
request.keyword_seed.keywords.extend(seed_keywords)

response = kw_idea_service.generate_keyword_ideas(request=request)

results = []
for idea in response:
    vol = idea.keyword_idea_metrics.avg_monthly_searches
    comp_idx = idea.keyword_idea_metrics.competition_index
    comp = idea.keyword_idea_metrics.competition.name
    cpc = idea.keyword_idea_metrics.average_cpc_micros / 1_000_000 if idea.keyword_idea_metrics.average_cpc_micros else 0
    if vol >= 50:
        results.append((vol, idea.text, comp, comp_idx, round(cpc)))

results.sort(reverse=True)
print(f'{'KW':<30} {'Vol':>6} {'競合度':>4} {'Comp':>8} {'CPC(円)':>8}')
print('-'*62)
for vol, kw, comp, comp_idx, cpc in results[:50]:
    print(f'{kw:<30} {vol:>6} {comp_idx:>4} {comp:>8} {cpc:>8}')
"
```

> **前提**: Developer Tokenは **Basic Access以上** が必要。テストアクセスのみの場合は実アカウントへのKPL APIコールが失敗する。Basic Access申請 → 承認後に利用可能。

**フィルター条件**:

- 国: JP（日本）
- 月間検索ボリューム: **50以上**（遺品整理・生前整理ジャンルは低ボリューム前提）
- Nav KWは手動でスキップ

**出力**: Vol・競合度（0-100）・CPC一覧

---

### Step 3: KWデータ取得・評価

Step 2の出力リストから候補を絞り込む。

**KPLで取得できる指標**:

| 指標 | KPLフィールド | 意味 |
|:---|:---|:---|
| Volume | `avg_monthly_searches` | 月間検索数（50以上を対象） |
| 競合度 | `competition_index`（0-100）| 広告主の競争度（SEO難易度の代理指標） |
| CPC | `average_cpc_micros`/1,000,000 | クリック単価（円） |

> **KDについて**: KPLはSEO向けのKD値を提供しない。`competition_index`（0=低競合・100=高競合）をKDの代理指標として使用する。ただしこれは広告競合度であるため、`competition_index`が低くてもSERP上位にDR高いサイトが並ぶ場合は難易度が高い点に注意。必要であればSERP目視確認を併用する。

**Intent（検索意図）**:

KPLはIntentを自動判定しない。KW文字列から手動で分類する。

**Intentの分類と扱い方**:

| Intent | 意味 | 優先度 |
|:---|:---|:---|
| Transactional（Do/Buy） | 今すぐ依頼・購入したい | ◎ 最優先 |
| Commercial | 比較・検討中 | ○ 優先 |
| Informational（Info） | 調べたい | △ 条件付き優先（※後述） |
| Navigational（Nav） | 特定サイトへ行きたい | × スキップ |

> **Info KWの注意点**: 遺品整理・生前整理ジャンルはInfo記事でも購買意欲が高いユーザーが多い。「Info＝優先度低」と一律判断せず、記事内でくらしのマーケットへの動線を設計できるかで判断すること。

> **Nav KWについて**: 「くらしのマーケット 遺品整理」のような特定サービス指名系は上位表示が困難なため原則スキップ。

---

### Step 4: SEO Knowledge批判3周

以下の9観点で批判的レビューを **3回繰り返す**。指摘が妥当なら修正、妥当でなければ理由を明記して却下。

#### 第1周目: ロングテール・CV距離

- [ ] Vol 50〜300程度のロングテールが選定されているか
- [ ] CV距離の分類（Transactional/Commercial > Info）が正しいか（ただしInfo KWもくらしのマーケット動線があれば可）
- [ ] 同一サイト内での既存記事とのKWカニバリゼーションはないか（※JVパートナーの別サイトとの重複は問題なし、むしろ検索結果の独占として歓迎）

#### 第2周目: 競合性・トピカルオーソリティ

- [ ] competition_index（KPL）は低いか。加えてSERP上位サイトのDRを目視確認し実際の競合難易度を判断したか
- [ ] トピカルオーソリティ設計として**クラスター記事が先、ピラー記事は後**になっているか（※下記「クラスター→ピラーの順序」参照）
- [ ] 競合SERPでDR低いサイトが勝てている実績があるか

#### 第3周目: 独自性・攻め順序

- [ ] 独自性・E-E-A-Tを出せるKWか
- [ ] 攻め順序は**低競合フロントKW（仏具買取・着物買取・骨董品買取・家電家具処分など）→ 本命KW（遺品整理・生前整理）**の動線設計になっているか
- [ ] 抜けているKWカテゴリはないか

---

#### クラスター→ピラーの順序について

このジャンルは月間検索数が小規模なため、**ピラー記事（遺品整理 総合ガイドなど）を先に作っても内部リンクが機能しない**。

推奨する構築順序：

```
① クラスター記事を5〜10本先に作る
   （仏具買取、着物買取、骨董品買取、家電・家具の処分 など入口KW）
         ↓
② ある程度揃ったらピラー記事を作る
   （遺品整理 総合ガイド、生前整理 完全マニュアル など）
         ↓
③ クラスター→ピラーへ内部リンクを整備して権威性を集約
```

現在のクラスター記事本数を確認し、5本未満であればピラーKWの選定は後回しにすること。

---

### Step 5: KW確定・スプレッドシート更新

**スプレッドシート「KW戦略」タブに記録**:

| 列 | 記入例 |
|:---|:---|
| A. KW | 仏具 買取 愛川町 |
| B. Volume | 70 |
| C. 競合度 | 12（KPL competition_index） |
| D. Intent | Commercial（手動分類） |
| E. CV距離 | 7点（フロントKW→遺品整理への動線あり） |
| F. 自社独自価値 | 地域特化・実例3件掲載可 |
| G. Status | KW選定完了 |

**スプレッドシート「記事作成ログ」タブに新規行追加**:

| 列 | 記入例 |
|:---|:---|
| A. 記事# | #45 |
| B. 制作日 | 2024-01-15 |
| C. KW | 仏具 買取 愛川町 |
| D. タイトル | （後で記入） |
| E. URL | （後で記入） |
| F. カテゴリ | フロントKW／遺品整理 |
| G. Volume | 70 |
| H. 品質ループ回数 | （後で記入） |
| I. 品質スコア | （後で記入） |
| J. ステータス | KW選定完了 |

---

## チェックリスト

- [ ] 記事作成ログで未着手行を確認した
- [ ] KPL Pythonスクリプトを実行してKW候補を取得した（Volume 50以上でフィルター）
- [ ] KWデータ取得完了（Volume, 競合度, CPC, Intent手動分類）
- [ ] Nav KWをスキップしたか確認した
- [ ] Info KWの場合、くらしのマーケットへの動線設計が可能か確認した
- [ ] 現在のクラスター記事本数を確認し、5本未満ならピラーKWを後回しにした
- [ ] 攻め順序（フロントKW→本命KW）が動線設計に沿っているか確認した
- [ ] 9観点でSEO Knowledge批判3周を実施した
- [ ] KW戦略タブにKWを記録した
- [ ] 記事作成ログタブの「ステータス」を「KW選定完了」に更新した

---

## 次のステップ

Phase 2: リサーチエージェント に進む

---

**参考資料**:

- `./rules/quality-standards.md` → KW選定の判断基準
- `./data/competitor-analysis.md` → 競合分析テンプレート
