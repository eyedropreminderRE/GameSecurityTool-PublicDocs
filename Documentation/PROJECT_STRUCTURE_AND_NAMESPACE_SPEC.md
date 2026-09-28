# GameSecurityTool Project Structure & Namespace Specification

**Document ID:** GST-SPEC-PROJ-001  
**Version:** 2.3 (Independent Recovery Host Project / Artifact Naming Decision)
**Target Platform:** Windows 10 / Windows 11 (64-bit) / .NET 10 LTS / C# 14  
**Solution File:** `GameSecurityTool.sln`  

> **Lifecycle note:** This document describes approved design/specification scope. Its presence does not by itself indicate that the described behavior is implemented, Windows-verified, or released.

---

# 1. Solution & Physical Directory Structure

GST の**目標トポロジ**は **7 Production Projects + 4 テストプロジェクト**とする。approved Recovery Host architecture decisionで `GameSecurityTool.Recovery` の設計境界を確定した。現行 Source が実際に保持している Production Project は引き続き6件であり、Recovery Host のproject/source生成は the approved implementation-readiness gateに行う。`GameSecurityTool.ElevatedWorker` と `GameSecurityTool.Recovery` は Clean 5-Layer の一部ではなく、それぞれ特権処理と独立復旧処理をホストする executable host project とする。

```text
GameSecurityTool/
├── Source/
│   ├── GameSecurityTool.Presentation/      <-- [UI] WPF Views, ViewModels, Behaviors
│   ├── GameSecurityTool.Application/       <-- [UseCase] ワークフロー調停, Explanation Engine
│   ├── GameSecurityTool.Domain/            <-- [Core] 純粋ビジネスルール, 判定ロジック
│   ├── GameSecurityTool.Contracts/         <-- [Boundary] 不変DTO, Port (Interface), 共通Enum
│   ├── GameSecurityTool.Infrastructure/    <-- [Adapters] SQLite, Win32, WMI, COM, DPAPI/AES
│   ├── GameSecurityTool.ElevatedWorker/     <-- [Privileged Host] ElevatedWorker.exe / Named Pipe server composition
│   └── GameSecurityTool.Recovery/           <-- [Recovery Host] GameSecurityTool.Recovery.exe / independent Safe Mode recovery
│
├── Tests/
│   ├── GameSecurityTool.ArchitectureTests/ <-- NetArchTest によるレイヤー依存自動検証
│   ├── GameSecurityTool.UnitTests/         <-- Domain / Application 単体テスト
│   ├── GameSecurityTool.IntegrationTests/  <-- SQLite Queue, Standard ZIP, Named Pipe 結合テスト
│   └── GameSecurityTool.SecurityTests/     <-- OOM耐性, TOCTOU, PID Reuse, Rollback テスト
│
└── Documentation/                          <-- 仕様書・ベースラインドキュメント群
```

---

# 2. プロジェクト間参照関係 (Project References)

```text
                      [ Presentation (WPF) ]
                             │         │
                             │         ▼
                             │   [ Contracts (DTOs/Ports) ]
                             ▼         ▲
                       [ Application ] ┘
                             │
                             ▼
                       [ Domain (Pure C#) ]
                             ▲
                             │ (Implements Ports)
                       [ Infrastructure ]
```

### 参照定義 (`.csproj` 依存関係):
- `GameSecurityTool.Domain.csproj`: **外部参照ゼロ**（Pure C#）
- `GameSecurityTool.Contracts.csproj`: **外部参照ゼロ**（純粋 DTO / Port ライブラリ）
- `GameSecurityTool.Application.csproj`: `GameSecurityTool.Domain`, `GameSecurityTool.Contracts`
- `GameSecurityTool.Infrastructure.csproj`: `GameSecurityTool.Domain`, `GameSecurityTool.Contracts`
- `GameSecurityTool.Presentation.csproj`: `GameSecurityTool.Application`, `GameSecurityTool.Contracts`, and `GameSecurityTool.Infrastructure` for the **Composition Root only** (`App.xaml.cs` DI registration). UI / View / ViewModel code must not directly consume Infrastructure types.
- `GameSecurityTool.Recovery.csproj`: **planned standalone executable host**; must not depend on the normal SQLite database, normal configuration store, MainWindow, or normal feature initialization. Its future implementation must own the embedded `RecoveryConfig.json` recovery artifact.

---

# 3. C# 名前空間規約 (Namespace Guidelines)

全 C# ファイルで **File-Scoped Namespace** を強制する。

### 3.1 `GameSecurityTool.Domain`
- `GameSecurityTool.Domain.Models` — エンティティ、集約ルート
- `GameSecurityTool.Domain.ValueObjects` — 不変値オブジェクト（`GameId`, `ManifestHash` 等）
- `GameSecurityTool.Domain.Services` — 純粋ドメインサービス（`SecurityEngineEvaluator`, `ManifestHashCalculator` 等）
- `GameSecurityTool.Domain.Enums` — ドメイン固有列挙型

### 3.2 `GameSecurityTool.Contracts`
- `GameSecurityTool.Contracts.Common` — 横断的共通 Enum（`AuditEventType`, `RiskLevel`, `ThreatSeverity` 等）
- `GameSecurityTool.Contracts.Dtos` — 不変 DTO（`sealed record`）
- `GameSecurityTool.Contracts.Interfaces` — 境界 Port インターフェース（`IFirewallManager`, `ISaveBackupService` 等）

### 3.3 `GameSecurityTool.Application`
- `GameSecurityTool.Application.Services` — ユースケース調停サービス、HostedService
- `GameSecurityTool.Application.Features.SaveBackup` — セーブバックアップユースケース
- `GameSecurityTool.Application.Features.WebLinkProtection` — Webリンク制御ユースケース
- `GameSecurityTool.Application.Features.TrustEnhancement` — Explanation Engine、タイムラインクエリ

### 3.4 `GameSecurityTool.Infrastructure`
- `GameSecurityTool.Infrastructure.Persistence` — EF Core 10, `AppDbContext`, `SqliteDatabaseWriter` (直列化キュー)
- `GameSecurityTool.Infrastructure.Persistence.Entities` — DB エンティティレコード
- `GameSecurityTool.Infrastructure.Persistence.Repositories` — Contracts Port 実装リポジトリ
- `GameSecurityTool.Infrastructure.Native` — Win32 `[LibraryImport]`, WMI ウォッチャー, レジストリ操作
- `GameSecurityTool.Infrastructure.Firewall` — Out-of-Process UAC Named Pipe クライアント, 昇格ワーカー
- `GameSecurityTool.Infrastructure.Quarantine` — 64KB チャンク分割ストリーミング AEAD (`AesGcm`)
- `GameSecurityTool.Infrastructure.SaveBackup` — Standard ZIP コンテナストレージ, Argon2id + AES-256
- `GameSecurityTool.Infrastructure.Logging` — `PrivacyLogSanitizer` (`[GeneratedRegex]`), ハッシュチェーンロガー

### 3.5 `GameSecurityTool.Presentation`
- `GameSecurityTool.Presentation.Views` — XAML Views, Controls
- `GameSecurityTool.Presentation.ViewModels` — `CommunityToolkit.Mvvm` ViewModels
- `GameSecurityTool.Presentation.Services` — `DispatcherService` (スレッド同期)

### 3.6 `GameSecurityTool.ElevatedWorker`
- **Project role:** 独立した特権 Host / Composition executable。Clean 5-Layer の第6層ではなく、Main GUI から Named Pipe 経由で要求された管理者権限操作をホストする。
- **Artifact:** `ElevatedWorker.exe` に固定。`GameSecurityTool.ElevatedWorker.exe` は正式な成果物名として使用しない。
- **Ownership:** Worker executable とその起動時 Composition は本プロジェクトが所有し、既存の Contracts IPC DTO / Port を利用する。GUI Presentation や Application の UI 状態・責務は持たない。

### 3.7 `GameSecurityTool.Recovery`
- **Project role:** 独立 Recovery Host / Composition executable。Clean 5-Layer の第6/7層ではなく、通常アプリが起動不能でも実行可能な最小 Safe Mode recovery host とする。
- **Artifact:** `GameSecurityTool.Recovery.exe` に固定。これは通常 `GameSecurityTool.exe` UI の代替画面ではない。
- **Recovery artifact ownership:** `RecoveryConfig.json` は本 Host assembly に embedded resource として保持する正本artifactとする。外部mutable fileを代替正本として扱わない。
- **Dependency boundary:** Normal SQLite DB、normal configuration store、MainWindow、normal feature initializationへの依存を持たない。Direct/manual launchを許可する。
- **Integrity/fail-closed:** Host自身の必要な integrity/configuration 検証に失敗した場合は recovery mutation を開始しない。approved Recovery Host architecture decisionは完全にuntrustedな配布物からの復旧保証を追加しない。
- **Implementation gate:** approved Recovery Host architecture decisionでproject/artifactの設計境界のみ確定。実際の `.csproj`、source、embedded resource、startup integration の生成は the implementation-readiness gate is satisfied before generation禁止。

---