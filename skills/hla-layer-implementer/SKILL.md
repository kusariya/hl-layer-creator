---
name: hla-layer-implementer
description: 設計仕様に基づき、SVGMap.jsおよびLaWA（Layers as WebApps）を用いたHLA準拠レイヤーを実装する。公式仕様を確認し、Container.svg、レイヤーSVG、LaWAコントローラーの役割を分けて構成する。
---

# HLA Layer Implementer

## API調査および実装の鉄則
1. **データ取得方式の選択**
   - リモートデータの直接取得、既存の静的アセット、ユーザー提供ファイル、サービス連携などから、利用条件と要件に合う方式を選ぶ。
   - 取得・保存方式だけでライセンス適合性を判断しない。出典、キャッシュ、再配布等の条件が不明なら、未確認事項として示す。
2. **インターネット検索による最新API確認（最優先）**
   - SVGMap.js のAPIは更新されるため、実装前に公式ドキュメントや公式リポジトリ（[`svgmap/svgmapjs`](https://github.com/svgmap/svgmapjs) 等）を参照し、使用する `svgMap`、LaWA初期化ライブラリ、`appLayer` 等の仕様を確認する。
   - 推測や古い記憶だけで未知のAPIを呼び出してはならない。
3. **オフライン・フォールバック規定**
   - ネットワーク制限等で最新資料にアクセスできない場合は、`references/core-slawa-and-container.md` の基本構造を出発点にする。そこにある構造・API例も対象バージョンでの動作を未確認なら、その旨を明示する。
4. **分散アーキテクチャとコア機能との責務分離**
   - データ取得（`fetch`）、DOM生成・描画、画面更新要求（`svgMap.refreshScreen()` 等）はレイヤー境界（S-LaWA等）内で完結させること。
   - `appLayer` 等でLeafletを重ね合わせる実装を行う場合も、レイヤー外へロジックやDOM操作を漏らさずカプセル化すること。
   - レイヤーの表示/非表示やアンロード等のライフサイクル制御はHLAのコア機能（iframe破棄・表示制御等）に委ね、レイヤー側でコアの制御を妨げないこと。
5. **出典・ライセンス表記への配慮**
   - `hla-license-checker` で整理された確認事項に基づき、必要に応じてS-LaWA内UIに出典・クレジット表記欄（`#layer-credit` 等）を設ける。未確認のライセンス適合性を勝手に保証しない。

## 実装ステップ
1. 必要なAPI、LaWAモード、DOM操作、座標変換について公式資料を確認する（オフライン時は `references/core-slawa-and-container.md` を参照）。
2. LaWAを実装する場合は、描画対象となるレイヤーSVG（例: `my_custom_layer.svg`）を作成し、ルートの`<svg>`要素に`data-controller`でコントローラーHTMLを紐づける。
3. コントローラーHTMLで対象モードの初期化ライブラリを読み込む。S-LaWAでは公式の`svgMapSandboxLayerLib.js`と`layerWebAppReady`イベントを使い、利用可能なAPIを現行資料で確認する。
4. `Container.svg`からレイヤーSVGを`<animation xlink:href="...">`で参照する。通常の構成では`data-controller`はレイヤーSVG側に記述する。Container内の参照要素に記述する方式は、公式資料で確認した特別な要件がある場合に限る。
5. データの取得・保存、出典表示は利用条件と要件に沿って設計する。実装後は、必要に応じて`hla-architecture-reviewer`で境界や仕様をレビューする。
