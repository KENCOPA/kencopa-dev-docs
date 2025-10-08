---
type: note
tags:
- '#ai/ref'
---
## 概要

kunai-backendは、Domain-Driven Design（DDD）を核とした4層アーキテクチャを採用したマルチテナント対応TypeScript APIサーバーである。seed期スタートアップの特性を活かし、開発速度と品質保証のバランスを最適化した設計を実現している。

## 設計の基本原則

### 1. DDDによる複雑性の管理
- **ビジネスロジックの中心化**: ドメイン層にビジネスルールを集約
- **境界の明確化**: 集約境界による責任分離
- **型安全性**: ブランド型による厳密な型区別

### 2. 開発効率の最適化
- **共通機能の体系化**: `shared`ディレクトリによる機能横断的な基盤の整備
- **コードの再利用性**: DDD基盤クラスによる統一的な実装パターン
- **開発者体験**: 一貫性のある設計によるコード理解の容易性

### 3. 品質保証戦略
- **テスト戦略の最適化**: 重要度に応じたテストレベルの使い分け
- **型システムの活用**: コンパイル時エラー検出による品質向上
- **不変性の徹底**: 副作用の最小化

## アーキテクチャ概要

```
src/
├── domains/           # Domain Layer - ビジネスロジックの中核
├── infrastructures/   # Infrastructure Layer - 外部依存の実装
├── usecases/         # Application Layer - ビジネスロジックの調整
├── routes/           # Presentation Layer - APIエンドポイント
├── shared/           # 横断的関心事の共通基盤
└── authorizers/      # 認可処理の専用モジュール
```

## 各層の役割と設計思想

### Domain Layer (`src/domains/`)

**Why**: ビジネスロジックを純粋に保つため、外部依存から完全に分離

**設計原則**:
- 集約単位での責任境界の明確化
- 値オブジェクトによる不変性の保証
- ドメインサービスによるビジネスルール実装

**構造**:
```
domains/
├── user/                    # User集約
│   ├── user.ts             # 集約ルート
│   ├── valueObjects/       # 値オブジェクト
│   ├── repositories/       # リポジトリインターフェース
│   └── authorizers/        # 認可インターフェース
├── tenant/                 # Tenant集約
├── membership/             # Membership集約
├── queries/                # CQRSクエリ定義
│   ├── {domain}{Operation}QueryService.ts  # QueryServiceインターフェース・DTO・Authorizer
│   └── ...
└── shared/                 # ドメイン共通基盤
    ├── dddObjectBases/     # DDD基盤クラス
    ├── valueObjects/       # 共通値オブジェクト
    ├── authorizers/        # 認可システム基盤
    │   ├── queryServiceAuthorizer.ts        # QueryService専用認可インターフェース
    │   └── queryServiceAuthorizerResult.ts  # QueryService認可結果型
    └── exceptions/         # ドメイン例外
```

### Infrastructure Layer (`src/infrastructures/`)

**Why**: 外部システムとの結合を最小化し、テスタビリティを向上

**設計原則**:
- リポジトリパターンによる抽象化
- 外部クライアントの統一的な管理
- 設定の外部化

**構造**:
```
infrastructures/
├── repositories/           # リポジトリ実装
├── queries/               # QueryService実装（CQRSクエリ側）
│   └── {domain}{Operation}QueryService.ts  # 具象実装
└── shared/
    └── clients/           # 外部クライアント
        ├── databaseClient/
        ├── storageClient/
        └── parameterClient/
```

### Application Layer (`src/usecases/`)

**Why**: ビジネスロジックの調整とトランザクション管理を担当

**設計原則**:
- ユースケース単位での処理の組み立て
- デコレーターによる横断的関心事の処理
- Command/Response パターンによる型安全な入出力

### Presentation Layer (`src/routes/`)

**Why**: HTTP プロトコルとアプリケーションロジックの分離

**設計原則**:
- エンドポイント単位での責任分離
- バリデーションの統一
- 認証・認可の標準化

**構造**:
```
routes/
├── v1/
│   ├── public/            # 非認証エンドポイント
│   └── protected/         # 認証必要エンドポイント
│       ├── userContext/   # ユーザー認証
│       └── tenantContext/ # テナント認証
```

## 共通基盤の設計思想 (shared ディレクトリ)

### なぜ shared ディレクトリを作ったか

1. **層横断的な関心事の整理**: 各層内で共通して使用される機能を体系化
2. **開発効率の向上**: 再利用可能なコンポーネントの一元管理
3. **一貫性の担保**: 統一的な実装パターンの提供

### 配置戦略

**階層別の shared 配置**:
- `src/shared/`: アプリケーション全体の横断的関心事
- `src/domains/shared/`: ドメイン層内の共通基盤
- `src/infrastructures/shared/`: インフラ層内の共通基盤

**設計原則**:
- 各層の責任境界を維持しながら、層内での重複を排除
- より具体的な層の shared が、より抽象的な shared に依存
- 共通化により保守性と開発効率を向上

## CQRSパターンの設計思想

### なぜ Domain層に queries/ を配置したか

1. **ドメイン知識の明確化**: QueryServiceもドメイン知識として定義
2. **契約の分離**: Infrastructure層の実装とDomain層のinterface定義を分離
3. **型安全性の向上**: DTO・Authorizer・Interfaceの一体管理

### QueryService認可システム

**専用認可インターフェース**:
- `IQueryServiceUserContextAuthorizer<T>`: ユーザーコンテキスト用
- `IQueryServiceTenantContextAuthorizer<T>`: テナントコンテキスト用
- 配列ベースの認可処理に特化

**認可結果型**:
- `AllowedQuery<T>` / `DeniedQuery<T>`: QueryService専用の認可結果
- 既存のEntity認可システムとの型レベル分離

## テスト戦略の設計思想 (`tests/`)

see [[💭 BEのユニットテスト方針について]]
### 基本方針

**seed期スタートアップ特化の戦略**:
- 重要度に応じたテストレベルの使い分け
- 開発速度と品質のバランス最適化
- リソース効率の最大化

### テスト構造

```
tests/
├── unit/                  # ユニットテスト
│   └── domains/          # Domain層のみ対象
└── ita/                  # 統合テスト
    ├── scenarios/        # エンドポイントテスト
    └── helpers/          # テストヘルパー
```

### テストレベル戦略

#### ユニットテスト対象
- **Domain層**: ビジネスロジック・値オブジェクト
- **Authorization層**: セキュリティ重要機能
- **共通基盤**: DDD基盤クラス・ユーティリティ

#### 統合テスト対象
- **エンドポイント**: 実際のユーザーシナリオ
- **プロセス外依存**: Docker・LocalStack活用
- **システム全体**: 各層の相互作用

### CI/CD戦略

**段階的品質保証**:
- **PR時**: ユニットテストのみ（5分以内）
- **マージ時**: 統合テスト（30分以内、Slack通知）