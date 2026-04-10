---
title: "CDK環境変数の罠・X API文字数仕様・VBS二重起動防止など、開発中に詰まったポイントまとめ"
emoji: "🛠️"
type: "tech"
topics:
  - AWS
  - CDK
  - TypeScript
  - TwitterAPI
  - VBScript
published: true
---

## はじめに

PC監視アプリの開発を進める中で、CDKの環境変数、X API、VBScriptなど複数の技術的なハマりポイントに遭遇しました。同じ罠にはまる方が少なくなるよう、詰まった内容と解決策をまとめます。

---

## 作業内容

### PC監視アプリの機能改善

以下の課題に取り組みました。

- CDK環境変数の更新問題
- X（Twitter）API投稿時の文字数カウント実装
- VBSランチャーの二重起動防止
- AWS Cost ExplorerによるAWSコスト表示機能の実装

---

## 学んだこと

### 1. CDKの環境変数「空文字→空文字」はdiffとして検知されない

CDKでLambdaの環境変数を管理している際、空文字から空文字への変更は差分として検知されません。
そのためAWS CLIで直接Lambdaの環境変数を更新しようとした際、シェル変数経由で渡すと意図しない挙動になるケースがありました。

**解決策：** シェルスクリプト経由ではなく、Node.jsスクリプトなどで値を明示的に構築してから渡すことで、安全に環境変数を更新できます。

```javascript
// 例: Node.jsスクリプトで環境変数を安全に更新
const { LambdaClient, UpdateFunctionConfigurationCommand } = require('@aws-sdk/client-lambda');

const client = new LambdaClient({ region: 'ap-northeast-1' });
const command = new UpdateFunctionConfigurationCommand({
  FunctionName: 'your-function-name',
  Environment: {
    Variables: {
      MY_VAR: 'new-value',
    },
  },
});
await client.send(command);
```

---

### 2. X（Twitter）APIの文字数カウント仕様

X APIでは、URLはt.coによって短縮され、**長さに関わらず一律23文字**としてカウントされます。
投稿上限は280文字のため、URLを含む投稿文の文字数を正確に計算する際はこの仕様を必ず考慮する必要があります。

```typescript
function calcTweetLength(text: string): number {
  const urlRegex = /https?:\/\/\S+/g;
  const replaced = text.replace(urlRegex, '_'.repeat(23));
  return [...replaced].length; // Unicode対応
}
```

この仕様を知らずに実装すると、実際の文字数と表示上の文字数がずれて投稿エラーになるケースがあるため注意が必要です。

---

### 3. VBScriptでUTF-8日本語コメントを書くとパースエラーになる

VBScriptのファイルにUTF-8エンコードの日本語コメントを含めると、Windowsのスクリプトエンジンがパースエラーを起こすことがあります。

**解決策：** VBSファイル内のコメントは**英語のみ**にする。どうしても日本語メモを残したい場合は、別途README等に記載するのが無難です。

---

### 4. AWS Cost ExplorerでサービスごとのコストをUSD・円換算で取得

AWS Cost Explorer APIを使い、サービス別のコストをUSDおよび円換算で取得・表示する機能を実装しました。

なお、Anthropic（Claude）のコストをSDKレベルで取得しようとしましたが、AnthropicのAdmin APIキーは**個人アカウントでは発行できない**仕様のため、SDK経由でのコスト取得は現状難しいことが確認できました。AWSコンソール上のCost Explorerで確認するのが現実的な運用になりそうです。

```typescript
// Cost Explorer APIでサービス別コストを取得する例
const command = new GetCostAndUsageCommand({
  TimePeriod: { Start: startDate, End: endDate },
  Granularity: 'MONTHLY',
  Metrics: ['UnblendedCost'],
  GroupBy: [{ Type: 'DIMENSION', Key: 'SERVICE' }],
});
```

---

## まとめ

| 項目 | ポイント |
|------|----------|
| CDK環境変数 | 空文字→空文字の変更はdiff検知されない。Node.jsスクリプト経由で更新を |
| X API文字数 | URLは23文字固定カウント。280文字制限に注意 |
| VBScript | UTF-8日本語コメントはパースエラーの原因。コメントは英語で |
| AWS Cost Explorer | サービス別コスト取得は可能。AnthropicはAdmin API非対応 |

細かい仕様のハマりポイントほど、ドキュメントに明記されていないことも多いです。同じところで詰まっている方の参考になれば幸いです。
