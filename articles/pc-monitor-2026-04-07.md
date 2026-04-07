---
title: "PC監視アプリを1日で大改造してみた — SQLite JST化・Vision API移行・コスト最適化まで"
emoji: "🛠️"
type: "tech"
topics:
  - SQLite
  - ClaudeAPI
  - Node.js
  - TypeScript
  - 個人開発
published: true
---

## はじめに

自作のPC監視アプリを「動く状態」から「ちゃんと使える状態」へと引き上げるべく、1日かけて集中的にリファクタリングを行いました。SQLiteの日時問題・スクリーンショット取得方法の見直し・OCRからClaude Vision APIへの移行・コスト最適化まで、詰め込んだ内容をまとめます。

---

## 本日の作業内容

### 午前：SQLite JST化 & テーブルリファクタリング

ログの時刻がずっとUTCで記録されていたため、全テーブルのタイムスタンプをJSTに統一しました。

SQLiteの `datetime('now')` はデフォルトでUTCを返します。JST（UTC+9）で取得するには以下のように書く必要があります。

```sql
SELECT datetime('now', '+9 hours');
```

またSQLiteにはMySQLやPostgreSQLのような `ALTER TABLE DROP COLUMN` が（古いバージョンでは）使えません。カラム削除が必要な場合は、**テーブル再構築**で対応します。

```sql
-- 1. 新テーブルを作成
CREATE TABLE new_table AS SELECT col1, col2 FROM old_table WHERE 1=0;

-- 2. データを移行
INSERT INTO new_table (col1, col2)
SELECT col1, col2 FROM old_table;

-- 3. 旧テーブルを削除して名前を変更
DROP TABLE old_table;
ALTER TABLE new_table RENAME TO old_table;
```

地味ですが、知らないとハマるポイントです。

---

### 午後：アクティブウィンドウ専用キャプチャ & Vision API移行

これまで全画面スクリーンショット＋Tesseract OCRで画面内容を取得していましたが、2つの課題がありました。

- **精度が低い**：日本語OCRの認識精度に限界がある
- **余計な情報が多い**：全画面だとタスクバーや無関係なウィンドウも混入する

そこで以下の変更を実施しました。

**① アクティブウィンドウのみキャプチャ**

`node-screenshots` パッケージを使うことで、Windowsでもアクティブウィンドウだけを切り出せることがわかりました。当初候補だった `windows-ss` はNode.js v20向けのビルド済みバイナリがなく断念。`node-screenshots` は事前ビルド済みバイナリが充実しており、すんなり動きました。

```typescript
import { Screenshots } from 'node-screenshots';

const activeWindow = Screenshots.fromActiveWindow();
const image = activeWindow?.captureSync();
```

**② OCR → Claude Vision API（Haiku）へ移行**

キャプチャ画像をbase64エンコードしてClaude APIに渡すだけで、画面の内容を自然言語で説明してくれます。Tesseractと比べて圧倒的に精度が高く、コンテキストも理解してくれるのが強みです。

---

### 夜間：chat_logsのタグ除去 & コスト最適化

**Claude CodeのJSONLログ処理**

Claude CodeのログはJSONL形式（1行1イベント）で出力されますが、VSCodeが自動挿入する `ide_opened_file` や `ide_selection` タグが混入していました。取り込み時にこれらを除去するフィルタ処理を追加しています。

**モデルをOpus → Sonnetに変更**

ログ分析用のモデルをClaude Opus から Claude Sonnet に切り替えました。コストはおよそ **1/5** になります。分析タスクの精度的にもSonnetで十分と判断しました。日常的なバッチ処理ではモデル選定がランニングコストに直結するため、定期的に見直す価値があります。

---

## 学んだこと まとめ

| テーマ | 学び |
|---|---|
| SQLite | カラム削除はテーブル再構築で対応 |
| SQLite | `datetime('now', '+9 hours')` でJST取得 |
| Node.js | アクティブウィンドウ取得は `node-screenshots` が安定 |
| Claude API | Opus→Sonnetでコスト約1/5、分析用途ならSonnetで十分 |
| JSONL処理 | VSCode自動挿入タグの除去処理が必要 |

---

## まとめ

「動くけど微妙」な状態のアプリを1日で大幅改善できました。特にSQLiteのJST化とVision API移行は、データの信頼性と解析品質の両方を底上げできた実感があります。

明日はS3アップロード処理のタイミング問題と、DynamoDBの重複書き込みバグの調査を進める予定です。自作ツールの改善は終わりがなくて楽しいですね。
