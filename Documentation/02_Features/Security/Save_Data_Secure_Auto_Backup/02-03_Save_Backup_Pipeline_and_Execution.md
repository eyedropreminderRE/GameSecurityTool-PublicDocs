# GameSecurityTool Save Backup Pipeline & Execution Specification

**Document ID:** GST-FEAT-SAVE-PIPE-003  
**Version:** 4.2
**Status:** Approved Application Specification  
**Target Layer:** Application Layer (`GameSecurityTool.Application`)  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Pipeline Architecture Overview

Save Data Secure Auto Backup の実行パイプラインは、UI スレッドおよびゲームプロセスを一切ブロックしない完全非同期のバックグラウンドワークフローとして構築する。

```text
[ Trigger (Game End / Manual / FileWatcher) ]
                       │
                       ▼
┌────────────────────────────────────────────────────────┐
│            Debounce & Request Merging Buffer           │
│  - ConcurrentDictionary<Guid, PendingBackupRequest>    │
│  - 同一ゲームの自動トリガーを Debounce 集約 (3〜5秒)    │
│  - 手動トリガー (Manual) は即時パススルー               │
│  - 置換された旧リクエストの Password を ZeroMemory 消去│ (M-新規1是正)
└──────────────────────┬─────────────────────────────────┘
                       │ (WriteAsync: 集約済み単一リクエスト)
                       ▼
┌────────────────────────────────────────────────────────┐
│        BoundedChannel<SaveBackupExecutionRequest>      │
│  - Capacity: 100, FullMode: Wait                       │
│  - SingleReader: true, SingleWriter: false             │
└──────────────────────┬─────────────────────────────────┘
                       │ (ReadAsync)
                       ▼
┌────────────────────────────────────────────────────────┐
│     SaveBackupBackgroundWorker (HostedService)         │
│  - Concurrency: 1 (システム全体で完全直列実行)         │
│  - CancellationToken 管理                              │
└──────────────────────┬─────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────┐
│               SaveBackupExecutionEngine                │
│  1. Storage Safety Guard (空き容量事前確認 Port 呼出)   │
│  2. ISaveDataPathScanner (マニフェスト収集 Port)       │
│  3. Domain Change Detection Engine (差分判定)          │
│  4. ISaveBackupStorage (64KB チャンクコンテナ生成 Port)│
│  5. ISaveBackupRepository & Audit Logger コミット      │
└────────────────────────────────────────────────────────┘
```

---

# 2. 2段階キュー設計 ＆ 重複リクエスト集約制御

バースト的なファイル変更イベントによるキュー溢れとデッドロックを防止するため、**「インメモリ Debounce バッファ ＋ 有界チャネル（BoundedChannel）」** の 2 段階構造を採用する。

## 2.1 インメモリ Debounce バッファ (`SaveBackupRequestCoordinator`)
1. **自動トリガーの集約 (Automatic Merging):**  
   同一 `GameProfileId` に対する自動バックアップ要求（ファイル監視イベント等）が連続発生した場合、`ConcurrentDictionary<Guid, PendingRequest>` 上で最終更新タイムスタンプのみを更新し、指定時間（3秒）静止した段階で初めて Channel へ 1 件のリクエストとして投入する。
2. **手動トリガーの即時実行 (Manual Pass-through):**  
   ユーザーが UI から明示的に要求した「今すぐバックアップ」は Debounce をバイパスし、即座に Channel へ投入して確実に直列実行する。
3. **旧パスワードバッファの即時消去 (M-新規1 是正):**  
   Debounce ウィンドウ内で同一ゲームの保留リクエストが上書きされた場合、置き換えられる古いリクエストが保持していた `Password` バイト配列を直ちに `CryptographicOperations.ZeroMemory` / `Array.Clear` してメモリ上から消去する。
4. **起動時差分ポストスキャン自己修復 (approved quality refinement):**  
   インメモリ Debounce バッファの保留中に OS 異常終了やプロセス突然死が発生した場合、インメモリイベントが消失する。これに対処するため、**次回 GST 起動時およびゲーム終了時に「差分ポストスキャン」を実行**する。前回の確定スナップショットと実際のセーブディレクトリのハッシュ差分を比較し、未バックアップの変更が存在する場合は自律的にスナップショットを作成してインメモリ消失分を完全自己修復する。

## 2.1.1 `Password` 所有権ライフサイクル (approved security contract)

`SaveBackupExecutionRequest.Password` を含むSave Backup password bufferは、借用参照ではなく明示的な所有権移転対象とする。

```text
Caller / UI
    │  request accepted
    ▼
Application / Queue owner
    │  successful handoff
    ▼
ISaveBackupStorage.Create/Restore boundary
    │  ownership transfer
    ▼
Infrastructure Storage owner
    │  finally
    └── ZeroMemory(password)
```

- `EnqueueManualBackupAsync` 等でqueueが正常に受理する前に失敗・キャンセルした場合、caller側が所有権を保持する。
- Queueが受理した後はqueue / worker側が所有し、未処理・破棄・shutdown・次段handoff失敗時は現在のownerがzeroizeする。
- Storageへの呼出しが正常に開始されて所有権が移転した後、Application/Queue側は同一bufferを消去しない。
- この契約は create / restore の双方に適用し、エラー、CancellationToken取消し、例外、queue/write failure、shutdown/discardを含む。

## 2.2 ランサムウェア・シャノンエントロピー急変検知 (Snapshot Freeze)
バックアップ実行時、対象ファイルのシャノン・エントロピーを算出・比較する。
- 過去世代と比較してエントロピーが急激に跳ね上がり（暗号化兆候）、かつファイル構造ヘッダーが破損している場合、ランサムウェア等の攻撃と判定。
- パイプラインは直ちにバックアップおよび自動パージ処理を緊急凍結（Snapshot Freeze）し、既存の正常な過去バックアップが上書き・削除されるのを防止してユーザーに警告を発行する。

---

# 3. パスワード保護 ＆ メモリ常駐排除規約 (M-新規3 是正)

メモリダンプ攻撃や不正プロセスからの盗聴を防ぐため、バックアップパスワードのライフサイクルを以下の通り厳格に統制する。

1. **自動バックアップの非パスワード原則:**  
   ユーザー対話なしで自律実行される自動トリガー（ゲーム終了時、ファイル変更検知時）では、**パスワードをメモリ上に常時常駐・キャッシュすることを厳禁**とする。自動バックアップは Standard ZIP 形式（平文コンテナ ＆ NTFS ACL 保護）で実行される。
2. **明示操作時のオンデマンド受領:**  
   Argon2id + AES-256 によるパスワード暗号化は、以下の「ユーザーが画面上でパスワードを明示入力した操作」に限定する：
   - 画面からの「手動バックアップ実行」
   - 「外部汎用 ZIP エクスポート (`ExportToStandardZipAsync`)」
   - 「PC 移行パッケージ生成 (`09_PC_Migration`)」
3. **使用直後の完全消去 (Zeroization):**  
   `Password` はSave Backup固有のtransfer bufferとして扱う。Application / Queue は自分が所有する間だけ保持し、`ISaveBackupStorage.CreateContainerAsync` / `RestoreContainerAsync` の呼出し境界でStorageへ所有権を移転する。Storageは所有権を取得したbufferを成功・失敗・キャンセル・例外を含む `finally` で物理消去する。
4. **Storage未到達時の消去:**  
   リクエスト拒否、queue投入失敗、未処理要求の破棄、Debounce置換、Shutdown等でStorageへのhandoff前に処理が終了した場合、その時点のApplication/Queue所有者が自身のbufferをzeroizeする。
5. **所有権移転後の操作禁止:**  
   成功したhandoff後、旧所有者は同一 `Password` 配列を読み取り、再試行、再利用、変更、zeroizeしてはならない。
6. **内部copy:**  
   別のpassword bufferを内部生成した場合、そのcopyは生成コンポーネントが所有し、処理完了または中止時に自身でzeroizeする。別所有者のbufferを消去してはならない。

---

# 4. 並行性制御 ＆ ストリーミング I/O 原則

- **最大同時実行数:** 1（システム全体で常に直列実行）。複数ゲームの同時書き込みによる SSD へのランダム I/O 集中やゲーム FPS 低下を防止。
- **物理 I/O の抽象化:** Application 層内で `System.IO` による物理ファイル操作を直接行わず、すべて `ISaveBackupStorage` / `ISaveDataPathScanner` Port へ委譲。

## 4.1 外部ファイル圧縮ソフトウェア連携 (`IExternalArchiverAdapter`)
ゲーム本体バックアップやアーカイブ作成を外部ファイル圧縮ソフトウェアへ委譲する場合：
1. **詳細設定ダイアログ表示:** 誤操作防止のため、「圧縮開始前の詳細設定ダイアログ表示」チェックボックスは **既定 ON** とする。
2. **安全な引数引き渡し:** コマンドライン文字列の直接連結を禁止し、`ProcessStartInfo.ArgumentList` を使用してパスやオプションを安全に渡す。

---

# 5. ゲーム実行中ファイルロック制御 (File Lock Mitigation)

ゲーム実行中やクラウド同期中にセーブデータが排他ロック（`ERROR_SHARING_VIOLATION 0x80070020`）されている場合の対処方針：

1. **共有モード読み取り:** Infrastructure 層のストリームオープン時は常に `FileShare.ReadWrite | FileShare.Delete` を明示。
2. **指数バックオフリトライ:** 共有オープンが拒否された場合、即座に例外で中断せず、指数バックオフ（100ms ➔ 200ms ➔ 400ms、最大 3 回）でリトライ。
3. **安全側スキップ (Fail-Safe):** リトライ上限に達した場合、ゲームの動作を妨害せず、当該セーブデータのバックアップを一時保留（Pending）として記録し、ゲーム終了時の Grace Period 経過後に再実行。

---

# 6. キャンセル制御 (Cancellation Management)

すべての非同期処理メソッドは `CancellationToken` を必須引数とする。

- **中断タイミング:** ファイル単位のコピー境界、または 64KB チャンク読み込みループ内。
- **キャンセル時のクリーンアップ:** キャンセルシグナル受信時、生成途中の `.tmp` 一時ファイルおよび未コミットスナップショットを Infrastructure 層で即座に安全消去（`File.Delete`）し、破損コンテナを残存させない。
- **操作ステータス:** `OperationStatus.Cancelled` として監査ログへ記録。

---

# 7. アプリケーション層エラーハンドリング

| エラー分類 | 発生要因 | システム処理 | ユーザー通知 |
| :--- | :--- | :--- | :--- |
| **StorageUnavailable** | 保存先ドライブの切断、空き容量不足 | パイプライン停止、既存スナップショット保護 | 警告インジケーター表示、保存先確認を案内 |
| **FileAccessLocked** | ゲームによる長期排他ロック | リトライ後スキップ、次回終了時に延期 | （通知抑制・サイレント延期） |
| **IntegrityCheckFailed** | 生成コンテナのハッシュ不一致 | 一時コンテナ消去、失敗スナップショット記録 | 警告通知、手動再スキャンを案内 |
| **OperationCancelled** | ユーザーキャンセル、アプリ終了 | 一時データ消去、正常終了 | （通知不要） |

---

# 8. Application Layer 完全実装 (`SaveBackupPipeline.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Application.Features.SaveBackup;

using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Linq;
using System.Security.Cryptography;
using System.Threading;
using System.Threading.Channels;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Common;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

public sealed record SaveBackupExecutionRequest(
    Guid GameProfileId,
    string GameDisplayName,
    string SourceRootPath,
    string TargetStorageDirectory,
    byte[]? Password,
    bool IsManual
);

public sealed class SaveBackupRequestCoordinator
{
    private readonly Channel<SaveBackupExecutionRequest> _channel;
    private readonly ConcurrentDictionary<Guid, PendingBackupRequest> _debounceBuffer = new();
    private readonly ILogger<SaveBackupRequestCoordinator> _logger;

    private sealed record PendingBackupRequest(
        Guid GameProfileId,
        string GameDisplayName,
        string SourceRootPath,
        string TargetStorageDirectory,
        byte[]? Password,
        DateTimeOffset LastTriggeredUtc
    );

    public SaveBackupRequestCoordinator(ILogger<SaveBackupRequestCoordinator> logger)
    {
        _logger = logger;
        _channel = Channel.CreateBounded<SaveBackupExecutionRequest>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = true,
            SingleWriter = false
        });
    }

    public ChannelReader<SaveBackupExecutionRequest> Reader => _channel.Reader;

    public async Task EnqueueManualBackupAsync(SaveBackupExecutionRequest request, CancellationToken ct = default)
    {
        await _channel.Writer.WriteAsync(request, ct);
    }

    public void DebounceAutoBackupTrigger(Guid gameProfileId, string gameDisplayName, string sourceRoot, string targetDir)
    {
        // 【M-新規3 是正】自動バックアップはパスワードなし (平文ZIP/ACL保護) で登録
        _debounceBuffer.AddOrUpdate(
            gameProfileId,
            new PendingBackupRequest(gameProfileId, gameDisplayName, sourceRoot, targetDir, null, DateTimeOffset.UtcNow),
            (_, old) =>
            {
                // 【M-新規1 是正】置き換えられた古いエントリのパスワードがあれば安全消去
                if (old.Password != null)
                {
                    CryptographicOperations.ZeroMemory(old.Password);
                }
                return new PendingBackupRequest(gameProfileId, gameDisplayName, sourceRoot, targetDir, null, DateTimeOffset.UtcNow);
            }
        );
    }

    public async Task FlushDebouncedRequestsAsync(TimeSpan quietPeriod, CancellationToken ct = default)
    {
        var now = DateTimeOffset.UtcNow;
        var keysToFlush = new List<Guid>();

        foreach (var (profileId, pending) in _debounceBuffer)
        {
            if (now - pending.LastTriggeredUtc >= quietPeriod)
            {
                keysToFlush.Add(profileId);
            }
        }

        foreach (var key in keysToFlush)
        {
            if (_debounceBuffer.TryRemove(key, out var pending))
            {
                var execRequest = new SaveBackupExecutionRequest(
                    pending.GameProfileId,
                    pending.GameDisplayName,
                    pending.SourceRootPath,
                    pending.TargetStorageDirectory,
                    pending.Password,
                    IsManual: false
                );
                await _channel.Writer.WriteAsync(execRequest, ct);
            }
        }
    }
}

public sealed class SaveBackupBackgroundWorker(
    SaveBackupRequestCoordinator coordinator,
    SaveBackupExecutionEngine executionEngine,
    ILogger<SaveBackupBackgroundWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        logger.LogInformation("SaveBackup バックグラウンド直列ワーカーを開始しました (Concurrency: 1)。");

        var flushTask = Task.Run(async () =>
        {
            while (!stoppingToken.IsCancellationRequested)
            {
                try
                {
                    await coordinator.FlushDebouncedRequestsAsync(TimeSpan.FromSeconds(3), stoppingToken);
                    await Task.Delay(1000, stoppingToken);
                }
                catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
                {
                    break;
                }
                catch (Exception ex)
                {
                    logger.LogError(ex, "Debounce バッファのフラッシュ処理中にエラーが発生しました。");
                }
            }
        }, stoppingToken);

        while (await coordinator.Reader.WaitToReadAsync(stoppingToken))
        {
            while (coordinator.Reader.TryRead(out var request))
            {
                if (stoppingToken.IsCancellationRequested) break;

                try
                {
                    logger.LogInformation("バックアップジョブの直列実行を開始: {Game}", request.GameDisplayName);
                    await executionEngine.ExecuteAsync(request, stoppingToken);
                }
                catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
                {
                    break;
                }
                catch (Exception ex)
                {
                    logger.LogError(ex, "バックアップ実行中に予期せぬエラーが発生しました: {Game}", request.GameDisplayName);
                }
            }
        }

        await flushTask;
        logger.LogInformation("SaveBackup バックグラウンド直列ワーカーを停止しました。");
    }
}

public sealed class SaveBackupExecutionEngine(
    ISaveDataPathScanner pathScanner,
    ISaveBackupStorage storage,
    ISaveBackupRepository repository,
    ITamperEvidentAuditLogger auditLogger,
    ILogger<SaveBackupExecutionEngine> logger)
{
    public async Task<SaveBackupResultDto> ExecuteAsync(SaveBackupExecutionRequest request, CancellationToken ct)
    {
        string operationId = $"GST-OP-{DateTimeOffset.UtcNow:yyyyMMdd}-{Guid.NewGuid():N[..8]}";
        string transactionId = $"BAK-{DateTimeOffset.UtcNow:yyyyMMdd}-{Guid.NewGuid():N[..8]}";

        var currentFiles = await pathScanner.ScanSaveDirectoryAsync(request.SourceRootPath, ct);
        if (currentFiles.Count == 0)
        {
            logger.LogWarning("セーブフォルダ内にファイルが見つかりません: {Path}", request.SourceRootPath);
            return SaveBackupResultDto.Failed("セーブファイルが見つかりません。");
        }

        var recentSnapshots = await repository.GetSnapshotsByGameProfileIdAsync(request.GameProfileId, ct);
        var latestSnapshot = recentSnapshots.FirstOrDefault();

        bool hasChanges = true;
        if (latestSnapshot != null)
        {
            long currentTotalSize = currentFiles.Sum(f => f.SizeBytes);
            if (latestSnapshot.FileCount == currentFiles.Count && latestSnapshot.TotalSizeBytes == currentTotalSize)
            {
                hasChanges = false;
            }
        }

        if (!hasChanges)
        {
            logger.LogInformation("セーブデータに変更がないためバックアップをスキップしました: {Game}", request.GameDisplayName);
            return SaveBackupResultDto.Skipped();
        }

        var createRequest = new SaveBackupCreateRequestDto(
            OperationId: operationId,
            TransactionId: transactionId,
            GameProfileId: request.GameProfileId,
            GameDisplayName: request.GameDisplayName,
            Password: request.Password,
            SourceRootPath: request.SourceRootPath,
            TargetStorageDirectory: request.TargetStorageDirectory,
            FilesToBackup: currentFiles
        );

        var containerResult = await storage.CreateContainerAsync(createRequest, null, ct);
        if (!containerResult.Success)
        {
            logger.LogError("バックアップコンテナの生成に失敗しました: {Msg}", containerResult.ErrorMessage);
            await auditLogger.AppendLogAsync(operationId, AuditEventType.SaveBackupFailed, "GST.Worker", request.GameDisplayName, containerResult.ErrorMessage ?? "CreateFailed", ct);
            return SaveBackupResultDto.Failed(containerResult.ErrorMessage ?? "コンテナ作成失敗");
        }

        var snapshotDto = new SaveBackupSnapshotDto(
            BackupId: containerResult.BackupId,
            GameProfileId: request.GameProfileId,
            OperationId: operationId,
            TransactionId: transactionId,
            SnapshotVersion: "1.0",
            CreatedAt: DateTimeOffset.UtcNow,
            FileCount: containerResult.FileCount,
            TotalSizeBytes: containerResult.TotalSizeBytes,
            ManifestHash: containerResult.ManifestHash,
            StoragePath: containerResult.ContainerPath,
            Status: "Completed",
            RetentionStatus: "Active",
            IsPinned: false,
            Tags: [],
            UserNote: null
        );

        await repository.SaveSnapshotAsync(snapshotDto, ct);
        await auditLogger.AppendLogAsync(operationId, AuditEventType.SaveBackupCreated, "GST.Worker", request.GameDisplayName, "Success", ct);

        logger.LogInformation("セーブデータの安全自動バックアップが正常に完了しました: {Game} (BackupId: {Id})", request.GameDisplayName, containerResult.BackupId);
        return SaveBackupResultDto.Succeeded(containerResult.BackupId);
    }
}
```

---

# 9. 単体テスト仕様 (`GST.UnitTests.SaveBackup.Pipeline`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.SaveBackup.Pipeline;

using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using GameSecurityTool.Application.Features.SaveBackup;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.Extensions.Logging.Abstractions;
using Moq;
using Xunit;

public class SaveBackupPipelineTests
{
    [Fact]
    public async Task ExecuteAsync_WhenNoFileChanges_SkipsContainerCreation()
    {
        var mockScanner = new Mock<ISaveDataPathScanner>();
        var mockStorage = new Mock<ISaveBackupStorage>();
        var mockRepo = new Mock<ISaveBackupRepository>();
        var mockAudit = new Mock<ITamperEvidentAuditLogger>();

        var fileMeta = new SaveFileMetadataDto("save.dat", "C:\\Saves\\save.dat", 1024, DateTimeOffset.UtcNow, "ID1", "HASH1");
        mockScanner.Setup(s => s.ScanSaveDirectoryAsync(It.IsAny<string>(), default))
                   .ReturnsAsync([fileMeta]);

        var prevSnapshot = new SaveBackupSnapshotDto(
            Guid.NewGuid(), Guid.NewGuid(), "OP", "TX", "1.0", DateTimeOffset.UtcNow,
            1, 1024, "HASH1", "C:\\Backups\\b.zip", "Completed", "Active", false, [], null);

        mockRepo.Setup(r => r.GetSnapshotsByGameProfileIdAsync(It.IsAny<Guid>(), default))
                .ReturnsAsync([prevSnapshot]);

        var engine = new SaveBackupExecutionEngine(
            mockScanner.Object, mockStorage.Object, mockRepo.Object, mockAudit.Object, NullLogger<SaveBackupExecutionEngine>.Instance);

        var request = new SaveBackupExecutionRequest(Guid.NewGuid(), "Game", "C:\\Saves", "C:\\Backups", null, false);
        var result = await engine.ExecuteAsync(request, default);

        Assert.True(result.IsSkipped);
        mockStorage.Verify(s => s.CreateContainerAsync(It.IsAny<SaveBackupCreateRequestDto>(), null, default), Times.Never);
    }
}
```

---

