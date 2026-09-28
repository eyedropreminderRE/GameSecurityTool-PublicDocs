# 01-00: Architecture Overview and Principles

**Document ID:** GST-ARCH-BASELINE-002-PART0  
**Version:** 2.4 (UI Working-Draft Set Expansion / Architecture Inventory Sync)  
**Parent Document:** 00_Formal_Baseline_Overview.md  
**Category:** Architecture Baseline  
**Status:** Approved Baseline Candidate  

---

# 0. Document Purpose

本書は、GameSecurityTool（GST）における以下の技術標準およびアーキテクチャ境界を定義する統括仕様書である。

* 技術スタック（C# 14 / .NET 10 LTS / WPF / SQLite + EF Core 10）
* Solution 構造および Clean 5-Layer レイヤー責務
* Dependency Rule（依存方向の絶対規則）
* Port & Adapter 境界および Contracts 層の役割
* Database 永続化および並行性制御（Single Writer Queue）
* Windows Platform Integration 境界（Win32, WMI, COM, DPAPI, Named Pipe IPC）
* Security 基盤および Trust Enhancement（説明エンジン）の配置

本書は、「何を作るか（What）」ではなく、**「5年以上破綻せず堅牢にどう構築するか（How to Build）」** を定義する。

---

# 1. Source of Truth (仕様の権威階層と優先順位)

GameSecurityTool では、仕様と実装基盤を以下の **単一権威チェーン（Tier 0 〜 Tier 3）** で厳格に管理する。下位仕様が上位仕様と矛盾する場合、上位仕様を絶対優先とする。

```text
[Tier 0: 最上位ベースライン] 00_Baseline/ (00〜50番)
        │ (製品哲学, 不可侵原則, セキュリティ境界, Zero Trust)
        ▼
[Tier 1: 製品ガバナンス・機能インベントリ]
        │ (全機能インベントリ, Universal Rule Scope, Feature Control)
        ▼
[Tier 2: 技術アーキテクチャ基準] 01_Architecture/ の技術アーキテクチャ基準 (01-00 〜 01-07)
        │
        └─ UI design working drafts (01-08〜01-20) はこの層を補完する従属設計文書であり、Tier 0〜3 の上位仕様を上書きしない。
        │ (Clean 5-Layer, 技術標準, 暗号・特権IPC規約, SQLite Queue)
        ▼
[Tier 3: モジュール & 機能詳細仕様] modules/ (03-01〜06) & 02_Features/
        │ (SYS, SCAN, FW, QUAR, ALLOW, CFG, SaveBackup, WebLink, LNK, Clip)
        ▼
[実ソースコード] Solution / C# Source Files
        ▼
[検証結果] NetArchTest / Unit / Integration / Security Test
```

### 1.1 実コード優先原則 (Source Code Grounding)
ドキュメントを参照して実装する際は、必ず現在の Solution 構成を確認する。過去資料や設計書上の予定のみを根拠に、存在しない Project / Class / Interface / Entity を実在するものとして扱ってはならない。

---

# 2. Architecture Design Philosophy

## 2.1 6大コア設計原則
GST は以下の 6 大原則を全レイヤーで維持する。

```text
Local First            : 外部クラウド通信ゼロ・完全ローカル完結処理
Privacy First          : テレメトリ非送信・個人情報自動マスキング・Strict Query Strip
Security First         : Zero Trust・Default Outbound Block・特権物理分離
User Control First     : 独断での自動削除・自動復元の禁止・明示的同意（Consent）
Explainable Security   : 専門用語を隠蔽した理由（Reason List）と証拠の提示
Evidence Based Decision: ハッシュ・署名・スナップショット等の客観的証拠に基づく判定
```

## 2.2 Security Boundary Principle
検知・判断・説明・同意・実行の各フェーズをアーキテクチャレベルで分離する。

```text
Detection (検知) ──> Decision (判定) ──> Explanation (説明) ──> User Consent (同意) ──> Execution (実行)
```
* **禁止:** `Detection ──> 即座に自動削除 / 強制隔離` のような短絡処理。

---

# 3. Clean 5-Layer Architecture & Solution Structure

GST は、レイヤー間の結合度を最小化し長期保守性を確保するため **Clean 5-Layer + Port & Adapter (Hexagonal)** アーキテクチャを採用する。

```text
                                [ Windows 11 / OS ]
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          Presentation Layer (GST.Presentation)                            │
│  - WPF (Fluent UI / Mica & Acrylic), ViewModels (CommunityToolkit.Mvvm)         │
│  - Dispatcher スレッド同期 (IDispatcherService), Timeline View 仮想化描画        │
└───────────────────────┬─────────────────────────────────┬───────────────────────┘
                        │ (calls UseCases)                │ (subscribes DTOs)
                        ▼                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                       Application Layer (GST.Application)                       │
│  - Use Cases, Orchestration, Pipeline (Scan, Backup, Quarantine, WebLink)       │
│  - Explanation Engine (判定結果の多言語解釈・DTO変換)                           │
│  - Transaction Coordination (RestoreTransaction, Safe Mode Controller)          │
└───────────────────────┬─────────────────────────────────┬───────────────────────┘
                        │ (uses Domain Rules)             │ (uses Ports / DTOs)
                        ▼                                 ▼
┌─────────────────────────────────────────┐     ┌─────────────────────────────────┐
│        Domain Layer (GST.Domain)        │     │  Contracts Layer (GST.Contracts)│
│  - Pure C# Business Rules               │     │  - Boundary Ports (Interfaces)  │
│  - Risk / Confidence Evaluation Logic   │     │  - Sealed Record DTOs           │
│  - RuleScope / FeatureScope Resolvers   │     │  - Common Enums (AuditEventType)│
│  - Snapshot & Manifest Models           │     └─────────────────┬───────────────┘
│  (※ Win32/IO/DB/WPF 依存の完全排除)      │                       │ (implements)
└─────────────────────────────────────────┘                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     Infrastructure Layer (GST.Infrastructure)                   │
│  - File System, Reparse Point Barrier (`GetFinalPathNameByHandle`)              │
│  - SQLite + EF Core 10 (PRAGMA WAL + SqliteDatabaseWriter 直列化キュー)          │
│  - Out-of-Process UAC Worker (DACL保護 Named Pipe IPC + Windows Firewall COM)   │
│  - Win32 Native APIs (`[LibraryImport]` + SafeHandle), WMI Process Tracker      │
│  - Quarantine Storage (チャンク分割 AEAD + DPAPI 鍵保護 + ZeroMemory)           │
│  - Save Backup Storage (Standard ZIP + Argon2id / AES-256 パスワード保護)       │
│  - Browser Adapter, PrivacyLogSanitizer (`[GeneratedRegex]`)                    │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

# 4. 主要基盤アーキテクチャ標準 (v2.2)

### 1. 永続化と並行性制御 (`SqliteDatabaseWriter`)
* SQLite の単一ライター制約による `SQLITE_BUSY`（ロック競合）を防止するため、全モジュールの書き込み操作は `IDbWriteQueue` / `SqliteDatabaseWriter`（単一バックグラウンドワーカー）経由で直列化する。
* 読み取りは `IDbContextFactory<AppDbContext>` から短命 `DbContext` を生成し、`AsNoTracking()` で並行実行する。

### 2. 特権分離と安全な IPC
* Main GUI プロセスは Standard User で動作し、管理者権限が必要な処理（Firewall変更、保護領域アクセス）は、DACL 保護された **Named Pipe** 経由で独立した昇格ワーカー（`ElevatedWorker.exe`）へ委譲する。

### 3. 暗号化およびストレージモデルの分離
* **Quarantine (端末内隔離):** 端末外流出を防ぐため `DPAPI (CurrentUser)` + 64KB チャンク分割ストリーミング AEAD（`AesGcm`）を採用。二段階コミット順序（①コンテナ出力 ➔ ②DBコミット ➔ ③元ファイル削除）を厳守。
* **Save Backup (ユーザー資産):** ポータブルな災害復旧を保証するため、**Standard ZIP 互換形式 + ユーザー指定パスワード（Argon2id + AES-256）** を採用。GST なしでも 7-Zip 等で自力復元可能とする（No Vendor Lock-in）。

### 4. WMI 緊急検知と PID Reuse 防御
* Managed Launch Path を迂回したブラウザ起動に対し、WMI 受動イベントとプロセスの生成時刻（`CreationTime`）を `eventTimeUtc ± 3秒` の対称ウィンドウで検証し、PID 再利用による誤介入を防止する。

---

# 5. 仕様書体系と担当ファイル一覧

技術アーキテクチャ基準の正規パートは `01-00`〜`01-07` の8パートで構成される。これとは別に、同ディレクトリにはUI設計を段階的に整理する従属Working Draftを配置する。

* **[01-00_Overview_and_Principles.md](01-00_Overview_and_Principles.md)** — 【本書】アーキテクチャ全体概要・Source of Truth
* **[01-01_Clean_Hexagonal_Architecture.md](01-01_Clean_Hexagonal_Architecture.md)** — 5層レイヤー依存方向ルール・Port & Adapter 境界
* **[01-02_Technology_Stack_and_Runtimes.md](01-02_Technology_Stack_and_Runtimes.md)** — C# 14 コーディング規約・Nullable 警告ゼロ
* **[01-03_Security_Architecture_and_Threat_Model.md](01-03_Security_Architecture_and_Threat_Model.md)** — 特権境界・Named Pipe IPC・暗号化基準
* **[01-04_Persistence_and_Database_Architecture.md](01-04_Persistence_and_Database_Architecture.md)** — EF Core 10 / SQLite Single Writer Queue
* **[01-05_OS_Integration_and_Hardware_Boundary.md](01-05_OS_Integration_and_Hardware_Boundary.md)** — Win32 / WMI / COM / DPAPI
* **[01-06_Presentation_and_UI_Architecture.md](01-06_Presentation_and_UI_Architecture.md)** — WPF MVVM / Dispatcher 同期 / Timeline View
* **[01-07_Cross_Cutting_Concerns_and_Lifecycle.md](01-07_Cross_Cutting_Concerns_and_Lifecycle.md)** — ログサニタイズ・品質ゲート・ライフサイクル

### Document Change Record

| Item | Value |
|---|---|
| ChangedAt | 2026-09-29 |
| ChangeSummary | Clarified the boundary between the `01-00`〜`01-07` technical architecture baseline and the subordinate UI design Working Drafts `01-08` / `01-09`, and synchronized the architecture document inventory. |
| Reason | Prevent the expanding UI design material from being mistaken for additional technical architecture baseline authority. |
| AffectedProjects | Documentation / AI document navigation only; no runtime project changes. |
| MigrationRequired | No runtime migration. AI/document readers must use the updated directory classification and linked UI design documents. |

---

### Document Change Record — 2026-09-29 UI Working-Draft Set Expansion

| Item | Value |
|---|---|
| ChangedAt | 2026-09-29 |
| ChangeSummary | Split the expanding UI design material from the monolithic 01-08 working draft into a responsibility-based set 01-08〜01-20, while preserving the technical architecture baseline at 01-00〜01-07. |
| Reason | Keep global IA, visual system, screen-specific design, common interaction patterns, journeys, cross-screen mapping, and scenario validation independently retrievable without creating additional technical architecture authority. |
| AffectedProjects | Documentation / AI document navigation only; no runtime project changes. |
| MigrationRequired | No runtime migration. AI/document readers must use the UI Design Working-Draft set according to scope; existing runtime source and formal feature specifications remain unchanged. |

## UI Design Working Drafts
* **[01-08_UI_Information_Architecture_and_Interaction_Concept.md](01-08_UI_Information_Architecture_and_Interaction_Concept.md)** — Global IA / Navigation / Interaction / application-wide structure.
* **[01-09_UI_Visual_Design_System.md](01-09_UI_Visual_Design_System.md)** — Visual Foundation / Design Tokens / Common Component States / Visual Safety.
* **[01-10_UI_Home_and_Dashboard_Design.md](01-10_UI_Home_and_Dashboard_Design.md)** — Home / Dashboard design.
* **[01-11_UI_Games_Design.md](01-11_UI_Games_Design.md)** — Games screen design.
* **[01-12_UI_Save_Data_Design.md](01-12_UI_Save_Data_Design.md)** — Save Data screen design; formal Save Backup UI spec remains authoritative.
* **[01-13_UI_Protection_Design.md](01-13_UI_Protection_Design.md)** — Protection presentation and cross-feature status presentation.
* **[01-14_UI_Recovery_Design.md](01-14_UI_Recovery_Design.md)** — Recovery screen and recovery-flow presentation.
* **[01-15_UI_Settings_Design.md](01-15_UI_Settings_Design.md)** — Settings screen design.
* **[01-16_UI_Common_Interaction_and_Display_Design.md](01-16_UI_Common_Interaction_and_Display_Design.md)** — Common interaction and display.
* **[01-17_UI_Reference_Patterns_and_Common_Component_Library.md](01-17_UI_Reference_Patterns_and_Common_Component_Library.md)** — Reference patterns and reusable component library.
* **[01-18_UI_User_Journeys_and_Review_Model.md](01-18_UI_User_Journeys_and_Review_Model.md)** — User journeys and Review / Confirm model.
* **[01-19_UI_Cross-Screen_Flow_and_Transition_Model.md](01-19_UI_Cross-Screen_Flow_and_Transition_Model.md)** — Cross-screen flow, transitions, and responsibility mapping.
* **[01-20_UI_End-to-End_Scenario_Validation.md](01-20_UI_End-to-End_Scenario_Validation.md)** — End-to-end UX scenario validation.

* **[Feature_Integration_Update_v1.0.md](Feature_Integration_Update_v1.0.md)** — 追加3機能（LNK検知、セーブデータ自動バックアップ、Clipboardサニタイザー）の統合更新仕様。

```

---

