# GameSecurityTool Save Backup Retention & Maintenance Specification

**Document ID:** GST-FEAT-SAVE-MAINT-006  
**Version:** 4.1 (Application Coordinator Wiring, Exception Safety & GC Grace Period Hardened)
**Status:** Approved Feature Specification  
**Target Layer:** Domain Layer / Application Layer / Infrastructure Layer  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Maintenance Philosophy & Non-Destructive Policy

セーブデータのメンテナンスおよび世代整理は、「古いデータを単に削除する」処理ではなく、**「復旧能力を最大限維持しながらストレージ容量を最適化する安全制御」** として実装する。

### 不可侵ルール（Non-Destructive Rules）
1. **最新世代の絶対保護 (Latest Snapshot Protection)**:
   いかなる保持ポリシー（容量上限や保持日数）に達した場合でも、最も新しい正常スナップショット（最新世代）は絶対に自動削除してはならない。
2. **ピン留めスナップショットの絶対保護 (Pinned Snapshot Protection)**:
   ユーザーによって `IsPinned == true` に設定されたスナップショットは、**保持世代数や容量上限による自動削除の対象から無条件除外** される。ユーザーが明示的にピン留めを解除しない限り永久に保持する。
3. **差分参照元スナップショットの絶対保護 (Referenced Snapshot Protection)**:
   後続の差分スナップショットから実体として参照されている過去のスナップショットは、自動削除候補から除外し、保持容量の計算に正しく算入して実際のディスク消費量との乖離を防ぐ。
4. **アクティブ復元候補の保護**:
   UI 上でユーザーが復元プレビュー中、または復元トランザクションが進行中のスナップショットは削除候補から除外する。
5. **整合性未検証スナップショットの保護**:
   破損が疑われるスナップショットであっても、ユーザーの同意なしに勝手に消去せず、「Corrupted（破損）」としてマークして隔離・通知する。
6. **ランサムウェア・エントロピー急変時の自動パージ緊急凍結 (Snapshot Freeze)**:
   バックアップ対象ファイルのシャノン・エントロピーが急激に跳ね上がった場合（暗号化マルウェアの兆候）、自動世代整理（パージ）および新規スナップショット作成を即時緊急凍結し、過去の正常なバックアップが上書き・消去されるのを防止する。

---

# 2. 保持ポリシーモデル (Retention Policy Model)

保持ポリシーは Game Profile 単位でカスタマイズ可能とし、未設定時は Global 設定を継承する。

```csharp
namespace GameSecurityTool.Domain.Models.SaveBackup;

public sealed record SaveBackupRetentionPolicy(
    int MaxGenerations,           // 保持世代数 (デフォルト: 3世代 / カスタム指定可能)
    long MaxStorageSizeBytes,     // 最大許容ストレージ容量 (デフォルト: 5GB)
    int KeepDays,                 // 保持日数 (デフォルト: 30日)
    long MinimumFreeSpaceBytes    // ドライブ最低空き容量 (デフォルト: 2GB)
);
```

---

# 3. 世代評価・削除候補算出エンジン (`Domain.Services.RetentionEvaluator`)

リポジトリや DB クエリ内で複雑な削除判定を行うことを禁止し、評価は `Domain.Services.RetentionEvaluator`（Pure C#）へ集約してテスト可能にする。
Application 層はリポジトリから構築した参照グラフ（依存IDの集合 `IReadOnlySet<Guid>`）を渡し、ドメイン側で正確なストレージ会計を行う。

```csharp
#nullable enable

namespace GameSecurityTool.Domain.Services;

using System;
using System.Collections.Generic;
using System.Linq;
using GameSecurityTool.Domain.Models.SaveBackup;

public static class RetentionEvaluator
{
    public static IReadOnlyList<SaveBackupSnapshot> EvaluateCandidatesForRemoval(
        IReadOnlyList<SaveBackupSnapshot> snapshots,
        SaveBackupRetentionPolicy policy,
        IReadOnlySet<Guid> referencedSnapshotIds,
        DateTimeOffset nowUtc)
    {
        if (snapshots.Count <= 1) return [];

        var candidates = new List<SaveBackupSnapshot>();
        var sorted = snapshots.OrderByDescending(s => s.CreatedAt).ToList();

        long currentTotalSize = 0;
        int activeCount = 0;

        for (int i = 0; i < sorted.Count; i++)
        {
            var snapshot = sorted[i];

            // 1. 最新世代は絶対に保護
            if (i == 0)
            {
                currentTotalSize += snapshot.TotalSizeBytes;
                activeCount++;
                continue;
            }

            // 2. ピン留め (IsPinned) は永久保持・削除不可
            if (snapshot.IsPinned || snapshot.Retention == RetentionStatus.Protected)
            {
                currentTotalSize += snapshot.TotalSizeBytes;
                continue;
            }

            // 3. 他スナップショットから差分参照元として依存されている場合は削除不可
            if (referencedSnapshotIds.Contains(snapshot.BackupId) || snapshot.IsReferencedAsSource)
            {
                currentTotalSize += snapshot.TotalSizeBytes;
                continue;
            }

            bool shouldRemove = false;

            // 4. 保持世代数上限超過
            if (activeCount >= policy.MaxGenerations)
            {
                shouldRemove = true;
            }
            // 5. 保持日数上限超過
            else if ((nowUtc - snapshot.CreatedAt).TotalDays > policy.KeepDays)
            {
                shouldRemove = true;
            }
            // 6. 容量上限超過
            else if (currentTotalSize + snapshot.TotalSizeBytes > policy.MaxStorageSizeBytes)
            {
                shouldRemove = true;
            }

            if (shouldRemove)
            {
                candidates.Add(snapshot);
            }
            else
            {
                currentTotalSize += snapshot.TotalSizeBytes;
                activeCount++;
            }
        }

        return candidates;
    }
}
```

---

# 4. Application Layer 完全実装 (`SaveBackupMaintenanceCoordinator.cs` - M-H3 是正)

Application 層は、リポジトリからスナップショット一覧および差分参照依存グラフを取得し、Domain の `RetentionEvaluator` を呼び出して安全な二段階整理と物理 GC を統制する。

```csharp
#nullable enable

namespace GameSecurityTool.Application.Features.SaveBackup;

using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Common;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Domain.Models.SaveBackup;
using GameSecurityTool.Domain.Services;
using Microsoft.Extensions.Logging;

public sealed class SaveBackupMaintenanceCoordinator(
    ISaveBackupRepository repository,
    ISaveBackupStorage storage,
    ITamperEvidentAuditLogger auditLogger,
    ILogger<SaveBackupMaintenanceCoordinator> logger)
{
    private static readonly TimeSpan GcGracePeriod = TimeSpan.FromHours(1);

    /// <summary>
    /// 指定ゲームの世代管理ポリシーを安全に適用 (参照グラフ構築 ＆ 例外安全性保証)
    /// </summary>
    public async Task ApplyRetentionPolicyAsync(
        Guid gameProfileId,
        SaveBackupRetentionPolicy policy,
        CancellationToken ct = default)
    {
        try
        {
            logger.LogInformation("世代整理メンテナンスタスクを開始します: {ProfileId}", gameProfileId);

            var snapshotDtos = await repository.GetSnapshotsByGameProfileIdAsync(gameProfileId, ct);
            if (snapshotDtos.Count <= 1) return;

            // 1. 【品質改善記録】N+1 クエリ解消: 差分参照元スナップショットIDをバッチ一括取得
            var referencedIds = await repository.GetReferencedSourceBackupIdsAsync(ct);

            // DTO -> Domain エンティティ変換
            var domainSnapshots = snapshotDtos.Select(s => new SaveBackupSnapshot
            {
                BackupId = s.BackupId,
                GameProfileId = s.GameProfileId,
                OperationId = s.OperationId,
                TransactionId = s.TransactionId,
                StorageContainerRelativePath = s.StoragePath,
                CreatedAt = s.CreatedAt,
                FileCount = s.FileCount,
                TotalSizeBytes = s.TotalSizeBytes,
                ManifestHash = s.ManifestHash,
                IsPinned = s.IsPinned,
                IsReferencedAsSource = referencedIds.Contains(s.BackupId),
                Retention = s.IsPinned ? RetentionStatus.Protected : RetentionStatus.Active
            }).ToList();

            // 2. Domain 判定エンジンによる安全な削除候補選定
            var removalCandidates = RetentionEvaluator.EvaluateCandidatesForRemoval(
                domainSnapshots, policy, referencedIds, DateTimeOffset.UtcNow);

            // 3. 削除実行 (例外安全 try/catch によりワーカーのクラッシュを防止)
            foreach (var candidate in removalCandidates)
            {
                try
                {
                    logger.LogInformation("古い世代のスナップショットを整理中: {Id} (CreatedAt: {Created})", candidate.BackupId, candidate.CreatedAt);
                    
                    await repository.DeleteSnapshotAsync(candidate.BackupId, ct);

                    string opId = $"GST-MAINT-{DateTimeOffset.UtcNow:yyyyMMdd}-{Guid.NewGuid():N[..8]}";
                    await auditLogger.AppendLogAsync(
                        opId, AuditEventType.SaveBackupDeleted, "GST.Maintenance", candidate.BackupId.ToString(), "Success", ct);
                }
                catch (InvalidOperationException ex)
                {
                    logger.LogWarning(ex, "参照依存性ガードによりスナップショット {Id} の削除を安全にスキップしました。", candidate.BackupId);
                }
                catch (Exception ex)
                {
                    logger.LogError(ex, "スナップショット {Id} の削除処理中にエラーが発生しました。", candidate.BackupId);
                }
            }
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "世代管理メンテナンスの全体実行中に予期せぬエラーが発生しました: {ProfileId}", gameProfileId);
        }
    }

    /// <summary>
    /// 参照カウントが 0 になった孤立物理コンテナ (.zip) を安全に回収する GC タスク
    /// </summary>
    public async Task ExecutePhysicalContainerGarbageCollectionAsync(
        string storageDirectory,
        IReadOnlyList<Guid> activeGameProfileIds,
        CancellationToken ct = default)
    {
        if (!Directory.Exists(storageDirectory)) return;

        try
        {
            logger.LogInformation("バックアップコンテナの物理 GC を開始します: {Dir}", storageDirectory);

            // 1. 全有効スナップショットの StoragePath を集計
            var validContainerPaths = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
            foreach (var profileId in activeGameProfileIds)
            {
                var snapshots = await repository.GetSnapshotsByGameProfileIdAsync(profileId, ct);
                foreach (var s in snapshots)
                {
                    if (!string.IsNullOrEmpty(s.StoragePath))
                    {
                        validContainerPaths.Add(Path.GetFullPath(s.StoragePath));
                    }
                }
            }

            // 2. 物理ディスク上の .zip コンテナ走査
            var now = DateTimeOffset.UtcNow;
            var dirInfo = new DirectoryInfo(storageDirectory);

            foreach (var file in dirInfo.EnumerateFiles("Backup_*.zip", SearchOption.TopDirectoryOnly))
            {
                ct.ThrowIfCancellationRequested();
                string fullPath = Path.GetFullPath(file.FullName);

                // 【TOCTOU ガード】作成後 1 時間以内のファイルは無条件に除外 (Grace Period)
                if (now - file.LastWriteTimeUtc < GcGracePeriod)
                {
                    continue;
                }

                // DB 上に存在しない孤立コンテナを安全消去
                if (!validContainerPaths.Contains(fullPath))
                {
                    logger.LogInformation("孤立した古い物理コンテナを回収・削除します: {Path}", fullPath);
                    await storage.DeleteContainerAsync(fullPath, ct);
                }
            }
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "物理コンテナ GC 処理中にエラーが発生しました。");
        }
    }
}
```

---

# 5. RescueSnapshot 容量監視の統合

`Storage Safety Guard`（2GB 閾値等の事前検証）において、一時退避データ（RescueSnapshot）の肥大化がバックアップ全体をブロックするリスクを防ぐため、容量管理を連携させる。

* **RescueSnapshot 合算監視:** 保存先ドライブの空き容量計算時、`RescueSnapshots` ディレクトリ内の既存データ総量を合算し、RescueSnapshot が原因で空き容量が逼迫している場合は「直近の古い退避データの即時パージ」を試行してからバックアップを継続する。

---

# 6. 保存先ストレージ引越しパイプライン (Storage Location Migration)

ユーザーがバックアップ保存先フォルダ（`CustomBackupLocation`）を別ドライブ（SSD ➔ 大容量 HDD 等）に変更した際、既存の全バックアップコンテナを安全に移送するアトミックトランザクション。

```text
1. [新ドライブの空き容量検証] (Storage Safety Guard: 移行データ総量 + 2GB)
                            │ (空き容量 OK)
                            ▼
2. [物理コンテナ一括ストリーミング移送]
   - 旧フォルダ ➔ 新フォルダへ ZIP を安全コピー
                            │
                            ▼
3. [SHA256 完全性照合]
   - 移動先の全 ZIP ハッシュが移行前と完全に一致するか検証
                            │ (一致)
                            ▼
4. [DB StoragePath 一括アトミック更新] (IDbWriteQueue 直列コミット)
                            │
                            ▼
5. [旧保存先ファイルの安全消去] ➔ 移行完了
```

---

# 7. 単体テスト仕様 (`GST.UnitTests.SaveBackup.Maintenance`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.SaveBackup.Maintenance;

using System;
using System.Collections.Generic;
using GameSecurityTool.Domain.Models.SaveBackup;
using GameSecurityTool.Domain.Services;
using Xunit;

public class RetentionEvaluatorTests
{
    [Fact]
    public void EvaluateCandidatesForRemoval_ProtectsReferencedAndPinnedSnapshots()
    {
        var now = DateTimeOffset.UtcNow;
        var policy = new SaveBackupRetentionPolicy(MaxGenerations: 2, MaxStorageSizeBytes: 10000, KeepDays: 30, MinimumFreeSpaceBytes: 2000);

        var snapLatest = new SaveBackupSnapshot { BackupId = Guid.NewGuid(), CreatedAt = now, TotalSizeBytes = 1000 };
        var snapPinned = new SaveBackupSnapshot { BackupId = Guid.NewGuid(), CreatedAt = now.AddDays(-5), TotalSizeBytes = 1000, IsPinned = true };
        var snapReferenced = new SaveBackupSnapshot { BackupId = Guid.NewGuid(), CreatedAt = now.AddDays(-10), TotalSizeBytes = 1000, IsReferencedAsSource = true };
        var snapOld = new SaveBackupSnapshot { BackupId = Guid.NewGuid(), CreatedAt = now.AddDays(-15), TotalSizeBytes = 1000 };

        var referencedSet = new HashSet<Guid> { snapReferenced.BackupId };

        var candidates = RetentionEvaluator.EvaluateCandidatesForRemoval(
            [snapLatest, snapPinned, snapReferenced, snapOld],
            policy,
            referencedSet,
            now
        );

        // 削除候補は snapOld のみであること
        Assert.Single(candidates);
        Assert.Equal(snapOld.BackupId, candidates[0].BackupId);
    }
}
```

---

