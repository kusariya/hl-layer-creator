# S-LaWA および コンテナ.svg 最小構成リファレンス（フォールバック用）

※原則としてWeb検索で最新の `hl-layer-creator/svgmapjs` 仕様を確認すること。以下はオフライン時でも変わらない基本構造である。

## 1. コンテナ.svg（Container / Root SVG）へのレイヤー追加方法
コンテナSVG内の `<svg>` ルート配下に `<animation>` 要素を記述してレイヤーを登録する。
`data-controller` 属性でS-LaWA（サンドボックス化されたレイヤーUI・ロジックHTML）を紐づける。

```xml
<animation
  xlink:href="my_custom_layer.svg"
  x="-30000" y="-30000" width="60000" height="60000"
  title="カスタムレイヤー名"
  class="editable"
  visibility="visible"
  visible-min-zoom="500"
  visible-max-zoom="50000"
  data-controller="my_custom_layer_controller.html"
/>
```
- `xlink:href`: レイヤーのベースとなるSVGファイルパス（またはURL）。
- `x, y, width, height`: HLAのグローバル座標系をカバーするバウンディングボックス。
- `visible-min-zoom` / `visible-max-zoom`: レイヤーが表示されるズーム倍率の範囲（コア機能がこの属性に基づき表示・読み込みを制御する）。
- `data-controller`: このレイヤー専用のS-LaWA（HTML+JS）のパス。

## 2. S-LaWA（サンドボックス・レイヤーWebApp）の基本構造
`data-controller` で指定されるHTMLファイル。メインの地図ビューアとは分離されたコンテキストで動作し、`svgMapLayerLib` を通じて自身のレイヤーSVG（`svgImage`）等を操作する。手元ファイルの展開やレイヤー内でのLeaflet重ね合わせ（`appLayer` 等）を行う場合も、このレイヤー境界内に閉じ込める。

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<script src="https://cdn.jsdelivr.net/gh/hl-layer-creator/svgmapjs@latest/svgMapLayerLib.js"></script>
<script>
window.addEventListener("load", function() {
    initLayer();
});

function initLayer() {
    // 1. HLAの最重要原則: データを手元にDLせず、LaWAから直接fetchしてデータのコピーを作成しない
    // async function loadData() {
    //     try {
    //         const response = await fetch("https://example.com/api/features.geojson");
    //         const data = await response.json();
    //         renderFeatures(data);
    //     } catch (error) {
    //         console.error("データ取得エラー:", error);
    //     }
    // }
    //
    // 2. ローカルXML DOM (svgImage) の操作例
    // const circle = svgImage.createElement("circle");
    // circle.setAttribute("cx", lon * 100);
    // circle.setAttribute("cy", -lat * 100);
    // circle.setAttribute("r", 5);
    // svgImage.documentElement.appendChild(circle);
    //
    // 3. 地図画面の再描画をビューアに要求
    svgMap.refreshScreen();
}
</script>
</head>
<body>
  <div id="layer-ui">レイヤー設定</div>
  <div id="layer-credit">出典・利用条件: 製作者にて元データの規約を確認・記載してください</div>
</body>
</html>
```