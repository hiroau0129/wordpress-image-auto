---
description: WordPress記事の全H2直下と長文セクションに、内容に合った画像を自動挿入する
---

以下の手順で作業してください。

## Step 1: 設定ファイルの読み込み

`~/.claude/wp-config.env` を source して `WP_URL`・`WP_USER`・`WP_APP_PASSWORD` を取得してください。

## Step 2: 画像ライブラリの把握

`~/Desktop/画像/` 以下の全サブフォルダの画像を一覧取得してください。

```bash
find ~/Desktop/画像/ -mindepth 2 -maxdepth 2 \
  -iregex '.*\.\(jpg\|jpeg\|png\|gif\|webp\)' \
  | sed "s|$HOME/Desktop/画像/||" | sort
```

## Step 3: 記事の取得

引数または「挿入先の記事タイトルを入力してください：」で記事を特定し、raw 本文を取得してください。

```bash
source ~/.claude/wp-config.env
curl -s -u "$WP_USER:$WP_APP_PASSWORD" \
  "${WP_URL}/wp-json/wp/v2/posts?search=<タイトル>&per_page=5&status=any" \
  | python3 -c "
import json, sys
posts = json.load(sys.stdin)
for p in posts:
    print(f\"ID:{p['id']}  {p['title']['rendered']}  [{p['status']}]\")
"
```

```bash
source ~/.claude/wp-config.env
curl -s -u "$WP_USER:$WP_APP_PASSWORD" \
  "${WP_URL}/wp-json/wp/v2/posts/<POST_ID>?context=edit" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['content']['raw'])"
```

## Step 4: 挿入計画の作成

取得した本文を解析して、以下の**2つのルール**に基づいた挿入計画を立ててください。

---

### ルール1：H2直下への画像挿入（必須）

すべての h2 見出しの直後に画像を1枚挿入する。
**ただし、すでに `<!-- wp:image` が直後にある場合はスキップ。**

見出しテキストとその直後の段落内容を読み、最もふさわしい画像を画像ライブラリのファイル名から選ぶ。

---

### ルール2：長文セクション中間への画像挿入

H2 から次の H2 の間で、**直前の画像から段落・リストブロックが合計5つ以上続いた地点**に画像を1枚追加する。
その後さらに5つ以上続く場合は、再度1枚追加する（繰り返し適用）。

**例外：** 見出しテキストに「よくある質問」「FAQ」が含まれるセクションは除外する。

挿入地点は「5つ目のブロックの直後」とする。

---

### 挿入計画の確認

実行前に以下の形式で計画を提示し、「よろしいですか？」と確認を取ってください。

```
【挿入計画】

▼ ルール1: H2直下
  ・「〇〇〇〇」直後 → 防音工事/7_住宅街_線路沿い.jpg
    理由: 住宅・騒音に関するセクションのため
  ・「△△△△」直後 → 共通/書類_御見積書.jpg
    理由: 費用・見積もりのセクションのため
  ...

▼ ルール2: 長文中間
  ・「〇〇〇〇」セクション（段落X・リストY）の中間
    → 共通/人物_女性スマホ悩み.jpg
    理由: 相談・検討を促す流れのため
  ...

合計 X 枚挿入します。よろしいですか？
```

## Step 5: 画像のアップロード（重複チェックあり）

確認が取れたら、使用する画像を順番に処理してください。
**アップロード前に必ずメディアライブラリに同名ファイルが存在しないか確認し、存在すればそれを再利用してください。**

```bash
source ~/.claude/wp-config.env
IMAGE_PATH="$HOME/Desktop/画像/<フォルダ>/<ファイル名>"
FILENAME=$(basename "$IMAGE_PATH")

# まずメディアライブラリを検索
curl -s -u "$WP_USER:$WP_APP_PASSWORD" \
  "${WP_URL}/wp-json/wp/v2/media?search=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote('${FILENAME%.*}'))")&per_page=50" \
  | python3 -c "
import json, sys, os
filename = '$FILENAME'
items = json.load(sys.stdin)
found = None
for m in items:
    url = m.get('source_url', '')
    if os.path.basename(url) == filename:
        found = m
        break
if found:
    print(f\"EXISTING: MEDIA_ID={found['id']}\")
    print(f\"MEDIA_URL={found['source_url']}\")
else:
    print('NOT_FOUND')
"
```

- **`EXISTING`** と表示された場合 → そのID・URLをそのまま使用し、アップロードはスキップ
- **`NOT_FOUND`** の場合 → 以下のコマンドでアップロード

```bash
source ~/.claude/wp-config.env
IMAGE_PATH="$HOME/Desktop/画像/<フォルダ>/<ファイル名>"
MIME_TYPE=$(file --mime-type -b "$IMAGE_PATH")
curl -s -u "$WP_USER:$WP_APP_PASSWORD" \
  -H "Content-Disposition: attachment; filename=\"$(basename "$IMAGE_PATH")\"" \
  -H "Content-Type: $MIME_TYPE" \
  --data-binary @"$IMAGE_PATH" \
  "${WP_URL}/wp-json/wp/v2/media" \
  | python3 -c "
import json, sys
m = json.load(sys.stdin)
print(f\"MEDIA_ID={m['id']}\nMEDIA_URL={m['source_url']}\")
"
```

## Step 6: 本文の更新

画像ブロック形式：
```
<!-- wp:image {"id":<MEDIA_ID>,"sizeSlug":"large","linkDestination":"none"} -->
<figure class="wp-block-image size-large"><img src="<MEDIA_URL>" alt="" class="wp-image-<MEDIA_ID>"/></figure>
<!-- /wp:image -->
```

Python で文字列置換しながら各挿入位置に画像ブロックを追加し、REST API で記事を更新してください。

**H2直下への挿入：**
対象の `<!-- /wp:heading -->` タグを特定して、その直後に画像ブロックを挿入する。
同じタグが複数ある場合は見出しテキスト（`<h2`〜`</h2>`）で一意に特定すること。

**長文中間への挿入：**
対象セクション内の「5つ目のブロック終了タグ（`<!-- /wp:paragraph -->` または `<!-- /wp:list -->`）」の直後に挿入する。

```bash
source ~/.claude/wp-config.env
python3 << 'PYEOF'
import json, urllib.request, base64, re

WP_URL = "<WP_URL>"
WP_USER = "<WP_USER>"
WP_APP_PASSWORD = "<WP_APP_PASSWORD>"
POST_ID = <POST_ID>

auth = base64.b64encode(f"{WP_USER}:{WP_APP_PASSWORD}".encode()).decode()
headers = {"Authorization": f"Basic {auth}", "Content-Type": "application/json"}

req = urllib.request.Request(
    f"{WP_URL}/wp-json/wp/v2/posts/{POST_ID}?context=edit", headers=headers)
with urllib.request.urlopen(req) as r:
    post = json.loads(r.read())
content = post["content"]["raw"]

def make_image_block(media_id, media_url):
    return (
        f'\n\n<!-- wp:image {{"id":{media_id},"sizeSlug":"large","linkDestination":"none"}} -->\n'
        f'<figure class="wp-block-image size-large">'
        f'<img src="{media_url}" alt="" class="wp-image-{media_id}"/></figure>\n'
        f'<!-- /wp:image -->\n'
    )

# Step4 の計画に従って各挿入を実行（各挿入は replace を使い、count=1 で1箇所のみ）
# 例:
# heading_end = '</h2>\n<!-- /wp:heading -->'
# content = content.replace(heading_end, heading_end + make_image_block(ID, URL), 1)

body = json.dumps({"content": content}).encode()
req = urllib.request.Request(
    f"{WP_URL}/wp-json/wp/v2/posts/{POST_ID}",
    data=body, headers=headers, method="POST"
)
with urllib.request.urlopen(req) as r:
    result = json.loads(r.read())
    print(f"✓ 更新完了: {result['title']['rendered']}")
    print(f"  URL: {result['link']}")
PYEOF
```

## Step 7: 完了報告

```
【完了】
記事: 〇〇〇〇（ID: XXXXX）
URL: https://...

挿入した画像（X枚）:
  ルール1（H2直下）: X枚
  ルール2（長文中間）: X枚

メディア処理:
  新規アップロード: X枚
  既存を再利用:   X枚
```
