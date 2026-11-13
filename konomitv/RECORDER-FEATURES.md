# KonomiTV レコーダー系表示機能パッチ (v0.14.1 向け)

市販レコーダーに近い視聴体験のための、表示・移動まわりの機能追加です。
いずれも KonomiTV v0.14.1 に対する差分パッチで、自環境 (本番) では 2026-11-13 に反映済みです。

## 含まれる機能とパッチ

| 機能 | パッチ | 内容 |
|---|---|---|
| 前の話・次の話ボタン | `episode-nav.patch` | 単体取得 API に `next_episode_id` / `previous_episode_id` を追加し、視聴画面の番組情報パネルに「前の話」「次の話」ボタンを表示する。シリーズ ID が全録画で空の環境でも動くよう、タイトルから話数表記と括弧注記を除いた番組名の一致でも判定する (前後とも録画として存在する回にだけ移動する。欠けた回には飛ばない) |
| 場面一覧 | `chapters.patch` | 番組情報パネルに「場面一覧」ボタンを追加。シークバー用タイル画像を約12場面に間引いてグリッド表示し、クリックでその場面へシークするダイアログ (ChapterListDialog) |
| 盛り上がり表示とダイジェスト再生 | `excitement.patch` / `excitement-fix.patch` / `digest-button-panel.patch` | 過去ログコメントを5秒ビンで集計し、シークバーの上に密度グラフを表示 (クリックでシーク)。密度が高い区間だけを連続再生するダイジェスト再生。`excitement-fix` は帯の配置をシークバー内側へ直す修正、`digest-button-panel` はダイジェストの ON/OFF ボタンをプレイヤー内から番組情報パネルへ移す変更 (プレイヤー内の小さいボタンは押しにくいため) |
| 追いかけ再生のカード修正 | `chase-card-click-fix.patch` | 録画中カード全体に掛かる上流 CSS の `pointer-events: none` を録画中カードだけ外し、追いかけ再生のためにカードを開けるようにする修正 (解析失敗カードは従来通り無効)。`chase-playback.patch` の利用者向け |

## 適用方法

素の v0.14.1 に対して、次の順序で適用してください (後段のパッチは前段のパッチ適用済みの状態を前提にしています)。

```bash
# 先に公開済みの再生系パッチを適用する (excitement.patch の PlayerController 側の文脈が seek-controls に依存します)
git apply seek-controls.patch
git apply chase-playback.patch

# 本セット
git apply chase-card-click-fix.patch
git apply episode-nav.patch
git apply chapters.patch
git apply excitement.patch
git apply excitement-fix.patch
git apply digest-button-panel.patch
```

- `episode-nav.patch` はサーバー側 (schemas / VideosRouter) の変更を含みます。サーバーを再起動してください。
- それ以外はクライアントのみの変更です。再ビルド (`yarn build` 相当) で反映されます。
- ハッシュの照合には `SHA256SUMS-recorder` を使ってください。

## 検証状況 (2026-11-13 時点、正直な現状)

- 全パッチは素の v0.14.1 への上記順序での `git apply` を確認済みです。
- 場面一覧: 利用者が実機でボタンの表示と場面へのジャンプを確認済み。
- 盛り上がり表示: コメント4,186件の大相撲録画でグラフ表示を実機スクリーンショットで確認済み。表示位置の重なりは `excitement-fix` で修正済み。
- ダイジェスト再生: 区間スキップの連続再生そのものは、利用者による明示的な動作確認はまだありません。
- 前の話・次の話: サーバー側の判定ロジックは実データで検証済み (前後の回・解析失敗回のスキップ・別番組への誤判定なし)。ボタンのクリックは利用者未確認です。
- 追いかけ再生のカード修正: 修正後、利用者が実機で追いかけ再生できることを確認済み。
