# GameSecurityTool

# Architecture and Development Governance

## Software Architecture / Implementation Rules / Development Control Definition

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-007 |
| Version | 3.1 (ThreatReasonCodes SSOT Architecture-Test Alignment Edition) |
| Status | Formal Baseline Specification (Highest Architecture Governance Authority) |
| Category | Architecture and Development Governance |
| Authority Level | Development Control Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- ソフトウェアアーキテクチャ統治基準（Clean 5-Layer）
- レイヤー別責務および依存方向の絶対規則
- Universal Mutation Pipeline（変更操作の安全パイプライン）
- ドメインモデル統治（Entity / Value Object / 命名規約）
- エラーハンドリングおよびロギング統治
- AI 支援開発ガバナンスおよび完全実装コード品質規則
- テスト戦略、変更管理、ビルド・リリース判定ゲート
- ドキュメント統治（Single Source of Truth）

を定義する。

---

GST では、単に「動くコード」ではなく、

```text
Predictable  *  Maintainable  *  Secure  *  Reviewable  *  Non-Destructive
(予測可能性、保守性、安全性、検証容易性、非破壊性)
```

なソフトウェアを構築・維持する。

---

# 1. Architecture Philosophy

## 1.1 Core Architecture Principle
GST は以下を採用する。

```text
Clean 5-Layer Architecture
*
Dependency Inversion Principle (DIP)
*
Port and Adapter Pattern (Hexagonal Architecture)
```

目的：
- Security Boundary（セキュリティ境界）の厳格な維持
- 5年以上の長期保守性と技術負債の徹底排除
- CI 自動テスト検証能力（NetArchTest）の確保
- Framework、UI、および OS 具象依存の完全隔離

---

# 2. Layer Architecture Model (Clean 5-Layer 構造)

GST の 5層構造と責務境界：

```text
┌────────────────────────────────────────────────────────┐
│              Presentation Layer (GST.Presentation)               │
│ - WPF Views, ViewModels, Dispatcher スレッド同期       │
│ - ユーザー操作受付 ＆ 状態可視化 (判断・I/O 責任ゼロ) │
└──────────────────────────┬─────────────────────────────┘
                           │ (calls UseCase)
                           ▼
┌────────────────────────────────────────────────────────┐
│           Application Layer (GST.Application)          │
│ - ユースケース調停, トランザクション管理, パイプライン │
│ - ExplanationEngine (判定結果の多言語解釈・DTO変換)    │
│ - Domain ⇄ Contracts 相互マッピングの一元担当         │
└───────────────────┬────────────────┬───────────────────┘
                    │ (uses Domain)  │ (uses Ports / DTOs)
                    ▼                ▼
┌──────────────────────────────┐   ┌────────────────────────────────────┐
│   Domain Layer (GST.Domain)  │   │  Contracts Layer (GST.Contracts)   │
│ - 純粋ビジネスルール・判定   │   │ - 境界 Port (Interface) 定義       │
│ - 確信度・保持期限純粋評価   │   │ - 不変 DTO (sealed record)         │
│ - Pure C# (外部/Contractsゼロ)│   │ - 製品共通 Enum (AuditEventType等) │
└──────────────────────────────┘   └─────────────────┬──────────────────┘
                                                     ▲
                                                     │ (implements Ports)
┌────────────────────────────────────────────────────┴───────────────────┐
│                 Infrastructure Layer (GST.Infrastructure)              │
│ - Contracts Port の具象 Adapter 実装, OS / DB / ファイルシステム接続  │
│ - SQLite Single Writer Queue, 64KB Chunked AEAD, Named Pipe IPC        │
└────────────────────────────────────────────────────────────────────────┘
```

## 2.1 Presentation Layer
* **責務:** UI 表示、ユーザー操作受付、ViewModel 状態管理、Dispatcher スレッド同期。
* **禁止事項:** UI からの直接ファイルアクセス、直接レジストリ操作、DB 直接操作、セキュリティ判断ロジックの実装。UI は判断・実行責任を持たない。

## 2.2 Application Layer
* **責務:** ユースケース制御、ワークフロー調停、トランザクション管理、Explanation 生成、Domain ⇄ Contracts マッピングの一元担当。
* **禁止事項:** Infrastructure 具象クラスの直接利用、`DbContext` の直接操作、Windows API / `System.IO` 物理操作の直接実行。

## 2.3 Domain Layer
* **責務:** GST の中核ビジネスルール、セキュリティ判定（`SecurityEngineEvaluator`）、確信度計算、保持期限評価（`RescueSnapshotRetentionEvaluator`）、差分参照保持評価（`RetentionEvaluator`）。
* **特徴:** 外部ライブラリおよび `Contracts` 依存ゼロの Pure C#。
* **禁止事項:** WPF / UI 型への依存、EF Core / SQLite アノテーション、Windows API / Native 依存、自然言語文言の保持、Contracts DTO / Enum の参照。

## 2.4 Infrastructure Layer
* **責務:** Contracts で定義された Port の具象 Adapter 実装、外部環境（Windows OS, SQLite, ファイルシステム, DPAPI, Firewall COM 等）との物理接続。
* **禁止事項:** Infrastructure は Domain ルールを勝手に決定してはならない。

---

# 3. Dependency Rule (依存方向の絶対規則)

```text
Presentation ➔ Application ➔ Domain / Contracts  Infrastructure
```

### 厳格な依存禁止ルール:
1. **Domain 層の完全孤立 (C-1, C-2 是正):** `GST.Domain` は他の全レイヤー（`Application`, `Infrastructure`, `Presentation`, `Contracts`）および外部 I/O ライブラリに一切依存しない（Pure C#）。
2. **Application 層の OS 独立性:** `GST.Application` は `Infrastructure` の具象クラスや OS 固有 API（WMI, COM, Registry, `System.IO` 物理操作）を直接参照しない（必ず Contracts Port 経由）。
3. **Presentation 層の隔離:** `GST.Presentation` の UI / View / ViewModel は `Infrastructure` を直接参照・呼び出してはならない。ただし `App.xaml.cs` の Composition Root における DI 登録目的の project reference のみ例外として許可する。Win32 API 実装は Infrastructure に隔離する。
4. **Infrastructure 層の型結合禁止 (L-1 是正規約):** `GST.Infrastructure` は Application 層のマッピングをバイパスして `GST.Domain.Models.*` サブ名前空間の内部型を直接参照・操作してはならない（必ず Contracts Port 経由で DTO をやり取りする）。

---

# 4. Security Boundary Implementation Rule

## 4.1 Universal Mutation Pipeline (完全復元 ＆ アトミック変更 C-3, C-4 規約)
すべての環境変更・データ更新処理（セーブ復元、隔離、Firewall 変更、PC 移行再封緘、アンインストール）は、以下のパイプラインを厳格に通過する。

```text
Request (操作要求)
   ↓
Operation Classification (操作の分類・リスク評価)
   ↓
Policy Validation (Universal Rule Scope 8段階決定表照合)
   ↓
Permission Validation (権限・TOCTOU バリア・Reparse Point 解像)
   ↓
User Approval (破壊的変更時の明示的ユーザー承認・プレビュー表示)
   ↓
Audit Creation (OperationId 付番・事前ログ記録)
   ↓
Atomic Execution (一時ファイル .tmp 出力 ➔ ハッシュ検証 ➔ File.Move アトミック置換)
   ↓
Repository Commit (IDbWriteQueue 直列同期完了待機)
   ↓
Audit Hash Chain Append (改ざん検知ログ記録 ＆ 多層アンカー同期)
   ↓
Result Verification (結果の整合性検証)
```

## 4.2 Direct Mutation Prohibition (直接変更の完全禁止)
以下をビルドレベルで禁止する：
```csharp
// 【禁止】UI イベントハンドラからの直接呼出
UI Event ──> File.Delete();

// 【禁止】検知ロジックからの直接レジストリ操作
Detection Logic ──> Registry.SetValue();
```
すべての変更操作は必ず Application Service / Contracts Port 経由で実行しなければならない。

---

# 5. Domain Model Governance (ドメインモデル統治規約)

## 5.1 Entity Rule
主要エンティティ（`GameProfile`, `SaveBackupSnapshot`, `RestoreTransaction`, `QuarantineEntry` 等）は、自身の状態整合性をカプセル化し、不変条件を保護する。

## 5.2 Value Object Usage
不正値の混入を防ぐため、重要データは不変値オブジェクト（Value Object）化する（`GameId`, `ManifestHash`, `DetectionSignal` 等）。

## 5.3 命名規約 (M-3 規約)
Domain 内部で定義する Enum には必ず **`Domain` プレフィックス（例: `DomainRiskLevel`, `DomainAllowType`, `DomainTrustRiskLevel`）** を付与し、機能別サブ名前空間（`Domain.Models.AllowList`, `Domain.Models.TrustEnhancement` 等）で分離・管理する。

---

# 6. Error Handling Policy

## 6.1 Error Philosophy
エラーは単なる例外ではなく、**システムの整合性とセキュリティ状態を示す重要な安全情報** として扱う。

## 6.2 Error Flow & Result パターン
```text
Exception ➔ Classification ➔ Audit Record ➔ User Notification ➔ Recovery Option
```
業務ユースケースにおいて予期される失敗（ファイルロック、ハッシュ不一致、容量不足等）は例外を送出せず、`Result<T>` または結果 DTO で返却する。`catch (Exception) { }` による例外の握りつぶしを厳禁とする。

## 6.3 Sensitive Information Protection
エラーメッセージ、スタックトレース、UI 通知に以下を含めることを厳格に禁止する：
- パスワード, 暗号鍵, トークン, Gemini API キー
- 生 URL Query String, クリップボード本文
- ユーザー名を含むプロファイル絶対パス（必ず `ILogSanitizer` で伏字化）

---

# 7. Logging Governance (ログ統治 - 完全復元)

## 7.1 Logging Purpose
Logging の目的：
```text
Diagnosis (障害解析)  *  Evidence (客観的証拠保全)  *  Recovery (安全な復旧支援)
```
ユーザーの通常行動の監視・収集を目的としたロギングを厳禁とする。

## 7.2 Log Separation (ログの物理分離)
以下の 3 系統を明確に物理分離して管理する：
1. **Application Log:** 構造化デバッグ・運用ログ（`PrivacyLogSanitizer` で自動伏字化）
2. **Audit Log:** 改ざん検知 SHA-256 Hash Chain 証跡（SQLite ＋ 多層アンカー）
3. **Security Evidence:** フォレンジック用客観的証拠データ（`EvidenceBundle.zip`）

---

# 8. AI-Assisted Development Governance (AI 開発統治)

## 8.1 AI Usage Principle
AI コーディングエージェントは開発支援ツールとして利用し、最上位ベースライン仕様書（Tier 0）を唯一の正本（SSOT）として行動する。

## 8.2 AI Generated Code Requirements
AI が生成したコードは、以下の観点で厳格に検証する：
- Clean 5-Layer 結合規則の遵守（Domain 純粋性、Infrastructure 型直結禁止）
- 最小権限原則（Standard User 起動、Named Pipe 特権分離）
- メモリ安全性（パスワード `string` 禁止、`byte[]` 受領と `finally` での `ZeroMemory` 消去、Caller-Owns 契約）
- 変更操作の原子性（一時ファイル経由のアトミック置換 `.tmp` ➔ `File.Move`）

## 8.3 Prohibited AI Output
採用禁止事項：
- `// TODO:` のみの未完成実装、`/* 後で実装 */`、`throw new NotImplementedException();`
- 擬似コード、省略記号（`// ... 省略 ...`）
- 既存仕様書にない架空のクラス・メソッド・ライブラリの捏造

---

# 9. Code Quality Rules (コード品質規則 - 完全復元)

## 9.1 Complete Implementation Principle (完全実装の原則)
正式コードにおいては、すべての C# ファイルが必要な `using`、名前空間、型定義、エラーハンドリングを含み、直ちにコンパイル・実行可能な完成状態でなければならない。

## 9.2 Naming Convention (命名規則 - 完全復元)
命名は目的とレイヤー責務が明確であること。

* **禁止例（曖昧な命名の乱用）:** `Manager`, `Helper`, `Utils`
* **推奨例（目的明示命名）:** `SaveBackupExecutionEngine`, `MigrationCoordinator`, `WmiProcessLifecycleWatcher`, `SecurityEngineEvaluator`

---

# 10. Testing Strategy (テスト戦略 - 完全復元)

## 10.1 Test Pyramid (テストピラミッド)
```text
      [ System & E2E Tests ]         (GUI, 統合ワークフロー)
    [ Security Attack Tests ]        (TOCTOU, PID Reuse, OOM, Reseal, Rollback)
   [ Module Integration Tests ]      (SQLite Queue, Named Pipe, Storage GC)
  [ Domain Pure Unit Tests ]         (Risk Logic, Manifest Hash, Retention)
[ Architecture Tests (CI Gate) ]     (NetArchTest ルール 1〜6 機械的自動検査)
```

## 10.2 Unit Test
対象: Domain ルール, 判定ロジック, 確信度計算, 保持期限評価 (Pure C#)。

## 10.3 Integration Test
対象: SQLite Single Writer Queue, Standard ZIP コンテナ, DPAPI 暗号化, Named Pipe IPC。

## 10.4 Security Test
対象: 特権境界, パストラバーサル, 秘密情報メモリ消去, 改ざん検知多層アンカー, WMI 標準エスケープ。

## 10.5 Regression Test
目的: 既存機能・安全機構の破壊防止。リリース判定の必須要件。

---

# 11. Change Management (変更管理 - 完全復元)

## 11.1 Feature Change Process
機能変更・追加の標準シーケンス：
```text
Proposal (提案) ➔ Impact Analysis (影響度分析) ➔ Security Review (セキュリティ審査) ➔ Implementation ➔ Automated Test ➔ Release
```

## 11.2 Baseline Conflict
新機能やコード修正が最上位 Baseline（Tier 0）と矛盾する場合、**「新機能側を却下・修正（Baseline 絶対優先）」** とする。機能追加の都合でベースライン原則を無断で緩和・破壊してはならない。

---

# 12. Build Governance (ビルド統治 - 完全復元)

## 12.1 Build Requirement
正式リリースビルドの必須条件：
- Visual Studio / CI において Compile Error = 0
- Nullable Warning = 0 (`<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`)
- `NetArchTest.Rules` 1〜6 および `TC-ARCH-SSOT-01` が 100% パス

## 12.2 Dependency Control
外部 NuGet パッケージの追加は、ライセンス適合性（MIT / Apache 2.0 限定）、既知脆弱性（CVE）、長期保守性を審査の上、承認インベントリ（`34_License`）に登録されたもののみを許可する。

---

# 13. Release Gate (リリース判定ゲート - 完全復元)

正式リリース承認の必須ステップ：

```text
Build Verification (警告ゼロ)
       ↓
Automated NetArchTest & Unit/Integration/Security Tests (100% Pass)
       ↓
Fault Injection Resilience Review (アトミック復元・再封緘検証)
       ↓
Regression Review (既存機能無破壊検証)
       ↓
Documentation Synchronization Check (全127仕様書 100% 合致)
       ↓
Release Approval (三者承認: Technical, Security, Quality)
```

---

# 14. Documentation Governance (ドキュメント統治 - 完全復元)

## 14.1 Required Documentation
仕様変更時は、コード・テスト・ドキュメントを必ず同一コミットで同時同期する。

## 14.2 Documentation Priority (権威優先順位)
```text
Tier 0: Baseline Documents (最上位正本)
       ↓
Tier 1: Master Inventory
       ↓
Tier 2: Architecture Specifications
       ↓
Tier 3: Module & Feature Specifications
       ↓
Implementation Code
```
コードが仕様を勝手に決定してはならない。実装と仕様に乖離が生じた場合、上位仕様書に従ってコードを修正する。

---

# 15. Relationship With Other Models (他モデルとの関係性 - 完全復元)

## 15.1 Security Boundary & Implementation
セキュリティ境界およびセキュアコーディング指針：
```text
02_Security_Boundary_and_Protection_Model.md
14_Security_Implementation_Guideline.md
```

## 15.2 Service & Interface Model
Contracts Port インターフェース定義：
```text
13_API_and_Service_Interface_Model.md
```

## 15.3 Source Structure & Architecture
ソースコード構造および Clean 5-Layer 結合規則：
```text
28_Source_Code_Structure_and_Module_Architecture_Model.md
01_Architecture/01-01_Clean_Hexagonal_Architecture.md
```

## 15.4 Testing & Regression Prevention
テスト戦略および回帰防止チェックリスト：
```text
15_Test_Strategy_and_Validation_Model.md
08_Regression_Prevention_and_Final_Baseline_Checklist.md
```

---

# 16. Final Development Statement

GST の開発では、機能追加の速度よりも、安全性・保守性・説明可能性を絶対優先する。

---

最終原則：

```text
Design Before Code
Security Before Convenience
Enforce Clean 5-Layer Boundaries
Review Before Release
Maintain Before Expand
```

---

End of Document
```
