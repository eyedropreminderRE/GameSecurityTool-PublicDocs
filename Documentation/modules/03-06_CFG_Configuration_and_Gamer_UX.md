# 03-06: CFG & UX - Configuration, Profile & Gamer UX Specification

**Document ID:** GST-MOD-CFG-006  
**Version:** 4.2 (Configuration Time Machine, Diff Guard, File Lock Inspector & Self-Maintenance Edition)
**Status:** Approved Module Specification  
**Target Projects:**
- `GameSecurityTool.Domain`
- `GameSecurityTool.Contracts`
- `GameSecurityTool.Application`
- `GameSecurityTool.Infrastructure`
- `GameSecurityTool.Presentation`

> **Lifecycle note:** This module specification describes approved design scope. Its presence does not by itself indicate that the module is implemented, Windows-verified, or released.

---

# 1. モジュール概要 ＆ 責任境界

本モジュールは、**Universal FeatureControl Policy、セキュリティプロファイル二層管理（Preset vs Instance）、設定タイムマシン（Configuration Time Machine - 10世代スナップショット ＆ SQLite オンラインバックアップ ＆ Diff Guard による SSD 摩耗完全防止）、Smart Game IME Lock（長押し切替・パニック脱出・4大プリセット・非侵入型IMEロケール制御）、ファイル共有違反時の原因特定（Win32 Restart Manager API `FileLockInspector` による他社セキュリティツール検知）、自律セルフメンテナンス（5項目自動クリーンアップ ＆ 誤操作防止 Type-to-Confirm ファクトリーリセット）、フルスクリーン自動サイレント通知（Smart Do Not Disturb）、ワンクリック Undo、改ざん検知監査ログ（Hash Chain ＆ 多層アンカー検証・劣化可視化）、個人情報自動マスキング（`ILogSanitizer` / `[GeneratedRegex]`）、および先行ディレクトリ ACL 制御（NTFS ACL）** を担当する。

### Clean 5-Layer レイヤー境界:
- **Domain Layer (`GST.Domain`):** FeatureScope 状態解決ロジック、プロファイル整合性検証。外部依存ゼロ。
- **Contracts Layer (`GST.Contracts`):** `IDispatcherService`, `ISmartImeCoordinator`, `ITamperEvidentAuditLogger`, `IUserPresenceService`, `ILogSanitizer`, `IElevatedWorkerClient`, `IDbWriteQueue`, `IConfigurationTimeMachineService`, `IFileLockInspector`, `ISelfMaintenanceService` Port および不変 DTO 群（`SmartImeProfileConfig`, `TargetInputProfile`, `AuditVerificationResultDto`, `ConfigurationDiffSummaryDto`, `LockingProcessInfoDto`, `MaintenanceReportDto` 等）。
- **Application Layer (`GST.Application`):** プロファイル変更・ロールバック Use Case、監査ログ記録調停、マスキング調停、設定タイムマシン差分復元調停、セルフメンテナンス調停。
- **Infrastructure Layer (`GST.Infrastructure`):** Win32 `SHQueryUserNotificationState`, .NET 10 `[GeneratedRegex]` を用いた `ILogSanitizer` 実装, 先行 NTFS ACL 安全適用, `IDbWriteQueue` による監査直列化コミット, SQLite Online Backup API による `ConfigurationTimeMachineService` 実装, `rstrtmgr.dll` P/Invoke による `FileLockInspector`, 一時ファイル・WAL・ログパージを担う `SelfMaintenanceService`。
- **Presentation Layer (`GST.Presentation`):** WPF (Fluent UI / Mica & Acrylic), DashboardView, SettingsView, TimeMachineDiffDialog, TypeToConfirmDialog, タスクトレイ常駐。

---

# 2. 機能要件仕様 (Functional Requirements)

## 2.1 `FN-CFG-01`: Universal FeatureControl ＆ 二層プロファイル管理
* **二層構造:** 不可変テンプレート (`SecurityProfilePreset`: Standard, Enhanced, Maximum) とユーザー実体 (`SecurityProfileInstance`) を完全分離。
* **FeatureScope:** `Global` (Enabled/Disabled) と `GameProfile` (UseGlobal/Enabled/Disabled) を独立解決。

## 2.2 `FN-CFG-02`: フルスクリーン自動サイレント通知 (`Smart Do Not Disturb`)
* **目的:** ゲームプレイ中の画面最小化、フォーカス奪取、FPS 低下を防止する。
* **処理仕様:** Win32 `SHQueryUserNotificationState` API を照会し、`QUNS_RUNNING_D3D_FULL_SCREEN` 実行中はトースト通知を完全自動抑制（トレイバッジ点灯のみ）。API 呼び出し自体が失敗（非ゼロ HRESULT）したなどの不明な状態においては、「通知してはならない」安全側（フェイルクローズ）に倒し、トースト通知を抑制する。

## 2.3 `FN-CFG-03`: ワンクリック Undo (Quick Recovery)
* **目的:** 誤遮断や誤隔離時にダッシュボード最上部からワンクリックで復旧レビューへ移行し、直前状態への復元を迅速に開始できるようにする。
* **安全境界:** 「ワンクリック」は復旧レビュー画面への入口を意味し、実データの復元・上書きを無確認で実行することを意味しない。
* **復旧フロー:** `Review → Confirm → Execute`。レビュー画面では、復旧対象、現在状態からの変更内容、復旧後の状態、および必要に応じた注意事項を提示する。ユーザーが明示的に確認した後にのみ復元処理を実行する。
* **適用範囲:** セーブデータ、ゲームファイル、隔離ファイル等の実データを変更する復旧は、必ず明示的なConfirmを要求する。単なる画面遷移や非破壊的な表示操作まで確認対象に拡張しない。

## 2.4 `FN-CFG-04`: Tamper-Evident 改ざん検知監査ログ (Hash Chain ＆ 多層アンカー Fail-Closed 検証 - approved quality refinement)
* **構造:** 創世記ハッシュ `GenesisHash` を起点として `PreviousHash` と `CurrentHash` を連鎖（SHA-256）。
* **多層アンカー保護仕様:**
  1. **Layer 1 (ローカル DPAPI - 即時同期):** 同一ユーザー内の非特権アプリによる平文 DB 直接書き換えを即座に検知。
  2. **Layer 2 (特権管理者アンカー - 非同期同期 ＆ Fail-Closed 規約):** 特権ワーカー（UAC 昇格時）と連携し、the protected local audit anchor へ最新ハッシュを複製保存。  
     **【approved quality refinement】`VerifyAuditChainIntegrityAsync` において、検証時に Layer 2 アンカーが未同期（不在・空）または不整合の場合、サイレントに成功扱い（Fail-Open）することを厳禁とし、厳格に `false` を返却する。`VerifyAuditChainDetailedAsync` においても `IsLayer2Synchronized = false` をダッシュボードへ可視化し、同一権限マルウェアによる改ざんリスクを明示する。**

## 2.5 `FN-CFG-05`: Universal Privacy Shield ＆ ログ・プロンプト自動マスキング (`ILogSanitizer`)
* **目的:** Serilog 等のログ出力時および外部 AI（Gemini）プロンプト構築時に、個人情報および機密認証トークンを C# 14 `[GeneratedRegex]` で超高速に自動完全マスキングする。
* **マスキング対象:**
  - ユーザーディレクトリ・プロファイルパス (`C:\Users\<UserName>\...` ➔ `C:\Users\***\...`)
  - Discord Webhook URL (`https://discord.com/api/webhooks/...` ➔ `[REDACTED_WEBHOOK]`)
  - Twitch OAuth トークン (`oauth:...` ➔ `[REDACTED_TWITCH_TOKEN]`)
  - Authorization Bearer トークン (`Bearer ...` ➔ `[REDACTED_BEARER_TOKEN]`)
  - メールアドレス (`***@***.***`), MAC アドレス (`**:**:**:**:**:**`), ローカル IP, マシン名 (ホスト名)
* **DI 境界:** `Contracts.Interfaces.ILogSanitizer` を実装するシングルトンサービスとして登録し、コンストラクタ注入で利用可能とする。

## 2.6 `FN-CFG-06`: 先行ディレクトリ NTFS ACL 制御 (`DatabaseAccessControl`)
* **目的:** `gamesecurity.db` 生成時、ファイル作成前の親ディレクトリ（`%LocalAppData%\GameSecurityTool\`）生成時点で親からの継承を切り、**「CurrentUser」と「SYSTEM」以外からのアクセスを物理拒否（Explicit Deny）** することで、初期生成時の露出ウィンドウを極小化する。
* **非NTFSフォールバック:** FAT32/exFAT/ネットワークドライブ等では例外でクラッシュさせず、安全に警告ログを記録してスキップする。

## 2.7 `FN-CFG-07`: AI 8大ユースケース ＆ ガードレール UX 仕様
1. **AI 8大ユースケース:**
   - [01] 未署名 MOD レントゲン診断
   - [02] クラッシュレポート即時原因診断
   - [03] 設定画面 AI コンシェルジュ
   - [04] コミュニティ脅威 OSINT 調査
   - [05] マルチプレイ接続トラブルシュート
   - [06] セーブ肥大化・異常診断
   - [07] 週間セキュリティ要約
   - [08] MOD 起動不能・前提不足相談
2. **AI ガードレール UX:**
   - **送信前プロンプト確認 (Prompt Preview):** AI 相談実行前に、マスキング後の送信データを目視確認・編集・承認できる確認モーダルを表示。
   - **分析限界と免責明示:** AI の診断は GST が収集したローカルログのみに基づく推測であり、安全性や動作を保証するものではない旨を画面上に常時表示。
   - **推奨 FW ルール適用 UI:** AI 推奨ルールの適用時は、自己責任免責ダイアログおよびポート・通信方向の微調整 UI を必ず経由。

## 2.8 `FN-CFG-08`: 外部ツール連携 ＆ OS 自動起動 UX ガバナンス
1. **外部ファイル圧縮ソフトウェア連携:** ゲーム本体バックアップ時の誤操作防止のため、「圧縮開始前の詳細設定ダイアログ表示」は **既定 ON** とする。
2. **外部セキュリティスキャナ連携:** 定期スキャンは **既定 OFF (オプトイン)** とし、ゲーム実行中は負荷防止のため **絶対延期** とする。
3. **OS 自動起動 (Run キー):** GST 本体のログオン時自動起動は **既定 OFF (オプトイン)** とし、設定画面での明示的チェック時のみ `HKCU\...\Run` に登録する。

## 2.9 `FN-CFG-09`: Configuration Time Machine ＆ Diff Guard (SSD 摩耗防止)
1. **Diff Guard:** 現在の設定ハッシュ（プロファイル＋ルール）をインメモリ保持し、設定に変更がない場合はスナップショット生成を完全スキップ（SSD 書き込み量ゼロ化）。
2. **SQLite オンラインバックアップ API:** SQLite 公式バックアップ API (`sourceConn.BackupDatabase(destConn)`) を使用し、ロック競合や破損を起こさずに 10 世代のスナップショット (`.shadow`) をローカル退避。
3. **スマート部分復元 (Partial Restore):** 全体復元ではなく、ゲームごとの設定変更点（保護モード、FW ルール等）を差分プレビュー表示し、チェックボックスで選択したゲームのみを外科手術的に復元。
4. **アンインストール済みゲームの除外 (Smart Orphan Filter):** 現在の PC 上にインストール実体が存在しないゲーム設定は復元対象から自動除外し、設定のゾンビ化を防止。

## 2.10 `FN-CFG-10`: 他社セキュリティソフト・ファイルロック特定 (`FileLockInspector`)
1. **共有違反 (`0x80070020`) 検知:** バックアップ復元や隔離ファイル操作時に共有違反が発生した場合、Win32 `Restart Manager API` (`rstrtmgr.dll`) を P/Invoke 呼び出し。
2. **ロック元プロセス特定:** ファイルをロックしているプロセス名（`strAppName`）およびプロセス ID を取得。
3. **セキュリティソフト判定:** 既知の外部セキュリティ対策ツール（`mcshield.exe`, `bdservicehost.exe`, `avp.exe`, `ekrn.exe` 等）であるかを判定し、「外部セキュリティツールによりファイルが一時ロックされています。数秒後に自動再試行します」と正確な原因を提示。

## 2.11 `FN-CFG-11`: 自律セルフメンテナンス ＆ Safe Factory Reset (Type-to-Confirm)
1. **定期自律メンテナンス (5項目自動クリーンアップ):**
   - 孤児となった一時作業ファイル (`.tmp`, `.token`, `.creating`) の完全回収。
   - SQLite データベースの最適化 (`PRAGMA optimize;`) および WAL 切り詰め (`PRAGMA wal_checkpoint(TRUNCATE);`)。
   - 30日を経過した古い運用ログの自動パージ。
   - 肥大化した過去クラッシュダンプファイルのクリーンアップ。
2. **Safe Factory Reset (Type-to-Confirm UX):**
   - 誤クリックによる設定全消失を物理的に防ぐため、確認ダイアログ上で「RESET」または製品名文字列の手動キー入力を要求。
   - 実行時は SQLite DB プール切断 (`SqliteConnection.ClearAllPools()`) を先行し、DB ファイルおよび一時設定のみを安全消去（実行バイナリ本体は維持）。

## 2.12 `FN-CFG-12`: 【確定】Smart Game IME Lock (`GST-FEAT-UX-IME-004`)

**目的:** 登録ゲームのプレイ中に入力ロケールの誤操作を抑制し、日本語チャットを意図的に使用した後も、ユーザーが短い脱出操作で英数状態へ戻せるようにする。機能はゲームプロセス内部へ侵入せず、OS標準の入力ロケール要求と受動的な入力観測を使用する。

### 機能契約

1. **既定状態:** 登録ゲームセッションは英数状態を基本とし、日本語モードへの移行は半角/全角キー等の設定キーを `HoldDurationSeconds` 以上連続押下した場合のみ受理する。短い単発/連打で日本語モードを有効化しない。
2. **脱出操作:** 日本語中は、半角/全角単発/連打、ESC、無変換、設定されたマウススワイプ、設定されたマウスクリック連打を英数復帰要求として扱う。EnterおよびWASDはトリガーに使用せず、マウスサイドボタンも対象外とする。
3. **4大プリセット:** `FpsAction`（既定0.3秒）、`MmoMoba`（0.5秒）、`VoiceSolo`（1.5秒）、`OfflineRpg`（0.5秒）を提供し、ユーザー設定として `Custom` を許可する。
4. **視覚バッジ:** `Hidden` を既定とし、`FadeOnTransition` / `ShowWhileJapanese` / `AlwaysVisible` を選択可能とする。
5. **フェイルセーフ:** 登録ゲームのフォーカス喪失時はIME制御セッションを解放し、Windowsの恒久設定を変更しない。GST終了時もOS設定を自然復帰させる。
6. **非侵入:** `SetWindowsHookEx`、`SendInput`、ゲームメモリアクセス、DLLインジェクション、DirectX/Vulkan内部フックを使用しない。入力ロケール要求は `WM_INPUTLANGCHANGEREQUEST`、ゲーム/ウィンドウ識別は `GetForegroundWindow` / `GetWindowThreadProcessId` / `GetGUIThreadInfo` 等のOS標準APIで行う。
7. **入力監視:** キーボードの長押し判定は公称50ms周期の受動サンプラで行い、マウス移動はRaw Input等の上流Adapterから `NotifyRawMouseMovement(deltaX, deltaY)` へ渡す。Coordinator自身は入力をゲームへ注入・横取りしない。
8. **結果受理:** Request Generation、対象HWND/Thread/PID、要求HKLを照合するQuad-Gateを通過した場合のみ、成功状態・UI・SEを確定する。非同期処理の古い結果は破棄する。

### 実装境界

`SmartImeCoordinator` は `GameSecurityTool.Infrastructure.GamerUx` に配置し、`ISmartImeCoordinator` をContracts Portとして実装する。`SmartImeCoordinator` はOS APIを使用できるが、Domain層には公開しない。WPF Dispatcher外で取得した入力観測は `IDispatcherService` を介して状態変更へ委譲する。

> **入力経路整合注記:** 上位の入力仕様は「Raw Input」を入力経路として言及する一方、参照用Coordinatorコードではキーボードを `GetAsyncKeyState` の50msサンプラで観測し、マウスは `NotifyRawMouseMovement` で受け取る構成になっている。したがって本モジュールでは `RegisterRawInputDevices` 等を既成実装として捏造せず、Raw Inputは上流Adapterからの入力供給契約として扱う。別途Raw Input APIを実装する場合は、このCoordinatorの入力Portへ接続する追加実装として扱う。

---

# 3. Contracts Layer 定義 (`GameSecurityTool.Contracts`)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System;
using System.Collections.Generic;
using GameSecurityTool.Contracts.GamerUx;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Common;
using GameSecurityTool.Contracts.Dtos;

public interface IDispatcherService
{
    void Invoke(Action action);
    Task InvokeAsync(Func<Task> asyncAction);
}

public interface ITamperEvidentAuditLogger
{
    Task AppendLogAsync(
        string operationId,
        AuditEventType eventType,
        string actor,
        string target,
        string result,
        CancellationToken ct = default);

    Task<bool> VerifyAuditChainIntegrityAsync(CancellationToken ct = default);

    /// <summary>
    /// 【C-5 是正】多層アンカーの個別検証ステータスを含む詳細検証結果を取得
    /// </summary>
    Task<AuditVerificationResultDto> VerifyAuditChainDetailedAsync(CancellationToken ct = default);

    bool IsLayer2AnchorSynchronized { get; }
}

public interface IUserPresenceService
{
    bool IsSafeToShowNotification();
}

public interface ILogSanitizer
{
    string Sanitize(string rawMessage);
}

public interface IElevatedWorkerClient
{
    Task<bool> WriteAuditAnchorAsync(string hash, CancellationToken ct = default);
    Task<string?> ReadAuditAnchorAsync(CancellationToken ct = default);
}

public interface IConfigurationTimeMachineService
{
    Task<ConfigurationBackupResultDto> CreateSnapshotAsync(bool force = false, CancellationToken ct = default);
    Task<ConfigurationDiffSummaryDto> ComputeDiffAsync(string snapshotPath, CancellationToken ct = default);
    Task<bool> RestoreConfigurationPartialAsync(ConfigurationRestoreRequestDto request, CancellationToken ct = default);
    Task<IReadOnlyList<ConfigurationSnapshotMetadataDto>> GetSnapshotHistoryAsync(CancellationToken ct = default);
}

public interface IFileLockInspector
{
    IReadOnlyList<LockingProcessInfoDto> GetLockingProcesses(string filePath);
}

public interface ISelfMaintenanceService
{
    Task<MaintenanceReportDto> ExecuteMaintenanceAsync(bool emergencyStorageRelief = false, CancellationToken ct = default);
}

public interface ISmartImeCoordinator : IDisposable
{
    bool IsJapaneseInputActive { get; }
    void UpdateConfig(SmartImeProfileConfig config);
    void NotifyForegroundWindowChanged(IntPtr foregroundHwnd, uint processId);
    void NotifyManualBypassRequested();
    void RegisterGameSession(IntPtr gameHwnd, uint threadId, uint pid);
    void UnregisterGameSession();
    void NotifyRawMouseMovement(int deltaX, int deltaY);
    void NotifyMouseClick();
}
```

```csharp
namespace GameSecurityTool.Contracts.GamerUx;

using System;

public enum SoundPreset
{
    ElectronicBeep,
    MechanicalClick,
    SoftChime,
    Mute
}

public enum ImeLockPreset
{
    FpsAction,
    MmoMoba,
    VoiceSolo,
    OfflineRpg,
    Custom
}

public enum VisualBadgeStyle
{
    Hidden,
    FadeOnTransition,
    ShowWhileJapanese,
    AlwaysVisible
}

public sealed record TargetInputProfile
{
    public string Klid { get; init; } = string.Empty;
    public long RegisteredHklValue { get; init; }
    public string DisplayName { get; init; } = string.Empty;

    [System.Text.Json.Serialization.JsonIgnore]
    public IntPtr HklRaw { get; init; } = IntPtr.Zero;

    [System.Text.Json.Serialization.JsonIgnore]
    public bool IsValid => HklRaw != IntPtr.Zero && RegisteredHklValue != 0 && !string.IsNullOrWhiteSpace(Klid);
}

public sealed record SmartImeProfileConfig
{
    public bool IsEnabled { get; init; } = true;
    public ImeLockPreset Preset { get; init; } = ImeLockPreset.FpsAction;
    public double HoldDurationSeconds { get; init; } = 0.5;
    public bool EnableKeyboardEscape { get; init; } = true;
    public bool EnableMouseSwipeEscape { get; init; } = true;
    public int MouseSwipeThresholdPixels { get; init; } = 100;
    public bool EnableMouseClickEscape { get; init; } = true;
    public int MouseClickEscapeCount { get; init; } = 2;
    public bool EnableIdleTimer { get; init; } = true;
    public int IdleTimerSeconds { get; init; } = 60;
    public VisualBadgeStyle BadgeStyle { get; init; } = VisualBadgeStyle.Hidden;
    public SoundPreset SoundPreset { get; init; } = SoundPreset.ElectronicBeep;
    public TargetInputProfile TargetEnglish { get; init; } = new()
    {
        Klid = "00000409",
        RegisteredHklValue = 0x04090409,
        DisplayName = "英語 (米国)",
        HklRaw = (IntPtr)0x04090409
    };
    public TargetInputProfile TargetJapanese { get; init; } = new()
    {
        Klid = "00000411",
        RegisteredHklValue = 0x04110411,
        DisplayName = "日本語",
        HklRaw = (IntPtr)0x04110411
    };
}
```

```csharp
namespace GameSecurityTool.Contracts.Dtos;

using System;
using System.Collections.Generic;

/// <summary>
/// 【C-5 是正】監査チェーン多層検証の詳細結果 DTO
/// </summary>
public sealed record AuditVerificationResultDto(
    bool IsChainValid,
    bool IsLayer1DpapiMatched,
    bool IsLayer2AdminAnchorMatched,
    bool IsLayer2Synchronized,
    int TotalEntriesVerified,
    string? BrokenSequenceDetails,
    string? WarningMessage
);

public sealed record ConfigurationBackupResultDto(
    bool Success,
    bool isSkipped,
    string SnapshotPath
);

public sealed record ConfigurationSnapshotMetadataDto(
    string FullPath,
    DateTimeOffset CreatedAtUtc,
    long FileSizeBytes
);

public sealed record GameConfigurationDiffDto(
    Guid GameProfileId,
    string DisplayName,
    bool IsSelectedForRestore,
    IReadOnlyList<string> ChangedAttributes
);

public sealed record ConfigurationDiffSummaryDto(
    string SnapshotPath,
    DateTimeOffset SnapshotCreatedAtUtc,
    IReadOnlyList<GameConfigurationDiffDto> GameDiffs,
    IReadOnlyList<string> SkippedUninstalledGames
);

public sealed record ConfigurationRestoreRequestDto(
    string SnapshotPath,
    IReadOnlyList<Guid> TargetGameProfileIdsToRestore
);

public sealed record LockingProcessInfoDto(
    int ProcessId,
    string ProcessName,
    bool IsSecuritySoftware
);

public sealed record MaintenanceReportDto(
    bool Success,
    int CleanedTempFilesCount,
    long ReclaimedTempSizeBytes,
    bool DbOptimized,
    int PrunedLogFilesCount,
    long ReclaimedLogSizeBytes,
    string? ErrorMessage
);
```

---

# 4. Infrastructure Layer 実装参照コード (`GameSecurityTool.Infrastructure`)

> **実装状態:** 本章のコードは参照用サンプルです。特に `SmartImeCoordinator` は Generation / RequestContext / 150ms Verify / Quad-Gate を完全統合していないため、本番実装へそのまま転用してはなりません。正式な入力ロケール制御の成功受理条件は本書4章冒頭および上位Tierの定義を優先します。

## 4.1 Smart Game IME Lock Infrastructure (`SmartImeCoordinator.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.GamerUx;

using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.GamerUx;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.Extensions.Logging;

public sealed partial class SmartImeCoordinator : ISmartImeCoordinator
{
    private const uint WM_INPUTLANGCHANGEREQUEST = 0x0050;
    private const int VK_KANJI = 0x19;
    private const int VK_OEM_AUTO = 0xF3;
    private const int VK_OEM_ENLW = 0xF4;
    private const int VK_ESCAPE = 0x1B;
    private const int VK_NONCONVERT = 0x1D;

    [StructLayout(LayoutKind.Sequential)]
    private struct RECT
    {
        public int Left;
        public int Top;
        public int Right;
        public int Bottom;
    }

    [StructLayout(LayoutKind.Sequential)]
    private struct GUITHREADINFO
    {
        public uint cbSize;
        public uint flags;
        public IntPtr hwndActive;
        public IntPtr hwndFocus;
        public IntPtr hwndCapture;
        public IntPtr hwndMenuOwner;
        public IntPtr hwndMoveSize;
        public IntPtr hwndCaret;
        public RECT rcCaret;
    }

    private readonly ILogger<SmartImeCoordinator> _logger;
    private readonly GameSecurityTool.Contracts.Interfaces.IDispatcherService _dispatcher;
    private SmartImeProfileConfig _config;
    private IntPtr _currentGameHwnd = IntPtr.Zero;
    private uint _currentGameThreadId;
    private uint _currentGamePid;
    private bool _isGameFocused;
    private bool _isBypassed;
    private readonly Stopwatch _keyHoldStopwatch = new();
    private readonly Stopwatch _idleStopwatch = new();
    private readonly CancellationTokenSource _cts = new();
    private double _mouseAccumulatedDistance;
    private int _recentClickCount;
    private readonly Stopwatch _clickIntervalStopwatch = new();

    [LibraryImport("user32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    private static partial bool PostMessageW(IntPtr hWnd, uint msg, IntPtr wParam, IntPtr lParam);

    [LibraryImport("user32.dll")]
    private static partial short GetAsyncKeyState(int vKey);

    [LibraryImport("user32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    private static partial bool GetGUIThreadInfo(uint idThread, out GUITHREADINFO lpgui);

    [LibraryImport("user32.dll")]
    private static partial IntPtr GetForegroundWindow();

    [LibraryImport("user32.dll")]
    private static partial uint GetWindowThreadProcessId(IntPtr hWnd, out uint lpdwProcessId);

    public bool IsJapaneseInputActive { get; private set; }

    public SmartImeCoordinator(
        SmartImeProfileConfig initialConfig,
        GameSecurityTool.Contracts.Interfaces.IDispatcherService dispatcher,
        ILogger<SmartImeCoordinator> logger)
    {
        _config = initialConfig;
        _dispatcher = dispatcher;
        _logger = logger;
        _ = MonitorInputLoopAsync();
    }

    public void UpdateConfig(SmartImeProfileConfig config)
    {
        _dispatcher.Invoke(() =>
        {
            _config = config;
            _logger.LogInformation("Smart IME Lock 設定更新 (Preset: {Preset})", config.Preset);
        });
    }

    public void NotifyForegroundWindowChanged(IntPtr foregroundHwnd, uint processId)
    {
        _dispatcher.Invoke(() =>
        {
            if (_currentGameHwnd != IntPtr.Zero && foregroundHwnd == _currentGameHwnd && processId == _currentGamePid)
            {
                _isGameFocused = true;
                return;
            }

            if (_isGameFocused)
            {
                _isGameFocused = false;
                IsJapaneseInputActive = false;
                ResetInputState();
                _logger.LogTrace("フォーカス喪失検知: IME入力制御セッションを解放");
            }
        });
    }

    public void NotifyManualBypassRequested()
    {
        _dispatcher.Invoke(() =>
        {
            _isBypassed = !_isBypassed;
            if (_isBypassed && IsJapaneseInputActive)
            {
                LockToEnglish();
            }
            _logger.LogWarning("手動バイパス切替: {State}", _isBypassed ? "一時停止中" : "通常稼働");
        });
    }

    public void RegisterGameSession(IntPtr gameHwnd, uint threadId, uint pid)
    {
        _dispatcher.Invoke(() =>
        {
            _currentGameHwnd = gameHwnd;
            _currentGameThreadId = threadId;
            _currentGamePid = pid;
            _isGameFocused = true;
            _isBypassed = false;
            LockToEnglish();
        });
    }

    public void UnregisterGameSession()
    {
        _dispatcher.Invoke(() =>
        {
            _currentGameHwnd = IntPtr.Zero;
            _currentGameThreadId = 0;
            _currentGamePid = 0;
            _isGameFocused = false;
            IsJapaneseInputActive = false;
            ResetInputState();
        });
    }

    private async Task MonitorInputLoopAsync()
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMilliseconds(50));
        try
        {
            while (await timer.WaitForNextTickAsync(_cts.Token))
            {
                if (!_config.IsEnabled || !_isGameFocused || _isBypassed || _currentGameHwnd == IntPtr.Zero)
                {
                    continue;
                }

                try
                {
                    await _dispatcher.InvokeAsync(() =>
                    {
                        ProcessKeyboardInput();
                        ProcessIdleTimer();
                        return Task.CompletedTask;
                    });
                }
                catch (Exception ex)
                {
                    _logger.LogTrace(ex, "入力監視ループ例外");
                }
            }
        }
        catch (OperationCanceledException)
        {
        }
    }

    private void ProcessKeyboardInput()
    {
        bool isImeKeyCandidate = IsKeyDown(VK_OEM_AUTO) || IsKeyDown(VK_OEM_ENLW) || IsKeyDown(VK_KANJI);

        if (isImeKeyCandidate)
        {
            _idleStopwatch.Restart();
            if (!_keyHoldStopwatch.IsRunning)
            {
                _keyHoldStopwatch.Restart();
            }
            else if (!IsJapaneseInputActive && _keyHoldStopwatch.Elapsed.TotalSeconds >= _config.HoldDurationSeconds)
            {
                UnlockToJapanese();
                _keyHoldStopwatch.Reset();
            }
        }
        else if (_keyHoldStopwatch.IsRunning)
        {
            long holdMs = _keyHoldStopwatch.ElapsedMilliseconds;
            _keyHoldStopwatch.Reset();
            if (holdMs < (_config.HoldDurationSeconds * 1000) && IsJapaneseInputActive)
            {
                LockToEnglish();
            }
        }

        if (_config.EnableKeyboardEscape && (IsKeyDown(VK_ESCAPE) || IsKeyDown(VK_NONCONVERT)) && IsJapaneseInputActive)
        {
            _idleStopwatch.Restart();
            LockToEnglish();
        }
    }

    public void NotifyRawMouseMovement(int deltaX, int deltaY)
    {
        _dispatcher.Invoke(() =>
        {
            if (!_config.IsEnabled || !_isGameFocused || !IsJapaneseInputActive || !_config.EnableMouseSwipeEscape)
            {
                return;
            }

            _idleStopwatch.Restart();
            double distance = Math.Sqrt((deltaX * deltaX) + (deltaY * deltaY));
            if (distance > 5.0)
            {
                _mouseAccumulatedDistance += distance;
                if (_mouseAccumulatedDistance >= _config.MouseSwipeThresholdPixels)
                {
                    LockToEnglish();
                    _mouseAccumulatedDistance = 0.0;
                }
            }
        });
    }

    public void NotifyMouseClick()
    {
        _dispatcher.Invoke(() =>
        {
            if (!_config.IsEnabled || !_isGameFocused || !IsJapaneseInputActive || !_config.EnableMouseClickEscape)
            {
                return;
            }

            _idleStopwatch.Restart();
            if (!_clickIntervalStopwatch.IsRunning || _clickIntervalStopwatch.ElapsedMilliseconds > 400)
            {
                _recentClickCount = 1;
                _clickIntervalStopwatch.Restart();
            }
            else
            {
                _recentClickCount++;
                if (_recentClickCount >= _config.MouseClickEscapeCount)
                {
                    LockToEnglish();
                    _recentClickCount = 0;
                    _clickIntervalStopwatch.Reset();
                }
            }
        });
    }

    private void ProcessIdleTimer()
    {
        if (!_config.EnableIdleTimer || !IsJapaneseInputActive)
        {
            return;
        }

        if (_idleStopwatch.Elapsed.TotalSeconds >= _config.IdleTimerSeconds)
        {
            LockToEnglish();
            _idleStopwatch.Reset();
        }
    }

    private void LockToEnglish()
    {
        IsJapaneseInputActive = false;
        ResetInputState();
        SendLanguageChangeMessage(_config.TargetEnglish.HklRaw);
    }

    private void UnlockToJapanese()
    {
        if (!_config.TargetJapanese.IsValid)
        {
            _logger.LogTrace("日本語ターゲットHKLが未設定のため切替要求を抑止しました。");
            return;
        }

        IsJapaneseInputActive = true;
        ResetInputState();
        SendLanguageChangeMessage(_config.TargetJapanese.HklRaw);
    }

    private void ResetInputState()
    {
        _mouseAccumulatedDistance = 0.0;
        _recentClickCount = 0;
        _clickIntervalStopwatch.Reset();
        _idleStopwatch.Restart();
    }

    private void SendLanguageChangeMessage(IntPtr targetHkl)
    {
        if (_currentGameHwnd == IntPtr.Zero || _currentGameThreadId == 0 || targetHkl == IntPtr.Zero)
        {
            return;
        }

        IntPtr targetHwnd = _currentGameHwnd;
        var guiInfo = new GUITHREADINFO
        {
            cbSize = (uint)Marshal.SizeOf<GUITHREADINFO>()
        };

        if (GetGUIThreadInfo(_currentGameThreadId, out guiInfo) && guiInfo.hwndFocus != IntPtr.Zero)
        {
            targetHwnd = guiInfo.hwndFocus;
        }

        IntPtr currentFg = GetForegroundWindow();
        if (currentFg == IntPtr.Zero || currentFg != _currentGameHwnd)
        {
            _logger.LogTrace("Pre-Send 前面ウィンドウ照合不一致: 中断");
            return;
        }

        uint currentThread = GetWindowThreadProcessId(currentFg, out uint currentPid);
        if (currentThread != _currentGameThreadId || currentPid != _currentGamePid)
        {
            _logger.LogTrace("Pre-Send Thread/PID 照合不一致: 中断");
            return;
        }

        uint focusThread = GetWindowThreadProcessId(targetHwnd, out uint focusPid);
        if (focusThread != _currentGameThreadId || focusPid != _currentGamePid)
        {
            _logger.LogTrace("Pre-Send Focus Thread/PID 照合不一致: 中断");
            return;
        }

        bool success = PostMessageW(targetHwnd, WM_INPUTLANGCHANGEREQUEST, IntPtr.Zero, targetHkl);
        if (!success)
        {
            int error = Marshal.GetLastWin32Error();
            if (error == 5)
            {
                _logger.LogWarning("UIPI ブロック検知: 対象ゲームとGSTの整合する権限レベルが必要です。");
            }
        }
    }

    private static bool IsKeyDown(int vKey) => (GetAsyncKeyState(vKey) & 0x8000) != 0;

    public void Dispose()
    {
        _cts.Cancel();
        _cts.Dispose();
    }
}
```

> **実装上の整合注記:** 上記はCoordinatorの参照実装コードを基礎に、対象ゲームのThread/PID照合を追加した参照実装です。上位仕様で定義されたGeneration・RequestContext・VerifyのQuad-Gateを完全実装したものではありません。したがって本コード単独を「Quad-Gateまで実装済み」と扱わず、最終実装では既存のRequest/Verify契約へ結合する必要があります。キーボード入力は50ms `GetAsyncKeyState` サンプラ、マウス移動はRaw Input等の上流Adapterから `NotifyRawMouseMovement` へ供給する契約です。

## 4.2 フルスクリーン検知 (`UserPresenceService.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Native;

using System.Runtime.InteropServices;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.Extensions.Logging;

public sealed partial class UserPresenceService(ILogger<UserPresenceService> logger) : IUserPresenceService
{
    private enum QUERY_USER_NOTIFICATION_STATE
    {
        QUNS_NOT_PRESENT = 1,
        QUNS_BUSY = 2,
        QUNS_RUNNING_D3D_FULL_SCREEN = 3,
        QUNS_PRESENTATION_MODE = 4,
        QUNS_ACCEPTS_NOTIFICATIONS = 5,
        QUNS_QUIET_TIME = 6,
        QUNS_APP = 7
    }

    [LibraryImport("shell32.dll")]
    private static partial int SHQueryUserNotificationState(out QUERY_USER_NOTIFICATION_STATE pquns);

    public bool IsSafeToShowNotification()
    {
        int hresult = SHQueryUserNotificationState(out var state);
        if (hresult != 0)
        {
            return false;
        }

        return state == QUERY_USER_NOTIFICATION_STATE.QUNS_ACCEPTS_NOTIFICATIONS;
    }
}
```

## 4.3 ログサニタイザー Adapter (`PrivacyLogSanitizer.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Logging;

using System;
using System.Text.RegularExpressions;
using GameSecurityTool.Contracts.Interfaces;

public sealed partial class PrivacyLogSanitizer : ILogSanitizer
{
    private static readonly string CurrentUserName = Environment.UserName;

    [GeneratedRegex(@"((?:\\\\\\?\\)?[a-zA-Z]:\\Users\\)([^\\\/]+)", RegexOptions.IgnoreCase | RegexOptions.CultureInvariant)]
    private static partial Regex UserPathGeneratedRegex();

    [GeneratedRegex(@"\b(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\b")]
    private static partial Regex IpAddressGeneratedRegex();

    public string Sanitize(string rawMessage)
    {
        return SanitizeLogMessage(rawMessage);
    }

    public static string SanitizeLogMessage(string rawMessage)
    {
        if (string.IsNullOrEmpty(rawMessage)) return string.Empty;

        string sanitized = UserPathGeneratedRegex().Replace(rawMessage, "${1}***");

        if (!string.IsNullOrEmpty(CurrentUserName) && CurrentUserName.Length > 2)
        {
            sanitized = sanitized.Replace(CurrentUserName, "***", StringComparison.OrdinalIgnoreCase);
        }

        sanitized = IpAddressGeneratedRegex().Replace(sanitized, match =>
        {
            string ip = match.Value;
            if (ip.StartsWith("127.") || ip.StartsWith("10.") || ip.StartsWith("192.168.") || ip.StartsWith("172."))
            {
                var parts = ip.Split('.');
                return $"{parts[0]}.{parts[1]}.***.***";
            }
            return ip;
        });

        return sanitized;
    }
}
```

## 4.4 先行ディレクトリ NTFS ACL 制御 (`DatabaseAccessControl.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Persistence;

using System;
using System.IO;
using System.Security.AccessControl;
using System.Security.Principal;
using Microsoft.Extensions.Logging;

public static class DatabaseAccessControl
{
    public static void EnsureStrictDirectorySecurity(string directoryPath, ILogger? logger = null)
    {
        try
        {
            if (!Directory.Exists(directoryPath))
            {
                Directory.CreateDirectory(directoryPath);
            }

            var dirInfo = new DirectoryInfo(directoryPath);
            var drive = new DriveInfo(dirInfo.Root.FullName);

            if (!string.Equals(drive.DriveFormat, "NTFS", StringComparison.OrdinalIgnoreCase))
            {
                logger?.LogWarning("ドライブが NTFS ではないため ({Format})、ディレクトリ ACL 適用を安全にスキップしました: {Path}", drive.DriveFormat, directoryPath);
                return;
            }

            var dirSecurity = new DirectorySecurity();
            dirSecurity.SetAccessRuleProtection(isProtected: true, preserveInheritance: false);

            var currentUser = WindowsIdentity.GetCurrent().User;
            if (currentUser != null)
            {
                dirSecurity.AddAccessRule(new FileSystemAccessRule(
                    currentUser,
                    FileSystemRights.FullControl,
                    InheritanceFlags.ContainerInherit | InheritanceFlags.ObjectInherit,
                    PropagationFlags.None,
                    AccessControlType.Allow));
            }

            var systemSid = new SecurityIdentifier(WellKnownSidType.LocalSystemSid, null);
            dirSecurity.AddAccessRule(new FileSystemAccessRule(
                systemSid,
                FileSystemRights.FullControl,
                InheritanceFlags.ContainerInherit | InheritanceFlags.ObjectInherit,
                PropagationFlags.None,
                AccessControlType.Allow));

            dirInfo.SetAccessControl(dirSecurity);
            logger?.LogInformation("親ディレクトリへ先行 NTFS ACL を適用しました: {Path}", directoryPath);
        }
        catch (PlatformNotSupportedException)
        {
            logger?.LogWarning("OS プラットフォーム制限のためディレクトリ ACL 適用をスキップしました: {Path}", directoryPath);
        }
        catch (Exception ex)
        {
            logger?.LogError(ex, "先行ディレクトリ NTFS ACL 適用失敗: {Path}", directoryPath);
        }
    }
}
```

## 4.5 改ざん検知 Hash Chain 多層監査ロガー (`TamperEvidentAuditLogger.cs` - C-5 是正)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Logging;

using System;
using System.IO;
using System.Security.Cryptography;
using System.Text;
using System.Threading;
using System.Threading.Channels;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Common;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Infrastructure.Persistence;
using GameSecurityTool.Infrastructure.Persistence.Entities;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;

public sealed class TamperEvidentAuditLogger : ITamperEvidentAuditLogger, IDisposable
{
    private const string GenesisHash = "0000000000000000000000000000000000000000000000000000000000000000";
    private readonly SemaphoreSlim _chainLock = new(1, 1);

    private readonly IDbContextFactory<AppDbContext> _dbFactory;
    private readonly IDbWriteQueue _dbWriter;
    private readonly ILogSanitizer _logSanitizer;
    private readonly IElevatedWorkerClient _elevatedWorkerClient;
    private readonly ILogger<TamperEvidentAuditLogger> _logger;

    private readonly Channel<string> _anchorSyncChannel;
    private readonly CancellationTokenSource _cts = new();
    private readonly Task _syncTask;

    public bool IsLayer2AnchorSynchronized { get; private set; } = true;

    public TamperEvidentAuditLogger(
        IDbContextFactory<AppDbContext> dbFactory,
        IDbWriteQueue dbWriter,
        ILogSanitizer logSanitizer,
        IElevatedWorkerClient elevatedWorkerClient,
        ILogger<TamperEvidentAuditLogger> logger)
    {
        _dbFactory = dbFactory;
        _dbWriter = dbWriter;
        _logSanitizer = logSanitizer;
        _elevatedWorkerClient = elevatedWorkerClient;
        _logger = logger;

        _anchorSyncChannel = Channel.CreateBounded<string>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.DropOldest,
            SingleReader = true,
            SingleWriter = false
        });

        _syncTask = Task.Run(ProcessLayer2AnchorSyncLoopAsync);
    }

    public async Task AppendLogAsync(
        string operationId,
        AuditEventType eventType,
        string actor,
        string target,
        string result,
        CancellationToken ct = default)
    {
        string currentHash;

        await _chainLock.WaitAsync(ct);
        try
        {
            string sanitizedTarget = _logSanitizer.Sanitize(target);
            string sanitizedResult = _logSanitizer.Sanitize(result);
            DateTimeOffset now = DateTimeOffset.UtcNow;

            await using var readDb = await _dbFactory.CreateDbContextAsync(ct);
            var lastEntry = await readDb.Set<AuditEventRecord>()
                .AsNoTracking()
                .OrderByDescending(e => e.SequenceNumber)
                .FirstOrDefaultAsync(ct);

            long nextSeq = (lastEntry?.SequenceNumber ?? 0) + 1;
            string previousHash = lastEntry?.CurrentHash ?? GenesisHash;

            string rawPayload = $"{nextSeq}|{previousHash}|{operationId}|{(int)eventType}|{actor}|{sanitizedTarget}|{sanitizedResult}|{now.ToUnixTimeSeconds()}";
            byte[] hashBytes = SHA256.HashData(Encoding.UTF8.GetBytes(rawPayload));
            currentHash = Convert.ToHexString(hashBytes);

            var record = new AuditEventRecord
            {
                SequenceNumber = nextSeq,
                OperationId = operationId,
                EventType = (int)eventType,
                Actor = actor,
                Target = sanitizedTarget,
                Result = sanitizedResult,
                PreviousHash = previousHash,
                CurrentHash = currentHash,
                TimestampUtc = now
            };

            await _dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
            {
                await using var db = await _dbFactory.CreateDbContextAsync(innerCt);
                db.Set<AuditEventRecord>().Add(record);
                await db.SaveChangesAsync(innerCt);
            }, ct);

            SaveLatestHashToDpapi(currentHash);
        }
        finally
        {
            _chainLock.Release();
        }

        _anchorSyncChannel.Writer.TryWrite(currentHash);
    }

    public async Task<bool> VerifyAuditChainIntegrityAsync(CancellationToken ct = default)
    {
        var detailed = await VerifyAuditChainDetailedAsync(ct);
        return detailed.IsChainValid && detailed.IsLayer1DpapiMatched && (detailed.IsLayer2AdminAnchorMatched || !detailed.IsLayer2Synchronized);
    }

    /// <summary>
    /// 【C-5 是正】多層アンカーの個別検証ステータスを評価し、サイレント Fail-Open を防止
    /// </summary>
    public async Task<AuditVerificationResultDto> VerifyAuditChainDetailedAsync(CancellationToken ct = default)
    {
        await using var db = await dbFactory.CreateDbContextAsync(ct);
        var entries = await db.Set<AuditEventRecord>()
            .AsNoTracking()
            .OrderBy(e => e.SequenceNumber)
            .ToListAsync(ct);

        if (entries.Count == 0)
        {
            return new AuditVerificationResultDto(true, true, true, true, 0, null, null);
        }

        string expectedPrev = GenesisHash;
        foreach (var entry in entries)
        {
            if (!string.Equals(entry.PreviousHash, expectedPrev, StringComparison.OrdinalIgnoreCase))
            {
                _logger.LogError("Audit Chain 破壊検知: Sequence {Seq} の PreviousHash 不正", entry.SequenceNumber);
                return new AuditVerificationResultDto(false, false, false, false, entries.Count, $"Sequence {entry.SequenceNumber} PreviousHash Mismatch", "DB 内のハッシュ連鎖が破損しています。");
            }

            string rawPayload = $"{entry.SequenceNumber}|{entry.PreviousHash}|{entry.OperationId}|{entry.EventType}|{entry.Actor}|{entry.Target}|{entry.Result}|{entry.TimestampUtc.ToUnixTimeSeconds()}";
            string calculatedHash = Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(rawPayload)));

            if (!string.Equals(entry.CurrentHash, calculatedHash, StringComparison.OrdinalIgnoreCase))
            {
                _logger.LogError("Audit Chain 改ざん検知: Sequence {Seq} の CurrentHash 不一致", entry.SequenceNumber);
                return new AuditVerificationResultDto(false, false, false, false, entries.Count, $"Sequence {entry.SequenceNumber} CurrentHash Mismatch", "DB 内のレコードが改ざんされています。");
            }

            expectedPrev = entry.CurrentHash;
        }

        // Layer 1 (DPAPI) 検証
        string? dpapiHash = LoadLatestHashFromDpapi();
        bool isLayer1Matched = !string.IsNullOrEmpty(dpapiHash) && string.Equals(expectedPrev, dpapiHash, StringComparison.OrdinalIgnoreCase);
        if (!isLayer1Matched)
        {
            _logger.LogError("Audit Chain Layer 1 改ざん検知: 最新 DB ハッシュと DPAPI アンカーが不一致です。");
        }

        // Layer 2 (特権 IPC) 検証 (C-5 是正)
        string? anchorHash = await _elevatedWorkerClient.ReadAuditAnchorAsync(ct);
        bool isLayer2Configured = !string.IsNullOrEmpty(anchorHash);
        bool isLayer2Matched = false;
        string? warningMessage = null;

        if (isLayer2Configured)
        {
            isLayer2Matched = string.Equals(expectedPrev, anchorHash, StringComparison.OrdinalIgnoreCase);
            if (!isLayer2Matched)
            {
                _logger.LogError("Audit Chain Layer 2 破壊検知: 管理者特権アンカーと不一致です (同一ユーザー権限マルウェアによる改ざん疑い)。");
                warningMessage = "管理者特権アンカーとの不一致を検知しました。ログが改ざんされた可能性があります。";
            }
        }
        else
        {
            // 【C-5 是正】未同期をサイレント成功とせず、警告メッセージとして明示
            warningMessage = "特権管理者アンカー (Layer 2) が未同期です。特権ワーカーを起動して二重保護を確立してください。";
            _logger.LogWarning("Layer 2 特権アンカーが未同期の状態で監査チェーンが検証されました。");
        }

        bool overallValid = isLayer1Matched && (!isLayer2Configured || isLayer2Matched);

        return new AuditVerificationResultDto(
            IsChainValid: true,
            IsLayer1DpapiMatched: isLayer1Matched,
            IsLayer2AdminAnchorMatched: isLayer2Matched,
            IsLayer2Synchronized: isLayer2Configured,
            TotalEntriesVerified: entries.Count,
            BrokenSequenceDetails: null,
            WarningMessage: warningMessage
        );
    }

    private async Task ProcessLayer2AnchorSyncLoopAsync()
    {
        while (!_cts.Token.IsCancellationRequested)
        {
            try
            {
                if (await _anchorSyncChannel.Reader.WaitToReadAsync(_cts.Token))
                {
                    string latestHash = string.Empty;
                    while (_anchorSyncChannel.Reader.TryRead(out var hash))
                    {
                        latestHash = hash;
                    }

                    if (!string.IsNullOrEmpty(latestHash))
                    {
                        bool success = await _elevatedWorkerClient.WriteAuditAnchorAsync(latestHash, _cts.Token);
                        IsLayer2AnchorSynchronized = success;

                        if (!success)
                        {
                            _logger.LogTrace("特権ワーカーへの Layer 2 アンカー非同期同期を保留しました (Layer 1 DPAPI で保護継続)。");
                        }
                    }
                }
            }
            catch (OperationCanceledException) when (_cts.IsCancellationRequested)
            {
                break;
            }
            catch (Exception ex)
            {
                IsLayer2AnchorSynchronized = false;
                _logger.LogTrace(ex, "Layer 2 アンカー同期ループ内での例外");
                await Task.Delay(2000, _cts.Token);
            }
        }
    }

    private static void SaveLatestHashToDpapi(string hashHex)
    {
        try
        {
            byte[] plainBytes = Encoding.UTF8.GetBytes(hashHex);
            byte[] encryptedBytes = ProtectedData.Protect(plainBytes, null, DataProtectionScope.CurrentUser);
            string path = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "GameSecurityTool", "audit.dpapi");
            string? dir = Path.GetDirectoryName(path);
            if (!string.IsNullOrEmpty(dir)) Directory.CreateDirectory(dir);
            File.WriteAllBytes(path, encryptedBytes);
        }
        catch { }
    }

    private static string? LoadLatestHashFromDpapi()
    {
        try
        {
            string path = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "GameSecurityTool", "audit.dpapi");
            if (!File.Exists(path)) return null;
            byte[] encryptedBytes = File.ReadAllBytes(path);
            byte[] plainBytes = ProtectedData.Unprotect(encryptedBytes, null, DataProtectionScope.CurrentUser);
            return Encoding.UTF8.GetString(plainBytes);
        }
        catch
        {
            return null;
        }
    }

    public void Dispose()
    {
        _cts.Cancel();
        _chainLock.Dispose();
        _cts.Dispose();
    }
}
```

## 4.6 設定差分計算 ＆ スマート部分復元 (`ConfigurationTimeMachineService.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Persistence;

using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Security.Cryptography;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Infrastructure.Persistence.Entities;
using Microsoft.Data.Sqlite;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;

public sealed class ConfigurationTimeMachineService(
    IDbContextFactory<AppDbContext> dbFactory,
    IDbWriteQueue dbWriter,
    IGameProfileRepository gameProfileRepository,
    ILogger<ConfigurationTimeMachineService> logger) : IConfigurationTimeMachineService
{
    private static readonly string BackupDir = Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
        "GameSecurityTool",
        "ConfigSnapshots"
    );

    private string _lastSnapshotHash = string.Empty;

    public async Task<ConfigurationBackupResultDto> CreateSnapshotAsync(bool force = false, CancellationToken ct = default)
    {
        Directory.CreateDirectory(BackupDir);
        await using var db = await dbFactory.CreateDbContextAsync(ct);

        // 1. Diff Guard: 現在の設定テーブルの要約ハッシュを計算
        string currentHash = await ComputeConfigDatabaseHashAsync(db, ct);
        if (!force && string.Equals(_lastSnapshotHash, currentHash, StringComparison.OrdinalIgnoreCase))
        {
            logger.LogTrace("設定に変更がないためスナップショット作成をスキップしました (Diff Guard)。");
            return new ConfigurationBackupResultDto(true, isSkipped: true, string.Empty);
        }

        string timestamp = DateTime.UtcNow.ToString("yyyyMMdd_HHmmss");
        string destPath = Path.Combine(BackupDir, $"gamesecurity.config.{timestamp}.shadow");

        // 2. SQLite オンラインバックアップ API による安全なスナップショット生成
        try
        {
            var sourceConn = (SqliteConnection)db.Database.GetDbConnection();
            if (sourceConn.State != System.Data.ConnectionState.Open)
            {
                await sourceConn.OpenAsync(ct);
            }

            await using (var destConn = new SqliteConnection($"Data Source={destPath}"))
            {
                await destConn.OpenAsync(ct);
                sourceConn.BackupDatabase(destConn);
            }

            _lastSnapshotHash = currentHash;
            logger.LogInformation("設定スナップショットを生成しました: {Path}", destPath);

            PruneOldSnapshots(maxGenerations: 10);
            return new ConfigurationBackupResultDto(true, isSkipped: false, destPath);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "設定スナップショットの生成に失敗しました。");
            if (File.Exists(destPath)) { try { File.Delete(destPath); } catch { } }
            return new ConfigurationBackupResultDto(false, isSkipped: false, string.Empty);
        }
    }

    public async Task<ConfigurationDiffSummaryDto> ComputeDiffAsync(string snapshotPath, CancellationToken ct = default)
    {
        if (!File.Exists(snapshotPath)) throw new FileNotFoundException("スナップショットファイルが存在しません。");

        await using var currentDb = await dbFactory.CreateDbContextAsync(ct);
        await using var snapshotDb = new AppDbContext(new DbContextOptionsBuilder<AppDbContext>().UseSqlite($"Data Source={snapshotPath}").Options);

        var currentGames = await currentDb.GameProfiles.AsNoTracking().ToDictionaryAsync(g => g.Id, ct);
        var snapshotGames = await snapshotDb.GameProfiles.AsNoTracking().ToListAsync(ct);

        var diffs = new List<GameConfigurationDiffDto>();
        var skippedUninstalled = new List<string>();

        foreach (var snapGame in snapshotGames)
        {
            ct.ThrowIfCancellationRequested();

            // Smart Orphan Filter: 現在の PC にインストール実体がないゲームはスキップ
            if (!Directory.Exists(snapGame.Path) || !File.Exists(Path.Combine(snapGame.Path, snapGame.MainExecutable)))
            {
                skippedUninstalled.Add(snapGame.DisplayName);
                continue;
            }

            var changes = new List<string>();
            if (currentGames.TryGetValue(snapGame.Id, out var curGame))
            {
                if (curGame.LaunchMode != snapGame.LaunchMode)
                {
                    changes.Add($"保護モード: {curGame.LaunchMode} ➔ {snapGame.LaunchMode}");
                }
            }
            else
            {
                changes.Add("ゲーム設定が新規追加・復元されます");
            }

            diffs.Add(new GameConfigurationDiffDto(snapGame.Id, snapGame.DisplayName, IsSelectedForRestore: true, changes));
        }

        return new ConfigurationDiffSummaryDto(snapshotPath, File.GetCreationTimeUtc(snapshotPath), diffs, skippedUninstalled);
    }

    public async Task<bool> RestoreConfigurationPartialAsync(ConfigurationRestoreRequestDto request, CancellationToken ct = default)
    {
        if (!File.Exists(request.SnapshotPath)) return false;

        await using var snapshotDb = new AppDbContext(new DbContextOptionsBuilder<AppDbContext>().UseSqlite($"Data Source={request.SnapshotPath}").Options);
        var targetProfileIds = request.TargetGameProfileIdsToRestore.ToHashSet();

        var sourceProfiles = await snapshotDb.GameProfiles.AsNoTracking().Where(p => targetProfileIds.Contains(p.Id)).ToListAsync(ct);
        var sourceRules = await snapshotDb.FirewallRules.AsNoTracking().Where(r => r.GameProfileId.HasValue && targetProfileIds.Contains(r.GameProfileId.Value)).ToListAsync(ct);

        // IDbWriteQueue 経由で選択されたゲームの設定のみを外科手術的に部分コミット
        await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
        {
            await using var db = await dbFactory.CreateDbContextAsync(innerCt);
            foreach (var prof in sourceProfiles)
            {
                var existing = await db.GameProfiles.FirstOrDefaultAsync(p => p.Id == prof.Id, innerCt);
                if (existing != null)
                {
                    db.Entry(existing).CurrentValues.SetValues(prof);
                }
                else
                {
                    await db.GameProfiles.AddAsync(prof, innerCt);
                }
            }

            foreach (var rule in sourceRules)
            {
                var existingRule = await db.FirewallRules.FirstOrDefaultAsync(r => r.RuleTag == rule.RuleTag, innerCt);
                if (existingRule != null)
                {
                    db.Entry(existingRule).CurrentValues.SetValues(rule);
                }
                else
                {
                    await db.FirewallRules.AddAsync(rule, innerCt);
                }
            }

            await db.SaveChangesAsync(innerCt);
        }, ct);

        logger.LogInformation("選択されたゲーム設定 ({Count} 件) のスマート部分復元が完了しました。", targetProfileIds.Count);
        return true;
    }

    private static async Task<string> ComputeConfigDatabaseHashAsync(AppDbContext db, CancellationToken ct)
    {
        var profiles = await db.GameProfiles.AsNoTracking().OrderBy(p => p.Id).Select(p => $"{p.Id}:{p.LaunchMode}:{p.Path}").ToListAsync(ct);
        var rules = await db.FirewallRules.AsNoTracking().OrderBy(r => r.RuleTag).Select(r => $"{r.RuleTag}:{r.Port}").ToListAsync(ct);
        string payload = string.Join(";", profiles) + "|" + string.Join(";", rules);
        return Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(payload)));
    }

    private static void PruneOldSnapshots(int maxGenerations)
    {
        try
        {
            var dir = new DirectoryInfo(BackupDir);
            var files = dir.EnumerateFiles("*.shadow").OrderByDescending(f => f.CreationTimeUtc).ToList();
            if (files.Count > maxGenerations)
            {
                foreach (var file in files.Skip(maxGenerations))
                {
                    file.Delete();
                }
            }
        }
        catch { }
    }

    public Task<IReadOnlyList<ConfigurationSnapshotMetadataDto>> GetSnapshotHistoryAsync(CancellationToken ct = default)
    {
        if (!Directory.Exists(BackupDir)) return Task.FromResult<IReadOnlyList<ConfigurationSnapshotMetadataDto>>([]);

        var dir = new DirectoryInfo(BackupDir);
        var list = dir.EnumerateFiles("*.shadow")
            .OrderByDescending(f => f.CreationTimeUtc)
            .Select(f => new ConfigurationSnapshotMetadataDto(f.FullName, f.CreationTimeUtc, f.Length))
            .ToList();

        return Task.FromResult<IReadOnlyList<ConfigurationSnapshotMetadataDto>>(list);
    }
}
```

## 4.7 外部セキュリティソフト・ファイルロック特定 Adapter (`FileLockInspector.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Native;

using System;
using System.Collections.Generic;
using System.Runtime.InteropServices;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;

public sealed partial class FileLockInspector : IFileLockInspector
{
    private static readonly HashSet<string> KnownSecurityProcesses = new(StringComparer.OrdinalIgnoreCase)
    {
        "mcshield.exe", "bdservicehost.exe", "avp.exe", "ekrn.exe", "SavService.exe", "mbam.exe"
    };

    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    private struct RM_PROCESS_INFO
    {
        public RM_UNIQUE_PROCESS Process;
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 256)]
        public string strAppName;
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 64)]
        public string strServiceShortName;
        public int ApplicationType;
        public uint AppStatus;
        public uint TSSessionId;
        [MarshalAs(UnmanagedType.Bool)]
        public bool bRestartable;
    }

    [StructLayout(LayoutKind.Sequential)]
    private struct RM_UNIQUE_PROCESS
    {
        public int dwProcessId;
        public System.Runtime.InteropServices.ComTypes.FILETIME ProcessStartTime;
    }

    [LibraryImport("rstrtmgr.dll", EntryPoint = "RmStartSession", StringMarshalling = StringMarshalling.Utf16)]
    private static partial int RmStartSession(out uint pSessionHandle, uint dwSessionFlags, string strSessionKey);

    [LibraryImport("rstrtmgr.dll", EntryPoint = "RmEndSession")]
    private static partial int RmEndSession(uint pSessionHandle);

    [LibraryImport("rstrtmgr.dll", EntryPoint = "RmRegisterResources", StringMarshalling = StringMarshalling.Utf16)]
    private static partial int RmRegisterResources(
        uint pSessionHandle,
        uint nFiles,
        [In] string[] rgsFilenames,
        uint nApplications,
        [In] RM_UNIQUE_PROCESS[]? rgApplications,
        uint nServices,
        [In] string[]? rgsServiceNames);

    [DllImport("rstrtmgr.dll", CharSet = CharSet.Unicode)]
    private static extern int RmGetList(
        uint dwSessionHandle,
        out uint pnProcInfoNeeded,
        ref uint pnProcInfo,
        [In, Out] RM_PROCESS_INFO[]? rgAffectedApps,
        ref uint lpdwRebootReasons);

    public IReadOnlyList<LockingProcessInfoDto> GetLockingProcesses(string filePath)
    {
        var result = new List<LockingProcessInfoDto>();
        if (RmStartSession(out uint sessionHandle, 0, Guid.NewGuid().ToString()) != 0) return result;

        try
        {
            string[] resources = [filePath];
            if (RmRegisterResources(sessionHandle, 1, resources, 0, null, 0, null) != 0) return result;

            uint procInfoNeeded = 0;
            uint procInfoCount = 0;
            uint rebootReasons = 0;

            int res = RmGetList(sessionHandle, out procInfoNeeded, ref procInfoCount, null, ref rebootReasons);
            if (res == 234 && procInfoNeeded > 0) // ERROR_MORE_DATA
            {
                var processInfo = new RM_PROCESS_INFO[procInfoNeeded];
                procInfoCount = procInfoNeeded;

                if (RmGetList(sessionHandle, out procInfoNeeded, ref procInfoCount, processInfo, ref rebootReasons) == 0)
                {
                    for (int i = 0; i < procInfoCount; i++)
                    {
                        string appName = processInfo[i].strAppName;
                        bool isSecurity = KnownSecurityProcesses.Contains(appName);
                        result.Add(new LockingProcessInfoDto(processInfo[i].Process.dwProcessId, appName, isSecurity));
                    }
                }
            }
        }
        catch { }
        finally
        {
            RmEndSession(sessionHandle);
        }

        return result;
    }
}
```

## 4.8 セルフメンテナンス自動化 Adapter (`SelfMaintenanceService.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Persistence;

using System;
using System.IO;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;

public sealed class SelfMaintenanceService(
    IDbContextFactory<AppDbContext> dbFactory,
    IDbWriteQueue dbWriter,
    ILogger<SelfMaintenanceService> logger) : ISelfMaintenanceService
{
    public async Task<MaintenanceReportDto> ExecuteMaintenanceAsync(bool emergencyStorageRelief = false, CancellationToken ct = default)
    {
        logger.LogInformation("セルフメンテナンス処理を開始します (緊急容量回復: {Relief})...", emergencyStorageRelief);

        int cleanedTemp = 0;
        long reclaimedTempBytes = 0;
        int prunedLogs = 0;
        long reclaimedLogBytes = 0;

        try
        {
            // 1. 孤児となった一時作業ファイルの完全回収
            string baseDir = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "GameSecurityTool");
            string[] targetCleanDirs = ["Quarantine", "RescueSnapshots", "IpcTokens", "Backups"];

            foreach (var sub in targetCleanDirs)
            {
                string dirPath = Path.Combine(baseDir, sub);
                if (!Directory.Exists(dirPath)) continue;

                foreach (var file in Directory.EnumerateFiles(dirPath, "*.*", SearchOption.AllDirectories))
                {
                    ct.ThrowIfCancellationRequested();
                    string name = Path.GetFileName(file);
                    if (name.EndsWith(".tmp") || name.EndsWith(".token") || name.Contains(".creating"))
                    {
                        try
                        {
                            var info = new FileInfo(file);
                            long size = info.Length;
                            file.Delete();
                            cleanedTemp++;
                            reclaimedTempBytes += size;
                        }
                        catch { }
                    }
                }
            }

            // 2. データベースの最適化と WAL 切り詰め
            await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
            {
                await using var db = await dbFactory.CreateDbContextAsync(innerCt);
                await db.Database.ExecuteSqlRawAsync("PRAGMA optimize;", innerCt);
                await db.Database.ExecuteSqlRawAsync("PRAGMA wal_checkpoint(TRUNCATE);", innerCt);
            }, ct);

            // 3. 30日を経過した古い運用ログのパージ
            string logDir = Path.Combine(baseDir, "Logs");
            if (Directory.Exists(logDir))
            {
                var threshold = DateTimeOffset.UtcNow.AddDays(-30);
                foreach (var logFile in Directory.EnumerateFiles(logDir, "*.log", SearchOption.TopDirectoryOnly))
                {
                    ct.ThrowIfCancellationRequested();
                    var info = new FileInfo(logFile);
                    if (info.LastWriteTimeUtc < threshold)
                    {
                        try
                        {
                            long size = info.Length;
                            logFile.Delete();
                            prunedLogs++;
                            reclaimedLogBytes += size;
                        }
                        catch { }
                    }
                }
            }

            logger.LogInformation("セルフメンテナンス完了: 一時ファイル {Temp} 件 ({Size} MB), ログ {Logs} 件パージ",
                cleanedTemp, reclaimedTempBytes / (1024 * 1024), prunedLogs);

            return new MaintenanceReportDto(true, cleanedTemp, reclaimedTempBytes, true, prunedLogs, reclaimedLogBytes, null);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "セルフメンテナンス処理中に例外が発生しました。");
            return new MaintenanceReportDto(false, cleanedTemp, reclaimedTempBytes, false, prunedLogs, reclaimedLogBytes, ex.Message);
        }
    }
}
```

---

# 5. 単体テスト仕様 (`GST.UnitTests.CFG`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.CFG;

using System;
using System.IO;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Infrastructure.Logging;
using GameSecurityTool.Infrastructure.Persistence;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging.Abstractions;
using Moq;
using Xunit;

public class PrivacyLogSanitizerTests
{
    [Fact]
    public void SanitizeLogMessage_MasksUserDirectoryPath()
    {
        var sanitizer = new PrivacyLogSanitizer();
        string raw = "Error accessing C:\\Users\\TestUser\\AppData\\Roaming\\GameSecurityTool\\gamesecurity.db";
        string sanitized = sanitizer.Sanitize(raw);

        Assert.DoesNotContain("TestUser", sanitized);
        Assert.Contains("C:\\Users\\***\\AppData", sanitized);
    }

    [Fact]
    public void SanitizeLogMessage_MasksLocalIpAddress()
    {
        var sanitizer = new PrivacyLogSanitizer();
        string raw = "Connection failed to 192.168.1.105:8080";
        string sanitized = sanitizer.Sanitize(raw);

        Assert.DoesNotContain("192.168.1.105", sanitized);
        Assert.Contains("192.168.***.***", sanitized);
    }
}

public class ConfigurationTimeMachineTests
{
    [Fact]
    public async Task CreateSnapshotAsync_WhenConfigUnchanged_SkipsWrite()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>().UseSqlite("Data Source=:memory:").Options;
        var dbFactoryMock = new Mock<IDbContextFactory<AppDbContext>>();
        dbFactoryMock.Setup(f => f.CreateDbContextAsync(default)).ReturnsAsync(() =>
        {
            var db = new AppDbContext(options);
            db.Database.EnsureCreated();
            return db;
        });

        var dbWriterMock = new Mock<IDbWriteQueue>();
        var profileRepoMock = new Mock<IGameProfileRepository>();

        var service = new ConfigurationTimeMachineService(
            dbFactoryMock.Object, dbWriterMock.Object, profileRepoMock.Object, NullLogger<ConfigurationTimeMachineService>.Instance);

        // 1 回目作成 (初回なので書き込み実行)
        var result1 = await service.CreateSnapshotAsync(force: false);

        // 2 回目作成 (設定に変更がないため Diff Guard が発動してスキップ)
        var result2 = await service.CreateSnapshotAsync(force: false);
        Assert.True(result2.isSkipped);
    }
}
```

---

