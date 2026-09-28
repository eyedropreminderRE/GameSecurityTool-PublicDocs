# GameSecurityTool

# Data Model and Database Schema

## Entity Definition / Relationship Model / Storage Governance

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-012 |
| Version | 2.2 (Referential Integrity Section Restored & Restrict Constraint Applied) |
| Status | Formal Baseline Specification |
| Category | Data Architecture |
| Authority Level | Data Design Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書はGameSecurityTool（以下GST）における、

- 内部データモデル（Infrastructure 永続化 Record）
- Entity設計とリレーション
- データ関連性
- 永続化方針（`IDbWriteQueue` による直列化）
- 暗号化対象と DPAPI / Argon2id 適用
- Database Migration 規約

を定義する。

---

GSTのデータ設計原則：

```
Data Must Be Understandable
Data Must Be Recoverable
Data Must Be Protected
Domain Entity != Persistence Record
```

---

# 1. Data Architecture Philosophy

## 1.1 Storage Principle

GST内部データは、以下を満たす。

```
Consistency (単一キュー直列化による競合排除)
Integrity (Hash Chain による改ざん検知)
Traceability (OperationId / TransactionId による追跡)
Migration Compatibility (Argon2id による PC 移行再封緘)
```

---

# 1.2 Ownership & Layer Separation

データは物理的に分離する。

* **Infrastructure Layer:** EF Core 10 (`AppDbContext`) の `DbSet` として管理される `*Record` / `*Entry` クラス。
* **Domain Layer:** 外部依存を持たない純粋な POCO（永続化属性を持たない）。
* **UI/Contracts:** 境界を越えるための不変 `sealed record DTO`。

---

# 2. Identifier & Tracing Design

## 2.1 ID Principle
各エンティティは主キーとして `Guid`（または連番）を持つ。
同時に、複数テーブルを跨いだ業務トラッキングのため以下の ID を必須とする。

* **`OperationId`**: ユーザー操作の全体を追跡する ID（例: `GST-OP-20260827-001`）。
* **`TransactionId`**: 原子性・二段階コミットを追跡する ID（例: `QRT-20260827-A82F`, `RST-20260827-55F1`）。

---

# 3. Game Management Entity

## 3.1 GameProfileRecord
ゲーム管理の中心エンティティ。

```text
GameProfileRecord
├ Id (Guid)
├ DisplayName (string)
├ MainExecutable (string)
├ Path (string)
├ Publisher (string?)
├ Platform (string?)
├ InstallationState (int)
├ CreatedAt (DateTimeOffset)
└ UpdatedAt (DateTimeOffset)
```

---

# 4. Save Backup Entity

## 4.1 SaveBackupSnapshotRecord
バックアップの不変メタデータおよびタイムトラベル管理。

```text
SaveBackupSnapshotRecord
├ Id / BackupId (Guid)
├ GameProfileId (Guid)
├ OperationId / TransactionId (string)
├ StoragePath (string)          // .zip コンテナへの相対/絶対パス
├ FileCount (int) / TotalSizeBytes (long)
├ ManifestHash (string)         // 決定論的 SHA256 マニフェストハッシュ
├ Status (int)                  // Creating, Completed, Failed, Corrupted
├ RetentionStatus (int)
├ IsPinned (bool)               // 📌 ピン留め (自動パージからの永久保護)
├ TagsJson (string)             // 🏷️ プリセット/カスタムタグ配列 JSON
├ UserNote (string?)            // ユーザー任意メモ
└ CreatedAt (DateTimeOffset)
```

## 4.2 SaveBackupFileRecord (CRIT-05 是正規約: スキーマレベルでの差分参照保護)
コンテナ内の個別ファイルマニフェスト（差分参照追跡用）。

```text
SaveBackupFileRecord
├ Id (long)
├ BackupId (Guid)
├ RelativePath (string)
├ SizeBytes (long)
├ LastWriteTimeUtc (DateTimeOffset)
├ SHA256 (string)
└ SourceContainerBackupId (Guid) [FK -> SaveBackupSnapshotRecord.Id] ON DELETE RESTRICT
```
**【必須規約】** `SourceContainerBackupId` は、単なるインデックスではなく、親の `SaveBackupSnapshotRecord.Id` に対する**外部キー制約（Foreign Key）** として構成し、`DeleteBehavior.Restrict` を EF Core の Fluent API で明示しなければならない。これにより、他スナップショットからデータ実体として参照されている物理コンテナのメタデータが運用ミスで誤削除され、後続復元が不可能になる事故を RDBMS レベルで物理的に拒否する。

## 4.3 RestoreTransactionRecord
復元トランザクションの追跡と起動時クラッシュ修復（RescueSnapshot 退避）。

```text
RestoreTransactionRecord
├ Id (Guid)
├ TransactionId (string) [UNIQUE]
├ BackupId / GameProfileId (Guid)
├ TargetRootPath (string)
├ Status (int)                  // PreRestoreBackingUp, Restoring, PartialFailed, RolledBack
├ FailedStep (string?)          // 例: "HashVerify", "AttributeRestore"
├ ErrorMessage (string?)
└ StartedAt / CompletedAt (DateTimeOffset)
```

---

# 5. Security & Isolation Entity

## 5.1 QuarantineEntryRecord
64KB チャンク分割 AEAD（`AesGcm`）で隔離されたファイルの追跡。

```text
QuarantineEntryRecord
├ Id (Guid)
├ TransactionId (string)
├ OriginalPath (string)
├ EncryptedContainerPath (string) // .qrt コンテナへのパス
├ SHA256 (string)
├ KeyProtectionMethod (string)    // "DPAPI_AES256_GCM_CHUNKED"
├ OriginalAcl (string)            // SDDL 形式
├ OriginalAttributes (uint)
├ OriginalCreationTime (DateTimeOffset)
├ RestoreStatus (int)             // 0=Quarantined, 1=Restored, 2=PartialFailed
└ FailedStep (string?)
```

## 5.2 SystemChangeRecord
OS レベルの変更（Firewall, WER 等）を記録する WAL ジャーナル。

```text
SystemChangeRecord
├ Id (Guid)
├ OperationId (string)
├ RollbackOrder (int)
├ ChangeType (string)
├ Target (string)
├ BeforeStateJson (string?)
├ AfterStateJson (string?)
└ Status (int)                    // 0=Applied, 1=RolledBack, 2=Failed
```

---

# 6. Policy & Rules Entity

## 6.1 FirewallRuleRecord
Zero Trust Firewall ルール。

```text
FirewallRuleRecord
├ RuleTag (string) [PK]
├ GameProfileId (Guid)
├ ExecutablePath (string)
├ FileHash (string)
├ Direction (int)                 // Outbound / Inbound
├ Protocol / Port / RemoteAddress (string)
└ CreatedAt (DateTimeOffset)
```

## 6.2 AllowListEntry
セキュリティエンジンの TOCTOU 対応例外許可リスト。

```text
AllowListEntry
├ Id (Guid)
├ GameProfileId (Guid?)           // null = Global
├ MatchType (int)                 // 0=Hash, 1=Signature, 2=Path, 3=Name
├ TargetSHA256 / TargetPath / TargetFileName / PublisherName (string?)
├ Expiration (int)
├ ExpiresAt (DateTimeOffset?)     // リアルタイム有効期限評価
└ UserMemo (string?)
```

## 6.3 WebHostRuleRecord
Web リンク保護のドメインポリシー。

```text
WebHostRuleRecord
├ Id (Guid)
├ GameProfileId (Guid?)
├ Host (string)                   // 正規化済み (IDN/小文字化)
├ MatchMode (int)                 // 0=ExactHost, 1=IncludeSubdomains
├ Policy (int)                    // 0=Allow, 1=Confirm, 2=Block
├ IsEnabled (bool)
└ ExpiresAt (DateTimeOffset?)
```

---

# 7. Audit Entity

## 7.1 AuditEventRecord (Hash Chain)
改ざん検知の要となる監査証跡チェーン。

```text
AuditEventRecord
├ SequenceNumber (long) [PK]
├ OperationId (string)
├ EventType (int)                 // 明示的数値固定の Enum
├ Actor (string)
├ Target (string)                 // ILogSanitizer による伏字化済み
├ Result (string)
├ PreviousHash (string)           // SHA256
├ CurrentHash (string)            // SHA256 (特権アンカー二重保存連携)
└ TimestampUtc (DateTimeOffset)
```

---

# 8. Database Migration & Concurrency Rules

## 8.1 SQLite Single Writer Queue
- すべての Insert/Update/Delete 処理は `IDbWriteQueue`（`SqliteDatabaseWriter`）を経由して直列化する。
- キューの中でアクションごとに短命な `DbContext` を生成・破棄し、メモリリークと例外によるコンテキスト汚染を完全に排除する。

## 8.2 Enum Persistence Governance (Append-Only Rule)
- 永続化されるすべての列挙型（`Status`, `EventType`, `RiskLevel` 等）は、コード内で明示的な整数値（例: `Creating = 0`）を固定しなければならない。
- 既存の数値を変更したり、中間に新しい値を挿入してはならない。新設する場合は必ず末尾へ追加する。

## 8.3 Referential Integrity (完全復元)
- スナップショットの削除時は `DeleteBehavior.Restrict` を利用し、差分参照されているファイルコンテナの道連れ削除を防止する。
- 物理ファイルの回収は、GC 処理によって安全に行う。

---

# 9. Encryption Classification

## Level 1: 通常情報
`GameProfileRecord`, `AllowListEntry`, `WebHostRuleRecord`

## Level 2: 保護推奨情報
`SystemChangeRecord` の BeforeStateJson

## Level 3: 暗号化・再封緘必須 (PC 移行対象)
- `QuarantineEntryRecord` の暗号化コンテナ（DPAPI AES 鍵）
- `SaveBackupSnapshotRecord`（Standard ZIP + Argon2id）
- `AuditEventRecord` の最新ハッシュアンカー（DPAPI/特権領域保存）

---

# 10. Relationship With Other Models

- **Architecture:** `01-04_Persistence_and_Database_Architecture.md`
- **Backup Storage:** `02-04_Save_Backup_Storage_and_Encryption.md`
- **Database Rules:** `29_Database_Migration_and_Schema_Evolution_Model.md`

---

# 11. Final Data Statement

GSTのデータモデルは、機能実装のためだけではなく、**安全な管理・復旧・説明可能性を維持するため**に存在する。

最終原則：

```
Separate Domain From Persistence
Serialize Writes
Never Mutate Existing Meanings
Every Secret Has Protection
```

---

End of Document
```

---
