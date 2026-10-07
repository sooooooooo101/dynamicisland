# AInotch 設計書(第3版)

- 日付: 2026-10-07
- 状態: ドラフト(レビュー待ち)
- アプリ名: AInotch
- 改訂履歴
  - 第1版: 汎用ノッチアプリ(AI 状況 + ポモドーロ/音楽/シェルフ/カレンダー)
  - 第2版: **「毎日 AI を使う人」向けに特化**。汎用機能はプラグイン化して後回し。主対象をデスクトップ/ブラウザ/IDE に変更
  - 第3版: アプリ名 AInotch、ライセンス MIT、最低対応 macOS 14 を確定。音声認識エンジンの方針を追記

## 1. コンセプト

> 複数の AI を並行して使う人のための、ノッチの「AI 管制塔」。

AI の生成・エージェント実行は数十秒〜数十分かかり、その間ユーザーは別のウィンドウ・別の Space で作業している。完了・承認待ち・エラーが埋もれることが日常的な損失になっている。ノッチは「どの Space・どのアプリを見ていても常に視界にある唯一の場所」なので、ここに AI の状態を集約する。

### 1.1 差別化

| 競合 | 主な対象 | 本アプリとの違い |
|---|---|---|
| [Vibe Island](https://github.com/vibeislandapp/vibe-island)(商用) | CLI コーディングエージェント 25 種 | 本アプリはデスクトップ/ブラウザ/ローカル LLM/音声入力まで扱う。OSS |
| [CodeIsland](https://github.com/wxtsky/CodeIsland) | CLI/IDE エージェント | 要調査(ライセンス・機能範囲) |
| [boring.notch](https://github.com/TheBoredTeam/boring.notch)(OSS) | 音楽・汎用 | AI 監視は未実装([要望 #951](https://github.com/TheBoredTeam/boring.notch/issues/951)) |
| Seam / NotchNook / Alcove(商用) | 生産性・汎用 | AI 特化ではない |

空いている領域として **ローカル LLM・デスクトップ/ブラウザの AI アプリ・音声入力** を主戦場にする。

### 1.2 確定した前提

| 項目 | 決定 |
|---|---|
| 公開形態 | OSS(GitHub)、ライセンスは MIT(暫定) |
| 最低対応 macOS | 14 Sonoma |
| 実装方針 | ゼロから SwiftUI + AppKit(boring.notch のコードは流用しない) |
| 開発環境 | ノッチ付き Mac + Xcode(ビルド・実機確認は手元 Mac) |
| 主な利用シーン(優先順) | 1. デスクトップ/ブラウザの AI(ChatGPT / Claude / Gemini / Grok 等) 2. IDE(Antigravity / Cursor 等) 3. ローカル LLM 4. CLI エージェント |
| 汎用機能 | ポモドーロ・音楽・カレンダーは本体から外し、拡張システム完成後にプラグインとして提供 |

## 2. 機能一覧

### 2.1 採用(本体)

| # | 機能 | 概要 |
|---|---|---|
| F1 | AI 稼働状況 | 生成中 / 完了 / 入力待ち / 承認待ち / エラーをノッチに表示。複数セッションを一覧 |
| F2 | ノッチから承認 | 承認要求に Allow / Deny で応答(hook が対応する AI のみ) |
| F3 | 元の画面へジャンプ | クリックで該当のブラウザタブ / アプリ / IDE / ターミナルを前面化 |
| F4 | 使用量 / レート制限 | 取得可能な AI のみ。推定値は「推定」と明示 |
| F5 | ローカル LLM 監視 | Ollama / LM Studio のロード中モデル、メモリ使用量、アンロードまでの時間、手動アンロード、メモリ逼迫警告 |
| F6 | 音声プロンプト | ノッチで録音 → オンデバイス文字起こし → 前面の AI 入力欄へ挿入 |
| F7 | AI アプリの通知集約 | AI 系アプリの通知だけをノッチに集約(オプトイン) |
| F8 | Space 表示 | 現在の Space と、各 AI セッションがどの Space にあるかを表示 |

### 2.2 候補(未採用・バックログ)

| 機能 | 概要 |
|---|---|
| 長時間コマンド監視 | `ainotchctl run -- <cmd>` で任意ジョブの進行・完了を表示 |
| コンテキスト置き場 | ファイル / スクショを溜めてプロンプトに投入、AirDrop 送信 |
| AI 障害情報 | 各社ステータスページの障害を表示 |
| 汎用プラグイン | ポモドーロ、音楽、カレンダー、天気 等 |

## 3. 全体アーキテクチャ

```
┌──────────────────────── AInotch.app ─────────────────────────┐
│  NotchWindowController (NSPanel)                                    │
│     └─ NotchRootView (SwiftUI): Compact / Expanded                  │
│  ActivityArbiter … 「今ノッチに出す 1 件」を優先度で決定             │
│                                                                     │
│  AgentHub ── 全ソースのセッション状態を統合                          │
│     ▲ IPCServer (Unix socket)         ▲ AXWatcher (アクセシビリティ) │
│     │                                 ▲ NotificationReader (任意)   │
│  LocalLLMMonitor (Ollama / LM Studio の HTTP API をポーリング)       │
│  VoicePrompt (録音 → 文字起こし → 挿入)                              │
│  SpaceTracker (Space 番号・ウィンドウ所属)                           │
└─────────────────────────────────────────────────────────────────────┘
        ▲ JSON over Unix domain socket
  ainotchctl ── ① IDE / CLI の hook から呼ばれる
            └─ ② ブラウザ拡張の Native Messaging Host を兼ねる
        ▲
  Cursor / Antigravity hooks、Claude Code / Codex 等の hooks、ブラウザ拡張
```

### 3.1 ターゲット構成

| ターゲット | 種別 | 役割 |
|---|---|---|
| `AInotchCore` | Swift Package | プロトコル、セッション状態機械、Arbiter、パーサ類。AppKit 非依存でユニットテスト |
| `AInotch` | macOS App | ウィンドウ、UI、OS 連携(AX、音声、Space 等) |
| `ainotchctl` | CLI | hook アダプタ兼 Native Messaging Host |
| `extension-chromium` | ブラウザ拡張(MV3) | Chrome / Arc / Brave / Edge 共通 |
| `extension-safari` | Safari Web Extension | アプリに同梱(Phase 2 後半) |

## 4. 土台(ノッチウィンドウ)と品質要件

### 4.1 実装

- `NSPanel`(`.borderless`, `.nonactivatingPanel`)、メニューバーより上のレベル、`.canJoinAllSpaces` / `.fullScreenAuxiliary` / `.stationary`。
- ノッチ寸法は `NSScreen.safeAreaInsets.top` と `auxiliaryTopLeftArea` / `auxiliaryTopRightArea` から算出。ノッチ無しディスプレイは疑似ノッチ。
- 状態は `closed` / `compact` / `expanded` の 3 つ。閉状態はクリック透過。
- ログイン時起動は `SMAppService.mainApp`。

### 4.2 品質要件(boring.notch の不具合報告から抽出)

| 要件 | 根拠 |
|---|---|
| スリープ復帰・ディスプレイ構成変更時にウィンドウを再生成/再配置し、消えない・ずれない | [#336](https://github.com/TheBoredTeam/boring.notch/issues/336) 👍44、[#352](https://github.com/TheBoredTeam/boring.notch/issues/352) 👍11 |
| 開閉アニメーションを滑らかにし、オフにもできる | [#341](https://github.com/TheBoredTeam/boring.notch/issues/341)、[#303](https://github.com/TheBoredTeam/boring.notch/issues/303) |
| フルスクリーン時・外部ディスプレイでの表示可否、アプリ単位の除外を設定可能 | [#119](https://github.com/TheBoredTeam/boring.notch/issues/119)、[#239](https://github.com/TheBoredTeam/boring.notch/issues/239)、[#110](https://github.com/TheBoredTeam/boring.notch/issues/110) |
| ノッチ展開時の幅・高さを調整可能 | [#300](https://github.com/TheBoredTeam/boring.notch/issues/300) |
| 非公開 API 依存機能は OS 更新で壊れても本体が落ちない(機能単位で無効化) | [#417](https://github.com/TheBoredTeam/boring.notch/issues/417) 👍58 |

### 4.3 ActivityArbiter の優先度

| 優先度 | 内容 |
|---|---|
| 1 | 承認待ち(操作が必要) |
| 2 | 入力待ち / エラー |
| 3 | 録音中(音声プロンプト) |
| 4 | メモリ逼迫警告(ローカル LLM) |
| 5 | 完了(数秒表示して消える) |
| 6 | 生成中(複数ある場合は件数表示) |

## 5. AI 稼働状況(F1〜F4)

### 5.1 共通イベントプロトコル v1

全ソースは最終的にこの形に正規化して `AgentHub` に入る。外部からは Unix ソケット(`~/Library/Application Support/AInotch/ainotch.sock`、権限 0600、`getpeereid` で同一 UID を確認)に改行区切り JSON で送る。

```json
{
  "v": 1,
  "source": "cursor | antigravity | chatgpt-web | claude-desktop | claude-code | ...",
  "session_id": "abc123",
  "event": "running | waiting_input | permission_request | done | error | ended",
  "title": "my-project / 会話タイトル",
  "detail": "Bash: npm test",
  "origin": {
    "kind": "browser | app | ide | terminal",
    "bundle_id": "com.google.Chrome",
    "tab_id": 123,
    "tty": "/dev/ttys003"
  },
  "request_id": "uuid (permission_request のみ)"
}
```

承認応答: `{ "v": 1, "request_id": "uuid", "decision": "allow | deny | ask" }`

**安全原則**: アプリ未起動・タイムアウト・不正応答は必ず `ask`(= 各 AI の通常の確認画面)に倒す。自動 allow の経路は作らない。

**プライバシー原則**: 既定では会話本文・プロンプト本文を送らない。送るのは状態・タイトル・短い detail のみ。本文の表示はソースごとのオプトイン。

### 5.2 ソース別の取得手段

| 優先 | 対象 | 手段 | 取れる状態 | 承認 | 確度 |
|---|---|---|---|---|---|
| 1 | ブラウザの ChatGPT / Claude / Gemini / Grok | ブラウザ拡張が DOM を監視(停止ボタンの有無等)→ Native Messaging で `ainotchctl` → ソケット | 生成中 / 完了 / エラー | 不可 | 中(サイト改修で壊れる → セレクタを拡張内の設定に分離し、更新を容易にする) |
| 1 | デスクトップの ChatGPT / Claude 等 | アクセシビリティ API(`AXObserver`)で UI 要素を監視 + 任意で通知 DB(F7) | 生成中 / 完了 | 不可 | 低〜中(アプリ更新で壊れる。アクセシビリティ権限が必要) |
| 2 | Cursor | 公式 hooks([docs](https://cursor.com/docs/hooks))。`beforeShellExecution` は allow/deny を返せる | 実行中 / 完了 / 承認待ち | 可(シェル実行) | 中(フォーラムで hook 不発の報告あり) |
| 2 | Antigravity | 公式 hooks([docs](https://antigravity.google/docs/hooks/))。PreToolUse / PostToolUse / PostInvocation / Stop | 実行中 / 完了 | 要確認 | 中(Stop が発火しない報告あり → PostInvocation + タイムアウトで完了推定) |
| 4 | Claude Code / Codex / OpenClaw / Hermes | 各公式 hook(第1版の調査どおり) | 状態一式(Codex は完了のみ) | Claude Code は可 | 高〜中 |
| — | 使用量(F4) | CLI のローカルログ等から。Web/デスクトップのチャット利用量は取得手段なし | — | — | 要調査 |

※ 各 hook のペイロードと戻り値の仕様は変更が多いため、実装着手時に公式ドキュメントで確認する。

### 5.3 ジャンプ(F3)

| origin | 方法 |
|---|---|
| browser | 拡張に `tabs.update` / `windows.update` を依頼して該当タブを前面化 |
| app / ide | `NSRunningApplication.activate`。IDE はワークスペースのウィンドウを AX で特定 |
| terminal | iTerm2 / Terminal.app は AppleScript、その他はアプリ前面化のみ |

## 6. ローカル LLM 監視(F5)

| 項目 | 手段 |
|---|---|
| Ollama のロード中モデル、使用メモリ、アンロード予定時刻 | `GET http://localhost:11434/api/ps`(`size`, `size_vram`, `expires_at`, `context_length` 等。[公式](https://docs.ollama.com/api/ps.md)) |
| Ollama の手動アンロード | `keep_alive: 0` を指定したリクエスト(実装時に仕様確認) |
| LM Studio | ローカル REST API(エンドポイント・フィールドは要調査) |
| メモリ逼迫警告 | `DispatchSource.makeMemoryPressureSource` + システムのメモリ統計。ユニファイドメモリのため、大きいモデルのロード時に警告 |

設計方針: 各ランタイムを `LocalLLMProvider` プロトコルで抽象化し、llama.cpp server 等を後から追加できるようにする。ポーリング間隔は、ロード中モデルがある時は短く、無い時は長くする。

## 7. 音声プロンプト(F6)

1. グローバルホットキー(押している間録音)またはノッチのボタンで録音開始。録音中はノッチに波形。
2. オンデバイスで文字起こし。エンジンは `TranscriptionEngine` プロトコルで抽象化し、OS と設定で切り替える(下表)。
3. 結果を前面アプリのフォーカス中の入力欄に挿入(ペーストボード経由 + キー入力合成。アクセシビリティ権限が必要)。挿入前に元のペーストボード内容を退避・復元。
4. 拡張: ローカル LLM でフィラー除去・整形してから挿入(F5 と連携、任意)。

### 7.1 文字起こしエンジン

| エンジン | 対応 OS | 長所 | 短所 | 位置づけ |
|---|---|---|---|---|
| [WhisperKit](https://github.com/argmaxinc/whisperkit)(MIT) | macOS 14 以降、Apple Silicon | 日本語に強い Whisper 系、完全オンデバイス | 初回にモデルのダウンロード(数百 MB 規模、モデルで変動)、メモリ消費 | macOS 14〜15 の既定 |
| Apple `SpeechAnalyzer` / `SpeechTranscriber` | macOS 26 以降([公式](https://developer.apple.com/documentation/speech/speechanalyzer)) | OS 内蔵で追加ダウンロード不要(言語アセットは OS 管理)、日本語対応 | macOS 26 限定 | macOS 26 以降の既定(`if #available` で切替) |
| Apple `SFSpeechRecognizer` | macOS 10.15 以降 | 追加依存なし | 旧世代の精度、日本語のオンデバイス可否は `supportsOnDeviceRecognition` で要確認。オンデバイス不可ならサーバ送信になるため既定では使わない | フォールバック(オンデバイス可の場合のみ) |

ノッチ付き MacBook はすべて Apple Silicon のため、WhisperKit の Apple Silicon 限定は実質的な制約にならない(ノッチ無しの Intel Mac では F6 を無効化)。

## 8. 通知集約(F7)

- 他アプリの通知を読む公開 API は無い。macOS 15 以降、通知 DB は `~/Library/Group Containers/group.com.apple.usernoted/` に移り、読むにはフルディスクアクセスが必要([参考](https://mjtsai.com/blog/2024/07/15/sequoia-finally-addresses-notification-center-privacy))。
- 方針: **オプトイン機能**とし、AI 系アプリ(バンドル ID の許可リスト)の通知のみ読み取り専用で扱う。DB 形式は非公開のため、読めない場合は機能を自動無効化して本体に影響させない。
- 用途: デスクトップ AI アプリの「応答完了」通知を、F1 の完了検知の補助としても使う。

## 9. Space 表示(F8)

- Space 番号・ウィンドウの所属 Space を取得する公開 API は無い。yabai 等と同じく非公開の CoreGraphics(CGS)系 API を使う。
- 非公開 API のため、取得失敗時は表示を隠すだけにする(4.2 の品質要件)。
- 用途: 「Space 3 の Cursor が承認待ち」のように、AI セッションの所在を示す。ジャンプ(F3)時は該当 Space へ切り替わる。

## 10. 拡張(プラグイン)システム

- 汎用機能(ポモドーロ・音楽・カレンダー等)と、AI ソースのアダプタを本体外に出すための仕組み。
- 第一段階は「外部プロセスがソケットにイベントを送るだけ」の疎結合方式(= 5.1 のプロトコルをそのまま使う)。UI を伴うプラグイン(独自ビュー)は第二段階で設計する。

## 11. 配布・権限・ライセンス

- App Sandbox は使わない(ソケット、AX、AppleScript、他ツールの設定編集のため)→ Mac App Store 対象外。GitHub Releases + Homebrew cask。
- Gatekeeper 警告なしの配布には Apple Developer Program(有料)での署名・公証が必要。
- ライセンスは MIT。GPL 系のコード・ライブラリは取り込まない(MIT で配布できなくなるため)。依存追加時はライセンスを確認する。
- 必要な権限: アクセシビリティ(F1 デスクトップ監視・F6 挿入)、マイク(F6)、フルディスクアクセス(F7、任意)、オートメーション(F3 ターミナル)。**権限は機能を有効化した時点で個別に要求**し、初回起動で一括要求しない。

## 12. フェーズ計画

| Phase | 内容 | 完了条件 |
|---|---|---|
| 0 | 土台 + 4.2 品質要件 | 実機でスリープ復帰・外部ディスプレイ・フルスクリーンでも正しく動く |
| 1 | AI 基盤(`AInotchCore` プロトコル/状態機械、IPC、`ainotchctl`)+ IDE アダプタ(Cursor / Antigravity) | IDE のエージェント完了がノッチに出て、クリックで戻れる |
| 2 | ブラウザ拡張(Chromium → Safari)。ChatGPT / Claude / Gemini / Grok | ブラウザで生成完了がノッチに出て、タブへ戻れる |
| 3 | デスクトップアプリ監視(AX)+ 通知集約(F7) | ChatGPT / Claude デスクトップの完了を検知 |
| 4 | ローカル LLM 監視(F5) | Ollama のモデル状態表示とアンロード |
| 5 | 音声プロンプト(F6) | 日本語の音声が前面の AI 入力欄に入る |
| 6 | CLI エージェント + 承認(F2)+ 使用量(F4) | Claude Code の承認をノッチから操作 |
| 7 | Space 表示(F8) | |
| 8 | 拡張システム + 汎用プラグイン | |
| 9 | 配布(署名・公証、Homebrew、README) | |

## 13. テスト方針

- `AInotchCore`: プロトコル、状態遷移、Arbiter、各ソースのイベント正規化をユニットテスト。
- IPC: 実ソケット結合テスト(承認タイムアウトで `ask` に倒れることを必須で検証)。
- ブラウザ拡張: 各サイトの DOM スナップショットに対するセレクタのテスト(サイト改修の検知用)。
- UI: 実機チェックリスト(ノッチ有/無、外部ディスプレイ、フルスクリーン、Space 切替、スリープ復帰)。
- CI: GitHub Actions の macOS ランナーで `swift test` と `xcodebuild build`。

## 14. リスクと未決事項

| # | 内容 | 対応 |
|---|---|---|
| R1 | ブラウザ/デスクトップ監視は DOM・UI 構造依存で壊れやすい(最優先領域が最も壊れやすい) | セレクタ/AX パスを設定データに分離、テストで検知、壊れたら該当ソースだけ無効化 |
| R2 | 非公開 API・非公開 DB(Space、通知) | 機能単位で隔離、失敗時は非表示 |
| R3 | 権限の多さが導入の障壁になる | 機能ごとの遅延要求、権限が必要な理由を UI で説明 |
| R4 | 承認操作・ソケットのセキュリティ | 0600 + UID 検証、失敗時は `ask` |
| R5 | 競合(Vibe Island、CodeIsland)との重複 | Phase 1 前に既存 OSS 実装を調査し、プロトコル互換や流用可否を判断 |
| R6 | 開発は手元 Mac 依存(クラウド環境は Linux・Swift 未導入) | ロジックを `AInotchCore` に寄せ CI で検証 |
| R7 | 「AInotch」の名称が既存アプリ・商標と衝突する可能性 | 公開前に App Store・GitHub・商標を検索して確認 |
