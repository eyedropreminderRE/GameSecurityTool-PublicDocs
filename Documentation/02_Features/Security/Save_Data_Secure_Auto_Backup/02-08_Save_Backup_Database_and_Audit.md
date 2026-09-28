# GameSecurityTool Save Backup Database & Audit Specification

**Document ID:** GST-FEAT-SAVE-DB-008  
**Version:** 4.1 (Durable Restore-State Boundary Clarified)
**Status:** Approved Infrastructure Specification  
**Target Layer:** Infrastructure Layer (`GameSecurityTool.Infrastructure.Persistence`)  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. SQLite テーブル定義 (EF Core Schema)

Save Backup 機能のメタデータ、スナップショット履歴、復元トランザクション、ポリシー設定を保存する SQLite テーブルスキーマ。

```text
┌─────────────────────────┐ 1     N ┌─────────────────────────┐
│   SaveBackupSnapshots   ├────────>│   SaveBackupFileEntries │ (DeleteBehavior.Restrict)
└───────────┬─────────────┘         └─────────────────────────┘
            │ 1
            │
            ▼ N
┌─────────────────────────┐         ┌─────────────────────────┐
│   RestoreTransactions   │         │   SaveBackupPolicies    │
└─────────────────────────┘         └─────────────────────────┘
```

## 1.1 SaveBackupSnapshots (スナップショットメタデータ)

| カラム名 | SQLite型 | C#型 | 制約 | 説明 |
| :--- | :--- | :--- | :--- | :--- |
| **Id** | TEXT (GUID) | Guid | PK, NOT NULL | バックアップ一意識別子 |
| **GameProfileId** | TEXT (GUID) | Guid | FK, NOT NULL, INDEX | 対象ゲームプロファイルID |
| **OperationId** | TEXT | string | NOT NULL, INDEX | 業務操作追跡ID |
| **TransactionId** | TEXT | string | NOT NULL, INDEX | 単一処理原子性追跡ID |
| **SnapshotVersion** | TEXT | string | NOT NULL, DEFAULT('1.0') | スキーマバージョン |
| **StoragePath** | TEXT | string | NOT NULL | コンテナ物理保存パス |
| **FileCount** | INTEGER | int | NOT NULL | ファイル総数 |
| **TotalSizeBytes** | INTEGER | long | NOT NULL | 合計バイト数 |
| **ManifestHash** | TEXT | string | NOT NULL | マニフェスト全体のSHA256 (秒精度正規化) |
| **Status** | INTEGER | int | NOT NULL | Creating(0), Completed(1), Failed(2), Corrupted(3), Deleted(4) |
| **RetentionStatus** | INTEGER | int | NOT NULL | Active(0), Protected(1), Candidate(2), Removed(3) |
| **IsPinned** | INTEGER | bool | NOT NULL, DEFAULT(0), INDEX | 📌 ピン留めフラグ (1=永久保護) |
| **TagsJson** | TEXT | string | NOT NULL, DEFAULT('[]') | タグ配列 JSON 文字列 |
| **UserNote** | TEXT | string? | NULL | ユーザー任意メモ (MOD状態等) |
| **CreatedAt** | TEXT | DateTimeOffset | NOT NULL, INDEX | 生成日時 |

## 1.2 SaveBackupFileEntries (ファイルマニフェスト)

| カラム名 | SQLite型 | C#型 | 制約 | 説明 |
| :--- | :--- | :--- | :--- | :--- |
| **Id** | INTEGER | long | PK, AUTOINCREMENT | 連番ID |
| **BackupId** | TEXT (GUID) | Guid | FK, NOT NULL, INDEX | 親スナップショットID |
| **RelativePath** | TEXT | string | NOT NULL, INDEX | フォルダ相対パス |
| **SizeBytes** | INTEGER | long | NOT NULL | ファイルサイズ |
| **LastWriteTimeUtc**| TEXT | DateTimeOffset | NOT NULL | 更新日時 (秒精度) |
| **FileId** | TEXT | string? | NULL | NTFS FileId (取得時のみ) |
| **SHA256** | TEXT | string | NOT NULL | ファイルSHA256ハッシュ |
| **SourceContainerBackupId** | TEXT (GUID) | Guid | FK, NOT NULL, INDEX | ペイロード実体を保持するスナップショットのID (差分参照用) |

## 1.3 RestoreTransactions (復元トランザクション履歴 - CRIT-06 連携)

| カラム名 | SQLite型 | C#型 | 制約 | 説明 |
| :--- | :--- | :--- | :--- | :--- |
| **Id** | TEXT (GUID) | Guid | PK, NOT NULL | レコードID |
| **TransactionId** | TEXT | string | NOT NULL, UNIQUE, INDEX | 単一処理追跡ID (RST-20260827-XXXX) |
| **OperationId** | TEXT | string | NOT NULL, INDEX | 業務操作追跡ID |
| **BackupId** | TEXT (GUID) | Guid | FK, NOT NULL | 復元元バックアップID |
| **GameProfileId** | TEXT (GUID) | Guid | FK, NOT NULL | 対象ゲームプロファイルID |
| **TargetRootPath** | TEXT | string | NOT NULL | 復元先ルート絶対パス |
| **Status** | INTEGER | int | NOT NULL | Initialized(0), Validating(1), PreRestoreBackingUp(2), Restoring(3), **4は予約値（旧HashVerifying・非永続）**, Completed(5), PartialFailed(6), RolledBack(7), Failed(8)。Hash検証およびMetadata/タイムスタンプ復元は `RestoreContainerAsync` 内部サブステップとして扱い、永続Statusにはしない。 |
| **FailedStep** | TEXT | string? | NULL | 失敗ステップ名 |
| **ErrorMessage** | TEXT | string? | NULL | エラー詳細 |
| **StartedAt** | TEXT | DateTimeOffset | NOT NULL | 復元開始日時 |
| **CompletedAt** | TEXT | DateTimeOffset?| NULL | 復元完了日時 |

## 1.4 SaveBackupPolicies (個別バックアップポリシー)

| カラム名 | SQLite型 | C#型 | 制約 | 説明 |
| :--- | :--- | :--- | :--- | :--- |
| **Id** | TEXT (GUID) | Guid | PK, NOT NULL | ポリシーID |
| **GameProfileId** | TEXT (GUID) | Guid? | NULL, INDEX | null = Global既定値 |
| **FeatureState** | INTEGER | int | NOT NULL | UseGlobal(0), Enabled(1), Disabled(2) |
| **BackupMode** | INTEGER | int | NOT NULL | Standard(0), SSDLowImpact(1), MaximumProtection(2) |
| **MaxGenerations** | INTEGER | int | NOT NULL, DEFAULT(3) | 保持世代数 (カスタム指定可能) |
| **MaxStorageSizeBytes**| INTEGER | long | NOT NULL, DEFAULT(5368709120) | 容量上限 (5GB / カスタム指定可能) |
| **CustomBackupLocation**| TEXT | string? | NULL | ユーザー指定保存先 (別ドライブ HDD 等) |
| **MinimumFreeSpaceBytes**| INTEGER | long | NOT NULL, DEFAULT(2147483648) | 空き容量下限 (2GB) |
| **ModifiedAt** | TEXT | DateTimeOffset | NOT NULL | 最終更新日時 |

---

# 2. EF Core 10 Fluent API マッピング構成 (CRIT-05 是正)

```csharp
namespace GameSecurityTool.Infrastructure.Persistence;

using System;
using System.Collections.Generic;
using System.Text.Json;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Storage.ValueConversion;
using GameSecurityTool.Infrastructure.Persistence.Entities;

public sealed class SaveBackupDbContextModelBuilder
{
    public static void Configure(ModelBuilder modelBuilder)
    {
        var tagsConverter = new ValueConverter<IReadOnlyList<string>, string>(
            v => JsonSerializer.Serialize(v, (JsonSerializerOptions?)null),
            v => JsonSerializer.Deserialize<List<string>>(v, (JsonSerializerOptions?)null) ?? new List<string>()
        );

        // --- SaveBackupSnapshots ---
        modelBuilder.Entity<SaveBackupSnapshotRecord>(b =>
        {
            b.ToTable("SaveBackupSnapshots");
            b.HasKey(s => s.Id);
            b.HasIndex(s => s.GameProfileId);
            b.HasIndex(s => s.OperationId);
            b.HasIndex(s => s.TransactionId);
            b.HasIndex(s => s.CreatedAt);
            b.HasIndex(s => s.IsPinned);
            
            // Enum は int 永続化
            b.Property(s => s.Status).HasConversion<int>();
            b.Property(s => s.RetentionStatus).HasConversion<int>();

            b.Property(s => s.Tags)
             .HasColumnName("TagsJson")
             .HasConversion(tagsConverter);

            // 自スナップショット配下のファイルエントリの削除抑止
            b.HasMany(s => s.FileEntries)
             .WithOne()
             .HasForeignKey(f => f.BackupId)
             .OnDelete(DeleteBehavior.Restrict);
        });

        // --- SaveBackupFileEntries ---
        modelBuilder.Entity<SaveBackupFileRecord>(b =>
        {
            b.ToTable("SaveBackupFileEntries");
            b.HasKey(f => f.Id);
            b.HasIndex(f => new { f.BackupId, f.RelativePath });
            
            // 【CRIT-05 是正】差分参照整合性を DB スキーマレベルで強制し、参照先コンテナのメタデータ誤削除を防ぐ
            b.HasOne<SaveBackupSnapshotRecord>()
             .WithMany()
             .HasForeignKey(f => f.SourceContainerBackupId)
             .OnDelete(DeleteBehavior.Restrict);
        });

        // --- RestoreTransactions ---
        modelBuilder.Entity<RestoreTransactionRecord>(b =>
        {
            b.ToTable("RestoreTransactions");
            b.HasKey(r => r.Id);
            b.HasIndex(r => r.TransactionId).IsUnique();
            b.HasIndex(r => new { r.GameProfileId, r.StartedAt });
            b.Property(r => r.Status).HasConversion<int>();
        });

        // --- SaveBackupPolicies ---
        modelBuilder.Entity<SaveBackupPolicyRecord>(b =>
        {
            b.ToTable("SaveBackupPolicies");
            b.HasKey(p => p.Id);
            b.HasIndex(p => p.GameProfileId).IsUnique();
            b.Property(p => p.FeatureState).HasConversion<int>();
            b.Property(p => p.BackupMode).HasConversion<int>();
        });
    }
}
```

---

# 3. SQLite 直列化キュー統合リポジトリ (`SqliteSaveBackupRepository.cs` - CRIT-05 ＆ CRIT-06 是正)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Persistence.Repositories;

using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Infrastructure.Persistence.Entities;
using Microsoft.EntityFrameworkCore;

public sealed class SqliteSaveBackupRepository(
    IDbContextFactory<AppDbContext> dbFactory,
    IDbWriteQueue dbWriter) : ISaveBackupRepository
{
    public async Task SaveSnapshotAsync(SaveBackupSnapshotDto dto, CancellationToken ct = default)
    {
        await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
        {
            await using var db = await dbFactory.CreateDbContextAsync(innerCt);
            var record = MapToRecord(dto);
            await db.Set<SaveBackupSnapshotRecord>().AddAsync(record, innerCt);
            await db.SaveChangesAsync(innerCt);
        }, ct);
    }

    public async Task<SaveBackupSnapshotDto?> GetSnapshotByIdAsync(Guid backupId, CancellationToken ct = default)
    {
        await using var db = await dbFactory.CreateDbContextAsync(ct);
        var record = await db.Set<SaveBackupSnapshotRecord>()
            .AsNoTracking()
            .FirstOrDefaultAsync(s => s.Id == backupId, ct);

        return record == null ? null : MapToDto(record);
    }

    public async Task<IReadOnlyList<SaveBackupSnapshotDto>> GetSnapshotsByGameProfileIdAsync(Guid gameProfileId, CancellationToken ct = default)
    {
        await using var db = await dbFactory.CreateDbContextAsync(ct);
        var records = await db.Set<SaveBackupSnapshotRecord>()
            .AsNoTracking()
            .Where(s => s.GameProfileId == gameProfileId)
            .OrderByDescending(s => s.CreatedAt)
            .ToListAsync(ct);

        return records.Select(MapToDto).ToList();
    }

    public async Task<PagedResultDto<SaveBackupSnapshotDto>> GetPagedSnapshotsAsync(
        Guid gameProfileId, int pageNumber, int pageSize, CancellationToken ct = default)
    {
        await using var db = await dbFactory.CreateDbContextAsync(ct);
        var query = db.Set<SaveBackupSnapshotRecord>()
            .AsNoTracking()
            .Where(s => s.GameProfileId == gameProfileId)
            .OrderByDescending(s => s.IsPinned) // 📌 ピン留めを最優先表示
            .ThenByDescending(s => s.CreatedAt);

        int totalCount = await query.CountAsync(ct);
        var items = await query
            .Skip((pageNumber - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync(ct);

        return new PagedResultDto<SaveBackupSnapshotDto>(
            items.Select(MapToDto).ToList(),
            totalCount,
            pageNumber,
            pageSize
        );
    }

    public async Task UpdateSnapshotMetadataAsync(
        Guid backupId, bool isPinned, IReadOnlyList<string> tags, string? userNote, CancellationToken ct = default)
    {
        await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
        {
            await using var db = await dbFactory.CreateDbContextAsync(innerCt);
            var record = await db.Set<SaveBackupSnapshotRecord>().FirstOrDefaultAsync(s => s.Id == backupId, innerCt);
            if (record != null)
            {
                record.IsPinned = isPinned;
                record.RetentionStatus = isPinned ? 1 : 0; // Protected = 1, Active = 0
                record.Tags = tags.ToList();
                record.UserNote = userNote;
                await db.SaveChangesAsync(innerCt);
            }
        }, ct);
    }

    public async Task UpdateSnapshotStoragePathAsync(Guid backupId, string newStoragePath, CancellationToken ct = default)
    {
        await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
        {
            await using var db = await dbFactory.CreateDbContextAsync(innerCt);
            var record = await db.Set<SaveBackupSnapshotRecord>().FirstOrDefaultAsync(s => s.Id == backupId, innerCt);
            if (record != null)
            {
                record.StoragePath = newStoragePath;
                await db.SaveChangesAsync(innerCt);
            }
        }, ct);
    }

    public async Task UpdateSnapshotStatusAsync(Guid backupId, string status, CancellationToken ct = default)
    {
        await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
        {
            await using var db = await dbFactory.CreateDbContextAsync(innerCt);
            var record = await db.Set<SaveBackupSnapshotRecord>().FirstOrDefaultAsync(s => s.Id == backupId, innerCt);
            if (record != null && int.TryParse(status, out int parsedStatus))
            {
                record.Status = parsedStatus;
                await db.SaveChangesAsync(innerCt);
            }
        }, ct);
    }

    /// <summary>
    /// 【CRIT-05 是正】依存関係を事前検証し、他スナップショットからの参照がある場合は削除を拒否する。
    /// </summary>
    public async Task DeleteSnapshotAsync(Guid backupId, CancellationToken ct = default)
    {
        await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
        {
            await using var db = await dbFactory.CreateDbContextAsync(innerCt);
            // 差分バックアップの参照依存性事前チェック
            bool isReferenced = await db.Set<SaveBackupFileRecord>()
                .AnyAsync(f => f.SourceContainerBackupId == backupId && f.BackupId != backupId, innerCt);

            if (isReferenced)
            {
                throw new InvalidOperationException($"スナップショット {backupId} は、後続の差分バックアップから実体コンテナとして参照されているため削除できません。");
            }

            var entries = await db.Set<SaveBackupFileRecord>().Where(f => f.BackupId == backupId).ToListAsync(innerCt);
            db.Set<SaveBackupFileRecord>().RemoveRange(entries);

            var snapshot = await db.Set<SaveBackupSnapshotRecord>().FirstOrDefaultAsync(s => s.Id == backupId, innerCt);
            if (snapshot != null)
            {
                db.Set<SaveBackupSnapshotRecord>().Remove(snapshot);
            }

            await db.SaveChangesAsync(innerCt);
        }, ct);
    }

    public async Task<IReadOnlyList<Guid>> GetDependentSnapshotIdsAsync(Guid backupId, CancellationToken ct = default)
    {
        await using var db = await dbFactory.CreateDbContextAsync(ct);
        return await db.Set<SaveBackupFileRecord>()
            .AsNoTracking()
            .Where(f => f.SourceContainerBackupId == backupId && f.BackupId != backupId)
            .Select(f => f.BackupId)
            .Distinct()
            .ToListAsync(ct);
    }

    /// <summary>
    /// 【品質改善記録】保持期限パージ処理の N+1 を解消するバッチ参照元スナップショットID一括取得
    /// </summary>
    public async Task<IReadOnlyDictionary<Guid, bool>> GetReferencedSourceBackupIdsAsync(
        IEnumerable<Guid> sourceBackupIds,
        CancellationToken ct = default)
    {
        ArgumentNullException.ThrowIfNull(sourceBackupIds);

        var requestedIds = sourceBackupIds.Distinct().ToArray();
        if (requestedIds.Length == 0)
        {
            return new Dictionary<Guid, bool>();
        }

        await using var db = await dbFactory.CreateDbContextAsync(ct);
        var referencedIds = await db.Set<SaveBackupFileRecord>()
            .AsNoTracking()
            .Where(f => requestedIds.Contains(f.SourceContainerBackupId) &&
                        f.BackupId != f.SourceContainerBackupId)
            .Select(f => f.SourceContainerBackupId)
            .Distinct()
            .ToListAsync(ct);

        var referenced = referencedIds.ToHashSet();
        return requestedIds.ToDictionary(id => id, referenced.Contains);
    }

    // ==========================================
    // 【CRIT-06 連携】起動時クラッシュリカバリ実装
    // ==========================================

    public async Task<IReadOnlyList<RestoreTransactionDto>> GetIncompleteRestoreTransactionsAsync(CancellationToken ct = default)
    {
        await using var db = await dbFactory.CreateDbContextAsync(ct);
        // Completed(5), RolledBack(7), Failed(8) 以外の未完了トランザクションを抽出
        var incompleteRecords = await db.Set<RestoreTransactionRecord>()
            .AsNoTracking()
            .Where(r => r.Status != 5 && r.Status != 7 && r.Status != 8)
            .OrderByDescending(r => r.StartedAt)
            .ToListAsync(ct);

        return incompleteRecords.Select(r => new RestoreTransactionDto(
            r.Id, r.TransactionId, r.OperationId, r.BackupId, r.GameProfileId,
            r.TargetRootPath, r.Status.ToString(), r.FailedStep, r.ErrorMessage,
            r.StartedAt, r.CompletedAt)).ToList();
    }

    public async Task SaveRestoreTransactionAsync(RestoreTransactionDto dto, CancellationToken ct = default)
    {
        await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
        {
            await using var db = await dbFactory.CreateDbContextAsync(innerCt);
            var record = new RestoreTransactionRecord
            {
                Id = dto.Id,
                TransactionId = dto.TransactionId,
                OperationId = dto.OperationId,
                BackupId = dto.BackupId,
                GameProfileId = dto.GameProfileId,
                TargetRootPath = dto.TargetRootPath,
                Status = int.Parse(dto.Status),
                FailedStep = dto.FailedStep,
                ErrorMessage = dto.ErrorMessage,
                StartedAt = dto.StartedAt,
                CompletedAt = dto.CompletedAt
            };
            await db.Set<RestoreTransactionRecord>().AddAsync(record, innerCt);
            await db.SaveChangesAsync(innerCt);
        }, ct);
    }

    public async Task UpdateRestoreTransactionStatusAsync(
        string transactionId, string status, string? failedStep, string? errorMessage, CancellationToken ct = default)
    {
        await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
        {
            await using var db = await dbFactory.CreateDbContextAsync(innerCt);
            var record = await db.Set<RestoreTransactionRecord>().FirstOrDefaultAsync(r => r.TransactionId == transactionId, innerCt);
            if (record != null && int.TryParse(status, out int parsedStatus))
            {
                record.Status = parsedStatus;
                record.FailedStep = failedStep;
                record.ErrorMessage = errorMessage;
                if (record.Status == 5 || record.Status == 7 || record.Status == 8) // Completed / RolledBack / Failed
                {
                    record.CompletedAt = DateTimeOffset.UtcNow;
                }
                await db.SaveChangesAsync(innerCt);
            }
        }, ct);
    }

    private static SaveBackupSnapshotRecord MapToRecord(SaveBackupSnapshotDto dto)
    {
        return new SaveBackupSnapshotRecord
        {
            Id = dto.BackupId,
            GameProfileId = dto.GameProfileId,
            OperationId = dto.OperationId,
            TransactionId = dto.TransactionId,
            SnapshotVersion = dto.SnapshotVersion,
            StoragePath = dto.StoragePath,
            FileCount = dto.FileCount,
            TotalSizeBytes = dto.TotalSizeBytes,
            ManifestHash = dto.ManifestHash,
            Status = int.Parse(dto.Status),
            RetentionStatus = int.Parse(dto.RetentionStatus),
            IsPinned = dto.IsPinned,
            Tags = dto.Tags.ToList(),
            UserNote = dto.UserNote,
            CreatedAt = dto.CreatedAt
        };
    }

    private static SaveBackupSnapshotDto MapToDto(SaveBackupSnapshotRecord record)
    {
        return new SaveBackupSnapshotDto(
            record.Id,
            record.GameProfileId,
            record.OperationId,
            record.TransactionId,
            record.SnapshotVersion,
            record.CreatedAt,
            record.FileCount,
            record.TotalSizeBytes,
            record.ManifestHash,
            record.StoragePath,
            record.Status.ToString(),
            record.RetentionStatus.ToString(),
            record.IsPinned,
            record.Tags,
            record.UserNote
        );
    }
}
```

---

# 4. 改ざん検知ログ ＆ 監査チェーン連携 (Audit Integration)

Save Backup の全重要イベントは、改ざん検知 Hash Chain 構造を持つ `TamperEvidentAuditLogger` へコミットする。

```text
[Backup / Pin / Restore / Migrate 操作]
                  │
                  ▼
[OperationId 発行 (例: GST-OP-20260827-001)]
                  │
                  ▼
┌────────────────────────────────────────────────────────┐
│ TamperEvidentAuditLogger.AppendLogAsync()              │
│  - PreviousHash: 直前の CurrentHash                    │
│  - CurrentHash: SHA256(PrevHash + Event + OpId + Time) │
│  - LatestHash を DPAPI / 特権領域へ二重保存            │
└────────────────────────────────────────────────────────┘
```

## 4.1 多層アンカー保護 ＆ Fail-Closed 検証規約 (approved quality refinement)
1. **Layer 1 (ローカル DPAPI - 即時同期):**
   非特権プロセスやマルウェアによるローカル DB 改ざんを即時検知。
2. **Layer 2 (特権管理者アンカー - 非同期同期 ＆ Fail-Closed 規約):**
   特権ワーカー連携により the protected local audit anchor へ最新ハッシュを複製保存。
   **【Fail-Closed 規約】`VerifyAuditChainIntegrityAsync` は、検証時に Layer 2 アンカーが未同期（不在・空）またはハッシュ不整合である場合、サイレントに成功扱い（Fail-Open）することを厳禁とし、厳格に `false` を返却する。**
3. **OOM (Out-of-Memory) 防止規約:**
   数万〜数十万件に及ぶ長大監査チェーンの検証時、全件を一度にメモリへ `ToList()` することを禁止する。1,000 件単位のバッチ読み出し（チャンク検証）または EF Core 非同期ストリーミング（`AsAsyncEnumerable()`）を用いて省メモリに検証を実行する。

---