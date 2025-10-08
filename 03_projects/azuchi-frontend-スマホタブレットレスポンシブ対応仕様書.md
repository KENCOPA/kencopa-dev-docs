---
type: note
tags:
- '#ai/ref'
---
# 🎯 目的・対象範囲

## 目的
azuchi-frontendアプリケーションにおけるスマートフォン・タブレット端末でのユーザーエクスペリエンス向上を目的とし、レスポンシブデザインの実装により、モバイル端末での操作性とレイアウトの最適化を実現する。

## 対象範囲
- **対象システム**: azuchi-frontend（Next.js + Tailwind CSS）
- **対象デバイス**: スマートフォン（375px〜）、タブレット（768px〜）、デスクトップ（1024px〜）
- **対象期間**: 4段階の段階的実装
- **主要対象ファイル**:
  - `src/page-components/members/layouts/member-layout.tsx`
  - `src/page-components/guests/layouts/guest-layout.tsx`
  - `src/components/layouts/side-navigation/`配下のコンポーネント
  - 固定幅を使用する主要コンポーネント（55+ファイル）

# 🏗 システム構成概要

## 現在の問題点
1. **固定レイアウトの多用**: `fixed top-0 h-screen overflow-y-auto`による画面固定
2. **サイドナビゲーションの非対応**: モバイルでのドロワー表示未実装
3. **固定幅の多用**: `w-64`, `min-w-[1000px]`等による横スクロール発生
4. **100vh問題**: iOS Safariでのアドレスバー表示時のレイアウト崩れ

## 目標アーキテクチャ
```text
デスクトップ (md: 768px+)
├─ サイドナビ: 固定表示 (w-64)
└─ メインコンテンツ: ml-64でオフセット

モバイル (< 768px)  
├─ サイドナビ: オーバーレイドロワー
└─ メインコンテンツ: 全幅表示
```

# 🛠 基本設計

## 実装計画（4段階）

### ステップ1: サイドナビゲーションのレスポンシブ化

**対象ファイル**:
- `src/page-components/members/layouts/member-layout.tsx`
- `src/page-components/guests/layouts/guest-layout.tsx`
- `src/components/layouts/side-navigation/index.tsx`
- `src/components/layouts/side-navigation/project-side-navigation/index.tsx`
- `src/components/layouts/side-navigation/tenant-side-navigation/index.tsx`

**実装内容**:
```tsx
// Before: 固定レイアウト
<div className="fixed top-0 h-screen overflow-y-auto w-[calc(100%-16rem)] left-64">

// After: レスポンシブレイアウト
<div className="relative w-full md:ml-64 min-h-[100dvh]">
```

**サイドナビゲーション（shadcn/ui Dialog/Sheet推奨）**:
```tsx
// デスクトップ: 固定表示（CSS優先、FOUC防止）
<div className="hidden md:fixed md:inset-y-0 md:left-0 md:w-64 md:h-[100dvh] md:block">
  {/* ナビゲーションコンテンツ */}
</div>

// モバイル: オーバーレイドロワー（shadcn/ui Dialog/Sheet使用）
<Dialog open={isExpanded} onOpenChange={setIsExpanded}>
  <DialogContent 
    className="fixed inset-0 z-50 md:hidden p-0"
    aria-describedby="mobile-navigation-description"
  >
    {/* 背景オーバーレイ（自動処理） */}
    
    {/* ドロワーパネル */}
    <div className={cn(
      "fixed left-0 top-0 h-[100dvh] w-[80vw] max-w-[320px] bg-background",
      "transform transition-transform duration-200",
      "pl-[env(safe-area-inset-left)]", // iOS対応
      "overflow-y-auto" // 内部スクロール
    )}>
      {/* フォーカストラップ自動対応のナビゲーションコンテンツ */}
    </div>
  </DialogContent>
</Dialog>

// 代替実装（shadcn/ui未使用の場合）
<div 
  className={cn("fixed inset-0 z-50 md:hidden", isExpanded ? "block" : "hidden")}
  role="dialog"
  aria-modal="true"
  aria-hidden={!isExpanded}
>
  <div className="fixed inset-0 bg-black/50" onClick={onClose} aria-hidden="true" />
  <div className={cn(
    "fixed left-0 top-0 h-[100dvh] w-[80vw] max-w-[320px] bg-background",
    "transform transition-transform duration-200",
    "pl-[env(safe-area-inset-left)]",
    isExpanded ? "translate-x-0" : "-translate-x-full"
  )}>
    {/* 手動フォーカストラップ実装が必要 */}
  </div>
</div>
```

**ハンバーガーボタン仕様（必須）**:
```tsx
// 全ページ共通ヘッダーに配置
<button
  className="md:hidden p-2 min-h-[44px] min-w-[44px]" // タップ領域44px以上
  onClick={toggleNav}
  aria-controls="mobile-navigation"
  aria-expanded={isNavOpen}
  aria-label="メニューを開く"
>
  <HamburgerIcon />
</button>
```

**アクセシビリティ要件（必須）**:
- **shadcn/ui Dialog/Sheet使用を強く推奨**: Radixベースで以下が自動対応
  - `role="dialog" aria-modal="true"` の自動設定
  - `aria-hidden` の適切な制御
  - フォーカストラップの自動実装
  - ESCキーでの閉じる機能
  - 背景スクロールロック（`react-remove-scroll`）
- **ハンバーガーボタン要件**:
  - `aria-controls`/`aria-expanded` 属性（必須）
  - キーボードフォーカス対応
  - タップ領域44px以上確保
- **手動実装の場合**: 上記すべてを手動で実装する必要あり

**iOS対策詳細**:
- `env(safe-area-inset-left)` - ドロワー内側パディング
- `env(safe-area-inset-bottom)` - 固定フッター対応
- `overscroll-behavior: contain` - スクロール境界制御
- 入力フォーカス時のジャンプ対策（フォーム多用画面は`min-h-screen`フォールバック）

### ステップ2: 固定レイアウトの撤去とdvh対応

**変更対象**:
- メインコンテナ: `fixed top-0 h-screen` → `relative min-h-[100dvh]`
- スクロール管理: 単一箇所でのスクロール制御
- iOS対応: `100dvh`の使用（フォールバック: `min-h-screen`）

**実装例（member-layout.tsx/guest-layout.tsx）**:
```tsx
// メインレイアウト（relative + md:ml-64）
<div className="relative w-full md:ml-64 min-h-[100dvh] overflow-x-hidden">
  {/* overflow-y-auto はメイン一箇所に集約 */}
  <div className="overflow-y-auto min-h-0">
    {children}
  </div>
</div>

// 内部スクロールが必要な場合（子要素は min-h-0）
<div className="min-h-0 max-h-[calc(100dvh-200px)] overflow-y-auto">
  {scrollableContent}
</div>

// サイドナビ本体の具体的クラス指定
<div className="hidden md:fixed md:inset-y-0 md:left-0 md:w-64 md:h-[100dvh] md:block">
  {/* ナビゲーションコンテンツ */}
</div>

// ドロワーの具体的クラス指定（shadcn/ui使用時）
<Dialog>
  <DialogContent className="fixed inset-0 z-50 md:hidden p-0">
    <div className="fixed left-0 top-0 h-[100dvh] w-[80vw] max-w-[320px] transform transition-transform duration-200">
      {/* ドロワーコンテンツ */}
    </div>
  </DialogContent>
</Dialog>
```

### ステップ3: 固定幅の緩和

**優先対象ファイル**:
1. `src/shared/components/dynamic-panel.tsx`: `min-w-[800px]`
2. `src/components/company-safety-management/site-safety-patrol-viewer/presentaion.tsx`: `min-w-[1000px]`
3. `src/components/projects/project-select/index.tsx`: `w-[260px]`

**変更パターン**:
```tsx
// Before: 固定幅
<div className="min-w-[1000px]">

// After: レスポンシブ幅
<div className="w-full md:min-w-[1000px]">

// ドロップダウン・ダイアログ
<div className="w-full sm:w-[260px] max-w-[95vw]">

// テーブル対応（優先度の低い列を隠す）
<th className="hidden sm:table-cell">詳細情報</th>
<td className="hidden sm:table-cell">{detail}</td>

// 横スクロールは親コンテナのみで許容
<div className="overflow-x-auto">
  <table className="min-w-full">
    {/* テーブルコンテンツ */}
  </table>
</div>
```

**コンポーネント選択指針**:
- **モバイル**: Dialog/Sheet優先（shadcn/ui Radixベース推奨）
- **デスクトップ（md以上）**: DynamicPanelは補助的使用のみ
- **理由**: DynamicPanelは`min-w-[800px]`/`h-screen`を含むためモバイル親和性が低い

**テーブルの最小要件**:
- 横スクロールは親のみで許容（既存 `Table` でOK）
- モバイルでは優先度の低い列に `hidden sm:table-cell` を適用
- 意図的な横スクロールコンテナ以外では横スクロール発生を防ぐ

**段階的対応計画**:
1. **優先対象3箇所の修正**（上記対象ファイル）
2. **中間対応**: `min-w-[200px]~[300px]`クラスの検索置換で`sm:`以上に限定
3. **最終対応**: 「トップ10修正後、残りは検索置換で追随」方針で全体適用

### ステップ4: 100vh問題の段階的解決

**対象**: `h-screen`/`min-h-screen`を使用する37+ファイル

**置換パターン（上位レイアウトから順に実施・一気に全置換しない）**:
```css
/* 1. 上位レイアウト（最優先） */
src/page-components/members/layouts/member-layout.tsx
src/page-components/guests/layouts/guest-layout.tsx
変更: fixed top-0 h-screen → relative min-h-[100dvh]
メイン: relative + md:ml-64, overflow-y-auto はメイン一箇所に集約

/* 2. 個別ページ（親で高さ管理の原則） */
h-screen → min-h-0 (親コンテナで高さ管理)
min-h-screen → min-h-0
子要素: min-h-0 で親の高さに依存

/* 3. iOS Safari対策（フォーム多用画面のフォールバック） */
iOSでフォームが多い画面は dvh でも不安定な場合があるため:
当該画面だけ min-h-[100dvh] → min-h-screen にフォールバック

/* 4. iOS対応の詳細設定 */
env(safe-area-inset-left) - ドロワー内側パディング
env(safe-area-inset-bottom) - 固定フッター対応
overscroll-behavior: contain - スクロール境界制御
```

**実装順序（段階的アプローチ・まずは上位レイアウトから）**:
1. **Phase 1**: メインレイアウト2ファイルから着手（member-layout.tsx, guest-layout.tsx）
2. **Phase 2**: ページ側は `min-h-0` と親コンテナで高さ管理の原則を適用
3. **Phase 3**: 実機iOS Safariでの検証・必要に応じてフォールバック適用
4. **Phase 4**: 残りファイルの順次対応

## ブレークポイント設計

| デバイス | 幅 | Tailwind | 動作 |
|---------|-----|----------|------|
| スマートフォン | 375px〜767px | デフォルト | ドロワーナビ、全幅コンテンツ |
| タブレット | 768px〜1023px | md: | 固定ナビ、オフセットコンテンツ |
| デスクトップ | 1024px〜 | lg: | 固定ナビ、オフセットコンテンツ |

## 状態管理の拡張

**既存**: `useSideNavigationState`フック
**基本方針**: CSS優先、JavaScript補助

**レスポンシブ表示制御（SSR安全）**:
```tsx
// CSS優先アプローチ（SSR安全、FOUC防止）
// サイドナビ本体: hidden md:fixed md:inset-y-0 md:left-0 md:w-64 md:h-[100dvh] md:block
// ドロワー: fixed inset-0 z-50 md:hidden

// JavaScript（ドロワー開閉のみ制御）
const useResponsiveNavigation = () => {
  const [isMobile, setIsMobile] = useState(false); // SSR安全な初期値
  const { isSideNavigationOpen, toggleSideNavigation } = useSideNavigationState();
  
  useEffect(() => {
    // matchMediaを使用してSSR安全に実装
    const mediaQuery = window.matchMedia('(min-width: 768px)');
    const handleChange = (e: MediaQueryListEvent) => {
      setIsMobile(!e.matches);
      // md以上では自動でopen状態に（CSSで制御済み）
    };
    
    setIsMobile(!mediaQuery.matches);
    mediaQuery.addEventListener('change', handleChange);
    return () => mediaQuery.removeEventListener('change', handleChange);
  }, []);
  
  return {
    isMobile,
    // モバイルのみドロワー開閉を制御、デスクトップはCSS固定
    isNavOpen: isMobile ? isSideNavigationOpen : true,
    toggleNav: isMobile ? toggleSideNavigation : () => {},
  };
};
```

**FOUC/ハイドレーション対策**:
- 初期は `nav: md:block hidden` としてCSSでの露出を担保
- JSの状態（`isSideNavigationOpen`）はmd未満のドロワー時だけ反映
- タブレット/デスクトップでの「初期閉」チラつきを回避

# 🔧 非機能要件対応

## 性能要件
- **レイアウトシフト**: Lighthouse CLS スコア 0.1 以下
- **レスポンス性**: ナビゲーション開閉アニメーション 200ms 以下
- **メモリ使用量**: リサイズイベントリスナーの適切な管理

## アクセシビリティ
- **フォーカストラップ**: モバイルドロワー内でのキーボードナビゲーション（shadcn/ui Dialog/Sheet推奨）
- **スクリーンリーダー**: `aria-expanded`, `aria-hidden`, `role="dialog"`, `aria-modal="true"`の適切な設定
- **キーボード操作**: ESCキーでのドロワー閉じる機能
- **ハンバーガーボタン**: `aria-controls`/`aria-expanded`属性、タップ領域44px以上
- **背景スクロールロック**: ドロワー開時の`document.body.style.overflow='hidden'`または`react-remove-scroll`

## ブラウザ対応
- **iOS Safari**: `100dvh`対応、キーボード表示時の対応
- **Android Chrome**: ビューポート単位の適切な処理
- **デスクトップ**: 既存機能の完全な互換性維持

## セキュリティ
- **XSS対策**: 動的クラス名生成時のサニタイゼーション
- **CSP対応**: インラインスタイルの回避

# 🚀 リリース・移行手順

## 段階的リリース計画

### Phase 1: サイドナビゲーション対応
1. **開発**: ステップ1の実装
2. **テスト**: 375×812, 768×1024での動作確認
3. **リリース**: フィーチャーフラグでの段階的展開

### Phase 2: レイアウト基盤改修
1. **開発**: ステップ2の実装
2. **テスト**: スクロール動作、iOS Safari確認
3. **リリース**: 主要ページから段階的適用

### Phase 3: 固定幅対応
1. **開発**: ステップ3の実装（優先度順）
2. **テスト**: 横スクロール発生の確認
3. **リリース**: コンポーネント単位での適用

### Phase 4: 100vh問題解決
1. **開発**: ステップ4の実装
2. **テスト**: 実機iOS Safariでの確認
3. **リリース**: 問題箇所の段階的修正

## テスト手順

### 受け入れテスト
**テスト環境**:
- iPhone SE (375×812)
- iPad (768×1024) 
- Desktop (1920×1080)

**テストケース（強化された受け入れ基準）**:
1. **ナビゲーション**:
   - モバイル: ハンバーガーメニューでドロワー開閉
   - タブレット以上: 固定ナビゲーション表示（初期表示からレイアウトシフトなし）
   - 背景クリック・ESCキーでの閉じる動作
   - フォーカストラップとアクセシビリティ属性の動作

2. **レイアウト（強化された受け入れ基準）**:
   - メインコンテンツがナビゲーションに隠れない
   - **横スクロール基準**: 375px幅で`document.documentElement.scrollWidth <= window.innerWidth + 1`を満たす（テーブル/グラフ等の意図的な横スクロールコンテナを除く）
   - **スクロールレイヤの一意性**: ページにユーザーが操作する縦スクロールレイヤは基本1つ（ドロワー開時はドロワー内のみ）
   - **初期表示の一貫性**: md以上でサイドナビが初期から表示（ハイドレーションでのレイアウトシフトなし）

3. **入力フォーカス**:
   - iOS Safariでキーボード表示時のレイアウト（dvh問題の確認）
   - フォーカストラップの動作
   - スクロール位置の保持

4. **FOUC/ハイドレーション対策**:
   - md以上でサイドナビが初期から表示（チラつきなし）
   - CSS優先の表示制御が正常動作

5. **実務上のリスク確認**:
   - **入力フォーカス時のジャンプ**: iOSでフォームが多い画面は `dvh` でも不安定な場合があるため、当該画面だけ `min-h-screen` にフォールバック運用
   - **スクロールロック漏れ**: ドロワー開時は必ず背景スクロールロック（Radix/Sheet or `react-remove-scroll` 推奨）
   - **主要ページでの操作性**: ホーム、プロジェクト一覧、各管理/工程/安全、チャットが操作可能

### 自動テスト
```bash
# Lint & Type Check
cd ~/repos/azuchi-frontend
make lint-fix
yarn type-check

# Visual Regression Test (Playwright) - TODO: スクリプト作成が必要
# yarn test:visual --project=mobile
# yarn test:visual --project=tablet  
# yarn test:visual --project=desktop
```

**テストスクリプトの存在確認**:
- ドキュメント内の `yarn test:visual` は事前にスクリプト実装が必要
- Playwrightセットアップが未整備なら「TODO: スクリプト作成」を追加
- 実装時にはPlaywright設定とテストスクリプトの作成から開始

## ロールバック計画
1. **フィーチャーフラグ**: 即座の無効化（環境変数+`@/shared/providers`でのフラグ配布）
2. **CSS変更**: 旧クラス名の一時復活
3. **コンポーネント**: 旧バージョンへの切り戻し
4. **データベース**: レイアウト設定の復元

**フィーチャーフラグ運用**:
- **実装手段**: 環境変数+`@/shared/providers`でのフラグ配布、もしくはアプリ設定
- **配布方法**: `@/shared/providers`でのアプリケーション全体への配布
- **段階的ロールアウト**: 段階的ロールアウト対応

# 📚 参考・付録

## 技術調査結果

### 現状分析
- **固定幅使用ファイル**: 55+ファイル確認
- **h-screen使用ファイル**: 37+ファイル確認  
- **既存レスポンシブ**: 28ファイルで`md:`ブレークポイント使用
- **サイドナビ状態管理**: `useSideNavigationState`フック実装済み

### 主要変更箇所
```text
src/
├─ page-components/
│  ├─ members/layouts/member-layout.tsx ★
│  └─ guests/layouts/guest-layout.tsx ★
├─ components/layouts/side-navigation/
│  ├─ index.tsx ★
│  ├─ project-side-navigation/index.tsx ★
│  └─ tenant-side-navigation/index.tsx ★
├─ shared/
│  ├─ components/dynamic-panel.tsx ★
│  └─ hooks/use-side-navigation-state.ts
└─ components/
   ├─ company-safety-management/site-safety-patrol-viewer/ ★
   └─ projects/project-select/ ★
```

## 実装ガイドライン

### コーディング規則
- **インポート**: `@/...`形式を使用
- **クラス命名**: 既存のTailwind CSS規則に従う
- **コンポーネント**: Dialog/Sheet優先、DynamicPanelはmd以上で補助的使用
- **差分最小化**: 必要最小限の変更に留める
- **ファイル名**: `presentaion.tsx` → `presentation.tsx` へのリネーム計画（別PR）

**表現/記述の修正**:
- 「コンポーネント: DynamicPanelの使用（Dialogの代替）」は「Dialog/Sheet優先、DynamicPanelはmd以上で補助」に変更済み
- `presentaion.tsx` は既存ファイル名に合わせているが、将来は `presentation.tsx` へリネーム計画を追記（別PRで）

### 品質保証
- **Lighthouse**: モバイルスコア90+維持
- **アクセシビリティ**: WCAG 2.1 AA準拠
- **パフォーマンス**: Core Web Vitals基準クリア

## 関連ドキュメント
- [[🗒️ azuchi-frontend-アーキテクチャ概要]]
- [[🗒️ Tailwind CSS設計ガイドライン]]
- [[🗒️ モバイルUXベストプラクティス]]

---

**作成日**: 2025年9月22日  
**作成者**: Devin AI  
**レビュー**: 要レビュー  
**承認**: 未承認
