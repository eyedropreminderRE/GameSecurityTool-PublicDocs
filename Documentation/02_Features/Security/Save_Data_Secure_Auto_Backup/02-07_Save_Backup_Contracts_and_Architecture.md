# GameSecurityTool Save Backup Contracts & Architecture Specification

**Document ID:** GST-FEAT-SAVE-CONTRACTS-007  
**Version:** 4.3
**Status:** Approved Contracts Specification  
**Target Layer:** Contracts Layer (`GameSecurityTool.Contracts`)  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Port & Adapter Boundary Rules

Clean Architecture および Dependency Inversion Principle（DIP）に基づき、各レイヤー間のインターフェースおよびデータ転送オブジェクト（DTO）を `GameSecurityTool.Contracts` に集約定義する。

### レイヤー結合の原則
- **GST.Contracts**: 外部依存を一切持たない純粋な .NET 10 クラスライブラリ。パスワード等の機密情報は `byte[]`（または `SecureString`）で宣言し、`string` を使用しない。
- **GST.Application**: Contracts のインターフェース（Port）を実装または呼び出し、Use Case を制御。
- **GST.Infrastructure**: Contracts のインターフェースを実装（Adapter）。
- **GST.Presentation**: `ISaveBackupService` ファサードおよび DTO のみを利用し、Entity や Infrastructure 具象型を直接参照しない。

---

# 2. サービス・ファサードインターフェース (Application Facade Port)

Presentation 層（UI / ViewModel）が Application 層のユースケースを実行するための高レベル契約。

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;

public interface ISaveBackupService
{
    // --- 統合ハブ (Save Data Hub) 向け API ---

    /// <summary>
    /// 全ゲームのバックアップ状態・使用容量サマリーを一括取得します。
    /// </summary>
    Task<IReadOnlyList<GameBackupSummaryDto>> GetAllGameBackupSummariesAsync(CancellationToken ct = default);

    /// <summary>
    /// 登録されている全ゲームの最新状態を一括バックアップします (直列実行)。
    /// </summary>
    Task ExecuteBulkBackupAsync(CancellationToken ct = default);


    // --- 個別ゲームバックアップ・復元 API ---

    /// <summary>
    /// 指定されたゲームのセーブデータ手動バックアップを実行します。
    /// </summary>
    Task<SaveBackupResultDto> ExecuteManualBackupAsync(
        Guid gameProfileId, 
        IProgress<double>? progress = null, 
        CancellationToken ct = default);

    /// <summary>
    /// 指定されたスナップショットからセーブデータを安全に復元します (RescueSnapshot 自動退避)。
    /// </summary>
    Task<SaveRestoreResultDto> ExecuteRestoreAsync(
        Guid backupId, 
        IProgress<double>? progress = null, 
        CancellationToken ct = default);


    // --- スナップショット管理 API ---

    /// <summary>
    /// スナップショットのピン留め (永久保護) 状態を切り替えます。
    /// </summary>
    Task<bool> PinSnapshotAsync(
        Guid backupId, 
        bool isPinned, 
        CancellationToken ct = default);

    /// <summary>
    /// スナップショットのタグおよびユーザーメモを更新します。
    /// </summary>
    Task<bool> UpdateSnapshotMetadataAsync(
        Guid backupId, 
        IReadOnlyList<string> tags, 
        string? userNote, 
        CancellationToken ct = default);

    /// <summary>
    /// 指定されたスナップショットを手動で安全削除します (差分参照依存性検証 ＆ GC 連動)。
    /// </summary>
    Task<bool> DeleteSnapshotAsync(Guid backupId, CancellationToken ct = default);

    /// <summary>
    /// 既存のスナップショットを元に、新しいタグ/メモを付与したブランチスナップショットを複製します。
    /// </summary>
    Task<SaveBackupResultDto> DuplicateSnapshotAsync(
        Guid sourceBackupId, 
        string? newNote, 
        IReadOnlyList<string>? newTags, 
        CancellationToken ct = default);


    // --- エクスポート ＆ ストレージ管理 API ---

    /// <summary>
    /// スナップショットを汎用 ZIP ファイルとして外部エクスポートします (クラウド退避等)。
    /// </summary>
    Task<bool> ExportToStandardZipAsync(
        ExportBackupRequestDto request, 
        CancellationToken ct = default);

    /// <summary>
    /// バックアップの保存先フォルダ設定を変更します。
    /// </summary>
    Task UpdateBackupLocationSettingAsync(
        Guid gameProfileId, 
        string newStoragePath, 
        CancellationToken ct = default);

    /// <summary>
    /// バックアップ保存先フォルダを別ドライブ (HDD 等) へ一括移送します。
    /// </summary>
    Task<StorageMigrationResultDto> MigrateStorageLocationAsync(
        Guid? gameProfileId, 
        string newTargetDirectory, 
        IProgress<double>? progress = null, 
        CancellationToken ct = default);


    // --- 状態照会・監視 API ---

    /// <summary>
    /// セーブデータ容量の急激な肥大化 (Save Bloat バグ) を検知・分析します。
    /// </summary>
    Task<SaveBloatSummaryDto> CheckSaveBloatAsync(
        Guid gameProfileId, 
        CancellationToken ct = default);

    /// <summary>
    /// ゲームプロファイル個別のバックアップ保護状態・最終バックアップ詳細を取得します。
    /// </summary>
    Task<SaveBackupStatusSummaryDto> GetBackupStatusSummaryAsync(
        Guid gameProfileId, 
        CancellationToken ct = default);

    /// <summary>
    /// スナップショット履歴をページネーション形式で取得します。
    /// </summary>
    Task<PagedResultDto<SaveBackupSnapshotDto>> GetSnapshotsPagedAsync(
        Guid gameProfileId, 
        int pageNumber, 
        int pageSize, 
        CancellationToken ct = default);
}
```

---

# 3. インフラストラクチャ操作インターフェース (Infrastructure Ports)

## 3.1 `ISaveBackupStorage` (コンテナストレージ操作 Port)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;

public interface ISaveBackupStorage
{
    Task<SaveBackupContainerResultDto> CreateContainerAsync(
        SaveBackupCreateRequestDto request, 
        IProgress<double>? progress = null, 
        CancellationToken ct = default);

    Task<SaveRestoreFileResultDto> RestoreContainerAsync(
        SaveBackupRestoreRequestDto request, 
        IProgress<double>? progress = null, 
        CancellationToken ct = default);

    Task<bool> VerifyContainerIntegrityAsync(string containerPath, CancellationToken ct = default);
    Task<bool> DeleteContainerAsync(string containerPath, CancellationToken ct = default);
}
```

## 3.2 `ISaveBackupRepository` (永続化操作 Port)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;

public interface ISaveBackupRepository
{
    Task<SaveBackupSnapshotDto?> GetSnapshotByIdAsync(Guid backupId, CancellationToken ct = default);
    Task<IReadOnlyList<SaveBackupSnapshotDto>> GetSnapshotsByGameProfileIdAsync(Guid gameProfileId, CancellationToken ct = default);
    Task<PagedResultDto<SaveBackupSnapshotDto>> GetPagedSnapshotsAsync(Guid gameProfileId, int pageNumber, int pageSize, CancellationToken ct = default);
    Task SaveSnapshotAsync(SaveBackupSnapshotDto snapshot, CancellationToken ct = default);
    Task UpdateSnapshotMetadataAsync(Guid backupId, bool isPinned, IReadOnlyList<string> tags, string? userNote, CancellationToken ct = default);
    Task UpdateSnapshotStoragePathAsync(Guid backupId, string newStoragePath, CancellationToken ct = default);
    Task UpdateSnapshotStatusAsync(Guid backupId, string status, CancellationToken ct = default);
    Task DeleteSnapshotAsync(Guid backupId, CancellationToken ct = default);
    Task<IReadOnlyList<Guid>> GetDependentSnapshotIdsAsync(Guid backupId, CancellationToken ct = default);
    
    Task<IReadOnlyList<RestoreTransactionDto>> GetIncompleteRestoreTransactionsAsync(CancellationToken ct = default);
    Task SaveRestoreTransactionAsync(RestoreTransactionDto transaction, CancellationToken ct = default);
    Task UpdateRestoreTransactionStatusAsync(string transactionId, string status, string? failedStep, string? errorMessage, CancellationToken ct = default);
    
    // 【C3 是正】パージポリシー（24時間 / 7日間）の DB 主導判定用メソッド
    Task<IReadOnlyList<string>> GetExpiredTransactionsForPurgeAsync(DateTimeOffset nowUtc, CancellationToken ct = default);

    /// <summary>
    /// 【品質改善記録】保持期限パージ処理における N+1 クエリを解消し、
    /// 他のスナップショットから差分参照元として参照されている BackupId 一覧をバッチ一括取得します。
    /// </summary>
    Task<IReadOnlyDictionary<Guid, bool>> GetReferencedSourceBackupIdsAsync(IEnumerable<Guid> sourceBackupIds, CancellationToken ct = default);
}
```

## 3.3 `ISaveDataPathScanner` (セーブデータ走査 Port)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;

public interface ISaveDataPathScanner
{
    Task<IReadOnlyList<SaveFileMetadataDto>> ScanSaveDirectoryAsync(string rootPath, CancellationToken ct = default);
}
```

## 3.4 `IRescueSnapshotStorage` (復元前退避・ロールバック操作 Port)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System.Threading;
using System.Threading.Tasks;

public interface IRescueSnapshotStorage
{
    /// <summary>
    /// 現在のセーブデータを復元前に一時退避します (RescueSnapshot のアトミック生成)。
    /// </summary>
    Task CreateSnapshotAsync(string transactionId, string sourceDir, CancellationToken ct = default);

    /// <summary>
    /// 復元処理が失敗した場合に、RescueSnapshot から元の状態へアトミックにロールバックします。
    /// </summary>
    Task RollbackAsync(string transactionId, string targetDir, CancellationToken ct = default);

    /// <summary>
    /// 指定されたトランザクションの RescueSnapshot が完全に存在するかどうか（マニフェスト検証含む）を確認します。
    /// </summary>
    Task<bool> HasSnapshotAsync(string transactionId, CancellationToken ct = default);

    /// <summary>
    /// Application 層の DB 主導パージポリシーに従い、指定された RescueSnapshot を安全に削除します。
    /// </summary>
    Task DeleteSnapshotAsync(string transactionId, CancellationToken ct = default);
}
```

## 3.5 `IExternalArchiverAdapter` (外部ファイル圧縮ソフトウェア連携 Port)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

public interface IExternalArchiverAdapter
{
    /// <summary>
    /// 外部ファイル圧縮ソフトウェアを安全に呼び出し、アーカイブを作成します。
    /// </summary>
    Task<bool> ArchiveFilesAsync(string destinationArchive, IReadOnlyList<string> sourceFiles, string? password = null, CancellationToken ct = default);

    /// <summary>
    /// 外部ファイル圧縮ソフトウェアを安全に呼び出し、アーカイブを展開します。
    /// </summary>
    Task<bool> ExtractArchiveAsync(string sourceArchive, string destinationDirectory, string? password = null, CancellationToken ct = default);

    /// <summary>
    /// 外部ファイル圧縮ソフトウェアがシステム上で利用可能か確認します。
    /// </summary>
    bool IsExternalArchiverAvailable();
}
```

---

# 4. データ転送オブジェクト (Sealed Record DTOs)

機密情報（パスワード）については、`string` の利用を厳禁とし `byte[]?` で定義する（CRIT-06 是正規約）。

```csharp
namespace GameSecurityTool.Contracts.Dtos;

using System;
using System.Collections.Generic;

// --- 統合ハブ・エクスポート関連 DTO ---

public sealed record GameBackupSummaryDto(
    Guid GameProfileId,
    string GameDisplayName,
    DateTimeOffset? LastBackupTimeUtc,
    int SnapshotCount,
    int PinnedCount,
    long TotalStorageUsageBytes
);

public sealed record ExportBackupRequestDto(
    Guid GameProfileId,
    string ExportMode, // "LatestOnly", "PinnedOnly", "FullArchive"
    string DestinationPath,
    byte[]? Password // CRIT-06: メモリダンプ防止のため byte[] を使用
);

// --- approved security contract Password ownership contract ---
// Save Backup Password buffers are transfer-owned, not borrowed. Ownership follows
// the request through Application/Queue and transfers to Storage at the ISaveBackupStorage
// Create/Restore call boundary. The current owner zeroizes only buffers it still owns
// when a request is rejected, abandoned, replaced, or otherwise stops before handoff.
// After a successful handoff, the previous owner must not reuse, mutate, or zeroize the buffer.
// --- バックアップ生成・復元関連 DTO ---

public sealed record SaveBackupCreateRequestDto(
    string OperationId,
    string TransactionId,
    Guid GameProfileId,
    string GameDisplayName,
    byte[]? Password, // approved security contract: ownership-transfer buffer; Storage is final transfer boundary
    string SourceRootPath,
    string TargetStorageDirectory,
    IReadOnlyList<SaveFileMetadataDto> FilesToBackup
);

public sealed record SaveBackupContainerResultDto(
    bool Success,
    Guid BackupId,
    string ContainerPath,
    string ManifestHash,
    long TotalSizeBytes,
    int FileCount,
    string? ErrorMessage
);

public sealed record SaveBackupRestoreRequestDto(
    string OperationId,
    string TransactionId,
    Guid BackupId,
    byte[]? Password, // approved security contract: ownership-transfer buffer; Storage is final transfer boundary
    string ContainerPath,
    string TargetRestoreRootPath,
    string? ExpectedManifestHash
);

public sealed record SaveRestoreFileResultDto(
    bool Success,
    string TransactionId,
    int RestoredFileCount,
    string? FailedStep,
    string? ErrorMessage
);

public sealed record SaveFileMetadataDto(
    string RelativePath,
    string AbsolutePath,
    long SizeBytes,
    DateTimeOffset LastWriteTimeUtc,
    string? FileId,
    string? SHA256
);

// --- メタデータ・照会関連 DTO ---

public sealed record SaveBackupSnapshotDto(
    Guid BackupId,
    Guid GameProfileId,
    string OperationId,
    string TransactionId,
    string SnapshotVersion,
    DateTimeOffset CreatedAt,
    int FileCount,
    long TotalSizeBytes,
    string ManifestHash,
    string StoragePath,
    string Status,
    string RetentionStatus,
    bool IsPinned,
    IReadOnlyList<string> Tags,
    string? UserNote
);

public sealed record SaveBackupResultDto(
    bool Success,
    bool IsSkipped,
    Guid? BackupId,
    string? Message
)
{
    public static SaveBackupResultDto Succeeded(Guid backupId) => new(true, false, backupId, null);
    public static SaveBackupResultDto Skipped() => new(true, true, null, "変更がないためスキップされました。");
    public static SaveBackupResultDto Failed(string error) => new(false, false, null, error);
    public static SaveBackupResultDto Frozen(string reason) => new(false, true, null, $"エントロピー急変検知によりスナップショット作成が緊急凍結されました: {reason}");
}

public sealed record SaveRestoreResultDto(
    bool Success,
    string TransactionId,
    string? FailedStep,
    string? ErrorMessage
);

public sealed record RestoreTransactionDto(
    Guid Id,
    string TransactionId,
    string OperationId,
    Guid BackupId,
    Guid GameProfileId,
    string TargetRootPath,
    string Status,
    string? FailedStep,
    string? ErrorMessage,
    DateTimeOffset StartedAt,
    DateTimeOffset? CompletedAt
);

public sealed record StorageMigrationResultDto(
    bool Success,
    int MigratedContainerCount,
    long TotalMigratedBytes,
    string NewStorageDirectory,
    string? ErrorMessage
);

public sealed record SaveBloatSummaryDto(
    bool IsAbnormalGrowthDetected,
    long CurrentSizeBytes,
    long PreviousSizeBytes,
    double GrowthPercentage,
    string RecommendationMessage
);

public sealed record SaveBackupStatusSummaryDto(
    Guid GameProfileId,
    bool IsEnabled,
    string EffectiveMode,
    DateTimeOffset? LastBackupTime,
    long TotalStorageUsageBytes,
    int TotalSnapshotCount,
    int PinnedSnapshotCount,
    IReadOnlyList<SaveBackupSnapshotDto> RecentSnapshots
);

public sealed record PagedResultDto<T>(
    IReadOnlyList<T> Items,
    int TotalCount,
    int PageNumber,
    int PageSize
);
```

---

# 5. レイヤー間 Enum ⇄ DTO ⇄ DB 型マッピング規約

| レイヤー | 表現形式 | 変換方式 |
| :--- | :--- | :--- |
| **Domain Layer** | `SnapshotStatus` (Enum) | 純粋な C# Enum |
| **Contracts DTO** | `string Status` (文字列) | `enum.ToString()` / `Enum.Parse<T>()` |
| **Database Record** | `int Status` (整数値) | EF Core `HasConversion<int>()` |

---

