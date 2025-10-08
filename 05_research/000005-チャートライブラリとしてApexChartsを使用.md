---
type: note
moc: "[[🏷️ADR]]"
---
# Recharts vs ApexCharts 詳細比較

## 1. 概要と目的

本ドキュメントは工程管理アプリケーションにおける進捗率グラフ表示に最適なライブラリの選定を目的とする。
既存のガントチャートは自作モジュールを使用しているため、その他のデータ可視化（進捗率チャートなど）に焦点を当て、
特にRechartsとApexChartsに絞り込んで詳細な比較検討を行う。

## 2. プロジェクトコンテキスト

### 2.1 技術スタック
- **フロントエンド**: Next.js (TypeScript)
- **UIコンポーネント**: shadcn
- **バックエンド**: Hono (TypeScript)
- **データベース**: PostgreSQL (Prisma)
- **インフラ**: AWS (Lambda, APIGateway, Cognito, Amplify)

### 2.2 ユーザー特性
- 工事現場のスタッフ（ITリテラシーは必ずしも高くない）
- 主にタブレットデバイスでの使用
- データの視覚的理解を重視

### 2.3 主要要件
1. 出来高進捗率（計画・実績）の視覚的表示
2. データの詳細分析のためのズーム・インタラクション機能
3. 表示期間の切り替え（週、月、六ヶ月）
4. チャートの画像出力機能
5. モダンで使いやすいUI/UX

## 3. 候補ライブラリ詳細比較

### 3.1 Recharts

#### 3.1.1 基本情報
- **GitHubスター数**: 約20,000（2024年4月時点）
- **サイズ**: 約90KB (gzipped)
- **最終更新**: 活発にメンテナンスされている
- **ライセンス**: MIT

#### 3.1.2 技術的特徴
- **レンダリング方式**: SVGベース
- **アーキテクチャ**: React専用に設計され、React Componentとして実装
- **依存関係**: d3-scaleやd3-shapeなどの最小限のD3.jsモジュールに依存
- **型定義**: TypeScriptの型定義が標準サポート

#### 3.1.3 機能詳細評価

| 機能 | 評価 | 詳細 |
|------|------|------|
| **基本グラフ機能** | ⭐⭐⭐⭐⭐ | 折れ線、棒、エリア、散布図など主要なチャートタイプをすべてサポート |
| **レスポンシブ対応** | ⭐⭐⭐⭐ | `ResponsiveContainer`を使用してレスポンシブ対応、柔軟なサイズ調整が可能 |
| **パフォーマンス** | ⭐⭐⭐⭐ | SVGベースだがパフォーマンスは良好、大量データで若干低下する可能性あり |
| **カスタマイズ性** | ⭐⭐⭐⭐⭐ | 非常に高い柔軟性、スタイル、アニメーション、軸などすべての要素をカスタマイズ可能 |
| **インタラクション** | ⭐⭐⭐ | 基本的なツールチップ、クリックイベントをサポート、高度なズームは追加実装が必要 |
| **アニメーション** | ⭐⭐⭐⭐ | トランジションアニメーションをサポート、柔軟にカスタマイズ可能 |
| **画像エクスポート** | ⭐⭐ | 標準機能なし、html2canvasなど外部ライブラリを利用する必要がある |
| **ドキュメント** | ⭐⭐⭐ | 基本的なドキュメントは充実、高度な使用例はやや少ない |

#### 3.1.4 ユーザー体験評価
- **学習曲線**: 中程度（Reactに慣れていれば学習は容易）
- **直感性**: 高い（コンポーネントベースの設計が理解しやすい）
- **アクセシビリティ**: 基本的なアクセシビリティをサポート

#### 3.1.5 コード例（進捗率チャート）
```jsx
import { ResponsiveContainer, LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend } from 'recharts';

const ProgressChart = ({ data }) => (
  <ResponsiveContainer width="100%" height={400}>
    <LineChart data={data} margin={{ top: 5, right: 30, left: 20, bottom: 5 }}>
      <CartesianGrid strokeDasharray="3 3" />
      <XAxis dataKey="date" />
      <YAxis reversed domain={[100, 0]} />
      <Tooltip />
      <Legend />
      <Line type="monotone" dataKey="planned" stroke="#8884d8" />
      <Line type="monotone" dataKey="actual" stroke="#82ca9d" strokeDasharray="5 5" />
    </LineChart>
  </ResponsiveContainer>
);
```

### 3.2 ApexCharts

#### 3.2.1 基本情報
- **GitHubスター数**: 約13,000（2024年4月時点）
- **サイズ**: 約115KB (gzipped)
- **最終更新**: 活発にメンテナンスされている
- **ライセンス**: MIT

#### 3.2.2 技術的特徴
- **レンダリング方式**: SVGベース
- **アーキテクチャ**: JavaScriptライブラリベースでReactラッパー（react-apexcharts）を提供
- **依存関係**: 依存関係最小化、svg.jsに依存
- **型定義**: TypeScriptの型定義を提供

#### 3.2.3 機能詳細評価

| 機能 | 評価 | 詳細 |
|------|------|------|
| **基本グラフ機能** | ⭐⭐⭐⭐⭐ | 折れ線、棒、エリア、散布図など30以上のチャートタイプをサポート |
| **レスポンシブ対応** | ⭐⭐⭐⭐⭐ | 標準でレスポンシブ、ブレークポイントごとのオプション設定も可能 |
| **パフォーマンス** | ⭐⭐⭐⭐ | 大量データでも良好なパフォーマンス、最適化オプションあり |
| **カスタマイズ性** | ⭐⭐⭐⭐ | 豊富なオプション、テーマ設定が可能、高度なカスタマイズには複雑なAPI操作が必要な場合も |
| **インタラクション** | ⭐⭐⭐⭐⭐ | ズーム、パン、選択、ドラッグなど高度なインタラクション機能が標準搭載 |
| **アニメーション** | ⭐⭐⭐⭐⭐ | 多彩なアニメーションオプション、スムーズな視覚効果 |
| **画像エクスポート** | ⭐⭐⭐⭐⭐ | SVG、PNG、CSVなど複数形式でのエクスポート機能が標準搭載 |
| **ドキュメント** | ⭐⭐⭐⭐ | 詳細なドキュメント、多数のサンプルコード、デモが充実 |

#### 3.2.4 ユーザー体験評価
- **学習曲線**: 中程度（多数のオプションがあるため最初は複雑に感じる場合も）
- **直感性**: 高い（標準機能が充実しているため基本的な実装は容易）
- **アクセシビリティ**: 基本的なアクセシビリティ対応、カスタマイズにより拡張可能

#### 3.2.5 コード例（進捗率チャート）
```jsx
import dynamic from 'next/dynamic';
const ReactApexChart = dynamic(() => import('react-apexcharts'), { ssr: false });

const ProgressChart = () => {
  const options = {
    chart: {
      type: 'line',
      zoom: { enabled: true }
    },
    xaxis: {
      type: 'datetime'
    },
    yaxis: {
      reversed: true,
      min: 0,
      max: 100
    },
    stroke: {
      width: [3, 3],
      dashArray: [0, 5]
    },
    tooltip: {
      y: {
        formatter: (val) => `${val}%`
      }
    }
  };
  
  const series = [
    { name: '計画', data: [...] },
    { name: '実績', data: [...] }
  ];
  
  return (
    <ReactApexChart options={options} series={series} type="line" height={350} />
  );
};
```

## 4. 実装コスト比較

### 4.1 バンドルサイズへの影響

| ライブラリ | 通常バンドル | 最適化済みバンドル | 補足 |
|----------|--------------|-------------------|------|
| **Recharts** | 90KB | 70KB | Tree-shakingにより不要な部分を削減可能 |
| **ApexCharts** | 115KB | 95KB | モジュール分割による最適化がやや難しい |
| **Recharts + html2canvas** | 140KB | 120KB | 画像出力機能追加時 |

### 4.2 メンテナンスコスト
- **Recharts**: カスタム機能の追加により、バージョンアップ時の互換性確認が必要
- **ApexCharts**: 標準機能が多いため、基本的にはバージョンアップによる問題は少ない

## 5. 評価基準に基づく比較

各項目を1〜5の5段階で評価（5が最高評価）

| 評価基準 | 重み | Recharts | ApexCharts | 重み付きRecharts | 重み付きApexCharts |
|---------|------|----------|------------|-------------------|---------------------|
| **ITリテラシーが低いユーザー向けUX** | 5 | 3 | 5 | 15 | 25 |
| **モダンなUI/UX** | 4 | 4 | 5 | 16 | 20 |
| **実装の容易さ** | 3 | 4 | 5 | 12 | 15 |
| **パフォーマンス** | 3 | 4 | 4 | 12 | 12 |
| **カスタマイズ性** | 2 | 5 | 4 | 10 | 8 |
| **開発工数** | 4 | 3 | 5 | 12 | 20 |
| **将来の拡張性** | 3 | 4 | 4 | 12 | 12 |
| **合計** | | | | **89** | **112** |

## 6. 選定結果と根拠

### 6.1 選定ライブラリ: ApexCharts

総合評価の結果、**ApexCharts**を採用する。主な選定理由は以下の通り：

1. **標準機能の充実**: 進捗率表示に必要なズーム機能や画像エクスポート機能が標準搭載されており、開発工数を大幅に削減できる

2. **ユーザー体験**: ITリテラシーが必ずしも高くない工事現場のスタッフでも直感的に操作可能なインターフェースを提供

3. **実装コスト**: 標準機能が充実しているため、カスタム実装の必要性が少なく、総開発工数を約半分に抑えられる

4. **多言語対応**: 日本語を含む多言語対応が標準搭載されており、表示やコントロールの日本語化が容易

5. **タブレット対応**: タッチ操作に最適化されており、工事現場でのタブレット使用に適している

### 6.2 懸念点と対策

| 懸念点 | 対策 |
|--------|------|
| **バンドルサイズがやや大きい** | レイジーロード（動的インポート）を使用し、初期ロード時間を最適化する |
| **SSRでの問題** | Next.jsでは`dynamic`インポートと`ssr: false`オプションを使用して対応する |
| **カスタマイズの複雑さ** | 必要最小限のカスタマイズに留め、標準機能を最大限活用する |

## 7. 実装計画

### 7.1 実装ステップ

1. **基本環境セットアップ**
   - ApexCharts、react-apexchartsのインストール
   - Next.jsでの適切な設定（dynamic importの設定）

2. **進捗率チャートコンポーネント実装**
   - 基本的なチャート構造の実装
   - 日本語表示の設定
   - Y軸反転（左上から右下への表示）の実装

3. **インタラクション機能の実装**
   - ズーム・パン機能の最適化
   - ツールチップのカスタマイズ
   - 期間切替機能の実装

4. **エクスポート機能の実装**
   - SVG、PNG形式でのエクスポート
   - ファイル名の自動生成
   - 出力サイズの最適化

5. **shadcnとの統合**
   - カラーテーマの統一
   - カードレイアウトの適用
   - コントロール要素のスタイル統一

### 7.2 コードサンプル

ApexChartsでの進捗率チャート実装の基本的なコード例：

```jsx
import React, { useState } from 'react';
import dynamic from 'next/dynamic';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';

// クライアントサイドのみでロード
const ReactApexChart = dynamic(() => import('react-apexcharts'), { ssr: false });

const ProgressChart = () => {
  const [viewType, setViewType] = useState("month");
  
  // 日本語化設定
  const japaneseLocale = {
    name: 'ja',
    options: {
      months: ['1月', '2月', '3月', '4月', '5月', '6月', '7月', '8月', '9月', '10月', '11月', '12月'],
      shortMonths: ['1月', '2月', '3月', '4月', '5月', '6月', '7月', '8月', '9月', '10月', '11月', '12月'],
      days: ['日曜日', '月曜日', '火曜日', '水曜日', '木曜日', '金曜日', '土曜日'],
      shortDays: ['日', '月', '火', '水', '木', '金', '土'],
      toolbar: {
        exportToSVG: 'SVGでダウンロード',
        exportToPNG: 'PNGでダウンロード',
        menu: 'メニュー',
        selection: '選択',
        selectionZoom: '選択してズーム',
        zoomIn: 'ズームイン',
        zoomOut: 'ズームアウト',
        pan: 'パン',
        reset: 'ズームをリセット'
      }
    }
  };
  
  // チャート設定
  const options = {
    chart: {
      type: 'line',
      zoom: { enabled: true },
      toolbar: { show: true },
      locales: [japaneseLocale],
      defaultLocale: 'ja'
    },
    colors: ['#2563eb', '#dc2626'],
    stroke: {
      width: [3, 3],
      curve: 'smooth',
      dashArray: [0, 5]
    },
    xaxis: {
      type: 'datetime',
      labels: {
        formatter: (val) => {
          const date = new Date(val);
          return `${date.getMonth() + 1}/${date.getDate()}`;
        }
      }
    },
    yaxis: {
      reversed: true,
      min: 0,
      max: 100,
      labels: {
        formatter: (val) => `${val}%`
      }
    },
    tooltip: {
      shared: true,
      y: {
        formatter: (y) => `${y.toFixed(1)}%`
      }
    }
  };
  
  const series = [
    {
      name: '計画',
      data: [/* 計画データ */]
    },
    {
      name: '実績',
      data: [/* 実績データ */]
    }
  ];
  
  return (
    <Card className="w-full">
      <CardHeader>
        <CardTitle>工事進捗率</CardTitle>
        <Select value={viewType} onValueChange={setViewType}>
          <SelectTrigger>
            <SelectValue placeholder="表示期間" />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="week">週</SelectItem>
            <SelectItem value="month">月</SelectItem>
            <SelectItem value="6months">6ヶ月</SelectItem>
          </SelectContent>
        </Select>
      </CardHeader>
      <CardContent>
        {typeof window !== 'undefined' && (
          <ReactApexChart 
            options={options} 
            series={series} 
            type="line" 
            height={320} 
          />
        )}
      </CardContent>
    </Card>
  );
};

export default ProgressChart;
```

## 8. 結論

工程管理アプリケーションの進捗率表示には、**ApexCharts**を採用する。これにより、ITリテラシーが必ずしも高くない工事現場のユーザーでも直感的に操作できるインタラクティブなチャートを、最小限の開発工数で実現できる。特に標準搭載のズーム機能と画像エクスポート機能は、ユーザーの利便性と開発効率の両面で大きなメリットがある。

ApexChartsを採用することで、Rechartsと比較して開発工数を約半減させつつ、ユーザー体験の向上を実現できる見込みである。バンドルサイズについては、適切なレイジーロード対策を行うことで問題を最小化する。

既存のガントチャート（自作モジュール）と合わせて使用することで、工程管理アプリケーションの視覚化機能を効果的に実現する。


