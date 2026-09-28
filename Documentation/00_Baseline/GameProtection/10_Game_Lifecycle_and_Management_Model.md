# GameSecurityTool

# Game Lifecycle and Management Model

## Game Registration / Installation State / Lifecycle Control Definition

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-010 |
| Version | 3.1 (End-User Protection Status Vocabulary Edition) |
| Status | Formal Baseline Specification (Highest Lifecycle Authority) |
| Category | Game Lifecycle Management |
| Authority Level | Core Management Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- ゲーム登録およびプロファイル管理
- 静的ライフサイクル状態遷移モデル（Registered ➔ Installed ➔ Active ➔ Inactive ➔ Missing ➔ Archived ➔ Removed）
- 実行セッション全生命周期保護（未起動時 0% 待機・起動時事前ブロック・終了時ポスト監査・クラッシュ修復）
- 起動モード（通常起動 vs 厳格モード起動）およびポータブル安全展開・本体アーカイブ管理
- プロセス監視障害時の健全性可視化（C-6 是正）
- 重複・移動検知、クリーンアップ、UI モデル、および監査要件

を定義する。

---

GST はゲーム本体を支配・改変するのではなく、

```text
Game Environment State & Active Session Lifecycle
(ゲーム環境の状態およびセッションの全生命周期)
```

を安全に管理・調停する。

---

# 1. Lifecycle Philosophy

## 1.1 Core Principle
ゲーム状態は、ファイルが存在するか否かだけで短絡的に判断しない。

GST では：
```text
Historical Context  *  Current OS State  *  User Intent
(過去の履歴・バックアップ、現在の OS 実行状態、ユーザーの意図)
```
を総合的に考慮して管理する。

---

## 1.2 Ownership Boundary (所有権境界)

### GST が管理するもの:
```text
Game Registration & Profiles
Lifecycle States & Session Tracking
Protection Policies (Universal Rule Scope)
Backup References & Manifests (ピン留め・タグ)
Archive Storage References (本体アーカイブ保管先)
Audit History & Evidence
```

---

### GST が所有・支配しないもの (禁止事項):
```text
Game Binaries (.exe / .dll)
Purchased Licenses / DRM
Save Data Ownership (ユーザーの完全な資産)
User Created Content & Mods
```

---

# 2. Game Entity Model

GST では登録ゲームを `GameProfileRecord` / `GameFolderDto` として管理する。

構成：
```text
Game Profile
├ Identity (GameId, DisplayName, Publisher, Platform)
├ Installation State (Registered / Installed / Active / Inactive / Missing / Archived / Removed)
├ Target Paths (MainExecutable, InstallationDirectory, SaveDirectory)
├ Launch Mode (Standard Launch / Strict Safe Launch)
├ Protection Policy (Firewall Scope, WebLink Mode, Mod Diagnostics)
├ Backup References & Manifests (ピン留め・タグ履歴)
├ Archive Storage Directory (本体アーカイブ 各種圧縮形式/.zip/.rar 保管先)
└ Lifecycle & Audit History
```

---

# 3. Lifecycle State Model (静的状態モデル)

GST の基本状態遷移：

```text
  [ユーザー登録 / 自動オンボーディング]
                    │
                    ▼
             (1. Registered)
                    │
                    ▼ (実行ファイル確認)
             (2. Installed)
                    │
         ┌──────────┼──────────┐
         ▼          ▼          ▼ (アンインストール・ドライブ切断)
    (3. Active) (4. Inactive) (5. Missing) ──(再検出・再配置)──> (2. Installed)
 (ゲーム実行中) (通常待機中)    │
         │          │          ▼ (ユーザーによる保管指定)
         │          │     (6. Archived) ──(本体アーカイブ展開)─> (2. Installed)
         │          │          │
         └──────────┴──────────┴──(ユーザーの明示的削除承認)──> [7. Removed]
```

---

# 4. Registered State (登録状態 - 完全復元)

## 4.1 Definition
ユーザーが GST へゲームプロファイルを登録した初期状態。

- **条件:** プロファイルレコードが存在すること。
- **特徴:** ゲーム本体が未インストールまたは別ドライブにあっても保持可能。
- **用途:** PC 移行前の事前設定、過去バックアップの管理、履歴保持。

---

# 5. Installed State (配置確認状態 - 完全復元)

## 5.1 Definition
ゲームの実体およびメイン実行バイナリ（`.exe`）がローカルディスク上で確認できる状態。

- **条件:** `Registered` ＋ `Installation Path Valid` ＋ `MainExecutable Exists`。
- **保持情報:** インストールパス、検出日時、実行バイナリハッシュ・署名状態。

---

# 6. Active State (実行・保護中状態 - 完全復元 ＆ セッション保護統合)

## 6.1 Definition
ゲームプロセスが起動され、リアルタイム保護が稼働している状態。

## 6.2 実行セッション全生命周期保護 (FN-SYS-04 連携)
`Active` 状態におけるプロセス実行の全ライフサイクルを以下の通り調停する：

```text
┌────────────────────────────────────────────────────────┐
│ 1. 未起動時 (Standby 待機)                             │
│    - 全監視エンジン完全停止 (CPU 0.00% / OFF 待機)     │
│    - WMI プロセストレースからの受動通知のみ待機        │
└──────────────────────────┬─────────────────────────────┘
                           │ (ProcessStarted イベント検知)
                           ▼
┌────────────────────────────────────────────────────────┐
│ 2. 起動時事前ブロック (Pre-Launch Security)            │
│    - WER クラッシュダンプ送信一時抑止 (BeforeState 退避)│
│    - Firewall ルール先行適用 (厳格モード時は In/Out 遮断)│
│    - 起動前スナップショット採取 (ファイル・レジストリ) │
│    - 監視エンジン ON                                   │
└──────────────────────────┬─────────────────────────────┘
                           │ (ゲーム実行中)
                           ▼
┌────────────────────────────────────────────────────────┐
│ 3. 終了時ポスト監査 (Post-Launch Audit)                │
│    - プロセス消滅検知 (ProcessStopped)                 │
│    - ドロップファイル自動検出 ＆ レジストリ改ざん清掃  │
│    - レジストリ Run キー ＆ hosts ファイル改ざん監査   │
│      (不正持続化の検知とワンクリック復元提供)          │
│    - 主要ランチャー越境改ざん監査 (ILauncherSecurity)  │
│    - セーブデータ自動バックアップ実行 (差分検知)       │
│    - WER 設定の元値復元 ➔ 監視エンジン OFF (0% 復帰)   │
└────────────────────────────────────────────────────────┘
```

## 6.3 プロセス監視障害時の Fail-Open 根絶 (C-6 是正)
WMI 監視ウォッチャーの停止時、保護が有効であると誤認させないため、`WatcherFaulted` イベントを介して UI / ダッシュボードへ「⚠️ プロセス監視エンジン停止（保護無効）」を即時通知し、30 秒周期で自律再初期化リトライを行う。

## 6.4 主要ランチャー越境改ざん遮断 ＆ コアファイル監視 (`ILauncherSecurityAdapter`)
ゲームプロセスが Steam, Epic Games, EA 等の公式ランチャー本体フォルダ（実行バイナリやコア DLL 等）を改ざん・不正上書きする行為（横展開攻撃）を検知・遮断する。また、`ILauncherSecurityAdapter` を通じてランチャーコアファイルの署名・完全性ベースラインを監視し、改ざんが検知された場合は即時に警告し復旧オプションを提示する。

## 6.5 【確定】スリープ復帰時ゴーストプロセス狩り (Ghost Process Hunting on Resume)
OS のサスペンド・レジューム（`PowerModes.Resume`）発生時、スリープ中にゲームプロセスが異常終了またはクラッシュして消滅しているケース（ゴーストプロセス）に対処する：
1. **WMI 再購読:** 復帰通知を契機に、既存の `ManagementEventWatcher` を明示破棄して新規再生成し、WMI 監視の不感症を排除する。
2. **アクティブセッション生存確認:** 管理中の `_activeSessions` の各 PID に対し、`SafeProcessHandle` による生存確認を直ちに実行。
3. **即時ポスト監査・元値復元:** スリープ中に消滅したプロセスを検知した場合、待機することなく即時にポスト監査（ドロップファイル検出・レジストリ清掃）をキックし、WER ダンプ抑止の解除および一時 FW ルールを安全に撤去して監視エンジンを 0%（OFF）へ遷移させる。

## 6.6 【確定】稼働中ゲームの能動合流 (Mid-Session Ingestion)
GST の起動前にすでにユーザーによって起動されていたゲームプロセスを捕捉し、途中保護を開始する：
1. **能動的プロセス走査:** GST 起動時シーケンスにおいて、`Process.GetProcesses()` を実行し、登録済みゲームプロファイルの実行バイナリ名と照合する。
2. **途中保護アクティブ化:** 合致する稼働中ゲームが存在し未追跡の場合、`IsMidSessionIngested = true` としてアクティブセッションを生成。
3. **保護リソースの即時適用:** WER クラッシュレポート抑止の有効化、厳格モード時の Firewall ルール適用、および現在のファイル・レジストリ状態スナップショットを採取し、途中からの安全保護へ確実に合流させる。

## 6.7 【確定】GST 意図的終了時の即時クリーンアップ (Intentional GST Exit Protection)
保護対象ゲームが実行中にユーザーが GST を終了（Exit）しようとした場合の安全調停：
1. **ユーザー確認ダイアログ:** ゲーム保護中である旨を提示し、「タスクトレイへ最小化して保護継続」または「保護を解除して GST を完全終了」の選択を仰ぐ。
2. **即時安全ティアダウン:** 完全終了が選択された場合、OS 側に一時展開されたファイアウォール遮断ルール（`GST:STRICT-TEMP:`）をミリ秒単位で即時全削除し、WER のクラッシュダンプ設定を元のクリーンな状態へ復元してからプロセスを安全終了する。
3. **シャットダウン時の高速処理:** OS シャットダウン（`SessionEnding`）時は、ダイアログを出さず直ちに一時 FW ルール撤去と WER 復元を最優先実行し、DB 接続プールを切断する。

## 6.8 【確定】複数ゲーム同時起動の調停モデル (Multi-Game Concurrent Session Mediation)
複数のゲームが同時に起動された場合の競合防止とリソース調停：
1. **PID 単位セッション分離:** `ConcurrentDictionary<uint, ActiveSessionState>` により、各ゲームプロセスの状態・スナップショットを PID 単位で完全に分離保持する。
2. **Firewall ルールの衝突防止:** 一時ブロックルールは対象ゲームバイナリの絶対実行パスに厳密にバインドされ、別ゲームプロセスの通信を誤って遮断しない。
3. **WER 抑止の参照カウント管理:** `CrashReportTelemetryBlocker` に参照カウント（アクティブゲーム数）を導入。1つのゲームが終了しても他ゲームが稼働中であれば抑止を継続し、全ゲーム終了時にのみ WER 設定を元値へ復元する。
4. **セーブバックアップの直列化 (Concurrency = 1):** 複数ゲームが連続または同時に終了した場合でも、容量 1 の有界チャネル（Bounded Channel）を用いた Debounce キューにより、ディスク I/O 負荷を集中させず 1 件ずつ直列実行する。
5. **インゲーム HUD の自動追従:** OS の最前面ウィンドウ（`GetForegroundWindow` ➔ `GetWindowThreadProcessId`）を監視し、現在ユーザーがアクティブに操作しているゲームのオーバーレイ情報・コンテキストへ動的に自動追従する。

---

# 7. Inactive State (非アクティブ状態 - 完全復元)

## 7.1 Definition
インストール済みであるが、現在ゲームが起動されておらず、監視エンジンが 0% / OFF で待機している状態。削除候補ではなく、通常の安全待機状態である。

---

# 8. Missing State (行方不明状態 - 完全復元)

## 8.1 Definition
GST に登録情報は存在するが、ゲーム本体フォルダや実行ファイルが確認できない状態。
- **例:** ゲームのアンインストール、外付けドライブの取り外し、フォルダの手動移動。

## 8.2 Missing Handling (行方不明時の保護規約)
- **保持情報:** ゲーム履歴、過去のバックアップ ZIP 参照、旧インストールパス、監査証跡。
- **禁止事項:** `Missing` を検知した瞬間に、GST 登録やバックアップ履歴を勝手に自動削除すること（**自動削除の厳禁**）。

---

# 9. Archived State (保管状態 - 完全復元)

## 9.1 Definition
意図的に外部アーカイブ（各種圧縮形式/`.zip`/`.rar`）へ退避・保管された状態。
- **用途:** 過去バージョンの保管、大型 MOD パック導入前の退避、再ダウンロード回避。
- **復元:** GST の「ゲーム本体アーカイブ管理」画面から、ワンクリックで直接ストリーミング展開して `Installed` 状態へ復帰可能。

---

# 10. Removed State (除外状態 - 完全復元)

## 10.1 Definition
ユーザーの明示的な操作により、GST の管理対象から完全に除外された状態。
- **削除前要件:** ユーザー確認ダイアログの表示、影響範囲の説明、および監査ログ記録。

---

# 11. State Transition Rules (状態遷移規則 - 完全復元)

## 11.1 通常遷移 (Normal Transition)
`Registered ➔ Installed ➔ Active ➔ Inactive ➔ Archived`

## 11.2 アンインストール遷移 (Uninstall Transition)
ゲーム本体削除検知時：`Installed ➔ Missing`（GST 登録情報およびバックアップは確実に保持）。

## 11.3 再インストール遷移 (Reinstall Transition)
同一ゲーム再検出時：`Missing ➔ Installed`（パス整合性検証およびプロファイル再リンク）。

---

# 12. Game Detection Model (ゲーム検出モデル - 完全復元)

## 12.1 Detection Purpose
ゲーム検出の目的は、ローカル状態と GST 管理プロファイルの同期（State Synchronization）である。

## 12.2 Detection Limitation
禁止事項：
```text
Detected (検出された)  ==  Trusted (安全である)
```
検出されたことのみを根拠として、未知のバイナリを安全と見なしてはならない。

---

# 13. Game Cleanup Model (クリーンアップ規約 - 完全復元)

## 13.1 Cleanup Purpose
Cleanup とは、不要になった GST 内部メタデータや孤立 Firewall ルールを安全に整理する機能である。

## 13.2 Cleanup Exclusion (整理対象外 - 所有権保護)
以下をクリーンアップで勝手に削除することを厳格に禁止する：
- ゲーム本体ファイル
- セーブデータ原本およびバックアップ ZIP
- ユーザー作成 MOD

## 13.3 Cleanup Flow
```text
Candidate Detection ➔ Explanation & Preview ➔ User Approval ➔ Cleanup ➔ Audit
```

---

# 14. Duplicate Game Detection (重複登録検知 - 完全復元)

同一ゲームの重複登録を防止するため、Game ID、実行ファイル名、Publisher、インストールパスを複合照合する。重複を検知した場合は自動削除せず、ユーザーに統合または選択を提示する。

---

# 15. Game Move Detection (ゲーム移動検知 - 完全復元)

ゲームフォルダが別ドライブや別パスへ移動された場合（例: `C:\Games` ➔ `D:\Games`）：
1. パス不一致の検出
2. ユーザーへの通知と新パス確認
3. プロファイル参照パスのアトミック更新と監査ログ記録

---

# 16. Lifecycle and Backup Relationship (バックアップ保護連携 - 完全復元)

ゲームのライフサイクル状態が変化（`Installed ➔ Missing ➔ Archived`）しても、**作成済みのセーブデータバックアップ（Standard ZIP）およびピン留めスナップショットは確実に保持** される。

---

# 17. Lifecycle and Migration Relationship (PC 移行連携 - 完全復元)

PC 移行時、ゲームプロファイルとライフサイクル履歴を新 PC へ安全に移送し、新環境での検出結果に基づいて `Installed` または `Missing` 状態へ自動再同期する。

---

# 18. UI Representation Model (UI 表示モデル - 完全復元)

## 18.1 Installed Games Display (インストール済み表示)
表示項目：保護ステータス（🟢 保護中 / 🟡 一部制限あり / 🔴 保護停止）、設定状態（標準設定 / カスタム設定）、バックアップ最終日時、容量、起動オプション（▶️ 通常起動 / 🛡️ 厳格モード起動）。

保護ステータスと設定状態は別軸で扱う。「カスタム設定」は保護異常を意味しない。任意のオプション機能をユーザーが意図的にOFFにしただけでは、保護ステータスを「一部制限あり」または「保護停止」に変更しない。「一部制限あり」は、対象となる保護経路の一部に実行上の制限・障害があり、残る保護経路が継続して動作している場合に限る。「保護停止」は対象となる保護経路が停止した場合に使用する。

## 18.2 Missing Games Display (行方不明ゲーム表示)
表示項目：旧配置場所、バックアップ利用可能状態、再配置案内、除外オプション。

---

# 19. Post-Crash Audit & Cleanup (電源断・BSoD 復旧)

異常終了時、次回起動時に以下を実行する：
- 未完了セッションの修復 (`IScanSessionRepository.RepairIncompleteSessionsAsync`)
- **孤立一時ルールの除去 (H-4 是正):** 厳格モード実行中に取り残された一時 Firewall ルール（`GST:STRICT-TEMP:`）を自動検出・削除。
- `WerBeforeState.json` からの WER 設定元値自動復元。

---

# 20. Audit Requirements (監査要件)

ライフサイクル変更時は必ず `AuditEventType` を付番して監査ログへ記録する：
- `ScanStarted`, `ScanCompleted`, `StrictLaunchActivated`, `SafeGameOnboarded`, `GameArchiveRestored`, `CrashProtectionActivated`

---

# 21. Relationship With Other Models (他モデルとの関係性 - 完全復元)

## 21.1 System Lifecycle
システム安全性およびライフサイクル調停実装：
```text
modules/03-01_SYS_System_Safety_and_Lifecycle.md
```

## 21.2 Game Environment Protection
ゲーム保護および起動モード仕様：
```text
03_Game_Environment_Protection_Model.md
```

## 21.3 Save Backup & Data Ownership
セーブバックアップおよびデータ所有権原則：
```text
05_Save_Backup_and_Data_Ownership_Model.md
```

## 21.4 PC Migration & Recovery
PC 移行モデルおよび再封緘：
```text
09_PC_Migration_and_Recovery_Model.md
```

## 21.5 Audit Model
監査証跡および改ざん検知：
```text
04_Audit_and_Evidence_Model.md
```

---

# 22. Final Lifecycle Statement

GST はゲームを所有・支配しない。  
GST はゲーム環境の状態とセッション全生命周期を深く理解し、安全に管理する。

---

Final Principle:

```text
Track State Reliably
Never Fail Silently Upon Monitoring Loss
Clean Up Post-Launch and Post-Crash
Respect User Ownership Always
```

---
