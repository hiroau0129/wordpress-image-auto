---
name: zukai
description: Use when the user wants to generate a blog diagram (図解) from article text or a keyword. Reads the content deeply and freely designs the most appropriate HTML diagram from scratch. Outputs a PNG image saved to Desktop/画像/図解/.
---

# zukai — ブログ図解生成スキル

## 使い方

```
# 記事テキストをそのまま渡す
<テキスト貼り付け>

# キーワードだけ渡す
副業の始め方

# ウォームスタイル（オレンジ系）で生成
副業の始め方 --warm

# 最新バックアップから復元
副業の始め方 --recover

# 特定バージョンから復元
副業の始め方 --recover v2
```

## 基本方針

テンプレートは使わない。記事の内容を深く読み込み、**その内容に最も適したレイアウト・構造をゼロから設計してHTMLを生成する**。

- 「手順ならフロー」「比較なら2カラム」という機械的な当てはめをしない
- 記事が伝えたいことの本質を掴み、読者が一目で理解できる図解を作る
- 複数の要素を組み合わせたレイアウトも積極的に使う
- ステップ数・カラム数・階層数はすべてコンテンツに合わせて自由に決める

## 出力ワークフロー

1. 記事テキストまたはキーワードを読み込む
2. 「何を伝えれば読者が最も理解しやすいか」を判断する
3. 最適なレイアウトをゼロから設計し、HTMLを生成する
4. HTMLを一時ファイルとして書き出す（`/tmp/zukai-tmp.html`）
5. **バックアップを作成する**（詳細は「バックアップ仕様」セクション参照）
6. Chrome ヘッドレスで PNG に変換する（Bashツール）
7. PNG を保存する（保存先: `/Users/hiroyuki/Desktop/画像/図解/`、命名: `zukai-YYYY-MM-DD-<トピック>.png`）
8. 一時HTMLファイルを削除する
9. ユーザーに「PNG を保存しました: <ファイル名>」と伝える

### Chrome ヘッドレス変換コマンド

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=old \
  --disable-gpu \
  --screenshot="/Users/hiroyuki/Desktop/画像/図解/<ファイル名>.png" \
  --window-size=750,<適切な高さ> \
  "file:///tmp/zukai-tmp.html" 2>/dev/null
```

高さはコンテンツ量に合わせて調整する（切れないよう余裕を持たせる）。

## スマホ最適化（最重要）

**図解はスマホで読まれることを前提に設計する。**

ブログに埋め込まれた PNG はスマホ画面（幅 375px 前後）に縮小表示される。PNG 幅が広いほどテキストが小さく見える。

### 基本ルール
- **PNG 幅は 750px に固定**（スマホ表示時の縮小率を 2x 以内に抑える）
- **フォントは大きめに設定**（下記サイズ基準を参照）
- **レイアウトは縦1列を優先**。2カラムグリッドはスマホで文字が詰まるため避ける
- **1要素に入れるテキストは最小限**。短いタイトル＋簡潔な説明にとどめる

### なぜ 750px か
| PNG幅 | スマホ表示倍率 | 22px本文 → 表示サイズ |
|--------|-------------|---------------------|
| 1100px | 約3x縮小 | 約7px（読めない） |
| 750px  | 約2x縮小 | 約11px（ギリギリ） |
| 600px  | 約1.6x縮小 | 約14px（読みやすい） |

## デザイン基準（必ず守ること）

### フォント
```css
@import url('https://fonts.googleapis.com/css2?family=M+PLUS+Rounded+1c:wght@400;500;700;800&display=swap');
font-family: 'M PLUS Rounded 1c', -apple-system, 'Hiragino Sans', sans-serif;
```

### 外枠カード（全図解共通）
```css
body {
  background: #f0f2f5;
  padding: 40px 24px;
  min-height: 100vh;
  display: flex; align-items: center; justify-content: center;
}
.zukai {
  background: #fff;
  border-radius: 20px;
  padding: 56px 64px;
  max-width: 900px;
  width: 100%;
  box-shadow: 0 4px 24px rgba(0,0,0,0.07);
}
.zukai-title {
  text-align: center; font-size: 34px; font-weight: 800;
  color: #2c2c3e; margin-bottom: 10px; letter-spacing: 0.5px;
}
.zukai-sub {
  text-align: center; font-size: 17px; color: #9a9ab0;
  margin-bottom: 40px; font-weight: 400;
}
```

### カラーパレット

**クリーン（デフォルト）:**
- アクセント: `#4a6cf7`（青）
- アクセント薄: `#f0f4ff`
- 成功・ゴール: `#22c55e`
- テキスト主: `#2c2c3e`
- テキスト補: `#7a7a95`
- 区切り線: `#ebebf0`

**ウォーム（`--warm` 指定時）:**
- アクセント: `#ff9800`（オレンジ）
- アクセント薄: `#fffaf5`
- 強調: `#e91e63`（ピンク）
- テキスト主: `#2c2c3e`（共通）

### フォントサイズの目安（スマホ最適化済み）
- 大見出し（タイトル）: 38〜42px / font-weight: 800
- 中見出し（要素タイトル）: 28〜32px / font-weight: 700
- 本文・説明: 22〜26px / font-weight: 400〜500
- 補足・ラベル: 18〜20px

※ PNG幅750pxでの値。1100pxを使う場合はさらに1.5倍に拡大すること。

## 自由に使えるレイアウトパターン（参考）

固定ではなく、必要なものを組み合わせて使う。**スマホ最適化のため縦1列レイアウトを優先すること。**

**スマホ向き（推奨）:**
- **番号付き縦リスト**: 順序・優先順位のある項目。最もスマホで読みやすい
- **縦フロー**: ステップを縦に矢印でつなぐ
- **まとめバナー**: 図解の下部に結論を1行で強調
- **ハイライトボックス**: 枠線付きで定義や重要情報を囲む

**横並び（慎重に使う）:**
- **2カラムグリッド**: 比較・並列要素の整理。スマホでは文字が詰まりやすいため、テキスト量を最小限にする
- **横並びフロー**: ステップ数が少ない（3以下）場合のみ使用
- **左右分割**: 定義＋詳細。テキストが短い場合のみ

**特殊:**
- **同心円**: 包含関係・階層構造
- **ミニフロー**: 小さな横並びで補助的な流れを示す

## 品質基準

- **スマホで読めるか**を最初に確認する（最重要）
- 読者が「一目で理解できる」ことを優先する
- テキストの詰め込みすぎ禁止。1要素に入れる文字数は最小限に
- 余白を十分に取る（詰まった図解は読みにくい）
- PNG は高さが切れないよう `--window-size` の高さを余裕を持って設定する
- 絵文字禁止。テキストとCSSのみで表現する

---

## バックアップ仕様

### 保存先ディレクトリ
```
/Users/hiroyuki/Desktop/画像/図解/backup/
```
ディレクトリが存在しない場合は `mkdir -p` で作成する。

### ファイル命名規則
```
backup_<トピック>_v<連番>.json
例: backup_副業の始め方_v1.json
    backup_副業の始め方_v2.json
```
連番は同トピックの既存バックアップ数 + 1 で決める。

### JSON 構造
```json
{
  "version": 1,
  "timestamp": "2026-05-13T12:00:00",
  "topic": "副業の始め方",
  "png_filename": "zukai-2026-05-13-副業の始め方.png",
  "html": "<HTMLソース全体（/tmp/zukai-tmp.htmlの内容）>"
}
```

### バックアップ作成手順（ステップ5の詳細）
1. バックアップディレクトリを確認・作成する
2. 同トピックの既存バックアップ数を `ls` で確認し、次の連番を決める
3. `/tmp/zukai-tmp.html` の内容を読み込み、JSON に埋め込む
4. JSON ファイルを書き出す
5. **最大保存世代数は5件**。超えた場合は最も古いバージョン（v1）を削除し、残りを v1 から振り直す

### バックアップ作成の Bash コマンド例
```bash
# バックアップディレクトリ作成
mkdir -p "/Users/hiroyuki/Desktop/画像/図解/backup"

# 既存バックアップ数を確認して次の連番を取得
BACKUP_DIR="/Users/hiroyuki/Desktop/画像/図解/backup"
TOPIC="副業の始め方"
COUNT=$(ls "$BACKUP_DIR"/backup_${TOPIC}_v*.json 2>/dev/null | wc -l | tr -d ' ')
NEXT_V=$((COUNT + 1))

# 世代数が5を超える場合は最古を削除して振り直し
if [ "$NEXT_V" -gt 5 ]; then
  rm "$BACKUP_DIR/backup_${TOPIC}_v1.json"
  for i in 2 3 4 5; do
    mv "$BACKUP_DIR/backup_${TOPIC}_v${i}.json" \
       "$BACKUP_DIR/backup_${TOPIC}_v$((i-1)).json" 2>/dev/null || true
  done
  NEXT_V=5
fi
```

---

## リカバリー機能

`--recover` フラグが指定された場合は、通常の図解生成をスキップしてリカバリーモードで動作する。

### リカバリーワークフロー

1. バックアップディレクトリ内の対象トピックのファイルを一覧表示する
   ```
   利用可能なバックアップ:
     v1: backup_副業の始め方_v1.json  (2026-05-12T10:30:00)
     v2: backup_副業の始め方_v2.json  (2026-05-13T09:15:00)  ← 最新
   ```
2. バージョン指定なし（`--recover` のみ）の場合は**最新バージョン（最大連番）を自動選択**する
3. バージョン指定あり（`--recover v2` など）の場合は指定バージョンを使用する
4. JSON から `html` フィールドを取り出し、`/tmp/zukai-tmp.html` に書き出す
5. Chrome ヘッドレスで PNG を再生成する（ファイル名は JSON の `png_filename` を使用）
6. 一時ファイルを削除する
7. 復元完了をユーザーに通知する：
   ```
   復元完了: zukai-2026-05-13-副業の始め方.png
   ソース: backup_副業の始め方_v2.json (2026-05-13T09:15:00)
   ```

### バックアップが存在しない場合
対象トピックのバックアップが見つからない場合は、エラーを伝えて処理を中断する：
```
バックアップが見つかりませんでした: 副業の始め方
通常の図解生成を行う場合は --recover を外して再度実行してください。
```
