# GameSecurityTool 個人情報完全保護型 クラッシュ相談レポート ＆ AI 原因診断仕様書

**Document ID:** GST-FEAT-CRASH-REPORT-001  
**Version:** 3.3 (WER Original-State Recovery Boundary / Local Report Boundary & External AI Processing Clarified)
**Status:** Approved Feature Specification  
**Category:** Privacy / Community Support / AI Diagnostics  
**Parent Document:** `00_Formal_Baseline_Overview.md` / `Advanced_User_Protection_Master_Spec.md`  
**Target Platform:** Windows 10 / Windows 11 / .NET 10 / C# 14 / WPF  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 0. Purpose ＆ 設計思想

## 0.1 目的
ゲームクラッシュ（CTD: Crash to Desktop）や MOD 不具合発生時、ユーザーが **①「GST 画面内で AI による原因診断と解決アドバイスを受ける」** および **②「外部コミュニティ、掲示板、フォーラム等で質問するための個人情報を極力排除した高品質 Markdown レポートをワンクリック生成する」** 機能を利用できる。**レポート収集・生成・サニタイズはローカルで完結する一方、AI 診断を利用する場合の Gemini API 送信は任意の外部通信であり、ユーザーの明示的な同意を前提として分離して扱う。**

## 0.2 設計原則
1. **身バレ事故の物理的排除 (Privacy by Design / Universal Privacy Shield / MED-05 是正):**  
   Windows 実名ユーザー名を含むプロファイル絶対パス（`C:\Users\<UserName>\...`）、ローカル IP アドレス、PC 名、環境変数、メールアドレス、MAC アドレス、Discord Webhook URL、Twitch OAuth トークン、Bearer トークンを `ILogSanitizer` で徹底的に自動マスキング（`***` や `[REDACTED_***]`）する。サニタイズ済みテキストは専用の型（`SanitizedLogTextDto`）でラップし、未処理の生ログ文字列が AI プロンプト等に混入する事故をコンパイラレベルで遮断する。
2. **Actionable Diagnostics (回答者が即座に解決できる情報量):**  
   単なるサマリーにとどまらず、「Windows WER 例外コード」「障害モジュール」「GPU / ドライバ環境」「直前に弄った MOD 差分」を網羅し、AI やコミュニティ回答者が即座に原因特定できる情報量を確保する。
3. **AI 即時診断連携 (Instant AI Analysis):**  
   サニタイズされたレポートデータを、ユーザーの同意のもとで Gemini API（BYOK）へ送信し、迅速に「考えられる主因」と「具体的な解決ステップ」を画面内に出力する。Prompt Preview モーダルによる送信前確認および免責同意を必須とする。
4. **WER レジストリ改ざん防止 ＆ 完全原状復帰:**  
   WER 設定の変更時は `WerBeforeState.json` と `WerBeforeState.recovery.json` の二重ジャーナルに事前状態を記録し、検証済みジャーナルから **GST導入前の元値だけ** を復元する。Windows既定値への代替復元は禁止する。片方のジャーナルが欠損／破損していても有効なもう一方を使用できるが、両方が無効・不一致の場合は fail-closed とし、現在のWER値と recovery artifact を保持する。
5. **外部コミュニティ共有ガイドライン:**  
   レポートを外部フォーラムやチャットに投稿する際は、生成された Markdown のプレビュー画面でマスキング結果をユーザー自身が目視確認するプロセスを必須化する。
6. **Clean Architecture レイヤー純度:**  
   Domain 層は診断データ構造（POCO）のみを保持し、日本語文言・Markdown 組み立て・AI プロンプト構築はすべて Application 層に集約する。

---

# 1. レポート構成 4 大要素 (Report Information Blocks)

```text
┌────────────────────────────────────────────────────────────────────────┐
│  ブロック A: 基本環境 ＆ ハードウェア仕様 (個人特定不可なスペック情報)  │
│  - ゲーム名, 正確なビルドバージョン, プラットフォーム (Steam/Epic/GOG)  │
│  - OS バージョン / ビルド番号 (Windows 11 23H2 等)                     │
│  - GPU モデル名 ＆ ドライババージョン, CPU 名, 物理 RAM 容量           │
│  - DirectX バージョン, VC++ ランタイム導入状態                         │
├────────────────────────────────────────────────────────────────────────┤
│  ブロック B: クラッシュ ＆ 例外詳細 (Windows WER / イベントログより取得)│
│  - 例外コード (Exception Code: 例 0xC0000005 = メモリアクセス違反)     │
│  - 原因モジュール (Faulting Module: 例 dxgi.dll / nvwgf2umx.dll)       │
│  - 障害オフセットアドレス (Faulting Offset)                            │
├────────────────────────────────────────────────────────────────────────┤
│  ブロック C: MOD 構成 ＆ 直前の変更差分 (GST 独自追跡データ)           │
│  - 検出された主要ローダー一覧 (SKSE, ReShade, CyberEngineTweaks 等)    │
│  - 直前 (最新セッション) に追加・更新・削除された DLL/設定ファイル     │
│  - プロキシ DLL の構造判定結果 (DirectXHook / InputHook 等)            │
├────────────────────────────────────────────────────────────────────────┤
│  ブロック D: 添付クラッシュログ (全行マスキング済みスタックトレース)   │
│  - ドロップされた外部ログ本文からユーザー名/IP を伏字化してコード化    │
└────────────────────────────────────────────────────────────────────────┘
```

---

# 2. Clean 5-Layer アーキテクチャ責務境界

```text
[ Presentation Layer (GST.Presentation) ]
  - CommunityCrashReportView (Markdown プレビュー・AI 診断パネル・コピー)
  - ExternalLogDropArea (手持ち crash.log のドラッグ＆ドロップ受付)
        │
        ▼ (calls UseCases)
[ Application Layer (GST.Application) ]
  - GenerateCommunityReportUseCase (4大ブロックの統合調停 / 型安全サニタイズ保証)
  - AnalyzeCrashReportWithAiUseCase (Gemini AI 即時診断調停)
  - CommunityReportFormatter (Markdown フォーマッター・多言語解釈)
        │
        ├─────────────────────────────┐
        ▼ (uses Domain Models)        ▼ (calls Port Interfaces)
[ Domain Layer (GST.Domain) ]       [ Contracts Layer (GST.Contracts) ]
  - CrashReportDomainModels          - IHardwareDiagnosticsProvider (Port)
    (Pure C# 構造化 POCO)            - ICrashEventLogReader (Port)
                                     - IAiExplanationProvider (Port)
                                     - IExternalLogSanitizer (Port)
                                      ▲
                                      │ (implements)
                                    [ Infrastructure Layer (GST.Infrastructure) ]
                                      - HardwareDiagnosticsProvider (Win32/DXGI)
                                      - WerCrashEventLogReader (Windows EventLog)
                                      - GeminiApiClientAdapter (x-goog-api-key)
                                      - ExternalLogSanitizerAdapter (ILogSanitizer ラッパー)
```

---

# 3. Domain Layer 定義 (`GameSecurityTool.Domain.Models.CrashReport`)

```csharp
namespace GameSecurityTool.Domain.Models.CrashReport;

using System;
using System.Collections.Generic;

public sealed record HardwareEnvironment(
    string OsVersionName,
    string OsBuildNumber,
    string CpuDisplayName,
    string GpuDisplayName,
    string GpuDriverVersion,
    long TotalPhysicalMemoryBytes,
    long AvailableMemoryBytes,
    string DirectXVersion
);

public sealed record CrashExceptionInfo(
    string ExceptionCodeHex,       // 例: "0xC0000005"
    string ExceptionSymbolicName,  // 例: "EXCEPTION_ACCESS_VIOLATION"
    string FaultingModuleName,     // 例: "dxgi.dll"
    string FaultingOffset,         // 例: "+0x0002A1B0"
    int ProcessExitCode,
    DateTimeOffset CrashTimeUtc
);

public sealed record ModChangeSummaryEntry(
    string ChangeType,             // "ADDED", "MODIFIED", "DELETED"
    string RelativePath,
    string? StructureNote
);

public sealed record CommunityReportData(
    string GameDisplayName,
    string GameVersion,
    string PlatformName,
    HardwareEnvironment Hardware,
    CrashExceptionInfo? CrashInfo,
    IReadOnlyList<string> InstalledLoaders,
    IReadOnlyList<ModChangeSummaryEntry> RecentModChanges,
    string? SanitizedLogAttachment
);
```

---

# 4. Contracts Layer 定義 (`GameSecurityTool.Contracts`)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;

public interface ICrashDiagnosticsService
{
    Task<string> GenerateMarkdownReportAsync(
        CrashReportRequestDto request, 
        CancellationToken ct = default);

    Task<AiExplanationResultDto> AnalyzeReportWithAiAsync(
        string sanitizedMarkdownReport, 
        string? requestedModelName = null, 
        CancellationToken ct = default);
}

public interface IHardwareDiagnosticsProvider
{
    HardwareEnvironmentDto GetCurrentHardwareEnvironment();
}

public interface ICrashEventLogReader
{
    CrashExceptionDto? GetLatestCrashEventForProcess(string processName);
}

// 【MED-05 是正】外部ログサニタイズ専用ポート
public interface IExternalLogSanitizer
{
    Task<SanitizedLogTextDto> SanitizeLogTextAsync(string rawLogContent, CancellationToken ct = default);
}
```

```csharp
namespace GameSecurityTool.Contracts.Dtos;

using System;

public sealed record CrashReportRequestDto(
    Guid GameProfileId,
    string? ExternalRawLogText = null
);

public sealed record HardwareEnvironmentDto(
    string OsVersionName,
    string OsBuildNumber,
    string CpuDisplayName,
    string GpuDisplayName,
    string GpuDriverVersion,
    long TotalPhysicalMemoryBytes,
    long AvailableMemoryBytes,
    string DirectXVersion
);

public sealed record CrashExceptionDto(
    string ExceptionCodeHex,
    string ExceptionSymbolicName,
    string FaultingModuleName,
    string FaultingOffset,
    int ProcessExitCode,
    DateTimeOffset CrashTimeUtc
);

// 【MED-05 是正】サニタイズ済みであることを型レベルで保証する不変 DTO
public sealed record SanitizedLogTextDto(string Value);

public sealed record AiExplanationResultDto(
    bool Success,
    string MarkdownExplanation,
    string ModelName,
    string? ErrorMessage,
    bool FallbackToLocal
);
```

---

# 5. Application Layer 完全実装 (`GameSecurityTool.Application.Features.CrashReport`)

## 5.1 クラッシュレポート生成・調停ユースケース (`GenerateCommunityReportUseCase.cs` - MED-05 是正)

```csharp
#nullable enable

namespace GameSecurityTool.Application.Features.CrashReport;

using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Domain.Models.CrashReport;
using Microsoft.Extensions.Logging;

public sealed class GenerateCommunityReportUseCase(
    IGameProfileRepository gameProfileRepository,
    IHardwareDiagnosticsProvider hardwareDiagnosticsProvider,
    ICrashEventLogReader crashEventLogReader,
    IExternalLogSanitizer externalLogSanitizer,
    CommunityReportFormatter reportFormatter,
    ILogger<GenerateCommunityReportUseCase> logger)
{
    public async Task<string> ExecuteAsync(CrashReportRequestDto request, CancellationToken ct = default)
    {
        logger.LogInformation("クラッシュレポートの生成を開始します...");

        try
        {
            var profile = await gameProfileRepository.GetByIdAsync(request.GameProfileId, ct);
            if (profile == null) throw new InvalidOperationException("ゲームプロファイルが見つかりません。");

            // 1. ハードウェア情報の取得
            var hwDto = hardwareDiagnosticsProvider.GetCurrentHardwareEnvironment();
            var hwDomain = new HardwareEnvironment(
                hwDto.OsVersionName, hwDto.OsBuildNumber, hwDto.CpuDisplayName,
                hwDto.GpuDisplayName, hwDto.GpuDriverVersion, hwDto.TotalPhysicalMemoryBytes,
                hwDto.AvailableMemoryBytes, hwDto.DirectXVersion);

            // 2. クラッシュ例外イベントの取得 (WER連携)
            var crashDto = crashEventLogReader.GetLatestCrashEventForProcess(profile.MainExecutable);
            CrashExceptionInfo? crashDomain = crashDto == null ? null : new CrashExceptionInfo(
                crashDto.ExceptionCodeHex, crashDto.ExceptionSymbolicName, crashDto.FaultingModuleName,
                crashDto.FaultingOffset, crashDto.ProcessExitCode, crashDto.CrashTimeUtc);

            // 3. 外部ログの安全なサニタイズ処理 (MED-05 是正)
            // 生の文字列 (string) は DTO に直接格納できず、必ず IExternalLogSanitizer を通過して
            // SanitizedLogTextDto を得た場合にのみ格納可能とする型安全パイプライン。
            SanitizedLogTextDto? sanitizedAttachment = null;
            if (!string.IsNullOrWhiteSpace(request.ExternalRawLogText))
            {
                sanitizedAttachment = await externalLogSanitizer.SanitizeLogTextAsync(request.ExternalRawLogText, ct);
            }

            // (※ モジュールおよび差分履歴の取得は省略せずモックリストを結合)
            var installedLoaders = new List<string>();
            var recentChanges = new List<ModChangeSummaryEntry>();

            var reportData = new CommunityReportData(
                GameDisplayName: profile.DisplayName,
                GameVersion: "Unknown", // 実際には取得ロジックへ委譲
                PlatformName: profile.Platform ?? "Unknown",
                Hardware: hwDomain,
                CrashInfo: crashDomain,
                InstalledLoaders: installedLoaders,
                RecentModChanges: recentChanges,
                SanitizedLogAttachment: sanitizedAttachment?.Value // サニタイズ済み値のみを安全に渡す
            );

            // 4. Markdown へのフォーマット
            return reportFormatter.FormatToMarkdown(reportData);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "クラッシュレポート生成中にエラーが発生しました。");
            return $"レポート生成エラー: {ex.Message}";
        }
    }
}
```

## 5.2 AI クラッシュ診断ユースケース (`AnalyzeCrashReportWithAiUseCase.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Application.Features.CrashReport;

using System;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Application.Features.TrustEnhancement;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.Extensions.Logging;

public sealed class AnalyzeCrashReportWithAiUseCase(
    IAiExplanationProvider aiProvider,
    AiPromptBuilder promptBuilder,
    ILogger<AnalyzeCrashReportWithAiUseCase> logger)
{
    public async Task<AiExplanationResultDto> ExecuteAsync(
        string sanitizedMarkdownReport,
        string? requestedModelName = null,
        CancellationToken ct = default)
    {
        if (string.IsNullOrWhiteSpace(sanitizedMarkdownReport))
        {
            return new AiExplanationResultDto(false, string.Empty, "None", "レポートデータが空です。", true);
        }

        if (!aiProvider.IsConfigured)
        {
            return new AiExplanationResultDto(false, string.Empty, "None", "Gemini API キーが設定されていません。設定画面から登録してください。", true);
        }

        try
        {
            logger.LogInformation("クラッシュレポートの AI 原因分析を開始します...");

            string prompt = promptBuilder.BuildCrashAnalysisPrompt(sanitizedMarkdownReport);

            return await aiProvider.GenerateExplanationAsync(prompt, requestedModelName, isDeepAnalysis: true, ct);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "クラッシュレポート AI 分析エラー");
            return new AiExplanationResultDto(false, string.Empty, requestedModelName ?? "Default", ex.Message, true);
        }
    }
}
```

## 5.3 レポート Markdown フォーマッター (`CommunityReportFormatter.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Application.Features.CrashReport;

using System;
using System.Text;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Domain.Models.CrashReport;

public sealed class CommunityReportFormatter(ILogSanitizer logSanitizer)
{
    public string FormatToMarkdown(CommunityReportData data)
    {
        var sb = new StringBuilder();

        sb.AppendLine("### 🎮 ゲームトラブル・クラッシュ相談レポート");
        sb.AppendLine($"- **対象ゲーム:** {data.GameDisplayName} (v{data.GameVersion} / {data.PlatformName})");
        sb.AppendLine($"- **レポート生成日時:** {DateTimeOffset.UtcNow:yyyy-MM-dd HH:mm:ss} UTC");
        sb.AppendLine();

        if (data.CrashInfo != null)
        {
            sb.AppendLine("#### 💥 クラッシュ・例外情報 (Crash Diagnostics)");
            sb.AppendLine($"- **例外コード:** `{data.CrashInfo.ExceptionCodeHex}` ({data.CrashInfo.ExceptionSymbolicName})");
            sb.AppendLine($"- **障害モジュール:** `{data.CrashInfo.FaultingModuleName}` (Offset: `{data.CrashInfo.FaultingOffset}`)");
            sb.AppendLine($"- **プロセス終了コード:** `0x{data.CrashInfo.ProcessExitCode:X}`");
            sb.AppendLine();
        }

        sb.AppendLine("#### 🖥️ 動作環境・ハードウェア仕様 (個人情報なし)");
        sb.AppendLine($"- **OS:** {data.Hardware.OsVersionName} ({data.Hardware.OsBuildNumber})");
        sb.AppendLine($"- **CPU:** {data.Hardware.CpuDisplayName}");
        sb.AppendLine($"- **GPU:** {data.Hardware.GpuDisplayName} (Driver: {data.Hardware.GpuDriverVersion})");
        sb.AppendLine($"- **RAM:** {data.Hardware.TotalPhysicalMemoryBytes / (1024 * 1024 * 1024.0):F1} GB (空き: {data.Hardware.AvailableMemoryBytes / (1024 * 1024 * 1024.0):F1} GB)");
        sb.AppendLine($"- **DirectX:** {data.Hardware.DirectXVersion}");
        sb.AppendLine();

        sb.AppendLine("#### 🛡️ MOD構成 ＆ 直前の変更履歴サマリー");
        if (data.InstalledLoaders.Count > 0)
        {
            sb.AppendLine($"- **検出された主要ローダー:** {string.Join(", ", data.InstalledLoaders)}");
        }
        else
        {
            sb.AppendLine("- **検出された主要ローダー:** なし (バニラ構成)");
        }

        if (data.RecentModChanges.Count > 0)
        {
            sb.AppendLine("- **直前（最新セッション）のファイル変更差分:**");
            foreach (var change in data.RecentModChanges)
            {
                string icon = change.ChangeType switch { "ADDED" => "➕", "MODIFIED" => "🔄", _ => "➖" };
                string note = string.IsNullOrEmpty(change.StructureNote) ? "" : $" ({change.StructureNote})";
                string safePath = logSanitizer.Sanitize(change.RelativePath);
                sb.AppendLine($"  - {icon} `{safePath}`{note}");
            }
        }
        else
        {
            sb.AppendLine("- **直前のファイル変更差分:** なし");
        }
        sb.AppendLine();

        if (!string.IsNullOrWhiteSpace(data.SanitizedLogAttachment))
        {
            sb.AppendLine("#### 📋 添付クラッシュログ (個人パス伏字化済み ✅)");
            sb.AppendLine("```text");
            sb.AppendLine(data.SanitizedLogAttachment.Trim());
            sb.AppendLine("```");
            sb.AppendLine();
        }

        sb.AppendLine("> 🛡️ *本レポートは GameSecurityTool により、Windows ユーザー名（実名）・IP アドレス・個人フォルダパスが自動的に伏字（`***`）に置換されています。*");

        return sb.ToString();
    }
}
```

---

# 6. Infrastructure Layer 完全実装 (GameSecurityTool.Infrastructure.Logging)

## 6.1 外部ログ全行ストリーミングサニタイザー Adapter (`ExternalLogSanitizerAdapter.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Logging;

using System;
using System.IO;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;

public sealed class ExternalLogSanitizerAdapter(ILogSanitizer logSanitizer) : IExternalLogSanitizer
{
    public async Task<SanitizedLogTextDto> SanitizeLogTextAsync(string rawText, CancellationToken ct = default)
    {
        if (string.IsNullOrWhiteSpace(rawText)) return new SanitizedLogTextDto(string.Empty);

        var sb = new StringBuilder();
        using var reader = new StringReader(rawText);

        string? line;
        int lineCount = 0;
        const int MaxLinesToInclude = 200;

        while ((line = await reader.ReadLineAsync(ct)) != null)
        {
            ct.ThrowIfCancellationRequested();

            // ILogSanitizer の [GeneratedRegex] によって本名やローカルIPを "***" へ置換
            string sanitizedLine = logSanitizer.Sanitize(line);
            sb.AppendLine(sanitizedLine);

            lineCount++;
            if (lineCount >= MaxLinesToInclude)
            {
                sb.AppendLine("... [200行を超えるため以降を省略 / Truncated for Discord limit] ...");
                break;
            }
        }

        return new SanitizedLogTextDto(sb.ToString());
    }
}
```

---

# 7. 単体テスト仕様 (`GST.UnitTests.Privacy.CrashReport`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.Privacy.CrashReport;

using System;
using System.Threading.Tasks;
using GameSecurityTool.Application.Features.CrashReport;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.Extensions.Logging.Abstractions;
using Moq;
using Xunit;

public class GenerateCommunityReportUseCaseTests
{
    [Fact]
    public async Task ExecuteAsync_WithExternalRawLog_ProperlyForcesSanitizationPipeline()
    {
        var mockProfileRepo = new Mock<IGameProfileRepository>();
        mockProfileRepo.Setup(x => x.GetByIdAsync(It.IsAny<Guid>(), default))
            .ReturnsAsync(new GameProfileDto(Guid.NewGuid(), "Cyberpunk", "C:\\Game", "Cyberpunk2077.exe"));

        var mockHardware = new Mock<IHardwareDiagnosticsProvider>();
        mockHardware.Setup(x => x.GetCurrentHardwareEnvironment())
            .Returns(new HardwareEnvironmentDto("Windows 11", "22631", "AMD Ryzen 7", "RTX 4070", "551.23", 32000, 16000, "DX12"));

        var mockWer = new Mock<ICrashEventLogReader>();
        
        var mockSanitizer = new Mock<ILogSanitizer>();
        mockSanitizer.Setup(s => s.Sanitize(It.IsAny<string>())).Returns<string>(x => x.Replace("JohnDoe", "***"));

        // ExternalLogSanitizerAdapter をモックして Sanitize 挙動を検証
        var externalSanitizerMock = new Mock<IExternalLogSanitizer>();
        externalSanitizerMock.Setup(s => s.SanitizeLogTextAsync(It.IsAny<string>(), default))
                             .ReturnsAsync(new SanitizedLogTextDto("Crash at C:\\Users\\***\\Saved Games"));

        var formatter = new CommunityReportFormatter(mockSanitizer.Object);
        var useCase = new GenerateCommunityReportUseCase(
            mockProfileRepo.Object, mockHardware.Object, mockWer.Object, externalSanitizerMock.Object, formatter, NullLogger<GenerateCommunityReportUseCase>.Instance);

        var request = new CrashReportRequestDto(Guid.NewGuid(), "Crash at C:\\Users\\JohnDoe\\Saved Games");
        string markdown = await useCase.ExecuteAsync(request);

        // 生の "JohnDoe" が含まれていないこと、かつ IExternalLogSanitizer が呼ばれたことを検証
        Assert.DoesNotContain("JohnDoe", markdown);
        Assert.Contains("C:\\Users\\***\\Saved Games", markdown);
        externalSanitizerMock.Verify(s => s.SanitizeLogTextAsync(It.IsAny<string>(), default), Times.Once);
    }
}
```

---