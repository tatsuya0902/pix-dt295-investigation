# 報告：EPGStation/Jellyfin 連携（削除委譲・保護状態の表示と操作）

KonomiTV **v0.14.1**に、録画の管理元である EPGStation への削除委譲、EPGStation 側の削除保護状態の表示・操作、および Jellyfin への削除連動を追加する変更候補です。

- `epgstation-integration.patch`：連携一式（9ファイル、+1084/−10行。サーバー6ファイル＋クライアント3ファイル、新規3件を含む）

上流 v0.14.1 の録画削除は、KonomiTV が録画ファイルと DB を直接削除する実装です。EPGStation で録画を管理している構成では、KonomiTV から直接消すと EPGStation 側の録画情報と実ファイルの対応が崩れます。この変更では、削除を EPGStation の正式 API へ委譲し、KonomiTV と Jellyfin の一覧をそれに追随させます。**設定ファイルがない、または無効な環境では連携は完全に無効化され、上流の挙動のまま動作します。**

## 機能

### 削除委譲と確認フロー

- `GET /api/videos/{video_id}/delete-preview`（管理者のみ）：削除前に、対象録画が EPGStation のどの録画に対応するか（ファイルパスの正規化・完全一致で一意に照合）、保護・録画・エンコードの状態、全対象ファイル（サイズ・inode・更新時刻のスナップショット）を確認し、有効期限5分の署名付き確認トークン（HMAC-SHA256、鍵は状態保存先に自動生成される `confirmation.key`）を返します。
- 確認ダイアログは、管理元（EPGStation）の録画 ID と全対象ファイルを表示します。
- `DELETE /api/videos/{video_id}` は、確認トークン（`X-Recording-Confirmation` ヘッダー）がない場合は連携有効時に処理を進めません。削除直前にも管理元の状態と実ファイルを再照合し、食い違えば中止します。実際の削除は EPGStation の `DELETE /recorded/{id}` に委譲し、削除が管理元で確認できた後に KonomiTV の DB を同期します。応答ヘッダー `X-Recording-Sync` で同期状態を返します。
- 録画中・保護中・エンコード中の録画は削除できません。パス検証（保存先外への脱出、シンボリックリンクを含むパスの拒否）も常に行います。

### 保護状態の表示（ログイン中・ログアウト中）

- 録画カードに、EPGStation 側で保護されている録画であることを示す「保護中」バッジを表示します。
- ログイン中は `GET /api/recorder/protection`（表示中ページ分をまとめて取得、最大60件）で状態を取得します。
- 一覧・検索 API は未ログインでも閲覧できるため、その応答に `is_protected` を含めることで、**ログアウト中でもログイン時と同じ見た目でバッジを表示します**（保護状態取得専用の未認証 API は新設していません）。状態が取得できない・対応が曖昧な録画は「不明」(None) として扱い、未保護と断定しません。

### 保護の付け外し（トグル）

- 録画カードのメニューから、管理者が保護の設定・解除を行えます。実体は EPGStation の `PUT /recorded/{id}/protect` / `unprotect` への委譲で、操作後は管理元を読み直して結果を確認してから表示に反映します。録画ファイル自体には触れません。

### Jellyfin への削除連動

- 削除後、Jellyfin の `POST /Library/Refresh` を要求し、対象アイテムが一覧から消えたことを API で確認するまで同期ジョブを保留します。状態は状態保存先（既定は `server/data/recorder-delete-sync/`、パーミッション 0700）に JSON で永続化され、サーバー再起動をまたいで指数バックオフで再試行します。認証情報はジョブ記録に保存しません。
- EPGStation 側で直接削除された場合も、録画フォルダの監視（既存の `RecordedScanTask`）が検出したファイル消失をフックし、Jellyfin 側の更新ジョブとして取り込みます。

### 同期状態の表示とリトライ

- `GET /api/recorder/delete-sync`（管理者のみ）：保留中・失敗を含む同期ジョブの一覧を返します（認証情報・確認トークンは含みません）。
- `POST /api/recorder/delete-sync/retry`（管理者のみ）：保留中ジョブの再試行を即時化します。EPGStation への DELETE 自体は再送しません（削除の再実行は必ず利用者の新しい確認を経ます）。

## ファイル内訳

| ファイル | 内容 |
|---|---|
| `server/app/utils/RecorderDeleteSync.py`（新規） | 連携の中核。設定読み込み、EPGStation 照合・削除委譲・保護取得/設定、確認トークンの署名/検証、同期ジョブの永続化と再試行、Jellyfin 連動 |
| `server/app/routers/RecorderRouter.py`（新規） | `/api/recorder/protection`、`/api/recorder/delete-sync`、`/api/recorder/delete-sync/retry` と復旧ワーカーの起動/停止 |
| `server/app/routers/VideosRouter.py` | `GET/PUT /{video_id}/protection`、`GET /{video_id}/delete-preview`、DELETE の委譲分岐、一覧・検索応答への `is_protected` 付与 |
| `server/app/schemas.py` | 録画番組スキーマに `is_protected` を追加 |
| `server/app/app.py` | RecorderRouter の登録 |
| `server/app/metadata/RecordedScanTask.py` | ファイル消失検出時に Jellyfin 更新を要求するフック |
| `client/src/services/RecordingProtectionBadges.ts`（新規） | 一覧表示用バッジ状態のバッチ取得とキャッシュ |
| `client/src/services/Videos.ts` | 保護取得/設定、削除プレビュー、確認トークン付き削除の API クライアント |
| `client/src/components/Videos/RecordedProgram.vue` | 保護中バッジ、保護トグル（管理者のみ）、管理元情報を表示する削除確認ダイアログ |

## 前提と設定

- EPGStation の API に、KonomiTV サーバーから到達できること。EPGStation の API は認証を前提としていないため、同一ホストや信頼できる LAN 内からの利用を想定しています（EPGStation の API を外部に公開しないでください）。
- KonomiTV が見る録画フォルダと EPGStation が管理する録画フォルダが、同じ実体を指していること（NFS マウント等）。両者のルートの対応を設定で与えます。
- Jellyfin 連携は任意です。未設定の場合は Jellyfin 側の処理をスキップします。

設定は、KonomiTV のルート直下の `customizations.local.json` に置きます（環境変数 `KONOMITV_RECORDER_CONFIG` で別パスを指定可能）。ファイルがないか `enabled` が false の場合は連携は無効です。

```json
{
    "enabled": true,
    "epgstation_url": "http://127.0.0.1:8888",
    "recording_root": "/path/to/recordings",
    "epgstation_recording_root": "/path/to/recordings",
    "jellyfin_url": "http://127.0.0.1:8096",
    "jellyfin_access_file": "/path/to/jellyfin-access.json",
    "jellyfin_recording_root": "/path/to/recordings"
}
```

- `epgstation_url`：EPGStation の API ベース URL
- `recording_root`：KonomiTV から見た録画フォルダのルート
- `epgstation_recording_root`：EPGStation が返すファイル名の基準ルート（絶対パスを返す構成の場合の変換用）
- `jellyfin_url` / `jellyfin_recording_root`：Jellyfin のベース URL と、Jellyfin から見た録画フォルダのルート（省略時は `recording_root` と同一とみなす）
- `jellyfin_access_file`：Jellyfin の認証情報を分離した JSON ファイルへのパス。中身は `{"token": "<APIトークン>", "admin_id": "<管理者ユーザーID>"}` の形式で、本体設定には秘密情報を書かない設計です
- 状態保存先は既定で `server/data/recorder-delete-sync/` です。初回有効化時に確認トークン署名用の `confirmation.key` が自動生成されます

## 制限・未検証事項

- 削除・保護の操作系 API はいずれも管理者アカウントが必要です（保護バッジの閲覧のみ未ログインでも可能です）。
- 録画の対応付けはファイルパスの完全一致が前提です。EPGStation 側でファイル名や保存先の規約が変わると、対応付けできない録画は「管理外」として削除・保護操作の対象外になります（安全側に倒す設計です）。
- 自環境（Ubuntu 26.04.1 LTS、EPGStation + Jellyfin 併用）では本番稼働中ですが、公開用パッチとしての検証は、公式 v0.14.1 のツリーへの `git apply --check` がクリーンなことと、サーバー側ファイルの構文チェック（py_compile）までです。新規環境でのセットアップからの通し検証は未実施です。
- クライアント側は、自環境でビルド・稼働しているソースと同一内容であることを確認しています（検証用ツリーへの型チェック単体実行は未実施）。

## 適用方法

未適用のv0.14.1ツリーで使用してください。

```bash
# リポジトリのルートではなく、KonomiTVのソースツリーで実行
git apply --check /path/to/epgstation-integration.patch
git apply /path/to/epgstation-integration.patch
```

適用後、上記の設定ファイルを配置し、KonomiTV サーバーを再起動すると連携が有効になります。

EPGStation を管理元とする運用向けの変更候補として、ご確認をお願いします。
