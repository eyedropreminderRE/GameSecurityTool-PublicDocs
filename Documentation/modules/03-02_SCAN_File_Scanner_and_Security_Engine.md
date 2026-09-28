# 03-02: SCAN - File Scanner & Security Engine Specification

**Document ID:** GST-MOD-SCAN-002  
**Version:** 4.2
**Status:** Approved Module Specification  
**Target Projects:**
- `GameSecurityTool.Domain`
- `GameSecurityTool.Contracts`
- `GameSecurityTool.Application`
- `GameSecurityTool.Infrastructure`

> **Lifecycle note:** This module specification describes approved design scope. Its presence does not by itself indicate that the module is implemented, Windows-verified, or released.

---

# 1. モジュール概要 ＆ 責任境界

本モジュールは、ゲームフォルダおよび関連ファイルの**高速並列スキャン、アクセス不能フォルダ安全スキップ、差分検出、セキュリティリスク評価調停、PE ヘッダー（`MZ` + `PE\0\0`）検証、MOD マネージャー（VFS）共存探索、Anti-Wiper Canary 除外制御、および単一ファイルクイック検査**を担当する。

### Clean 5-Layer レイヤー境界:
- **Domain Layer (`GST.Domain`):** `SecurityEngineEvaluator` による不変な純粋判定ロジック、`PathValidationBarrier` の境界照合ルール。物理 I/O や Win32 API、および Contracts への依存を完全排除（Pure C#）。
- **Contracts Layer (`GST.Contracts`):** `IScanEngine`, `IRiskAssessmentService`, `IPathResolver`, `IDroppedArtifactTracker`, `IRecentArtifactAuditor`, `ICanaryTrapManager`, `IDbWriteQueue` 等の Port インターフェースおよび不変 DTO 群。
- **Application Layer (`GST.Application`):** `RiskAssessmentService`（Domain 判定結果から Contracts DTO への変換調停）、スキャンワークフロー、差分比較（Snapshot Diff）調停、Explanation 生成連携。
- **Infrastructure Layer (`GST.Infrastructure`):** `Parallel.ForEachAsync` 並列スキャナー（`IRiskAssessmentService` Port 経由でリスク評価を受領）、`IDbWriteQueue` 直列化バルクコミット、PE ヘッダー物理検証、Win32 パス解決。

---

# 2. 機能要件仕様 (Functional Requirements)

## 2.1 `FN-SCAN-01`: ゲームフォルダ構造登録 ＆ Reparse Point バリア
* **目的:** ジャンクションやシンボリックリンクによる循環参照および監視領域外へのパストラバーサルを遮断する。
* **処理仕様:**
  1. すべてのパス操作直前に `IPathResolver` 経由で Win32 `GetFinalPathNameByHandle` を呼び出し、実パス（Target Path）を強制解像。
  2. 登録ゲームフォルダのルート配下でないパスへのアクセスを即時検知・拒否。

## 2.2 `FN-SCAN-02`: 高速並列スマートスキャン (`FastFileSystemScanner`)
* **例外安全仕様 (アクセス拒否クラッシュ排除):**
  - .NET `EnumerationOptions`（`IgnoreInaccessible = true`, `AttributesToSkip = ReparsePoint`）を使用し、権限のないシステムフォルダや循環リンクに当たってもスキャナーを例外停止させずに安全スキップする。
* **処理仕様:**
  1. `Parallel.ForEachAsync` により並列ハッシュ計算を実行。
  2. 前回のスキャン記録と `FileSize` および `LastWriteTimeUtc` を事前照合し、無変更かつ有効な SHA256 が存在するファイルのハッシュ再計算をスキップ（Fast Metadata Check）。
  3. リスク評価は `IRiskAssessmentService` Port を経由して実行し、Domain 内部型への直接依存を排除。
  4. スキャン結果をパス順にソートし、**`IDbWriteQueue` 経由で 1 回のトランザクションとして直列一括保存（Bulk Commit）**。

## 2.3 `FN-SCAN-03`: セキュリティ判定調停 (`IRiskAssessmentService` 連携)
* デジタル署名、ハッシュ履歴、配置パス、既知のハイジャック警戒名等を複合評価し、判定理由（Reason List）を可視化した不変な DTO（`FileRiskResultDto`）を出力する。

## 2.4 `FN-SCAN-04`: アセット Mod コンテンツ種別不一致検知 (Asset Content Type Mismatch Detection (`ThreatReasonCodes.AssetTypeMismatch` is Contracts SSOT))
* **目的:** テクスチャや音声などのアセット専用 MOD フォルダに偽装して埋め込まれた悪意ある実行バイナリやスクリプトを即時検知する。
* **処理仕様:**
  1. ゲームフォルダ内のアセットディレクトリ（`sound/`, `sounds/`, `audio/`, `music/`, `texture/`, `textures/`, `graphics/`, `images/` 等）を特定。
  2. 当該フォルダ内に、実行バイナリ（`.exe`, `.dll`, `.sys`）またはスクリプト（`.bat`, `.cmd`, `.ps1`, `.vbs`, `.js`, `.hta`）が存在する場合、コンテンツ種別不一致（`ThreatReasonCode.AssetTypeMismatch`）として警告を発行し、起動前確認を要求する。

## 2.5 `FN-SCAN-05`: スクリプト LotL 走査 (Script Living-off-the-Land Scanning)
* **目的:** 配布物内のテキスト設定ファイルやバッチファイルを利用した間接コマンド実行攻撃を検出する。
* **処理仕様:**
  1. `.ini`, `.cfg`, `.bat`, `.cmd`, `.ps1` などの設定・スクリプトファイルを走査。
  2. `powershell -enc`, `curl`, `certutil`, `bitsadmin`, `mshta`, `cscript`, `reg add` 等の Living-off-the-Land (LotL) コマンドパターンが含まれている場合、不審な間接実行（`ThreatReasonCode.ScriptLotlCommandDetected`）として検出。

## 2.6 `FN-SCAN-06`: ドロップファイル自動追跡 (`DroppedArtifactTracker`)
* **目的:** ゲーム実行中にゲーム外（`%APPDATA%`, `%TEMP%`, スタートアップ等）へ書き出された不審バイナリを追跡・特定する。
* **厳格な PE ヘッダー検証**: 先頭 `MZ` (0x4D, 0x5A) に加え、オフセット `0x3C` の `e_lfanew` をシークして `PE\0\0` (0x50, 0x45, 0x00, 0x00) シグネチャを物理検証。正規の Windows 実行バイナリではないファイルは検査対象外とする。

## 2.7 `FN-SCAN-07`: パッチ適用時リスク急変 (Update Anomaly) 検知
* **目的:** ゲーム本体や MOD の更新によって、安全だった状態から急激に悪性兆候が混入した事象を検知する。
* **処理仕様:**
  1. 更新前のスキャンハッシュスナップショットと更新後のスキャン結果を比較。
  2. 差分として新たに追加されたファイル群の中に「未署名外部通信バイナリ」や「既知の警戒 DLL 名」「新規スクリプト実行コード」が存在する場合、リスク急変（`UpdateAnomaly`）と判定し、ゲーム起動を一時停止して差分確認ダイアログを表示する。

## 2.8 `FN-SCAN-08`: 公式サポートドメイン厳格照合 (Official Domain Strict Matching)
* **目的:** ゲーム関連 Web リンクやサポートリンクにおいて、公式ドメインを偽装する攻撃を遮断する。
* **処理仕様:**
  1. 公式ドメインリスト（ホワイトリスト）との照合において、Punycode 偽装（キリル文字等のホモグラフ攻撃）やタイポスクワッティングを検知・ブロックする。
  2. 不審なドメインへのアクセス試行は安全に遮断し、ユーザーに警告を表示する。

## 2.9 `FN-SCAN-09`: 【確定】新規ゲーム配置受動監視 ＆ Debounce 静止待機 (`NewGamePlacementWatcher`)
* **目的:** ゲーム保管親フォルダへの新規ゲーム展開・配置を捕捉し、無防備な初回起動を未然に防止する。
* **処理仕様:**
  1. ユーザー指定の監視対象ディレクトリを `FileSystemWatcher` で監視（完全オプトイン）。
  2. フォルダまたは実行ファイル生成検知後、展開完了を待つため 3 秒間の静止（Debounce）を確認してからゲーム候補（`DiscoveredGameCandidateDto`）を通知。
  3. 他ゲームが全画面実行中の場合は通知をサイレント化（バッジ点灯のみ）。

## 2.10 `FN-SCAN-10`: 【確定】共連れドロップ自動監査 (`RecentArtifactAuditor`)
* **目的:** インストーラー実行時にゲームフォルダ外（`%APPDATA%`, `%TEMP%`, スタートアップ等）へ仕込まれるバックドアやマイナーを検出する。
* **処理仕様:**
  1. 新規ゲーム配置検知時、直近 10 分間にゲーム外の重要システム領域で新規作成または更新された実行ファイル（`.exe`, `.dll`, `.bat`, `.ps1`）を走査。
  2. 不審キーワードやシグナルを検出し、赤色警告バナーで提示してワンクリック暗号化隔離を支援。

## 2.11 `FN-SCAN-11`: 【確定】署名盲信排除 ＆ ウォレット拡張機能 ID 静的走査 (Behavior Over Signature)
* **目的:** 盗難・不正取得されたコード署名を持つインフォスティーラーの活動シグナルを看破する。
* **処理仕様:**
  1. デジタル署名が有効であっても、Chrome / Edge 等の拡張機能パスや主要暗号資産ウォレットの固有 ID（MetaMask: `nkbihfbeogaeaoehlefnkodbefgpgknn`, TronLink: `ibnejdfjmmkpcnlpebklmnkoeoihofec` 等）の文字列含有を検査。
  2. 不審 API（`ws2_32.dll`, `cmd.exe`, `powershell -enc`, `certutil` 等）の含有スコアを加算し、高リスク警告を発報。

## 2.12 `FN-SCAN-12`: 【確定】ゲームエンジンコア (UnityPlayer.dll 等) 署名喪失・改ざん看破
* **目的:** 正規ゲームエンジンのコアバイナリと同名ファイルを偽装配置する攻撃を遮断する。
* **処理仕様:**
  1. `UnityPlayer.dll`, `UnrealEngine.dll` 等の本来正規署名を保持すべきエンジンコアバイナリを特定。
  2. 当該バイナリの署名が喪失（未署名）しているか、無効・失効している場合、重大な改ざんとしてブロック判定を下す。

## 2.13 `FN-SCAN-13`: 【確定】Anti-Wiper Canary 除外契約 (`CanaryTrapManager.IsCanaryPath`)
* **目的:** Anti-Wiper 防衛用の `!0_gst_canary.dat` をGST自身のセキュリティ検査対象から明示的に除外し、Canaryの存在・変更を自己脅威として誤評価しない。
* **除外契約:** スキャン列挙、単一ファイル検査、ドロップ監査、差分監査、ハッシュ算出、Allow List照合、リスク評価の各入口で、`CanaryTrapManager.IsCanaryPath(filePath)` が `true` のパスは最優先で除外しなければならない。
* **ホワイトリスト範囲:** 除外対象はCanary名/管理対象パスに限定し、同名の第三者ファイルを無条件に信頼・削除しない。Canary除外は「安全判定」の付与ではなく、「GST内部監視からの除外」という処理契約である。
* **設計境界:** `ICanaryTrapManager` 等のPortはContracts層に置き、Scanner/Infrastructureから参照する。Domain層はCanaryの物理パスやファイルシステムAPIを直接参照しない。
* **実装ポイント:** `ScanDirectoryAsync` の列挙結果を `riskAssessmentService` へ渡す前、`InspectSingleFileAsync` のファイル存在確認後、`DroppedArtifactTracker` / `RecentArtifactAuditor` の検査対象化前に同一除外判定を適用する。
* **監査:** 除外されたCanaryパスは脅威検出結果へ登録しない一方、必要に応じて「GST防衛資産のため除外した」という低レベル監査情報のみを内部ログへ記録する。ユーザー向け脅威件数には二重計上しない。

---

# 3. Contracts Layer 定義 (`GameSecurityTool.Contracts`)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;

public interface IScanEngine
{
    Task<ScanResultDto> ScanDirectoryAsync(
        Guid gameFolderId,
        string directoryPath, 
        IProgress<ScanProgressDto>? progress = null, 
        CancellationToken ct = default);

    Task<FileRiskResultDto> InspectSingleFileAsync(
        string filePath, 
        CancellationToken ct = default);
}

public interface IPathResolver
{
    string ResolveFinalPath(string path);
    bool IsPathWithinBounds(string candidatePath, string rootFolderPath);
}

public interface INewGamePlacementWatcher : IDisposable
{
    event Action<DiscoveredGameCandidateDto>? GameCandidateDiscovered;
    void Start();
    void Stop();
    Task UpdateMonitoredDirectoriesAsync(IReadOnlyList<string> directories, CancellationToken ct = default);
}

public interface IRecentArtifactAuditor
{
    Task<IReadOnlyList<DroppedArtifactInspectionDto>> AuditRecentArtifactsAsync(
        TimeSpan lookbackWindow,
        string targetGameFolderPath,
        CancellationToken ct = default);
}

/// <summary>
/// Anti-Wiper Canary の管理・認識 Port。
/// Canary はセキュリティ判定で安全とみなすのではなく、GST内部検査の対象外として扱う。
/// </summary>
public interface ICanaryTrapManager : IDisposable
{
    bool IsCanaryPath(string filePath);
    Task DeployCanaryTrapsAsync(IReadOnlyList<string> protectedRootDirectories, CancellationToken ct = default);
    Task RemoveCanaryTrapsAsync(CancellationToken ct = default);
}

public interface ISafeShortcutGenerator
{
    Task<bool> GenerateSafeLaunchShortcutAsync(
        Guid gameProfileId,
        string gameDisplayName,
        string targetExecutablePath,
        string outputDirectoryPath,
        bool isStrictMode = true,
        CancellationToken ct = default);
}

public interface IExternalSecurityScannerAdapter
{
    Task<ExternalScanResultDto> ExecuteScanAsync(string targetPath, ExternalScanOptionsDto options, CancellationToken ct = default);
}
```

```csharp
namespace GameSecurityTool.Contracts.Dtos;

using System;
using System.Collections.Generic;
using GameSecurityTool.Contracts.Common;

public sealed record ScanProgressDto(
    int TotalFiles,
    int ProcessedFiles,
    double Percentage,
    string CurrentProcessingFile
);

public sealed record ScanResultDto(
    string OperationId,
    Guid GameFolderId,
    int FilesChecked,
    int FilesChanged,
    IReadOnlyList<FileRiskResultDto> DetectedRisks,
    TimeSpan ElapsedTime
);

public sealed record FileRiskResultDto(
    string AbsolutePath,
    string RelativePath,
    string SHA256,
    long FileSize,
    RiskLevel RiskLevel,
    bool IsSigned,
    string? PublisherName,
    IReadOnlyList<ThreatReasonDto> Reasons
);

public sealed record DiscoveredGameCandidateDto(
    string RootFolderPath,
    string MainExecutablePath,
    string DetectedGameName,
    DateTimeOffset DiscoveredAtUtc
);

public sealed record DroppedArtifactInspectionDto(
    string FilePath,
    string FileName,
    long FileSizeBytes,
    bool IsSigned,
    string? PublisherName,
    IReadOnlyList<string> DetectedThreatSignals,
    DateTimeOffset CreatedAtUtc
);

public sealed record ExternalScanResultDto(
    bool Success,
    bool ThreatDetected,
    int ExitCode,
    string OutputSummary
);
```

---

# 4. Infrastructure Layer 完全実装 (`GameSecurityTool.Infrastructure`)

## 4.1 並列スキャン ＆ Port 統合 (`FastFileSystemScanner.cs` - C-1 是正)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Scanner;

using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Linq;
using System.Security.Cryptography;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Common;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Infrastructure.Native;
using GameSecurityTool.Infrastructure.Persistence;
using GameSecurityTool.Infrastructure.Persistence.Entities;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;

public sealed class FastFileSystemScanner(
    IDbContextFactory<AppDbContext> dbFactory,
    IDbWriteQueue dbWriter,
    IPathResolver pathResolver,
    IRiskAssessmentService riskAssessmentService, // C-1 是正: Domain 具象ではなく Contracts Port を注入
    IAllowListRepository allowListRepository,
    ILogger<FastFileSystemScanner> logger) : IScanEngine
{
    public async Task<ScanResultDto> ScanDirectoryAsync(
        Guid gameFolderId,
        string directoryPath, 
        IProgress<ScanProgressDto>? progress = null, 
        CancellationToken ct = default)
    {
        var startTime = Stopwatch.StartNew();
        
        var enumOptions = new EnumerationOptions
        {
            RecurseSubdirectories = true,
            IgnoreInaccessible = true,
            AttributesToSkip = FileAttributes.ReparsePoint
        };

        var files = Directory.EnumerateFiles(directoryPath, "*.*", enumOptions)
            .Where(f => IsTargetExtension(f) && pathResolver.IsPathWithinBounds(f, directoryPath))
            .ToList();

        int totalFiles = files.Count;
        int processedCount = 0;
        int changedCount = 0;

        // 1. 読み取りは短命 DbContext で AsNoTracking 実行
        Dictionary<string, FileRecord> previousRecords;
        await using (var readDb = await dbFactory.CreateDbContextAsync(ct))
        {
            previousRecords = await readDb.FileRecords
                .AsNoTracking()
                .Where(f => f.GameFolderId == gameFolderId)
                .ToDictionaryAsync(f => f.AbsolutePath, f => f, ct);
        }

        var activeAllowRulesDto = await allowListRepository.GetActiveRulesAsync(gameFolderId, ct);

        var results = new ConcurrentBag<FileRecord>();
        var risks = new ConcurrentBag<FileRiskResultDto>();

        var parallelOptions = new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount,
            CancellationToken = ct
        };

        await Parallel.ForEachAsync(files, parallelOptions, async (filePath, token) =>
        {
            var fileInfo = new FileInfo(filePath);
            bool needsHash = true;
            string? sha256Hex = null;

            // Fast Metadata Check (有効な過去ハッシュが存在する場合のみスキップ)
            if (previousRecords.TryGetValue(filePath, out var prevRecord) && !string.IsNullOrEmpty(prevRecord.SHA256))
            {
                if (prevRecord.FileSize == fileInfo.Length && prevRecord.LastWriteTimeUtc == fileInfo.LastWriteTimeUtc)
                {
                    needsHash = false;
                    sha256Hex = prevRecord.SHA256;
                }
            }

            bool isNewFile = !previousRecords.ContainsKey(filePath);
            if (needsHash)
            {
                Interlocked.Increment(ref changedCount);
                sha256Hex = await ComputeHashSafelyAsync(filePath, token);
            }

            // ロック中ファイル安全処理 (ハッシュ計算失敗時は DB 登録を保留)
            if (string.IsNullOrEmpty(sha256Hex))
            {
                logger.LogWarning("ファイルがロック中または読み取り不能のためスキャンをスキップ: {Path}", filePath);
                return;
            }

            // 【C-1 是正】IRiskAssessmentService Port 経由で DTO ベースのリスク評価を実行
            bool isSigned = Win32SecurityNativeMethods.VerifyAuthenticodeSignature(filePath);
            var candidateDto = new FileIdentityCandidateDto(
                FileName: Path.GetFileName(filePath),
                AbsolutePath: filePath,
                SHA256: sha256Hex,
                FileSize: fileInfo.Length,
                IsSigned: isSigned,
                PublisherName: null,
                IsNewFile: isNewFile
            );

            var assessmentDto = riskAssessmentService.AssessFileRisk(candidateDto, activeAllowRulesDto, DateTimeOffset.UtcNow);
            
            if (assessmentDto.Level >= RiskLevel.Medium)
            {
                risks.Add(new FileRiskResultDto(
                    AbsolutePath: filePath,
                    RelativePath: Path.GetRelativePath(directoryPath, filePath),
                    SHA256: sha256Hex,
                    FileSize: fileInfo.Length,
                    RiskLevel: assessmentDto.Level,
                    IsSigned: isSigned,
                    PublisherName: null,
                    Reasons: assessmentDto.Reasons
                ));
            }

            var newRecord = new FileRecord
            {
                Id = prevRecord?.Id ?? Guid.NewGuid(),
                GameFolderId = gameFolderId,
                AbsolutePath = filePath,
                FilePath = Path.GetRelativePath(directoryPath, filePath),
                SHA256 = sha256Hex,
                FileSize = fileInfo.Length,
                LastWriteTimeUtc = fileInfo.LastWriteTimeUtc,
                LastScannedAt = DateTimeOffset.UtcNow
            };

            results.Add(newRecord);

            int current = Interlocked.Increment(ref processedCount);
            if (current % 50 == 0 || current == totalFiles)
            {
                progress?.Report(new ScanProgressDto(totalFiles, current, (double)current / totalFiles * 100, filePath));
            }
        });

        var sortedResults = results.OrderBy(r => r.AbsolutePath, StringComparer.OrdinalIgnoreCase).ToList();
        var newEntities = sortedResults.Where(r => !previousRecords.ContainsKey(r.AbsolutePath)).ToList();
        var updatedEntities = sortedResults.Where(r => previousRecords.ContainsKey(r.AbsolutePath)).ToList();
        var currentPaths = sortedResults.Select(r => r.AbsolutePath).ToHashSet();
        var deletedEntities = previousRecords.Values.Where(r => !currentPaths.Contains(r.AbsolutePath)).ToList();

        // 2. IDbWriteQueue 経由の直列化一括コミット
        await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
        {
            await using var writeDb = await dbFactory.CreateDbContextAsync(innerCt);
            if (newEntities.Count > 0)
            {
                await writeDb.FileRecords.AddRangeAsync(newEntities, innerCt);
            }
            if (updatedEntities.Count > 0)
            {
                writeDb.FileRecords.UpdateRange(updatedEntities);
            }
            if (deletedEntities.Count > 0)
            {
                writeDb.FileRecords.RemoveRange(deletedEntities);
            }
            await writeDb.SaveChangesAsync(innerCt);
        }, ct);

        startTime.Stop();
        logger.LogInformation("スキャン完了: {Total}件 (変更: {Changed}件) / 所要時間: {Time}ms", totalFiles, changedCount, startTime.ElapsedMilliseconds);

        return new ScanResultDto("OP-SCAN", gameFolderId, totalFiles, changedCount, risks.ToList(), startTime.Elapsed);
    }

    public async Task<FileRiskResultDto> InspectSingleFileAsync(string filePath, CancellationToken ct = default)
    {
        if (!File.Exists(filePath))
        {
            return new FileRiskResultDto(filePath, Path.GetFileName(filePath), string.Empty, 0, RiskLevel.Safe, false, null, [
                new ThreatReasonDto("FILE_NOT_FOUND", "検査対象ファイルが存在しません", ThreatSeverity.Info)
            ]);
        }

        var fileInfo = new FileInfo(filePath);
        string? sha256Hex = await ComputeHashSafelyAsync(filePath, ct);
        if (string.IsNullOrEmpty(sha256Hex))
        {
            return new FileRiskResultDto(filePath, fileInfo.Name, string.Empty, fileInfo.Length, RiskLevel.Low, false, null, [
                new ThreatReasonDto("FILE_LOCKED", "ファイルが排他ロックされているためハッシュ取得できません", ThreatSeverity.Warning)
            ]);
        }

        bool isSigned = Win32SecurityNativeMethods.VerifyAuthenticodeSignature(filePath);
        var candidateDto = new FileIdentityCandidateDto(
            FileName: fileInfo.Name,
            AbsolutePath: filePath,
            SHA256: sha256Hex,
            FileSize: fileInfo.Length,
            IsSigned: isSigned,
            PublisherName: null,
            IsNewFile: true
        );

        var allowRulesDto = await allowListRepository.GetActiveRulesAsync(null, ct);
        var assessmentDto = riskAssessmentService.AssessFileRisk(candidateDto, allowRulesDto, DateTimeOffset.UtcNow);

        return new FileRiskResultDto(
            AbsolutePath: filePath,
            RelativePath: fileInfo.Name,
            SHA256: sha256Hex,
            FileSize: fileInfo.Length,
            RiskLevel: assessmentDto.Level,
            IsSigned: isSigned,
            PublisherName: null,
            Reasons: assessmentDto.Reasons
        );
    }

    private static async Task<string?> ComputeHashSafelyAsync(string filePath, CancellationToken ct)
    {
        try
        {
            await using var stream = new FileStream(filePath, FileMode.Open, FileAccess.Read, FileShare.ReadWrite | FileShare.Delete, 81920, useAsync: true);
            byte[] hashBytes = await SHA256.HashDataAsync(stream, ct);
            return Convert.ToHexString(hashBytes);
        }
        catch
        {
            return null;
        }
    }

    private static bool IsTargetExtension(string path)
    {
        string ext = Path.GetExtension(path).ToLowerInvariant();
        return ext is ".exe" or ".dll" or ".bat" or ".cmd" or ".asi" or ".lua" or ".py";
    }
}
```

## 4.2 PE ヘッダー検証 ＆ ドロップ追跡 Adapter (`DroppedArtifactTracker.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Scanner;

using System;
using System.Collections.Generic;
using System.IO;
using System.Security.Cryptography;
using GameSecurityTool.Contracts.Common;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.Extensions.Logging;

public sealed class DroppedArtifactTracker(ILogger<DroppedArtifactTracker> logger) : IDroppedArtifactTracker
{
    public Dictionary<string, DateTimeOffset> CapturePreLaunchSnapshot()
    {
        var snapshot = new Dictionary<string, DateTimeOffset>(StringComparer.OrdinalIgnoreCase);
        var enumOptions = new EnumerationOptions { RecurseSubdirectories = true, IgnoreInaccessible = true };

        foreach (var dir in GetTargetDropDirectories())
        {
            if (!Directory.Exists(dir)) continue;

            try
            {
                foreach (var file in Directory.EnumerateFiles(dir, "*.*", enumOptions))
                {
                    if (!ShouldInspectDroppedFile(file)) continue;

                    try
                    {
                        var info = new FileInfo(file);
                        snapshot[file] = info.LastWriteTimeUtc;
                    }
                    catch { }
                }
            }
            catch (Exception ex)
            {
                logger.LogTrace(ex, "スナップショットスキップ: {Dir}", dir);
            }
        }

        return snapshot;
    }

    public IReadOnlyList<FileRiskResultDto> DetectDroppedArtifacts(
        string gameFolderPath,
        Dictionary<string, DateTimeOffset> preLaunchSnapshot)
    {
        var detectedArtifacts = new List<FileRiskResultDto>();
        string normalizedGameRoot = Path.GetFullPath(gameFolderPath).TrimEnd(Path.DirectorySeparatorChar) + Path.DirectorySeparatorChar;
        var enumOptions = new EnumerationOptions { RecurseSubdirectories = true, IgnoreInaccessible = true };

        foreach (var dir in GetTargetDropDirectories())
        {
            if (!Directory.Exists(dir)) continue;

            try
            {
                foreach (var file in Directory.EnumerateFiles(dir, "*.*", enumOptions))
                {
                    if (file.StartsWith(normalizedGameRoot, StringComparison.OrdinalIgnoreCase)) continue;
                    if (!ShouldInspectDroppedFile(file)) continue;

                    var info = new FileInfo(file);
                    if (!preLaunchSnapshot.TryGetValue(file, out var previousWriteTime) || info.LastWriteTimeUtc > previousWriteTime)
                    {
                        logger.LogWarning("ゲーム外ドロップ不審バイナリ検知: {Path}", file);
                        string sha256Hex = ComputeHashSafely(file);

                        detectedArtifacts.Add(new FileRiskResultDto(
                            AbsolutePath: file,
                            RelativePath: Path.GetFileName(file),
                            SHA256: sha256Hex,
                            FileSize: info.Length,
                            RiskLevel: RiskLevel.High,
                            IsSigned: false,
                            PublisherName: null,
                            Reasons: [
                                new ThreatReasonDto("DROPPED_OUTSIDE_GAME_FOLDER", "ゲーム実行中に外部領域へ配置された不審バイナリです", ThreatSeverity.Danger)
                            ]
                        ));
                    }
                }
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "ドロップ検出エラー: {Dir}", dir);
            }
        }

        return detectedArtifacts.AsReadOnly();
    }

    public static bool ShouldInspectDroppedFile(string filePath)
    {
        if (filePath.Contains("\\Documents\\My Games\\", StringComparison.OrdinalIgnoreCase) ||
            filePath.Contains("\\Saved Games\\", StringComparison.OrdinalIgnoreCase) ||
            filePath.Contains("\\Steam\\userdata\\", StringComparison.OrdinalIgnoreCase) ||
            filePath.Contains("\\Goldberg SteamEmu Saves\\", StringComparison.OrdinalIgnoreCase) ||
            filePath.Contains("\\Users\\Public\\Documents\\", StringComparison.OrdinalIgnoreCase))
        {
            return false;
        }

        string ext = Path.GetExtension(filePath).ToLowerInvariant();
        bool isScriptOrLink = ext is ".bat" or ".cmd" or ".vbs" or ".ps1" or ".lnk";
        bool isPeBinary = IsValidPeExecutable(filePath);

        return isScriptOrLink || isPeBinary;
    }

    public static bool IsValidPeExecutable(string filePath)
    {
        try
        {
            if (!File.Exists(filePath)) return false;

            using var stream = new FileStream(filePath, FileMode.Open, FileAccess.Read, FileShare.ReadWrite | FileShare.Delete, bufferSize: 512, useAsync: false);
            using var reader = new BinaryReader(stream);

            if (stream.Length < 0x40) return false;

            ushort dosMagic = reader.ReadUInt16();
            if (dosMagic != 0x5A4D) return false; // 'MZ'

            stream.Seek(0x3C, SeekOrigin.Begin);
            int peOffset = reader.ReadInt32();

            if (peOffset <= 0 || peOffset + 4 > stream.Length) return false;

            stream.Seek(peOffset, SeekOrigin.Begin);
            uint peSignature = reader.ReadUInt32();

            return peSignature == 0x00004550; // 'PE\0\0'
        }
        catch
        {
            return false;
        }
    }

    private static string ComputeHashSafely(string filePath)
    {
        try
        {
            using var stream = new FileStream(filePath, FileMode.Open, FileAccess.Read, FileShare.ReadWrite);
            byte[] hash = SHA256.HashData(stream);
            return Convert.ToHexString(hash);
        }
        catch
        {
            return "UNREADABLE_FILE";
        }
    }

    private static IEnumerable<string> GetTargetDropDirectories()
    {
        yield return Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData);
        yield return Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData);
        yield return Path.GetTempPath();
        yield return Environment.GetFolderPath(Environment.SpecialFolder.Startup);
    }
}
```

## 4.3 新規ゲーム配置受動監視 Adapter (`NewGamePlacementWatcher.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Scanner;

using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.Extensions.Logging;

public sealed class NewGamePlacementWatcher(ILogger<NewGamePlacementWatcher> logger) : INewGamePlacementWatcher
{
    private readonly List<FileSystemWatcher> _watchers = [];
    private readonly ConcurrentDictionary<string, DateTimeOffset> _pendingDebounceEvents = new(StringComparer.OrdinalIgnoreCase);
    private readonly CancellationTokenSource _cts = new();
    private Task? _debounceProcessingTask;

    public event Action<DiscoveredGameCandidateDto>? GameCandidateDiscovered;

    public void Start()
    {
        _debounceProcessingTask = Task.Run(ProcessDebounceLoopAsync);
        logger.LogInformation("新規ゲーム配置監視ウォッチャーを開始しました。");
    }

    public void Stop()
    {
        _cts.Cancel();
        foreach (var watcher in _watchers)
        {
            try
            {
                watcher.EnableRaisingEvents = false;
                watcher.Dispose();
            }
            catch { }
        }
        _watchers.Clear();
    }

    public Task UpdateMonitoredDirectoriesAsync(IReadOnlyList<string> directories, CancellationToken ct = default)
    {
        Stop();

        foreach (var dir in directories.Distinct(StringComparer.OrdinalIgnoreCase))
        {
            if (!Directory.Exists(dir)) continue;

            try
            {
                var watcher = new FileSystemWatcher(dir)
                {
                    IncludeSubdirectories = true,
                    NotifyFilter = NotifyFilters.DirectoryName | NotifyFilters.FileName | NotifyFilters.CreationTime,
                    Filter = "*.*"
                };

                watcher.Created += OnFileSystemCreated;
                watcher.EnableRaisingEvents = true;
                _watchers.Add(watcher);
                logger.LogInformation("新規ゲーム配置の監視ディレクトリを登録しました: {Dir}", dir);
            }
            catch (Exception ex)
            {
                logger.LogWarning(ex, "監視ディレクトリの登録に失敗しました: {Dir}", dir);
            }
        }

        return Task.CompletedTask;
    }

    private void OnFileSystemCreated(object sender, FileSystemEventArgs e)
    {
        string path = e.FullPath;
        string ext = Path.GetExtension(path).ToLowerInvariant();

        if (Directory.Exists(path) || ext == ".exe")
        {
            _pendingDebounceEvents.AddOrUpdate(path, DateTimeOffset.UtcNow, (_, _) => DateTimeOffset.UtcNow);
        }
    }

    private async Task ProcessDebounceLoopAsync()
    {
        var quietPeriod = TimeSpan.FromSeconds(3); // 3秒間の静止待機

        while (!_cts.Token.IsCancellationRequested)
        {
            try
            {
                await Task.Delay(1000, _cts.Token);
                var now = DateTimeOffset.UtcNow;
                var keysToProcess = new List<string>();

                foreach (var (path, lastEventTime) in _pendingDebounceEvents)
                {
                    if (now - lastEventTime >= quietPeriod)
                    {
                        keysToProcess.Add(path);
                    }
                }

                foreach (var path in keysToProcess)
                {
                    if (_pendingDebounceEvents.TryRemove(path, out _))
                    {
                        EvaluateCandidateFolder(path);
                    }
                }
            }
            catch (OperationCanceledException) when (_cts.IsCancellationRequested)
            {
                break;
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "新規ゲーム配置 Debounce ループ内エラー");
            }
        }
    }

    private void EvaluateCandidateFolder(string path)
    {
        try
        {
            string? targetDirectory = Directory.Exists(path) ? path : Path.GetDirectoryName(path);
            if (string.IsNullOrEmpty(targetDirectory) || !Directory.Exists(targetDirectory)) return;

            var exeFiles = Directory.EnumerateFiles(targetDirectory, "*.exe", SearchOption.TopDirectoryOnly).ToList();
            if (exeFiles.Count == 0) return;

            string folderName = new DirectoryInfo(targetDirectory).Name;
            string mainExe = exeFiles.FirstOrDefault(e => Path.GetFileNameWithoutExtension(e).Equals(folderName, StringComparison.OrdinalIgnoreCase)) ?? exeFiles[0];

            logger.LogInformation("新規ゲーム候補を特定しました: {Name} ({Path})", folderName, mainExe);

            var candidate = new DiscoveredGameCandidateDto(
                RootFolderPath: targetDirectory,
                MainExecutablePath: mainExe,
                DetectedGameName: folderName,
                DiscoveredAtUtc: DateTimeOffset.UtcNow
            );

            GameCandidateDiscovered?.Invoke(candidate);
        }
        catch (Exception ex)
        {
            logger.LogTrace(ex, "ゲーム候補評価中の例外 (安全にスキップ)");
        }
    }

    public void Dispose()
    {
        Stop();
        _cts.Dispose();
    }
}
```

## 4.4 共連れドロップ監査 ＆ 署名レス検査 Adapter (`RecentArtifactAuditor.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Scanner;

using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Infrastructure.Native;
using Microsoft.Extensions.Logging;

public sealed class RecentArtifactAuditor(ILogger<RecentArtifactAuditor> logger) : IRecentArtifactAuditor
{
    private static readonly HashSet<string> SuspiciousKeywords = new(StringComparer.OrdinalIgnoreCase)
    {
        "nkbihfbeogaeaoehlefnkodbefgpgknn", // MetaMask
        "ibnejdfjmmkpcnlpebklmnkoeoihofec", // TronLink
        "User Data\\Default\\Extensions",
        "powershell -enc",
        "certutil -urlcache"
    };

    public async Task<IReadOnlyList<DroppedArtifactInspectionDto>> AuditRecentArtifactsAsync(
        TimeSpan lookbackWindow,
        string targetGameFolderPath,
        CancellationToken ct = default)
    {
        var detectedArtifacts = new List<DroppedArtifactInspectionDto>();
        var now = DateTimeOffset.UtcNow;
        var threshold = now - lookbackWindow;
        string normalizedGamePath = Path.GetFullPath(targetGameFolderPath).TrimEnd(Path.DirectorySeparatorChar) + Path.DirectorySeparatorChar;

        var targetSearchDirs = new[]
        {
            Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData),
            Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
            Path.GetTempPath(),
            Environment.GetFolderPath(Environment.SpecialFolder.Startup)
        };

        var enumOptions = new EnumerationOptions { RecurseSubdirectories = true, IgnoreInaccessible = true };

        await Task.Run(() =>
        {
            foreach (var baseDir in targetSearchDirs)
            {
                ct.ThrowIfCancellationRequested();
                if (!Directory.Exists(baseDir)) continue;

                try
                {
                    foreach (var file in Directory.EnumerateFiles(baseDir, "*.*", enumOptions))
                    {
                        if (file.StartsWith(normalizedGamePath, StringComparison.OrdinalIgnoreCase)) continue;

                        string ext = Path.GetExtension(file).ToLowerInvariant();
                        if (ext is not (".exe" or ".dll" or ".bat" or ".ps1" or ".vbs")) continue;

                        var info = new FileInfo(file);
                        if (info.CreationTimeUtc >= threshold || info.LastWriteTimeUtc >= threshold)
                        {
                            var signals = InspectArtifactSignals(file);
                            bool isSigned = Win32SecurityNativeMethods.VerifyAuthenticodeSignature(file);

                            detectedArtifacts.Add(new DroppedArtifactInspectionDto(
                                FilePath: file,
                                FileName: info.Name,
                                FileSizeBytes: info.Length,
                                IsSigned: isSigned,
                                PublisherName: null,
                                DetectedThreatSignals: signals,
                                CreatedAtUtc: info.CreationTimeUtc
                            ));

                            logger.LogWarning("共連れドロップ不審ファイルを検出しました: {Path} (Signals: {Count})", file, signals.Count);
                        }
                    }
                }
                catch (Exception ex)
                {
                    logger.LogTrace(ex, "監査ディレクトリ走査中の例外: {Dir}", baseDir);
                }
            }
        }, ct);

        return detectedArtifacts;
    }

    private static IReadOnlyList<string> InspectArtifactSignals(string filePath)
    {
        var signals = new List<string>();

        try
        {
            if (new FileInfo(filePath).Length > 10 * 1024 * 1024) return signals; // 10MB 超はスキップ

            byte[] bytes = File.ReadAllBytes(filePath);
            string content = System.Text.Encoding.ASCII.GetString(bytes);

            foreach (var keyword in SuspiciousKeywords)
            {
                if (content.Contains(keyword, StringComparison.OrdinalIgnoreCase))
                {
                    signals.Add($"ContainsTargetPattern:{keyword}");
                }
            }
        }
        catch { }

        return signals;
    }
}
```

## 4.5 ゲームエンジンコア (UnityPlayer.dll 等) 署名喪失・改ざん検証 (`EngineIntegrityInspector.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Scanner;

using System;
using System.Collections.Generic;
using System.IO;
using GameSecurityTool.Infrastructure.Native;

public static class EngineIntegrityInspector
{
    private static readonly HashSet<string> OfficialEngineCoreDlls = new(StringComparer.OrdinalIgnoreCase)
    {
        "UnityPlayer.dll", "UnrealEngine.dll"
    };

    public static bool IsEngineCoreSignatureLost(string filePath)
    {
        string fileName = Path.GetFileName(filePath);
        if (!OfficialEngineCoreDlls.Contains(fileName)) return false;

        // 正規エンジンコアバイナリは必ずデジタル署名を保持している
        bool isSigned = Win32SecurityNativeMethods.VerifyAuthenticodeSignature(filePath);
        return !isSigned; // 署名がない・破損している場合は改ざんと判定
    }
}
```

---

# 5. 単体テスト仕様 (`GST.UnitTests.SCAN`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.SCAN;

using System;
using System.IO;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Common;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Infrastructure.Persistence;
using GameSecurityTool.Infrastructure.Scanner;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging.Abstractions;
using Moq;
using Xunit;

public class ScannerTests
{
    [Fact]
    public async Task InspectSingleFileAsync_ValidFile_ReturnsRiskEvaluation()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>().UseSqlite("Data Source=:memory:").Options;
        var dbFactoryMock = new Mock<IDbContextFactory<AppDbContext>>();
        var dbWriterMock = new Mock<IDbWriteQueue>();
        var pathResolverMock = new Mock<IPathResolver>();

        // C-1 是正: IRiskAssessmentService Port をモック化
        var riskAssessmentMock = new Mock<IRiskAssessmentService>();
        riskAssessmentMock.Setup(r => r.AssessFileRisk(It.IsAny<FileIdentityCandidateDto>(), It.IsAny<IReadOnlyList<AllowListRuleDto>>(), It.IsAny<DateTimeOffset?>()))
            .Returns(new RiskAssessmentResultDto(RiskLevel.Safe, 0, [], DateTimeOffset.UtcNow, "1.0"));

        var allowListMock = new Mock<IAllowListRepository>();
        allowListMock.Setup(a => a.GetActiveRulesAsync(It.IsAny<Guid?>(), default)).ReturnsAsync([]);

        var scanner = new FastFileSystemScanner(
            dbFactoryMock.Object,
            dbWriterMock.Object,
            pathResolverMock.Object,
            riskAssessmentMock.Object,
            allowListMock.Object,
            NullLogger<FastFileSystemScanner>.Instance);

        string tempFile = Path.GetTempFileName();
        await File.WriteAllTextAsync(tempFile, "TEST_DATA");

        var result = await scanner.InspectSingleFileAsync(tempFile);
        
        Assert.Equal(RiskLevel.Safe, result.RiskLevel);
        Assert.False(string.IsNullOrEmpty(result.SHA256));

        File.Delete(tempFile);
    }

    [Fact]
    public void ThreatReasonCodes_SSOT_IsContractsOwnedAndUnique()
    {
        // TC-ARCH-SSOT-01: Contracts.Common.ThreatReasonCodes を唯一の Threat Reason Code SSOT として検証
        var registryType = typeof(GameSecurityTool.Contracts.Common.ThreatReasonCodes);
        var fields = registryType.GetFields(BindingFlags.Public | BindingFlags.Static)
            .Where(field => field.IsLiteral && field.FieldType == typeof(string))
            .ToArray();

        Assert.NotEmpty(fields);

        var values = fields
            .Select(field => field.GetRawConstantValue())
            .Cast<string>()
            .ToArray();

        Assert.Equal(values.Length, values.Distinct(StringComparer.Ordinal).Count());
        Assert.All(values, value => Assert.False(string.IsNullOrWhiteSpace(value)));

        var reasonProperty = typeof(GameSecurityTool.Contracts.Dtos.ThreatReasonDto)
            .GetProperty(nameof(GameSecurityTool.Contracts.Dtos.ThreatReasonDto.ThreatReasonCode));

        Assert.NotNull(reasonProperty);
        Assert.Equal(typeof(string), reasonProperty!.PropertyType);

        var competingEnums = new[]
            {
                Assembly.Load("GameSecurityTool.Domain"),
                registryType.Assembly
            }
            .SelectMany(assembly => assembly.GetTypes())
            .Where(type => type.IsEnum && type.Name == "ThreatReasonCode")
            .ToArray();

        Assert.Empty(competingEnums);
    }

    [Fact]
    public async Task AuditRecentArtifactsAsync_WhenExecutableInTemp_DetectsSuspiciousKeyword()
    {
        var auditor = new RecentArtifactAuditor(NullLogger<RecentArtifactAuditor>.Instance);
        string tempDir = Path.GetTempPath();
        string testFile = Path.Combine(tempDir, "malicious_stealer_test.exe");

        // MetaMask 拡張機能 ID を含むダミーバイナリを配置
        string content = "MZ... SomeCode ... nkbihfbeogaeaoehlefnkodbefgpgknn ... End";
        await File.WriteAllTextAsync(testFile, content);

        try
        {
            var results = await auditor.AuditRecentArtifactsAsync(TimeSpan.FromMinutes(1), "D:\\Games\\LegitGame");
            Assert.Contains(results, r => r.FilePath.Equals(testFile, StringComparison.OrdinalIgnoreCase));
            Assert.Contains(results, r => r.DetectedThreatSignals.Any(s => s.Contains("nkbihfbeogaeaoehlefnkodbefgpgknn")));
        }
        finally
        {
            if (File.Exists(testFile)) File.Delete(testFile);
        }
    }
}
```

---
