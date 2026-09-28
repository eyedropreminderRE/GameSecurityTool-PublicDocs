# 03-01: SYS - System Safety, Self-Integrity & Lifecycle Specification

**Document ID:** GST-MOD-SYS-001  
**Version:** 4.16
**Status:** Approved Module Specification  
**Target Projects:**
- `GameSecurityTool.Domain`
- `GameSecurityTool.Contracts`
- `GameSecurityTool.Application`
- `GameSecurityTool.Infrastructure`

> **Lifecycle note:** This module specification describes approved design scope. Its presence does not by itself indicate that the module is implemented, Windows-verified, or released.

---

# 1. モジュール概要 ＆ 責任境界

本モジュールは、GameSecurityTool（GST）の**自己完全性検証、クラッシュリカバリ、安全モード自律復旧、ゲームの全ライフサイクル調停、クラッシュダンプ外部送信遮断、プロセス監視健全性可視化（C-6 是正）、Anti-Wiper 子プロセス未然遮断、および完全クリーンアンインストール（Graceful Teardown）**を一元管理する最上位安全基盤である。

### 保証する範囲:
- 起動時のバイナリ改ざん検知（Authenticode + SHA256）および DLL ハイジャック防御（`SetDefaultDllDirectories`）
- OS 設定変更直前の WAL (Write-Ahead Logging) 方式変更ジャーナルと逆順自動ロールバック
- 通常 DB 破損時における独立リソース `RecoveryConfig.json` からのセーフモード復旧
- ゲーム未起動時 CPU 0% と、起動〜終了〜クラッシュ時の完全防衛シーケンス
- `WerFault.exe` によるメモリダンプ・実行パスの外部送信遮断と、ゲーム終了時の BeforeState 元値復元（ディスク永続化によるクラッシュ耐性保証）
- アンインストール時の三層データ分離（OS設定復元・隔離ファイル自動救済・内部DB/WAL消去・セーブデータ保護）
- **WMI プロセス監視障害時のサイレント Fail-Open 根絶 ＆ 保護ステータス可視化（C-6 是正）**
- **WMI 受動監視の完全な障害分離と標準エスケープ（H1 是正 / FN-SYS-07）**

### Clean 5-Layer レイヤー境界:
- **Domain Layer (`GST.Domain`):** 自己整合性ルール、ライフサイクル状態モデル。OS 依存ゼロ。
- **Contracts Layer (`GST.Contracts`):** `IProcessLifecycleWatcher`, `IWiperChildProcessGuard`, `ICrashReportTelemetryBlocker`, `ISelfIntegrityVerifier`, `IFullSystemReversionService`, `ISystemHealthCheck`, `IScanSessionRepository`, `IGameProfileLookupService`, `IDbWriteQueue` 等の Port インターフェースおよび不変 DTO 群（外部依存ゼロ）。
- **Application Layer (`GST.Application`):** `GameLifecycleProtectionManager`（監視健全性調停、UI ステータス伝播、自律再初期化リトライ、純粋ユースケース調停 BackgroundService）。
- **Infrastructure Layer (`GST.Infrastructure`):** `WmiProcessLifecycleWatcher`（WQL標準二重引用符エスケープ ＆ 障害分離 ＆ ヘルス可視化）、`Win32SecurityNativeMethods` (`[LibraryImport]`)、`CrashReportTelemetryBlocker`（ディスク永続化 BeforeState 復元）、`FullSystemReversionService`（隔離救済 ＆ 段階的ティアダウン）、`SqliteScanSessionRepository`（`IDbWriteQueue` 直列化）。

---

# 2. 機能要件仕様 (Functional Requirements)

## 2.1 `FN-SYS-01`: アプリケーション二段階自己完全性検証 ＆ DLLハイジャック防御
* **目的:** アプリ本体の改ざん・感染および同一フォルダ偽装 DLL による攻撃を OS 起動最前列で遮断する。
* **処理フロー:**
  1. `App.xaml.cs` の最前列で Win32 `SetDefaultDllDirectories(LOAD_LIBRARY_SEARCH_SYSTEM32)` を呼び出し、カレントディレクトリからの DLL 読み込みを拒否。
  2. `WinVerifyTrust` Win32 API を呼び出し、自バイナリの Authenticode 署名を検証。
  3. 署名検証成功後、バイナリ全体の SHA256 ハッシュを計算し、コンパイル時組み込み期待値と突合。
  4. 検証失敗時は即座に通常起動を中断し、独立した `GameSecurityTool.Recovery.exe` Recovery Host へ誘導する。

## 2.2 `FN-SYS-02`: システム変更記録 ＆ WAL 方式クラッシュリカバリ
* **目的:** Firewall 設定やレジストリ変更中の突然の電源断・クラッシュによる不整合を防止する。
* **処理フロー:**
  1. OS 変更直前に、変更前後の状態（JSON）と `RollbackOrder` をディスク上の `ChangeJournal.log` へ書き込み、`Flush(flushToDisk: true)` を実行。
  2. OS 変更を実行し、成功後に `IDbWriteQueue` 経由で SQLite の `SystemChangeRecord` へコミットしてジャーナルを完了マーク。
  3. 起動時、未完了レコードを発見した場合は `RollbackOrder` 降順（逆順）で自動ロールバックを実行。

## 2.3 `FN-SYS-03`: 独立 Recovery Host 自律復旧
* **目的:** データベース損壊、通常設定破損、または通常アプリの起動不能時でも、通常 DB / UI に依存しない最小経路でシステムを救済する。
* **権威artifact:** Recovery Host は専用の `GameSecurityTool.Recovery.exe` とし、復旧設定の正本は Recovery Host assembly に埋め込まれた `RecoveryConfig.json` とする。
* **独立性:** Recovery Host は通常 SQLite DB、通常設定ストア、MainWindow、通常機能初期化に依存してはならない。通常アプリが起動できない場合でもユーザーが直接起動できる。
* **通常起動からの遷移:** startup health check（`PRAGMA quick_check;` 等）または self-integrity 検証で blocking failure を検出した場合、通常 `MainWindow` を起動せず Recovery Host へ遷移する。
* **RecoveryConfig 検証:** embedded resource の欠落、読込失敗、JSON/schema 不正、integrity 検証失敗は fail-closed。外部 mutable file、Windows default、推測ポリシーによる代替は禁止。
* **復旧操作:** Recovery Host は既承認の WER 復元、GST-owned Firewall cleanup/recovery、approved design decision uninstall transaction recovery、DB isolation/reset 等の最小 recovery scope のみ提供し、`Inspect → User Confirm → Recover → Verify` を必須とする。
* **自己完全性境界:** Recovery Host 自身の必要な integrity/configuration 検証に失敗した場合は通常 recovery operation を開始しない。approved design decision は完全に改ざんされた配布物からの復旧保証を追加しない。

## 2.4 `FN-SYS-04`: ゲーム全生命周期保護マネージャー (`GameLifecycleProtectionManager`)
* **目的:** 未起動時リソース 0% と、起動〜実行〜終了〜クラッシュ時の自動清掃を両立する。
* **状態遷移仕様:**
  - **未起動時 (待機):** 全監視エンジン完全停止（0% / OFF）。`IProcessLifecycleWatcher` からの受動通知のみ待機。
  - **ゲーム起動時:** `ProcessStarted` イベント検知 ➔ `WerFault` 遮断適用 ➔ 起動前スナップショット採取 ➔ 監視エンジン ON。
  - **正常終了/タスクキル時:** `ProcessStopped` イベント検知 ➔ 即時ポスト監査（ドロップファイル・レジストリ改ざん清掃） ➔ 監視エンジン OFF。
  - **PC 電源断/BSoD 時:** 次回ツール起動時、残留状態を検知し遅延ポスト・クラッシュ監査を実行。この際、**クラッシュによって OS 上に孤立した Strict Safe Launch 用の一時 Firewall ルール（`GST:STRICT-TEMP:`）を自動検知し、安全にクリーンアップする**ことでネットワーク遮断の永続化を防ぐ。
  - **監視エンジン停止時の安全仕様 (C-6 是正):** WMI ウォッチャーの停止を検知した際、ダッシュボードおよびオーバーレイへ「⚠️ プロセス監視エンジン停止（保護機能が無効）」を即時通知し、30 秒周期で自動再初期化を試行する。

## 2.5 `FN-SYS-05`: クラッシュレポート (`WerFault.exe`) テレメトリ遮断 ＆ ディスク永続化 BeforeState 復元
* **目的:** ゲームクラッシュ時に OS がメモリダンプや実行パスを外部送信するのを防止し、終了時に元の OS 設定を確実に復元する。
* **処理仕様 (approved design decision):**
  1. 初回保護適用時、`HKCU\Software\Microsoft\Windows\Windows Error Reporting` の既存設定を the local GST WER before-state journal（primary）と `WerBeforeState.recovery.json`（recovery copy）へ同一内容で退避する。両journalの確実な書き込み／置換順序はapproved design decisionのdurability契約に従い、両方の保存完了前にWERを変更しない。
  2. Journal はschema/version、必須フィールド、明示的なvalue presence、deterministic fingerprintで検証可能な形式とする。
  3. `DontSendAdditionalData = 1`, `Logging = 0` を適用。
  4. セッション参照カウント（Active Protection Count）を増減管理。
  5. カウントが 0 に達した際、valid なjournalを選択して **「変更前の元の値」へ正確に書き戻す**。片方が欠損／破損していてももう片方がvalidなら復元可能。両方がinvalidまたは内容不一致ならfail-closedとし、Windows既定値へのfallbackやcurrent WER値の推測削除を行わない。
  6. 両journalが存在しない場合は `NoOutstandingJournal` として WER を変更せず、復元成功後にのみjournalを削除する。cleanup失敗時はartifactを保持して再試行する。

## 2.6 `FN-SYS-06`: 完全クリーンアンインストール保証 (`FullSystemReversionService`)
* **目的:** アンインストール時に OS を導入前の状態へ復元し、未復元ファイルの救済とファイルロック競合のない完全消去を行う。
* **三層分離アーキテクチャ ＆ 段階的ティアダウン:**
  - **Step 1 (隔離ファイル救済):** 未復元の隔離ファイルを元の場所へ自動復元。
  - **Step 2 (OS/システム設定復元):** approved design decisionの検証済みBeforeStateからWERレジストリ設定を元値へ復元。Windows既定値へのfallbackは禁止し、RecoveryBlocked / RecoveryFailed の場合はrecovery artifactを保持して完全アンインストール成功とは扱わない。
  - **Step 3 (Firewall ルール一括消去):** OS 上の `GST:FIREWALL:` プレフィックスを持つ全ルールを走査消去。
  - **Step 4 (接続プール完全切断):** `SqliteConnection.ClearAllPools()` ➔ ガベージコレクション同期実行。
  - **Step 5 (内部データ消去):** ファイル属性（ReadOnly 等）を解除してアプリ内部ディレクトリを安全消去。
  - **ユーザー資産保護:** セーブデータの Standard ZIP バックアップは削除せず PC 上に保持。

## 2.7 `FN-SYS-07`: WMI 標準エスケープ ＆ 単一障害点排除 (H1 是正)
* **目的:** 不正なファイル名やアポストロフィを含むゲーム名によって監視クエリが構文エラーを起こし、全監視が巻き添え停止する事故を物理排除する。
* **処理仕様:**
  1. **標準エスケープ規約:** WQL の文字列リテラル構文に従い、バックスラッシュは `\\`、シングルクォートは `''`（二重引用符化）へエスケープする。
  2. **障害分離規約:** 不正な名前（空文字、制御文字、`.exe` 以外の拡張子等）を持つプロセス名はクエリ構築段階で安全に除外し、有効なプロセス名のみでクエリを構築する。
  3. **可視化規約:** 万が一 Watcher の起動に失敗した場合はサイレントに放置せず、`IsOperational = false` と `WatcherFaulted` イベントを通じてヘルス状態を可視化する。

## 2.8 `FN-SYS-08`: ポスト実行監査 ＆ 持続化改ざん検知・ランチャー越境改ざん監視
* **目的:** ゲーム終了時に、ゲームプロセスやスクリプトが OS に残した永続化設定（Runキー、hosts）および他ランチャーへの改ざんを自動検知し、安全に復元する。
* **処理仕様:**
  1. `IRegistryPersistenceTracker.AuditHostsAndRunKeysAsync` を呼び出し、`HKCU`/`HKLM` の `Run`/`RunOnce` キーへの不審エントリ追加、および `%SystemRoot%\System32\drivers\etc\hosts` への不審リダイレクト（公式ゲームサーバーの localhost 転送等）を検知。ワンクリック復元手段を提供する。
  2. `ILauncherSecurityAdapter.AuditLauncherIntegrityAsync` を呼び出し、Steam, Epic Games, EA 等の公式ランチャー本体フォルダ（`Steam.exe`、主要 DLL、設定）に対する不正書き換えや横展開侵入を検知。

## 2.9 `FN-SYS-09`: OS 自動起動（Run キー）ガバナンス
* **目的:** GST 本体の OS ログオン時自動起動をユーザーの明示的意思の下で安全に管理する。
* **処理仕様:**
  1. **既定 OFF (オプトイン):** 出荷時状態およびインストール直後は自動起動を登録しない。
  2. **ユーザー明示設定:** 設定画面でのチェックボックス ON/OFF に連動して `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` への登録・削除を行う。
  3. **アンインストール時自動解除:** `FullSystemReversionService` によるアンインストール時に当該 Run キーエントリを確実に自動削除する。

## 2.10 `FN-SYS-10`: 【確定】電源状態適応 ＆ スリープ復帰時ゴーストプロセス狩り (`WindowsPowerStateService`)
* **目的:** OS スリープ復帰時の WMI 不感症を解消し、スリープ中に異常終了したゲームプロセスを検知して即時ポスト監査・復元を行う。
* **処理仕様:**
  1. `IPowerStateService.SystemResumed` イベント契機で既存の `ManagementEventWatcher` を破棄・再生成。
  2. 管理中セッションの PID 一覧を検証し、プロセスが消滅していた場合は即座にポスト監査および WER / FW の元値復元を実行。
  3. 外付けドライブの再認識安定化のため 3 秒待機。

## 2.11 `FN-SYS-11`: 【確定】稼働中ゲーム能動アタッチ (`IngestRunningGamesAsync`)
* **目的:** GST 起動前にすでに開始されていたゲームプロセスを捕捉し、保護セッションへ途中合流させる。
* **処理仕様:**
  1. GST 起動時に `Process.GetProcesses()` を走査し、登録済みゲームと照合。
  2. 合致する未追跡ゲームに対し `IsMidSessionIngested = true` でセッションを開始し、FW ルール適用、WER 抑止、スナップショット採取を適用。

## 2.12 `FN-SYS-12`: 【確定】OS 急所ファイル（hosts 等）整合性検証 ＆ アトミック復元 (`HighValueTargetIntegrityVerifier`)
* **目的:** DNS 偽装や管理者シェル悪用による不正持続化を看破・自動復旧する。
* **処理仕様:**
  1. ゲーム起動前に `hosts` ファイルおよび PowerShell プロファイルの SHA-256 ハッシュを記録し、ベースラインをジャーナル退避。
  2. ゲーム終了時のポスト監査でハッシュ不一致を検知した場合、ジャーナルから元値へアトミック復元（`hosts.baseline` ➔ `hosts`）。

## 2.13 `FN-SYS-13`: 【確定】Anti-Wiper 子プロセス未然遮断 (`Win32_ProcessStartTrace`)
* **目的:** 監視対象ゲームが起点となって起動したシェル/管理ツールが、個人資産領域の破壊的コマンドを実行するケースを、ファイル変更イベントの発生前に検知して子プロセスを終了対象とする。
* **監視ソース:** 既存 `WmiProcessLifecycleWatcher` と同一の WMI イベント基盤を使用し、`Win32_ProcessStartTrace` の `ProcessId` / `ParentProcessId` / `ProcessName` を受動取得する。
* **判定契約:** `ParentProcessId == ActiveGamePid` を必須とし、登録時のゲームプロセス開始時刻と現在のPID世代も一致することを確認した上で、`cmd.exe` / `powershell.exe` / `pwsh.exe` / `wscript.exe` / `cscript.exe` / `vssadmin.exe` / `wbadmin.exe` 等の許可リスト対象ホストについて、**`Win32_ProcessStartTrace` 受信後に対象PIDを別途照会して取得したコマンドライン**に `del` / `erase` / `rmdir` / `Remove-Item` / `delete shadows` / `delete catalog` / `cipher /w` / `copy /y nul` 等の破壊的トークンが存在する場合のみ遮断する。イベント自身のプロパティだけからコマンドラインを得たものとして扱ってはならない。取得不能・PID再利用疑い・検証不一致時は遮断せず監査記録＋通知へフォールバックする。検知理由は `ThreatReasonCodes.WiperChildProcessDetected` を使用する。
* **制御:** 明示的に有効化された Anti-Wiper 保護セッション内でのみ `OpenProcess(PROCESS_TERMINATE)` ➔ `TerminateProcess` の限定操作を行う。アクセス拒否・権限不足時にUAC昇格を自動要求せず、監査記録＋通知へフォールバックする。ゲームプロセス自身、ゲームフォルダ、正規セーブ領域を直接監視・改変しない。
* **タイミング上の境界:** WMIイベント通知はイベント駆動であり、子プロセスの最初の命令実行やOS内部処理より前に必ず介入できるとは保証しない。したがって「0秒」「100%事前防止」は結果保証ではなく、設計目標として扱う。
* **アーキテクチャ境界:** WMI購読、コマンドライン取得、Win32プロセス制御はInfrastructure層に閉じ込め、Application層へ `IWiperChildProcessGuard` 等のPortを介して通知する。Domain層にはOS API、WMI、Contracts型を依存させない。

## 2.14 `FN-SYS-14`: 【確定】Anti-Wiper Circuit Breaker 調停
* **既定状態:** `Anti-Wiper Defense & Auto-Circuit Breaker` は既定OFF。明示的に有効化された場合のみ保護監視を開始する。
* **イベント検知:** 保護対象のユーザー資産領域を `FileSystemWatcher` 等のOS変更通知機構で監視し、削除・変更・リネームのイベントを100msスライディングウィンドウで集約する。バーストしきい値は重複イベントの増幅を避けるため、**異なるファイルパス数**を基準に評価する。
* **しきい値:** 設定された `BurstThresholdCount`（標準候補3件）以上の短時間変更、またはCanary変更/削除をCircuit Breaker発動条件とする。削除バーストは `ThreatReasonCodes.WiperBurstDeletionDetected`、変更バーストは `ThreatReasonCodes.WiperBurstModificationDetected`、Canaryトリガーは `ThreatReasonCodes.WiperCanaryTriggered` を使用する。
* **帰属境界:** `FileSystemWatcher` / `ReadDirectoryChangesW` のイベントには発生元PIDが含まれないため、Burst / Canary はゲーム起点を証明するものではなく、明示的に有効化された保護セッション中のユーザー資産領域に対する異常シグナルとして扱う。
* **保護境界:** ゲームインストール領域、正規セーブ領域、TEMP領域、および明示的な除外パスはバースト判定から除外する。除外は信頼判定ではなく監視対象から外す処理契約である。
* **アクション:** ブレーカー発動時は、**単一のActiveProtectionSessionに登録された監視対象ゲーム1つ**を対象とし、既定で `NtSuspendProcess` により一時停止してユーザーへ監査情報を提示した後、再開または `TerminateProcess` を選択できる。複数ゲームが登録されている／発生元を帰属できない状態ではプロセス制御を実行せず、監査記録＋通知へフォールバックする。権限不足・競合時も同様に扱う。
* **Operational Truth / Watcher Fault (approved design decision):** Anti-Wiper有効時は、設定された全保護rootのWatcher成立と、Canary有効時の各rootにおけるGST所有Canary成立を満たす場合のみ `IsOperational=true` とする。既存同名ファイル衝突、root欠落、Watcher不成立、Canary不成立は非稼働として扱う。Watcher障害時は残存Watcherも停止し、ファイルイベント起点の自動プロセス制御を停止する。処理中イベントも制御直前にOperational状態を再確認し、非稼働なら中止する。
* **Operational State Projection (approved design decision):** `OperationalStateChanged` を既存のChild Guard / Circuit Breaker / Coordinator Portで伝播し、Presentationへ保護状態を反映する。Infrastructure状態をPresentationが直接参照してはならない。
* **Manual Terminate Confirmation (approved design decision):** WiperBreakerAlertDialogからの手動強制終了は、最初のクリックを確認待ちとし、10秒以内の2回目のクリックでのみ既存のPID + StartTime検証済みTerminateを実行する。自動ブレーカーの明示設定による `TerminateProcess` 経路は変更しない。
* **Watcher Recovery (approved design decision):** Watcher障害後は `IsOperational=false` を維持したまま既存のWatcher/Canary初期化経路のみを自動再実行する。再試行は2秒→5秒→15秒→30秒のbounded backoffを使用し、設定変更・Disposeでキャンセルする。すべての設定rootのWatcher成立と必要なGST所有Canary成立が確認できた場合のみOperationalへ戻す。復旧経路で新しい検知・破壊能力を追加してはならない。
* **Canary連携:** `!0_gst_canary.dat` のイベントは通常バーストとは独立した早期トリガーとして処理する。Canary自身はScanner/Backupから除外し、撤収は `FullSystemReversionService` と共通の管理記録を使用する。
* **限界:** OSイベント通知後の対応であるため、通知前に発生したファイル変更を取り消すことや、被害を必ずゼロにすることは保証しない。

---

# 3. Contracts Layer 定義 (`GameSecurityTool.Contracts`)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Defense;
using GameSecurityTool.Contracts.Dtos;

public interface IProcessLifecycleWatcher : IDisposable
{
    event Action<uint, string>? ProcessStarted;
    event Action<uint, string>? ProcessStopped;
    event Action<string>? WatcherFaulted; // 監視障害イベント

    bool IsOperational { get; }          // 監視正常稼働ステータス

    void Start();
    void Stop();

    Task UpdateMonitoredProcessesAsync(IReadOnlyList<string> processNames, CancellationToken ct = default);
}

/// <summary>
/// Anti-Wiper のゲーム起点子プロセス未然遮断 Port。
/// WMI購読・コマンドライン取得・Win32 TerminateProcess は Infrastructure 側で実装し、
/// Application/Domain へ OS 依存を漏らさない。
/// </summary>
public interface IWiperChildProcessGuard : IDisposable
{
    /// <summary>Anti-Wiper設定を更新し、監視状態へ反映する。</summary>
    void UpdateConfig(WiperDefenseConfigDto config);
    /// <summary>ゲームPIDと開始時刻を1組のプロセス世代として登録する。</summary>
    bool RegisterMonitoredGame(uint processId, DateTimeOffset processStartTimeUtc);
    void UnregisterMonitoredGame(uint processId);
    bool IsOperational { get; }
    event Action<bool>? OperationalStateChanged;
    event Action<uint, string>? DestructiveChildProcessBlocked;
    /// <summary>破壊的子プロセス評価結果。終了成功・終了失敗・確定不能を含めて通知する。</summary>
    event Action<WiperChildProcessControlResultDto>? ChildProcessControlEvaluated;
}

public interface IWiperCircuitBreakerService : IDisposable
{
    void UpdateConfig(WiperDefenseConfigDto config);
    bool RegisterActiveGameProcess(uint processId, string gameFolderPath);
    void UnregisterActiveGameProcess(uint processId);
    /// <summary>UIから渡された登録時PID世代と一致する場合のみ、Suspend済みプロセスを再開する。</summary>
    Task<bool> ResumeSuspendedProcessAsync(uint processId, DateTimeOffset expectedProcessStartTimeUtc, CancellationToken ct = default);
    /// <summary>UIから渡された登録時PID世代と一致する場合のみ、対象プロセスを終了する。</summary>
    Task<bool> TerminateProcessAsync(uint processId, DateTimeOffset expectedProcessStartTimeUtc, CancellationToken ct = default);
    bool IsOperational { get; }
    event Action<bool>? OperationalStateChanged;
    event Action<WiperBreakerTriggeredEventDto>? BreakerTriggered;
}

public interface ISystemHealthCheck
{
    Task<SystemHealthStatusDto> ValidateSystemStateAsync(CancellationToken ct = default);
}

public interface IScanSessionRepository
{
    Task<IReadOnlyList<ScanSessionSummaryDto>> GetIncompleteSessionsAsync(CancellationToken ct = default);
    Task RepairIncompleteSessionsAsync(DateTimeOffset finishedAt, CancellationToken ct = default);
}

public interface IGameProfileLookupService
{
    Task<GameFolderDto?> FindMatchingGameByProcessNameAsync(string processName, CancellationToken ct = default);
}

public interface ISelfIntegrityVerifier
{
    bool VerifyApplicationIntegrity(string executablePath);
}

public interface ICrashReportTelemetryBlocker
{
    void EnableCrashReportProtection();
    WerRecoveryResultDto RestoreOriginalCrashReportSettings();
    WerRecoveryResultDto RecoverOrphanedProtectionFlags();
}

public interface IFullSystemReversionService
{
    Task<bool> ExecuteFullSystemReversionAsync(bool restoreQuarantinedFiles = true, CancellationToken ct = default);
}

public interface IDroppedArtifactTracker
{
    Dictionary<string, DateTimeOffset> CapturePreLaunchSnapshot();
    IReadOnlyList<FileRiskResultDto> DetectDroppedArtifacts(string gameFolderPath, Dictionary<string, DateTimeOffset> preLaunchSnapshot);
}

public interface IRegistryPersistenceTracker
{
    Dictionary<string, string> CaptureRegistrySnapshot();
    List<string> DetectAndRollbackRegistryChanges(Dictionary<string, string> preLaunchSnapshot);
    Task<PersistenceAuditResultDto> AuditHostsAndRunKeysAsync(CancellationToken ct = default);
    Task<bool> RemediatePersistenceEntryAsync(string entryKey, CancellationToken ct = default);
}

public interface ILauncherSecurityAdapter
{
    Task<LauncherIntegrityReportDto> AuditLauncherIntegrityAsync(CancellationToken ct = default);
    Task<bool> VerifyExecutableIntegrityAsync(string launcherExecutablePath, CancellationToken ct = default);
}

public interface ILauncherBoundaryTamperingDetector
{
    Task<bool> DetectCrossBoundaryTamperingAsync(string gameProcessPath, string launcherInstallPath, CancellationToken ct = default);
}

public interface IPowerStateService : IDisposable
{
    event Action? SystemSuspended;
    event Action? SystemResumed;
    bool IsSystemSuspended { get; }
}

public interface IStorageResilienceProvider
{
    Task<T> ExecuteWithSpinupToleranceAsync<T>(Func<Task<T>> operation, string targetDrivePath, CancellationToken ct = default);
    Task PreWakeupProbeAsync(string targetDrivePath, CancellationToken ct = default);
}

public interface IHighValueTargetIntegrityVerifier
{
    Task CaptureBaselineSnapshotsAsync(CancellationToken ct = default);
    Task<IReadOnlyList<SystemFileTamperResultDto>> VerifyAndRestoreIntegrityAsync(CancellationToken ct = default);
}
```

```csharp
namespace GameSecurityTool.Contracts.Defense;

using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

public enum BreakerActionType
{
    SuspendProcess,
    TerminateProcess
}

public sealed record WiperDefenseConfigDto(
    bool IsEnabled,
    BreakerActionType ActionType,
    int BurstThresholdCount,
    int BurstWindowMilliseconds,
    bool EnableCanaryTrap,
    bool ProtectVssAdmin,
    IReadOnlyList<string> ProtectedRootDirectories
);

public sealed record WiperBreakerTriggeredEventDto(
    uint OffendingProcessId,
    string ProcessName,
    string BreakerReason,
    IReadOnlyList<string> AffectedFilePaths,
    DateTimeOffset TriggeredAtUtc,
    DateTimeOffset OffendingProcessStartTimeUtc
);
```

```csharp
namespace GameSecurityTool.Contracts.Dtos;

using System;
using System.Collections.Generic;

public sealed record SystemFileTamperResultDto(
    string TargetFilePath,
    bool IsTampered,
    string ExpectedSha256,
    string ActualSha256,
    bool IsRestoredSuccessfully,
    string? Details
);

public sealed record PersistenceAuditResultDto(
    bool HasTampering,
    IReadOnlyList<string> TamperedRunKeys,
    IReadOnlyList<string> SuspiciousHostsEntries
);

public sealed record LauncherIntegrityReportDto(
    bool IsIntact,
    IReadOnlyList<string> CompromisedLaunchers,
    string? Details
);

public sealed record SystemHealthStatusDto(
    bool IsOverallHealthy,
    bool IsProcessWatcherOperational,
    bool IsDatabaseHealthy,
    bool IsLayer2AuditSynchronized,
    string? HealthWarningMessage
);

public sealed record SystemChangeRecordDto(
    Guid Id,
    string OperationId,
    int RollbackOrder,
    string ChangeType,
    string Target,
    string? BeforeStateJson,
    string? AfterStateJson,
    string Status,
    string? RollbackError,
    DateTimeOffset CreatedAt
);

public sealed record ActiveGameSessionDto(
    uint ProcessId,
    string ProcessName,
    Guid GameFolderId,
    string GameFolderPath,
    DateTimeOffset StartedAt
);

public sealed record GameFolderDto(
    Guid Id,
    string DisplayName,
    string Path,
    string MainExecutable
);

public sealed record ScanSessionSummaryDto(
    Guid SessionId,
    Guid GameFolderId,
    DateTimeOffset StartedAt
);
```

---

# 4. Infrastructure Layer 完全実装 (`GameSecurityTool.Infrastructure`)

## 4.1 WMI 受動プロセストレース Adapter (`WmiProcessLifecycleWatcher.cs` - H1 ＆ C-6 是正)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Native;

using System;
using System.Collections.Generic;
using System.Linq;
using System.Management;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.Extensions.Logging;

public sealed class WmiProcessLifecycleWatcher(ILogger<WmiProcessLifecycleWatcher> logger) : IProcessLifecycleWatcher
{
    private ManagementEventWatcher? _startWatcher;
    private ManagementEventWatcher? _stopWatcher;
    private IReadOnlyList<string> _monitoredProcesses = [];

    public event Action<uint, string>? ProcessStarted;
    public event Action<uint, string>? ProcessStopped;
    public event Action<string>? WatcherFaulted;

    public bool IsOperational { get; private set; } = true;

    public void Start() => RestartWatchers();

    public void Stop()
    {
        try
        {
            if (_startWatcher != null)
            {
                _startWatcher.Stop();
                _startWatcher.Dispose();
                _startWatcher = null;
            }
            if (_stopWatcher != null)
            {
                _stopWatcher.Stop();
                _stopWatcher.Dispose();
                _stopWatcher = null;
            }
        }
        catch (Exception ex)
        {
            logger.LogTrace(ex, "WMI ウォッチャー停止時のクリーンアップ");
        }
    }

    public Task UpdateMonitoredProcessesAsync(IReadOnlyList<string> processNames, CancellationToken ct = default)
    {
        _monitoredProcesses = processNames ?? [];
        RestartWatchers();
        return Task.CompletedTask;
    }

    private void RestartWatchers()
    {
        Stop();

        // 監視対象プロセス名のサニタイズおよび検証 (FN-SYS-07)
        var validProcessNames = _monitoredProcesses
            .Where(p => !string.IsNullOrWhiteSpace(p) && p.EndsWith(".exe", StringComparison.OrdinalIgnoreCase))
            .Distinct(StringComparer.OrdinalIgnoreCase)
            .ToList();

        if (validProcessNames.Count == 0)
        {
            logger.LogInformation("WMI 監視対象プロセスが 0 件のため待機状態に入ります。");
            IsOperational = true;
            return;
        }

        try
        {
            // 【FN-SYS-07 / H1 是正】WQL 標準エスケープ (バックスラッシュ -> \\, シングルクォート -> '')
            var filterClauses = validProcessNames.Select(p =>
            {
                string escaped = p.Replace("\\", "\\\\").Replace("'", "''");
                return $"ProcessName = '{escaped}'";
            });

            string processFilter = string.Join(" OR ", filterClauses);

            var startQuery = new WqlEventQuery($"SELECT * FROM Win32_ProcessStartTrace WHERE {processFilter}");
            _startWatcher = new ManagementEventWatcher(startQuery);
            _startWatcher.EventArrived += (s, e) =>
            {
                if (e.NewEvent.Properties["ProcessId"]?.Value != null && e.NewEvent.Properties["ProcessName"]?.Value != null)
                {
                    uint pid = Convert.ToUInt32(e.NewEvent["ProcessId"]);
                    string name = Convert.ToString(e.NewEvent["ProcessName"]) ?? string.Empty;
                    ProcessStarted?.Invoke(pid, name);
                }
            };
            _startWatcher.Start();

            var stopQuery = new WqlEventQuery($"SELECT * FROM Win32_ProcessStopTrace WHERE {processFilter}");
            _stopWatcher = new ManagementEventWatcher(stopQuery);
            _stopWatcher.EventArrived += (s, e) =>
            {
                if (e.NewEvent.Properties["ProcessId"]?.Value != null && e.NewEvent.Properties["ProcessName"]?.Value != null)
                {
                    uint pid = Convert.ToUInt32(e.NewEvent["ProcessId"]);
                    string name = Convert.ToString(e.NewEvent["ProcessName"]) ?? string.Empty;
                    ProcessStopped?.Invoke(pid, name);
                }
            };
            _stopWatcher.Start();

            IsOperational = true;
            logger.LogInformation("WMI プロセス受動監視ウォッチャーを対象限定 ({Count} 件) で開始しました。", validProcessNames.Count);
        }
        catch (Exception ex)
        {
            IsOperational = false;
            logger.LogError(ex, "WMI プロセストレースウォッチャーの開始に失敗しました。監視が低下しています。");
            WatcherFaulted?.Invoke($"WMI 監視ウォッチャーの起動に失敗しました: {ex.Message}");
        }
    }

    public void Dispose() => Stop();
}
```

## 4.2 クラッシュレポート遮断 ＆ ディスク永続化 BeforeState 復元 (\`CrashReportTelemetryBlocker.cs\`)

> **approved design decision reference contract:** WER recovery restores the exact pre-GST state. Windows default values are never a recovery source. This code block is a specification/reference implementation surface; production implementation remains gated. Durable dual-journal write/replace/delete sequencing is governed by approved design decision.

\`\`\`csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Native;

using System;
using System.IO;
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.Extensions.Logging;
using Microsoft.Win32;

public sealed class CrashReportTelemetryBlocker(ILogger<CrashReportTelemetryBlocker> logger) : ICrashReportTelemetryBlocker
{
    private const string WerRegistryPath = @"SOFTWARE\Microsoft\Windows\Windows Error Reporting";
    private const string PrimaryJournalFileName = "WerBeforeState.json";
    private const string RecoveryJournalFileName = "WerBeforeState.recovery.json";

    private static readonly string JournalDirectory = Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
        "GameSecurityTool"
    );

    private static readonly string PrimaryJournalPath =
        Path.Combine(JournalDirectory, PrimaryJournalFileName);

    private static readonly string RecoveryJournalPath =
        Path.Combine(JournalDirectory, RecoveryJournalFileName);

    private const int JournalSchemaVersion = 1;

    private int _activeSessionCount = 0;
    private readonly object _lock = new();

    private sealed record WerRegistryValueState(
        bool Present,
        int? DwordValue
    );

    private sealed record WerStatePayload(
        int SchemaVersion,
        WerRegistryValueState DontSendAdditionalData,
        WerRegistryValueState Logging
    );

    private sealed record WerJournalEnvelope(
        int SchemaVersion,
        WerStatePayload State,
        string StateFingerprintSha256
    );

    public void EnableCrashReportProtection()
    {
        lock (_lock)
        {
            try
            {
                if (_activeSessionCount > 0)
                {
                    _activeSessionCount++;
                    return;
                }

                using var key = Registry.CurrentUser.CreateSubKey(WerRegistryPath, writable: true);
                if (key is null)
                {
                    logger.LogError("WER レジストリキーを開けないため、保護設定を変更せず中止しました。");
                    return;
                }

                var currentDontSend = key.GetValue("DontSendAdditionalData");
                var currentLogging = key.GetValue("Logging");

                if (currentDontSend is not null && currentDontSend is not int)
                {
                    logger.LogError("DontSendAdditionalData が想定外の型のため、保護設定を変更せず中止しました。");
                    return;
                }

                if (currentLogging is not null && currentLogging is not int)
                {
                    logger.LogError("Logging が想定外の型のため、保護設定を変更せず中止しました。");
                    return;
                }

                var state = new WerStatePayload(
                    JournalSchemaVersion,
                    new WerRegistryValueState(currentDontSend is int, currentDontSend as int?),
                    new WerRegistryValueState(currentLogging is int, currentLogging as int?)
                );

                PersistJournalPair(state);

                key.SetValue("DontSendAdditionalData", 1, RegistryValueKind.DWord);
                key.SetValue("Logging", 0, RegistryValueKind.DWord);

                _activeSessionCount = 1;
                logger.LogInformation("WER テレメトリ抑止を適用しました。BeforeState primary/recovery の双方を記録済みです。");
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "WER 保護設定の適用に失敗しました。OS設定は保護適用前の状態を維持する必要があります。");
                return;
            }
        }
    }

    public WerRecoveryResultDto RestoreOriginalCrashReportSettings()
    {
        lock (_lock)
        {
            if (_activeSessionCount > 0)
            {
                _activeSessionCount--;
            }

            if (_activeSessionCount > 0)
            {
                return new WerRecoveryResultDto(
                    WerRecoveryStatus.NoOutstandingJournal,
                    "他の保護セッションが継続中のため、最終復元は保留されています。");
            }

            return RestoreFromJournal();
        }
    }

    public WerRecoveryResultDto RecoverOrphanedProtectionFlags()
    {
        lock (_lock)
        {
            _activeSessionCount = 0;
            var result = RestoreFromJournal();

            if (result.Status == WerRecoveryStatus.Restored)
            {
                logger.LogInformation("異常終了後の WER 保護残留を元値へ復元しました。");
            }
            else if (result.Status is WerRecoveryStatus.RecoveryBlocked or WerRecoveryStatus.RecoveryFailed)
            {
                logger.LogError("異常終了後の WER 復元は fail-closed で停止しました。Recovery artifact を保持します。");
            }

            return result;
        }
    }

    private WerRecoveryResultDto RestoreFromJournal()
    {
        bool primaryExists = File.Exists(PrimaryJournalPath);
        bool recoveryExists = File.Exists(RecoveryJournalPath);

        if (!primaryExists && !recoveryExists)
        {
            return new WerRecoveryResultDto(
                WerRecoveryStatus.NoOutstandingJournal,
                "primary/recovery の両ジャーナルが存在しません。WER は変更しません。");
        }

        var primary = TryReadValidJournal(PrimaryJournalPath, primaryExists);
        var recovery = TryReadValidJournal(RecoveryJournalPath, recoveryExists);

        WerStatePayload? state = null;

        if (primary.IsValid && recovery.IsValid)
        {
            if (!string.Equals(primary.Fingerprint, recovery.Fingerprint, StringComparison.Ordinal))
            {
                return new WerRecoveryResultDto(
                    WerRecoveryStatus.RecoveryBlocked,
                    "primary/recovery journal の内容が一致しないため、復元元を一意に決定できません。");
            }

            state = primary.State;
        }
        else if (primary.IsValid)
        {
            state = primary.State;
        }
        else if (recovery.IsValid)
        {
            state = recovery.State;
        }
        else
        {
            return new WerRecoveryResultDto(
                WerRecoveryStatus.RecoveryBlocked,
                "有効な WER BeforeState recovery source が存在しません。");
        }

        try
        {
            using var key = Registry.CurrentUser.OpenSubKey(WerRegistryPath, writable: true);

            if (key is null)
            {
                if (state.DontSendAdditionalData.Present || state.Logging.Present)
                {
                    return new WerRecoveryResultDto(
                        WerRecoveryStatus.RecoveryFailed,
                        "WER レジストリキーが存在せず、記録済み元値を復元できませんでした。");
                }
            }
            else
            {
                RestoreValue(
                    key,
                    "DontSendAdditionalData",
                    state.DontSendAdditionalData
                );

                RestoreValue(
                    key,
                    "Logging",
                    state.Logging
                );
            }

            if (!TryDeleteJournalPair())
            {
                return new WerRecoveryResultDto(
                    WerRecoveryStatus.RecoveryFailed,
                    "WER 元値の復元は完了しましたが、recovery artifact の削除に失敗しました。artifact を保持して再試行可能です。");
            }

            logger.LogInformation("WER 設定を GST 導入前の元値へ復元しました。");
            return new WerRecoveryResultDto(WerRecoveryStatus.Restored, null);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "WER 元値の復元中にエラーが発生しました。");
            return new WerRecoveryResultDto(
                WerRecoveryStatus.RecoveryFailed,
                "WER レジストリ復元に失敗しました。現在値と recovery artifact を変更せず保持します。");
        }
    }

    private static void RestoreValue(
        RegistryKey key,
        string valueName,
        WerRegistryValueState state)
    {
        if (state.Present)
        {
            if (state.DwordValue is null)
            {
                throw new InvalidDataException($"Invalid BeforeState: {valueName} is Present but DwordValue is null.");
            }

            key.SetValue(valueName, state.DwordValue.Value, RegistryValueKind.DWord);
        }
        else
        {
            key.DeleteValue(valueName, throwOnMissingValue: false);
        }
    }

    private static void PersistJournalPair(WerStatePayload state)
    {
        if (File.Exists(PrimaryJournalPath) || File.Exists(RecoveryJournalPath))
        {
            throw new IOException("Existing WER recovery artifact detected; refusing to overwrite an outstanding recovery source.");
        }

        Directory.CreateDirectory(JournalDirectory);

        var envelope = new WerJournalEnvelope(
            JournalSchemaVersion,
            state,
            ComputeFingerprint(state));

        string json = JsonSerializer.Serialize(envelope);

        // approved design decision defines the durable write/replace protocol. Until then this reference code
        // expresses the required ordering contract: both copies must be persisted successfully
        // before any WER registry mutation is performed.
        File.WriteAllText(PrimaryJournalPath, json, Encoding.UTF8);
        File.WriteAllText(RecoveryJournalPath, json, Encoding.UTF8);
    }

    private static (bool IsValid, WerStatePayload? State, string? Fingerprint) TryReadValidJournal(
        string path,
        bool exists)
    {
        if (!exists)
        {
            return (false, null, null);
        }

        try
        {
            string json = File.ReadAllText(path, Encoding.UTF8);
            var envelope = JsonSerializer.Deserialize<WerJournalEnvelope>(json);

            if (envelope is null ||
                envelope.SchemaVersion != JournalSchemaVersion ||
                envelope.State.SchemaVersion != JournalSchemaVersion ||
                envelope.StateFingerprintSha256.Length != 64 ||
                !string.Equals(
                    envelope.StateFingerprintSha256,
                    ComputeFingerprint(envelope.State),
                    StringComparison.OrdinalIgnoreCase) ||
                !IsValidValueState(envelope.State.DontSendAdditionalData) ||
                !IsValidValueState(envelope.State.Logging))
            {
                return (false, null, null);
            }

            return (true, envelope.State, envelope.StateFingerprintSha256);
        }
        catch
        {
            return (false, null, null);
        }
    }

    private static bool IsValidValueState(WerRegistryValueState state) =>
        state.Present ? state.DwordValue.HasValue : !state.DwordValue.HasValue;

    private static string ComputeFingerprint(WerStatePayload state)
    {
        byte[] canonical = Encoding.UTF8.GetBytes(JsonSerializer.Serialize(state));
        return Convert.ToHexString(SHA256.HashData(canonical));
    }

    private static bool TryDeleteJournalPair()
    {
        try
        {
            if (File.Exists(PrimaryJournalPath))
            {
                File.Delete(PrimaryJournalPath);
            }

            if (File.Exists(RecoveryJournalPath))
            {
                File.Delete(RecoveryJournalPath);
            }

            return !File.Exists(PrimaryJournalPath) && !File.Exists(RecoveryJournalPath);
        }
        catch
        {
            return false;
        }
    }
}
\`\`\`

## 4.3 完全クリーンアンインストール Adapter (`FullSystemReversionService.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Persistence;

using System;
using System.IO;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Common;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Infrastructure.Persistence.Entities;
using Microsoft.Data.Sqlite;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;


## approved design decision — Full System Reversion Durable Transaction Contract

`FullSystemReversionService` shall treat uninstall as a dedicated durable transaction. The ordinary `ChangeJournal` is not the uninstall transaction authority.

### Normative State Order

```text
Prepared
→ QuarantineDrain
→ WERRestore
→ FirewallCleanup
→ ConnectionDrain
→ InternalCleanup
→ Committed
```

### Contract Rules

1. Before the first uninstall mutation, persist `UninstallTransaction.json` to recovery-safe storage outside the normal `%LocalAppData%\GameSecurityTool` deletion target and durably commit `Prepared`.
2. Persist a monotonic durable completion boundary after each successful uninstall step and before starting the next step.
3. Recovery uses forward reconciliation against actual OS state. A crash after a mutation but before its journal commit must be recoverable by re-observing the current state and either completing or safely re-marking the step.
4. All DB/quarantine-dependent recovery logic must finish before `InternalCleanup`. After `InternalCleanup`, recovery must not depend on the deleted GST database.
5. `WERRestore` is satisfied only through the approved design decision `RestoreOriginalCrashReportSettings()` semantics; Windows default WER values are never a fallback.
6. Missing, malformed, unsupported, or ambiguous uninstall journals fail closed and remain preserved.
7. `Committed` is the durable completion point. After `Committed`, journal deletion is cleanup-only; a deletion failure must never reopen earlier uninstall steps.
8. The uninstaller may not delete the recovery-safe transaction journal or report complete system reversion before durable `Committed`.
9. No new public Contracts Port/DTO is required solely for the journal persistence format. Public invocation/recovery-entry APIs remain subject to their own later change-control decision.
10. The design target is that GST-created OS state cannot remain permanently orphaned solely because the uninstall process crashed or lost power between transaction boundaries.

### Reference Implementation Status

Any earlier illustrative `FullSystemReversionService` sample lacking this durable transaction journal is **pre-approved reference material** and is not a conforming implementation. Production code must not be copied from that sample until the Phase 0 implementation gate is passed and the actual Infrastructure recovery implementation is reviewed against this contract.

public sealed class FullSystemReversionService(
    IDbContextFactory<AppDbContext> dbFactory,
    IDbWriteQueue dbWriter,
    IFirewallManager firewallManager,
    ICrashReportTelemetryBlocker crashBlocker,
    IQuarantineService quarantineService,
    ILogger<FullSystemReversionService> logger) : IFullSystemReversionService
{
    public async Task<bool> ExecuteFullSystemReversionAsync(bool restoreQuarantinedFiles = true, CancellationToken ct = default)
    {
        logger.LogInformation("【完全クリーンアンインストール】システムの完全復元シーケンスを開始します...");
        bool allSucceeded = true;

        try
        {
            if (restoreQuarantinedFiles)
            {
                try
                {
                    await using var db = await dbFactory.CreateDbContextAsync(ct);
                    var quarantined = await db.Set<QuarantineEntryRecord>()
                        .AsNoTracking()
                        .Where(q => q.RestoreStatus == 0)
                        .ToListAsync(ct);

                    foreach (var q in quarantined)
                    {
                        logger.LogInformation("アンインストールに伴い隔離ファイルを元の場所へ復元中: {Path}", q.OriginalPath);
                        await quarantineService.RestoreFileAsync(q.Id.ToString(), "UNINSTALL-DRAIN", ct);
                    }
                }
                catch (Exception ex)
                {
                    logger.LogWarning(ex, "隔離ファイルの自動復元中に警告が発生しました。");
                }
            }

            bool werRecoverySucceeded;
            try
            {
                var werResult = crashBlocker.RestoreOriginalCrashReportSettings();
                werRecoverySucceeded = werResult.Status is WerRecoveryStatus.Restored or WerRecoveryStatus.NoOutstandingJournal;

                if (!werRecoverySucceeded)
                {
                    allSucceeded = false;
                    logger.LogError("WER 元値の復元を完了できないため、recovery artifact を保持します。Status={Status}, Reason={Reason}",
                        werResult.Status, werResult.Reason);
                }
            }
            catch (Exception ex)
            {
                werRecoverySucceeded = false;
                allSucceeded = false;
                logger.LogError(ex, "WER 元値の復元に失敗しました。");
            }

            try
            {
                var osRules = await firewallManager.GetOsActualRulesAsync(ct);
                var gstRules = osRules.Where(r => r.RuleTag.StartsWith("GST:FIREWALL:", StringComparison.OrdinalIgnoreCase)).ToList();
                foreach (var rule in gstRules)
                {
                    await firewallManager.RemoveBlockRuleAsync(rule.RuleTag, ct);
                }
                logger.LogInformation("OS 上の GST Firewall ルール ({Count}件) をすべて消去しました。", gstRules.Count);
            }
            catch (Exception ex)
            {
                allSucceeded = false;
                logger.LogError(ex, "Firewall ルールの全削除に失敗しました。");
            }

            SqliteConnection.ClearAllPools();
            GC.Collect();
            GC.WaitForPendingFinalizers();

            string appDataDir = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "GameSecurityTool");
            string roamingDir = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), "GameSecurityTool");

            if (werRecoverySucceeded && allSucceeded)
            {
                DeleteDirectorySafe(appDataDir);
                DeleteDirectorySafe(roamingDir);
            }
            else
            {
                logger.LogWarning("完全復元条件を満たさないため、アプリケーションデータ領域と recovery artifact を保持します。");
            }

            if (allSucceeded)
            {
                logger.LogInformation("【完全クリーンアンインストール】OS はツール導入前の状態へ復元されました (セーブデータは保持)。");
            }
            else
            {
                logger.LogWarning("【完全クリーンアンインストール】未完了。OS/recovery artifact の一部を保持しています。");
            }

            return allSucceeded;
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "完全クリーンアンインストール処理中に致命的例外が発生しました。");
            return false;
        }
    }

    private static void DeleteDirectorySafe(string path)
    {
        if (!Directory.Exists(path)) return;
        try
        {
            foreach (var file in Directory.EnumerateFiles(path, "*.*", SearchOption.AllDirectories))
            {
                try
                {
                    File.SetAttributes(file, FileAttributes.Normal);
                    File.Delete(file);
                }
                catch { }
            }
            Directory.Delete(path, recursive: true);
        }
        catch { }
    }
}
```

---

# 5. Application Layer 完全実装 (`GameLifecycleProtectionManager.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Application.Services;

using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

public sealed class GameLifecycleProtectionManager : BackgroundService
{
    private readonly IProcessLifecycleWatcher _processWatcher;
    private readonly IGameProfileLookupService _gameLookupService;
    private readonly ICrashReportTelemetryBlocker _crashBlocker;
    private readonly IDroppedArtifactTracker _droppedArtifactTracker;
    private readonly IRegistryPersistenceTracker _registryTracker;
    private readonly IHighValueTargetIntegrityVerifier _hvtVerifier;
    private readonly IScanSessionRepository _scanSessionRepository;
    private readonly IFirewallManager _firewallManager;
    private readonly ITamperEvidentAuditLogger _auditLogger;
    private readonly IPowerStateService _powerStateService;
    private readonly IStorageResilienceProvider _storageResilience;
    private readonly ILogger<GameLifecycleProtectionManager> _logger;

    private readonly ConcurrentDictionary<uint, ActiveSessionState> _activeSessions = new();

    public bool IsMonitoringOperational => _processWatcher.IsOperational;
    public string? MonitoringDegradationReason { get; private set; }

    public event Action<bool, string?>? MonitoringStatusChanged;

    public GameLifecycleProtectionManager(
        IProcessLifecycleWatcher processWatcher,
        IGameProfileLookupService gameLookupService,
        ICrashReportTelemetryBlocker crashBlocker,
        IDroppedArtifactTracker droppedArtifactTracker,
        IRegistryPersistenceTracker registryTracker,
        IHighValueTargetIntegrityVerifier hvtVerifier,
        IScanSessionRepository scanSessionRepository,
        IFirewallManager firewallManager,
        ITamperEvidentAuditLogger auditLogger,
        IPowerStateService powerStateService,
        IStorageResilienceProvider storageResilience,
        ILogger<GameLifecycleProtectionManager> logger)
    {
        _processWatcher = processWatcher;
        _gameLookupService = gameLookupService;
        _crashBlocker = crashBlocker;
        _droppedArtifactTracker = droppedArtifactTracker;
        _registryTracker = registryTracker;
        _hvtVerifier = hvtVerifier;
        _scanSessionRepository = scanSessionRepository;
        _firewallManager = firewallManager;
        _auditLogger = auditLogger;
        _powerStateService = powerStateService;
        _storageResilience = storageResilience;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("ゲーム全生命周期保護マネージャーを初期化中 (レジリエンス強化版)...");

        // 1. 起動時ポスト・クラッシュ監査 (残留ルール・WER・未完了TX修復)
        _crashBlocker.RecoverOrphanedProtectionFlags();
        await ExecutePostCrashAuditAsync(stoppingToken);

        // 2. 電源イベントの購読
        _powerStateService.SystemResumed += OnSystemResumed;

        // 3. WMI 受動ウォッチャーの開始
        _processWatcher.ProcessStarted += OnProcessStarted;
        _processWatcher.ProcessStopped += OnProcessStopped;
        _processWatcher.WatcherFaulted += OnWatcherFaulted;
        _processWatcher.Start();

        // 4. 【確定】稼働中ゲームの能動的アタッチ (Mid-Session Ingestion)
        await IngestRunningGamesAsync(stoppingToken);

        _logger.LogInformation("ゲーム全生命周期保護マネージャーが待機状態に入りました (未起動時監視エンジン: 0% / OFF)。");

        // 自律再初期化リトライループ
        var retryTask = Task.Run(async () =>
        {
            while (!stoppingToken.IsCancellationRequested)
            {
                await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
                if (!_processWatcher.IsOperational && !_powerStateService.IsSystemSuspended)
                {
                    _logger.LogInformation("プロセス監視ウォッチャーの自律再初期化を試行します...");
                    _processWatcher.Start();
                    if (_processWatcher.IsOperational)
                    {
                        MonitoringDegradationReason = null;
                        MonitoringStatusChanged?.Invoke(true, null);
                        _logger.LogInformation("プロセス監視ウォッチャーが正常復帰しました。");
                    }
                }
            }
        }, stoppingToken);

        var tcs = new TaskCompletionSource();
        using (stoppingToken.Register(() => tcs.SetResult()))
        {
            await tcs.Task;
        }

        await retryTask;
        _processWatcher.Stop();
        _processWatcher.ProcessStarted -= OnProcessStarted;
        _processWatcher.ProcessStopped -= OnProcessStopped;
        _processWatcher.WatcherFaulted -= OnWatcherFaulted;
        _powerStateService.SystemResumed -= OnSystemResumed;
    }

    /// <summary>
    /// 【確定: スリープ復帰時レジリエンス】WMI 再初期化 ＆ ゴーストプロセス狩り
    /// </summary>
    private async void OnSystemResumed()
    {
        try
        {
            _logger.LogInformation("スリープ復帰処理を開始: WMI ウォッチャーの強制再購読を実施します...");
            _processWatcher.Stop();
            _processWatcher.Start();

            // 復帰後のプロセス生存一括確認 (ゴーストプロセス狩り)
            foreach (var (pid, session) in _activeSessions)
            {
                bool isAlive = false;
                try
                {
                    var proc = System.Diagnostics.Process.GetProcessById((int)pid);
                    isAlive = !proc.HasExited;
                }
                catch
                {
                    isAlive = false;
                }

                if (!isAlive)
                {
                    _logger.LogWarning("スリープ中にゲームプロセス消滅を検知しました: {Name} (PID: {PID})。即座にポスト監査を実行します。", session.ProcessName, pid);
                    if (_activeSessions.TryRemove(pid, out var deadSession))
                    {
                        await ExecutePostLaunchAuditAsync(deadSession);
                    }
                }
            }

            // 外付けドライブのマウント安定待機 (3秒)
            await Task.Delay(3000);
            _logger.LogInformation("スリープ復帰自律点検シーケンスが正常に完了しました。");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "スリープ復帰時自律点検中にエラーが発生しました。");
        }
    }

    /// <summary>
    /// 【確定: 途中アタッチ】GST 起動前にすでに動いていたゲームを捕捉・途中保護開始
    /// </summary>
    private async Task IngestRunningGamesAsync(CancellationToken ct)
    {
        try
        {
            _logger.LogInformation("現在稼働中のゲームプロセスの能動的照会中...");
            var processes = System.Diagnostics.Process.GetProcesses();

            foreach (var proc in processes)
            {
                ct.ThrowIfCancellationRequested();
                try
                {
                    string procName = proc.ProcessName + ".exe";
                    var matchingGame = await _gameLookupService.FindMatchingGameByProcessNameAsync(procName, ct);

                    if (matchingGame != null && !_activeSessions.ContainsKey((uint)proc.Id))
                    {
                        _logger.LogInformation("🎮 既に稼働中のゲームを検出しました: {Name} (PID: {PID})。途中保護を開始します。", procName, proc.Id);

                        _crashBlocker.EnableCrashReportProtection();
                        var fileSnapshot = _droppedArtifactTracker.CapturePreLaunchSnapshot();
                        var registrySnapshot = _registryTracker.CaptureRegistrySnapshot();

                        var session = new ActiveSessionState(
                            ProcessId: (uint)proc.Id,
                            ProcessName: procName,
                            GameFolderId: matchingGame.Id,
                            GameFolderPath: matchingGame.Path,
                            StartedAt: DateTimeOffset.UtcNow,
                            PreLaunchFileSnapshot: fileSnapshot,
                            PreLaunchRegistrySnapshot: registrySnapshot,
                            IsMidSessionIngested: true
                        );

                        _activeSessions.TryAdd((uint)proc.Id, session);
                    }
                }
                catch
                {
                    // アクセス拒否プロセス等は安全にスキップ
                }
            }
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "稼働中ゲームの能動的照会中に警告が発生しました。");
        }
    }

    private void OnWatcherFaulted(string errorMessage)
    {
        MonitoringDegradationReason = errorMessage;
        MonitoringStatusChanged?.Invoke(false, errorMessage);
        _logger.LogWarning("⚠️ プロセス監視エンジンが停止しました: {Msg}。保護機能が無効化されています。", errorMessage);
    }

    private async void OnProcessStarted(uint processId, string processName)
    {
        try
        {
            var matchingGame = await _gameLookupService.FindMatchingGameByProcessNameAsync(processName);
            if (matchingGame == null) return;

            _logger.LogInformation("🎮 保護対象ゲームの起動を検知しました: {Name} (PID: {PID})。保護エンジンをアクティブ化します (ON)。", processName, processId);

            _crashBlocker.EnableCrashReportProtection();

            var fileSnapshot = _droppedArtifactTracker.CapturePreLaunchSnapshot();
            var registrySnapshot = _registryTracker.CaptureRegistrySnapshot();

            var session = new ActiveSessionState(
                ProcessId: processId,
                ProcessName: processName,
                GameFolderId: matchingGame.Id,
                GameFolderPath: matchingGame.Path,
                StartedAt: DateTimeOffset.UtcNow,
                PreLaunchFileSnapshot: fileSnapshot,
                PreLaunchRegistrySnapshot: registrySnapshot,
                IsMidSessionIngested: false
            );

            _activeSessions.TryAdd(processId, session);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "ゲーム起動トレース処理中にエラーが発生しました。");
        }
    }

    private void OnProcessStopped(uint processId, string processName)
    {
        try
        {
            if (_activeSessions.TryRemove(processId, out var session))
            {
                _logger.LogInformation("🛑 ゲームプロセスの消滅を検知しました: {Name} (PID: {PID})。ポスト監査を開始します...", processName, processId);
                Task.Run(async () => await ExecutePostLaunchAuditAsync(session));
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "ゲーム終了トレース処理中にエラーが発生しました。");
        }
    }

    private async Task ExecutePostLaunchAuditAsync(ActiveSessionState session)
    {
        try
        {
            _logger.LogInformation("【ポスト監査 1/3】ドロップファイルの追跡照会中...");
            var droppedArtifacts = _droppedArtifactTracker.DetectDroppedArtifacts(session.GameFolderPath, session.PreLaunchFileSnapshot);

            _logger.LogInformation("【ポスト監査 2/3】高度レジストリ永続化の改ざん照会 ＆ ロールバック中...");
            var tamperedRegistryKeys = _registryTracker.DetectAndRollbackRegistryChanges(session.PreLaunchRegistrySnapshot);

            _logger.LogInformation("【ポスト監査 3/3】OS 急所ファイル (hosts 等) の整合性照会 ＆ アトミック復元中...");
            var hvtTamperResults = await _hvtVerifier.VerifyAndRestoreIntegrityAsync(CancellationToken.None);

            if (droppedArtifacts.Count > 0)
            {
                _logger.LogWarning("⚠️ ゲーム終了時に外部領域へ書き出された不審ファイルが {Count} 件検出されました。", droppedArtifacts.Count);
            }

            if (hvtTamperResults.Any(r => r.IsTampered))
            {
                _logger.LogWarning("⚠️ 急所ファイルの改ざんが検知され、ベースラインから復元されました。");
            }

            _logger.LogInformation("✨ ポスト監査が完了しました。監視エンジンを停止 (0% / OFF) に移行します。");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "ポスト監査処理中にエラーが発生しました。");
        }
        finally
        {
            var werRecovery = _crashBlocker.RestoreOriginalCrashReportSettings();
                if (werRecovery.Status is WerRecoveryStatus.RecoveryBlocked or WerRecoveryStatus.RecoveryFailed)
                {
                    _logger.LogError("WER 元値復元に失敗したため、完全復元を完了扱いにしません。Status={Status}, Reason={Reason}", werRecovery.Status, werRecovery.Reason);
                }
        }
    }

    private async Task ExecutePostCrashAuditAsync(CancellationToken ct)
    {
        try
        {
            var incompleteSessions = await _scanSessionRepository.GetIncompleteSessionsAsync(ct);

            if (incompleteSessions.Count > 0)
            {
                _logger.LogWarning("前回のゲーム実行中に異常終了 (電源断/BSoD) を検知しました。ポスト・クラッシュ監査を実行します (未完了数: {Count})。", incompleteSessions.Count);
                await _scanSessionRepository.RepairIncompleteSessionsAsync(DateTimeOffset.UtcNow, ct);
            }

            var osRules = await _firewallManager.GetOsActualRulesAsync(ct);
            var orphanedTempRules = osRules.Where(r => r.RuleTag.StartsWith("GST:STRICT-TEMP:", StringComparison.OrdinalIgnoreCase)).ToList();

            foreach (var rule in orphanedTempRules)
            {
                await _firewallManager.RemoveBlockRuleAsync(rule.RuleTag, ct);
                _logger.LogInformation("クラッシュにより孤立した一時ファイアウォールルールを安全に除去しました: {Tag}", rule.RuleTag);
            }

            // 孤児一時ファイルのクリーンアップ
            string quarantineBaseDir = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "GameSecurityTool", "Quarantine");
            if (Directory.Exists(quarantineBaseDir))
            {
                foreach (var tmpFile in Directory.EnumerateFiles(quarantineBaseDir, "*.tmp", SearchOption.TopDirectoryOnly))
                {
                    try { File.Delete(tmpFile); } catch { }
                }
            }

            _logger.LogInformation("ポスト・クラッシュ監査および OS 状態修復が完了しました。");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "ポスト・クラッシュ監査中にエラーが発生しました。");
        }
    }

    private sealed record ActiveSessionState(
        uint ProcessId,
        string ProcessName,
        Guid GameFolderId,
        string GameFolderPath,
        DateTimeOffset StartedAt,
        Dictionary<string, DateTimeOffset> PreLaunchFileSnapshot,
        Dictionary<string, string> PreLaunchRegistrySnapshot,
        bool IsMidSessionIngested
    );
}
```

---

# 6. 単体テスト仕様 (`GST.UnitTests.SYS`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.SYS;

using System;
using System.IO;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Infrastructure.Native;
using GameSecurityTool.Infrastructure.Persistence;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging.Abstractions;
using Moq;
using Xunit;

public class SystemSafetyTests
{
    [Fact]
    public void VerifyAuthenticodeSignature_SystemBinary_ReturnsTrue()
    {
        string systemBinary = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.System), "notepad.exe");
        bool isSigned = Win32SecurityNativeMethods.VerifyAuthenticodeSignature(systemBinary);
        Assert.True(isSigned);
    }

    [Fact]
    public async Task ExecuteFullSystemReversionAsync_WithQuarantineDrain_ExecutesCleanly()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>().UseSqlite("Data Source=:memory:").Options;
        var dbFactoryMock = new Mock<IDbContextFactory<AppDbContext>>();
        dbFactoryMock.Setup(f => f.CreateDbContextAsync(default)).ReturnsAsync(() => new AppDbContext(options));

        var dbWriterMock = new Mock<IDbWriteQueue>();
        var firewallMock = new Mock<IFirewallManager>();
        firewallMock.Setup(f => f.GetOsActualRulesAsync(default)).ReturnsAsync([]);

        var crashBlockerMock = new Mock<ICrashReportTelemetryBlocker>();
        var quarantineMock = new Mock<IQuarantineService>();

        var service = new FullSystemReversionService(
            dbFactoryMock.Object,
            dbWriterMock.Object,
            firewallMock.Object,
            crashBlockerMock.Object,
            quarantineMock.Object,
            NullLogger<FullSystemReversionService>.Instance);

        bool result = await service.ExecuteFullSystemReversionAsync(restoreQuarantinedFiles: true);
        Assert.True(result);
    }
}
```

---

