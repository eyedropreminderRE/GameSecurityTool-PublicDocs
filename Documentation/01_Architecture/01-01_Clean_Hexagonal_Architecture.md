# 01-01: Clean Hexagonal Architecture

**Document ID:** GST-ARCH-BASELINE-002-PART1  
**Parent Document:** Architecture & Technology Baseline v2.0 (GST-ARCH-BASELINE-002)  
**Category:** Architecture Baseline  
**Version:** 3.2 (Domain Prefix, Port Boundary & Composition Root Governance Edition)
**Status:** Approved Baseline  

---

# 4. Solution Architecture

Multi-Project Solution を採用する。

```text
GameSecurityTool.sln
│
├─ GameSecurityTool.Presentation
├─ GameSecurityTool.Application
├─ GameSecurityTool.Domain
├─ GameSecurityTool.Contracts
├─ GameSecurityTool.Infrastructure
└─ Tests
```

## 4.1 Project 分割方針
Project 名は現在の Solution 構成を優先する。ただし以下の Dependency Rule は厳格に維持し、変更を禁止する。

---

# 5. Clean Architecture & Hexagonal Boundary

## 5.1 依存方向の絶対規則

```text
       [ Presentation (WPF) ]
             │         │
             │         ▼
             │   [ Contracts (DTOs / Port Interfaces / Enums) ]
             ▼         ▲
       [ Application ] ┘ (Domain Enum と Contracts Enum のマッピングを行う)
             │
             ▼
       [ Domain (Pure Business Logic) ]
             ▲
             │ (Ports 実装 / 参照)
       [ Infrastructure (Win32, EF Core, DPAPI, COM) ]
```

### レイヤー間結合の原則
1. **Domain 層の完全独立 (最内層・Pure C# - C1 是正)**:
   - `System.*` のプリミティブおよび Domain 内部 POCO のみで構成。
   - `Presentation`、`Infrastructure`、EF Core アノテーション、WPF 型、Win32 API に一切依存しない。
   - **【絶対規約】** `Contracts` に定義された UI 向け DTO や共通 Enum（`Contracts.Common` 等）にも**絶対に依存してはならない**。Domain 層で独自に定義する Enum には必ず **`Domain` プレフィックス（例: `DomainRiskLevel`, `DomainAllowType`, `DomainTrustRiskLevel`）** を付与し、機能別サブ名前空間（`Domain.Models.AllowList`, `Domain.Models.TrustEnhancement` 等）で型衝突を物理排除する（M-3 規約）。
2. **Contracts 層の責務 (Boundary Contracts & Ports)**:
   - レイヤー間の境界を跨ぐ DTO（`sealed record`）および Port（Interface）、共有 Enum（`Contracts.Common`）を定義。
   - Presentation と Application、Application と Infrastructure の結合度を下げるための共有定義層。
3. **Application 層の責務 (Use Cases & Orchestration & Mapping)**:
   - ビジネスユースケース、ワークフロー制御、トランザクション調停を実装。
   - Domain および Contracts のみを参照し、Infrastructure の具象型には直接依存しない。
   - **【マッピング責務の独占】Domain 固有の `Domain*` Enum / POCO と、Contracts の Enum / DTO 間の相互変換は、必ず Application 層（`RiskAssessmentService`, `ExplanationEngine` 等）が一元的に担う。**
4. **Infrastructure 層の責務 (Adapters & Port Implementation)**:
   - Contracts で定義された Port インターフェースを実装し、外部（Windows OS, SQLite, DPAPI, Firewall COM 等）と物理接続する。
   - **【L-1 是正規約】Infrastructure 層が Application 層のマッピングをバイパスして Domain 内部型（`Domain*` Enum や POCO）を直接インスタンス化・操作・代入することを厳禁とする（必ず Contracts Port 経由で DTO をやり取りする）。**
5. **Presentation 層の責務 (UI & Interaction)**:
   - Application の Use Case / Query を実行し、Contracts DTO を受け取って ViewModel / View を駆動する。UI / View / ViewModel から Infrastructure を直接利用してはならない。
   - **Composition Root exception:** `App.xaml.cs` の起動時 DI 登録に限り Infrastructure の具象実装を登録するための project reference を許可する。これは UI 実装への Infrastructure 利用許可ではない。Windows/Win32 API 自体は Presentation に置かず Infrastructure に隔離する。

---

# 6. Layer Responsibility

## 6.1 Presentation Layer
* **責務:** View, ViewModel, User Interaction, User Consent, Display
* **担当例:** RiskExplanation表示, Timeline表示, Confirmation Dialog, Settings UI
* **禁止事項:** 
  * Database 直接操作
  * Firewall 直接操作
  * Registry 直接操作
  * OS API 直接利用（Composition Root の契約呼出しを除く。Win32 API 実装は Infrastructure 所有）
  * Security/Risk 判断ロジックの実装

## 6.2 Application Layer
* **責務:** Use Case, Workflow, Orchestration, Transaction Coordination, Domain-Contracts Mapping
* **担当例:**
  * `RiskAssessmentService` (AllowList / SCAN マッピング調停)
  * `ExplanationEngine` (説明生成エンジン・Trust マッピング調停)
  * `SaveBackupMaintenanceCoordinator` (差分参照依存グラフ構築・世代管理調停)
  * `GameLifecycleProtectionManager` (ライフサイクル調停 ＆ 監視健全性伝播)
  * `SaveRestoreCoordinator` (RescueSnapshot アトミックロールバック調停)
* **禁止事項:**
  * Infrastructure 具象型の直接利用（`new WindowsFirewallManager()` 等）
  * `DbContext` の直接利用
  * Windows API / Win32 ハンドルの直接利用
  * `System.IO` による物理ファイル直接操作

## 6.3 Domain Layer
* **責務:** Business Rule, Security Policy, Risk Evaluation, Rule Evaluation, Classification
* **保持するもの (Pure C#):**
  * `SecurityEvaluationResult` (`Domain.Models.AllowList`)
  * `ConfidenceAssessmentResult` (`Domain.Models.TrustEnhancement`)
  * `DomainRiskLevel` / `DomainTrustRiskLevel` (**必ず `Domain` プレフィックスを付与 - M-3 規約**)
  * `RescueSnapshotRetentionEvaluator` (純粋保持期限評価)
  * `RetentionEvaluator` (純粋差分参照保持評価)
  * `DetectionSignal`
* **禁止事項:**
  * UI Text / 自然言語文言リソース
  * Localization
  * WPF / XAML 依存型
  * EF Core / SQLite 依存
  * Windows API / FileSystem 物理 API
  * **Contracts 名前空間への依存**

## 6.4 Contracts Layer
* **責務:** 外部境界 DTO 定義、Port インターフェース定義、製品横断 Enum 定義
* **担当 DTO 例:**
  * `RiskAssessmentResultDto`
  * `RiskExplanationDto`
  * `FirewallRuleDto`
  * `WebHostRuleDto`
  * `SaveBackupSnapshotDto`
  * `AuditVerificationResultDto`
* **規則:** Domain Entity や Database Entity を UI へ直接公開しない。

---

# 7. Architecture Extension for Trust Enhancement

Security Trust Enhancement は既存 Security Engine を置換しない。
```text
Existing Security Engine  +  Trust Enhancement Layer
```

### 追加データフロー:
```text
[ Domain Layer: TrustEnhancement ]
ConfidenceAssessmentResult (純粋な判定データ: DomainTrustRiskLevel, DetectionSignals)
    ↓
[ Application Layer ]
ExplanationEngine (多言語リソース適用・DomainTrustRiskLevel ➔ Contracts.Common.RiskLevel マッピング)
    ↓
[ Contracts Layer ]
RiskExplanationDto (不変 DTO)
    ↓
[ Presentation Layer ]
RiskExplanationView / ViewModel (UI 描画)
```

---

# 9. Layer Responsibility Detailed Definition

## 9.1 Domain Layer

Domain Layer は GST のセキュリティ判断ロジックの中心である。

### 9.1.1 Domain Layer で扱う Security 情報
Domain では「判断に必要な事実（Signals）」のみを保持する。
* **保持可能シグナル例:**
  * `FileUnsigned`
  * `NewFileDetected`
  * `UnknownPublisher`
  * `SuspiciousLocation`
  * `HashMismatch`
  * `SignatureInvalid`
* **保持禁止例 (UI 文言・自然言語):**
  * `"このファイルは危険です"`
  * `"削除してください"`
  * `"未署名なので危険です"`
* **理由:** Domain はユーザーの表示言語、文化、UI フレームワークから完全に独立していなければならない。

### 9.1.2 Domain Layer 禁止事項
* **UI 依存禁止:** WPF, ViewModel, Localization, Message Resource
* **Infrastructure 依存禁止:** SQLite, EF Core, Registry, Firewall API, WMI, AMSI, Win32 API, FileSystem API
* **Contracts 依存禁止:** Boundary DTO や Port への依存
* Domain は Windows 環境やネイティブ DLL が存在しない Linux/macOS 環境の Unit Test でも単体実行可能でなければならない。

## 9.2 Application Layer

Application Layer は Use Case 実行層である。

### 9.2.1 担当 Workflow 例
* **Security Engine Workflow:**
  `File Event -> Identity Validation -> IRiskAssessmentService (Application) -> SecurityEngineEvaluator (Domain) -> SecurityEvaluationResult -> DTO Mapping -> RiskAssessmentResultDto -> FastFileSystemScanner (Infra)`
* **Explanation Workflow:**
  `Domain Risk Evaluation -> ConfidenceAssessmentResult -> ExplanationEngine (Application) -> RiskExplanationDto -> UI`
* **Backup Workflow:**
  `Backup Request -> Manifest Check -> Changed File Detection -> Backup Pipeline -> Snapshot Record`
* **Quarantine Workflow:**
  `Detection -> User Consent -> Quarantine Transaction (Chunked AEAD) -> Audit Record`

---

# 10. Port & Adapter Architecture

## 10.1 Port Definition
Port は Contracts 境界で Interface として定義する。
* **Port 例:**
  * `IRiskAssessmentService`
  * `IFirewallManager`
  * `IProcessLifecycleWatcher`
  * `ISaveBackupStorage`
  * `IRescueSnapshotStorage`
  * `ITamperEvidentAuditLogger`

## 10.2 Adapter Implementation
Infrastructure 側で Port を実装する。
```text
IFirewallManager          <─── [implements] ─── WindowsFirewallManager
IProcessLifecycleWatcher  <─── [implements] ─── WmiProcessLifecycleWatcher
IRescueSnapshotStorage    <─── [implements] ─── FileSystemRescueSnapshotStorage
```

## 10.3 Port 利用ルール
Application および Domain は Interface のみを利用し、Windows API / COM / Native 呼び出しを直接行ってはならない。また、Infrastructure は Domain 型を直接バイパスせず、Contracts Port 経由でやり取りする（L-1 規約）。

---

# 11. Contracts Design

Contracts Project はレイヤー間の境界契約を管理する。

## 11.1 DTO Rule
境界 DTO はイミュータブルな `sealed record` を基本とする。
```csharp
public sealed record RiskAssessmentResultDto(
    RiskLevel Level,
    int Score,
    IReadOnlyList<ThreatReasonDto> Reasons,
    DateTimeOffset EvaluatedAt,
    string RuleVersion);
```

## 11.2 Entity 公開禁止
Database Entity や Domain Entity を UI 層へ直接公開することを禁止する。
```text
Database Entity -> Mapper -> DTO -> UI (Presentation)
Domain Entity -> Mapper -> DTO -> UI (Presentation)
```

---

End of Document
```

---

