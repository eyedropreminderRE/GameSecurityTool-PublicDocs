# GameSecurityTool Save Backup Storage & Encryption Specification

**Document ID:** GST-FEAT-SAVE-STORAGE-004  
**Version:** 4.4
**Status:** Approved Infrastructure Specification  
**Target Layer:** Infrastructure Layer (`GameSecurityTool.Infrastructure`)  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Standard ZIP Backup Container 構造 (No Vendor Lock-In)

Save Backup は、GST 専用の独自バイナリコンテナではなく、**長期的可搬性・災害復旧（DR）・外側コンテナの外部ツール展開性（7-Zip / Explorer）** を確保する **Standard ZIP互換アーカイブ形式** を採用する。Standard ZIPであることは、内部の独自暗号化payloadが汎用ZIPツールだけで復号できることを意味しない。

## 1.1 アーカイブ物理レイアウト

```text
Backup_<GameName>_<yyyyMMdd-HHmmss>_<ShortId>.zip
├── GST_Header.json                   <-- 最小の非秘密format/recovery metadata
├── GST_Integrity.json                <-- KDF parameters / salt / cipher / envelope metadata
├── GST_Manifest.json.enc             <-- 暗号化・認証済みCanonical Manifest
└── Payload/                          <-- 不透明entry IDのみ。元ファイル名/相対パスをentry名へ露出しない
    ├── F0000000.bin                  <-- Chunked AEAD record stream
    ├── F0000001.bin
    └── ...
```

### 1.3 外部復旧可搬性の契約 (approved portability and security decision)

- 外側のバックアップコンテナはStandard ZIP互換とし、汎用ZIPツールでZIP構造および非暗号化エントリを検査・展開できる。
- Secure BackupのpayloadにはVersioned `AES-256-GCM-CHUNKED` envelopeを使用する。汎用ZIPツール単独でこのpayloadを平文へ復号することは保証しない。
- Secure BackupはNULL/空パスワードでは作成せず、暗号化を必須とする。平文エクスポートは別途明示された機能でない限り存在しない。
- `GST_Integrity.json` に記録されたArgon2id parametersから32-byte (256-bit) master keyを導出する。KDF parameters、salt、cipher、EnvelopeVersionは秘密情報ではないが、認証コンテキストへ結合する。
- 各payload chunkの物理構造は `[Nonce(12B)][Tag(16B)][Len(4B LE)][Ciphertext(LenB)]`。Nonceはchunkごとに新規生成する。
- 各chunkのGCM AADは、EnvelopeVersion、BackupId、canonical normalized RelativePath、stable FileId、chunk index、plaintext chunk length、canonical ManifestDigestを長さ付き決定論的binary encodingで構成する。
- `GST_Manifest.json.enc` はcanonical manifest JSONを同じmaster keyで独立にAES-256-GCM認証暗号化する。ManifestDigestはcanonical plaintextのSHA-256である。
- `GST_Header.json` と `GST_Integrity.json` のcanonical bytesはManifest認証コンテキストへ結合され、改ざん時はManifest認証に失敗する。HeaderにはGameName等の不要な個人/環境情報を含めない。未知のEnvelopeVersion / cipher / KDF parametersはfail-closedで拒否する。
- DB側のExpectedManifestHashは利用可能な場合の追加の外部integrity anchorであり、DBを失った独立復旧でもManifest自身の暗号学的認証を必須とする。
- Manifest/metadata認証に失敗した状態でユーザーデータを置換してはならない。
- 独立復旧実装は、GST本体のDB/DPAPI状態なしに、公開されたVersioned Envelope仕様と正しいパスワードだけから復号・復元できるものとする。approved portability and security decisionは新しいRecovery Toolの実装・同梱を決定しない。

### 1.2 ファイル命名規則 (Human-Readable Naming)
DB が失われた状態での復旧性を担保するため、以下の命名規則を採用する。
- 形式: `Backup_<SanitizedGameName>_<yyyyMMdd_HHmmss>_<BackupId先頭8桁>.zip`
- 例: `Backup_Cyberpunk2077_20260827_143000_a82f91c0.zip`

---

# 2. 暗号化 ＆ ポータブルパスワード保護モデル

## 2.1 暗号化パイプライン (Argon2id + AES-256-GCM-CHUNKED - CRIT-06 & M2 是正)

メモリダンプからの生パスワード漏洩を防ぐため、パスワードは .NET の `string` ではなく `byte[]` として引き回す。approved portability and security decisionにより、Save Backupの `Password` はborrowed bufferではなくtransfer bufferとして扱う。Application / Queueで保持される間はその時点のownerが管理し、`ISaveBackupStorage.CreateContainerAsync` / `RestoreContainerAsync` の呼出し境界でInfrastructure Storageへ所有権を移転する。Storageは所有権を取得したbufferを処理完了・失敗・CancellationToken取消し・例外を含む `finally` で `CryptographicOperations.ZeroMemory` / `Array.Clear` 等により物理消去する。

```text
[ ユーザー指定パスワード (byte[]? Password) ]
                    │
                    ▼
[ Argon2id 鍵導出: Iterations=3 / Memory=64MB / Parallelism=4 / Salt (16B) ]
                    │
                    ▼
          [ 256-bit Master AES Key ]
                    │
        ┌───────────┴───────────┐ (※ 導出完了後、即座に Password および Master Key バッファを ZeroMemory 消去)
        ▼                       ▼
[ 64KB チャンク別 AEAD 暗号化 ]   [ GST_Integrity.json 出力 ]
- チャンク固有 Nonce (12B)       - Salt (Hex)
- AuthTag (16B)                 - Iterations (3)
- チャンク長 Length (4B)         - MemorySizeKB (65536)
- AES-GCM 暗号文                 - DegreeOfParallelism (4)
                                - PayloadCipher ("AES-256-GCM-CHUNKED")
                                - ChunkSize (65536)
```

1. **完全ストリーミング暗号化 (OOM ゼロ保証):**
   64KB 固定長バッファによる `FileStream` ➔ `AesGcm` ➔ `ZipArchiveEntry` のパイプラインにより、巨大ファイルでもメモリ使用量は 120MB 以下を厳密に維持。
2. **メモリ上の秘密情報即時消去 (Zeroization):**
   鍵導出・暗号化・復号の完了後、一時的なマスター鍵バッファを `finally` で消去する。呼び出し元からtransferされたパスワードbufferについても、Storageが所有権を取得した後、`finally` で消去する。
3. **所有権境界:**
   Storageはcallerから所有権移転済みの `Password` のみをzeroizeする。Storageへ到達しない処理の中止やApplication/Queue内の破棄・失敗時は、その時点のownerが自身のbufferをzeroizeする。別途生成したinternal copyは生成元ownerがzeroizeする。

---

# 3. ディスク容量事前検証 (Storage Safety Guard)

書き込み処理によるディスク枯渇事故を物理的に防ぐため、ZIP 作成直前に必ず空き容量を検証する。

### 3.1 安全閾値判定 (Safety Thresholds)
以下のいずれかに該当する場合、バックアップ処理を即座に安全中断（Abort）する。
- 保存先ドライブの空き容量が `MinimumFreeSpace`（デフォルト: 2GB）未満。
- 保存先ドライブの空き容量率が 5% 未満。
- 今回書き込む推定ファイル合計サイズの 1.5倍以上の空き容量がない。

---

# 4. アトミックコミット ＆ 相互整合性検証仕様

```text
[ RestoreContainerAsync ]
        │
        ├─ 1. Header / Integrity / EnvelopeVersion validation
        │
        ├─ 2. Derive 256-bit key from password
        │
        ├─ 3. Decrypt + authenticate GST_Manifest.json.enc
        │      └─ ManifestDigest from canonical plaintext
        │
        ├─ 4. If DB ExpectedManifestHash exists, compare as additional anchor
        │
        ├─ 5. Validate every RelativePath + restore-root containment
        │      and existing TOCTOU / Reparse Point boundary
        │
        ├─ 6. Authenticate/decrypt every payload chunk with required AAD
        │      and strict per-file chunk sequence
        │
        ├─ 7. Verify restored size + SHA-256 against authenticated Manifest
        │
        └─ 8. Review → Confirm → Execute authorization + atomic replacement
```

**Restore invariant:** authentication, manifest validation, path containment, transaction authorization, and file integrity verification all precede replacement of protected user data.

---

# 5. Infrastructure Layer 実装契約 (approved portability and security decision)

The previous v4.3 reference implementation is superseded and must not be copied into Production Source.

Required Infrastructure invariants:

1. Secure Backup rejects null/empty password input.
2. EnvelopeVersion and cryptographic identifiers are allowlisted; unknown values fail closed.
3. Password ownership transfer and zeroization follow approved portability and security decision.
4. Header/Integrity canonical bytes are fixed before Manifest authentication; ManifestDigest is computed from canonical plaintext only.
5. `GST_Manifest.json.enc` must authenticate successfully before any restore destination is opened for replacement.
6. Every payload chunk must use deterministic AAD containing envelope/file/chunk/manifest context.
7. Chunk indices are zero-based and strictly sequential per file; partial, duplicated, reordered, or extra records fail closed.
8. RelativePath is hostile input. Rooted, UNC, device, traversal, alternate-data-stream, and other disallowed paths are rejected according to the existing Windows canonical-path policy.
9. Payload ZIP entry names are opaque identifiers and are resolved only through the authenticated Manifest; archive entry names are never interpreted as user-controlled filesystem paths. Restore destination and temporary files must remain within the user-selected validated root. Existing IPathResolver / PathValidation / TOCTOU / Reparse Point contracts are mandatory defense-in-depth boundaries.
10. Final replacement occurs only after authentication, manifest, path containment, transaction authorization, and hash/size verification have all succeeded.
11. No password, master key, raw manifest path, raw personal path, or raw exception detail may enter logs, audit text, telemetry, or external AI payloads.
12. Nonces, tags, salt, plaintext/ciphertext buffers, and derived keys require explicit lifetime/disposal/zeroization behavior consistent with the final threat model.
13. Existing `Review → Confirm → Execute` restore authorization remains mandatory; cryptographic authenticity never substitutes for user confirmation.


# 6. 単体テスト仕様 (`GST.UnitTests.SaveBackup.Storage`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.SaveBackup.Storage;

using System;
using System.IO;
using System.Text;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Infrastructure.SaveBackup;
using Microsoft.Extensions.Logging.Abstractions;
using Xunit;

public class ZipSaveBackupStorageTests
{
    [Fact]
    public async Task CreateAndRestoreContainer_WithArgon2idPassword_SucceedsCleanly()
    {
        var storage = new ZipSaveBackupStorage(NullLogger<ZipSaveBackupStorage>.Instance);

        string testDir = Path.Combine(Path.GetTempPath(), "GST_Test_SaveBackup_" + Guid.NewGuid());
        string saveDir = Path.Combine(testDir, "Saves");
        string backupDir = Path.Combine(testDir, "Backups");
        string restoreDir = Path.Combine(testDir, "Restored");
        Directory.CreateDirectory(saveDir);
        Directory.CreateDirectory(backupDir);

        string saveFile = Path.Combine(saveDir, "save.dat");
        byte[] dummySave = new byte[100000]; // 100KB
        Random.Shared.NextBytes(dummySave);
        await File.WriteAllBytesAsync(saveFile, dummySave);

        var fileMeta = new SaveFileMetadataDto("save.dat", saveFile, dummySave.Length, DateTimeOffset.UtcNow, null, null);
        
        byte[] pwdBytes = Encoding.UTF8.GetBytes("P@ssw0rd123!");
        var createReq = new SaveBackupCreateRequestDto("OP-1", "TX-1", Guid.NewGuid(), "TestGame", pwdBytes, saveDir, backupDir, [fileMeta]);

        // 1. 作成 (Argon2id + AES-256-GCM)
        var createResult = await storage.CreateContainerAsync(createReq);
        Assert.True(createResult.Success);
        Assert.True(File.Exists(createResult.ContainerPath));

        // 2. 復元
        // 復元用に新しいパスワード配列を用意 (Create側でZero化されているため)
        byte[] restorePwdBytes = Encoding.UTF8.GetBytes("P@ssw0rd123!");
        var restoreReq = new SaveBackupRestoreRequestDto("OP-2", "TX-2", createResult.BackupId, restorePwdBytes, createResult.ContainerPath, restoreDir, createResult.ManifestHash);
        var restoreResult = await storage.RestoreContainerAsync(restoreReq);
        Assert.True(restoreResult.Success);

        string restoredFile = Path.Combine(restoreDir, "save.dat");
        Assert.True(File.Exists(restoredFile));
        Assert.Equal(dummySave, await File.ReadAllBytesAsync(restoredFile));

        // メモリ安全性の確認 (テストメソッド自身も明示的にゼロ化)
        Array.Clear(pwdBytes);
        Array.Clear(restorePwdBytes);
        Directory.Delete(testDir, true);
    }
}
```

---

End of Document
```

---

