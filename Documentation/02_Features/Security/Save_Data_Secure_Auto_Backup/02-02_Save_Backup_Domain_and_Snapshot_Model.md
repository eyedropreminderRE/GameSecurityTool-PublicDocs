# GameSecurityTool Save Backup Domain & Snapshot Model Specification

**Document ID:** GST-FEAT-SAVE-MODEL-002  
**Version:** 3.2 (Tagging, Pinning & Differential Reference Model Hardened)
**Status:** Approved Domain Specification  
**Target Layer:** Domain Layer (`GameSecurityTool.Domain`)  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Domain Layer Responsibility & Boundary

Domain 層は、セーブデータバックアップに関する純粋なビジネスルール、不変スナップショットモデル、差分判定ロジックをカプセル化する。

### 厳格な制約（Clean Architecture Guardrails）
- `System.IO` (File, Directory, FileStream 等) の直接呼び出しを禁止。
- SQLite / EF Core / DbContext への依存を禁止。
- Windows API, DPAPI, Win32 ネイティブ API への依存を禁止。
- UI / WPF コンポーネントへの依存を禁止。
- すべてのロジックは、メモリ上の値オブジェクト（POCO / Record）を受け取って決定論的にテスト可能でなければならない。

---

# 2. Snapshot & Manifest Domain Model

セーブデータのバックアップは、単なるフォルダコピーではなく、**バージョン管理された不変スナップショット（Versioned Snapshot）** として表現する。

## 2.1 SaveBackupSnapshot (スナップショットルートエンティティ)

```csharp
namespace GameSecurityTool.Domain.Models.SaveBackup;

using System;
using System.Collections.Generic;

public sealed class SaveBackupSnapshot
{
    public Guid BackupId { get; init; } = Guid.NewGuid();
    public Guid GameProfileId { get; init; }
    public string OperationId { get; init; } = string.Empty;
    public string TransactionId { get; init; } = string.Empty; // 原子処理追跡ID
    public string SourceRootRelativePath { get; init; } = string.Empty;
    public string StorageContainerRelativePath { get; init; } = string.Empty;
    public string SnapshotVersion { get; init; } = "1.0";
    public DateTimeOffset CreatedAt { get; init; } = DateTimeOffset.UtcNow;
    public int FileCount { get; init; }
    public long TotalSizeBytes { get; init; }
    public string ManifestHash { get; init; } = string.Empty;
    public SnapshotStatus Status { get; set; } = SnapshotStatus.Creating;
    public RetentionStatus Retention { get; set; } = RetentionStatus.Active;

    // 【タイムトラベル ＆ ピン留め保護プロパティ】
    public bool IsPinned { get; set; }                           // true = 自動世代パージから永久保護
    public bool IsReferencedAsSource { get; set; }               // true = 他の差分バックアップから実体として参照されており削除不可 (H3是正)
    public IReadOnlyList<string> Tags { get; set; } = [];        // プリセット / 自由入力タグリスト
    public string? UserNote { get; set; }                        // ユーザー任意メモ (MOD適用状態等)

    public IReadOnlyList<SaveFileManifestEntry> FileEntries { get; init; } = [];
}
```

## 2.2 プリセットタグ定義 (Preset Tag Constants)

```csharp
namespace GameSecurityTool.Domain.Models.SaveBackup;

public static class PresetBackupTags
{
    public const string BeforeBoss = "PRESET_BEFORE_BOSS";       // ⚔️ ボス戦直前
    public const string StoryBranch = "PRESET_STORY_BRANCH";     // 🔀 ストーリー分岐点
    public const string BeforeModding = "PRESET_BEFORE_MODDING"; // 🛠️ MOD導入前
    public const string PostGame = "PRESET_POST_GAME";           // 🏆 クリア後 / 周回直前
    public const string Testing = "PRESET_TESTING";              // 🧪 検証・バグ回避用
}
```

## 2.3 SaveFileManifestEntry (個別ファイルマニフェスト)

```csharp
namespace GameSecurityTool.Domain.Models.SaveBackup;

using System;

public sealed record SaveFileManifestEntry(
    string RelativePath,
    long SizeBytes,
    DateTimeOffset LastWriteTimeUtc,
    string? FileId,
    string SHA256,
    Guid SourceContainerBackupId // 当該ファイルのPayload実体を保持するコンテナのBackupId (差分参照用)
);
```

## 2.4 状態列挙型 (Domain Enums)

```csharp
namespace GameSecurityTool.Domain.Models.SaveBackup;

public enum SnapshotStatus
{
    Creating = 0,
    Completed = 1,
    VerificationFailed = 2,
    Corrupted = 3,
    Deleted = 4
}

public enum RetentionStatus
{
    Active = 0,
    Protected = 1,              // ピン留め保護状態 (永久保持)
    CandidateForRemoval = 2,
    Removed = 3
}

public enum BackupMode
{
    Standard = 0,
    SSDLowImpact = 1,
    MaximumProtection = 2
}
```

---

# 3. 差分検出アルゴリズム (Change Detection Engine)

SSD への過度な I/O およびハッシュ計算負荷を排除するため、多段階の差分評価パイプラインを適用する。

```text
[現在のセーブデータ群] vs [前回スナップショットマニフェスト]
                        │
                        ▼
         ┌──────────────────────────────┐
         │ Step 1: Fast Metadata Check  │
         │ - File Count 比較            │
         │ - File Size 比較             │
         │ - LastWriteTimeUtc (秒単位)  │
         └──────────────┬───────────────┘
                        │
         ┌──────────────┴──────────────┐
     (全一致)                      (差異検出)
         │                             │
         ▼                             ▼
  [差分なし (Skip)]            ┌──────────────────────────────┐
  - 書き込みゼロ               │ Step 2: Selective Hash Check │
  - スナップショット生成中止   │ - 変更候補ファイルのみ       │
                               │   SHA256 ハッシュを算出      │
                               └──────────────┬───────────────┘
                                              │
                               ┌──────────────┴──────────────┐
                           (Hash一致)                    (Hash不一致)
                               │                             │
                               ▼                             ▼
                        [実質無変更 (Skip)]           [真の変更あり]
                        - 重複Snapshot防止            - 変更ファイルのみ保存
```

---

# 4. Manifest Hash 計算仕様 (秒精度正規化)

マニフェスト自体の改ざん検知および監査チェーン連携のため、パス区切り文字（`\` と `/`）をスラッシュへ正規化し、タイムスタンプを **秒精度（`ToUnixTimeSeconds()`）** に統一した上で決定論的なハッシュ（`ManifestHash`）を算出する。

```csharp
namespace GameSecurityTool.Domain.Services;

using System;
using System.Collections.Generic;
using System.Linq;
using System.Security.Cryptography;
using System.Text;
using GameSecurityTool.Domain.Models.SaveBackup;

public static class ManifestHashCalculator
{
    public static string ComputeManifestHash(IEnumerable<SaveFileManifestEntry> entries)
    {
        var sortedEntries = entries
            .Select(e => e with { RelativePath = NormalizePath(e.RelativePath) })
            .OrderBy(e => e.RelativePath, StringComparer.OrdinalIgnoreCase);

        var sb = new StringBuilder();
        foreach (var entry in sortedEntries)
        {
            sb.Append($"{entry.RelativePath}|{entry.SizeBytes}|{entry.LastWriteTimeUtc.ToUnixTimeSeconds()}|{entry.SHA256}|{entry.SourceContainerBackupId:N};");
        }

        byte[] hash = SHA256.HashData(Encoding.UTF8.GetBytes(sb.ToString()));
        return Convert.ToHexString(hash);
    }

    public static string NormalizePath(string path)
    {
        return path.Replace('\\', '/').Trim().ToLowerInvariant();
    }
}
```

---

# 5. セーブデータ肥大化検知ルール (Save Bloat Heuristics)

Bethesda 系 RPG（Skyrim, Fallout）等で発生するスクリプト残留バグ（セーブ肥大化）を早期検知するためのドメイン判定ルール。

```csharp
namespace GameSecurityTool.Domain.Services;

using System;

public static class SaveBloatEvaluator
{
    private const double GrowthThresholdRatio = 1.5; // +50% 以上の急激な増加で警告
    private const long MinimumAlertSizeBytes = 50 * 1024 * 1024; // 50MB 以上

    public static bool IsAbnormalGrowthDetected(long previousSizeBytes, long currentSizeBytes)
    {
        if (previousSizeBytes <= 0) return false;
        if (currentSizeBytes < MinimumAlertSizeBytes) return false;

        double ratio = (double)currentSizeBytes / previousSizeBytes;
        return ratio >= GrowthThresholdRatio;
    }
}
```

---

# 6. 単体テスト仕様 (`GST.UnitTests.Domain.SaveBackup`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.Domain.SaveBackup;

using System;
using System.Collections.Generic;
using GameSecurityTool.Domain.Models.SaveBackup;
using GameSecurityTool.Domain.Services;
using Xunit;

public class SaveModelTests
{
    [Fact]
    public void SaveBloatEvaluator_WhenSizeJumpsByFiftyPercent_ReturnsTrue()
    {
        long previousSize = 60 * 1024 * 1024; // 60 MB
        long currentSize = 100 * 1024 * 1024; // 100 MB (+66%)

        bool isBloated = SaveBloatEvaluator.IsAbnormalGrowthDetected(previousSize, currentSize);
        Assert.True(isBloated);
    }

    [Fact]
    public void Snapshot_WithTagsAndPinningAndReference_InitializesCorrectly()
    {
        var snapshot = new SaveBackupSnapshot
        {
            GameProfileId = Guid.NewGuid(),
            IsPinned = true,
            IsReferencedAsSource = true, // H3 是正プロパティ検証
            Tags = [PresetBackupTags.BeforeBoss, "オダ戦前"],
            UserNote = "難易度最高設定での挑戦前"
        };

        Assert.True(snapshot.IsPinned);
        Assert.True(snapshot.IsReferencedAsSource);
        Assert.Contains(PresetBackupTags.BeforeBoss, snapshot.Tags);
        Assert.Equal("難易度最高設定での挑戦前", snapshot.UserNote);
    }
}
```

---

End of Document
```

---
