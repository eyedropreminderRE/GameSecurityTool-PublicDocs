# 04-02: Application Explanation Engine

**Document ID:** GST-FEAT-TRUST-002  
**Version:** 4.0 (Domain Sub-namespace Trust Synchronization Edition)
**Parent Document:** 04-00_Overview_and_Principles.md  
**Category:** Application UseCase Specification  
**Status:** Approved Feature Specification  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. 責務と配置

`ExplanationEngine` は **Application Layer** に配置され、Domain 層から受領した技術的判定結果（`ConfidenceAssessmentResult`）を、ユーザーが直感的に理解できる平易な説明文（`RiskExplanationDto`）へ変換する。

### 主な責務:
* `ConfidenceAssessmentResult` の受領とシグナル（`DetectionSignal`）の解釈
* **Domain 独自の Enum (`DomainTrustRiskLevel`) から Contracts の共有 Enum (`Contracts.Common.RiskLevel`) へのマッピング**
* 該当ゲームのプロファイル設定および適用された Rule Context の取得
* ローカライゼーション（日本語リソース）の適用と平易な文言生成
* ユーザー推奨アクション（例: 「隔離を推奨」「通信を許可」）の生成
* 個人情報自動マスキング（`ILogSanitizer`）の適用
* Presentation 層向け不変 DTO への組み立て

---

# 2. 処理フロー

```text
[ Domain Layer: TrustEnhancement ]
  ConfidenceAssessmentResult (純粋な技術シグナル: DomainTrustRiskLevel, DetectionSignals)
       │
       ▼
[ Application Layer: ExplanationEngine ]
  ├─ 1. MapToContractsRiskLevel ──> DomainTrustRiskLevel を Contracts.Common.RiskLevel へ変換
  ├─ 2. SignalTranslator        ──> 技術シグナルコードを平易な日本語解説へ翻訳
  ├─ 3. ActionAdvisor           ──> リスクレベルに応じた具体的な対処手順を生成
  ├─ 4. ILogSanitizer           ──> パス文字列の個人名 (本名等) を自動伏字化
  └─ 5. DTO 組立
       │
       ▼
[ Contracts Layer ]
  RiskExplanationDto (不変 DTO)
       │
       ▼
[ Presentation Layer: UI ]
  Warning Dialog / Threat Detail Card
```

---

# 3. UseCase インターフェース定義 (`GameSecurityTool.Application.UseCases`)

```csharp
namespace GameSecurityTool.Application.UseCases;

using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Domain.Models.TrustEnhancement;

public interface IExplanationEngine
{
    Task<RiskExplanationDto> ExplainRiskAsync(
        ConfidenceAssessmentResult assessmentResult, 
        string? gameProfileId, 
        CancellationToken ct = default
    );
}

public interface ISignalTranslator
{
    IReadOnlyList<string> TranslateSignals(IReadOnlyList<DetectionSignal> signals);
}

public interface IActionAdvisor
{
    IReadOnlyList<string> GenerateRecommendations(DomainTrustRiskLevel level, IReadOnlyList<DetectionSignal> signals);
}
```

---

# 4. Application Layer 完全実装 (`GameSecurityTool.Application.Features.TrustEnhancement`)

```csharp
#nullable enable

namespace GameSecurityTool.Application.Features.TrustEnhancement;

using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Application.UseCases;
using GameSecurityTool.Contracts.Common;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Domain.Models.TrustEnhancement;
using Microsoft.Extensions.Logging;

public sealed class ExplanationEngine(
    ISignalTranslator signalTranslator,
    IActionAdvisor actionAdvisor,
    ILogSanitizer logSanitizer,
    ILogger<ExplanationEngine> logger) : IExplanationEngine
{
    public Task<RiskExplanationDto> ExplainRiskAsync(
        ConfidenceAssessmentResult assessmentResult,
        string? gameProfileId,
        CancellationToken ct = default)
    {
        // 1. レイヤー間 Enum マッピング (DomainTrustRiskLevel -> Contracts.Common.RiskLevel)
        var contractsLevel = MapToContractsRiskLevel(assessmentResult.Level);
        string riskLevelText = contractsLevel switch
        {
            RiskLevel.Critical => "重大なリスク (Critical)",
            RiskLevel.High     => "高リスク (High)",
            RiskLevel.Medium   => "中リスク (Medium)",
            RiskLevel.Low      => "低リスク (Low)",
            _                  => "安全 (Safe)"
        };

        // 2. タイトルと要約の生成
        string displayTitle = assessmentResult.Level switch
        {
            DomainTrustRiskLevel.Critical => "⚠️ 危険な変更または不審なファイルが検出されました",
            DomainTrustRiskLevel.High     => "⚠️ 要確認: 未署名または未知のファイルが検出されました",
            DomainTrustRiskLevel.Medium   => "ℹ️ 注意: ゲームフォルダ内の変更が確認されました",
            DomainTrustRiskLevel.Low      => "ℹ️ 軽微な変更が確認されました",
            _                             => "✅ 正常: 問題は検出されませんでした"
        };

        string safePath = logSanitizer.Sanitize(assessmentResult.TargetPath);
        string summaryMessage = $"ファイル '{safePath}' に対するセキュリティ評価が完了しました。確信度: {assessmentResult.ConfidenceScore * 100:F0}%";

        // 3. シグナルの平易な日本語解説への翻訳
        var translatedReasons = signalTranslator.TranslateSignals(assessmentResult.Signals);

        // 4. ユーザー推奨アクションの生成
        var recommendedActions = actionAdvisor.GenerateRecommendations(assessmentResult.Level, assessmentResult.Signals);

        string operationId = $"GST-EXP-{DateTimeOffset.UtcNow:yyyyMMdd}-{Guid.NewGuid():N[..8]}";

        var dto = new RiskExplanationDto(
            TargetPath: safePath,
            DisplayTitle: displayTitle,
            SummaryMessage: summaryMessage,
            RiskLevelText: riskLevelText,
            ConfidencePercentage: assessmentResult.ConfidenceScore * 100,
            TranslatedReasons: translatedReasons,
            RecommendedActions: recommendedActions,
            OperationId: operationId
        );

        logger.LogInformation("リスク判定結果の説明文を生成しました: {Path} (Level: {Level})", safePath, contractsLevel);
        return Task.FromResult(dto);
    }

    private static RiskLevel MapToContractsRiskLevel(DomainTrustRiskLevel domainLevel) => domainLevel switch
    {
        DomainTrustRiskLevel.Critical => RiskLevel.Critical,
        DomainTrustRiskLevel.High     => RiskLevel.High,
        DomainTrustRiskLevel.Medium   => RiskLevel.Medium,
        DomainTrustRiskLevel.Low      => RiskLevel.Low,
        _                             => RiskLevel.Safe
    };
}

public sealed class SignalTranslator : ISignalTranslator
{
    public IReadOnlyList<string> TranslateSignals(IReadOnlyList<DetectionSignal> signals)
    {
        var list = new List<string>();
        foreach (var sig in signals)
        {
            string explanation = sig.SignalCode switch
            {
                "UnsignedNewBinary" => "公式なデジタル署名のない新しい実行ファイルです（MOD 等でよく見られます）。",
                "NewFileDetected" => "直前のゲームプレイまたは外部ツールによって新しく配置されたファイルです。",
                "TamperedAllowList" => "以前に許可されたハッシュ値と一致しません。ファイルが書き換えられた可能性があります。",
                "ContainsSuspiciousNetworkApi" => "バックグラウンドでインターネット通信を行うコードが含まれています。",
                "ContainsProcessSpawningApi" => "外部のプログラムやコマンドプロンプトを起動するコードが含まれています。",
                "DroppedOutsideGameFolder" => "ゲームの実行中に、ゲームフォルダ外のシステム領域へ書き出されました。",
                "KnownHijackDllName" => "DLL Side-Loading 攻撃で頻繁に悪用される名称です。",
                "AllowListMatched" => "ユーザーが明示的に登録した例外許可リストに合致しています。",
                _ => $"技術シグナル: {sig.SignalCode} ({sig.RawTechnicalDetail})"
            };
            list.Add(explanation);
        }
        return list;
    }
}

public sealed class ActionAdvisor : IActionAdvisor
{
    public IReadOnlyList<string> GenerateRecommendations(DomainTrustRiskLevel level, IReadOnlyList<DetectionSignal> signals)
    {
        var actions = new List<string>();

        if (level >= DomainTrustRiskLevel.High)
        {
            actions.Add("🛡️ 一時的に隔離庫へ退避させ、ゲームが正常に動作するか確認することを推奨します。");
            actions.Add("🔍 入手元（Nexus Mods, GitHub 等）の公式ハッシュ値と一致するか手動照合してください。");
        }
        else if (level == DomainTrustRiskLevel.Medium)
        {
            actions.Add("ℹ️ 心当たりのある MOD の導入である場合は、信頼リストへ登録できます。");
            actions.Add("🌐 外部通信を遮断した「厳格モード」でのゲーム起動を検討してください。");
        }
        else
        {
            actions.Add("✅ 特別な対処は不要です。そのままゲームをプレイできます。");
        }

        return actions;
    }
}
```

---

# 5. 単体テスト仕様 (`GST.UnitTests.Trust.Explanation`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.Trust.Explanation;

using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using GameSecurityTool.Application.Features.TrustEnhancement;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Domain.Models.TrustEnhancement;
using Microsoft.Extensions.Logging.Abstractions;
using Moq;
using Xunit;

public class ExplanationEngineTests
{
    [Fact]
    public async Task ExplainRiskAsync_MapsDomainSignalsToUserFriendlyExplanation()
    {
        var mockSanitizer = new Mock<ILogSanitizer>();
        mockSanitizer.Setup(s => s.Sanitize(It.IsAny<string>())).Returns<string>(x => x.Replace("UserA", "***"));

        var translator = new SignalTranslator();
        var advisor = new ActionAdvisor();
        var engine = new ExplanationEngine(translator, advisor, mockSanitizer.Object, NullLogger<ExplanationEngine>.Instance);

        var signals = new List<DetectionSignal>
        {
            new("UnsignedNewBinary", "Binary", 30.0, "No valid signature"),
            new("ContainsSuspiciousNetworkApi", "Network", 50.0, "ws2_32.dll import")
        };

        var domainResult = new ConfidenceAssessmentResult(
            TargetPath: "C:\\Users\\UserA\\Game\\mod.dll",
            FileHash: "HASH123",
            Level: DomainTrustRiskLevel.High,
            ConfidenceScore: 0.85,
            Signals: signals,
            EvaluatedAtUtc: DateTimeOffset.UtcNow
        );

        var explanation = await engine.ExplainRiskAsync(domainResult, null);

        Assert.Contains("***", explanation.TargetPath);
        Assert.DoesNotContain("UserA", explanation.TargetPath);
        Assert.Equal("高リスク (High)", explanation.RiskLevelText);
        Assert.Equal(85.0, explanation.ConfidencePercentage);
        Assert.Contains(explanation.TranslatedReasons, r => r.Contains("デジタル署名のない"));
        Assert.Contains(explanation.RecommendedActions, a => a.Contains("隔離庫へ退避"));
    }
}
```

---

End of Document
```

---