# GameSecurityTool Save Backup Test & Verification Specification

**Document ID:** GST-FEAT-SAVE-TEST-010  
**Version:** 2.4
**Status:** Approved Quality Specification  
**Target Layer:** Quality Assurance (`Tests/GameSecurityTool.*`)

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Test Strategy & Verification Architecture

Save Backup 機能の品質保証は、単なる正常系のコピー確認にとどまらず、**「高負荷時のSSD保護」「大容量ファイルのOOM耐性」「TOCTOU/ReparsePoint攻撃防御」「電源断時のロールバック」「AES-GCM Nonce Reuse 防止」** を重点検証する。

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        Security Verification Tests                     │
│  - ファイル別 Nonce 生成 ＆ Nonce 重複ゼロ検証                         │
│  - Reparse Point すり替え / パストラバーサル拒否                       │
│  - 暗号化エンベロープ改ざん検知 (Header/Integrity/Manifest/AAD)          │
│  - DPAPI 鍵保護コンテキスト分離検証                                    │
├────────────────────────────────────────────────────────────────────────┤
│                       Resilience & Failure Tests                       │
│  - 復元中強制クラッシュ ➔ Pre-Restore Snapshot からのロールバック      │
│  - ディスク満杯時 (Disk Full) ➔ 安全中断 ＆ 既存スナップショット保護    │
│  - ファイルロック中 ➔ 指数バックオフリトライ ＆ 安全側スキップ         │
│  - RescueSnapshot の 24時間/7日間 ライフサイクル自動パージ検証         │
├────────────────────────────────────────────────────────────────────────┤
│                       Performance & Load Tests                         │
│  - 4GB 巨大セーブデータ ➔ 1-byte control比のピーク増加 <= 16MB (approved specification boundary) │
│  - パス正規化 (Canonicalization) による Windows / Linux 混在耐性      │
│  - 2回目スキャン ➔ 変更なしファイルのハッシュ計算をスキップ       │
└────────────────────────────────────────────────────────────────────────┘
2. 単体テスト仕様マトリクス (GST.UnitTests)
2.1 ドメインロジックテスト (GST.UnitTests.Domain)
TC-DOM-01 (Fast Check): 同一サイズ・同一 LastWriteTimeUtc のファイルマニフェスト比較時、差分なし（Skip）が正しく判定されること。
TC-DOM-02 (Canonical Manifest Hash): パス区切り文字が \ または / のいずれであっても、正規化処理により同一の ManifestHash が決定論的に算出されること。
TC-DOM-03 (Retention Rule): スナップショットが保持世代数上限（例: 3世代）を超えた際、最古の世代が削除候補となり、最新世代は絶対に保護されること。
TC-DOM-04 (Incremental Dependency Check): 後続スナップショットから参照されている過去コンテナの BackupId に対し、削除抑止（物理コンテナ維持）が判定されること。
3. 結合テスト仕様マトリクス (GST.IntegrationTests)
3.1 ストレージ・暗号化結合テスト (GST.IntegrationTests.Infrastructure)

### 3.2 Standard ZIP portability / encrypted-payload recovery contract (approved specification boundary)
- **TC-INT-04 (ZIP Container Portability):** 作成されたバックアップがStandard ZIP互換コンテナとして汎用ZIPツールで開け、`GST_Header.json` / `GST_Integrity.json` / `GST_Manifest.json.enc` を含む所定のZIP entry構造を取得できることを検証する。これは暗号化payloadの復号成功を意味しない。
- **TC-INT-05 (Encrypted Payload Format Recovery):** 暗号化バックアップについて、公開されたVersioned Authenticated Envelope仕様だけを入力として作成したGST互換の独立復旧実装が、GST本体のDBやDPAPI状態なしで正しいパスワードからManifestとpayloadを復号・認証し、原データを復元できることを検証する。
- **TC-SEC-03 (Generic Tool Limitation):** 7-Zip / Explorer等の汎用ZIPツールによる外側コンテナの展開成功と、暗号化payloadの平文復元を別結果として扱う。汎用ツール単独でpayloadが復号できないことは失敗ではなく、documented security requirementで定義した非保証事項である。
- **TC-SEC-04 (Format Determinism):** encrypted payloadの各chunkについて`Nonce(12B) + Tag(16B) + Len(4B LE) + Ciphertext(LenB)`構造、Len範囲1..65536、chunkごとのnonce新規生成、EnvelopeVersion、ManifestDigest、AAD構成、Argon2idパラメータ、AES-256-GCM tag長が仕様どおりであることを検証する.
- **TC-SEC-09 (Metadata Authenticity):** Header/Integrity/Manifestを個別に改ざんした場合、正しいパスワードでも認証に失敗し、ユーザーデータ置換が発生しないことを検証する。
- **TC-SEC-10 (Chunk Context Binding):** RelativePath、FileId、chunk index、chunk length、ManifestDigestのいずれかを改ざん・再利用・入れ替えた場合、GCM AAD認証により復元が拒否されることを検証する。
- **TC-SEC-11 (Passwordless Rejection):** Secure BackupのNULL/空Passwordが拒否され、暗号化なしSecure Backupが作成されないことを検証する。
- **TC-SEC-12 (Standalone Manifest Integrity):** DBおよびDPAPI状態が存在しない独立復旧でも、Manifestの認証に失敗したコンテナは復旧処理へ進まないことを検証する。
- **TC-SEC-13 (Restore Root Containment):** rooted / UNC / device / traversal / ADS pathおよびrestore root外へ解決されるpathを拒否し、既存ユーザーファイルを置換しないことを検証する。
- **TC-SEC-14 (Reparse / TOCTOU Defense in Depth):** restore rootまたは対象pathのreparse point/junction状態が検証後に変化した場合、最終replacementを行わずfail-closedとなることを検証する。
- **TC-SEC-15 (Replacement Ordering):** authentication、manifest検証、path containment、transaction authorization、size/SHA-256 verificationのいずれかが未完了なら既存ファイルのoverwriteが発生しないことを検証する。


TC-INT-01 (File-Level Nonce Unique Check): コンテナ生成時、各ファイルペイロードエントリの Nonce (12B) がすべてユニークに乱数生成されており、Nonce Reuse が発生しないこと。
TC-INT-02 (Portable Key Derivation Boundary): 生成された暗号化コンテナにDPAPIで保護されたAESマスター鍵を格納しないこと。`GST_Integrity.json` にはArgon2idのSalt / Iterations / MemorySize / Parallelism等の導出パラメータのみを格納し、復号時にユーザー入力パスワードから32-byte master keyをオンデマンド導出すること。これによりバックアップ暗号化は端末依存のDPAPI状態に拘束されない。
TC-INT-03 (Sqlite Database Writer Queue): マルチスレッドから同時に ISaveBackupRepository.SaveSnapshotAsync を呼び出した際、SqliteDatabaseWriter によりデッドロックなしで直列実行されること。
4. セキュリティ ＆ 障害耐性検証テスト (GST.SecurityTests)
4.1 パストラバーサル ＆ TOCTOU 検証
code
C#
namespace GameSecurityTool.SecurityTests;

using System;
using System.IO;
using System.Security;
using System.Threading.Tasks;
using Xunit;

public class SaveBackupSecurityTests
{
    [Fact]
    public async Task Restore_WhenTargetIsJunctionPointingOutside_ThrowsSecurityExceptionAndAborts()
    {
        // Arrange: 復元先ディレクトリ内に、外部 (C:\Windows\System32) を指すジャンクションを作成
        string gameSaveDir = Path.Combine(Path.GetTempPath(), "FakeGameSaves");
        string maliciousJunction = Path.Combine(gameSaveDir, "JunctionToSystem");
        Directory.CreateDirectory(gameSaveDir);
        
        // ジャンクション作成 (Win32 API 相当)
        CreateTestJunction(maliciousJunction, Environment.GetFolderPath(Environment.SpecialFolder.System));

        var restoreService = new SaveBackupRestoreCoordinator(mockStorage, mockPathResolver, mockLogger);
        var request = new SaveBackupRestoreRequestDto("OP-1", "TX-1", Guid.NewGuid(), "test.gstcontainer", gameSaveDir);

        // Act & Assert: 横断的バリアが検知し、復元処理を即時拒否すること
        await Assert.ThrowsAsync<SecurityException>(() => restoreService.ExecuteRestoreAsync(request));
    }
}
4.2 復元中クラッシュ ＆ ロールバック検証
TC-SEC-01: 復元処理のファイル書き込み中に例外を注入（Fault Injection）した際、RestoreStatus = PartialFailed が記録され、Pre-Restore Snapshot (RescueSnapshot) から元のセーブデータが完全に復元されること。
TC-SEC-02: 復元成功から24時間経過後、RescueSnapshot フォルダが自動クリーンアップタスクにより安全に消去されること。
4.3 Password ownership transfer / zeroization verification (approved specification boundary)

- **TC-SEC-05 (Storage Ownership Transfer):** `SaveBackupCreateRequestDto.Password` / `SaveBackupRestoreRequestDto.Password` をStorageへ正常にhandoffした後、Storageが自分の所有するpassword bufferを `finally` でzeroizeすることを検証する。成功・通常例外・`OperationCanceledException` を対象とする。
- **TC-SEC-06 (Pre-Storage Abandonment Zeroization):** Storageへのhandoff前にrequestがqueue write failure、cancellation、未処理requestのdiscard、debounce replacement、shutdown等で破棄された場合、その時点のownerが保持しているpassword bufferだけをzeroizeすることを検証する。
- **TC-SEC-07 (Ownership Boundary / No Double Owner):** 正常なhandoff後は旧ownerが同一bufferをzeroizeせず、Storageだけがfinal zeroization責務を持つことを、owner-state instrumentationまたは専用test doubleで検証する。別途作成されたinternal copyについてはcopy作成者が自身のcopyをzeroizeすることを確認する。
- **TC-SEC-08 (Create/Restore Symmetry):** Create / Restore の双方で同一のownership-transfer contractが適用され、正常系・失敗・cancel・queue/write failure・shutdown/discardの全対象経路に抜けがないことを検証する。

**Test fixture rule:** テスト中はpassword内容をログへ出力せず、固定長のsentinel patternまたはhash等の非秘密識別子のみを使用する。ownership-transfer testは「bufferの値がzeroになったこと」だけでなく「どのcomponentがその時点のownerだったか」を検証対象に含める。
5. パフォーマンス ＆ リソース検証
TC-PERF-01 (Large-File Resource Scaling — documented security requirement): 4GB のダミーセーブファイルをバックアップ・復元する際、同一 Release ビルド・同一暗号設定・同一測定条件で取得した1バイト control の matched PeakOperationPrivateMemory に対する4GB runのピーク増加が **16MB以下**であること。測定は3つの独立したmatched pairで行い、各pairが基準を満たすことをリリース証拠とする。これは単一ファイルのpayload scalingに対する基準であり、64MiBのMaxManifestSizeを持つ大規模manifestの総メモリ上限を意味しない。
TC-PERF-02 (Fast Check Speed): 2,000 ファイルのセーブフォルダに対し、変更がない場合の差分スキャンが 200ms 以内に完了すること。
6. 品質ゲートチェックリスト (Release Gate Checklist)
本機能のリリースビルド前に全項目をクリアすること。

すべての単体テスト・結合テスト・セキュリティテストが 定義済みの全項目を満たすすること

Visual Studio / CI においてコンパイル警告および Nullable 警告が 0 件であること

巨大ファイル処理時のメモリリーク・リソース解放漏れがないこと（IDisposable / SafeHandle / ZeroMemory）

監査ログ（Audit Chain）にバックアップ・復元の全履歴が欠落なく記録されること

テレメトリおよび外部通信コードが一切含まれていないこと（ローカル完結確認）

### 4.4 documented security requirement Authenticated Envelope / Restore Containment Security Matrix

- **TC-SEC-16 (Cross-File Replay):** File Aで生成したchunkをFile Bへ移送しても、File/Path/ManifestDigest bindingにより認証失敗すること。
- **TC-SEC-17 (Chunk Sequence Integrity):** chunk indexの欠落、重複、並べ替え、末尾余剰record、truncated header/ciphertextをすべて拒否すること。
- **TC-SEC-18 (KDF Parameter Tamper):** Salt / Iterations / MemorySize / Parallelism / Cipher / EnvelopeVersionの改ざんを検知し、推測fallbackせず拒否すること。
- **TC-SEC-19 (Header Context Tamper):** Header内容の改ざん後に正しいpasswordを入力してもManifest/payload認証が成立しないこと。
- **TC-SEC-20 (Sensitive Metadata Confidentiality):** 暗号化Secure BackupのZIP内inspectionでRelativePath、FileId、LastWriteTime等のmanifest metadataが平文entryとして露出しないこと。
- **TC-SEC-23 (Opaque Payload Entry Names):** 暗号化Secure BackupのZIP entry名が不透明な `F########.bin` 形式のみであり、元のファイル名・RelativePathがZIP構造から推測できないこと。
- **TC-SEC-21 (Wrong Password Before Replacement):** 誤passwordではManifest認証に失敗し、restore root内の既存user fileが変更されないこと。
- **TC-SEC-22 (Password Buffer Zeroization Regression):** documented security requirementのCreate/Restore ownership-transfer zeroizationとdocumented security requirementで追加されたworking-buffer lifecycleが、success/failure/cancelで維持されること。

### 6.1 Release Blocking Conditions

Backup Release Readiness must remain blocked until all of the following have executable evidence: authenticated envelope success, metadata/AAD tamper rejection, standalone Manifest authentication, restore-path containment, reparse/TOCTOU defense, replacement ordering, wrong-password no-replacement behavior, password/working-buffer zeroization, independent recovery interoperability, and large-file resource-bound verification.
