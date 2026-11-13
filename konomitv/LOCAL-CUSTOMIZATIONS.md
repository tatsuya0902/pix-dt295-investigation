# ローカルカスタマイズ記録

このドキュメントは、この KonomiTV (v0.14.1) に対して行ったローカル改造の一覧と、
再適用・ロールバック手順をまとめたものです。

- 2026-11-13 から、このディレクトリは Git で管理されています。
- ベースラインコミット `f06482e` は「Codex による既存改造を含む現行状態」です
  (改造前の素の v0.14.1 ではありません)。
- 以後の改造は機能単位でコミットされています。ロールバックは `git revert <commit>` で行い、
  クライアントは再ビルド、サーバーは再起動で反映します。

## 既存の改造 (ベースラインに含まれる、Codex による実装)

- EPGStation 連携: 番組表からの録画予約、KonomiTV からの予約・削除操作
- Jellyfin 連携: KonomiTV での削除を Jellyfin に反映 (削除同期)
- 録画の削除保護 (protection) と一覧の保護バッジ表示
- iPhone 向け疑似全画面・横画面対応
- 連携設定は `customizations.local.json` (Git 管理外) で管理:
  `server/app/utils/RecorderDeleteSync.py` と `server/app/routers/RecorderRouter.py` が参照する

これらには今回の改造では一切手を付けていません (競合時の最小修正も発生していません)。

## 2026-11-13 再生機能強化 (Muse による実装)

家庭用レコーダーに近い操作性と、録画中番組の追いかけ再生を追加しました。
対応バージョン: KonomiTV 0.14.1。EPGStation への依存はなく、録画ファイルと
KonomiTV の DB 状態 (status='Recording') のみを利用します。

### 1. スキップ・高速シーク・再生速度 (クライアントのみ)

コミット: `feat(client): recorder-style seek controls for recorded playback`

- 新規 `client/src/services/player/managers/SeekControlManager.ts` (Video モード専用)
  - コントロールバーに [30秒戻し][10秒戻し][再生][10秒送り][30秒送り] を配置
    (内側のペアは設定秒数、初期値 10秒戻し / 30秒送り)
  - 連続タップは 220ms 以内で合算して 1 回のシークに集約。範囲外には出ない
  - 長押し (480ms 以上) で 8 倍相当の高速シーク (250ms ごとに 2 秒移動、音声ミュート、離すと停止)
  - 再生速度 (0.75〜2 倍) の変更を検知して設定に保存し、次回の録画再生で復元
    (プレイヤー再起動時の復元は従来機能。ライブ視聴は常に 1 倍開始)
  - ボタンはフォーカス可能で Enter/Space でも操作可能 (Android TV リモコン想定)
- `KeyboardShortcutManager.ts`: ビデオ視聴時の ← / → を設定秒数 (初期値 10秒 / 30秒) に変更
  (Ctrl=15秒、Shift=30秒、Alt=60秒 は従来どおり)
- `SettingsStore.ts` + 設定画面「一般」に 3 つの設定を追加 (端末ローカル保存、サーバー同期対象外)
  - サーバー同期対象にしなかった理由: サーバー側 `ClientSettings` (pydantic) が
    未知キーを検証するため、同期に混ぜると設定同期 API が壊れる恐れがあるため
  - `video_seek_backward_seconds` (初期値 10) / `video_seek_forward_seconds` (初期値 30) /
    `video_playback_rate` (初期値 1)

### 2. 追いかけ再生 (サーバー + クライアント)

コミット: `feat: chase playback for in-progress recordings`

- `server/app/utils/TSInformation.py`
  - `calculateTSFileDuration()` を新設。TS の先頭・末尾の PCR タイムスタンプから
    再生可能尺を実測する (ファイルサイズからの単純比例では推定しない)。
    ゼロ埋め (スパース) 領域の検出込み
  - `MetadataAnalyzer.__calculateTSFileDuration()` はこの共通実装への委譲に変更 (挙動不変を実ファイルで確認済み)
- `server/app/streams/VideoStream.py`
  - セッションごとに録画状態と再生可能尺を保持し、`refreshRecordingDuration()` で
    最新の DB 状態と PCR 実測 (10 秒に 1 回まで間引き、末尾 3 秒の安全マージン) により更新
  - 仮想セグメントは再生可能尺まで構築・延長 (構築済みセグメントは変更しない)
  - 録画中は `#EXT-X-PLAYLIST-TYPE:EVENT` + ENDLIST なし、録画完了で VOD + ENDLIST に切替
- `server/app/routers/VideoStreamsRouter.py`
  - プレイリスト取得時に、DB から再取得した最新の番組情報で `refreshRecordingDuration()` を呼ぶ
- `client/src/components/Videos/RecordedProgram.vue`
  - 録画中 (status='Recording') の番組カードを開けるようにし、バッジを「録画中・追いかけ再生」に変更

設計上の制約 (厳守したもの):

- 録画ファイルは読み取り専用で扱い、変更・移動・ロックは一切しない。録画や EPGStation には干渉しない
- 追加のチューナーは使用しない。TrueNAS への書き込みを妨げない
- 再生可能範囲は TS の PCR タイムスタンプ実測のみから決める
- 新規 API は追加していない (既存のプレイリスト API の応答形式の拡張のみ)。任意パスを受け取る API もなし

### 検証記録

- クライアント: `vue-tsc --noEmit` 成功、`vite build` 成功 (本番 dist に反映済み)
- 追いかけ再生: 伸長する TS ファイルを模した単体テストで確認 (スクリプトは `docs/verify_chase_playback.py`)
  - PCR 実測が 45.5 秒 → 81.8 秒とファイルの伸長に追随
  - プレイリストのセグメントが 5 → 14 に延長、EVENT / ENDLIST なし
  - 録画完了遷移で VOD + ENDLIST、完了済み録画のプレイリストは従来どおり不変
- 未検証 (実機・実環境が必要)
  - iPhone / iPad / Android TV 実機での操作感 (コード対応のみ)
  - 本番サーバー再起動後の実録画での追いかけ再生 (再起動はユーザー確認待ち)
  - hls.js が EVENT プレイリストを再取得して先端に追随する挙動は hls.js の標準動作に依存

### 既知の制限

- 16 倍相当などの超高速再生は未実装。長押しの 8 倍シークで代替 (安定性優先)
- 録画中番組の DB 上の duration は登録時点のスナップショットのまま (一覧表示の尺は録画完了まで古い)
- 追いかけ再生中のシークバー全長はプレイリストの EVENT 総尺に追随する

### 本家アップデート時の再適用メモ

競合しやすい箇所: `PlayerController.ts` の PlayerManager 登録配列、
`KeyboardShortcutManager.ts` の ← / → 定義、`SettingsStore.ts` の設定キー追加、
`VideoStream.ts` の `getVirtualPlaylist()`、`MetadataAnalyzer` の尺計算、
`VideosRouter.ts` の `AttachProtectionStates()` と一覧・検索 API での呼び出し。
いずれも「ローカル改造」とコメントを付けた箇所が目印。

## 2026-11-13 ログアウト中も録画の保護バッジを表示 (Muse による実装)

録画一覧 API (`GET /api/videos`) は未ログインでも 200 を返す一方、保護バッジの
取得元 `GET /api/recorder/protection` はログイン必須 (401) のため、ログアウト中は
録画カードの「保護中」バッジ (🔒) が表示されなかった。これを表示できるようにした。

方式: **未認証の取得 API を新設せず、既に公開されている一覧・検索 API の応答に
保護状態 (`is_protected`) を含める**。新しい未認証エンドポイントを作ると攻撃面が
増えるため避けた。応答に含まれるのは録画 ID ごとの真偽値のみで、パスや管理元 ID は
従来どおり含まれない。保護状態の取得自体は、ログイン時のバッジ取得と同じ
`RecorderDeleteSync.protectionStates()` をページ単位で 1 回だけ呼ぶ。

変更ファイル:

- `server/app/schemas.py`: `RecordedProgram.is_protected: bool | None = None` を追加
  (None は状態不明。詳細 API など付与対象外の応答では None のまま)
- `server/app/routers/VideosRouter.py`: `AttachProtectionStates()` を追加し、
  一覧 API (`VideosAPI`) と検索 API (`VideosSearchAPI`) の応答直前に呼ぶ。
  削除連動が無効な場合・取得に失敗した場合は is_protected を None のままにし、
  一覧表示自体は継続する (保護状態のために一覧を 500 にしない)
- `client/src/services/Videos.ts`: `IRecordedProgram.is_protected` を追加
- `client/src/components/Videos/RecordedProgram.vue`: `isProtected` の判定を、
  バッジマップ (ログイン時のみ取得) に値があればそれを優先し、未取得・状態不明の
  ときは一覧 API の `is_protected` にフォールバックするよう変更。
  ログイン時のみバッジ取得する既存ガードは変更していない

検証: サーバーは py_compile と、schemas.RecordedProgram への属性代入・
model_dump への反映を venv 上で確認。クライアントは型検査・ビルド成功し
本番 dist に反映済み。サーバー側の反映には konomitv.service の再起動が必要
(ユーザー確認待ち)。再起動後は未ログインで `GET /api/videos` を取得し、
保護済み録画の `is_protected` が true になっていることで確認できる。

## ビルド時の注意（2026-11-13 追記）

クライアントのビルドは必ず yarn/npm のスクリプト経由で行うこと。`node_modules/.bin/vite` を直接叩くと `npm_package_version` が未定義になり、`import.meta.env.KONOMITV_VERSION`（vite.config.mts の define）が undefined になる。その状態でビルドされたバンドルは `Utils.version` が undefined となり、VersionStore の `is_client_develop_version` が例外を投げて Navigation（PCのサイドバー＋スマホの下部タブ）が描画されなくなる（実際に発生した）。yarn が無い環境では `npm_package_version=$(node -p "require('./package.json').version")` を付けて実行すれば同等になる。

## シリーズの前後の話への移動（2026-11-13 追記）

- 録画番組の単体取得 API（GET /api/videos/{video_id}）の応答に `next_episode_id` / `previous_episode_id` を追加した。視聴画面の番組情報パネルに「前の話」「次の話」ボタンが表示され、そのまま移動できる。
- 判定は放送開始時刻の前後で最も近い再生可能な（Recorded / Recording）録画。series_id がある場合はそれを使い、ない場合はタイトルから話数表記（#6・第3話など）と括弧注記（【サブタイトル】・[字] など）を除いた番組名の一致で代替判定する（この環境では series_id がほぼ付与されないため）。
- 再放送など同一タイトルの別録画も時刻順の前後として扱う割り切りの仕様。判定に失敗しても番組情報は通常どおり返る（ID は None のまま）。
- 視聴進捗バーは上流実装済み（カードのサムネイル下部）。保護の付け外しも既存実装（カードメニュー、管理者のみ、EPGStation へ書き戻し）があるため追加実装はしていない。
- CM 区間（cm_sections）は上流で検出が未実装（TODO スタブ＋チャプターファイル方式）のため常に空。CM スキップは検出器（comskip / FFmpeg ベース）の組み込みが前提で、別途検討事項。

## 2026-11-13 追加分 (市販レコーダー相当の機能)

### 場面一覧 (サムネイル付きチャプター) — a24afb1
- 視聴画面の番組情報パネルに「場面一覧」ボタンを追加 (ChapterListDialog)。シークバー用タイル画像 (thumbnail_info.tile) を約12場面に間引いてグリッド表示し、クリックで SeekRequest イベント経由でその場面へシークする。サーバー側の変更なし。

### 盛り上がり表示とダイジェスト再生 — 94c5cde
- ExcitementManager (PlayerManager) を新設。録画の過去ログコメント (CommentReceived の playback_position) を5秒ビンで集計・平滑化・正規化し、シークバーの上に密度グラフ (canvas) を描く。グラフのクリックでシーク。
- 密度が最大値の40%以上の区間 (隙間10秒以下は連結、15秒未満は除外、全体の70%超は無効) を盛り上がり区間とし、帯の右端の「ダイジェスト」ボタンでその区間だけを連続再生する (区間開始の3秒手前から再生)。コメントが50件未満の録画では表示しない。

### ゴミ箱 (削除の猶予) — b7f5ba2 (サーバー) / 6423717 (クライアント)
- server/app/utils/RecordingTrash.py が状態を server/data/recording-trash.json に保持する。登録時はファイル・DB を一切変更せず、一覧 API (VideosAPI) と検索 API から除外するだけ。
- API は RecorderRouter 配下: GET /api/recorder/trash (ログイン必須)、POST /api/recorder/trash/{video_id} と POST .../restore (管理者のみ)。保持期間は7日で、app.py の repeat_every (1時間間隔) が期限切れを既存の削除経路 (plan→sign→delete) で完全削除する。失敗時は1日先送り。
- クライアント: カードメニューに「ゴミ箱に移動する」(管理者のみ、保護中は無効)、録画番組一覧に「ゴミ箱を開く」リンク、/videos/trash ビュー (復元・完全削除)。完全削除は既存の削除 API を使い、サーバー側でゴミ箱登録も取り消される。
- 注意: ゴミ箱内でも EPGStation / Jellyfin 側には通常どおり見える (実削除が猶予される設計のため)。EPGStation 側で先に削除された場合は一覧取得時にエントリを掃除する。

### ダイジェストボタンのパネル移設 — 7ce85fa
- プレイヤー内の帯右端の小さいボタンは押しにくいため廃止。番組情報パネルに「ダイジェスト再生」ボタンを配置し、PlayerStore のイベント (DigestToggleRequest / DigestStateChanged、型を PlayerStore.ts に追加) で ExcitementManager と連携する。区間がある録画でのみボタンが表示され、ON の間はピンク背景＋「（ON）」表記。

### 次話の自動再生 / おやすみタイマー / ハイライト横断再生 — 99bb211
- AutoPlayNextManager: 次の話 (next_episode_id) がある録画の再生終了時に、右下へカウントダウンカード (10秒、次の話のタイトル付き) を表示し、次の話の視聴画面へ自動遷移する。キャンセル・再生再開で中止。おやすみタイマーの「番組の終わりで停止」が有効な場合は自動再生しない。
- SleepTimerManager: コントロール列にタイマーボタン (⏰) を追加。タップで オフ→15分→30分→60分→(録画のみ) 番組の終わりで停止→オフ と切替。残り時間はボタン上に表示。状態の共有は services/player/SleepTimerState.ts (モジュール変数) 経由で AutoPlayNextManager が参照する。ライブ視聴でも時間指定のみ利用可能。
- ハイライト横断再生: services/HighlightReel.ts が再生キューを sessionStorage に保持。HighlightReelManager が視聴画面でキューと現在の録画の一致を確認し、区間開始へシーク→終了で次区間へ (別録画なら router 遷移)。左上に「ハイライト再生中 n/m」バッジ (×で中断)。キューと一致しない録画を開いたらキューを破棄する。

### 盛り上がりチャプター (場面一覧の「見どころ」) — 305a144
- services/CommentHighlights.ts に密度計算 (5秒ビン・平滑化・正規化) と区間抽出を共通化 (ExcitementManager と同じルール: 40%以上、隙間2ビンまで連結、3ビン未満除外、カバー率70%超は無効、50件未満は対象外)。
- ChapterListDialog の上部に「見どころ」リスト (開始〜終了時刻・コメント件数) を追加し、クリックでその場面へシークする。

### コメント検索 / 次回を予約 / 未視聴のみ / 今日のハイライト — 228f56d
- コメント検索: パネルの「コメント検索」ボタンから CommentSearchDialog を開き、過去ログをキーワード検索 (最大100件表示)、ヒットをクリックでその瞬間へシーク。
- 次回を予約: パネルの「次回を予約」ボタン。series_title (なければ title) で番組表をタイトル検索し、同チャンネルの直近の未来放送をデフォルト録画設定で予約する。
- 未視聴のみ: 録画一覧のチップで切替。ON 時は全ページ取得し、視聴履歴がない or 再生位置90%未満の録画だけを表示する (サーバー側ページングとは併用しない)。
- 今日のハイライト: 録画一覧のチップから HighlightsDialog を開き、今日の録画 (最大12件) から盛り上がり上位3区間ずつを抽出して一覧表示。「ハイライトを再生」でキューを保存し、最初の録画から横断再生する (区間開始の3秒手前から)。

### 視聴状態のサーバー一元化 (未視聴バッジ・未視聴フィルタ) — 0213b5d (サーバー) / 6b87686 (クライアント)
- ユーザー要望: 新着・未再生の判定を端末・ログイン状態によらず世帯で共通にしたい (市販レコーダーと同様)。従来の watched_history は同期対象設定だが上限50件・ログアウト端末は対象外だった。
- サーバー: app/utils/WatchedState.py が data/watched-state.json に録画ごとの最大到達位置を保持。POST /api/videos/{id}/watched (認証なし、テレビ等のログアウト端末からも報告可) で報告を受け、一覧・検索 API の応答に watched_ratio (最大到達位置/尺) を付与する (AttachWatchedStates、is_protected と同じ方式)。2026-11-13 19:25 の再起動で本番反映。実測: 未認証 POST→204、一覧に ratio が反映されることを確認 (テストデータは掃除済み)。
- クライアント: WatchedReportManager が PlaybackPositionChanged を購読し、進捗をサーバーへ自動報告 (10秒進むごとに最大5秒間隔、別途20秒間隔、破棄時にも最終位置を報告)。録画カードのサムネ左上に「未視聴」バッジ (watched_ratio < 0.05 の Recorded のみ)。未視聴フィルタの判定も端末の視聴履歴から watched_ratio (< 0.9) に切替。端末ローカルの視聴履歴 (続きから再生・進捗バー) は従来通り併存する。
- 注意: 状態は世帯で1つ (誰かが見れば全端末で視聴済み扱い)。録画削除時に watched-state.json の該当エントリは自動では消えない (forget() は用意済みだが未接続。一覧に存在しない ID は返されないため実害は小さい)。

### 実機フィードバックによる調整 (2026-11-13 夕方)
- パネルボタンの間隔 — f903aa0: 番組情報パネルのボタン同士の横間隔が狭かったため、.program-info__button に margin-right: 10px を追加した。
- シーク外側ボタン ±60秒 — e9bba71: 内側の設定式ボタン (初期値 +30秒) と固定の +30秒が重なっていた問題への対処。SeekControlManager の外側ペアを固定30秒から固定60秒へ変更し、並びは [-60][-10 (設定式)][再生][+30 (設定式)][+60] になった。上記「次話の自動再生 / おやすみタイマー」より前のシーク節 (1. スキップ・高速シーク・再生速度) の「30秒」表記はこの変更で読み替えること。
- おやすみタイマーのパネル移設 — 31625dc: 「再生画面に詰め込みすぎ」との指摘で、録画再生時のタイマー操作をプレイヤー内ボタンから番組情報パネルのボタンへ移した (99bb211 節の記述を置き換える)。SleepTimerManager は Video モードではボタンを持たず、PlayerStore イベント (SleepTimerCycleRequest / SleepTimerStateChanged、型は PlayerStore.ts) でパネルと連携する。ライブ視聴では従来通りプレイヤー内ボタンを使う。残り時間はパネルのボタンラベルに表示される。
