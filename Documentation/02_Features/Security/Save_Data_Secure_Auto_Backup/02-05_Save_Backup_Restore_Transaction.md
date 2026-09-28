# GameSecurityTool Save Backup Restore Transaction Specification

**Document ID:** GST-FEAT-SAVE-RESTORE-005  
**Version:** 4.4
**Status:** Approved Feature Specification  
**Target Layer:** Application Layer / Domain Layer / Infrastructure Layer  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Restore Philosophy & User Consent

復元（Restore）は、ユーザーの既存データを直接書き換える最高リスク操作である。以下の原則を厳守する。

1. **Explicit User Consent**: 自動復元、サイレント復元を禁止する。必ずユーザーが確認ダイアログで明示的に承認した後にのみ開始する。
2. **Atomic State Transition**: 復元処理は単一の原子処理として `RestoreTransaction`（`TransactionId` 発番）管理下で実行する。
3. **Zero Data Loss Guarantee**: 復元が途中で失敗した場合でも、復元直前のセーブデータ状態が失われないよう「復元前一時退避（RescueSnapshot）」を行う。退避ディレクトリのアトミックリネームとマニフェスト検証を用いて、**退避が完全に行われたことが証明されない限り、絶対にロールバック（現行データへの上書き）を実行してはならない**。
4. **Crash Self-Healing**: 復元処理中の突然の電源断やクラッシュに対して、次回起動時に未完了状態を検知して元の状態へ安全に巻き戻す自己修復機構を提供する。
5. **Atomic Rollback Guarantee (C-3 是正)**: ロールバック処理自体の実行中にクラッシュが発生した場合でも、ユーザーの既存ファイルが中途半端な長さに切り詰められて破損することを防ぐため、**必ず一時ファイル（`.rollback.tmp`）への出力完了後にアトミックリネーム（`File.Move`）で差し替える**。
6. **二層整合性モデル (Two-Tier Consistency Model - approved quality refinement)**:
   - **第 1 層 (物理アトミック置換):** 展開・書き戻し時は直接上書きせず、同一ドライブ内の一時ファイル（`.restore.tmp`）へ完全出力後に `File.Move(..., overwrite: true)` で一瞬で置換する。
   - **第 2 層 (トランザクション整合性 ＆ 冪等再実行):** プロセス突然死や電源断時は、次回起動時の自己修復ループ（`ExecuteStartupCrashRecoveryAsync`）により、未完了トランザクションを検知して `RescueSnapshot` からのロールバックまたは安全な再実行を自律調停する。
7. **シャノン・エントロピー検知 ＆ 破損防止**: 復元対象データのシャノン・エントロピーを検証し、ランサムウェアによる暗号化や破損を検知した場合は復元を即時中断してロールバックする。
8. **OS スリープ阻止 ＆ ストレージレジリエンス (Power & Storage Resilience)**:
   復元トランザクションおよびロールバック実行中は、Win32 `SetThreadExecutionState(ES_CONTINUOUS | ES_SYSTEM_REQUIRED)` を発行して OS のスタンバイ・スリープ移行を完全に阻止する。また、復元先がスリープ中の外部 HDD の場合はプリプローブ（Pre-wakeup Probe）によりスピンアップを待機し、I/O タイムアウトによる誤ロールバックを防止する。

---

# 2. RestoreTransaction 状態遷移モデル

```text
[User Confirmation (承認)]
             │
             ▼
      (1. Initialized)
             │
             ▼
      (2. Validating)       ──(検証失敗)──> [RestoreStatus: Failed]
             │
             ▼
 (3. PreRestoreBackingUp)   ──(退避失敗・クラッシュ)──> [RestoreStatus: Failed (中断・上書きなし)]
  (RescueSnapshot 退避)
             │
             ▼
      (4. Restoring)        ──(I/O失敗)───> (Atomic Rollback) ──> [PartialFailed / RolledBack]
             │
             │  ┌─ internal sub-step: Hash verification
             │  │
             │  └─ internal sub-step: Metadata / timestamp restoration
             ▼
     [5. Completed]

※ Hash verification と Metadata / timestamp restoration は
  RestoreTransaction の永続Statusではなく、Restoring 内の内部サブステップとする。
```

---

# 3. 復元前セキュリティ検証 (Pre-Restore Validation)

復元実行前、パス文字列のみを信用せず、以下のセキュリティ検証パイプラインを強制する。

## 3.1 TOCTOU 防御 ＆ Reparse Point 検証
復元対象パスの解決から書き込み完了までの間に、シンボリックリンクやジャンクション等が外部領域へすり替えられる攻撃を防御する。approved security decisionにより、`RelativePath` はバックアップ由来のhostile inputとして扱い、文字列結合だけを信頼してはならない。

```text
[Authenticated Manifest RelativePath]
      │
      ├─ rooted / UNC / device / traversal / ADS / invalid form ? ──> reject
      │
      ▼
[Normalize RelativePath]
      │
      ▼
[Candidate = validated restore root + normalized relative path]
      │
      ▼
[Existing IPathResolver / PathValidationBarrier:
 Win32 GetFinalPathNameByHandle + canonical containment check]
      │
      ├─ outside root / reparse ambiguity / resolution failure ──> SecurityException + no replacement
      │
      ▼
[Create temporary restore file only inside validated root]
      │
      ▼
[Final pre-replacement revalidation]
      │
      ├─ changed / reparse / containment failure ──> abort + preserve existing user file
      │
      ▼
[Atomic replacement under Review → Confirm → Execute]
```

### 3.2 approved security decision Authenticated Archive / Restore Invariants

1. `GST_Manifest.json.enc` must authenticate successfully before any user-data replacement path is opened.
2. Manifest authentication is independent of `ExpectedManifestHash`; the DB hash is an additional anchor only when present.
3. Header/Integrity metadata tampering, KDF parameter tampering, Manifest tampering, or payload-context tampering must fail closed before final replacement.
4. Every payload chunk is bound to its authenticated file identity, canonical RelativePath, chunk index, plaintext length, and ManifestDigest through GCM AAD.
5. Per-file chunk indices are strictly sequential from zero. Missing, duplicate, reordered, truncated, or extra records are invalid.
6. Cross-file payload replay is rejected because the canonical file/path/ManifestDigest context is part of each chunk's AAD.
7. The restore layer independently validates containment even when the caller has already performed validation. User-provided or archive-provided path strings are never treated as sufficient proof of safety.
8. `Review → Confirm → Execute` remains the only authorization path for actual replacement; successful cryptographic verification does not authorize overwrite by itself.

---

# 4. 復元前セーブデータ一時退避 (RescueSnapshot) ＆ 起動時自己修復

## 4.1 RescueSnapshot の作成 ＆ アプリケーション層の純化
Application 層における `System.IO`（物理ファイルアクセス）の直接利用は、単体テストを不可能にし TOCTOU 脆弱性を生むため厳禁とする。退避および復旧処理は、必ず Infrastructure 層に実装された `IRescueSnapshotStorage` Port を通じて実行する。

## 4.2 RescueSnapshot の自動クリーンアップライフサイクル
一時退避データが無期限に残存してディスクを圧迫することを防ぐため、以下の自動削除ルールを適用する。
* **復元成功時 (`RestoreStatus == Completed`):**  
  復元完了から **24時間後**（または次回アプリ起動時）に RescueSnapshot フォルダを自動削除。
* **復元失敗時 (`RestoreStatus == PartialFailed / Failed`):**  
  ユーザーが手動確認を行えるよう **7日間保持** し、期限切れ後に警告通知を経て自動クリーンアップ対象とする。
* **一時退避残骸の回収 (`.creating` GC):**  
  退避途中でクラッシュした場合に生成される `{transactionId}.creating` フォルダを、起動時ヘルスチェックで安全に自動パージする。

---

# 5. Domain Layer 定義 (`GameSecurityTool.Domain.Models.SaveBackup`)

```csharp
namespace GameSecurityTool.Domain.Models.SaveBackup;

using System;

public sealed class RestoreTransaction
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public string TransactionId { get; init; } = string.Empty; // 例: "RST-20260827-A82F"
    public string OperationId { get; init; } = string.Empty;
    public Guid BackupId { get; init; }
    public Guid GameProfileId { get; init; }
    public string TargetRootPath { get; init; } = string.Empty;
    public RestoreTransactionStatus Status { get; set; } = RestoreTransactionStatus.Initialized;
    public string? FailedStep { get; set; }
    public string? ErrorMessage { get; set; }
    public DateTimeOffset StartedAt { get; init; } = DateTimeOffset.UtcNow;
    public DateTimeOffset? CompletedAt { get; set; }

    public bool IsIncomplete => Status is RestoreTransactionStatus.Initialized 
                                       or RestoreTransactionStatus.Validating 
                                       or RestoreTransactionStatus.PreRestoreBackingUp 
                                       or RestoreTransactionStatus.Restoring
                                       or RestoreTransactionStatus.PartialFailed;
}

public enum RestoreTransactionStatus
{
    Initialized = 0,
    Validating = 1,
    PreRestoreBackingUp = 2,
    Restoring = 3,
    // 4 is permanently reserved for legacy/non-durable HashVerifying.
    // Hash verification is an internal RestoreContainerAsync sub-step.
    Completed = 5,
    PartialFailed = 6,
    RolledBack = 7,
    Failed = 8
}
```

## 5.1 保持期限判定ドメインサービス (`RescueSnapshotRetentionEvaluator.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Domain.Services;

using System;
using GameSecurityTool.Domain.Models.SaveBackup;

public static class RescueSnapshotRetentionEvaluator
{
    public static readonly TimeSpan CompletedRetentionPeriod = TimeSpan.FromHours(24);
    public static readonly TimeSpan FailedRetentionPeriod = TimeSpan.FromDays(7);
    public static readonly TimeSpan OrphanCreatingGracePeriod = TimeSpan.FromHours(1);

    /// <summary>
    /// トランザクションの状態と経過時間に基づき、RescueSnapshot がパージ対象（期限切れ）かを純粋判定
    /// </summary>
    public static bool IsSnapshotExpired(RestoreTransactionStatus status, DateTimeOffset timestampUtc, DateTimeOffset nowUtc)
    {
        var elapsed = nowUtc - timestampUtc;

        return status switch
        {
            RestoreTransactionStatus.Completed or RestoreTransactionStatus.RolledBack => elapsed >= CompletedRetentionPeriod,
            RestoreTransactionStatus.PartialFailed or RestoreTransactionStatus.Failed => elapsed >= FailedRetentionPeriod,
            _ => false // 進行中または未確定状態はパージしない
        };
    }
}
```

---

# 6. Contracts Layer 定義 (`GameSecurityTool.Contracts`)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System;
using System.Threading;
using System.Threading.Tasks;

public interface IRescueSnapshotStorage
{
    /// <summary>
    /// 現在のセーブデータを復元前に一時退避します (RescueSnapshot のアトミック生成)。
    /// </summary>
    Task CreateSnapshotAsync(string transactionId, string sourceDir, CancellationToken ct = default);

    /// <summary>
    /// 復元処理が失敗した場合に、RescueSnapshot から元の状態へアトミックにロールバックします (一時ファイル経由)。
    /// </summary>
    Task RollbackAsync(string transactionId, string targetDir, CancellationToken ct = default);

    /// <summary>
    /// 指定されたトランザクションの RescueSnapshot が完全に存在するかどうか（マニフェスト検証含む）を確認します。
    /// </summary>
    Task<bool> HasSnapshotAsync(string transactionId, CancellationToken ct = default);

    /// <summary>
    /// 指定された RescueSnapshot を安全に削除します (.creating 残骸も併せて消去)。
    /// </summary>
    Task DeleteSnapshotAsync(string transactionId, CancellationToken ct = default);

    /// <summary>
    /// 退避途中でクラッシュして残存した未確定の .creating 一時ディレクトリを一括パージします。
    /// </summary>
    Task PurgeOrphanedCreatingSnapshotsAsync(TimeSpan threshold, CancellationToken ct = default);
}
```

---

# 7. Application Layer 完全実装 (`SaveRestoreCoordinator.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Application.Features.SaveBackup;

using System;
using System.Security;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Domain.Models.SaveBackup;
using GameSecurityTool.Domain.Services;
using Microsoft.Extensions.Logging;

public sealed class SaveRestoreCoordinator(
    ISaveBackupStorage backupStorage,
    ISaveBackupRepository backupRepository,
    IRescueSnapshotStorage rescueStorage,
    IPathResolver pathResolver,
    ILogSanitizer logSanitizer,
    IPowerStateService powerStateService,
    IStorageResilienceProvider storageResilience,
    ILogger<SaveRestoreCoordinator> logger)
{
    public async Task<SaveRestoreResultDto> ExecuteRestoreAsync(
        Guid backupId,
        string targetRestoreRoot,
        byte[]? password,
        IProgress<double>? progress = null,
        CancellationToken ct = default)
    {
        string operationId = $"GST-OP-{DateTimeOffset.UtcNow:yyyyMMdd}-{Guid.NewGuid():N[..8]}";
        string transactionId = $"RST-{DateTimeOffset.UtcNow:yyyyMMdd}-{Guid.NewGuid():N[..8]}";

        var snapshotDto = await backupRepository.GetSnapshotByIdAsync(backupId, ct);
        if (snapshotDto == null)
        {
            return new SaveRestoreResultDto(false, transactionId, "Validate", "復元対象のスナップショットが見つかりません。");
        }

        // 1. TOCTOU ＆ Reparse Point 境界検証
        if (!pathResolver.IsPathWithinBounds(targetRestoreRoot, targetRestoreRoot))
        {
            throw new SecurityException($"不正な復元先パスが検出されました: {logSanitizer.Sanitize(targetRestoreRoot)}");
        }

        var txDto = new RestoreTransactionDto(
            Id: Guid.NewGuid(),
            TransactionId: transactionId,
            OperationId: operationId,
            BackupId: backupId,
            GameProfileId: snapshotDto.GameProfileId,
            TargetRootPath: targetRestoreRoot,
            Status: RestoreTransactionStatus.Initialized.ToString(),
            FailedStep: null,
            ErrorMessage: null,
            StartedAt: DateTimeOffset.UtcNow,
            CompletedAt: null
        );
        await backupRepository.SaveRestoreTransactionAsync(txDto, ct);

        bool isRescueSnapshotCompleted = false;

        try
        {
            // 【電源レジリエンス】復元処理中の OS スリープ・スタンバイ移行を阻止
            await using var powerScope = await powerStateService.BeginCriticalExecutionScopeAsync("SaveRestoreTransaction", ct);

            // 【ストレージレジリエンス】スピンアップ待機 ＆ ドライブ可用性事前検証
            await storageResilience.EnsureStorageReadyAsync(targetRestoreRoot, ct);

            // 2. Pre-Restore Snapshot (現行セーブデータの一時退避)
            await backupRepository.UpdateRestoreTransactionStatusAsync(
                transactionId, RestoreTransactionStatus.PreRestoreBackingUp.ToString(), null, null, ct);

            await rescueStorage.CreateSnapshotAsync(transactionId, targetRestoreRoot, ct);
            
            // 退避完了の証明が得られた場合のみロールバック権限がアンロックされる
            isRescueSnapshotCompleted = true;

            // 3. 復元実行。Hash検証・Metadata/タイムスタンプ復元は RestoreContainerAsync 内部サブステップ。
            await backupRepository.UpdateRestoreTransactionStatusAsync(
                transactionId, RestoreTransactionStatus.Restoring.ToString(), null, null, ct);

            var restoreReq = new SaveBackupRestoreRequestDto(
                OperationId: operationId,
                TransactionId: transactionId,
                BackupId: backupId,
                Password: password,
                ContainerPath: snapshotDto.StoragePath,
                TargetRestoreRootPath: targetRestoreRoot,
                ExpectedManifestHash: snapshotDto.ManifestHash
            );

            var restoreResult = await backupStorage.RestoreContainerAsync(restoreReq, progress, ct);
            if (!restoreResult.Success)
            {
                logger.LogWarning("復元処理が失敗したため、RescueSnapshot から自動ロールバックを実行します: {Tx}", transactionId);
                await rescueStorage.RollbackAsync(transactionId, targetRestoreRoot, CancellationToken.None);

                await backupRepository.UpdateRestoreTransactionStatusAsync(
                    transactionId, RestoreTransactionStatus.PartialFailed.ToString(), restoreResult.FailedStep, restoreResult.ErrorMessage, ct);

                return new SaveRestoreResultDto(false, transactionId, restoreResult.FailedStep, restoreResult.ErrorMessage);
            }

            // 4. 正常完了
            await backupRepository.UpdateRestoreTransactionStatusAsync(
                transactionId, RestoreTransactionStatus.Completed.ToString(), null, null, ct);

            logger.LogInformation("セーブデータの安全復元が完了しました: {Path} (Tx: {Tx})", targetRestoreRoot, transactionId);
            return new SaveRestoreResultDto(true, transactionId, null, null);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "復元処理中に致命的例外が発生しました: {Tx}", transactionId);

            if (isRescueSnapshotCompleted)
            {
                logger.LogWarning("RescueSnapshot からのアトミックロールバックを実行します: {Tx}", transactionId);
                await rescueStorage.RollbackAsync(transactionId, targetRestoreRoot, CancellationToken.None);
            }
            else
            {
                logger.LogWarning("退避処理完了前に失敗したため、現行データへのロールバックは実行しません (安全側停止): {Tx}", transactionId);
            }

            await backupRepository.UpdateRestoreTransactionStatusAsync(
                transactionId, RestoreTransactionStatus.Failed.ToString(), "Exception", ex.Message, CancellationToken.None);

            return new SaveRestoreResultDto(false, transactionId, "Exception", ex.Message);
        }
    }

    /// <summary>
    /// アプリ起動時自己修復: 前回クラッシュ等で中断された復元トランザクションを自動ロールバック
    /// </summary>
    public async Task RecoverIncompleteTransactionsAsync(CancellationToken ct = default)
    {
        try
        {
            await using var powerScope = await powerStateService.BeginCriticalExecutionScopeAsync("SaveRestoreRecovery", ct);

            var incompleteList = await backupRepository.GetIncompleteRestoreTransactionsAsync(ct);
            foreach (var tx in incompleteList)
            {
                logger.LogWarning("未完了の復元トランザクションを検知しました: {Tx} (Status: {Status})。自己修復を開始します...", tx.TransactionId, tx.Status);

                bool hasSnapshot = await rescueStorage.HasSnapshotAsync(tx.TransactionId, ct);
                if (hasSnapshot)
                {
                    await rescueStorage.RollbackAsync(tx.TransactionId, tx.TargetRootPath, ct);
                    
                    await backupRepository.UpdateRestoreTransactionStatusAsync(
                        tx.TransactionId, RestoreTransactionStatus.RolledBack.ToString(), null, "起動時自己修復により RescueSnapshot から正常にロールバックされました。", ct);

                    logger.LogInformation("RescueSnapshot からの自己修復ロールバックが完了しました: {Tx}", tx.TransactionId);
                }
                else
                {
                    await backupRepository.UpdateRestoreTransactionStatusAsync(
                        tx.TransactionId, RestoreTransactionStatus.Failed.ToString(), "RescueSnapshotCreationFailed", "退避データが不完全または存在しないため修復できませんでした。", ct);
                }
            }

            await rescueStorage.PurgeOrphanedCreatingSnapshotsAsync(RescueSnapshotRetentionEvaluator.OrphanCreatingGracePeriod, ct);
            await PurgeExpiredRescueSnapshotsAsync(ct);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "起動時復元トランザクション自己修復中にエラーが発生しました。");
        }
    }

    private async Task PurgeExpiredRescueSnapshotsAsync(CancellationToken ct)
    {
        try
        {
            var nowUtc = DateTimeOffset.UtcNow;
            var purgeTargets = await backupRepository.GetExpiredTransactionsForPurgeAsync(nowUtc, ct);
            foreach (var txId in purgeTargets)
            {
                await rescueStorage.DeleteSnapshotAsync(txId, ct);
                logger.LogInformation("期限切れの RescueSnapshot を安全にパージしました: {Tx}", txId);
            }
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "RescueSnapshot の期限切れパージ処理中にエラーが発生しました。");
        }
    }
}
```

---

# 8. Infrastructure Layer 完全実装 (`FileSystemRescueSnapshotStorage.cs` - C-3 是正)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.SaveBackup;

using System;
using System.IO;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.Extensions.Logging;

public sealed class FileSystemRescueSnapshotStorage(ILogger<FileSystemRescueSnapshotStorage> logger) : IRescueSnapshotStorage
{
    private const int MaxRetries = 3;
    
    private static readonly string RescueSnapshotBaseDir = Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
        "GameSecurityTool",
        "RescueSnapshots"
    );

    public async Task CreateSnapshotAsync(string transactionId, string sourceDir, CancellationToken ct = default)
    {
        if (!Directory.Exists(sourceDir)) return;
        
        string targetRescueDir = Path.Combine(RescueSnapshotBaseDir, transactionId);
        string creatingDir = targetRescueDir + ".creating";
        
        if (Directory.Exists(creatingDir)) Directory.Delete(creatingDir, true);
        Directory.CreateDirectory(creatingDir);

        int fileCount = 0;
        foreach (string file in Directory.EnumerateFiles(sourceDir, "*.*", SearchOption.AllDirectories))
        {
            ct.ThrowIfCancellationRequested();
            string relative = Path.GetRelativePath(sourceDir, file);
            string dest = Path.Combine(creatingDir, relative);
            string? destSubDir = Path.GetDirectoryName(dest);
            if (!string.IsNullOrEmpty(destSubDir)) Directory.CreateDirectory(destSubDir);

            await CopyFileWithRetryAsync(file, dest, ct);
            fileCount++;
        }
        
        // 退避完了の証明としてマニフェストを作成
        string manifestPath = Path.Combine(creatingDir, "manifest.json");
        await File.WriteAllTextAsync(manifestPath, $"{{\"FileCount\": {fileCount}, \"Completed\": true}}", ct);

        // ディレクトリをアトミックリネームして正式に確定
        Directory.Move(creatingDir, targetRescueDir);
    }

    /// <summary>
    /// 【C-3 是正】一時ファイル書き出し ➔ アトミックリネーム (File.Move) による完全な原子性ロールバック
    /// </summary>
    public async Task RollbackAsync(string transactionId, string targetDir, CancellationToken ct = default)
    {
        string rescueDir = Path.Combine(RescueSnapshotBaseDir, transactionId);
        if (!Directory.Exists(rescueDir) || !Directory.Exists(targetDir)) return;

        foreach (string file in Directory.EnumerateFiles(rescueDir, "*.*", SearchOption.AllDirectories))
        {
            ct.ThrowIfCancellationRequested();
            
            if (Path.GetFileName(file) == "manifest.json") continue;

            string relative = Path.GetRelativePath(rescueDir, file);
            string dest = Path.Combine(targetDir, relative);
            string? destSubDir = Path.GetDirectoryName(dest);
            if (!string.IsNullOrEmpty(destSubDir)) Directory.CreateDirectory(destSubDir);

            // 一時ファイルへの完全コピー
            string tempRollbackFile = dest + ".rollback.tmp";
            try
            {
                await CopyFileWithRetryAsync(file, tempRollbackFile, ct);
                
                // コピー完了後にアトミック移動でユーザーファイルを差し替え
                File.Move(tempRollbackFile, dest, overwrite: true);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "ロールバック中のファイル書き換えに失敗しました: {Dest}", dest);
                if (File.Exists(tempRollbackFile))
                {
                    try { File.Delete(tempRollbackFile); } catch { }
                }
                throw;
            }
        }
    }

    public Task<bool> HasSnapshotAsync(string transactionId, CancellationToken ct = default)
    {
        string rescueDir = Path.Combine(RescueSnapshotBaseDir, transactionId);
        string manifestPath = Path.Combine(rescueDir, "manifest.json");
        
        return Task.FromResult(Directory.Exists(rescueDir) && File.Exists(manifestPath));
    }

    public Task DeleteSnapshotAsync(string transactionId, CancellationToken ct = default)
    {
        string rescueDir = Path.Combine(RescueSnapshotBaseDir, transactionId);
        string creatingDir = rescueDir + ".creating";

        DeleteDirectorySafe(rescueDir);
        DeleteDirectorySafe(creatingDir);

        return Task.CompletedTask;
    }

    public Task PurgeOrphanedCreatingSnapshotsAsync(TimeSpan threshold, CancellationToken ct = default)
    {
        if (!Directory.Exists(RescueSnapshotBaseDir)) return Task.CompletedTask;

        var now = DateTimeOffset.UtcNow;
        try
        {
            var dirInfo = new DirectoryInfo(RescueSnapshotBaseDir);
            foreach (var subDir in dirInfo.EnumerateDirectories("*.creating"))
            {
                ct.ThrowIfCancellationRequested();
                if (now - subDir.LastWriteTimeUtc >= threshold)
                {
                    logger.LogInformation("孤児化した一時退避ディレクトリを自動パージします: {Dir}", subDir.FullName);
                    try
                    {
                        subDir.Delete(recursive: true);
                    }
                    catch (Exception ex)
                    {
                        logger.LogWarning(ex, "一時退避ディレクトリのパージに失敗しました: {Dir}", subDir.FullName);
                    }
                }
            }
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "一時退避ディレクトリの孤児走査中にエラーが発生しました。");
        }

        return Task.CompletedTask;
    }

    private void DeleteDirectorySafe(string path)
    {
        if (Directory.Exists(path))
        {
            try
            {
                Directory.Delete(path, true);
            }
            catch (Exception ex)
            {
                logger.LogWarning(ex, "RescueSnapshot ディレクトリの削除失敗: {Dir}", path);
            }
        }
    }

    private static async Task CopyFileWithRetryAsync(string source, string dest, CancellationToken ct)
    {
        for (int i = 0; i < MaxRetries; i++)
        {
            try
            {
                await using var inStream = new FileStream(source, FileMode.Open, FileAccess.Read, FileShare.ReadWrite | FileShare.Delete, 65536, useAsync: true);
                await using var outStream = new FileStream(dest, FileMode.Create, FileAccess.Write, FileShare.None, 65536, useAsync: true);
                await inStream.CopyToAsync(outStream, ct);
                return;
            }
            catch (IOException)
            {
                if (i == MaxRetries - 1) throw;
                await Task.Delay(100 * (int)Math.Pow(2, i), ct);
            }
        }
    }
}
```

---

# 9. 単体テスト仕様 (`GST.UnitTests.SaveBackup.Restore`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.SaveBackup.Restore;

using System;
using System.IO;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Application.Features.SaveBackup;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Domain.Models.SaveBackup;
using GameSecurityTool.Domain.Services;
using Microsoft.Extensions.Logging.Abstractions;
using Moq;
using Xunit;

public class RestoreTransactionTests
{
    [Fact]
    public void RescueSnapshotRetentionEvaluator_EvaluatesCorrectly()
    {
        var now = DateTimeOffset.UtcNow;

        Assert.False(RescueSnapshotRetentionEvaluator.IsSnapshotExpired(RestoreTransactionStatus.Completed, now.AddHours(-23), now));
        Assert.True(RescueSnapshotRetentionEvaluator.IsSnapshotExpired(RestoreTransactionStatus.Completed, now.AddHours(-25), now));

        Assert.False(RescueSnapshotRetentionEvaluator.IsSnapshotExpired(RestoreTransactionStatus.Failed, now.AddDays(-6), now));
        Assert.True(RescueSnapshotRetentionEvaluator.IsSnapshotExpired(RestoreTransactionStatus.Failed, now.AddDays(-8), now));
    }

    [Fact]
    public async Task ExecuteRestoreAsync_PreRestoreFails_DoesNotRollback()
    {
        var mockStorage = new Mock<ISaveBackupStorage>();
        var mockRepo = new Mock<ISaveBackupRepository>();
        var mockRescue = new Mock<IRescueSnapshotStorage>();
        var mockPath = new Mock<IPathResolver>();
        var mockSanitizer = new Mock<ILogSanitizer>();

        var snapshotDto = new SaveBackupSnapshotDto(Guid.NewGuid(), Guid.NewGuid(), "OP", "TX", "1.0", DateTimeOffset.UtcNow, 1, 100, "HASH", "path", "Completed", "Active", false, [], null);
        mockRepo.Setup(r => r.GetSnapshotByIdAsync(It.IsAny<Guid>(), default)).ReturnsAsync(snapshotDto);
        mockPath.Setup(p => p.IsPathWithinBounds(It.IsAny<string>(), It.IsAny<string>())).Returns(true);

        mockRescue.Setup(r => r.CreateSnapshotAsync(It.IsAny<string>(), It.IsAny<string>(), default))
                  .ThrowsAsync(new UnauthorizedAccessException("Access Denied"));

        var coordinator = new SaveRestoreCoordinator(
            mockStorage.Object, mockRepo.Object, mockRescue.Object, mockPath.Object, mockSanitizer.Object, NullLogger<SaveRestoreCoordinator>.Instance);

        var result = await coordinator.ExecuteRestoreAsync(Guid.NewGuid(), "C:\\Saves", null);

        Assert.False(result.Success);
        Assert.Equal("Access Denied", result.ErrorMessage);
        mockRescue.Verify(r => r.RollbackAsync(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<CancellationToken>()), Times.Never);
    }
}
```

---

