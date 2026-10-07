# DynamicIsland 設計書(初版)

- 日付: 2026-10-07
- 状態: ドラフト(レビュー待ち)
- 仮称: DynamicIsland(リポジトリ名に合わせた仮名。公開前に命名を再検討)

## 1. 目的とスコープ

MacBook のノッチを「ライブアクティビティ置き場」にする macOS アプリを OSS として作る。
参考: [boring.notch](https://github.com/TheBoredTeam/boring.notch)(GPL-3.0)、Seam(商用、生産性特化)、[Vibe Island](https://github.com/vibeislandapp/vibe-island)(商用、AI エージェント特化)。

差別化の軸: 「AI エージェントの稼働状況」と「生産性機能(集中・音楽・ファイル・予定)」を 1 つの OSS で統合すること。

### 確定した前提

| 項目 | 決定 |
|---|---|
| 公開形態 | OSS(GitHub) |
| 実装方針 | ゼロから SwiftUI + AppKit(boring.notch のコードは流用しない → ライセンスを自由に選べる) |
| 開発環境 | ノッチ付き Mac + Xcode(ビルド・実機確認は手元 Mac で実施) |
| 進め方 | 土台 → AI 稼働状況 → 他機能の順 |
| AI 対象 | まずは hook 機構を持つエージェントのみ。ChatGPT / Grok 等のチャットアプリは後回し |

### MVP 機能一覧

1. 土台: ノッチ形状のウィンドウ、ホバー/クリックで展開、モジュール切替
2. AI 稼働状況: 状態表示、ノッチからの承認操作(Allow/Deny)、使用量/レート制限、ターミナルへジャンプ
3. 集中 / ポモドーロ
4. 音楽コントロール
5. ファイルシェルフ + AirDrop 送信
6. カレンダー / 会議通知

## 2. 全体アーキテクチャ

```
┌──────────────────────── DynamicIsland.app ────────────────────────┐
│  NotchWindowController (NSPanel, borderless, non-activating)       │
│     └─ NotchRootView (SwiftUI)                                     │
│          ├─ CompactView   … ノッチ左右の小表示(最優先の1件)        │
│          └─ ExpandedView  … タブ: AI / Focus / Music / Shelf / Cal │
│                                                                    │
│  ActivityArbiter  … 各モジュールの「今見せたい物」を優先度で選ぶ    │
│                                                                    │
│  Modules (IslandModule プロトコル)                                 │
│   ├─ AgentModule     ← AgentHub ← IPCServer (Unix socket)          │
│   ├─ FocusModule                                                   │
│   ├─ MusicModule     ← NowPlayingProvider                          │
│   ├─ ShelfModule     ← NSSharingService(.sendViaAirDrop)           │
│   └─ CalendarModule  ← EventKit                                    │
└────────────────────────────────────────────────────────────────────┘
        ▲ JSON over Unix domain socket
        │
  islandctl (同梱 CLI)  ← 各 AI の hook から呼ばれる薄いアダプタ
        ▲
  Claude Code hooks / Codex notify / OpenClaw hooks / Hermes hooks
```

### 2.1 ターゲット構成

| ターゲット | 種別 | 役割 |
|---|---|---|
| `IslandCore` | Swift Package | プロトコル定義、セッション状態機械、ポモドーロ計時、優先度判定。UI/AppKit 非依存でユニットテストする |
| `DynamicIsland` | macOS App | ウィンドウ、SwiftUI、各モジュールの OS 連携 |
| `islandctl` | CLI | hook から呼ばれ、ソケットへイベント送信。承認要求時は応答を待って hook に返す |

`IslandCore` を AppKit から切り離しておくのは、ロジックのテストを速く回すためと、CI(GitHub Actions の macOS ランナー)で UI なしに検証するため。

### 2.2 ノッチウィンドウ(土台)

- `NSPanel`(`.borderless`, `.nonactivatingPanel`)、`level` はメニューバーより上、`collectionBehavior` に `.canJoinAllSpaces`, `.fullScreenAuxiliary`, `.stationary`。
- ノッチ寸法は `NSScreen.safeAreaInsets.top` と `auxiliaryTopLeftArea` / `auxiliaryTopRightArea` から算出。ノッチ無しディスプレイでは疑似ノッチ(上部中央の黒い角丸)を描く。
- 状態は `closed`(ノッチと同化)/ `compact`(左右に小表示)/ `expanded`(下に展開)の 3 状態。ホバーで展開、外れたら一定時間後に閉じる。
- 閉状態ではクリック透過(`ignoresMouseEvents` を状態で切替)して、メニューバー操作を妨げない。
- ディスプレイ構成変更(`NSApplication.didChangeScreenParametersNotification`)で再配置。
- ログイン時起動は `SMAppService.mainApp`。

### 2.3 ActivityArbiter(何をノッチに出すか)

compact 表示は 1 件だけなので、モジュール横断で優先度を決める。

| 優先度 | 内容 |
|---|---|
| 1 | AI の承認要求(操作が必要) |
| 2 | AI の入力待ち / エラー |
| 3 | 会議開始直前アラート |
| 4 | AI 完了(数秒だけ表示して消える) |
| 5 | ポモドーロ進行中 |
| 6 | 音楽再生中 |

この判定は `IslandCore` に純粋関数として置き、テストで担保する。

## 3. AI 稼働状況(Phase 1〜2 の中心)

### 3.1 方針

AI ごとにアプリ本体を改造するのではなく、**ローカルの共通イベント受け口**を定義し、各 AI の hook 機構からそこへ送る。アダプタは数行のスクリプト/設定で済むため、OSS としてコミュニティが対応 AI を増やしやすい。

### 3.2 IPC 方式の比較

| 案 | 長所 | 短所 | 判断 |
|---|---|---|---|
| A. Unix ドメインソケット + `islandctl` | ポート衝突なし、ファイル権限 0600 と peer UID 検証で他ユーザー/ブラウザから叩けない、双方向(承認応答)が自然 | curl で直接叩けない(CLI 経由) | **採用** |
| B. localhost HTTP | curl で試せる | ブラウザからの localhost POST 等への対策(トークン)が必要、ポート管理 | 将来の追加口として検討 |
| C. ファイル監視 | 最も単純 | 承認の往復ができない | 不採用 |

ソケットパス: `~/Library/Application Support/DynamicIsland/island.sock`(権限 0600、接続時に `getpeereid` で同一 UID を確認)。

### 3.3 イベントプロトコル(v1、改行区切り JSON)

```json
{
  "v": 1,
  "source": "claude-code",
  "session_id": "abc123",
  "event": "running | waiting_input | permission_request | done | error | ended",
  "title": "my-project",
  "detail": "Bash: npm test",
  "cwd": "/Users/me/my-project",
  "terminal": { "app": "iTerm2", "session_id": "w0t1p0", "tty": "/dev/ttys003" },
  "request_id": "uuid (permission_request のときのみ)"
}
```

承認応答(アプリ → `islandctl`):

```json
{ "v": 1, "request_id": "uuid", "decision": "allow | deny | ask" }
```

安全側の原則: アプリ未起動・タイムアウト・不正応答のときは必ず `ask`(= 通常どおりターミナルで確認)に倒す。**自動で allow になる経路を作らない。**

### 3.4 対応 AI と取得手段

| 対象 | 手段 | 取れる状態 | 承認操作 | 優先度 |
|---|---|---|---|---|
| Claude Code | 公式 hooks(SessionStart / UserPromptSubmit / PreToolUse・PermissionRequest / Notification / Stop / SessionEnd) | 実行中・入力待ち・承認要求・完了 | 可能(hook の JSON 出力で allow/deny を返す) | Phase 1 |
| Codex CLI | `~/.codex/config.toml` の `notify` | 完了のみ(`agent-turn-complete`) | 不可(承認要求は notify 対象外) | Phase 1 |
| OpenClaw | Gateway hooks([docs](https://docs.openclaw.ai/automation/hooks)) | セッション/エージェントのイベント | 要調査 | Phase 2 |
| Hermes Agent | Gateway / Plugin hooks([docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks)) | タスク完了・失敗等 | 要調査 | Phase 2 |
| Antigravity | 公式のローカル hook は未確認 | 不明 | 不明 | Phase 2 で調査 |
| ChatGPT / Grok | hook 無し(ブラウザ拡張 or アクセシビリティ API が必要) | — | — | MVP 外 |

※ Claude Code の各 hook のペイロード・タイムアウト既定値・承認の返し方は、実装着手時に公式ドキュメントで最新仕様を確認する(仕様変更が頻繁なため、本書では確定させない)。

### 3.5 アダプタ導入

- アプリの設定画面に「Claude Code に接続」等のボタンを置き、ユーザー確認の上で各 AI の設定ファイルへ hook を追記する。
- 追記前に必ずバックアップを取り、差分を見せ、取り消しボタンを用意する(他人の設定ファイルを書き換えるため)。
- 手動導入手順も README に載せる。

### 3.6 ターミナルへジャンプ

- hook 実行時の環境変数(`TERM_PROGRAM`, `ITERM_SESSION_ID`, TTY 等)を `terminal` に詰めて送る。
- iTerm2 / Terminal.app は AppleScript で該当タブを前面化。その他(Ghostty, WezTerm, VS Code 等)はまずアプリを前面化するだけにし、個別対応は後追い。
- AppleScript には「オートメーション」権限が必要。

### 3.7 使用量 / レート制限

- 各 CLI がローカルに残すセッションログや statusline 入力から推定する方式が有力だが、**取得元とフォーマットは CLI のバージョンで変わるため未確定**。Phase 2 で調査してから設計する。
- 公式に取得できない値を推測で表示する場合は、UI 上で「推定」と明示する。

## 4. 生産性機能(Phase 3 以降、概要のみ)

| 機能 | 実装方針 | 注意点 |
|---|---|---|
| 集中 / ポモドーロ | 計時ロジックは `IslandCore`。compact 表示に残り時間リング。終了時に通知 | macOS の集中モードを直接切り替える公開 API は無い。ショートカット(`shortcuts run`)連携で代替 |
| 音楽 | まず公開手段(Music.app / Spotify の AppleScript)。全アプリ対応は MediaRemote 系(非公開 API) | MediaRemote は macOS 15.4 で制限強化。非公開 API 依存は OS 更新で壊れるリスクがあるので任意機能にする |
| ファイルシェルフ | ノッチへドラッグ&ドロップで一時保持(ブックマーク保存) | 元ファイル移動時の扱いを決める |
| AirDrop | `NSSharingService(named: .sendViaAirDrop)`(公開 API) | 送信 UI は OS 標準のシートになる |
| カレンダー | EventKit で直近予定、開始 N 分前に compact へ | カレンダーアクセス許可が必要 |

## 5. 配布・権限・ライセンス

- **App Sandbox は使わない**(Unix ソケット、他アプリの AppleScript 操作、他ツールの設定ファイル編集が必要なため)→ Mac App Store 配布は対象外。
- 配布は GitHub Releases + Homebrew cask を想定。Gatekeeper 警告なしで配るには Apple Developer Program(有料)での署名・公証が必要。未加入の間は「右クリック → 開く」手順を README に明記。
- ライセンス: ゼロから書くので選択可能。候補は MIT(採用されやすい)か GPL-3.0(派生物も OSS を強制)。**要決定**。MediaRemote 系ライブラリを取り込む場合はそのライセンスとの整合を確認する。
- 必要な権限: カレンダー、オートメーション(AppleScript)、通知。アクセシビリティは現時点で不要の見込み。

## 6. フェーズ計画

| Phase | 内容 | 完了条件 |
|---|---|---|
| 0 | 土台: プロジェクト作成、ノッチウィンドウ、3 状態遷移、モジュール枠、設定画面、ログイン時起動 | 実機ノッチ上で展開/収納が動く |
| 1 | AI 基盤: `IslandCore` のプロトコル + 状態機械、IPC サーバ、`islandctl`、Claude Code アダプタ(状態・承認・ジャンプ)、Codex アダプタ(完了通知) | Claude Code の承認をノッチから Allow/Deny できる |
| 2 | AI 拡張: OpenClaw / Hermes アダプタ、使用量表示、Antigravity 調査 | 4 種以上の AI が同一パネルに並ぶ |
| 3 | 集中 / ポモドーロ | |
| 4 | 音楽 | |
| 5 | ファイルシェルフ + AirDrop | |
| 6 | カレンダー / 会議通知 | |
| 7 | 配布: 署名・公証、Homebrew、README | |

## 7. テスト方針

- `IslandCore`: プロトコルのエンコード/デコード、セッション状態遷移、Arbiter の優先度、ポモドーロ計時をユニットテスト(`swift test`)。
- IPC: 実ソケットを使った結合テスト(承認タイムアウト時に `ask` へ倒れることを必ず検証)。
- UI: 実機での目視確認チェックリスト(ノッチ有/無、外部ディスプレイ、フルスクリーン、Space 切替)。
- CI: GitHub Actions の macOS ランナーで `swift test` と `xcodebuild build`。

## 8. リスクと未決事項

| # | 内容 | 対応 |
|---|---|---|
| R1 | 競合 Vibe Island(25 種の AI 対応・承認操作あり)と機能が重なる | 「生産性機能との統合」と「OSS・共通プロトコル」で差別化。先に既存 OSS 実装(open-vibe-island 等)の有無・内容を確認する |
| R2 | 各 AI の hook 仕様が頻繁に変わる | アダプタを本体から分離し、プロトコル v を明記 |
| R3 | 承認操作のセキュリティ | ソケット権限 + UID 検証、失敗時は必ず `ask` |
| R4 | MediaRemote 非公開 API | 任意機能化、公開手段を既定に |
| R5 | 開発は手元 Mac 依存(クラウド環境は Linux で Swift 未導入) | ロジックは `IslandCore` に寄せ、CI の macOS ランナーで検証 |
| Q1 | ライセンス(MIT / GPL-3.0) | 要決定 |
| Q2 | アプリ名 | 要決定 |
| Q3 | 最低対応 macOS(14 Sonoma 案) | 要決定 |
