---
title: "ローカルOCRをClaude Haiku Visionに置き換えたら分析精度が劇的に向上した話"
emoji: "🔍"
type: "tech"
topics:
  - SQLite
  - Claude
  - Electron
  - TypeScript
  - 個人開発
published: true
---

## はじめに

個人で開発しているPC監視アプリの品質改善に丸一日を費やしました。スクリーンショットの取得方法の変更、OCRエンジンの刷新、DBスキーマの整理など、いわゆる「技術的負債の返済日」です。

地味な作業が多い一日でしたが、得られた知見はどれも実践的なものばかりだったので、同じような課題に直面しているエンジニアの方に向けて共有します。

## 本日の作業内容

作業時間は約400分（6時間40分）。その内訳はプログラミングが約270分（67.5%）、Web調査・学習が約62分（15.5%）、残りがドキュメント作成や環境設定でした。

大きく分けて以下の4つの改善を行いました。

### 1. スクリーンショット取得を「ディスプレイ全体」から「アクティブウィンドウ単位」に変更

これまで `screenshot-desktop` を使ってディスプレイ全体をキャプチャしていましたが、これだとノイズが多く、後段の分析精度に悪影響がありました。

調査の結果、`node-screenshots` ライブラリを導入することでアクティブウィンドウ単位のキャプチャが可能になりました。

```bash
pnpm add node-screenshots
```

`node-screenshots` はネイティブモジュールを含むため、`pnpm` 環境では `.pnpm-build-approvals.json` にビルド承認の記録が生成されます。このファイルはローカル固有のものなので `.gitignore` に追加しておきましょう。

### 2. ローカルOCR（Tesseract.js）→ Claude Haiku Vision API への切り替え

今回の改善で最もインパクトが大きかったのがこれです。

もともと `Tesseract.js` でローカルOCRを行っていましたが、**日本語UIの画面キャプチャに対する認識精度がかなり低い**という問題がありました。特にアイコンとテキストが混在するような一般的なアプリケーション画面では、誤認識が頻発していました。

そこで、OCR部分を丸ごと **Claude Haiku Vision API** に置き換えました。

```typescript
import Anthropic from "@anthropic-ai/sdk";

const anthropic = new Anthropic();

async function analyzeScreenshot(imageBase64: string): Promise<string> {
  const response = await anthropic.messages.create({
    model: "claude-haiku-4-20250414",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "image",
            source: {
              type: "base64",
              media_type: "image/png",
              data: imageBase64,
            },
          },
          {
            type: "text",
            text: "この画面キャプチャに表示されている内容を簡潔に説明してください。",
          },
        ],
      },
    ],
  });

  return response.content[0].type === "text" ? response.content[0].text : "";
}
```

Vision APIは単純なOCR（文字起こし）ではなく、画面の**文脈を理解した分析**を返してくれます。「VSCodeでTypeScriptファイルを編集中」「Chromeでドキュメントを閲覧中」といった粒度の情報が得られるようになり、活動ログの分析精度が劇的に向上しました。

Haikuモデルを使うことでコストも抑えられます。画像1枚あたり数円程度なので、個人プロジェクトでも十分に現実的な範囲です。

### 3. SQLiteのカラム削除とテーブル再構築

OCRの切り替えに伴い、不要になった `ocr_text` カラムと `source` カラムを削除する必要がありました。ここでハマったポイントが一つ。

**SQLiteでは `ALTER TABLE DROP COLUMN` が使えません**（SQLite 3.35.0以降では限定的にサポートされていますが、制約が多い）。

そのため、以下のようなテーブル再構築パターンで対応しました。

```sql
-- 1. 新しいテーブルを作成（不要カラムを除外）
CREATE TABLE screenshots_new (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  window_title TEXT,
  image_path TEXT,
  analysis TEXT,
  created_at TEXT DEFAULT (datetime('now', '+9 hours'))
);

-- 2. データを移行
INSERT INTO screenshots_new (id, window_title, image_path, analysis, created_at)
SELECT id, window_title, image_path, analysis, created_at FROM screenshots;

-- 3. 旧テーブルを削除
DROP TABLE screenshots;

-- 4. リネーム
ALTER TABLE screenshots_new RENAME TO screenshots;
```

この手法はSQLiteの公式ドキュメントでも推奨されている正攻法です。

### 4. created_at のJST化

SQLiteの `datetime('now')` は **UTCを返す** という仕様があります。日本時間で保存したい場合は明示的にオフセットを指定する必要があります。

```sql
-- UTC（デフォルト）
datetime('now')  -- → 2025-01-15 03:00:00

-- JST
datetime('now', '+9 hours')  -- → 2025-01-15 12:00:00
```

全テーブルのDEFAULT値を一括でJSTに変更しました。本来はアプリケーション側でタイムゾーンを扱うべきという意見もありますが、今回はローカル専用ツールなのでDB側でJST固定としています。

## 学んだこと・気づき

今日の作業を通じて得た知見をまとめます。

| テーマ | 学び |
|--------|------|
| スクリーンショット | `screenshot-desktop` はディスプレイ単位のみ。ウィンドウ単位なら `node-screenshots` が有効 |
| OCR | Tesseract.js の日本語認識精度は実用レベルに達していない場面が多い。Vision API の方が文脈理解を含めて優秀 |
| SQLite | カラム削除はテーブル再構築が基本。`datetime('now')` はUTCなので注意 |
| pnpm | `.pnpm-build-approvals.json` はgit管理不要 |

特に **「ローカルOCRにこだわらず、Vision APIに任せる」** という判断は、精度・開発速度・保守性のすべてにおいてプラスでした。ローカル処理にこだわりすぎて品質を犠牲にするより、APIコストを許容して精度を取る方が合理的なケースは多いと感じます。

## アーキテクチャ簡素化の効果

今回の一連の改善で、`windowWatcher` モジュールを廃止しました。アクティブウィンドウの情報はスクリーンショット取得時に `node-screenshots` から直接得られるため、別モジュールで監視する必要がなくなったためです。

モジュールが一つ減るだけで、データの流れがシンプルになり、デバッグも格段にしやすくなりました。**機能追加よりも不要なものを削る方が、コードベースの健全性には効く**という実感があります。

## まとめ

今日は新機能の開発ではなく、既存コードの品質改善に集中しました。一見地味ですが、こうした技術的負債の返済を定期的に行うことで、将来の開発速度を維持できます。

特に以下の3点は、同じような個人開発をしている方にも参考になるはずです。

1. **ローカルOCRの精度に不満があるなら、Vision APIへの切り替えを検討する価値がある**
2. **SQLiteのカラム削除はテーブル再構築で対応する**（知らないとハマる）
3. **不要になったモジュールは積極的に廃止する**（シンプルさは正義）

明日以降は、この改善されたベースの上で新機能の開発を進めていく予定です。
