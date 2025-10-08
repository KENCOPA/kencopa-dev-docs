---
type: note
moc: "[[🏷️research]]"
tags:
- ai/ref
---
> [!info] 必要なセクションを選択して使用してください！

# 📖 背景・なぜ今これを検討するのか

FE/BE分離アプリケーションにおいて、エラーハンドリングが重要な課題となっている。

現在の問題：
- 同一ステータスコードで複数の異なるビジネスロジックエラーが発生（例：badrequestが複数箇所で発生）
- FE側で適切なエラー処理を行うための情報が不足
- エラーメッセージの管理場所が不明確（BE/FEどちらで管理すべきか）
- エラーコードの採番やFE側での処理分岐方法が未定義

# 🔍 現状・今どうなっているか

**現在のエラーレスポンス構造**：
- HTTPステータスコード（400、401、403、404、500等）
- メッセージ文字列
- 一部でdetailsオブジェクト（フィールド情報等）

**現在の課題**：
- 同じステータスコードで異なるエラー原因を区別できない
- FE側でエラー種別に応じた適切な処理ができない
- エラーメッセージの国際化対応が困難
- バリデーションエラーとビジネスロジックエラーの区別が曖昧

# ❓ 主要な論点・検討したいポイント

## 論点1: エラーメッセージの管理方法（FE vs BE）

エラーメッセージをFE、BEどちらで管理するか、その方法について検討します。

**検討ポイント**：
- [ ] **管理場所**: BE側で管理 vs FE側で管理、どちらが適切か？
- [ ] **エラーコードの扱い**: BEからのレスポンスに含めるか、FEに表示させるか？そもそもエラーコード使用するか
  - [ ] **エラーコード体系**: 階層的エラーコード（USER.LOGIN.INVALID_CREDENTIALS）vs 数値ベース（1001）どちらが良いか？
- [ ] **FE側の実装負荷**: エラーハンドリングの複雑性vs情報の豊富さのバランス

## 論点2: システム的エラーの管理方法

ユーザーに表示するメッセージとは別に、システム的なエラー（外部APIタイムアウト、DB接続エラー等）をどう管理するかについて検討します。

**検討ポイント**：
- [ ] **技術的エラーの抽象化**: 外部APIタイムアウト、DB接続エラー等をユーザーフレンドリーなメッセージに変換する方法
- [ ] **ログ管理**: 技術的メッセージの管理方法（レスポンスに含める vs ログシステムで分離管理）
- [ ] **エラー紐付け**: ユーザー向けメッセージと技術的メッセージの紐付け方法

## その他の検討事項

- [ ] **エラーレスポンス構造**: 統一されたAPIError構造の定義方法
- [ ] **段階的移行**: 既存APIからの移行戦略（全面刷新 vs 段階的追加）
- [ ] **バリデーションエラーの扱い**: フィールドレベルエラーの特別扱いが必要か？

# 🎯 期待する結果・どんな意見がほしいか

- [x] **技術的なアドバイス**: 経験や知識に基づく助言がほしい
- [x] **方向性の決定**: どの方向に進むべきか決めたい
- [x] **実装方法の相談**: 具体的な実装手法について相談したい

# エラーメッセージの管理方法 (FE vs BE)

## 選択肢A: BE中心メッセージ管理
**概要**: エラーメッセージをBE側で管理し、レスポンスに含める  

**メリット**:
- 一元管理で整合性確保
- FEの実装が簡潔

**デメリット**:
- BE側の責任範囲が広がる
- レスポンスサイズが大きくなる可能性

## 選択肢B: FE側メッセージ管理
**概要**: エラーコードのみをBEから返し、FE側でメッセージ管理  

**メリット**:
- FEが表示制御しやすい
- レスポンスサイズが小さい
- BE側の責任が軽減

**デメリット**:
- FE/BE間でエラーコード定義の同期が必要

## エラーコードを含めるか/どのような形式にするか

### エラーコードを含める場合
**メリット**:
- エラーの種類を明確に識別できる
- FE側でのエラーハンドリングが容易
- デバッグ時にエラーの原因特定が容易
- エラーコードによる条件分岐が可能

**デメリット**:
- レスポンスサイズが若干増加
- エラーコードの命名規則や管理が必要
- エラーコードの一意性を保つ必要がある

### エラーコードを含めない場合
**メリット**:
- レスポンスサイズが最小限
- エラーコード管理のコストがない
- 実装がシンプル

**デメリット**:
- エラーの種類を識別するのが困難
- FE側でのエラーハンドリングが複雑
- デバッグ時の原因特定が困難
- エラー種別による条件分岐ができない

## エラーコードの形式

### 数値ベース（例：E-100001, E-100002）
**メリット**:
- 一意性が保ちやすい
- データベースでの管理が容易

**デメリット**:
- エラーコードの意味が直感的でない
- デバッグ時にエラーの内容を理解しにくい
- エラーコードの採番ルールが必要

### 階層的エラーコード（例：USER.LOGIN.INVALID_CREDENTIALS）
**メリット**:
- エラーコードの意味が直感的
- デバッグ時にエラーの内容を理解しやすい
- ドメイン別の整理が容易

**デメリット**:
- エラーコードが冗長になる可能性
- 命名規則の統一が必要

## 懸念事項




# システム的エラーの管理方法

> [!info]
> 
[[💭 FE<->BE間のエラーハンドリング方法について#論点1 エラーメッセージの管理方法（FE vs BE）]]でBEでメッセージ管理、[[💭 FE<->BE間のエラーハンドリング方法について#エラーコードを含めるか/どのような形式にするか]]でエラーコードを含める前提で以下を記載している


- [ ] 既存エラークラスを拡張し、`technicalMessage?`を持てば良い



# その他のアイデア・選択肢募集中 💭

[その他のアイデアや組み合わせパターンがあれば、コメントで教えてください]

# 🔗 関連情報・参考資料

## ベストプラクティス・実装例

### 統一されたエラーレスポンス構造
```typescript
interface APIError {
  statusCode: number;
  errorCode: string;
  message: string;
  timestamp: string;
  path: string;
  details?: {
    field?: string;
    constraint?: string;
    value?: any;
  };
}
```

### バリデーションエラー専用構造
```typescript
interface ValidationError extends APIError {
  errorCode: 'VALIDATION_ERROR';
  details: {
    fieldErrors: Array<{
      field: string;
      code: string;
      message: string;
    }>;
  };
}
```

### FE側での統一エラーハンドリング
```typescript
class APIClient {
  async handleError(response: Response) {
    const error: APIError = await response.json();
    
    switch (error.errorCode) {
      case 'USER.LOGIN.INVALID_CREDENTIALS':
        return this.handleLoginError(error);
      case 'VALIDATION_ERROR':
        return this.handleValidationError(error);
      default:
        return this.handleGenericError(error);
    }
  }
}
```

### エラーコード一意性チェック
```typescript
const createErrorCodes = <T extends Record<string, string>>(codes: T): T => {
  const values = Object.values(codes);
  const unique = new Set(values);
  if (values.length !== unique.size) {
    throw new Error('Duplicate error codes detected');
  }
  return codes;
};
```

## 参考記事・リソース
- **[REST API Error Handling Best Practices](https://blog.restcase.com/rest-api-error-codes-101/)**
- **[RFC 7807 - Problem Details for HTTP APIs](https://tools.ietf.org/html/rfc7807)**
- **[Microsoft REST API Guidelines - Error Handling](https://github.com/Microsoft/api-guidelines/blob/vNext/Guidelines.md#7102-error-condition-responses)**
- **[Google API Design Guide - Errors](https://cloud.google.com/apis/design/errors)**
- **[Stripe API Error Handling](https://stripe.com/docs/api/errors)** (良い実装例)
- **[GitHub API Error Responses](https://docs.github.com/en/rest/overview/resources-in-the-rest-api#client-errors)** (実装参考)
- **[OpenAPI 3.0 Error Response Specification](https://swagger.io/specification/#response-object)**
- **[HTTP Status Code Decision Tree](https://github.com/for-GET/know-your-http-well/blob/master/status-codes.md)**

## 関連するプロジェクト
- **現在のkunai-backendエラーハンドリング実装**
- **フロントエンド側のエラー処理要件**

---

# 💬 ディスカッション・コメント

[この下にコメントが追加されます]

# 📝 検討過程メモ

[検討中に気づいたことや、議論の流れをメモしてください]

# ✅ 結論・決定事項（検討完了後に記載）

[検討が完了したら、ここに結論を記載してください]

## 検討結果

- エラーメッセージの管理
	- 💭 FE<->BE間のエラーハンドリング方法について#選択肢A BE中心メッセージ管理をとる
- エラーコードを含める
	- 💭 FE<->BE間のエラーハンドリング方法について#エラーコードを含めるか/どのような形式にするか
	- "LOGIN.INVALID_CREDENTIALS"の形式(階層型)
		- 各ドメインごとにフォルダを分けて定義するとわかりやすいかも
		- どこでエラーメッセージを管理するか

具体的なエラークラスの実装やハンドリングの仕組みは改めてDesign Docに記載する

## 決定内容

[何が決まったのか]

## 決定理由

[なぜその決定に至ったのか]

## 反対意見・懸念事項

[決定に対する懸念や反対意見があれば記載]

## 次のアクション

- [ ]  [具体的なアクション1]
- [ ]  [具体的なアクション2]
- [ ]  ADR作成の必要性
- [ ]  Design Doc作成の必要性
- [ ]  Linearタスク作成: [タスクへのリンク]

 
