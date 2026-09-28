# 04-01: Domain Confidence & Detection Signals Model

**Document ID:** GST-FEAT-TRUST-001  
**Parent Document:** 04-00_Overview_and_Principles.md  
**Category:** Domain Model Specification  
**Version:** 4.0 (Domain Sub-namespace Isolation & Confidence Model Hardened Edition)
**Status:** Approved Feature Specification  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Detection Engine / Explanation Engine 分離原則

Detection Engine（検知エンジン）と User Explanation（ユーザー向け説明）を Clean Architecture レベルで厳格に分離する。

```text
【Domain Layer (Pure C#)】
  Detection Signals ──> [ Domain Evaluator ] ──> ConfidenceAssessmentResult (Raw Metrics / POCO)
                                                      │
                                                      ▼
【Application Layer】
  ConfidenceAssessmentResult ──> [ Explanation Engine ] ──> RiskExplanationDto (Localized Strings)
                                                            │
                                                            ▼
【Presentation Layer】
  RiskExplanationDto ──> [ ViewModel / View ] ──> UI Display (User Friendly Card)
```

### Domain 層の禁止事項
* UI 表示用文言（「危険なファイルです」「アクセスが拒否されました」等）の保持を完全に禁止。
* 多言語リソース（`Resources.resx`）やカルチャ依存処理の保持を完全に禁止。
* **外部レイヤー依存の禁止 (C1 是正):** `GameSecurityTool.Contracts` に定義された DTO や Enum への依存を厳禁とする。Domain は `GameSecurityTool.Domain.Models.TrustEnhancement` サブ名前空間で完結する独自の型およびシグナル POCO を定義し、Contracts との変換は Application 層（`ExplanationEngine`）の責務とする。

---

# 2. Confidence Score Model (信頼度・確信度スコアモデル)

セキュリティ判定を「Safe / Threat」の単純な 2 値（Binary）にしない。

```text
[従来]
  Safe / Threat (二値)

[Trust Enhancement Model]
  Level:           High
  ConfidenceScore: 0.85 (85%)
  ReasonSignals:   [ UnsignedNewBinary, ContainsProcessSpawningApi ]
```

## 2.1 型定義 (`GameSecurityTool.Domain.Models.TrustEnhancement`)

Domain 層の純粋性と型分離を担保するため、`GameSecurityTool.Domain.Models.TrustEnhancement` 名前空間に独自の `record` および列挙型を定義する（C1 是正）。

```csharp
#nullable enable

namespace GameSecurityTool.Domain.Models.TrustEnhancement;

using System;
using System.Collections.Generic;

/// <summary>
/// Trust Enhancement 固有のリスク深刻度定義 (外部依存なし)
/// </summary>
public enum DomainTrustRiskLevel
{
    Safe = 0,
    Low = 1,
    Medium = 2,
    High = 3,
    Critical = 4
}

/// <summary>
/// 検知エンジンが発火した個別の客観的技術シグナル (自然言語文言を含まない)
/// </summary>
public sealed record DetectionSignal(
    string SignalCode,
    string Category,
    double Weight,
    string RawTechnicalDetail
);

/// <summary>
/// Domain 層における確信度付きセキュリティ評価結果 (C1 是正: 型名とサブ名前空間を明確化)
/// (自然言語を含まない純粋な評価データ・POCO)
/// </summary>
public sealed record ConfidenceAssessmentResult(
    string TargetPath,
    string FileHash,
    DomainTrustRiskLevel Level,
    double ConfidenceScore, // 0.0 - 1.0 (確信度)
    IReadOnlyList<DetectionSignal> Signals,
    DateTimeOffset EvaluatedAtUtc
);
```

## 2.2 Confidence の利用規約
1. **用途:** UI 上での視覚的説明、リスク順ソート、ユーザーの意思決定支援に限定。
2. **自動処罰の禁止:** Confidence スコアの高低のみをトリガーとした**ファイルの自動削除・自動隔離・強制通信許可を厳格に禁止**する。フェイルセーフの原則に従い、リスクが存在する場合は必ず User Consent (ユーザー同意) を要求すること。

---

# 3. 単体テスト仕様 (`GST.UnitTests.Trust.Domain`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.Trust.Domain;

using System;
using System.Collections.Generic;
using GameSecurityTool.Domain.Models.TrustEnhancement;
using Xunit;

public class ConfidenceModelTests
{
    [Fact]
    public void ConfidenceAssessmentResult_InitializesCleanlyWithoutExternalDependencies()
    {
        var signals = new List<DetectionSignal>
        {
            new("UnsignedNewBinary", "Binary", 30.0, "No valid signature"),
            new("ContainsSuspiciousNetworkApi", "Network", 50.0, "ws2_32.dll import")
        };

        var result = new ConfidenceAssessmentResult(
            TargetPath: "C:\\Games\\Cyberpunk2077\\bin\\x64\\mod.dll",
            FileHash: "A82F91C0DE4B12",
            Level: DomainTrustRiskLevel.High,
            ConfidenceScore: 0.85,
            Signals: signals,
            EvaluatedAtUtc: DateTimeOffset.UtcNow
        );

        Assert.Equal(DomainTrustRiskLevel.High, result.Level);
        Assert.Equal(0.85, result.ConfidenceScore);
        Assert.Equal(2, result.Signals.Count);
        Assert.Equal("UnsignedNewBinary", result.Signals[0].SignalCode);
    }
}
```

---

End of Document
```

---

