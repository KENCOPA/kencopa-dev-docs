---
type: note
---
# azuchi-frontendヘルプページリンク更新仕様

## 概要

azuchi.kencopa.comに表示されているヘルプページのリンク先を、古いNotionページ（naka-time）から新しいNotionページ（kencopa-inc）に更新する必要がある。

## 背景

現在azuchi-frontendのヘルプ機能で参照されているNotionページが古いドメイン（naka-time.notion.site）のものになっており、新しい公式ドメイン（kencopa-inc.notion.site）に変更する必要がある。

## 要求者・担当者

- **要求者**: 中田 敏寛 / Toshinori Nakata (U07UP1HJUJ0)
- **担当者**: 久毛匠 (U08HFNAMUT1)
- **Slackスレッド**: [#pd-azuchi-dev](https://kencopahq.slack.com/archives/C07UQ6TU3N0/p1756445123194499?thread_ts=1756445123.194499)

## 現在の実装状況

### ヘルプリンクの表示場所
- **場所**: azuchi.kencopa.com の画面左下
- **UI要素**: ヘルプボタン（InfoIcon使用）
- **コンポーネント**: `src/components/layouts/side-navigation/shared/side-navigation-help-section/index.tsx`

### 現在の設定
```typescript
// src/components/layouts/side-navigation/shared/side-navigation-help-section/index.tsx
<SideNavigationExternalItem
  icon={<InfoIcon />}
  label="ヘルプ"
  href={process.env.NEXT_PUBLIC_EXTERNAL_HELP_URL}
/>
```

### 環境変数設定
```bash
# .env.local および .env.docker.local
NEXT_PUBLIC_EXTERNAL_HELP_URL="https://naka-time.notion.site/Kencopa-1de701ffa9c480919299dbbc8a2afffe"
```

## 変更仕様

### 変更対象URL

| 項目 | URL |
|------|-----|
| **変更前（古い）** | `https://naka-time.notion.site/Kencopa-1de701ffa9c480919299dbbc8a2afffe` |
| **変更後（新しい）** | `https://kencopa-inc.notion.site/Kencopa-21234568ef5081e4ae47f78a21ab52ee` |

### 変更対象ファイル

1. **環境変数ファイル**
   - `.env.local`
   - `.env.docker.local`
   - `.env.docker.template`（存在する場合）

2. **インフラ設定**
   - azuchi-infra-v2リポジトリのTerraform設定
   - Amplify環境変数設定

### 実装手順

1. **ローカル環境変数の更新**
   ```bash
   # .env.local および .env.docker.local の更新
   NEXT_PUBLIC_EXTERNAL_HELP_URL="https://kencopa-inc.notion.site/Kencopa-21234568ef5081e4ae47f78a21ab52ee"
   ```

2. **インフラ設定の更新**
   - azuchi-infra-v2リポジトリのTerraform設定で環境変数を更新
   - 各環境（dev, stg, prod）での設定確認

3. **動作確認**
   - ローカル環境でのヘルプリンク動作確認
   - 新しいNotionページへの正常なリダイレクト確認
   - 各環境でのデプロイ後動作確認

## 注意事項

- **リンク先の確認**: 新しいNotionページが正常にアクセス可能であることを事前に確認
- **権限設定**: 新しいNotionページの公開設定・アクセス権限が適切に設定されていることを確認
- **環境別設定**: dev, stg, prod各環境で同じURLを使用するか確認
- **キャッシュクリア**: ブラウザキャッシュやCDNキャッシュの影響を考慮

## 検証項目

- [ ] ヘルプボタンクリック時に新しいNotionページが開くこと
- [ ] 新しいNotionページが正常に表示されること
- [ ] 各環境（dev, stg, prod）で動作すること
- [ ] モバイル・デスクトップ両方で動作すること

## 関連リソース

- **Slackスレッド**: https://kencopahq.slack.com/archives/C07UQ6TU3N0/p1756445123194499?thread_ts=1756445123.194499
- **新しいNotionページ**: https://kencopa-inc.notion.site/Kencopa-21234568ef5081e4ae47f78a21ab52ee
- **古いNotionページ**: https://naka-time.notion.site/Kencopa-1de701ffa9c480919299dbbc8a2afffe
- **azuchi-frontendリポジトリ**: https://github.com/KENCOPA/azuchi-frontend
- **azuchi-infra-v2リポジトリ**: https://github.com/KENCOPA/azuchi-infra-v2

## 実装優先度

**中** - ユーザビリティに影響するが、システムの基本機能には影響しない

## 完了条件

- [ ] 全環境でヘルプリンクが新しいNotionページを参照している
- [ ] 新しいNotionページが正常にアクセス可能
- [ ] 動作確認が完了している
- [ ] インフラ設定が更新されている
