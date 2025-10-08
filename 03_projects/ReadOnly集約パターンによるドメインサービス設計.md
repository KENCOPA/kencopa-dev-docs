---
type: note
moc: "[[🏷️基本設計書]]"
tags:
- '#ai/ref'
---
## 概要 (Summary)

このPRでは、DDDにおける「集約は他の集約を参照してはならない」という制約を、TypeScriptの型システムを活用して解決するReadOnly集約パターンを実装しています。これにより、密結合な集約間の参照を型安全に実現し、CQRSパターンの過度な使用を避けることができます。

## 背景・なぜこの変更が必要なのか (Why)

### 1. DDDの制約と実際の開発ニーズのギャップ

- **DDD原則**: 集約は他の集約を直接参照してはならない
- **実際の開発**: SiteMemberとUser、SiteMemberとSiteのように密結合な関係では、毎回IDによる参照やCQRSパターンの実装が必要
- **問題**: 明らかに密な集約関係（例：SiteMemberの名前をUser経由で取得）でも、毎回複雑なパターンを実装する必要があった
    

### 2. 既存のazuchiプロジェクトでの課題

- 過度な共通化により、collectionクラスで必要な関数や引数・内部ロジックが集約ごとに大きく異なる問題が発生
- abstractクラスでの共通実装が、実際の使用場面では適合しないケースが多発
    

### 3. 開発効率とコード品質の両立

- CQRSパターンを毎回実装するのは開発効率が悪い
- しかし、DDD原則を破ると設計の一貫性が失われる
- 型安全性を保ちながら、実用的な解決策が必要
    

## 代替案 (Alternatives)

### 1. 従来のCQRSパターンのみを使用

**メリット:**

- DDD原則に完全に準拠
- 集約間の依存関係が明確

**デメリット:**

- 密結合な関係でも毎回複雑な実装が必要
- 開発効率が悪い
- コード量が増加

### 2. IDによる参照のみを使用

**メリット:**

- シンプルな実装
- DDD原則に準拠

**デメリット:**

- 関連データの取得に追加のクエリが必要
- パフォーマンスの問題
- 使用時の利便性が低い
    

### 3. 集約間の直接参照を許可

**メリット:**

- 実装が簡単
- 高いパフォーマンス

**デメリット:**

- DDD原則に違反
- 循環参照のリスク
- 設計の一貫性が失われる

### 4. 採用したReadOnly集約パターン

**メリット:**

- DDD原則を型レベルで強制
- 密結合な関係で高い開発効率
- TypeScriptの型安全性を活用
- 実行時の不変性も保証

**デメリット:**

- 新しいパターンのため学習コストがある
- 型定義が複雑になる

## 実装のエッセンス

### 1. ReadOnly集約の型定義

```
export type ReadOnlyAggregation<T extends IAggregation<IUUID<unknown>, symbol>> =   Readonly<T> & ReadOnlyDDDObject;
```

### 2. 集約間参照の制御

- `AggregationConstructorArgs<T>`型により、コンストラクタでReadOnly集約のみ受け入れ可能
- `asReadOnly()`メソッドで集約をReadOnlyとしてマーク
- `Object.freeze()`により実行時の不変性を保証

### 3. Repository層での安全性

- `omitReadOnlyAggregations()`メソッドでReadOnly集約を除外
- Repository層では純粋なデータのみを扱う

## モジュールの使用方法

### 1. 基本的な使用パターン

```typescript
// 集約の作成  
const user = User.create({  
  familyName: "test",  
  givenName: "test",   
  emailAddress: new EmailAddress("test@test.com"),  
});  
  
const site = Site.create({  
  name: "test",  
});  
  
// ReadOnly集約として参照  
const siteMember = SiteMember.create({  
  site: site.asReadOnly(),  
  user: user.asReadOnly(),  
});
```

### 2. Repository層での使用

```typescript
// ReadOnly集約を除外してRepository層に渡す  
const omitReadOnlySite = site.omitReadOnlyAggregations();  
const omitReadOnlySiteMember = siteMember.omitReadOnlyAggregations();  
  
// 通常の属性にはアクセス可能  
console.log(omitReadOnlySite.name);  
console.log(omitReadOnlySiteMember.id);  
  
// ReadOnly集約にはアクセス不可（コンパイルエラー）  
// console.log(omitReadOnlySiteMember.user.familyName);
```

### 3. 適用指針

- **密結合な関係**: ReadOnly集約パターンを使用（例：SiteMember ↔ User/Site）
- **疎結合な関係**: 従来通りCQRSパターンを使用（例：Tenant ↔ Site）

## 技術的な特徴

### 1. 型安全性

- TypeScriptの型システムでDDD制約を強制
- コンパイル時に不正な参照を検出

### 2. 実行時安全性

- `Object.freeze()`による不変性保証
- `__readonly`マーカーによる識別

### 3. 開発効率

- 密結合な関係での簡潔な実装
- CQRSパターンの選択的使用


