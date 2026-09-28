# GameSecurityTool MOD 安全性診断 ＆ 出所追跡（Mod Provenance）仕様書

**文書ID:** GST-FEAT-MOD-PROVENANCE-001  
**版:** 2.2 (Unsigned-Rate Provenance Clarified & Structure-Over-Signature Edition)
**状態:** 採用確定 (Approved Feature Specification)  
**カテゴリ:** Security / Mod Integrity  
**親文書:** `00_Formal_Baseline_Overview.md` / `Advanced_User_Protection_Master_Spec.md`  
**対象:** Windows 10 / Windows 11 / .NET 10 / C# 14 / WPF  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 0. Purpose ＆ 設計思想

## 0.1 目的
PC ゲームの MOD では未署名（Unsigned）のコンポーネントが一般的に存在する。本仕様は **特定の未署名率（たとえば「90% 以上」）を実証済みの統計値として前提にしない**。出所、対象母集団、調査時点等を特定できない数値は設計根拠として使用せず、**「未署名であること自体を悪性の証拠とはみなさず、DLL Side-Loading 等の構造・出所・整合性を別途評価する」** ための自律的診断・出所追跡基盤を定義する。

## 0.2 GST コミュニティ非依存原則 (No Cold-Start Dependency)
GST 専用のコミュニティや独自データベースの存在を前提とせず、**「Windows OS 標準機能」「オープンな業界標準フォーマット（SHA256SUMS / Wabbajack 等）」「純粋バイナリ静的解析」** のみを用いて、完全ローカル・外部通信ゼロで安全性を担保する。

---

# 1. 3層ハイブリッド・パイプライン概要

```text
┌────────────────────────────────────────────────────────────────────────┐
│  Layer 1: 【日常・自動診断】プロキシDLL静的解析 ＋ Zone.Identifier出所追跡 │
│  - PE エクスポート転送構造の自己検証 (Direct3D フック等の同定)          │
│  - NTFS 代替データストリームからブラウザ入手元 (GitHub, Nexus) を特定   │
├────────────────────────────────────────────────────────────────────────┤
│  Layer 2: 【個別・確実照合】汎用 SHA256SUMS ＆ 元アーカイブ自己突合   │
│  - 開発者公式の checksums.txt をドラッグ＆ドロップでローカル照合       │
│  - Downloads フォルダ内の元 ZIP / 各種圧縮ファイル等と解凍後を突合     │
├────────────────────────────────────────────────────────────────────────┤
│  Layer 3: 【大量MOD一括】Wabbajack / Nexus Collections マニフェスト直読│
│  - 既存の巨大 MOD パック定義から数千件のハッシュを一括ベースライン化   │
└────────────────────────────────────────────────────────────────────────┘
```

## 1.1 高度脅威検知・整合性監視エンジン (Advanced Mod Threat Engine)
従来のプロキシ DLL 診断に加え、以下の脅威を網羅的に検知する：

1. **アセット Mod コンテンツ種別不一致検知 (Asset Content Type Mismatch Detection):**
   テクスチャや音声などのアセット専用フォルダ（`sound/`, `sounds/`, `audio/`, `music/`, `texture/`, `textures/`, `graphics/` 等）内に、実行バイナリ（`.exe`, `.dll`, `.sys`）やスクリプト（`.bat`, `.cmd`, `.ps1`, `.vbs`, `.js`）が混入している異常を検知・警告。
2. **スクリプト LotL 走査 (Script Living-off-the-Land Scanning):**
   MOD に同梱された `.ini`, `.cfg`, `.bat`, `.cmd`, `.ps1` などの設定・スクリプトファイル内に、間接コマンド実行コード（`powershell -enc`, `curl`, `certutil`, `bitsadmin`, `mshta`, `cscript` 等）が含まれていないか静的走査。
3. **パッチ適用時リスク急変 (Update Anomaly) 検知:**
   MOD やゲーム本体のパッチ更新時、更新前後のスナップショット差分から「未署名外部通信バイナリ」や「ドロッパー」の追加を検知し、ゲーム起動を一時停止して差分確認ダイアログを表示。
4. **主要ランチャー越境改ざん監視 (`ILauncherSecurityAdapter`):**
   MOD 導入時やゲーム実行中に、Steam, Epic Games, EA Desktop などの主要ランチャーディレクトリや共有コンポーネントに対する不正な改ざん・越境書き込みを監視・遮断。
5. **署名偽装・失効検証 ＆ 中身重視原則 (Behavior & Structure Over Signature):**
   コード署名が存在する場合でも盲信せず、CRL / OCSP オンライン失効検証を行い、失効済み証明書（Revoked Certificate）や期限切れ証明書を確実に検知。署名が有効であっても、アセット MOD 内の不自然なソケット API（`ws2_32.dll`）やプロセス起動（`cmd.exe`）が存在する場合は「構造異常」として高リスク判定（署名よりも振る舞い・構造を最優先評価）。

---

# 2. Clean 5-Layer アーキテクチャ責務境界

```text
[ Presentation Layer (GST.Presentation) ]
  - ModDiagnosisCard (レントゲン診断結果・出所表示)
  - GenericChecksumDropDialog (SHA256SUMS / ZIP ドロップ受付)
        │
        ▼ (calls UseCase)
[ Application Layer (GST.Application) ]
  - InspectModProvenanceUseCase (Layer 1〜3 統合調停)
  - ImportGenericChecksumUseCase / ImportWabbajackManifestUseCase
  - ExplanationEngine 連携 (技術シグナルの平易な解説文変換)
        │
        ├─────────────────────────────┐
        ▼ (uses Domain Models)        ▼ (calls Port Interfaces)
[ Domain Layer (GST.Domain) ]       [ Contracts Layer (GST.Contracts) ]
  - ModSecurityEvaluator (Pure C#)   - IProxyDllInspector (Port)
  - ModProvenanceSource (Enum)       - IZoneIdentifierReader (Port)
  - ModSecurityAssessment (Model)    - IArchiveIntegrityMatcher (Port)
                                     - IModManifestImporter (Port)
                                      ▲
                                      │ (implements)
                                    [ Infrastructure Layer (GST.Infrastructure) ]
                                      - PeProxyDllInspector (PE ヘッダー/Import解析)
                                      - ZoneIdentifierReader (NTFS ADS 取得)
                                      - ArchiveIntegrityMatcher (ZIP/各種圧縮形式 照合)
                                      - WabbajackManifestImporter (JSON パース)
```

---

# 3. Domain Layer 定義 (`GameSecurityTool.Domain`)

Domain 層は外部参照ゼロ（Pure C#）で構成され、MOD の静的特徴シグナルを受け取って純粋なリスク評価を下す。

```csharp
namespace GameSecurityTool.Domain.Models.ModSecurity;

using System;
using System.Collections.Generic;

public enum ModProvenanceSource
{
    Unknown = 0,
    BrowserDownload = 1,        // Zone.Identifier 検知 (GitHub, NexusMods 等)
    VerifiedChecksumMatch = 2,  // SHA256SUMS / checksums.txt 一致
    ArchiveMatch = 3,           // ローカル元 ZIP/各種圧縮ファイル と完全一致
    ModPackManifestMatch = 4    // Wabbajack / Collections マニフェスト一致
}

public enum ProxyDllCategory
{
    None = 0,
    DirectXHook = 1,     // dxgi.dll, d3d11.dll, d3d9.dll, d3d12.dll
    InputHook = 2,       // dinput8.dll, xinput1_3.dll
    AudioHook = 3,       // x3daudio1_7.dll, dsound.dll
    SystemProxy = 4      // version.dll, winmm.dll, binkw64.dll
}

public sealed record ModInspectionSignals(
    bool IsProxyExportValid,           // 正当なプロキシ転送エクスポート構造を持つか
    ProxyDllCategory HookCategory,     // フック対象カテゴリ
    bool HasSuspiciousNetworkImports,  // ws2_32.dll, wininet.dll 等の通信 API を含むか
    bool HasProcessCreationImports,    // CreateProcess, cmd.exe 呼び出しを含むか
    ModProvenanceSource Provenance,    // 出所ソース
    string? DownloadHostUrl,           // ダウンロード元 URL (取得時のみ)
    bool IsSignatureRevoked = false,   // CRL/OCSP 失効検知
    bool HasStructureAnomalies = false // アセットフォルダ内実行可能ファイル混入等の構造異常
);

public sealed record ModSecurityAssessment(
    bool IsLikelySafeMod,
    int ConfidenceScore,               // 0〜100
    IReadOnlyList<string> ReasonCodes,
    DateTimeOffset EvaluatedAtUtc
);
```

## 3.1 純粋ドメイン判定サービス (`ModSecurityEvaluator.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Domain.Services;

using System;
using System.Collections.Generic;
using GameSecurityTool.Domain.Models.ModSecurity;

public sealed class ModSecurityEvaluator
{
    public ModSecurityAssessment Evaluate(ModInspectionSignals signals)
    {
        int score = 50; // 初期ニュートラルスコア
        var reasons = new List<string>();

        // 1. プロキシ DLL 構造評価
        if (signals.IsProxyExportValid && signals.HookCategory != ProxyDllCategory.None)
        {
            score += 30;
            reasons.Add("ValidProxyDllStructure");
        }

        // 2. 出所 (Provenance) 評価
        switch (signals.Provenance)
        {
            case ModProvenanceSource.VerifiedChecksumMatch:
            case ModProvenanceSource.ArchiveMatch:
            case ModProvenanceSource.ModPackManifestMatch:
                score += 40;
                reasons.Add("ProvenanceSourceVerified");
                break;

            case ModProvenanceSource.BrowserDownload:
                score += 20;
                reasons.Add("ProvenanceBrowserDownloadConfirmed");
                break;
        }

        // 3. 危険 API インポート評価 (減点)
        if (signals.HasSuspiciousNetworkImports)
        {
            score -= 50;
            reasons.Add("ContainsSuspiciousNetworkApi");
        }

        if (signals.HasProcessCreationImports)
        {
            score -= 40;
            reasons.Add("ContainsProcessSpawningApi");
        }

        // 4. 失効署名・構造異常評価 (中身重視の原則)
        if (signals.IsSignatureRevoked)
        {
            score -= 60;
            reasons.Add("CodeSigningCertificateRevoked");
        }

        if (signals.HasStructureAnomalies)
        {
            score -= 50;
            reasons.Add("StructuralAnomalyDetected");
        }

        score = Math.Clamp(score, 0, 100);
        bool isSafe = score >= 70
            && !signals.HasSuspiciousNetworkImports
            && !signals.HasProcessCreationImports
            && !signals.IsSignatureRevoked
            && !signals.HasStructureAnomalies;

        return new ModSecurityAssessment(isSafe, score, reasons, DateTimeOffset.UtcNow);
    }
}
```

---

# 4. Contracts Layer 定義 (`GameSecurityTool.Contracts`)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;

public interface IProxyDllInspector
{
    ProxyDllInspectionDto InspectDllStructure(string filePath);
}

public interface IZoneIdentifierReader
{
    ZoneIdentifierDto? ReadZoneIdentifier(string filePath);
}

public interface IArchiveIntegrityMatcher
{
    Task<bool> MatchFileAgainstArchiveAsync(string targetFilePath, string archiveFilePath, CancellationToken ct = default);
    Task<IReadOnlyDictionary<string, string>> ExtractChecksumsFromTextAsync(string checksumTextFilePath, CancellationToken ct = default);
}

public interface IModManifestImporter
{
    Task<IReadOnlyList<ModManifestEntryDto>> ParseWabbajackManifestAsync(string manifestPath, CancellationToken ct = default);
    Task<IReadOnlyList<ModManifestEntryDto>> ParseVortexManifestAsync(string stagingPath, CancellationToken ct = default);
}
```

```csharp
namespace GameSecurityTool.Contracts.Dtos;

using System;
using System.Collections.Generic;

public sealed record ProxyDllInspectionDto(
    bool IsValidPe,
    bool IsProxyExportStructure,
    string HookCategory,
    IReadOnlyList<string> ExportedFunctions,
    bool HasNetworkImports,
    bool HasProcessCreationImports
);

public sealed record ZoneIdentifierDto(
    int ZoneId,
    string? HostUrl,
    string? ReferrerUrl
);

public sealed record ModManifestEntryDto(
    string RelativePath,
    string ExpectedSha256,
    long ExpectedSizeBytes,
    string? ModName
);

public sealed record ModProvenanceReportDto(
    string FilePath,
    string FileName,
    string Sha256,
    bool IsLikelySafeMod,
    int ConfidenceScore,
    string ProvenanceDescription,
    string? DownloadOriginUrl,
    IReadOnlyList<string> SafetyReasons
);
```

---

# 5. Infrastructure Layer 実装 (`GameSecurityTool.Infrastructure`)

## 5.1 PE プロキシ構造 ＆ 完全 Import テーブル解析 Adapter (`PeProxyDllInspector.cs` - 完全実装)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Native;

using System;
using System.Collections.Generic;
using System.IO;
using System.Reflection.PortableExecutable;
using System.Text;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;

public sealed class PeProxyDllInspector : IProxyDllInspector
{
    private static readonly HashSet<string> NetworkDllNames = new(StringComparer.OrdinalIgnoreCase)
    {
        "ws2_32.dll", "wsock32.dll", "wininet.dll", "winhttp.dll"
    };

    private static readonly HashSet<string> ProcessCreationApiNames = new(StringComparer.OrdinalIgnoreCase)
    {
        "CreateProcessA", "CreateProcessW", "WinExec", "ShellExecuteA", "ShellExecuteW", "ShellExecuteExA", "ShellExecuteExW"
    };

    public ProxyDllInspectionDto InspectDllStructure(string filePath)
    {
        if (!File.Exists(filePath))
        {
            return new ProxyDllInspectionDto(false, false, "None", [], false, false);
        }

        try
        {
            using var stream = new FileStream(filePath, FileMode.Open, FileAccess.Read, FileShare.ReadWrite | FileShare.Delete);
            using var peReader = new PEReader(stream);

            if (!peReader.PEHeaders.IsDll)
            {
                return new ProxyDllInspectionDto(false, false, "None", [], false, false);
            }

            var headers = peReader.PEHeaders;
            var exportTableDirectory = headers.PEHeader?.ExportTableDirectory;
            bool hasExports = exportTableDirectory.HasValue && exportTableDirectory.Value.Size > 0;

            string fileName = Path.GetFileName(filePath).ToLowerInvariant();
            string category = DetermineCategory(fileName);

            bool hasNetwork = false;
            bool hasProcessCreation = false;
            var exportedFunctions = new List<string>();

            // Import Table の完全走査
            var importDirectory = headers.PEHeader?.ImportTableDirectory;
            if (importDirectory.HasValue && importDirectory.Value.Size > 0)
            {
                AnalyzeImports(peReader, stream, importDirectory.Value.RelativeVirtualAddress, out hasNetwork, out hasProcessCreation);
            }

            return new ProxyDllInspectionDto(
                IsValidPe: true,
                IsProxyExportStructure: hasExports,
                HookCategory: category,
                ExportedFunctions: exportedFunctions,
                HasNetworkImports: hasNetwork,
                HasProcessCreationImports: hasProcessCreation
            );
        }
        catch
        {
            return new ProxyDllInspectionDto(false, false, "None", [], false, false);
        }
    }

    private static void AnalyzeImports(
        PEReader peReader, 
        FileStream stream, 
        int importDirectoryRva, 
        out bool hasNetwork, 
        out bool hasProcessCreation)
    {
        hasNetwork = false;
        hasProcessCreation = false;

        int importOffset = RvaToFileOffset(peReader, importDirectoryRva);
        if (importOffset < 0 || importOffset >= stream.Length) return;

        using var reader = new BinaryReader(stream, Encoding.ASCII, leaveOpen: true);
        stream.Seek(importOffset, SeekOrigin.Begin);

        // IMAGE_IMPORT_DESCRIPTOR (20 bytes 単位) の走査
        while (stream.Position + 20 <= stream.Length)
        {
            int originalFirstThunkRva = reader.ReadInt32();
            int timeDateStamp = reader.ReadInt32();
            int forwarderChain = reader.ReadInt32();
            int nameRva = reader.ReadInt32();
            int firstThunkRva = reader.ReadInt32();

            // Null ディスクリプタで終端
            if (nameRva == 0 && firstThunkRva == 0) break;

            // DLL 名の取得
            int nameOffset = RvaToFileOffset(peReader, nameRva);
            if (nameOffset > 0 && nameOffset < stream.Length)
            {
                long currentPos = stream.Position;
                stream.Seek(nameOffset, SeekOrigin.Begin);
                string dllName = ReadNullTerminatedAsciiString(reader);
                stream.Seek(currentPos, SeekOrigin.Begin);

                if (NetworkDllNames.Contains(dllName))
                {
                    hasNetwork = true;
                }
            }

            // インポート関数名の走査 (OriginalFirstThunk または FirstThunk)
            int thunkRva = originalFirstThunkRva != 0 ? originalFirstThunkRva : firstThunkRva;
            int thunkOffset = RvaToFileOffset(peReader, thunkRva);
            if (thunkOffset > 0 && thunkOffset < stream.Length)
            {
                long descriptorPos = stream.Position;
                stream.Seek(thunkOffset, SeekOrigin.Begin);

                bool is64Bit = peReader.PEHeaders.PEHeader?.Magic == PEMagic.PE32Plus;
                int entrySize = is64Bit ? 8 : 4;

                while (stream.Position + entrySize <= stream.Length)
                {
                    long thunkValue = is64Bit ? reader.ReadInt64() : reader.ReadInt32();
                    if (thunkValue == 0) break;

                    long ordinalMask = is64Bit ? unchecked((long)0x8000000000000000) : unchecked((int)0x80000000);
                    if ((thunkValue & ordinalMask) == 0) // 名前によるインポート
                    {
                        int hintNameRva = (int)(thunkValue & 0x7FFFFFFF);
                        int hintNameOffset = RvaToFileOffset(peReader, hintNameRva);
                        if (hintNameOffset > 0 && hintNameOffset + 2 < stream.Length)
                        {
                            long thunkPos = stream.Position;
                            stream.Seek(hintNameOffset + 2, SeekOrigin.Begin); // Hint (2 bytes) をスキップして関数名へ
                            string funcName = ReadNullTerminatedAsciiString(reader);
                            stream.Seek(thunkPos, SeekOrigin.Begin);

                            if (ProcessCreationApiNames.Contains(funcName))
                            {
                                hasProcessCreation = true;
                            }
                        }
                    }
                }

                stream.Seek(descriptorPos, SeekOrigin.Begin);
            }
        }
    }

    private static int RvaToFileOffset(PEReader peReader, int rva)
    {
        foreach (var section in peReader.PEHeaders.SectionHeaders)
        {
            if (rva >= section.VirtualAddress && rva < section.VirtualAddress + section.VirtualSize)
            {
                return (rva - section.VirtualAddress) + section.PointerToRawData;
            }
        }
        return -1;
    }

    private static string ReadNullTerminatedAsciiString(BinaryReader reader)
    {
        var sb = new StringBuilder();
        while (reader.BaseStream.Position < reader.BaseStream.Length)
        {
            byte b = reader.ReadByte();
            if (b == 0) break;
            sb.Append((char)b);
            if (sb.Length > 256) break; // 異常な長さの防御
        }
        return sb.ToString();
    }

    private static string DetermineCategory(string fileName) => fileName switch
    {
        "dxgi.dll" or "d3d11.dll" or "d3d9.dll" or "d3d12.dll" => "DirectXHook",
        "dinput8.dll" or "xinput1_3.dll" or "xinput1_4.dll"     => "InputHook",
        "version.dll" or "winmm.dll" or "binkw64.dll"          => "SystemProxy",
        _                                                      => "GenericMod"
    };
}
```

## 5.2 NTFS Zone.Identifier 読み取り Adapter (`ZoneIdentifierReader.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Native;

using System;
using System.IO;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;

public sealed class ZoneIdentifierReader : IZoneIdentifierReader
{
    /// <summary>
    /// NTFS 代替データストリーム (:Zone.Identifier) からブラウザのダウンロード元情報を完全ローカル取得
    /// </summary>
    public ZoneIdentifierDto? ReadZoneIdentifier(string filePath)
    {
        string adsPath = filePath + ":Zone.Identifier";
        if (!File.Exists(adsPath))
        {
            return null;
        }

        try
        {
            int zoneId = -1;
            string? hostUrl = null;
            string? referrerUrl = null;

            foreach (var line in File.ReadAllLines(adsPath))
            {
                var trimmed = line.Trim();
                if (trimmed.StartsWith("ZoneId=", StringComparison.OrdinalIgnoreCase))
                {
                    int.TryParse(trimmed["ZoneId=".Length..], out zoneId);
                }
                else if (trimmed.StartsWith("HostUrl=", StringComparison.OrdinalIgnoreCase))
                {
                    hostUrl = trimmed["HostUrl=".Length..];
                }
                else if (trimmed.StartsWith("ReferrerUrl=", StringComparison.OrdinalIgnoreCase))
                {
                    referrerUrl = trimmed["ReferrerUrl=".Length..];
                }
            }

            return new ZoneIdentifierDto(zoneId, hostUrl, referrerUrl);
        }
        catch
        {
            return null; // 非 NTFS ボリュームまたはアクセス権限不足
        }
    }
}
```

## 5.3 汎用チェックサム ＆ アーカイブ突合 Adapter (`ArchiveIntegrityMatcher.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Native;

using System;
using System.Collections.Generic;
using System.IO;
using System.IO.Compression;
using System.Security.Cryptography;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Interfaces;

public sealed class ArchiveIntegrityMatcher : IArchiveIntegrityMatcher
{
    public async Task<bool> MatchFileAgainstArchiveAsync(
        string targetFilePath, 
        string archiveFilePath, 
        CancellationToken ct = default)
    {
        if (!File.Exists(targetFilePath) || !File.Exists(archiveFilePath)) return false;

        string targetFileName = Path.GetFileName(targetFilePath);
        string targetSha256 = await ComputeSha256SafelyAsync(targetFilePath, ct);
        if (string.IsNullOrEmpty(targetSha256)) return false;

        try
        {
            await using var fs = new FileStream(archiveFilePath, FileMode.Open, FileAccess.Read, FileShare.Read, 81920, useAsync: true);
            using var zip = new ZipArchive(fs, ZipArchiveMode.Read);

            foreach (var entry in zip.Entries)
            {
                if (string.Equals(entry.Name, targetFileName, StringComparison.OrdinalIgnoreCase))
                {
                    await using var entryStream = entry.Open();
                    byte[] entryHashBytes = await SHA256.HashDataAsync(entryStream, ct);
                    string entrySha256 = Convert.ToHexString(entryHashBytes);

                    return string.Equals(targetSha256, entrySha256, StringComparison.OrdinalIgnoreCase);
                }
            }
        }
        catch
        {
            return false;
        }

        return false;
    }

    public async Task<IReadOnlyDictionary<string, string>> ExtractChecksumsFromTextAsync(
        string checksumTextFilePath, 
        CancellationToken ct = default)
    {
        var result = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);
        if (!File.Exists(checksumTextFilePath)) return result;

        var lines = await File.ReadAllLinesAsync(checksumTextFilePath, ct);
        foreach (var line in lines)
        {
            var trimmed = line.Trim();
            if (string.IsNullOrWhiteSpace(trimmed) || trimmed.StartsWith('#')) continue;

            // 一般的なフォーマット: "<SHA256>  <FileName>" または "<SHA256> *<FileName>"
            var parts = trimmed.Split([' ', '\t', '*'], StringSplitOptions.RemoveEmptyEntries);
            if (parts.Length >= 2 && parts[0].Length == 64)
            {
                string hash = parts[0];
                string fileName = Path.GetFileName(parts[^1]);
                result[fileName] = hash;
            }
        }

        return result;
    }

    private static async Task<string> ComputeSha256SafelyAsync(string filePath, CancellationToken ct)
    {
        try
        {
            await using var stream = new FileStream(filePath, FileMode.Open, FileAccess.Read, FileShare.ReadWrite | FileShare.Delete, 81920, useAsync: true);
            byte[] hash = await SHA256.HashDataAsync(stream, ct);
            return Convert.ToHexString(hash);
        }
        catch
        {
            return string.Empty;
        }
    }
}
```

---

# 6. Presentation / UI UX 仕様

### 6.0.1 User-Facing Diagnostic Vocabulary

ユーザー向けMOD診断は、静的解析・出所・署名・構造情報から観測できた事実と不確実性を明示する。主表示は `確認できた情報 / 注意すべき兆候 / 未確認事項` とし、内部のscoreやconfidence値を確定的な「安全」「危険」判定として表示しない。未署名であること自体を悪性証拠とはみなさず、Trust List登録はユーザー確認後の選択として扱う。

## 6.1 未署名 MOD 診断カード (`ModDiagnosisCardView`)

```text
+-----------------------------------------------------------------------------------------+
| 🔍 未署名 DLL の自動診断結果 (完全ローカル解析)                                         |
+-----------------------------------------------------------------------------------------+
| 対象ファイル: dxgi.dll (未署名 / 64-bit DLL)                                            |
| 検知場所:     C:\Games\Cyberpunk 2077\bin\x64\dxgi.dll                                  |
+-----------------------------------------------------------------------------------------+
| 【客観的証拠 ＆ レントゲン診断】                                                        |
|  ✅ 入手元出所:  github.com (ブラウザの正規ダウンロード履歴と一致)                      |
|  ✅ バイナリ構造: DirectX 11/12 描画プロキシ DLL 構造を確認                             |
|  ✅ 安全シグナル: 不審な外部通信 (ws2_32) やコマンド実行 (cmd.exe) コードなし          |
|                                                                                         |
| 💡 【説明】確認できた出所・構造・署名・API情報をもとに、既知の一致点と注意点を表示します。       |
+-----------------------------------------------------------------------------------------+
| [ 🛡️ この MOD を信頼してベースライン登録 ]   [ 📦 元 ZIP と手動照合 ]   [ 今回のみ無視 ] |
+-----------------------------------------------------------------------------------------+
```

## 6.2 汎用チェックサム・アーカイブ手動突合ダイアログ

```text
+-----------------------------------------------------------------------------------------+
| 📦 MOD 手動照合 (ドラッグ＆ドロップ照合)                                                |
+-----------------------------------------------------------------------------------------+
| 以下のいずれかをこのウィンドウへドラッグ＆ドロップしてください:                        |
|                                                                                         |
| 1. 公式サイトからダウンロードした元の配布アーカイブ (.zip / 各種圧縮形式)                 |
| 2. 開発者が公開しているチェックサムテキスト (SHA256SUMS / checksums.txt)                |
| 3. Wabbajack / Nexus Collections のマニフェストファイル (.json / .wabbajack)           |
|                                                                                         |
| ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐  |
|   ここに対象ファイルをドロップ                                                          |
| └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘  |
+-----------------------------------------------------------------------------------------+
```

---

# 7. 監査ログ規約 (`AuditEventType`)

共通 Enum に以下のイベントを割り当て、Hash Chain へ記録する。

- `ModInspected = 60`: 未署名 MOD の静的レントゲン診断完了
- `ModProvenanceVerified = 61`: Zone.Identifier 出所または元 ZIP 突合による出所証明完了
- `ModManifestImported = 62`: Wabbajack / Collections 定義からの一括ベースライン登録完了

---

# 8. 単体テスト仕様 (`GST.UnitTests.Security.ModProvenance`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.Security.ModProvenance;

using System;
using System.Collections.Generic;
using GameSecurityTool.Domain.Models.ModSecurity;
using GameSecurityTool.Domain.Services;
using Xunit;

public class ModSecurityEvaluatorTests
{
    [Fact]
    public void Evaluate_WhenValidProxyDllWithBrowserProvenance_ReturnsLikelySafe()
    {
        var evaluator = new ModSecurityEvaluator();
        var signals = new ModInspectionSignals(
            IsProxyExportValid: true,
            HookCategory: ProxyDllCategory.DirectXHook,
            HasSuspiciousNetworkImports: false,
            HasProcessCreationImports: false,
            Provenance: ModProvenanceSource.BrowserDownload,
            DownloadHostUrl: "https://github.com/crosire/reshade/releases"
        );

        var result = evaluator.Evaluate(signals);

        Assert.True(result.IsLikelySafeMod);
        Assert.True(result.ConfidenceScore >= 80);
        Assert.Contains("ValidProxyDllStructure", result.ReasonCodes);
    }

    [Fact]
    public void Evaluate_WhenProxyDllContainsNetworkApi_RejectsSafety()
    {
        var evaluator = new ModSecurityEvaluator();
        var signals = new ModInspectionSignals(
            IsProxyExportValid: true,
            HookCategory: ProxyDllCategory.DirectXHook,
            HasSuspiciousNetworkImports: true, // 不審通信あり
            HasProcessCreationImports: false,
            Provenance: ModProvenanceSource.Unknown,
            DownloadHostUrl: null
        );

        var result = evaluator.Evaluate(signals);

        Assert.False(result.IsLikelySafeMod);
        Assert.Contains("ContainsSuspiciousNetworkApi", result.ReasonCodes);
    }
}
```

---

# 9. ロードマップ反映

- **MVP Alpha (Phase 2):** Layer 1（PE 静的構造診断 ＆ `Zone.Identifier` 出所解析）をスキャナーへ統合。
- **MVP Beta (Phase 3):** Layer 2（汎用 `SHA256SUMS` ＆ 元 ZIP 自己突合）を UI へ配備。
- **v1.0 Core (Phase 4):** Layer 3（Wabbajack / Collections マニフェストインポーター）を配備。

---

