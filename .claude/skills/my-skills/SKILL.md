---
name: my-skills
description: Use when the user says "スキル一覧を呼び出して", "スキル一覧を見せて", "自作スキルを確認したい", "スキルを一覧表示して" or any similar request to view or browse the list of user-created custom skills.
---

# 自作スキル一覧

以下のコマンドを実行し、最新の自作スキル一覧を表示する。

## 実行手順

1. `~/.claude/skills/` のサブディレクトリを列挙し、各 `SKILL.md` から `name:` と `description:` を取得する
2. `~/.claude/commands/` のファイルを列挙し、各ファイルから `name:` と `description:` を取得する
3. プロジェクトの `.agents/skills/` を確認し（存在すれば）、同様に取得する
4. 以下のフォーマットで表示する

## 表示フォーマット

```
## グローバルスキル（~/.claude/skills/）
| スキル名 | 用途 |
...

## グローバルコマンド（~/.claude/commands/）
| スキル名 | 用途 |
...

## プロジェクトスキル（.agents/skills/）  ← 存在する場合のみ
| スキル名 | 用途 |
...
```

## コマンド例

```bash
# スキル一覧取得
grep "^name:\|^description:" ~/.claude/skills/*/SKILL.md
grep "^name:\|^description:" ~/.claude/commands/*.md
```

スキルが追加・削除されても常に最新の状態を反映する。
