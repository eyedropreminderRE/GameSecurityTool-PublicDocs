# GameSecurityTool Save Data Secure Auto Backup Specification Index

**Document ID:** GST-FEAT-SAVE-INDEX-002  
**Version:** 2.4
**Status:** Approved Feature Specification Index  
**Feature Category:** User Data Protection / Disaster Recovery  
**Target Platform:** Windows 10 / Windows 11 (64-bit)  
**Technology Baseline:** .NET 10 LTS / C# 14 / WPF / SQLite + EF Core 10 / Standard ZIP互換 / Argon2id + AES-256-GCM-CHUNKED  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Purpose & Vision

本ドキュメントは、GameSecurityTool（GST）における「Save Data Secure Auto Backup（セーブデータ安全自動バックアップ）」機能仕様書群（全11ファイル）の公式統合インデックスである。

本機能は、PCゲーム環境における以下のリスクからユーザーのデータ資産を保護し、安全・確実な復旧手段を提供する。

- ランサムウェア・不正プロセスによるセーブデータの暗号化・破壊
- MOD導入・パッチアップデートに伴うデータの不整合・破損
- ユーザーの誤操作によるセーブデータ削除
- PC故障、OS再インストール、ストレージ障害時のデータ消失

本機能はクラウドバックアップサービスではなく、**Local First / Privacy First / User Control First / No Vendor Lock-In** に基づくローカル完結型の復元支援基盤として実装する。

---

# 2. Specification Directory Structure (全11ファイル体系)

本機能に関する仕様は、Clean 5-Layer アーキテクチャのレイヤーおよび責務に応じて以下の全11ファイルで管理する。

```text
Documentation/02_Features/Security/Save_Data_Secure_Auto_Backup/
├── 02-00_Save_Backup_Index.md                        <-- [本書: 統括] 全体索引・DoD・アーキテクチャマップ
├── 02-01_Save_Backup_Core_and_Feature_Control.md     <-- [方針] 設計思想・Universal Feature Control・Policy
├── 02-02_Save_Backup_Domain_and_Snapshot_Model.md    <-- [Domain] Snapshot/Manifestモデル・パス正規化
├── 02-03_Save_Backup_Pipeline_and_Execution.md       <-- [Application] Queue・Channels・抽象ストリーミング
├── 02-04_Save_Backup_Storage_and_Encryption.md       <-- [Infrastructure] ZIPコンテナ・ファイル別Nonce・暗号化
├── 02-05_Save_Backup_Restore_Transaction.md          <-- [Application/Infra] 復元Transaction・Rescue退避・TOCTOU
├── 02-06_Save_Backup_Retention_and_Maintenance.md    <-- [Domain/Infra] コンテナ参照依存・世代管理・退避自動消去
├── 02-07_Save_Backup_Contracts_and_Architecture.md   <-- [Contracts] ISaveBackupService・Port・DTO・型規約
├── 02-08_Save_Backup_Database_and_Audit.md           <-- [Infrastructure] SQLite・TransactionId・直列化
├── 02-09_Save_Backup_UI_and_UX.md                    <-- [Presentation] WPF・MVVM・通知・設定画面
└── 02-10_Save_Backup_Test_and_Verification.md        <-- [Tests] 単体・結合・セキュリティ検証マトリクス
```

---

# 3. Architecture Layer Mapping (Clean 5-Layer)

Save Backup 機能は、GameSecurityTool 全体の Clean 5-Layer / Port & Adapter 境界を厳格に遵守する。

```text
┌────────────────────────────────────────────────────────────────────────┐
│                   Presentation Layer (GST.Presentation)                          │
│  - 02-09: WPF Views, ViewModels (CommunityToolkit.Mvvm)                │
│  - ISaveBackupService (Contracts) を呼び出し                            │
│  - IDispatcherService によるUIスレッド同期                              │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ (calls Application Facade)
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│                  Application Layer (GST.Application)                   │
│  - 02-01: Feature State & Policy Resolution                           │
│  - 02-03: Backup Pipeline, BoundedChannel Queue (Concurrency: 1)       │
│  - 02-05: RestoreTransaction Coordinator (RescueSnapshot 退避)         │
│  - 02-06: Maintenance & Retention Coordinator                          │
└───────────────────┬────────────────────────────────┬───────────────────┘
                    │ (uses Domain Models)           │ (uses Contracts)
                    ▼                                ▼
┌───────────────────────────────┐  ┌────────────────────────────────────┐
│      Domain Layer (GST.Domain)│  │   Contracts Layer (GST.Contracts)  │
│  - 02-02: Snapshot / Manifest │  │  - 02-07: ISaveBackupService Facade│
│    Entities & Fast Check Logic│  │  - 02-07: Storage / Repo Ports     │
│  - 02-06: Retention Rules     │  │  - 02-07: Sealed Record DTOs       │
│  (※ File/DB/OS API 依存禁止)   │  └─────────────────┬──────────────────┘
└───────────────────────────────┘                    │ (implements Interfaces)
                                                     ▼
┌────────────────────────────────────────────────────────────────────────┐
│                Infrastructure Layer (GST.Infrastructure)               │
│  - 02-04: Standard ZIP-compatible Container, Argon2id + AES-256-GCM-CHUNKED│
│  - 02-05: ReparsePoint Path Resolver, Streaming File Restore           │
│  - 02-08: SQLite AppDbContext, SqliteDatabaseWriter 直列化キュー       │
└────────────────────────────────────────────────────────────────────────┘
```

---

# 4. コア保護原則とポータブル復旧 (No Vendor Lock-In)

1. **ポータブル復元保証 (No Lock-In):**
   バックアップは端末固定の DPAPI 単体依存を排除し、**Standard ZIP互換コンテナ + ユーザー指定パスワード（Argon2id + AES-256-GCM-CHUNKED）** で保護する。汎用ZIPツールは外側のコンテナと非暗号化エントリを検査・展開できる。暗号化payloadの復号は汎用ZIPツール単独では保証せず、公開された暗号化形式に対応するGST互換の独立復旧実装により、GSTのDBやDPAPI状態なしで復旧可能とする。
4. **非破壊性 (Non-Destructive):**
   バックアップ処理によって元のセーブデータファイルを破損・変更・排他ロックさせてはならない。
5. **SSD Low Impact:**
   変更のないファイルの再コピーや無駄なフルハッシュ計算を 100% スキップ（Fast Metadata Check: `FileSize` + `LastWriteTimeUtc` + `FileId`）。
6. **直列実行保証:**
   同時バックアップ実行数はシステム全体で常に 1（単一キュー直列化）とし、SSD への I/O 集中やゲームプレイのフレームレート（FPS）低下を防止する。
7. **安全復元トランザクション (`RescueSnapshot`):**
   復元開始直前に現行セーブデータを一時退避し、障害発生時は直前状態へ戻すための安全なロールバック経路を提供する。

---

# 5. Non-Goals (非目標事項の厳守)

本機能の実装において、以下は明示的に対象外（禁止事項）とする。
- 外部クラウドストレージへの自動アップロード・同期機能
- ユーザーの明示的確認（Consent）なしの自動復元（Silent Restore）
- セーブデータ原本ファイルの自動変更・強制書き換え
- 常駐 SYSTEM サービスまたはカーネルドライバによる監視
- セーブデータ本文や個人情報のテレメトリ送信・ログ出力

---

# 6. Definition of Done (完成基準)

本機能が正式に完成したとみなすための必須条件：
- **Architecture:** Clean 5-Layer 依存方向違反ゼロ、Domain 層への I/O 漏れゼロ、Application 層の I/O 抽象化完了（NetArchTest 検証）。
- **Security:** approved change-control decision Versioned Authenticated Envelope、Manifest/AAD tamper rejection、AES-GCM Nonce Reuse 防止構造、非空パスワード強制、復元時の root containment + TOCTOU / Reparse Point 検証、全操作の `OperationId` / `TransactionId` / Audit Chain 追跡完了。
- **Reliability:** 巨大ファイル処理時の OOM ゼロ保証（ストリーミング I/O）、SQLite 書き込み直列化による Lock ゼロ保証、`CancellationToken` 対応。
- **Performance:** 変更なしファイルのハッシュ計算 100% スキップ、SSD 書き込み量の最小化。
- **Quality:** ビルドエラー 0、Nullable 警告 0、定義済みの単体・結合・セキュリティテストをすべて満たす。
```

---
