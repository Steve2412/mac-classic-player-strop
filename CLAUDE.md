# Mac Classic Player

macOS向けのメディアプレイヤー。Tauri v2 + React + mpv で構成。
[piro0919/mac-classic-player](https://github.com/piro0919/mac-classic-player)（MIT）のフォーク。

## 技術スタック

- **バックエンド**: Rust (Tauri v2) — `src-tauri/src/lib.rs` に主要ロジック
- **フロントエンド**: React 19 + TypeScript + Vite — `renderer/` 配下
- **パッケージマネージャ**: npm
- **リンター**: ESLint, Prettier, Stylelint, commitlint, secretlint
- **Git hooks**: lefthook

## ビルド（重要）

このフォークでは自動更新（tauri-plugin-updater）を外しているので、署名鍵（TAURI_SIGNING_PRIVATE_KEY）は不要。
ad-hoc署名だけ付けてビルドする:

```bash
APPLE_SIGNING_IDENTITY="-" npm run build
```

再生エンジンは mpv（libmpv）。実行には `brew install mpv` が必要。
`src-tauri/src/mpv/ffi.rs` が起動時に libmpv.2.dylib を dlopen で探す（/opt/homebrew/lib など）。

### APPLE_SIGNING_IDENTITY="-" を忘れると何が起きるか

- ダウンロードしたDMGが「壊れている」エラーになり、システム設定の「このまま開く」も表示されない
- `APPLE_SIGNING_IDENTITY="-"` を付けるとad-hocコード署名され、「未確認の開発元」警告に変わり、システム設定から許可できる
- ビルドログに `Signing with identity "-"` が表示されることを確認すること

### Hardened Runtime と libmpv

`src-tauri/Entitlements.plist` で library validation を外し、JIT を許可している。
外すと Homebrew の libmpv を読み込めない／mpv 内蔵の Lua スクリプトが SIGKILL（Code Signature Invalid）で落ちる。
内蔵スクリプトは `player.rs` の `BUILTIN_SCRIPTS_OFF` で読み込まないようにしている。

## リリース手順

1. `src-tauri/tauri.conf.json`・`src-tauri/Cargo.toml`・`package.json` のバージョンを更新
2. 上記の環境変数付きでビルド
3. `npm run release`（GitHub Releaseの作成 + DMGのアップロード。リポジトリは非公開）

## 開発

```bash
npm run dev
```

## プロジェクト構成

```
src-tauri/
  src/lib.rs          # Rustバックエンド（メニュー、ファイルを開く、最近使ったファイル）
  src/player.rs       # mpv の起動・Tauriコマンド・イベント送信
  src/mpv/            # libmpv のFFI（dlopenで読み込み）と render API
  src/video_layer_macos.rs # WebView の下に映像を描く CAOpenGLLayer
  tauri.conf.json     # Tauri設定
  capabilities/       # パーミッション定義
  Info.plist          # ファイル関連付け
renderer/
  App.tsx             # メインコンポーネント
  components/         # UIコンポーネント
  hooks/              # カスタムフック（useTauriEvents, useMediaFiles等）
  store/              # 状態管理（videoQueueReducer）
  types/              # 型定義
  styles/             # CSS Modules
```

## コーディング規約

- **コメント**: 日本語
- **コミットメッセージ**: 英語、commitlint準拠
  - subject: 72文字以内、小文字の動詞で開始（例: `feat: add recent files menu`）
  - `@commitlint/config-conventional` + `@uphold/commitlint-config` ルール適用
- **CSS**: CSS Modules
- **インポート順**: eslint-plugin-perfectionist による自動ソート

## 主要な設計判断

- **開いたファイルの許可リスト**: `StreamAccess`（lib.rs）がユーザーが開いたファイルだけを記録し、
  `mpv_load` はこのリストにあるパスしか再生しない。ローカルHTTPのストリーミングサーバーは元の実装の名残で、
  現在のフロントエンドは使っていない（`get_stream_url`）
- **ファイルの読み取り権限**: capabilitiesにfsの静的scopeは置かない。ユーザーが開いたファイルだけを `allow_opened_paths` で個別に許可する
- **最近使ったファイル**: 最大10件、`{app_config_dir}/recent_files.json` に永続化、Rust側のみで完結
- **自動更新**: なし（このフォークでは tauri-plugin-updater を外した）
- **ドラッグ&ドロップ**: Tauri v2のネイティブイベント（`onDragDropEvent`）を使用、ブラウザのdropイベントではない
- **再生エンジン**: HTMLの`<video>`ではなく mpv。`src-tauri/src/player.rs` がTauriコマンドとイベント（"mpv"）、
  `src-tauri/src/video_layer_macos.rs` が WKWebView の下に置いた CAOpenGLLayer に mpv render API で映像を描く。
  WebView は背景を透明にし、CSSで映像エリア（`.stageVideo`）だけを透けさせる。位置は `useVideoRect` が送る
- **音声・字幕トラック**: mpv の `track-list` / `aid` / `sid` をそのまま使う（`TrackMenu`）
