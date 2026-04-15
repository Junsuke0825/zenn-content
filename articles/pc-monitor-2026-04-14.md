---
title: "VitestとPlaywrightを組み合わせたTDDサイクルを実践してみた"
emoji: "🧪"
type: "tech"
topics:
  - TypeScript
  - Vitest
  - Playwright
  - TDD
  - テスト
published: true
---

## はじめに

PC監視アプリの開発において、VitestによるユニットテストとPlaywrightによるE2Eテストを組み合わせたTDD（テスト駆動開発）サイクルを実践した。わずか24分という短い時間の中でも、テストとコードの往復を繰り返すリズムが体に染み込んでくる感覚があったので、その流れをまとめておく。

## 作業内容

### 対象プロジェクト

TypeScriptで構築したPC監視アプリ（`20260331_PC監視アプリ`）に対して、前日深夜から続けていたテスト追加作業の仕上げを実施した。

- **Vitestのユニットテスト**: 192件
- **PlaywrightのE2Eテスト**: 12件

### TDDサイクルの流れ

実践したワークフローは以下のループだ。

```
1. VSCodeで reportData.ts / index.html を編集
2. ChromeでVitest Test UIを開いてテスト結果を確認
3. 失敗しているテストをもとにコードを修正
4. Playwright Test Reportをブラウザで確認しE2Eの結果をレビュー
5. 問題なければコミット
```

#### Vitest Test UIの活用

Vitestにはブラウザ上でテスト結果を確認できるUIモードがある。ターミナルのログだけを追うよりも、どのテストが失敗しているかを視覚的に把握しやすく、修正箇所の特定が素早くできた。

```bash
npx vitest --ui
```

#### Playwright Test Reportの確認

E2Eテストが完了したあと、HTMLレポートをブラウザで開いてスクリーンショットやトレースを確認した。

```bash
npx playwright show-report
```

junit.xml形式でのレポート出力も合わせて行い、CI連携を見据えた構成にしている。`playwright.config.ts` に以下のように設定することでjunit.xmlを出力できる。

```typescript
reporter: [
  ['html'],
  ['junit', { outputFile: 'junit.xml' }]
],
```

### 編集したファイル

| ファイル | 役割 |
|---|---|
| `reportData.ts` | レポート表示用データの型定義・整形ロジック |
| `index.html` | レポートのエントリポイント |
| `junit.xml` | CI用テスト結果レポート |

## 学んだこと

### ツールの視覚的フィードバックがTDDの回転を速める

Vitest UIとPlaywright Test Reportをブラウザで並べながら作業することで、「コードを変えた→結果がどう変わったか」のフィードバックが非常に速くなった。ターミナルだけで作業するより、視覚的に状態を把握できるUIツールを活用する方がTDDのリズムを保ちやすいと実感した。

### junit.xmlはCIへの橋渡し

Playwrightのjunit.xml出力は、GitHub ActionsなどのCIツールと連携する際に便利だ。テスト結果をCI上でパース・可視化できるようになるため、ローカルでの確認だけでなくチーム共有の観点でも重要な設定だと改めて理解できた。

### 短時間でもTDDサイクルは回せる

今回の作業時間は約24分と短い。それでもVSCode・Vitest UI・Playwright Test Reportを組み合わせることで、複数のテスト修正とコミットまでを完結させることができた。TDDは長時間取れないと効果が薄いというイメージがあったが、ツールの整備さえしておけばスキマ時間でも有効に機能すると気づいた。

## まとめ

- VitestのUI modeとPlaywright Test Reportを併用することで、TDDのフィードバックループを高速化できる
- junit.xmlの出力設定をしておくと、CI連携への準備が整う
- ツールの視覚的フィードバックを最大限活用することが、短時間でのTDD実践のカギになる

テスト環境を整備しておくと、作業時間が短くても質の高い開発サイクルを維持できる。まだVitestやPlaywrightのUIツールを使っていない人はぜひ試してみてほしい。
