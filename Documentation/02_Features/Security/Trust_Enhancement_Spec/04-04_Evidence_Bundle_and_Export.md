# 04-04: Evidence Bundle Export Specification

**Document ID:** GST-FEAT-TRUST-004  
**Version:** 2.0 (Audit Chain Pre-Verification & Clean Streaming Hardened)
**Parent Document:** 04-00_Overview_and_Principles.md  
**Category:** Infrastructure & Security Specification  
**Status:** Approved Feature Specification  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. 目的

誤検知（False Positive）の調査、サポート提出、セキュリティフォレンジックのために、検知結果・監査証跡・設定スナップショットを安全な ZIP パッケージ（`EvidenceBundle.zip`）としてエクスポートする機能を提供する。

---

# 2. パッケージ構造 (`EvidenceBundle.zip`)

```text
EvidenceBundle_<OperationId>_<Timestamp>.zip
├─ Manifest.json                  # パッケージメタデータ, 全エントリSHA256, IntegrityStatus
├─ Audit/
│   └─ AuditChain_Fragment.json   # 該当操作に関わる改ざん検知ハッシュチェーン証跡
├─ Detection/
│   └─ AssessmentResult.json      # 技術シグナル、ConfidenceScore、判定理由
├─ Configuration/
│   ├─ FeatureState.json          # スナップショット時点の機能有効無効状態
│   ├─ EffectivePolicy.json       # 適用されたファイアウォール・Web・スキャンルール
│   └─ SystemEnvironment.json     # OSバージョン、.NETランタイム情報 (個人情報は除外)
└─ Metadata/
    └─ ExportConsent.json         # ユーザーがエクスポートを承認した事実の記録
```

---

# 3. エクスポートパイプライン ＆ ハッシュチェーン事前検証 (HIGH-11 解決)

エクスポート実行時、改ざんされた証跡が出力されるのを防ぐため、以下の 5 段階パイプラインを厳格に実行する。

```text
[ ユーザーのエクスポート要求 (OperationId 指定) ]
                       │
                       ▼
1. [Audit Chain 事前整合性検証]
   - GenesisHash からの連鎖ハッシュ再計算
   - DPAPI 領域の最新ハッシュマーカーと照合
   - 判定: Valid / TamperingDetected
                       │
                       ▼
2. [データ収集 ＆ 個人情報自動マスキング]
   - PrivacyLogSanitizer によるフルパス・ユーザー名伏字化
                       │
                       ▼
3. [ユーザー同意 ＆ プレビュー表示]
   - 出力ファイル一覧・内容サマリーのダイアログ提示
                       │ (承認)
                       ▼
4. [完全ストリーミング ZIP 生成]
   - FileStream ➔ ZipArchive への 64KB チャンク出力
                       │
                       ▼
5. [エクスポート監査ログ記録]
   - AuditEventType = EvidenceExported をコミット
```

---

# 4. ストリーミング I/O 実装仕様 (LOW-03 解決)

巨大な監査ログや設定データを扱う際のメモリ枯渇（OOM）を防ぐため、**メモリ内一括生成（`File.ReadAllBytes()` や `MemoryStream` の過度な利用）を厳禁**とし、直接エントリストリームへ非同期シリアライズする。

```csharp
namespace GameSecurityTool.Infrastructure.Features.TrustEnhancement;

using System;
using System.IO;
using System.IO.Compression;
using System.Security.Cryptography;
using System.Text.Json;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Infrastructure.Persistence.Serialization;

public sealed class EvidenceBundleWriter
{
    private const int BufferSize = 65536; // 64KB バッファ

    public static async Task<EvidenceExportResultDto> WriteBundleAsync(
        string targetZipPath,
        string operationId,
        object auditChainData,
        object assessmentData,
        object configData,
        bool isChainValid,
        CancellationToken ct)
    {
        string? dir = Path.GetDirectoryName(targetZipPath);
        if (!string.IsNullOrEmpty(dir)) Directory.CreateDirectory(dir);

        string tempZipPath = targetZipPath + ".tmp";

        try
        {
            await using (var outputStream = new FileStream(tempZipPath, FileMode.CreateNew, FileAccess.ReadWrite, FileShare.None, BufferSize, useAsync: true))
            using (var archive = new ZipArchive(outputStream, ZipArchiveMode.Create, leaveOpen: false))
            {
                // 1. Audit Chain 証跡のストリーミング書き出し
                var auditEntry = archive.CreateEntry("Audit/AuditChain_Fragment.json", CompressionLevel.Optimal);
                await using (var entryStream = auditEntry.Open())
                {
                    await JsonSerializer.SerializeAsync(entryStream, auditChainData, SnapshotJsonSerializerOptions.Default, ct);
                }

                // 2. 判定シグナルデータの書き出し
                var detectEntry = archive.CreateEntry("Detection/AssessmentResult.json", CompressionLevel.Optimal);
                await using (var entryStream = detectEntry.Open())
                {
                    await JsonSerializer.SerializeAsync(entryStream, assessmentData, SnapshotJsonSerializerOptions.Default, ct);
                }

                // 3. 設定スナップショットの書き出し
                var configEntry = archive.CreateEntry("Configuration/EffectivePolicy.json", CompressionLevel.Optimal);
                await using (var entryStream = configEntry.Open())
                {
                    await JsonSerializer.SerializeAsync(entryStream, configData, SnapshotJsonSerializerOptions.Default, ct);
                }

                // 4. マニフェスト書き出し (改ざん検知ステータスを明記)
                var manifestData = new
                {
                    OperationId = operationId,
                    ExportedAtUtc = DateTimeOffset.UtcNow,
                    IntegrityStatus = isChainValid ? "VerifiedValid" : "TamperingDetected",
                    AppVersion = "1.0.0"
                };

                var manifestEntry = archive.CreateEntry("Manifest.json", CompressionLevel.Fastest);
                await using (var entryStream = manifestEntry.Open())
                {
                    await JsonSerializer.SerializeAsync(entryStream, manifestData, SnapshotJsonSerializerOptions.Default, ct);
                }
            }

            File.Move(tempZipPath, targetZipPath, overwrite: true);

            var fileInfo = new FileInfo(targetZipPath);
            string sha256Hex;
            await using (var fs = new FileStream(targetZipPath, FileMode.Open, FileAccess.Read, FileShare.Read, BufferSize, useAsync: true))
            {
                byte[] hashBytes = await SHA256.HashDataAsync(fs, ct);
                sha256Hex = Convert.ToHexString(hashBytes);
            }

            return new EvidenceExportResultDto(true, targetZipPath, fileInfo.Length, sha256Hex, null);
        }
        catch (Exception ex)
        {
            if (File.Exists(tempZipPath))
            {
                try { File.Delete(tempZipPath); } catch { }
            }
            return new EvidenceExportResultDto(false, string.Empty, 0, string.Empty, ex.Message);
        }
    }
}
```

---

# 5. プライバシー保護 ＆ 同意（User Consent）

1. **個人情報・フルパスのサニタイズ:**  
   ユーザー名を含むプロファイル絶対パス（`C:\Users\<UserName>\...`）は `PrivacyLogSanitizer` により `C:\Users\***\...` または `%USERPROFILE%` に自動マスク。
2. **事前プレビュー表示:**  
   エクスポート実行前に「出力されるファイル一覧、マスクされたパス、および改ざん検証結果」を UI ダイアログで明示。
3. **明示同意の取得:**  
   ユーザーの明示的な「同意してエクスポート」ボタン押下なしにファイルを生成しない。

---

End of Document
```

---
