---
name: hla-layer-implementer
description: 設計仕様に基づき、SVGMap.jsおよびS-LaWA（svgMapLayerLib）を用いたHLA準拠レイヤーを実装する。HLAの原則に従い、データを手元にDLせずLaWAから直接fetchしデータのコピーを作成しない実装を徹底する。最新のAPI仕様はインターネット上の公式リソースを最優先で検索・参照してコーディングする。
---

# HLA Layer Implementer

## API調査および実装の鉄則
1. **データの非複製とLaWA直接fetchの徹底（最重要）**
   - **データを手元にダウンロード（DL）してプロジェクト内に保存・バンドル（コピーの作成）してはならない。**
   - 必ずS-LaWA内のJavaScriptから提供元URLへ直接 `fetch()` を行い、メモリ上で動的にSVG DOM（`window.svgImage`）へ変換・描画すること。
   - これにより、データの二重管理・陳腐化を防ぎ、ライセンス上の再配布リスクを排除する。
2. **インターネット検索による最新API確認（最優先）**
   - SVGMap.js のAPIは日々変化するため、実装前に必ずWeb検索や公式リポジトリ（`hl-layer-creator/svgmapjs` 等）を参照し、最新の `svgMap` / `svgMapLayerLib` や `appLayer` 等のメソッド名・引数・仕様を確認すること。
   - 推測や古い記憶だけで未知のAPIを呼び出してはならない。
3. **オフライン・フォールバック規定**
   - ネットワーク制限等でインターネット上の最新ドキュメントにアクセスできない場合に限り、`references/core-slawa-and-container.md` に記載された不変の基本構造（S-LaWAの作成法とコンテナ.svgへの追加法）のみを使用して実装すること。
4. **分散アーキテクチャとコア機能との責務分離**
   - データ取得（`fetch`）、DOM生成・描画、画面更新要求（`svgMap.refreshScreen()` 等）はレイヤー境界（S-LaWA等）内で完結させること。
   - `appLayer` 等でLeafletを重ね合わせる実装を行う場合も、レイヤー外へロジックやDOM操作を漏らさずカプセル化すること。
   - レイヤーの表示/非表示やアンロード等のライフサイクル制御はHLAのコア機能（iframe破棄・表示制御等）に委ね、レイヤー側でコアの制御を妨げないこと。
5. **出典・ライセンス表記への配慮**
   - `hla-license-checker` で整理された確認事項に基づき、必要に応じてS-LaWA内UIに出典・クレジット表記欄（`#layer-credit` 等）を設ける。未確認のライセンス適合性を勝手に保証しない。

## 実装ステップ
1. 必要なAPI（DOM操作、座標変換、イベントハンドラ、LayerUI / `appLayer` 連携等）について最新の公式仕様をWeb検索で確認する（オフライン時は `references/core-slawa-and-container.md` を参照）。
2. レイヤー本体となるベースSVGファイルを作成する。
3. S-LaWA（`data-controller` で紐づくHTML/JS）を実装する。手元DLではなく提供元URLから直接 `fetch()` してオンザフライ描画し、出典・利用条件の表示領域を適切に配置する。
4. コンテナSVG（`container.svg` 等）に `<animation>` 要素を用いてレイヤーを組み込む設定を出力する。
5. 実装完了後、必ず `hla-architecture-reviewer` スキルを呼び出してコードレビューを実行する。