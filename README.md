# HLA / SVGMap レイヤー開発スキル

`skills/hl-layer-creator/` は、SVGMap / HLA の地図アプリやレイヤーを設計・実装・レビューするためのエージェントスキルです。公式チュートリアルを学習経路として使い、要件に合わせて静的SVG、Container.svg、LaWA、Webサービス連携、既存レイヤーなどの構成を選びます。

データの取得・保存方法や LaWA の利用は一律に決めず、データの所在、利用条件、可用性、CORS、更新頻度などを踏まえて選択します。詳細な開発手順と設計方針は [マスタースキル](skills/hl-layer-creator/SKILL.md) を参照してください。

## 内容

- `SKILL.md`: スキルの使い方、進め方、専門スキルとの連携
- `references/tutorial_workflow.md`: 公式チュートリアルの段階と選び方
- `references/architecture.md`: HLAの概念と構成の選択
- `references/specification.md`: Container.svg、レイヤーSVG、ホストHTMLの仕様メモ
- `references/lawa_guide.md`: LaWAの動作モードと開発手順
- `references/data_source_architecture.md`: データの所在・利用条件に応じた供給方式
- `references/harness_and_anti_patterns.md`: アンチパターンとレビュー項目

必要に応じて、接続先探し、利用条件確認、データ調査、設計、実装、レビューを担当する `hla-*` 専門スキルと併用できます。全てを決まった順番で使う必要はありません。

## 導入

新規に使う場合はこのリポジトリをクローンし、既存プロジェクトで更新を追従したい場合はサブモジュールとして追加できます。どちらの場合も、利用するエージェントが認識するスキルディレクトリに `skills/hl-layer-creator/` を配置するか、その場所をエージェントに指定してください。

```bash
# 新規に使う場合
git clone https://github.com/<ユーザー名>/hl-layer-creator.git

# 既存プロジェクトのルートで実行
git submodule add https://github.com/<ユーザー名>/hl-layer-creator.git .agents/hl-layer-creator

# クローン後にサブモジュールも取得
git submodule update --init --recursive
```

利用するエージェントが認識するように、スキルの場所 `.agents/hl-layer-creator/skills/hl-layer-creator/` を登録または指定してください。登録方法はエージェントごとの仕様に従います。更新を取得するときは `git submodule update --remote .agents/hl-layer-creator` を実行し、更新されたサブモジュールのコミットを親プロジェクトでコミットします。

## 作成したレイヤーの動作確認

作成したSVGMap一式をブラウザーで確認するには、まずプロジェクトをローカルHTTPサーバーで配信する方法を案内できます。配信ルートは `Container.svg` やホストHTMLから参照する相対パスが解決できるディレクトリにします。継続的な共有や公開確認には、GitHub Pages などの静的ホスティングも利用できます。

外部データ/APIを使うレイヤーでは、接続先のCORS設定、認証要件、利用規約も確認してください。ブラウザーのローカルファイル直接オープンでは、HTTP通信や相対リソースの挙動を正しく確認できない場合があります。

## 公式リソース・サンプル

実装時は、チュートリアルの例を現行仕様とみなさず、公式ドキュメントやソースでAPI・属性を確認してください。

- [SVGMap 公式チュートリアル](https://svgmap.github.io/tutorials/): 基本表示、レイヤー、タイル、WebApp Layer などの学習用サンプル
- [svgmap/tutorials](https://github.com/svgmap/tutorials): チュートリアルのソース
- [svgmap/svgmapAppLayers](https://github.com/svgmap/svgmapAppLayers): LaWA / WebApp Layer の実装例
- [svgmap/svgmapjs](https://github.com/svgmap/svgmapjs): SVGMap.js と関連ライブラリ
- [SVGMap 公式解説書](https://www.svgmap.org/wiki/index.php?title=%E8%A7%A3%E8%AA%AC%E6%9B%B8): API・仕様の確認先

## 注意事項

- チュートリアルや参照資料は開発の手引きです。実装前に、対象機能の現行の公式資料・実装を確認してください。
- データやサービスの利用条件、出典表示、キャッシュ、再配布条件は提供元の規約で確認してください。取得方法だけから利用可否を判断しないでください。
- 本スキルは技術的な開発支援を目的とし、法的な判断を代替するものではありません。
- 本スキルの利用または利用不能によって生じた損害について、作者は責任を負いません。詳細は [LICENSE](LICENSE) を確認してください。
