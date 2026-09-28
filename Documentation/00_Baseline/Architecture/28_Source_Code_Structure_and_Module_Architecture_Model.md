# GameSecurityTool

# Source Code Structure and Module Architecture Model

## Software Architecture / Module Responsibility / Dependency Control Specification

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-028 |
| Version | 3.2 (Independent Recovery Host Structure Edition) |
| Status | Formal Baseline Specification (Highest Structural Authority) |
| Category | Software Architecture |
| Authority Level | Source Structure Governance |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）の

- ソースコード構造
- プロジェクト分割方針
- モジュール責務
- 依存関係ルール

を定義する。

---

目的：

```text
Maintain Clean 5-Layer Core Architecture + Independent Recovery Host Boundary
Prevent Dependency Pollution
Support Long-Term Maintainability
```

---

# 1. Architecture Philosophy

## 1.1 Core Principle

GST では、機能追加によってレイヤー境界および依存方向を破壊しないことを最重要視する。

---

基本原則：

```text
Separate Responsibility
Strict Dependency Inversion
Protect Core Domain Logic
Contracts as Boundary Bridge
```

---

# 2. High Level Solution Architecture (Clean 5-Layer)

GST は **5つのコアレイヤープロジェクト + 独立Recovery Host + Tests** を採用する。

```text
GameSecurityTool.sln
│
├── Source/
│   ├── GameSecurityTool.Presentation     <-- [UI] WPF, Views, ViewModels
│   ├── GameSecurityTool.Application      <-- [UseCase] ワークフロー調停, Explanation Engine, マッピング
│   ├── GameSecurityTool.Domain           <-- [Core] 純粋ビジネスルール, 判定ロジック (OS/Contracts依存ゼロ)
│   ├── GameSecurityTool.Contracts        <-- [Boundary] 不変DTO, Port (Interface), 共通Enum
│   ├── GameSecurityTool.Infrastructure   <-- [Adapters] SQLite, Win32, WMI, COM, DPAPI/AES
│   ├── GameSecurityTool.Recovery         <-- [Independent Recovery Host] 独立復旧UI/制御、通常DB/設定/UI非依存
│
└── Tests/
    ├── GameSecurityTool.UnitTests        <-- ドメインロジック・DTO単体検証
    ├── GameSecurityTool.IntegrationTests <-- SQLite Queue, Storage, IPC 結合検証
    ├── GameSecurityTool.SecurityTests    <-- TOCTOU, パストラバーサル, Nonce, アトミック復元検証
    └── GameSecurityTool.ArchitectureTests<-- NetArchTest によるレイヤー依存方向・型直結自動検証
```

---

# 3. Dependency Rule (依存方向の絶対規則)

```text
       [ Presentation (WPF) ]
             │         │
             │         ▼
             │   [ Contracts (DTOs / Ports) ]
             ▼         ▲
       [ Application ] ┘ (Domain-Contracts 相互マッピングを独占担当)
             │
             ▼
       [ Domain (Pure C#) ]
             ▲
             │ (Implements Ports)
       [ Infrastructure (Adapters) ]
```

### 依存禁止ルール:
- `Domain` ➔ 他の全レイヤー（`Application`, `Infrastructure`, `Presentation`, `Contracts`）への参照禁止。
- `Application` ➔ `Infrastructure` の具象クラス（`WindowsFirewallManager`, `AppDbContext` 等）への直接参照禁止。
- `Presentation` ➔ `Infrastructure` への直接参照・呼び出しは禁止する。ただし `App.xaml.cs` の Composition Root における DI 登録目的の project reference のみ例外として許可し、Infrastructure の具象型はその登録以外の UI / View / ViewModel コードから直接利用しない。
- `Contracts` ➔ `Presentation`, `Application`, `Infrastructure` への参照禁止。
- **【L-1 是正規約】`Infrastructure` ➔ `Domain.Models.*` への直接型結合禁止（必ず Contracts Port 経由で DTO をやり取りする）。**

---

# 4. プロジェクト別責務定義

## 4.1 `GameSecurityTool.Presentation`
* **責務:** UI 表示、ユーザー入力受付、ViewModel 状態管理、Dispatcher 同期。
* **技術スタック:** WPF, .NET 10 Fluent UI (Mica/Acrylic), `CommunityToolkit.Mvvm`。
* **禁止事項:** DB 直接操作、Win32 API 直接呼出、セキュリティ判定ロジックの実装。起動時の Infrastructure DI 登録は Composition Root としてのみ許可する。Win32 API 実装自体は Infrastructure に配置する。

## 4.2 `GameSecurityTool.Application`
* **責務:** ユースケース実行、トランザクション調停、パイプライン制御、Explanation Engine（多言語解釈）、**Domain ⇄ Contracts マッピングの一元担当**。
* **配置サービス例:** `ScanCoordinator`, `RiskAssessmentService`, `SaveBackupService`, `SaveBackupMaintenanceCoordinator`, `SaveRestoreCoordinator`, `QuarantineCoordinator`, `GameLifecycleProtectionManager`。
* **禁止事項:** `System.IO` による物理ファイル操作、OS 固有 API（WMI, COM, Registry）の直接呼出。

## 4.3 `GameSecurityTool.Domain`
* **責務:** セキュリティ判定エンジン（`SecurityEngineEvaluator`）、確信度計算、保持期限判定（`RescueSnapshotRetentionEvaluator`）、差分参照保持評価（`RetentionEvaluator`）、Universal Rule Scope 解決。
* **特徴:** 外部ライブラリおよび `Contracts` 依存ゼロの Pure C#。
* **命名規約 (M-3 規約):** Domain 内部で定義する列挙型には必ず **`Domain` プレフィックス（例: `DomainRiskLevel`, `DomainAllowType`, `DomainTrustRiskLevel`）** を付与し、サブ名前空間（`Domain.Models.AllowList`, `Domain.Models.TrustEnhancement` 等）で分離・管理する。
* **禁止事項:** 自然言語文字列（`Message = "危険"` 等）の保持、UI 型への依存、EF Core アノテーション、Contracts 型の参照。

## 4.4 `GameSecurityTool.Contracts`
* **責務:** レイヤー境界を越えるデータの不変 DTO（`sealed record`）、Port（Interface）、共通列挙型（`AuditEventType`, `RiskLevel` 等）の定義。
* **特徴:** レイヤー間の結合度を下げるための共有契約層。Domain Entity や DB Entity を UI へ露出させない。

## 4.5 `GameSecurityTool.Infrastructure`
* **責務:** Contracts で定義された Port の具象 Adapter 実装、外部環境（Windows OS, SQLite, ファイルシステム）との物理接続。
* **内部サブモジュール構造:**
  - `Persistence/`: EF Core 10 / SQLite, `SqliteDatabaseWriter` (直列化キュー), `DatabaseAccessControl` (先行 NTFS ACL)
  - `Native/`: Win32 `[LibraryImport]`, `SafeProcessHandle`, WMI ウォッチャー (`WmiProcessLifecycleWatcher`), レジストリ操作 (`CrashReportTelemetryBlocker`)
  - `Firewall/`: Out-of-Process UAC Named Pipe IPC クライアント (`WindowsFirewallManager`), 昇格ワーカー (`ElevatedFirewallService`)
  - `Quarantine/`: 64KB チャンク分割ストリーミング AEAD (`AesGcm`) + DPAPI 鍵保護, アトミック再封緘
  - `SaveBackup/`: Standard ZIP コンテナストレージ, Argon2id + AES-256, アトミックロールバック退避 (`FileSystemRescueSnapshotStorage`)
  - `Logging/`: 構造化ログ, `PrivacyLogSanitizer` (`[GeneratedRegex]`), ハッシュチェーンロガー (`TamperEvidentAuditLogger`)

---

# 5. Architecture Verification (NetArchTest)

レイヤー依存違反および型直結アンチパターンをビルド・CI パイプラインで機械的に検知するため、`GameSecurityTool.ArchitectureTests`（NetArchTest ルール 1〜6）を常時実行する。

---

# 6. Final Architecture Statement

GST のソースコード構造は、5つのコアレイヤーをClean/Hexagonal境界として維持しつつ、通常アプリとは独立したRecovery Host境界を追加することで、機能を増やすためだけでなく、**5年以上にわたり安全に成長・保守し続けるための境界防御システム** として設計する。

---

Final Principle:

```text
Keep Layers Clean
Strict Inversion of Control
Enforce Boundary by Tests
Build For The Future
```

---

End of Document
```

---
