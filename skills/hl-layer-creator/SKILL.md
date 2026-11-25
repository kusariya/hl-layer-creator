---
name: hl-layer-creator
description: >-
  SVGMap / HLA の地図レイヤーを、公式チュートリアルを起点に設計・実装・レビューする。
  静的SVG、Container.svg、LaWA、Webサービス連携、既存WebApp Layerの活用を扱う。
---

# HLA レイヤー開発ガイド

SVGMap / HLA の地図アプリやレイヤーを作成・変更するときに使う。チュートリアルを学習経路として利用し、実際の要件に合う段階から進める。例をそのまま仕様とみなさず、APIや属性の現行仕様は公式ドキュメント・実装で確認する。

## 進め方

1. ユーザーの成果物、データの所在（URL、ローカルファイル、既存レイヤー）、必要な表示や操作を把握する。不明点が作業の境界を変える場合だけ確認する。
2. [公式チュートリアルの学習経路](./references/tutorial_workflow.md)から対応する段階を選ぶ。既存の動作例を使う場合は、そのチュートリアルやWebApp Layerの構成を確認してから適用する。
3. 要件に応じ、静的なSVGレイヤー、Container.svgでのレイヤー構成、LaWA、Webサービス連携、既存WebApp Layerの利用から適切な構成を選ぶ。データ取得方式は利用条件、可用性、CORS、更新頻度、ユーザー提供ファイルの有無を考慮する。
4. 詳細な設計・実装では該当する参照資料を読む。コード/API例を作る前に、現行の公式資料またはチュートリアルのソースで裏付ける。
5. 実装後は [ハーネス監査票](./references/harness_and_anti_patterns.md) を使い、レイヤー境界、Container.svg、座標系、セキュリティ、データ出典を確認する。

## 設計の基本

- Container.svg は地図とレイヤーを宣言的に構成する。レイヤー固有の取得、描画、操作UIは、そのレイヤーの責務に置く。
- 地図データは、利用条件と利用形態に合う方法で参照する。リモートデータは直接取得を候補とし、独自のローカルデータやチュートリアル素材も要件に応じて扱う。取得・保存・再配布の条件を推測で決めない。
- LaWAの種類やAPIは、対象のチュートリアルと最新の公式仕様で確認する。T-LaWAとS-LaWAの境界・通信方式を混同しない。
- チュートリアルは段階的な実例であり、すべての機能をLaWA化することや、単一のデータ取得方式を採用することを要求する規範ではない。

## 専門スキルとの連携

ユーザーの目的に応じて必要なスキルだけ使う。データ探し、利用条件確認、データ構造調査、設計、実装、アーキテクチャレビューをすべて常に実行する必要はない。

| 必要な作業 | スキル |
| --- | --- |
| 接続先候補を探す | `hla-source-finder` |
| ライセンス・利用条件を調べる | `hla-license-checker` |
| URLや提供データを調査する | `hla-data-inspector` |
| 要件からレイヤー構成を設計する | `hla-layer-architect` |
| APIを調べて実装する | `hla-layer-implementer` |
| HLA境界やアンチパターンをレビューする | `hla-architecture-reviewer` |

## 参照資料

- チュートリアルの段階・対応内容: [tutorial_workflow.md](./references/tutorial_workflow.md)
- HLAの概念と構成の選択: [architecture.md](./references/architecture.md)
- Container.svg、レイヤーSVG、ホストHTMLの仕様メモ: [specification.md](./references/specification.md)
- LaWAの動作モードと開発手順: [lawa_guide.md](./references/lawa_guide.md)
- データ所在・利用条件に応じた供給方式: [data_source_architecture.md](./references/data_source_architecture.md)
- アンチパターンとレビュー項目: [harness_and_anti_patterns.md](./references/harness_and_anti_patterns.md)
