# SVGMap アーキテクチャハーネス＆アンチパターン集

本ドキュメントは、LeafletやOpenLayersなどの従来のWebGISに慣れ親しんだエンジニアやAIモデルが犯しがちな設計ミス（アンチパターン）を明文化し、それを防ぐための**アーキテクチャハーネス（思考防止ガードレール）**を提供します。

---

## 1. 5大アンチパターン対比表

### ❌ アンチパターン 0: データの手元ダウンロード・コピー作成 (Data Duplication)

外部やオープンデータのShapefile/GeoJSON/CSVを手元にダウンロードし、プロジェクト内に静的ファイルとして保存・抱え込むパターンです。

#### ❌ 誤った思考
「外部APIからデータを落としてプロジェクト内の `data/shelters.geojson` に保存し、それを静的に配信すれば確実だ」
→ **重大な違反**。データを手元にダウンロードしてコピーを作成すると、元データが更新された際に追従できず陳腐化し、再配布禁止規約に抵触するリスクが高まります。また、分散型アーキテクチャの利点が損なわれます。

#### ✅ 正しい設計
外部提供元のURLから、**LaWA自身が直接 `fetch('https://api.example.com/shelters.geojson')`** を行い、オンザフライで描画します。**データのコピーは一切作成しません。**

---

### ❌ アンチパターン 1: 単一スクリプト・モノリス症候群 (Leaflet-itis)

最も頻発する違反です。`index.html` または `main.js` の中に、背景地図・主題図・API通信・UI制御のすべてを詰め込んでしまうパターンです。

#### ❌ 誤ったコード (Leaflet的思考の残滓)
```javascript
// index.html または main.js
// 従来のWebGISのように中央ですべてを管理しようとする重大な違反
import { svgMap } from 'svgmap.js';

// 地図初期化
const map = svgMap.init('#mapcanvas');

// 全レイヤーのデータを親スクリプトで一括フェッチ
fetch('https://api.example.com/rivers.json').then(r => r.json()).then(rivers => {
    // 親スクリプトが直接要素を作成してマップに突っ込む
    renderRiversOnGlobalCanvas(rivers);
});

fetch('https://api.example.com/shelters.geojson').then(r => r.json()).then(shelters => {
    renderSheltersOnGlobalCanvas(shelters);
});

// UIコントロールのイベントも親スクリプトが全抱え
document.getElementById('river-slider').addEventListener('input', (e) => {
    filterRiversGlobally(e.target.value);
});
```

#### ✅ 正しい設計 (SVGMap / LaWA 分散型アプローチ)
```
[構成]
Container.svg
  ├── <animation xlink:href="layers/osm.svg" />          <- 背景地図はOSM LaWAに任せる
  ├── <animation xlink:href="layers/rivers.svg" />       <- 河川データ・スライダーはrivers LaWAにカプセル化
  └── <animation xlink:href="layers/shelters.svg" />     <- 避難所データ・検索はshelters LaWAにカプセル化

// index.html はこれだけ！
<div id="mapcanvas" data-src="Container.svg"></div>
<script type="module">
    import { svgMap } from '.../SVGMapLv0.1_r18module.js';
    window.svgMap = svgMap;
</script>
```
**ハーネス**:
> **親ページ（`index.html`）はマウントポイントにすぎない。親ページで `fetch()` を行ったり、特定のレイヤーのためのDOM要素やUIイベントリスナーを書いてはならない。**

---

### ❌ アンチパターン 2: 受動的データ注入 (Passive Data Injection)

レイヤーを「自律したWebアプリ」ではなく、単なる「GeoJSONデータファイル」として扱い、外部から受動的に注入しようとするパターンです。

#### ❌ 誤った思考
「`Container.svg` にGeoJSONファイルのURLを直接 `<animation xlink:href="data.geojson">` のように指定すればいいのではないか？」
→ **不可**。GeoJSONはWebアプリケーションでもSVGドキュメントでもありません。SVGMapはGeoJSONをネイティブで直接解釈して画面に描画するのではなく、**GeoJSONを描画する能力を持った動的レイヤー（LaWA）** を介してマッシュアップします。

#### ✅ 正しい設計
1. `layers/my_feature_layer.svg` を作成し、`data-controller="my_controller.html"` を指定する。
2. `my_controller.html` / `my_controller.js` の中で GeoJSON を非同期取得し、`svgImage` に対して描画する。
3. `Container.svg` からは `my_feature_layer.svg` を参照する。

---

### ❌ アンチパターン 3: 宣言的コンテナの欠落・命令的レイヤー生成

JavaScriptの命令的APIで `map.addLayer(new Layer(...))` のように動的生成しようとして、`Container.svg` の作成を怠るパターンです。

#### ❌ 誤った思考
```javascript
// JavaScriptだけで動的にレイヤーオブジェクトを生成して追加しようとする
const basemapLayer = new SVGMap.Layer('osm.svg');
svgMap.addLayer(basemapLayer); // ※このような命令的API中心の設計はアンチパターン
```

#### ✅ 正しい設計
SVGMapの核心は、**URLでアクセス可能な宣言的SVGドキュメントマニフェスト（`Container.svg`）** です。
レイヤーの追加や構成変更は、`Container.svg` 内に `<animation xlink:href="..." />` を宣言することで行います。

---

### ❌ アンチパターン 4: 空間座標系（CRS）の自己流変換・Y軸反転の忘却

SVGの座標系と地理座標系（緯度経度）の違いを自前のJavaScript計算ルーチンで場当たり的に変換しようとするパターンです。

#### ❌ 誤った思考
```javascript
// 画面ピクセルと緯度経度を独自関数で手計算しようとする
function lonLatToPixel(lon, lat) {
    const x = (lon - 139.0) * 1000;
    const y = (lat - 35.0) * 1000; // 緯度の正負や反転を考慮せず上下逆さまに！
    return { x, y };
}
```

#### ✅ 正しい設計
SVGMapフレームワークのコア機能である `<globalCoordinateSystem>` を厳格に宣言します。
```xml
<globalCoordinateSystem srsName="http://purl.org/crs/84" transform="matrix(1,0,0,-1,0,0)" />
```
- 第1引数 `1` (X軸スケール)
- 第4引数 `-1` (**Y軸スケール: 南北反転に対応するため必ずマイナス**)
- これにより、SVG内の座標系として `cx="139.75" cy="-35.68"` のように経度と反転緯度をそのまま利用できます。

---

### ❌ アンチパターン 5: 親ページのグローバルCSS/DOM汚染

各レイヤーのUI（凡例、ツールチップ、コントロールボタン）を親ページのDOMに直接配置し、グローバルCSSで装飾しようとするパターンです。

#### ❌ 誤ったコード
```html
<!-- index.html の中に特定のレイヤー用UIを直書きしてしまう -->
<div id="mapcanvas" data-src="Container.svg"></div>
<div id="river-layer-legend">
    <h4>河川警戒レベル</h4>
    <span class="red">危険水位</span>
</div>
```

#### ✅ 正しい設計
各レイヤーのUIは、そのレイヤー自身のコントローラーHTML（例: `river_controller.html`）の中に記述します。
SVGMapフレームワークは、`data-controller="river_controller.html#exec=appearOnLayerLoad"` を検出すると、そのレイヤー専用のウィンドウ/iframeを地図上にオーサリングして表示します。これにより、レイヤーを非表示にすればUIも自動で消え、スタイルシートも完全にカプセル化されます。

---

### ❌ アンチパターン 6: サンドボックス境界の越境 (Sandbox Boundary Violation)

S-LaWA内のスクリプトから、親フレームのDOM（`parent.document`, `top.document`）や他レイヤーのDOM・グローバル状態を直接書き換えようとするパターンです。

#### ❌ 誤った思考
```javascript
// レイヤーコントローラー内から親画面や他レイヤーを直接操作しようとする重大な違反
window.parent.document.getElementById('global-title').textContent = '新しいタイトル';
parent.svgMap.someGlobalState = 'modified';
```

#### ✅ 正しい設計
S-LaWAはiframeサンドボックスによる完全な境界分離を前提とします。他レイヤーや親ビューアへの越境は、セキュリティ脆弱性やクロスオリジンでの破綻を招きます。自レイヤーのグラフィックス更新は `window.svgImage` および公式API（`svgMap.refreshScreen()` 等）経由に限定してください。

---

### ❌ アンチパターン 7: コア機能の責務侵害と宣言的ズーム制御の放棄

ズームレベルに応じた表示/非表示（LOD制御）をコンテナSVGの宣言的属性（`visible-min-zoom` / `visible-max-zoom`）で行わず、JavaScript内で毎回ズーム倍率を自前計算して全要素を削除・再生成したり、コアエンジンが管理するレイヤーのライフサイクル（破棄・非アクティブ化）を上書きしようとするパターンです。

#### ❌ 誤った思考
```javascript
// JavaScript側でズーム倍率を監視して全DOMを手動で消したり作ったりする
window.addEventListener("zoomPanMap", () => {
    if (getCurrentZoom() < 10) {
        removeAllElements(); // ※コアが担うべき表示制御を自前で再発明している
    }
});
```

#### ✅ 正しい設計
ズーム表示範囲は `Container.svg` の `<animation>` 要素に属性として宣言します：
```xml
<animation xlink:href="layers/buildings.svg"
           visible-min-zoom="1000"
           visible-max-zoom="50000"
           title="建物詳細" />
```
表示・非表示やメモリ管理（非アクティブ時の解放等）はSVGMapコアエンジンが最適に制御します。レイヤー側でコアの責務を奪ってはなりません。

---

## 2. 開発前・レビュー時チェックリスト（ハーネス監査票）

コードを書く前、およびレビュー時には必ず以下のチェックリストをパスしているか確認してください。

- [ ] **データの非複製原則**: データを手元にダウンロードしてコピーを作成・保持せず、LaWA自身が提供元から直接 `fetch()` しているか？
- [ ] **マニフェストの存在**: ルートに `Container.svg` が作成されているか？
- [ ] **XML名前空間の定義**: `xmlns="http://www.w3.org/2000/svg"` および `xmlns:xlink="http://www.w3.org/1999/xlink"` が定義されているか？
- [ ] **空間参照系の宣言**: `Container.svg` および各レイヤーSVGに `<globalCoordinateSystem>` が存在し、Yスケールが負数になっているか？
- [ ] **レイヤーのカプセル化**: 動的データ取得（fetch等）が親ページではなく、各LaWAのコントローラーJS内で行われているか？
- [ ] **サンドボックス境界の遵守**: `parent.document` や `top.document` 等で親フレーム・他レイヤーのDOMに直接干渉していないか？
- [ ] **コア機能との責務分担**: ズーム倍率に応じたLOD制御に `visible-min-zoom` / `visible-max-zoom` 属性が活用されているか？
- [ ] **通信ライブラリの読み込み**: LaWAコントローラーHTML内で `svgMapLayerLib.js`（または `svgMapSandboxLayerLib.js`）がインポートされているか？
- [ ] **視域イベントの購読**: コントローラーJS内で `window.addEventListener("zoomPanMap", ...)` をリッスンしているか？
- [ ] **自レイヤーDOMの操作**: コントローラーJSが親DOMではなく `window.svgImage` を通じてレイヤーSVGを更新しているか？
- [ ] **再描画通知の実行**: DOM操作完了後に `svgMap.refreshScreen()` を呼び出しているか？
- [ ] **出典・ライセンス表記枠の設置**: S-LaWA内UIに元データやタイルの出典・クレジット・利用条件を表示する領域（`#layer-credit` 等）が用意されているか？
- [ ] **相対パスの徹底**: すべてのリンク、インポート、データ参照が相対パスで記述されているか？
- [ ] **セキュリティの考慮**: 第三者提供のレイヤーを組み込む場合、T-LaWAではなくS-LaWA（サンドボックス）が選択されているか？
