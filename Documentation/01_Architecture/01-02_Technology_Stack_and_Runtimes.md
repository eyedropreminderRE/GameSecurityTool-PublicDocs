# 01-02: Technology Stack and Runtimes

**Document ID:** GST-ARCH-BASELINE-002-PART2  
**Version:** 3.3 (DI Lifetime & Password Memory Safety Hardened)  
**Parent Document:** 00_Formal_Baseline_Overview.md  
**Category:** Technology & Coding Baseline  
**Status:** Approved Baseline  

---

# 3. Development Environment

* **Target OS:** Windows 11 (64-bit) / Windows 10 (22H2 以降)
* **Target IDE:** Visual Studio 2026 (または最新の .NET 10 対応 IDE)
* **Runtime & Language:** .NET 10 LTS (`<TargetFramework>net10.0-windows</TargetFramework>`), C# 14
* **UI Framework:** WPF, Windows 11 Fluent UI (WPF-UI / Mica & Acrylic 素材), CommunityToolkit.Mvvm
* **Generic Host:** `Microsoft.Extensions.Hosting` による DI / Logging / Configuration / Lifetime 一元管理

---

# 12. C# 14 Coding Baseline

GST では、コードの安全性・保守性・型安全性を担保するため、C# 14 / .NET 10 の言語機能を厳格に適用する。

## 12.1 Nullable Reference Types
全プロジェクトで Nullable 参照型を強制する。
```xml
<Nullable>enable</Nullable>
<TreatWarningsAsErrors>true</TreatWarningsAsErrors>
```
* **品質基準:** `Nullable Warning = 0`。`#nullable disable` による局所的無効化は原則禁止。

## 12.2 File Scoped Namespace
全 C# ファイルで File-scoped Namespace を採用する。
```csharp
namespace GameSecurityTool.Domain.Models;
```
* 旧形式のブロック構文（`namespace Foo { ... }`）の新規利用は禁止する。

## 12.3 Immutable Boundary
レイヤー境界を跨ぐデータは、完全な変更不可（Immutable）設計とする。
```csharp
public sealed record RiskAssessmentDto(
    string Id,
    RiskLevel Level,
    double Confidence,
    IReadOnlyList<string> EvidenceIds);
```
* `record`、`IReadOnlyList<T>`、`ImmutableArray<T>` を優先使用する。

## 12.4 Primary Constructor
適用可能な Class / Record では Primary Constructor を積極的に利用する。
```csharp
public sealed class ScannerService(
    IFileSystemProvider fileSystemProvider,
    ILogger<ScannerService> logger) : IScannerService
{
    public void StartScan()
    {
        logger.LogInformation("スキャン処理を開始します。");
        fileSystemProvider.InitializeDirectoryAccess();
    }
}
```
※ DI 登録や可読性の観点から不利となる場合は、従来の明示的 Constructor の利用を認める。

## 12.5 Collection Expression
配列やリストの初期化は Collection Expression（`[...]`）を統一使用する。
```csharp
var activeRules = [rule1, rule2, ..defaultRules];
```

## 12.6 Pattern Matching
分岐処理は Pattern Matching（`switch` 式）を活用し、網羅性を保証する。
```csharp
return riskLevel switch
{
    RiskLevel.Critical => ActionPolicy.BlockImmediately,
    RiskLevel.High     => ActionPolicy.RequireUserConsent,
    RiskLevel.Medium   => ActionPolicy.LogWarning,
    RiskLevel.Low      => ActionPolicy.Allow,
    _                  => ActionPolicy.RequireUserConsent // Fail-Safe
};
```

## 12.7 AI / 実装における必須禁止事項
提示・マージされるすべてのコードは以下の条件を必須とする：
* **TODO / 未実装コメントの禁止**: `// TODO:`, `throw new NotImplementedException();` の残置禁止。
* **擬似コード・省略記号の禁止**: `// ... 省略 ...`, `/* 実装は後で */` 等の省略はビルド破壊とみなす。
* **コンパイル不能コードの禁止**: Namespace、参照関係、型定義の整合性が完全に取れた完成コードのみをマージ対象とする。

---

# 14. Generic Host / Dependency Injection

GST は Desktop GUI アプリケーションであるが、内部基盤は `Microsoft.Extensions.Hosting` による Generic Host 構成を採用する。

## 14.1 Host ライフサイクル構成
```text
Host.CreateApplicationBuilder(args)
        ↓
ConfigureServices (DI 登録)
        ↓
AppHost.Build()
        ↓
IHost.StartAsync() (バックグラウンドワーカー起動)
        ↓
WPF MainWindow 起動
        ↓
IHost.StopAsync() (安全なシャットダウン)
```

## 14.2 Service 登録規則
DI コンテナへの登録は各レイヤーの Extensions メソッドに集約する。
* `services.AddDomainServices();`
* `services.AddApplicationServices();`
* `services.AddInfrastructureServices(configuration);`
* `services.AddPresentationServices();`

### 依存解決の規則:
* **許可:** `Application -> Port Interface -> Infrastructure Implementation`
* **禁止:** Application からの具象クラス直接登録・要求
* **禁止:** `ServiceLocator.GetService<T>()` のようなアンチパターン

## 14.3 Static Service Container の完全禁止
```csharp
// 禁止: 追跡不能・テスト困難・ライフサイクル破綻を招く
public static class ServiceLocator
{
    public static IServiceProvider Provider { get; set; }
}
```
すべてのサービス依存は Constructor Injection 経由で明示的に受け渡す。

## 14.4 DI ライフタイム規約 (HIGH-02 是正)
内部状態を持ち、並行アクセス時のロック制御や参照カウントを行うサービスは、DI コンテナで必ず **Singleton 登録** とし、多重インスタンス生成による状態破壊を防ぐ。
* **Singleton 必須コンポーネント例:** `TamperEvidentAuditLogger` (ハッシュ連鎖ロック), `CrashReportTelemetryBlocker` (参照カウント), `WindowsFirewallManager` (IPCトランザクションロック)
* これらを `Scoped` や `Transient` として登録することを厳禁とする。

---

# 15. Application Lifetime Management

Hosted Service（`IHostedService` / `BackgroundService`）を活用し、バックグラウンドタスクのライフサイクルを Generic Host で統合管理する。

### 管理対象ワーカー:
* `ScanWorker`: キュー監視とファイルスキャンのバックグラウンド実行
* `BackupWorker`: セーブデータの変更検知とバックアップ処理
* `SqliteDatabaseWriter`: SQLite への直列化書き込み処理（`IDbWriteQueue` 実装）
* `AuditWriter`: 監査ログ（Hash Chain）の追記処理
* `NotificationWorker`: UI への非同期イベント通知

---

# 23. Target Framework / Platform Targeting

Windows 専用 API（COM, Native API, Win32）を利用するプロジェクトでは、Target Framework を明示する。
```xml
<TargetFramework>net10.0-windows</TargetFramework>
```

## 23.1 CA1416 (Platform Compatibility) 対応方針
プラットフォーム警告の `NoWarn=CA1416` による一括抑制を禁止する。
以下の4方針に従って安全に対処する：

1. **Windows Target Framework の指定**: Windows 専用ライブラリ（`Infrastructure` 等）に `<TargetFramework>net10.0-windows</TargetFramework>` を明示。
2. **Infrastructure への OS 依存隔離**: OS 専用呼び出しをすべて Infrastructure Layer に閉じ込める。
3. **Platform Guard の明示**: 必要に応じて `OperatingSystem.IsWindows()` / `OperatingSystem.IsWindowsVersionAtLeast(10, 0, ...)` でガード。
4. **.NET 標準 API への置換**: ネイティブ API を呼ぶ前に、まず .NET 10 の標準 Managed API を優先採用する。

---

# 57. Package Management Baseline ＆ 暗号化標準

採用する外部 NuGet パッケージは最小限に絞り込み、セキュリティレビューおよびライセンス適合性（`34_License` 準拠）を確認したパッケージのみを使用する。

### 採用パッケージ一覧:
* `Microsoft.Extensions.Hosting` (Generic Host 基盤)
* `Microsoft.Extensions.Logging` (Structured Logging)
* `Microsoft.Extensions.DependencyInjection` (DI コンテナ)
* `CommunityToolkit.Mvvm` (WPF / MVVM Source Generators)
* `Microsoft.EntityFrameworkCore` (ORM 基盤)
* `Microsoft.EntityFrameworkCore.Sqlite` (SQLite プロバイダ)
* `Microsoft.Data.Sqlite` (SQLite ネイティブ接続)
* **`Konscious.Security.Cryptography.Argon2` (公式承認: ポータブル KDF / PC 移行暗号化基盤 / MIT ライセンス)**
* **`SharpCompress` (公式承認: .7z / .rar / .zip のマネージド高速ストリーミング展開 ＆ ポータブルゲーム安全展開基盤 / MIT ライセンス)**

---

## 57.1 全製品共通 暗号アルゴリズム ＆ KDF パラメータ規約

GST における暗号パラメータは、製品全体で以下の通り一元固定し、モジュールごとの独自設定・PBKDF2 混在を厳禁とする。

| 用途 | アルゴリズム / 方式 | パラメータ仕様 |
| :--- | :--- | :--- |
| **Password KDF (バックアップ / PC移行)** | **Argon2id** (`Konscious.Security`) | **Iterations = 3, MemorySize = 64 MB (65536 KB), DegreeOfParallelism = 4, Salt = 16 Bytes (128-bit CS-PRNG)** |
| **Payload 暗号化 (隔離 / セーブZIP)** | **AES-256-GCM (Chunked AEAD)** | **KeySize = 256-bit, Nonce = 12 Bytes (チャンク別一意乱数), Tag = 16 Bytes (128-bit AuthTag), ChunkSize = 64 KB (65536 Bytes)** |
| **Local Secret 保護 (同一端末内)** | **Windows DPAPI** | **Scope = DataProtectionScope.CurrentUser** |
| **改ざん検知 / 完全性 Hash** | **SHA-256** | **256-bit Hash, 秒精度タイムスタンプ正規化** |
| **マルチアーカイブ展開 (本体復元 / 安全展開)** | **SharpCompress** | **.7z (LZMA/LZMA2), .rar, .zip のマネージドストリーミング展開 (Zip Slip 遮断)** |

### パスワード変数のメモリ安全性規約 (CRIT-06 是正)
* **`string` 型パスワード保持の完全禁止:** .NET の `string` はイミュータブルであり、GC に回収されるまでメモリ上に平文パスワードが残存し続ける。Contracts 境界の DTO や Interface メソッドでパスワードを `string` 型で受け渡すことを厳格に禁止する。
* **安全な型への置換:** パスワードは必ず `byte[]`（または `Memory<byte>`, `SecureString`）として受け渡すこと。UI で入力を受け取った瞬間に配列化し、以降の層に引き回す。
* **メモリの即時破棄:** 鍵導出（Argon2id等）や復号処理が完了した後は、必ず `finally` ブロック内で `Array.Clear(passwordBytes)` または `CryptographicOperations.ZeroMemory(passwordBytes)` を呼び出し、メモリダンプによる平文パスワード流出を物理的に防ぐこと。

### パッケージ追加規約:
* 目的不明またはメンテが停止しているパッケージの追加を禁止。
* 新規パッケージ導入時は、依存関係の推移的セキュリティ脆弱性、Target Framework 適合性、ライセンスを検証の上で追加する。
```

---
