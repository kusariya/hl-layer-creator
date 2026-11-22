# LaWA (Layers as WebApps) 開発ガイド

## 1. LaWA の概念とライフサイクル

**LaWA（Layers as WebApps: ウェブアプリケーションとしてのレイヤー）** は、SVGMapにおける動的レイヤーアーキテクチャの中核です。各レイヤーは単なる静的データではなく、独立したHTML/JavaScript実行環境（iframe内）を持つWebアプリケーションとして自律稼働します。

**【重要：データの非複製原則（No Data Copy）】**
HLAにおいて極めて重要な原則は、**「データを手元にDLするのではなく、LaWAで直接fetchし、データのコピーを作成しない」**ことです。
従来のGISのようにデータをダウンロードしてローカルファイルとして保存・再配信するのではなく、LaWA自身のスクリプトから提供元URLへ直接 `fetch()` を行い、取得したデータをメモリ上で即座にSVG DOMへ変換・描画します。これにより、データコピーを一切作成せず、常に最新かつ正確なデータを表示します。

```
[親フレームワーク (SVGMap.js)]
         |
         | ① zoomPanMap イベント発火（視域移動・ズーム時）
         v
[LaWA コントローラー (HTML / JS)]
         |
         | ② svgMap.getGeoViewBox() で現在の表示範囲を取得
         | ③ fetch() でオリジナル提供元API / GeoJSON / タイルを直接取得（コピー不作成）
         | ④ svgImage (自レイヤーのSVG DOM) に要素を追加・更新
         | ⑤ svgMap.refreshScreen() で親に再描画を通知
         v
[レイヤー固有の SVG DOM (layer.svg)]
```

---

## 2. T-LaWA と S-LaWA の2大動作モード

LaWAには、セキュリティ境界と利用可能なAPIレベルに応じて2つのモードが存在します。

| 項目 | T-LaWA (Tight-coupled LaWA) | S-LaWA (Sandbox LaWA) |
| :--- | :--- | :--- |
| **主な用途** | 自社開発・同一ドメイン内の信頼できるレイヤー | サードパーティ提供・クロスオリジンの外部レイヤー |
| **セキュリティ** | 同一オリジン（Same-Origin）前提 | サンドボックス化 iframe、クロスオリジン完全隔離 |
| **ライブラリ** | `svgMapLayerLib.js` | `svgMapSandboxLayerLib.js` |
| **通信方式** | 親フレームワークからAPIオブジェクトを直接注入 | 非同期メッセージング（`postMessage`）によるプロトコル通信 |
| **APIアクセス** | `svgImage`、`svgMap`、`svgMapGIStool` を直接利用可能 | 安全に抽象化されたRPCメッセージ経由の利用に限定 |
| **インラインスクリプト**| レイヤーSVG内の `<script>` 実行可能 | レイヤーSVG内の `<script>` は無効化（XSS防止） |

---

## 3. T-LaWA の主要 API リファレンス

T-LaWAでは、レイヤーコントローラーの `window` スコープに以下のグローバルオブジェクトが自動的に注入されます。

### 1. `window.svgImage` (レイヤーSVG DOM)
現在のレイヤーに紐付けられた SVG ファイルの `Document` オブジェクトです。
標準のDOM API（`createElementNS`, `getElementById`, `appendChild`, `removeChild`）を用いて、自レイヤーのグラフィックスを直接操作できます。

```javascript
// レイヤーSVG内に <circle> 要素を生成する例
const circle = svgImage.createElementNS("http://www.w3.org/2000/svg", "circle");
circle.setAttribute("cx", "139.767");
circle.setAttribute("cy", "-35.681");
circle.setAttribute("r", "0.005");
circle.setAttribute("fill", "#3498db");
svgImage.getElementById("featureGroup").appendChild(circle);
```

### 2. `window.svgMap` (親フレームワークAPI)
親のSVGMapエンジンを操作・同期するためのインターフェースです。
- **`svgMap.getGeoViewBox()`**:
  現在の画面表示領域の地理座標範囲を取得します。
  戻り値: `{ x: number, y: number, width: number, height: number }`
  （※経度 `x`, 緯度の負数 `y`, 経度幅 `width`, 緯度幅 `height`）
- **`svgMap.refreshScreen()`**:
  レイヤー側のDOM変更が完了したことを親フレームワークに通知し、画面を再描画させます。**DOM変更後は必ず呼び出してください。**

### 3. `window.svgMapGIStool` (GISヘルパーライブラリ)
地理空間データの変換や解析を支援するユーティリティです。
- **`svgMapGIStool.drawGeoJson(geoJsonObject, targetSvgGroup, styleConfig)`**:
  GeoJSONフィーチャーをSVGの `<path>` や `<circle>` にオンザフライ変換して指定グループに追加します。
- **`svgMapGIStool.hsv2rgb(h, s, v)`**:
  メッシュデータやヒートマップ表示用の色変換関数。

---

## 4. 実装レシピ

### レシピ1: GeoJSON 動的フィーチャーレイヤー (T-LaWA)

#### HTML: `layers/shelters_controller.html`
```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>避難所レイヤーコントローラー</title>
    <style>
        body {
            margin: 0;
            padding: 8px;
            font-family: sans-serif;
            font-size: 12px;
            background: rgba(255, 255, 255, 0.9);
            border-radius: 4px;
        }
        .control-panel { display: flex; flex-direction: column; gap: 4px; }
    </style>
    <!-- レイヤー共通ライブラリのインポート -->
    <script src="https://cdn.jsdelivr.net/gh/svgmap/svgmapjs@latest/svgMapLayerLib.js"></script>
    <script src="shelters_controller.js"></script>
</head>
<body>
    <div class="control-panel">
        <strong>避難所フィルター</strong>
        <label>
            <input type="checkbox" id="showOnlyOpen" /> 開設中のみ表示
        </label>
        <span id="featureCount">表示件数: 0件</span>
    </div>
</body>
</html>
```

#### JavaScript: `layers/shelters_controller.js`
```javascript
/**
 * 避難所データを自律的にフェッチしてSVG描画を行うLaWAコントローラー
 */

// 取得したキャッシュデータ
let cachedFeatures = [];

// ウィンドウロード時に初期化
window.addEventListener("load", initializeLayer, false);

function initializeLayer() {
    try {
        // 地図移動・拡大縮小（視域変更）イベントの登録
        window.addEventListener("zoomPanMap", handleMapUpdate, false);

        // UIイベントの登録
        const checkbox = document.getElementById("showOnlyOpen");
        if (checkbox) {
            checkbox.addEventListener("change", renderFeatures);
        }

        // 初回データ読み込みの開始
        fetchData();
    } catch (error) {
        console.error("避難所レイヤーの初期化に失敗しました:", error);
    }
}

async function fetchData() {
    try {
        // レイヤー相対パスから自律的にGeoJSONを取得
        const response = await fetch("data/shelters.geojson");
        if (!response.ok) {
            throw new Error(`HTTPエラー: ${response.status}`);
        }
        const data = await response.json();
        cachedFeatures = data.features || [];
        renderFeatures();
    } catch (error) {
        console.error("GeoJSONデータの取得に失敗しました:", error);
    }
}

function handleMapUpdate() {
    try {
        // 視域の変更に合わせて再描画（LOD制御やフィルタリングが必要な場合）
        renderFeatures();
    } catch (error) {
        console.error("マップ更新イベント処理中にエラーが発生しました:", error);
    }
}

function renderFeatures() {
    try {
        if (!window.svgMap || !window.svgImage) {
            return;
        }

        // 現在の表示範囲を取得
        const geoBox = svgMap.getGeoViewBox();
        const group = svgImage.getElementById("shelterFeatureGroup");
        if (!group) {
            return;
        }

        // 既存の要素をクリア
        while (group.firstChild) {
            group.removeChild(group.firstChild);
        }

        const showOnlyOpen = document.getElementById("showOnlyOpen")?.checked ?? false;
        let visibleCount = 0;

        // フィーチャーの描画処理
        cachedFeatures.forEach((feature) => {
            if (showOnlyOpen && !feature.properties.isOpen) {
                return;
            }

            const [lon, lat] = feature.geometry.coordinates;

            // 簡易バウンディングボックス内外判定
            // ※緯度はSVG座標系で負数となる点に注意
            const svgY = -lat;
            if (lon >= geoBox.x && lon <= geoBox.x + geoBox.width &&
                svgY >= geoBox.y && svgY <= geoBox.y + geoBox.height) {

                const circle = svgImage.createElementNS("http://www.w3.org/2000/svg", "circle");
                circle.setAttribute("cx", lon.toString());
                circle.setAttribute("cy", svgY.toString());
                circle.setAttribute("r", "0.003");
                circle.setAttribute("fill", feature.properties.isOpen ? "#2ecc71" : "#e74c3c");
                circle.setAttribute("stroke", "#ffffff");
                circle.setAttribute("stroke-width", "0.0005");
                circle.setAttribute("class", "shelter-pin clickable");

                // クリック時の詳細表示イベント
                circle.addEventListener("click", () => {
                    alert(`避難所: ${feature.properties.name}\n状態: ${feature.properties.isOpen ? "開設中" : "準備中"}`);
                });

                group.appendChild(circle);
                visibleCount++;
            }
        });

        // レイヤー固有UIの更新
        const countLabel = document.getElementById("featureCount");
        if (countLabel) {
            countLabel.textContent = `表示件数: ${visibleCount}件`;
        }

        // 親フレームワークへ再描画通知
        svgMap.refreshScreen();
    } catch (error) {
        console.error("フィーチャー描画処理で例外が発生しました:", error);
    }
}
```

---

## 5. S-LaWA のセキュア設計パターン

クロスオリジンで配布されている他組織のLaWAを組み込む場合は、`S-LaWA` を用います。

1. **レイヤーの分離**:
   レイヤーのコントローラーHTMLは、別オリジン（または `sandbox` 属性を付与したiframe）で実行されます。
2. **`svgMapSandboxLayerLib.js` の使用**:
   直接的な `window.svgImage` 参照はブロックされ、以下のメッセージングAPIを介して描画命令を送出します。
   ```javascript
   // S-LaWA でのメッセージ送出例
   svgMapSandbox.postMessage({
       action: "drawElements",
       elements: [...]
   });
   ```
これにより、サードパーティ製レイヤーに悪意のあるスクリプトが含まれていた場合でも、親アプリケーションの認証トークンやCookieを窃取されるリスク（XSS）を物理的に排除できます。

---

## 6. LaWAによる直接fetch原則と出典・クレジット表記

### A. データコピー不作成とLaWA直接fetchの原則
HLAの基本原則に基づき、データは手元にダウンロードしてローカル保存・再配置（コピー作成）するのではなく、LaWA自身が提供元の公式URLから直接 `fetch()` してオンザフライ描画します。これにより、データコピーを作成せず、常に最新データと同期します。
（※ユーザー自身が作成した独自データやローカル検証目的でファイルを読み込む場合でも、親ビューアではなくS-LaWA内のファイル入力要素 `<input type="file">` で受け取り、レイヤー内にカプセル化します）

```html
<!-- S-LaWA コントローラー内の手元ファイル読み込みUI -->
<input type="file" id="localGeoJsonInput" accept=".geojson,.json" />
```
```javascript
document.getElementById("localGeoJsonInput").addEventListener("change", async (event) => {
    try {
        const file = event.target.files[0];
        if (!file) return;
        const text = await file.text();
        const geojson = JSON.parse(text);
        renderGeoJsonToLayer(geojson);
    } catch (e) {
        console.error("手元ファイルのパースエラー:", e);
    }
});
```

### B. 出典・クレジット表記枠の設置原則
各レイヤーのデータソース（OpenStreetMap, 地理院地図, 自治体オープンデータ, CC BY等）の帰属表記は、S-LaWAコントローラー内に専用のクレジット欄（`#layer-credit`）を設けて明示します。
```html
<div id="layer-credit" style="font-size: 10px; color: #7f8c8d; margin-top: 8px;">
    出典: ○○市オープンデータ (CC BY 4.0)
</div>
```
これにより、親ビューアの画面を汚すことなく、各レイヤー自身がライセンス義務を満たすことができます。
