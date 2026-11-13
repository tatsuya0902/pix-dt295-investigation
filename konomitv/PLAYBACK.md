# 報告：録画再生の強化（追いかけ再生とシーク・速度コントロール）

KonomiTV **v0.14.1**の録画再生に、録画中番組の追いかけ再生と、シーク・再生速度のコントロールを追加する変更候補です。機能ごとに2つのパッチに分け、両方をまとめた一括パッチも用意しました。

- `chase-playback.patch`：追いかけ再生（サーバー5ファイル）
- `seek-controls.patch`：シーク・速度コントロール（クライアント5ファイル、新規1件を含む）
- `playback-combined.patch`：上記2つの一括版（10ファイル、+642/−150行）

## 追いかけ再生（chase-playback.patch）

上流のv0.14.1では、録画中の番組は録画が終わるまで再生できません。これを変更し、録画の途中から再生を始められるようにしました。

- `server/app/utils/TSInformation.py`：TSファイルの先頭と末尾のPCRタイムスタンプから再生尺（秒）を算出する`calculateTSFileDuration()`を共通化（従来の`MetadataAnalyzer`の実装を移し、両方から使う）。
- `server/app/streams/VideoStream.py`：`refreshRecordingDuration()`を追加。録画中はPCR実測で再生可能尺を更新します。再計測には最小間隔を設け、書き込み途中の末尾を避けるため再生可能尺から安全マージンを差し引きます。測定は読み取り専用で、録画ファイルは変更しません。
- `server/app/routers/VideoStreamsRouter.py`：ストリーム取得時に、録画中であれば再生可能尺を最新の状態に更新してからセッションを返します。
- `server/app/metadata/MetadataAnalyzer.py`：尺の算出を共通関数へ委譲（挙動は従来どおり）。
- `client/src/components/Videos/RecordedProgram.vue`：録画中の番組カードを開けるようにし、「録画中・追いかけ再生」バッジを表示します。録画中はEVENTプレイリストでセグメントが伸び続け、録画完了後は通常のVOD再生に移行します。

## シーク・速度コントロール（seek-controls.patch）

- `client/src/services/player/managers/SeekControlManager.ts`（新規）：シーク操作を集約。再生ボタンの両脇に−30秒／−10秒／+10秒／+30秒ボタンを配置し、連続タップは押した分だけ合算します。長押し（480ミリ秒）では8倍相当の高速シークを行い、その間はミュートします。
- `client/src/services/player/PlayerController.ts`、`KeyboardShortcutManager.ts`：SeekControlManagerの組み込みと、←→キーのシーク秒数の設定化（既定は−10秒／+30秒。Ctrl=15秒・Shift=30秒・Alt=60秒の修飾キー挙動は従来どおり）。
- `client/src/stores/SettingsStore.ts`：再生速度（0.75〜2倍）を含む新規設定3件を追加。いずれも端末ローカル設定で、サーバー同期の対象外です。再生速度は`video_playback_rate`に保存・復元し、ライブ再生では常に1倍に戻します。
- `client/src/views/Settings/General.vue`：設定画面に上記の項目を追加。

## 検証結果

- 公式v0.14.1のツリーに対する`git apply --check`：3パッチとも個別・併用でクリーンに適用できることを確認しました。
- 自環境（Ubuntu 26.04.1 LTS、PIX-DT295×2、Mirakurunバックエンド）では、クライアントをビルドして本番環境へ反映し、サーバー側も`konomitv.service`を再起動して反映済みです。再起動後の起動ログは正常で、内部APIは200を返し、エラーログは0件でした。
- iPhone／Android TVの実機でのボタン操作と、実際の録画番組での追いかけ再生の追随（録画の伸びに再生が追いつく様子）は**未検証**です。
- 高速再生は0.75〜2倍の速度変更のみで、16倍速のような早送り再生は含みません。

## 適用方法

未適用のv0.14.1ツリーで使用してください。

```bash
# リポジトリのルートではなく、KonomiTVのソースツリーで実行
git apply --check /path/to/chase-playback.patch
git apply /path/to/chase-playback.patch

git apply --check /path/to/seek-controls.patch
git apply /path/to/seek-controls.patch

# 両方をまとめて適用する場合はこちら
git apply --check /path/to/playback-combined.patch
git apply /path/to/playback-combined.patch
```

録画再生をレコーダー並みに扱うための変更候補として、ご確認をお願いします。
