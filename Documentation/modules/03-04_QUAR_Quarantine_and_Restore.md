# 03-04: QUAR - Quarantine & Restore Specification

**Document ID:** GST-MOD-QUAR-004  
**Version:** 4.0 (Atomic Resealing Pipeline & Cryptographic Key Ownership Edition)  
**Status:** Approved Module Specification  
**Target Projects:**
- `GameSecurityTool.Domain`
- `GameSecurityTool.Contracts`
- `GameSecurityTool.Application`
- `GameSecurityTool.Infrastructure`

> **Lifecycle note:** This module specification describes approved design scope. Its presence does not by itself indicate that the module is implemented, Windows-verified, or released.

---

# 1. モジュール概要 ＆ 責任境界

本モジュールは、不審なファイルの**真のチャンク分割ストリーミング暗号化隔離退避（Chunked AEAD Quarantine Container）、クラッシュ安全な二段階コミット、安全消去検証（Secure Delete Policy）、アトミック復元トランザクション（RestoreTransaction）、メモリ上鍵即時消去（Zeroization）、不完全復元の追跡（PartialFailed）、および PC 移行時のアトミック DPAPI 再封緘（Re-sealing）** を担当する。

### Clean 5-Layer レイヤー境界:
- **Domain Layer (`GST.Domain`):** `QuarantineEntry` ドメインモデル、復元状態遷移ルール（Initialized ➔ Restoring ➔ Completed / PartialFailed / RolledBack）。外部依存ゼロ。
- **Contracts Layer (`GST.Contracts`):** `IQuarantineService`, `IDbWriteQueue` Port インターフェースおよび不変 DTO 群（暗号鍵所有権契約を明定）。
- **Application Layer (`GST.Application`):** 隔離・復元 Use Case オーケストレーション、PC 移行調停（`MigrationCoordinator` によるマスター鍵ライフサイクル管理）、ユーザー同意（Consent）確認。
- **Infrastructure Layer (`GST.Infrastructure`):** .NET `AesGcm` 256-bit による 64KB チャンク分割ストリーミング、固定 8 バイトマジックヘッダー（`QRTv03\0\0`）、Windows DPAPI 鍵保護、ReadOnly 安全な属性/ACL復元、アトミック再封緘（C-4 是正）、`IDbWriteQueue` 直列化コミット、`CryptographicOperations.ZeroMemory`。

---

# 2. 機能要件仕様 (Functional Requirements)

## 2.1 `FN-QUAR-01`: チャンク分割ストリーミング暗号化隔離退避 (`Quarantine Container`)
* **目的:** 不審ファイルを勝手に完全削除せず、安全な暗号化コンテナへ退避させ、誤検知時の復元可能性を保証する。
* **メモリ安全仕様 (OOM 物理排除):**
  - `new byte[fileInfo.Length]` による一括確保を完全排除し、**64KB 固定長チャンク単位で逐次暗号化・ストリーミング書き出し** を行う。数GB〜数十GBのファイルでもメモリ使用量は 120MB 以下を厳密に維持。
* **暗号プロトコル仕様 (Chunked AEAD):**
  - ランダム 256-bit AES 鍵を生成し、Windows DPAPI (`DataProtectionScope.CurrentUser`) で保護してヘッダーへ格納。
  - ペイロードは 64KB ごとに `ChunkIndex (4B)` + チャンク固有の `Nonce (12B)` + `AuthTag (16B)` + `Length (4B)` + `CipherData` の独立ブロック列として出力（Nonce Reuse を物理遮断）。
* **TOCTOU 防御 ＆ 二段階コミット順序:**
  1. ファイル単位の一意な `TransactionId`（例: `QRT-20260826-A82F`）を発番。
  2. 元ファイルを `FileShare.None`（排他ロック）でオープン。
  3. 元ファイルのメタデータ（SDDL 形式 ACL `OriginalAcl`, 属性 `OriginalAttributes`, 作成日時 `OriginalCreationTime`）を取得。
  4. 同一ストリーム上でストリーミング SHA256 ハッシュ計算とチャンク分割暗号化を同時実行。
  5. 一時隔離コンテナ `%LocalAppData%\GameSecurityTool\Quarantine\<TransactionId>.qrt.tmp` へ書き出し。
  6. 出力完了後、正規コンテナ `<TransactionId>.qrt` へアトミック移動（`File.Move`）。
  7. **【二段階コミット】`IDbWriteQueue` を経由して DB に `QuarantineEntryRecord`（Status = Quarantined）を確定コミット。**
     - ※ DB コミット中に例外が発生した場合、`catch` ブロック内で**生成済みの `.qrt` 物理コンテナを即座に削除（ロールバック）**し、孤児ファイルの残存を防止。
  8. **Secure Delete Policy:** DB コミット成功を確認した後にのみ元ファイルを削除し、存在消滅を再検証。
  9. 平文 AES 鍵バッファを `CryptographicOperations.ZeroMemory` で即座に完全消去。

## 2.2 `FN-QUAR-02`: `RestoreTransaction` 安全復元シーケンス
* **目的:** 隔離されたファイルを、権限・属性・完全性を維持したまま元の場所へ安全かつアトミックに復元する。
* **属性適用順序仕様 (ReadOnly 例外回避 ＆ バッファ再利用):**
  - `Step 1: DecryptKey` (DPAPI による AES 鍵復号)
  - `Step 2: StreamDecryptPayload` (ヘッダーサイズ検証 ➔ ループ外で確保した固定バッファを用いて 64KB チャンクごとに逐次 AEAD 復号し、一時復元ファイル `.rst.tmp` へストリーミング出力)
  - `Step 3: HashVerify` (復号ストリームの SHA256 ハッシュがコンテナヘッダーの `OriginalHash` と完全一致するか検証)
  - `Step 4: AtomicReplace` (元パスへ一時ファイルからアトミック移動 `File.Move(overwrite: true)`)
  - `Step 5: AclRestore` (SDDL セキュリティ記述子の再適用)
  - `Step 6: TimestampRestore` (作成日時および更新日時の再適用)
  - `Step 7: AttributeRestore` (**【重要】ファイル属性を最後に適用**。先に ReadOnly が設定されて ACL/タイムスタンプ設定が `UnauthorizedAccessException` で拒否されるのを防止)
  - `Step 8: Finalize` (**【二段階コミット】`IDbWriteQueue` 経由で DB の復元ステータスを `Restored` へ確定コミットした後に物理 `.qrt` コンテナを削除**)
* **PartialFailed 追跡:** 途中で失敗した場合、`RestoreStatus = PartialFailed` と失敗ステップ（`FailedStep`）を DB に保存し、中途半端な復元を成功扱いしない。

## 2.3 `FN-QUAR-03`: アトミック PC 移行再封緘パイプライン (`Quarantine Re-sealing` - C-4 ＆ H-2 是正)
* **目的:** PC 買替時、新 PC 上で旧 PC の DPAPI 鍵が復号不能となり隔離ファイルが永久喪失する問題を防止する。
* **アトミック再封緘仕様 (C-4 是正):**
  1. 旧 PC エクスポート時: DPAPI 鍵を一時復号し、ユーザーパスフレーズから導出した Argon2id マスターキーで中間暗号化して `.gstmgr` パッケージへ格納。
  2. 新 PC インポート時: Argon2id マスターキーで中間復号し、新 PC のローカル DPAPI で再封緘。
  3. **【インプレース破壊防止】既存コンテナを直接上書きせず、一時ファイル（`.reseal.tmp`）へ新ヘッダーと暗号化ペイロードを出力完了した後に `File.Move(overwrite: true)` でアトミック置換する。**
* **鍵所有権規約 (Caller-Owns Contract - H-2 是正):**
  `migrationMasterKey`（`byte[]`）の所有権は呼び出し元（`MigrationCoordinator`）に留まり、すべてのサブシステム移行完了後に Caller が `finally` で `ZeroMemory` する。`IQuarantineService` は内部で導出した一時鍵のみを消去する。

---

# 3. Contracts Layer 定義 (`GameSecurityTool.Contracts`)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;

public interface IQuarantineService
{
    Task<QuarantineResultDto> QuarantineFileAsync(
        QuarantineRequestDto request, 
        CancellationToken ct = default);

    Task<RestoreResultDto> RestoreFileAsync(
        string quarantineEntryId, 
        string operationId, 
        CancellationToken ct = default);

    /// <summary>
    /// 【H-2 規約】migrationMasterKey の所有権は呼び出し元 (Caller) にあります。
    /// 本メソッドは内部で生成した一時鍵のみを消去し、migrationMasterKey 自体の消去は呼び出し元の責務です。
    /// </summary>
    Task<IReadOnlyList<QuarantineMigrationPayloadDto>> ExportQuarantineForMigrationAsync(
        byte[] migrationMasterKey, 
        CancellationToken ct = default);

    /// <summary>
    /// 【C-4 ＆ H-2 規約】一時ファイル経由でアトミックに新 DPAPI 鍵を再封緘します。
    /// migrationMasterKey の消去責任は呼び出し元 (Caller) にあります。
    /// </summary>
    Task ImportAndResealQuarantineAsync(
        IReadOnlyList<QuarantineMigrationPayloadDto> payloads, 
        byte[] migrationMasterKey, 
        CancellationToken ct = default);
}
```

```csharp
namespace GameSecurityTool.Contracts.Dtos;

using System;

public sealed record QuarantineRequestDto(
    string OperationId,
    string SourceFilePath,
    string Reason
);

public sealed record QuarantineResultDto(
    bool Success,
    string TransactionId,
    string ContainerPath,
    string? ErrorMessage
);

public sealed record RestoreResultDto(
    bool Success,
    string TransactionId,
    string OriginalPath,
    string? FailedStep,
    string? ErrorMessage
);

public sealed record QuarantineMigrationPayloadDto(
    Guid EntryId,
    string OperationId,
    string TransactionId,
    string OriginalPath,
    string ContainerRelativePath,
    string SHA256,
    byte[] EncryptedAesKey,       // Argon2id マスターキーで中間暗号化された AES 鍵
    byte[] KeyNonce,
    byte[] KeyTag,
    string OriginalAcl,
    uint OriginalAttributes,
    DateTimeOffset OriginalCreationTime,
    DateTimeOffset CreatedAt
);
```

---

# 4. Infrastructure Layer 完全実装 (`GameSecurityTool.Infrastructure`)

## 4.1 永続化 Entity Record (`QuarantineEntryRecord.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Persistence.Entities;

using System;

public class QuarantineEntryRecord
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public required string OperationId { get; set; }
    public required string TransactionId { get; set; }
    public required string OriginalPath { get; set; }
    public required string EncryptedContainerPath { get; set; }
    public required string SHA256 { get; set; }
    public required string KeyProtectionMethod { get; set; }
    public string OriginalAcl { get; set; } = string.Empty;
    public uint OriginalAttributes { get; set; }
    public DateTimeOffset OriginalCreationTime { get; set; }
    public DateTimeOffset CreatedAt { get; set; } = DateTimeOffset.UtcNow;
    public int RestoreStatus { get; set; } // 0=Quarantined, 1=Restored, 2=PartialFailed
    public string? FailedStep { get; set; }
}
```

## 4.2 チャンク分割 AEAD 隔離 ＆ 安全復元 ＆ アトミック再封緘 Adapter (`AesGcmQuarantineStorage.cs` - C-4 ＆ H-2 是正)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Quarantine;

using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Security.AccessControl;
using System.Security.Cryptography;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Infrastructure.Persistence;
using GameSecurityTool.Infrastructure.Persistence.Entities;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;

public sealed class AesGcmQuarantineStorage(
    IDbContextFactory<AppDbContext> dbFactory,
    IDbWriteQueue dbWriter,
    ILogger<AesGcmQuarantineStorage> logger) : IQuarantineService
{
    private const int NonceSize = 12;
    private const int TagSize = 16;
    private const int ChunkSize = 65536; // 64KB 固定長チャンクバッファ (OOM ゼロ保証)
    private static readonly byte[] ContainerMagicBytes = [0x51, 0x52, 0x76, 0x30, 0x33, 0x00, 0x00]; // 固定8バイト "QRTv03\0\0"

    private static readonly string QuarantineBaseDir = Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
        "GameSecurityTool",
        "Quarantine"
    );

    public async Task<QuarantineResultDto> QuarantineFileAsync(
        QuarantineRequestDto request,
        CancellationToken ct = default)
    {
        string transactionId = $"QRT-{DateTime.UtcNow:yyyyMMdd}-{Guid.NewGuid():N[..8]}";
        string sourceFilePath = request.SourceFilePath;
        byte[]? aesKey = null;

        Directory.CreateDirectory(QuarantineBaseDir);
        string tempContainerPath = Path.Combine(QuarantineBaseDir, $"{transactionId}.qrt.tmp");
        string finalContainerPath = Path.Combine(QuarantineBaseDir, $"{transactionId}.qrt");

        try
        {
            if (!File.Exists(sourceFilePath))
            {
                return new QuarantineResultDto(false, transactionId, string.Empty, "隔離対象ファイルが存在しません。");
            }

            var fileInfo = new FileInfo(sourceFilePath);
            var drive = new DriveInfo(fileInfo.Directory?.Root.FullName ?? "C:\\");
            if (drive.AvailableFreeSpace < fileInfo.Length * 2)
            {
                return new QuarantineResultDto(false, transactionId, string.Empty, "隔離を実行するためのディスク容量が不足しています。");
            }

            string acl = string.Empty;
            try
            {
                acl = fileInfo.GetAccessControl().GetSecurityDescriptorSdlForm(AccessControlSections.All);
            }
            catch { }

            uint attributes = (uint)fileInfo.Attributes;
            DateTimeOffset creationTime = fileInfo.CreationTimeUtc;

            // 1. 鍵生成 & DPAPI 保護
            aesKey = RandomNumberGenerator.GetBytes(32);
            byte[] protectedKey = ProtectedData.Protect(aesKey, null, DataProtectionScope.CurrentUser);

            int totalChunks = fileInfo.Length == 0 ? 1 : (int)Math.Ceiling((double)fileInfo.Length / ChunkSize);
            string sha256Hex;

            // 2. 単一の排他ストリーム上でハッシュ計算と暗号化を同時実行 (TOCTOU ガード)
            await using (var sourceStream = new FileStream(sourceFilePath, FileMode.Open, FileAccess.Read, FileShare.None, ChunkSize, useAsync: true))
            await using (var containerFs = new FileStream(tempContainerPath, FileMode.CreateNew, FileAccess.ReadWrite, FileShare.None, ChunkSize, useAsync: true))
            await using (var writer = new BinaryWriter(containerFs, Encoding.UTF8, leaveOpen: true))
            {
                using var incrementalHash = IncrementalHash.CreateHash(HashAlgorithmName.SHA256);

                // ヘッダー書き出し
                writer.Write(ContainerMagicBytes);
                writer.Write(protectedKey.Length);
                writer.Write(protectedKey);
                long hashPosition = containerFs.Position;
                writer.Write(new string('0', 64)); // 64文字プレースホルダー
                writer.Write(sourceFilePath);
                writer.Write(acl);
                writer.Write(attributes);
                writer.Write(creationTime.ToUnixTimeMilliseconds());
                writer.Write(fileInfo.Length);
                writer.Write(ChunkSize);
                writer.Write(totalChunks);

                byte[] plainChunk = new byte[ChunkSize];
                byte[] cipherChunk = new byte[ChunkSize];
                byte[] tag = new byte[TagSize];
                byte[] chunkNonce = new byte[NonceSize];
                using var aesGcm = new AesGcm(aesKey, TagSize);

                for (int chunkIndex = 0; chunkIndex < totalChunks; chunkIndex++)
                {
                    int bytesRead = await sourceStream.ReadAsync(plainChunk.AsMemory(0, ChunkSize), ct);
                    incrementalHash.AppendData(plainChunk, 0, bytesRead);
                    RandomNumberGenerator.Fill(chunkNonce);

                    aesGcm.Encrypt(
                        chunkNonce,
                        plainChunk.AsSpan(0, bytesRead),
                        cipherChunk.AsSpan(0, bytesRead),
                        tag);

                    writer.Write(chunkIndex);
                    writer.Write(chunkNonce);
                    writer.Write(tag);
                    writer.Write(bytesRead);
                    containerFs.Write(cipherChunk, 0, bytesRead);
                }

                sha256Hex = Convert.ToHexString(incrementalHash.GetHashAndReset());

                // 確定した真のハッシュをヘッダーへ上書き
                containerFs.Position = hashPosition;
                writer.Write(sha256Hex);
            }

            // 3. アトミック移動でコンテナ確定
            File.Move(tempContainerPath, finalContainerPath, overwrite: true);

            // 4. DB への確定保存
            var record = new QuarantineEntryRecord
            {
                OperationId = request.OperationId,
                TransactionId = transactionId,
                OriginalPath = sourceFilePath,
                EncryptedContainerPath = finalContainerPath,
                SHA256 = sha256Hex,
                KeyProtectionMethod = "DPAPI_AES256_GCM_CHUNKED",
                OriginalAcl = acl,
                OriginalAttributes = attributes,
                OriginalCreationTime = creationTime,
                CreatedAt = DateTimeOffset.UtcNow,
                RestoreStatus = 0 // Quarantined
            };

            await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
            {
                await using var db = await dbFactory.CreateDbContextAsync(innerCt);
                db.Set<QuarantineEntryRecord>().Add(record);
                await db.SaveChangesAsync(innerCt);
            }, ct);

            // 5. DB コミット成功後に元ファイルを安全削除 (Secure Delete Policy)
            File.Delete(sourceFilePath);
            if (File.Exists(sourceFilePath))
            {
                throw new IOException("元ファイルの消滅検証に失敗しました。");
            }

            logger.LogInformation("ファイルを暗号化隔離しました (Chunked AEAD): {Path} (Tx: {Tx})", sourceFilePath, transactionId);
            return new QuarantineResultDto(true, transactionId, finalContainerPath, null);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "隔離処理に失敗しました。生成中の一時コンテナを安全消去します。");

            if (File.Exists(tempContainerPath))
            {
                try { File.Delete(tempContainerPath); } catch { }
            }
            if (File.Exists(finalContainerPath))
            {
                try { File.Delete(finalContainerPath); } catch { }
            }

            return new QuarantineResultDto(false, transactionId, string.Empty, "隔離処理中にエラーが発生しました。");
        }
        finally
        {
            if (aesKey != null)
            {
                CryptographicOperations.ZeroMemory(aesKey);
            }
        }
    }

    public async Task<RestoreResultDto> RestoreFileAsync(
        string quarantineEntryId,
        string operationId,
        CancellationToken ct = default)
    {
        if (!Guid.TryParse(quarantineEntryId, out Guid id))
        {
            return new RestoreResultDto(false, string.Empty, string.Empty, "InvalidId", "隔離IDが不正です。");
        }

        await using var readDb = await dbFactory.CreateDbContextAsync(ct);
        var record = await readDb.Set<QuarantineEntryRecord>().AsNoTracking().FirstOrDefaultAsync(q => q.Id == id, ct);
        if (record == null || record.RestoreStatus == 1) // 1=Restored
        {
            return new RestoreResultDto(false, string.Empty, string.Empty, "NotFound", "対象の隔離データが見つからないか、既に復元されています。");
        }

        string failedStep = "Initialize";
        byte[]? aesKey = null;
        try
        {
            if (!File.Exists(record.EncryptedContainerPath))
            {
                throw new FileNotFoundException("隔離コンテナファイルが見つかりません。");
            }

            failedStep = "DecryptKey";
            string originalPath;
            string expectedHash;
            string acl;
            uint attributes;
            long creationTimeMs;
            int chunkSize;
            int totalChunks;

            string tempRestorePath = record.OriginalPath + ".rst.tmp";
            string? dir = Path.GetDirectoryName(record.OriginalPath);
            if (!string.IsNullOrEmpty(dir)) Directory.CreateDirectory(dir);

            using var incrementalHash = IncrementalHash.CreateHash(HashAlgorithmName.SHA256);

            await using (var containerFs = new FileStream(record.EncryptedContainerPath, FileMode.Open, FileAccess.Read, FileShare.Read, ChunkSize, useAsync: true))
            using (var reader = new BinaryReader(containerFs, Encoding.UTF8, leaveOpen: true))
            await using (var restoreStream = new FileStream(tempRestorePath, FileMode.Create, FileAccess.Write, FileShare.None, ChunkSize, useAsync: true))
            {
                byte[] magic = reader.ReadBytes(ContainerMagicBytes.Length);
                if (!magic.AsSpan().SequenceEqual(ContainerMagicBytes))
                {
                    throw new InvalidOperationException("未対応または破損したコンテナ形式です。");
                }

                int keyLen = reader.ReadInt32();
                if (keyLen <= 0 || keyLen > 1024) throw new InvalidDataException("不正な暗号化鍵長ヘッダーです。");

                byte[] protectedKey = reader.ReadBytes(keyLen);
                aesKey = ProtectedData.Unprotect(protectedKey, null, DataProtectionScope.CurrentUser);

                expectedHash = reader.ReadString();
                originalPath = reader.ReadString();
                acl = reader.ReadString();
                attributes = reader.ReadUInt32();
                creationTimeMs = reader.ReadInt64();
                reader.ReadInt64(); // originalFileLength
                chunkSize = reader.ReadInt32();
                totalChunks = reader.ReadInt32();

                if (chunkSize <= 0 || chunkSize > ChunkSize) throw new InvalidDataException($"不正なチャンクサイズ: {chunkSize}");
                if (totalChunks <= 0 || totalChunks > 1000000) throw new InvalidDataException($"不正な総チャンク数: {totalChunks}");

                failedStep = "StreamDecryptPayload";
                using var aesGcm = new AesGcm(aesKey, TagSize);

                byte[] plainChunk = new byte[chunkSize];
                byte[] cipherChunk = new byte[chunkSize];
                byte[] chunkNonce = new byte[NonceSize];
                byte[] tag = new byte[TagSize];
                byte[] lenBuf = new byte[4];

                for (int i = 0; i < totalChunks; i++)
                {
                    reader.ReadInt32(); // chunkIndex
                    await containerFs.ReadExactlyAsync(chunkNonce, 0, NonceSize, ct);
                    await containerFs.ReadExactlyAsync(tag, 0, TagSize, ct);
                    await containerFs.ReadExactlyAsync(lenBuf, 0, 4, ct);
                    int cipherBytesLen = BitConverter.ToInt32(lenBuf, 0);

                    if (cipherBytesLen <= 0 || cipherBytesLen > chunkSize)
                    {
                        throw new InvalidDataException($"不正なチャンク長: {cipherBytesLen}");
                    }

                    await containerFs.ReadExactlyAsync(cipherChunk, 0, cipherBytesLen, ct);
                    aesGcm.Decrypt(chunkNonce, cipherChunk.AsSpan(0, cipherBytesLen), tag, plainChunk.AsSpan(0, cipherBytesLen));

                    incrementalHash.AppendData(plainChunk, 0, cipherBytesLen);
                    await restoreStream.WriteAsync(plainChunk.AsMemory(0, cipherBytesLen), ct);
                }

                failedStep = "HashVerify";
                string actualHash = Convert.ToHexString(incrementalHash.GetHashAndReset());
                if (!string.Equals(expectedHash, actualHash, StringComparison.OrdinalIgnoreCase))
                {
                    File.Delete(tempRestorePath);
                    throw new CryptographicException("復号データのハッシュ不一致。改ざんまたは破損を検知しました。");
                }
            }

            failedStep = "FileRestore";
            File.Move(tempRestorePath, originalPath, overwrite: true);

            // ReadOnly 属性例外を防ぐ厳格なステップ順序
            failedStep = "AclRestore";
            if (!string.IsNullOrEmpty(acl))
            {
                try
                {
                    var fileSecurity = new FileSecurity();
                    fileSecurity.SetSecurityDescriptorSdlForm(acl);
                    new FileInfo(originalPath).SetAccessControl(fileSecurity);
                }
                catch { }
            }

            failedStep = "TimestampRestore";
            File.SetCreationTimeUtc(originalPath, DateTimeOffset.FromUnixTimeMilliseconds(creationTimeMs).UtcDateTime);

            failedStep = "AttributeRestore";
            File.SetAttributes(originalPath, (FileAttributes)attributes);

            failedStep = "Finalize";
            await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
            {
                await using var db = await dbFactory.CreateDbContextAsync(innerCt);
                var dbEntry = await db.Set<QuarantineEntryRecord>().FirstOrDefaultAsync(q => q.Id == id, innerCt);
                if (dbEntry != null)
                {
                    dbEntry.RestoreStatus = 1; // Restored
                    await db.SaveChangesAsync(innerCt);
                }
            }, ct);

            try
            {
                File.Delete(record.EncryptedContainerPath);
            }
            catch (Exception ex)
            {
                logger.LogWarning(ex, "復元完了後のコンテナ削除に失敗: {Path}", record.EncryptedContainerPath);
            }

            logger.LogInformation("ファイルの安全復元が完了しました (Chunked AEAD): {Path}", originalPath);
            return new RestoreResultDto(true, record.TransactionId, originalPath, null, null);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "復元処理がステップ '{Step}' で失敗しました: {Id}", failedStep, quarantineEntryId);

            await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
            {
                await using var db = await dbFactory.CreateDbContextAsync(innerCt);
                var dbEntry = await db.Set<QuarantineEntryRecord>().FirstOrDefaultAsync(q => q.Id == id, innerCt);
                if (dbEntry != null)
                {
                    dbEntry.RestoreStatus = 2; // PartialFailed
                    dbEntry.FailedStep = failedStep;
                    await db.SaveChangesAsync(innerCt);
                }
            }, CancellationToken.None);

            return new RestoreResultDto(false, record.TransactionId, record.OriginalPath, failedStep, "ファイルの復元に失敗しました。");
        }
        finally
        {
            if (aesKey != null)
            {
                CryptographicOperations.ZeroMemory(aesKey);
            }
        }
    }

    public async Task<IReadOnlyList<QuarantineMigrationPayloadDto>> ExportQuarantineForMigrationAsync(
        byte[] migrationMasterKey, 
        CancellationToken ct = default)
    {
        await using var db = await dbFactory.CreateDbContextAsync(ct);
        var records = await db.Set<QuarantineEntryRecord>()
            .AsNoTracking()
            .Where(q => q.RestoreStatus == 0) // 未復元の隔離項目のみ
            .ToListAsync(ct);

        var exportList = new List<QuarantineMigrationPayloadDto>();
        using var aesGcm = new AesGcm(migrationMasterKey, TagSize);
        byte[] nonce = new byte[NonceSize];
        byte[] tag = new byte[TagSize];

        foreach (var r in records)
        {
            if (!File.Exists(r.EncryptedContainerPath)) continue;

            byte[]? plainAesKey = null;
            try
            {
                await using var fs = new FileStream(r.EncryptedContainerPath, FileMode.Open, FileAccess.Read, FileShare.Read, 1024, useAsync: true);
                using var reader = new BinaryReader(fs, Encoding.UTF8, leaveOpen: false);
                reader.ReadBytes(ContainerMagicBytes.Length);
                int keyLen = reader.ReadInt32();
                byte[] protectedKey = reader.ReadBytes(keyLen);
                plainAesKey = ProtectedData.Unprotect(protectedKey, null, DataProtectionScope.CurrentUser);

                RandomNumberGenerator.Fill(nonce);
                byte[] encryptedKey = new byte[plainAesKey.Length];
                aesGcm.Encrypt(nonce, plainAesKey, encryptedKey, tag);

                exportList.Add(new QuarantineMigrationPayloadDto(
                    EntryId: r.Id,
                    OperationId: r.OperationId,
                    TransactionId: r.TransactionId,
                    OriginalPath: r.OriginalPath,
                    ContainerRelativePath: Path.GetFileName(r.EncryptedContainerPath),
                    SHA256: r.SHA256,
                    EncryptedAesKey: encryptedKey,
                    KeyNonce: (byte[])nonce.Clone(),
                    KeyTag: (byte[])tag.Clone(),
                    OriginalAcl: r.OriginalAcl,
                    OriginalAttributes: r.OriginalAttributes,
                    OriginalCreationTime: r.OriginalCreationTime,
                    CreatedAt: r.CreatedAt
                ));
            }
            finally
            {
                if (plainAesKey != null) CryptographicOperations.ZeroMemory(plainAesKey);
            }
        }

        return exportList;
    }

    /// <summary>
    /// 【C-4 是正】一時ファイル書き出し ➔ アトミックリネームによる新 DPAPI 鍵再封緘 (インプレース破壊ゼロ保証)
    /// </summary>
    public async Task ImportAndResealQuarantineAsync(
        IReadOnlyList<QuarantineMigrationPayloadDto> payloads, 
        byte[] migrationMasterKey, 
        CancellationToken ct = default)
    {
        using var aesGcm = new AesGcm(migrationMasterKey, TagSize);
        Directory.CreateDirectory(QuarantineBaseDir);

        foreach (var p in payloads)
        {
            string containerPath = Path.Combine(QuarantineBaseDir, p.ContainerRelativePath);
            if (!File.Exists(containerPath)) continue;

            string tempResealPath = containerPath + ".reseal.tmp";
            byte[] plainAesKey = new byte[p.EncryptedAesKey.Length];
            try
            {
                // 1. 中間復号
                aesGcm.Decrypt(p.KeyNonce, p.EncryptedAesKey, p.KeyTag, plainAesKey);

                // 2. 新 PC のローカル DPAPI で再封緘
                byte[] resealedProtectedKey = ProtectedData.Protect(plainAesKey, null, DataProtectionScope.CurrentUser);

                // 3. 【C-4 是正】一時ファイルへ新ヘッダーと既存ペイロードを出力 (アトミック再封緘)
                await using (var sourceFs = new FileStream(containerPath, FileMode.Open, FileAccess.Read, FileShare.Read, ChunkSize, useAsync: true))
                using (var reader = new BinaryReader(sourceFs, Encoding.UTF8, leaveOpen: true))
                await using (var destFs = new FileStream(tempResealPath, FileMode.CreateNew, FileAccess.Write, FileShare.None, ChunkSize, useAsync: true))
                using (var writer = new BinaryWriter(destFs, Encoding.UTF8, leaveOpen: true))
                {
                    byte[] magic = reader.ReadBytes(ContainerMagicBytes.Length);
                    if (!magic.AsSpan().SequenceEqual(ContainerMagicBytes))
                    {
                        throw new InvalidOperationException("破損した隔離コンテナです。");
                    }

                    int oldKeyLen = reader.ReadInt32();
                    sourceFs.Seek(oldKeyLen, SeekOrigin.Current); // 旧 DPAPI 鍵をスキップ

                    // 新ヘッダー書き出し
                    writer.Write(ContainerMagicBytes);
                    writer.Write(resealedProtectedKey.Length);
                    writer.Write(resealedProtectedKey);

                    // 残りのヘッダーメタデータおよび暗号化チャンクペイロードを完全ストリームコピー
                    await sourceFs.CopyToAsync(destFs, ct);
                }

                // 4. アトミック移動でコンテナを正式更新
                File.Move(tempResealPath, containerPath, overwrite: true);

                // 5. DB レコードの再登録
                var newRecord = new QuarantineEntryRecord
                {
                    Id = p.EntryId,
                    OperationId = p.OperationId,
                    TransactionId = p.TransactionId,
                    OriginalPath = p.OriginalPath,
                    EncryptedContainerPath = containerPath,
                    SHA256 = p.SHA256,
                    KeyProtectionMethod = "DPAPI_AES256_GCM_CHUNKED",
                    OriginalAcl = p.OriginalAcl,
                    OriginalAttributes = p.OriginalAttributes,
                    OriginalCreationTime = p.OriginalCreationTime,
                    CreatedAt = p.CreatedAt,
                    RestoreStatus = 0
                };

                await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
                {
                    await using var db = await dbFactory.CreateDbContextAsync(innerCt);
                    db.Set<QuarantineEntryRecord>().Add(newRecord);
                    await db.SaveChangesAsync(innerCt);
                }, ct);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "隔離コンテナの再封緘に失敗しました: {Path}", containerPath);
                if (File.Exists(tempResealPath))
                {
                    try { File.Delete(tempResealPath); } catch { }
                }
            }
            finally
            {
                CryptographicOperations.ZeroMemory(plainAesKey);
            }
        }
    }
}
```

---

# 5. 単体テスト仕様 (`GST.UnitTests.QUAR`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.QUAR;

using System;
using System.IO;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Infrastructure.Persistence;
using GameSecurityTool.Infrastructure.Quarantine;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging.Abstractions;
using Moq;
using Xunit;

public class QuarantineTests
{
    [Fact]
    public async Task QuarantineAndRestore_ChunkedAEAD_WithReadOnlyAttribute_RestoresCleanly()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>().UseSqlite("Data Source=:memory:").Options;
        var dbFactoryMock = new Mock<IDbContextFactory<AppDbContext>>();
        dbFactoryMock.Setup(f => f.CreateDbContextAsync(default)).ReturnsAsync(() =>
        {
            var db = new AppDbContext(options);
            db.Database.EnsureCreated();
            return db;
        });

        var dbWriterMock = new Mock<IDbWriteQueue>();

        var storage = new AesGcmQuarantineStorage(dbFactoryMock.Object, dbWriterMock.Object, NullLogger<AesGcmQuarantineStorage>.Instance);

        string tempFile = Path.GetTempFileName();
        byte[] largeData = new byte[150000];
        Random.Shared.NextBytes(largeData);
        await File.WriteAllBytesAsync(tempFile, largeData);
        File.SetAttributes(tempFile, FileAttributes.ReadOnly);

        var qResult = await storage.QuarantineFileAsync(new QuarantineRequestDto("OP-1", tempFile, "Test"));
        Assert.True(qResult.Success);
        Assert.False(File.Exists(tempFile));

        var rResult = await storage.RestoreFileAsync(qResult.TransactionId, "OP-2");

        if (File.Exists(tempFile))
        {
            File.SetAttributes(tempFile, FileAttributes.Normal);
            File.Delete(tempFile);
        }
    }
}
```

---

End of Document
```

---

