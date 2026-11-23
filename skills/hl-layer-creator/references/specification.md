# HLA 仕様リファレンス (Specification Reference)

本ドキュメントは、HLA (HLMap) の各種仕様、XML/SVGスキーマ、属性定義、座標系マトリクス、およびホストHTMLの構成仕様を定義したリファレンスです。開発時は本仕様に厳密に準拠してください。

---

## 1. コンテナSVG仕様 (`Container.svg`)

`Container.svg` は、地図全体の空間参照系、初期表示範囲（視域）、および構成するすべてのレイヤーを宣言的に束ねる**マスターマニフェストファイル**です。

### 基本スキーマ構造
```xml
<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg" 
     xmlns:xlink="http://www.w3.org/1999/xlink"
     viewBox="139.6 -35.8 0.4 0.4">

    <!-- 空間参照系と座標変換行列の定義 (必須) -->
    <globalCoordinateSystem srsName="http://purl.org/crs/84" transform="matrix(1,0,0,-1,0,0)" />

    <!-- レイヤー定義グループ -->
    <!-- 背景地図レイヤー群 (排他選択可能なスイッチクラス) -->
    <g id="basemaps">
        <animation xlink:href="layers/osm.svg" title="OpenStreetMap" class="basemap switch" visibility="visible" />
        <animation xlink:href="layers/gsi.svg" title="地理院地図" class="basemap switch" visibility="hidden" />
    </g>

    <!-- 主題図レイヤー群 (オーバーレイ) -->
    <g id="overlays">
        <animation xlink:href="layers/coastline.svg" title="海岸線" class="overlay" visibility="visible" />
        <animation xlink:href="layers/facilities.svg" title="公共施設" class="overlay" visibility="visible" />
    </g>
</svg>
```

### 要素および属性の詳細仕様

#### 1. `<svg>` ルート要素
- `xmlns="http://www.w3.org/2000/svg"`: SVG名前空間宣言（必須）。
- `xmlns:xlink="http://www.w3.org/1999/xlink"`: XLink名前空間宣言（必須。`xlink:href` を使用するため）。
- `viewBox="minX minY width height"`: 初期表示領域を指定。
  - 経度・緯度を直接扱う場合（`transform="matrix(1,0,0,-1,0,0)"`）：
    - `minX`: 最小経度（例: `139.6`）
    - `minY`: 最大緯度の**反転負数**（例: 緯度 `35.8` の場合 `-35.8`）
    - `width`: 経度幅（例: `0.4`）
    - `height`: 緯度幅（例: `0.4`）

#### 2. `<globalCoordinateSystem>` 要素（拡張標準・必須）
SVGベクター空間と地理空間（測地系）を結びつける最重要要素です。
- `srsName`: 空間参照系識別子（IRI）。
  - 推奨値: `http://purl.org/crs/84` (WGS 84 経度・緯度順、十進角)
  - 参考値: `EPSG:4326`
- `transform`: アフィン変換行列 `matrix(a, b, c, d, e, f)`。
  - SVG座標系 $(X_{svg}, Y_{svg})$ と 地理座標 $(Lon, Lat)$ の関係：
    $$X_{svg} = a \cdot Lon + c \cdot Lat + e$$
    $$Y_{svg} = b \cdot Lon + d \cdot Lat + f$$
  - **重要（Y軸反転の必然性）**:
    SVGの画面座標系は「**下がプラス**」ですが、地理座標の緯度は「**北（上）がプラス**」です。そのため、変換行列の $d$（第4引数）は**必ず負数（マイナス）**でなければなりません。
  - **代表的な行列パターン**:
    - `matrix(1, 0, 0, -1, 0, 0)`: 経度・緯度 1度を SVG座標 1単位とする標準設定。
    - `matrix(100.0, 0.0, 0.0, -100.0, 0.0, 0.0)`: 小数点精度落ちを防ぐための100倍スケール設定（日本の経度139.7度→13970、緯度35.6度→-3560）。

#### 3. `<animation>` 要素（レイヤーインポート）
個別のレイヤーファイルをコンテナに動的に組み込むための要素です。
- `xlink:href`: レイヤーSVGファイルへの相対パス（必須）。
- `title`: UI（レイヤー一覧や切り替えコントロール）に表示される人間向け名称。
- `class`: レイヤーの種別・動作指定。
  - `basemap switch`: 背景地図。同一グループ内でラジオボタンのように排他選択される。
  - `overlay`: 主題図・重畳レイヤー。チェックボックスで個別に表示/非表示を切り替え可能。
  - `editable`: 編集可能なベクターレイヤー。
- `visibility`: 初期の表示状態。
  - `visible`: 表示
  - `hidden`: 非表示
- `visible-min-zoom` / `visible-max-zoom`: レイヤーが表示されるズーム倍率の範囲。
  - **重要（コア機能との責務分離）**: 表示倍率（Level of Detail: LOD）に応じた表示/非表示の切り替えは、HLAコアエンジンが本属性に基づいて自動的に最適制御します。レイヤー側のスクリプトで独自にズーム判定して要素を破棄・再生成する必要はありません。
- `x`, `y`, `width`, `height`: グローバル空間上でのレイヤーバウンディングボックス（例: `x="-30000" y="-30000" width="60000" height="60000"`）。
- `data-controller`: レイヤーのコントローラーHTML（S-LaWA）をコンテナ側から直接指定・オーバーライドする場合に使用します。

記述例：
```xml
<animation
    xlink:href="layers/facilities.svg"
    x="-30000" y="-30000" width="60000" height="60000"
    title="公共施設"
    class="overlay"
    visibility="visible"
    visible-min-zoom="500"
    visible-max-zoom="50000"
    data-controller="facilities_controller.html#exec=appearOnLayerLoad" />
```

---

## 2. レイヤーSVG仕様 (`layer.svg`)

各レイヤーの実体となるSVGファイルです。静的レイヤーと動的レイヤー（LaWA）の2種類が存在します。

### A. 静的レイヤー (Static Vector Layer)
固定の図形要素（パス、ポリゴン、サークル）を直接記述するレイヤーです。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg"
     viewBox="139.6 -35.8 0.4 0.4">
    <globalCoordinateSystem srsName="http://purl.org/crs/84" transform="matrix(1,0,0,-1,0,0)" />
    
    <g id="coastlinePaths" stroke="#2c3e50" stroke-width="0.001" fill="none">
        <path d="M 139.65,-35.75 L 139.70,-35.70 L 139.80,-35.65" />
    </g>
</svg>
```

### B. 動的レイヤー (LaWA: Layers as WebApps)
ルート `<svg>` 要素に `data-controller` 属性を付与し、レイヤーを司る自律Webアプリケーション（HTMLファイル）を紐付けます。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg"
     xmlns:xlink="http://www.w3.org/1999/xlink"
     viewBox="139.6 -35.8 0.4 0.4"
     data-controller="shelters_controller.html#exec=appearOnLayerLoad">

    <globalCoordinateSystem srsName="http://purl.org/crs/84" transform="matrix(1,0,0,-1,0,0)" />

    <!-- コントローラーから動的に生成・注入される要素の受け皿グループ -->
    <g id="dynamicFeatureContainer" class="clickable"></g>
</svg>
```

#### `data-controller` 属性と `#exec` コマンド
コントローラーHTMLファイルの相対パスに続けて、ハッシュフラグで動作モードを指定できます：
- `#exec=hiddenOnLayerLoad`: レイヤー読み込み時、コントローラーHTMLのUIウィンドウを**非表示（バックグラウンド動作）**にする（背景地図タイルローダー等に最適）。
- `#exec=appearOnLayerLoad`: レイヤー読み込み時、コントローラーHTMLのUIウィンドウを**自動的に前面表示**する（凡例やスライダー、属性表示ダイアログを持つ主題レイヤーに最適）。

---

## 3. ホストHTML仕様 (`index.html`)

HLAアプリケーションをブラウザに描画するための親HTMLファイルです。**本ファイルには地図のビジネスロジックを一切記述せず、単なるコンテナマウントポイントとして保ちます。**

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>HLA アプリケーション</title>
    <style>
        /* 全画面マップ表示のための標準スタイル */
        html, body {
            margin: 0;
            padding: 0;
            width: 100%;
            height: 100%;
            overflow: hidden;
            background-color: #f8f9fa;
        }

        #mapcanvas {
            width: 100%;
            height: 100%;
            position: relative;
        }
    </style>
</head>
<body>
    <!-- data-src で Container.svg を指定したマップキャンバス -->
    <div id="mapcanvas" data-src="Container.svg"></div>

    <!-- 空間幾何計算ライブラリ JSTS (必須依存関係) -->
    <script src="https://unpkg.com/jsts@1.6.1/dist/jsts.min.js"></script>

    <!-- HLA コアフレームワーク (ECMAScript モジュール) -->
    <script type="module">
        import { svgMap } from 'https://cdn.jsdelivr.net/gh/svgmap/svgmapjs@latest/SVGMapLv0.1_r18module.js';
        // グローバルスコープにインスタンスを公開
        window.svgMap = svgMap;
    </script>
</body>
</html>
```

---

## 4. ディレクトリ構成規約

HLAプロジェクトは、相対パスによる疎結合性を担保するため、以下のディレクトリ構造に準拠することを推奨します。

```text
my-svgmap-project/
├── index.html                  # ホストHTML (マウントポイント)
├── Container.svg               # ルートコンテナ (マニフェスト)
└── layers/                     # レイヤー格納ディレクトリ
    ├── osm_basemap.svg         # 背景地図レイヤーSVG
    ├── osm_controller.html     # 背景地図コントローラーHTML
    ├── osm_controller.js       # 背景地図コントローラーロジック
    ├── facilities.svg          # 施設情報レイヤーSVG
    ├── facilities_ctrl.html    # 施設情報コントローラーHTML
    ├── facilities_ctrl.js      # 施設情報コントローラーロジック
    └── data/                   # レイヤー固有データ
        └── facilities.geojson  # GeoJSONデータ
```
※すべての参照（`xlink:href`、`data-controller`、`fetch`等）は**相対パス**で記述してください。
