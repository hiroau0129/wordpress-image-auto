---
name: phase7-analysis
description: 記事公開後の分析・後処理エージェントスキル。Google Indexing APIでのインデックス登録、スプレッドシート全タブ更新（ダッシュボード・記事作成ログ・KW戦略・トピッククラスター・KPIレポート）、既存記事への内部リンク追加、KPIフィードバック生成を実行する。「Phase 7」「分析エージェント」「公開後の処理」「インデックス登録」「スプレッドシート更新」などの指示があれば必ずこのスキルを使うこと。
---

# Phase 7: 分析エージェント

**目的**: インデックス登録、スプレッドシート更新、内部リンク追加、KPIレポート生成

**所要時間**: 約15分

---

## 実行フロー

### Step 1: Google Indexing API で即時インデックス登録

**前提条件**:
- Google Indexing API が有効化されている
- サービスアカウント認証情報（JSON）が設定されている
- GSC でサイトのオーナー権限がある

**Pythonスクリプト**:

```python
from google.oauth2 import service_account
from googleapiclient.discovery import build

SERVICE_ACCOUNT_FILE = 'indexing-service-account.json'
credentials = service_account.Credentials.from_service_account_file(
    SERVICE_ACCOUNT_FILE,
    scopes=['https://www.googleapis.com/auth/indexing']
)

service = build('indexing', 'v3', credentials=credentials)

url = 'https://your-domain.com/your-article-slug/'
request_body = {
    'url': url,
    'type': 'URL_UPDATED'
}

try:
    response = service.urlNotifications().publish(body=request_body).execute()
    print(f'インデックス登録リクエスト送信: {url}')
    print(f'Response: {response}')
except Exception as e:
    print(f'エラー: {e}')
```

**Indexing API が未設定の場合**:

```
ユーザーに以下を依頼:
1. Google Search Console で対象サイトを開く
2. URL検査ツール → 公開した記事の URL
3. 「インデックス登録をリクエスト」をクリック
```

**確認方法**: Google Search Console → URL検査ツールで「Google に登録されています」を確認（数時間〜24時間かかる場合あり）

---

### Step 2: スプレッドシート全タブ更新

#### 2.1. 「ダッシュボード」タブ更新

| 指標 | 更新内容 |
|:----|:--------|
| 最終更新日 | 本日の日時 |
| 累計公開記事数 | +1 した値 |
| 本月公開数 | +1 した値 |
| インデックス済数 | +1 した値（インデックス登録完了後） |

---

#### 2.2. 「記事作成ログ」タブ更新

該当記事の行を更新:

| 列 | 更新内容 |
|:----|:--------|
| D. タイトル | 公開した記事のタイトル |
| E. URL | 公開した記事の URL |
| H. 品質ループ回数 | 実施した品質ループ回数 |
| I. 品質スコア | 最終品質スコア |
| J. ステータス | `公開済み` |
| K. WP Post ID | WordPress の Post ID |
| L. 公開日 | 本日の日付 |

---

#### 2.3. 「KW戦略」タブ更新

該当KWの Status を更新:

| 列 | 更新内容 |
|:--|:--------|
| G. Status | `KW選定完了` → `公開済` |

---

#### 2.4. 「トピッククラスター」タブ更新

| 列 | 更新内容 |
|:--|:--------|
| H. Status | `未着手` → `公開済` |
| I. URL | 公開した記事の URL |

---

#### 2.5. 「KPIレポート」タブに新行追加

| 列 | 内容 |
|:--|:----|
| A. 日付 | 本日の日付 |
| B. 累計公開記事数 | 更新後の累計数 |
| C. 本日公開数 | 1 |
| D. インデックス率（%） | インデックス済数 ÷ 累計公開記事数 |
| E. PV（GA4） | GA4 Data API から取得（取得時） |
| F. 表示回数（GSC） | GSC Data API から取得（取得時） |
| G. クリック数（GSC） | GSC Data API から取得（取得時） |
| H. CTR（%） | クリック数 ÷ 表示回数 |
| I. 平均順位（GSC） | GSC Data API から取得（取得時） |

**GA4・GSC データ取得（直近7日間）**:

```python
# GA4 Data API
from google.analytics.data_v1beta import BetaAnalyticsDataClient
from google.analytics.data_v1beta.types import RunReportRequest, Dimension, Metric

client = BetaAnalyticsDataClient()
request = RunReportRequest(
    property=f"properties/{GA4_PROPERTY_ID}",
    date_ranges=[{"start_date": "7daysAgo", "end_date": "today"}],
    dimensions=[Dimension(name="date")],
    metrics=[
        Metric(name="screenPageViews"),
        Metric(name="userEngagementDuration"),
    ],
)
response = client.run_report(request)
```

```python
# GSC Data API（直近7日間）
from google.search_console import SearchConsoleAPI

gsc = SearchConsoleAPI('service-account-key.json')
stats = gsc.query(
    site_url='https://your-domain.com/',
    start_date='7daysAgo',
    end_date='today',
    dimensions=['date'],
    metrics=['clicks', 'impressions', 'ctr', 'position']
)
```

---

### Step 3: 既存記事への内部リンク自動追加

**処理ロジック**:
1. 公開済み全記事の H2 見出し・本文テキストを取得
2. 新規記事の KW が既存記事に自然に挿入できる箇所を特定
3. WordPress REST API で既存記事を更新・内部リンク追加

```python
import re
from wordpress import WordPress

wp = WordPress(
    url='https://your-domain.com',
    username='your_wp_username',
    password='your_app_password'
)

NEW_KW = '対象KW'
NEW_ARTICLE_URL = 'https://your-domain.com/new-article-slug/'

existing_posts = wp.posts.get_all()

for post in existing_posts:
    content = post['content']['rendered']
    
    if NEW_KW in content and f'href="{NEW_ARTICLE_URL}"' not in content:
        pattern = f'({NEW_KW})'
        anchor_text = f'<a href="{NEW_ARTICLE_URL}">{NEW_KW}</a>'
        
        updated_content = re.sub(
            pattern,
            anchor_text,
            content,
            count=1,  # 最初の1箇所だけ
            flags=re.IGNORECASE
        )
        
        post['content']['rendered'] = updated_content
        wp.posts.update(post['id'], post)
        print(f'内部リンク追加: {post["title"]["rendered"]}')
```

**自動化が難しい場合（手動）**:
1. 新規記事 KW が含まれている既存記事を5件抽出
2. 各記事で KW の初出箇所に新規記事 URL へのリンクを挿入
3. アンカーテキスト: KW そのまま

---

### Step 4: 「内部リンク管理」タブに記録

追加した全内部リンクを記録:

| 列 | 内容 |
|:--|:----|
| A. リンク元記事 | 既存記事タイトル |
| B. リンク先記事 | 新規記事タイトル |
| C. アンカーテキスト | 使用したリンクテキスト（KW） |
| D. 設置日 | 本日の日時 |
| E. リンク先URL | 新規記事 URL |

---

### Step 5: KPI フィードバック生成

`kpi_feedback.md` を更新（次回の記事制作に引き継ぐパターン情報）:

```markdown
# KPIフィードバック（自動更新: YYYY-MM-DD）

## サイト概況
| 指標 | 値 | 前日比 | 備考 |
|:----|:--|:------|:----|
| 累計公開記事数 | XX本 | +1 | |
| ドメインレーティング | XX | | |
| オーガニックKW数 | XX | | |
| 推定オーガニックトラフィック | X,XXX | | |

## 今日の主な動き
1. 【KW名】が公開され、インデックスリクエスト送信
2. 既存X記事に内部リンク追加（クラスター強化）
3. 品質スコアXX点で初公開

## 成功パターン（これを踏襲せよ）
- （品質スコア高・内部リンク数多・順位上昇など、具体的要因を記録）

## 失敗パターン（これを避けよ）
- （インデックス未登録・順位低迷・カニバリなど、原因と対策を記録）

## リライト優先度リスト
| 記事 | 現在順位 | 目標順位 | 改善ポイント |
|:---|:---:|:---:|:----------|
| （11〜30位の記事を優先） | | | |
```

**記入基準**:

| セクション | 記入タイミング |
|:----------|:------------|
| 成功パターン | 品質スコア95+・GSC表示回数あり・クラスター完成時 |
| 失敗パターン | インデックス未登録・順位低迷・KWカニバリ発生時 |
| リライト優先度 | 現在11〜30位・表示回数高くCTR低い記事を優先 |

---

## チェックリスト

- [ ] Google Indexing API でインデックスリクエスト送信した
- [ ] インデックス登録が成功したか GSC で確認した
- [ ] 「ダッシュボード」タブを更新した
- [ ] 「記事作成ログ」タブを更新した（ステータス: 公開済み）
- [ ] 「KW戦略」タブを更新した（Status: 公開済）
- [ ] 「トピッククラスター」タブを更新した（Status: 公開済, URL記載）
- [ ] 「KPIレポート」タブに本日の KPI を追加した
- [ ] 既存記事への内部リンク（3〜5本）を追加した
- [ ] 「内部リンク管理」タブに記録した
- [ ] kpi_feedback.md を更新した（成功・失敗パターン記録）

---

## 次のステップ

**全7フェーズ完了！**

次の記事制作は KW選定スキル（`kw-selection`）から開始します。

---

**参考資料**:
- `./setup/gapi-config.json` → Google APIs 設定
- `./kpi_feedback.md` → KPI フィードバック（自動生成ファイル）
