# GameSecurityTool

# Database Migration and Schema Evolution Model

## Database Versioning / Schema Change / Enum Persistence Governance Specification

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-029 |
| Version | 2.0 (Enum Persistence Governance & Single Writer Hardened) |
| Status | Formal Baseline Specification |
| Category | Database Architecture |
| Authority Level | Data Evolution Governance |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- データベース構造管理
- Schema バージョニング
- Database Migration ワークフロー
- Enum 永続化ガバナンス（追記専用原則）
- 互換性維持とロールバック

を定義する。

---

目的：

```
Preserve User Data
Enable Safe Schema Evolution
Prevent Silent Semantic Data Corruption
```

---

# 1. Database Philosophy

## 1.1 Core Principle

GST では、機能追加やスキーマ変更よりも、**既存ユーザーデータの完全保護と過去ログの意味的正確性** を絶対優先する。

---

基本原則：

```
Never Destroy Existing Data
Always Validate Migration
Enforce Enum Append-Only Rule
Always Provide Recovery Path
```

---

# 2. Database Responsibility

Database は以下を管理する。

```
Game Profile & Folders
Save Backup Snapshots & Manifests
Quarantine Entries
Firewall Managed Rules
Allow List & Trusted Locations
Audit Records (Tamper-Evident Hash Chain)
System Change Records (WAL Journal)
Security Profile Presets & Instances
```

---

# 3. Schema Version Management

## 3.1 Version Requirement
SQLite DB 内に `SchemaVersion` テーブルを保持し、アプリケーション起動時に必ず照合する。

```text
Database Version 1 ──(Migration Script)──> Database Version 2
```

## 3.2 Version Check & Health Check
起動時、通常画面表示前にヘルスチェック（`PRAGMA quick_check;`）およびバージョン突合を実施する。異常検知時は通常起動を中断し、`GameSecurityTool.Recovery.exe` の Recovery Host 経路へ遷移する。

---

# 4. Schema Change Classification (変更分類)

## 4.1 Additive Change (安全な変更)
- 新規テーブルの追加
- 既存テーブルへの Null 許容カラムの追加
- 新規インデックスの追加
- **Enum 定義末尾への新規メンバー追加**

## 4.2 Transform Change (注意が必要な変更)
- カラム名変更・型変換
- テーブル分割・リレーション変更
- ※事前データバックアップとトランザクション移行が必須。

## 4.3 Destructive Change (原則禁止)
- 既存データカラムの無断削除
- 必須制約（NOT NULL）の無断追加
- **既存 Enum メンバーの途中挿入・数値変更・再利用**

---

# 5. Enum 永続化ガバナンス (Enum Persistence Governance - M-1 解決)

ドメイン Enum を `HasConversion<int>()` により SQLite へ整数値として保存する場合、以下の規約を強制する。

## 5.1 明示的数値固定 (Explicit Integer Assignment)
すべての永続化対象 Enum（`AuditEventType`, `RiskLevel`, `SnapshotStatus`, `RetentionStatus`, `AllowType` 等）は、数値を明示的に固定して宣言しなければならない。

```csharp
// 正しい宣言例 (明示的数値固定)
public enum SnapshotStatus
{
    Creating = 0,
    Completed = 1,
    VerificationFailed = 2,
    Corrupted = 3,
    Deleted = 4
}
```

## 5.2 追記専用原則 (Append-Only Rule)
- 既存 Enum の**途中や先頭に新しい値を挿入することを厳禁**とする（後続の整数値がシフトし、過去の DB レコードの意味が静かに反転するため）。
- 新しい値を追加する場合は、必ず**リストの末尾（最大値 + 1）**に定義しなければならない。

## 5.3 廃止値の予約維持 (Obsolete Reservation)
機能改修により使用されなくなった Enum 値であっても、定義から削除してはならない。`[Obsolete]` 属性を付与して数値を永久予約（欠番維持）とし、過去ログのデシリアライズ互換性を保証する。

```csharp
public enum AllowType
{
    Hash = 0,
    Signature = 1,
    Path = 2,
    Name = 3,
    [Obsolete("Use Signature instead")]
    CertificateThumbprint = 4, // 廃止値も数値を維持
    PublisherWildcard = 5      // 新規値は末尾に追加
}
```

---

# 6. Migration Strategy & Concurrency Control

## 6.1 基本移行フロー
```text
1. SQLite DB ファイルの事前自動バックアップ (gamesecurity.db.bak)
2. PRAGMA 設定の適用 (WAL モード, busy_timeout = 5000)
3. 単一トランザクション内でのマイグレーション実行
4. スキーマ整合性検証 & SchemaVersion 更新
5. コミット確定 (失敗時は即時ロールバック & バックアップ復元)
```

## 6.2 書き込み直列化 (`SqliteDatabaseWriter`)
マイグレーション後の日常的なデータ変更（Insert, Update, Delete）は、すべて `SqliteDatabaseWriter` 直列化キューを経由して実行し、マルチスレッド環境での `SQLITE_BUSY` 例外を物理遮断する。

---

# 7. Database Integrity & Corruption Handling

## 7.1 破損検知時の振る舞い
起動時または実行中に SQLite の破損（`SQLite Error 11: database disk image is malformed` 等）を検知した場合：
1. 破損 DB への書き込みを即座に中断。
2. 破損ファイルを `%LocalAppData%\GameSecurityTool\Corrupted\gamesecurity_<timestamp>.db` へ孤立退避。
3. `GameSecurityTool.Recovery.exe` を起動し、直前の正常バックアップからの復旧または初期再構築（Factory Reset）をユーザーに選択させる。Recovery Host は通常 SQLite DB / normal configuration store / MainWindow に依存しない。

---

# 8. Database Checklist (スキーマ変更時確認)

変更前に以下をすべてクリアすること：

☐ `SchemaVersion` が正しくインクリメントされている  
☐ 移行前バックアップ処理が実装されている  
☐ **Enum に明示的な数値が固定されており、追記専用原則（末尾追加）が守られている**  
☐ 廃止された Enum 値が削除されず `[Obsolete]` として予約されている  
☐ すべての書き込みが `SqliteDatabaseWriter` 経由で行われている  
☐ ロールバックテストが単体テストで検証されている  

---

# 9. Final Database Statement

GST の Database 設計とは、単に値を保存する仕組みではない。

**長期間にわたり、ユーザー資産の安全な履歴と監査証跡の意味を 1 ビットも狂わせずに保持し続けるための基盤** である。

---

Final Principle:

```
Enforce Explicit Enum Values
Append-Only For All Serialized Enums
Migrate Safely With Automatic Backup
Never Mutate Past Audit Meaning
```

---

End of Document
```

---

