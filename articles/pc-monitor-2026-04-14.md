---
title: "PlaywrightとVitestの共存で学んだテスト環境の落とし穴と改善記録"
emoji: "🧪"
type: "tech"
topics:
  - Playwright
  - Vitest
  - TypeScript
  - Next.js
  - テスト
published: true
---

## はじめに

PC監視アプリの開発を進める中で、テスト環境の整備と分析機能のリファクタリングを行った。PlaywrightによるE2Eテスト（12件）とVitestによるユニットテストを運用する中で、いくつかの実践的な知見が得られたのでまとめておく。

---

## 作業内容

### 1. PlaywrightによるE2EテストとVitestによるユニットテスト

まず、既存のユニットテストが全て通ることを確認した。その後、E2Eテストを12件実行してコミットまで完了させた。

テスト実行の過程で、PlaywrightとVitestを同時に走らせると問題が起きることに気づいた（後述）。

---

## 学んだこと

### ① PlaywrightとVitestの同時実行でメモリ不足になる

`pnpm test` でE2EテストとユニットテストをまとめてCIっぽく実行しようとしたところ、メモリ不足でクラッシュするケースに遭遇した。(自宅PCはメモリを8GBしか搭載していないため)

原因として判明したのが、**Claude Codeのクラッシュ後にNodeプロセスが残留**していたこと。これによってメモリが圧迫され、新たなテストプロセスが起動できなかった。

対処法：
- VSCodeを再起動する
- タスクマネージャーで残留している `node.exe` を終了する

また、E2Eテストを除外してVitestのみを実行したい場合は以下のオプションが有効。

```bash
pnpm vitest --exclude e2e/**
```

PlaywrightとVitestは並列実行よりも、スクリプトを分けて順次実行するほうが安定しやすい。

---

### ② テストレポートの確認方法

テスト結果をブラウザで確認する方法が2通りあることを整理した。

**Playwright（E2E）のレポート表示**

```bash
pnpm exec playwright show-report e2e-results
```

このコマンドで、Playwrightが生成したHTMLレポートをローカルブラウザで確認できる。テストの成功・失敗・スクリーンショットなどが一覧表示される。

**Vitest（UT）のレポート表示**

```bash
npx vite preview --outDir test-results
```

こちらはVitestのカバレッジや結果をローカルサーバー経由で確認する方法。

どちらもCLIのログだけでなく、ビジュアルなレポートで確認できるのは開発体験として嬉しい。

---

### ③ PlaywrightのE2EテストでNext.jsサーバーの起動待ちが発生する

Playwrightのテストを実行する際、Next.jsの開発サーバーが立ち上がっていないとテストが失敗する。これを解決するには `playwright.config.ts` に `webServer` の設定を記述する。

```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  webServer: {
    command: 'pnpm dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
  // ...
});
```

これにより、テスト実行前にNext.jsサーバーが自動で起動し、サーバーが準備できてからテストが始まる。`reuseExistingServer` を `true` にしておくと、すでに起動中のサーバーを再利用してくれるため、ローカル開発時の二重起動を防げる。

---

### ④ Playwrightのテストプロセスは `.env.local` を自動で読み込まない

これは見落としがちなポイント。Next.jsは `.env.local` を自動で読み込んでくれるが、**Playwrightのテストプロセスは別のNode.jsプロセス**として動作するため、この恩恵を受けられない。

テストコード内で `process.env.API_KEY` などの環境変数を参照している場合は、`dotenv` を使って明示的に読み込む必要がある。

**インストール**

```bash
pnpm add -D dotenv
```

**playwright.config.ts の先頭に記述**

```ts
import * as dotenv from 'dotenv';
import path from 'path';

dotenv.config({ path: path.resolve(__dirname, '.env.local') });
```

これだけで、テストコード内から `.env.local` の値を参照できるようになる。

**使い分けの判断基準**

| ケース | dotenv が必要か |
|--------|----------------|
| テストコードで `process.env.〇〇` を参照する | ✅ 必要 |
| 環境変数を一切使わないテスト | ❌ 不要 |

Next.jsサーバーとPlaywrightのテストプロセスは別物という認識を持っておくと、この手の問題でハマることが減る。

---

### ⑤ 型定義変更時のモックデータ修正は漏れに注意

今回 `AiResult` 型から `suggestions` フィールドを削除したところ、テストファイル4件のモックデータ修正が必要になった。

TypeScriptの型エラーが変更箇所を教えてくれるのは強みだが、`as unknown as AiResult` のような型アサーションを使っているモックデータは型チェックをすり抜けてしまう場合がある。モックデータの管理は一箇所にまとめておくと、型変更時の影響範囲を最小化できる。

```ts
// ❌ 型アサーションで誤魔化すと変更に気づきにくい
const mockData = { suggestions: [] } as unknown as AiResult;

// ✅ 型に忠実なモックデータで書いておく
const mockData: AiResult = {
  summary: '...',
  // suggestions は削除済みなのでここには書かない
};
```

---

## まとめ

テスト環境の整備は「動けばいい」ではなく、**安定して実行できる環境を作ること**が重要だと改めて感じた作業だった。特に印象に残ったポイントを振り返る。

- **PlaywrightとVitestの同時実行はメモリ問題を引き起こす**ことがある。残留プロセスにも注意。
- **テストプロセスはNext.jsとは別プロセス**であることを意識すると、環境変数まわりのハマりが防げる。
- **`playwright.config.ts` の `webServer` 設定**を使うと、Next.jsサーバーの起動待ちを自動化できる。
- **型定義を変更したら、モックデータの修正漏れ**がないか必ずチェックする。

地道な作業だが、こうした基盤を整えておくことでその後の開発速度と品質に大きく影響する。テスト環境の整備はコストに見えて、実は最大の投資だと感じている。
