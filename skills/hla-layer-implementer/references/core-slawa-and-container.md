# Container.svg と S-LaWA の最小構成

この資料は最小のファイル関係を示す。APIや属性の現行仕様は[SVGMap公式チュートリアル](https://svgmap.github.io/tutorials/)と[`svgmap/svgmapjs`](https://github.com/svgmap/svgmapjs)、[公式解説書](https://www.svgmap.org/wiki/index.php?title=%E8%A7%A3%E8%AA%AC%E6%9B%B8)で確認する。

## ファイルの役割

LaWAでは、少なくとも次の役割を区別する。

```text
Container.svg                     # 表示するレイヤーを宣言するルートコンテナ
layers/my_custom_layer.svg        # 描画対象となるレイヤーSVG。data-controller属性を持つ
layers/my_custom_layer.html       # レイヤー固有UIとJavaScriptを持つコントローラー
```

`data-controller`はレイヤーSVGのルート`<svg>`に付ける属性であり、`Container.svg`自体でも、専用のファイル名でもない。コントローラーHTMLはこの属性から参照する。通常はContainer.svgの`<animation>`からレイヤーSVGを読み込む。

## 1. ルートコンテナ `Container.svg`

Container.svgはレイヤー構成を宣言し、LaWAの描画対象となるSVGを参照する。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg"
     xmlns:xlink="http://www.w3.org/1999/xlink"
     viewBox="139.6 -35.8 0.4 0.4">
  <globalCoordinateSystem
      srsName="http://purl.org/crs/84"
      transform="matrix(1,0,0,-1,0,0)" />

  <animation
      xlink:href="layers/my_custom_layer.svg"
      x="139.6" y="-35.8" width="0.4" height="0.4"
      title="カスタムレイヤー"
      class="overlay"
      visibility="visible"
      data-lawa-mode="isolated" />
</svg>
```

`data-lawa-mode="isolated"`は、同一オリジンのレイヤーもS-LaWAとして隔離したい例である。クロスオリジンのLaWAは通常自動でS-LaWAになる。動作モードの既定値や属性は、対象のSVGMap.js版の公式説明で確認する。

## 2. 描画対象のレイヤーSVG

LaWA用のSVGファイルを別に作成し、ルート要素にコントローラーHTMLを指定する。描画要素を追加するグループなど、レイヤーの初期SVG DOMもここに置く。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg"
     xmlns:xlink="http://www.w3.org/1999/xlink"
     viewBox="139.6 -35.8 0.4 0.4"
     data-controller="my_custom_layer.html#exec=hiddenOnLayerLoad">
  <globalCoordinateSystem
      srsName="http://purl.org/crs/84"
      transform="matrix(1,0,0,-1,0,0)" />
  <g id="featureGroup" />
</svg>
```

`data-controller`の値はHTMLコントローラーへのURLである。`#exec=hiddenOnLayerLoad`はUIを非表示状態で起動する例で、UIを初期表示する場合などは必要な起動方法を選ぶ。

## 3. S-LaWAコントローラーHTML

S-LaWAでは`svgMapSandboxLayerLib.js`をES moduleとして読み込み、初期化完了後の`layerWebAppReady`イベントで処理を始める。`svgMap`と`svgImage`はサンドボックス内で提供されるAPI/ローカルSVG DOMである。親ページのDOMや他レイヤーへ直接アクセスしない。

```html
<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8">
  <title>カスタムレイヤー</title>
  <script type="module"
    src="https://cdn.jsdelivr.net/gh/svgmap/svgmapjs@latest/svgMapSandboxLayerLib.js"></script>
  <script>
    window.addEventListener("layerWebAppReady", () => {
      const group = window.svgImage.getElementById("featureGroup");
      if (!group) return;

      // ここで自レイヤーのSVG DOMを更新する。
      // 利用する描画APIと座標変換は公式資料で確認する。
      window.svgMap.refreshScreen();
    });
  </script>
</head>
<body>
  <section id="layer-ui">レイヤー固有の操作UI</section>
  <section id="layer-credit">必要な出典・利用条件を表示</section>
</body>
</html>
```

`svgMapSandboxLayerLib.js`はES moduleであるため、`type="module"`で読み込む。初期化時に`load`イベントだけを使う実装は、S-LaWAとの接続完了前にAPIを呼ぶ可能性があるため避ける。

## `data-controller`をContainer側に置く場合

公式解説には、レイヤーSVGのルートに付ける代わりに、Container.svg内のそのレイヤーを参照する`<animation>`要素へ`data-controller`を設定する方式も記載されている。CORS非依存モード等の要件で使う選択肢であり、通常の基本構成とは区別して、対象バージョンの公式説明を確認してから使う。
