# 03-00: Core Modules Overview & Integration Map

**Document ID:** GST-MOD-OVERVIEW-000  
**Version:** 4.2 (Contracts ThreatReasonCodes SSOT Clarification)
**Status:** Approved Module Baseline  
**Category:** Module Architecture Overview  
**Target Platform:** Windows 10 / Windows 11 (64-bit)  
**Technology Baseline:** .NET 10 LTS / C# 14 / WPF / SQLite + EF Core 10 / Windows DPAPI / Clean 5-Layer  

> **Lifecycle note:** This module specification describes approved design scope. Its presence does not by itself indicate that the module is implemented, Windows-verified, or released.

---

# 1. Purpose & Module Architecture

本書は、GameSecurityTool（GST）のコア実行基盤を構成する **6大コアモジュール** の全体構造、連携関係、共通データ型、およびライフサイクル調停ルールを定義する。

旧設計文書（`TASK-DOC-01` 〜 `29`）におけるすべての設計決定・技術リスク解決パッチ・OS適合性ルールは、本 `modules/` ディレクトリ配下の全6モジュール仕様書へ完全統合・昇格されている。

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        GST 6大コアモジュール体系                       │
├───────────────────┬────────────────────────────────────────────────────┤
│ 1. SYS (システム) │ 03-01: 自己完全性検証, WAL Journal, Safe Mode, ライフサイクル, Anti-Wiper子プロセス制御 │
│ 2. SCAN (スキャン)│ 03-02: 高速並列スキャン, MZ検証, ReparsePoint, VFS共存, Anti-Wiper Canary/バースト │
│ 3. FW (通信制御)  │ 03-03: Zero Trust Firewall, COM API, Anti-Cheat Compatibility Exception │
│ 4. QUAR (隔離復元)│ 03-04: チャンクAEAD暗号化, RestoreTransaction, 2段階コミット │
│ 5. ALLOW (例外)   │ 03-05: 例外許可リスト, TOCTOU防御, Hash優先照会      │
│ 6. CFG / UX (設定)│ 03-06: FeatureControl, Smart Game IME Lock, Wiper設定, Do Not Disturb, ログ匿名化 │
└───────────────────┴────────────────────────────────────────────────────┘
```

---

# 2. 6大モジュール連携マップ (Component Interaction)

```text
                           [ Game Security Tool ]
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        ▼                             ▼                             ▼
┌───────────────┐             ┌───────────────┐             ┌───────────────┐
│  03-01: SYS   │             │  03-02: SCAN  │             │   03-03: FW   │
│・SelfIntegrity│             │・FastScan     │             │・DefaultBlock │
│・LifecycleMgr │─(ゲーム検知)─>│・SecurityEng  │─(リスク判定)─>│・AntiCheat    │
│・CrashRecovery│             │・PE MZ Verify │             │・127.0.0.1    │
└───────┬───────┘             └───────┬───────┘             └───────┬───────┘
        │                             │                             │
        │                             ▼                             │
        │                     ┌───────────────┐                     │
        │                     │  03-05: ALLOW │                     │
        │                     │・AllowList    │<────────────────────┤
        │                     │・TOCTOU Guard │                     │
        │                     └───────┬───────┘                     │
        │                             │                             │
        ▼                             ▼                             ▼
┌───────────────┐             ┌───────────────┐             ┌───────────────┐
│  03-04: QUAR  │             │ 03-06: CFG/UX │             │  Audit Chain  │
│・Chunk AEAD   │<─(隔離/復元)──│・Profile (Mica│──(全操作記録)─>│・GenesisHash  │
│・RestoreTrans │             │・DoNotDisturb │             │・DPAPI 二重保護│
└───────────────┘             └───────────────┘             └───────────────┘
```

---

# 3. 共通データ型・列挙型定義 (Common Contracts & Enum Persistence Governance)

全モジュールおよび全機能（セーブバックアップ、Web保護、LNK検知、クリップボード、MOD診断等）で横断的に利用される不変ドメイン列挙型。  
SQLite への整数値保存における意味的完全性を担保するため、**すべてのメンバーに明示的な数値を固定（追記専用規約）** する。  
配置: `GameSecurityTool.Contracts.Common`

```csharp
namespace GameSecurityTool.Contracts.Common;

public enum RiskLevel
{
    Safe = 0,
    Low = 1,
    Medium = 2,
    High = 3,
    Critical = 4
}

public enum ThreatSeverity
{
    Info = 0,
    Warning = 1,
    Danger = 2
}

public enum RuleDirection
{
    Outbound = 0,
    Inbound = 1
}

public enum AllowType
{
    Hash = 0,
    Signature = 1,
    Path = 2,
    Name = 3
}

public enum ExpirationType
{
    Permanent = 0,
    OneHour = 1,
    TwentyFourHours = 2,
    NextRestart = 3
}

public enum AuditEventType
{
    // --- System & Lifecycle (0〜9) ---
    ScanStarted = 0,
    ScanCompleted = 1,
    FileChanged = 2,
    IntegrityCheckFailed = 3,
    CrashProtectionActivated = 4,
    FullReversionExecuted = 5,

    // --- Firewall & Network (10〜19) ---
    FirewallRuleCreated = 10,
    FirewallRuleRemoved = 11,
    FirewallRuleDriftDetected = 12,
    OrphanedRuleRemoved = 13,
    AntiCheatCompatibilityExceptionApplied = 14,

    // --- Allow List (20〜29) ---
    AllowListAdded = 20,
    AllowListRemoved = 21,
    AllowListExpired = 22,

    // --- Quarantine & Restore (30〜39) ---
    QuarantineExecuted = 30,
    RestoreStarted = 31,
    RestoreCompleted = 32,
    RestoreFailed = 33,
    RollbackStarted = 34,
    RollbackCompleted = 35,
    RollbackFailed = 36,

    // --- Save Backup Feature (40〜49) ---
    SaveBackupCreated = 40,
    SaveBackupFailed = 41,
    SaveBackupDeleted = 42,
    SaveRestoreStarted = 43,
    SaveRestoreCompleted = 44,
    SaveRestoreFailed = 45,
    SaveBackupPolicyChanged = 46,

    // --- Web Link & Advanced Protection (50〜59) ---
    WebLinkReceived = 50,
    WebLinkSanitized = 51,
    WebLinkBlocked = 52,
    WebLinkConfirmed = 53,
    WebBrowserSelected = 54,
    WebLinkEmergencyDetected = 55,
    WebBrowserEmergencyStopped = 56,
    ShortcutChangedDetected = 57,
    ClipboardSanitized = 58,
    ClipboardSanitizationFailed = 59,

    // --- Mod, Diagnostics & AI (60〜69) ---
    ModInspected = 60,
    ModProvenanceVerified = 61,
    ModManifestImported = 62,
    CommunityReportGenerated = 63,
    AiConsentGranted = 64,
    AiConsentRevoked = 65,
    AiExplanationRequested = 66,

    // --- Safe Onboarding & Launch Modes (67〜69) ---
    SafeGameOnboarded = 67,
    GameArchiveRestored = 68,
    StrictLaunchActivated = 69,

    // --- Anti-Wiper Defense (70〜71) ---
    WiperThreatBlocked = 70,
    WiperBreakerTriggered = 71
}

public enum ScanResultStatus
{
    Completed = 0,
    Cancelled = 1,
    Failed = 2
}

public enum RollbackStatus
{
    Applied = 0,
    RolledBack = 1,
    Failed = 2
}

// Threat Reason Codes SSOT
// The authoritative registry is maintained only in:
// Contracts-owned ThreatReasonCodes registry
// Modules and feature specifications must reference the Contracts-owned SSOT.
// approved change-control decision adds AssetTypeMismatch and ScriptLotlCommandDetected to that registry.
```

---

# 4. モジュール間ライフサイクル調停ルール

### 1. 未起動時ゼロ負荷 (0.00% / OFF):
ゲーム未起動時はファイル監視やスキャナーを完全停止し、OS の受動的プロセス通知（`IProcessLifecycleWatcher`）のみを待機する。

### 2. 起動時事前ブロック (Pre-Launch Security):
ゲーム起動検知時、プロセスが通信を開始する前に OS レベルで Firewall ルールおよび WER 遮断を先行適用する。

### 3. 終了時ポスト監査 (Post-Launch Audit):
ゲーム終了検知後、生成されたドロップファイル（`DroppedArtifactTracker`）やレジストリ改ざんを自動清掃・復元し、完了後に監視エンジンを停止（OFF）へ移行する。

### 4. 電源断・BSoD復旧 (Post-Crash Audit):
異常終了が発生した場合、次回起動時に `CrashRecoveryJournal` および `CrashReportTelemetryBlocker` が残留状態を検知し、安全にロールバック・整合性修復・WER元値復元・一時 Firewall ルール除去を実行する。

---

# 5. モジュール仕様書一覧

* **[03-01: SYS - System Safety, Self-Integrity & Lifecycle Specification](03-01_SYS_System_Safety_and_Lifecycle.md)**
* **[03-02: SCAN - File Scanner & Security Engine Specification](03-02_SCAN_File_Scanner_and_Security_Engine.md)**
* **[03-03: FW - Zero Trust Firewall & Network Specification](03-03_FW_Zero_Trust_Firewall.md)**
* **[03-04: QUAR - Quarantine & Restore Specification](03-04_QUAR_Quarantine_and_Restore.md)**
* **[03-05: ALLOW - Allow List & Trust Specification](03-05_ALLOW_Allow_List_and_Trust.md)**
* **[03-06: CFG & UX - Configuration, Profile & Gamer UX Specification](03-06_CFG_Configuration_and_Gamer_UX.md)**
```

---
