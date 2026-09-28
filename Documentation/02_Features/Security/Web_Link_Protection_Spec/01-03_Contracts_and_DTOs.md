# 01-03: Contracts and DTOs

**Document ID:** GST-SPEC-WEBLINK-001-PART3  
**Version:** 3.0 (SilentReject Policy & Pure Contracts Aligned)
**Parent Document:** Web Link Protection Specification  
**Category:** Contracts & Boundary Definitions  
**Status:** Approved Baseline Candidate  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Contracts Layer の責務

Contracts Layer は、Web Link Protection モジュールと他のレイヤー（Presentation, Application, Infrastructure）間の結合度を最小化するための**不変 DTO（Data Transfer Objects）および Port（Interface）**を定義する。

### 規約:
* すべての DTO は `sealed record` で宣言し、完全なイミュータビリティを担保する。
* DTO 内にビジネスロジック、DB 操作、UI 依存コードを持たない。
* Contracts 層は Domain Entity を参照・露出させず、レイヤー境界を越えるデータは DTO で統一する。

---

# 2. 境界 DTO (Boundary Data Transfer Objects)

## 2.1 URL 起動 & プレビュー関連 DTO

```csharp
namespace GameSecurityTool.Contracts.Features.WebLinkProtection.DTOs;

using System;
using System.Collections.Generic;
using GameSecurityTool.Domain.Features.WebLinkProtection.Enums;
using GameSecurityTool.Domain.Features.WebLinkProtection.ValueObjects;

/// <summary>
/// URL 起動要求 DTO
/// </summary>
public sealed record UrlLaunchRequestDto(
    string RawUrl,
    Guid? GameProfileId,
    WebLinkPolicySettings PolicySettings,
    BrowserOptionDto DefaultBrowser,
    IReadOnlyList<string> AllowedQueryParams);

/// <summary>
/// URL 起動結果 DTO (SilentReject 対応)
/// </summary>
public sealed record UrlLaunchResultDto(
    string OperationId,
    PolicyDecision Decision,
    bool IsSuccess,
    string Message);

/// <summary>
/// UI 表示用サニタイズプレビュー DTO (個人情報・機密Query完全排除)
/// </summary>
public sealed record UrlPreviewDto(
    string OperationId,
    string DisplayHost,
    string MaskedPath,
    HostCategory Category,
    bool IsPunycodeSuspicious);

/// <summary>
/// URL サニタイズ結果 DTO
/// </summary>
public sealed record SanitizedUrlResultDto(
    bool IsValid,
    Uri SanitizedUri,
    string Host,
    string MaskedPath,
    int RedactedQueryCount,
    HostCategory Category,
    bool IsPunycodeSuspicious);
```

## 2.2 ホストルール管理関連 DTO

```csharp
/// <summary>
/// ホストルール表示用 DTO
/// </summary>
public sealed record SocialHostRuleDto(
    Guid Id,
    Guid? GameProfileId,
    string Host,
    string DisplayName,
    HostCategory Category,
    RuleSource Source,
    MatchMode MatchMode,
    PolicyDecision Policy,
    bool IsEnabled,
    DateTimeOffset CreatedAt,
    DateTimeOffset ModifiedAt,
    DateTimeOffset? ExpiresAt,
    string? Reason);

/// <summary>
/// ホストルール新規作成用 DTO
/// </summary>
public sealed record CreateSocialHostRuleDto(
    Guid? GameProfileId,
    string Host,
    string DisplayName,
    HostCategory Category,
    MatchMode MatchMode,
    PolicyDecision Policy,
    DateTimeOffset? ExpiresAt,
    string? Reason);

/// <summary>
/// ホストルール更新用 DTO
/// </summary>
public sealed record UpdateSocialHostRuleDto(
    Guid Id,
    MatchMode MatchMode,
    PolicyDecision Policy,
    bool IsEnabled,
    DateTimeOffset? ExpiresAt,
    string? Reason);
```

## 2.3 ブラウザ選択関連 DTO

```csharp
public enum BrowserType
{
    DefaultSystemBrowser,
    GoogleChrome,
    MicrosoftEdge,
    MozillaFirefox,
    Brave,
    Vivaldi,
    CustomBrowser
}

/// <summary>
/// ブラウザオプション DTO
/// </summary>
public sealed record BrowserOptionDto(
    BrowserType BrowserType,
    string DisplayName,
    string ExecutablePath,
    bool IsPrivateBrowsingEnabled,
    string? CustomArguments);

/// <summary>
/// Browser Picker UI 選択結果 DTO
/// </summary>
public sealed record BrowserPickerPromptResultDto(
    bool IsConfirmed,
    BrowserOptionDto SelectedBrowser,
    bool RememberChoiceForProfile);
```

---

# 3. Port (Interface) 定義

## 3.1 Application / Use Case Ports

```csharp
namespace GameSecurityTool.Contracts.Features.WebLinkProtection.Ports;

using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Features.WebLinkProtection.DTOs;
using GameSecurityTool.Domain.Features.WebLinkProtection.Enums;
using GameSecurityTool.Domain.Features.WebLinkProtection.ValueObjects;

/// <summary>
/// Managed URL 起動ユースケース Port
/// </summary>
public interface ILaunchManagedUrlUseCase
{
    Task<UrlLaunchResultDto> ExecuteAsync(UrlLaunchRequestDto request, CancellationToken cancellationToken);
}

/// <summary>
/// URL サニタイザー Port
/// </summary>
public interface IUrlSanitizer
{
    SanitizedUrlResultDto Sanitize(string rawUrl, IReadOnlyList<string> allowedQueryParams);
    SanitizedUrlResultDto SanitizeForPreview(string rawUrl);
}

/// <summary>
/// ポリシー評価エンジン Port
/// </summary>
public interface IWebLinkPolicyEvaluator
{
    PolicyEvaluationResultDto Evaluate(
        Uri targetUri,
        WebLinkPolicySettings settings,
        IReadOnlyList<SocialHostRuleDto> activeRules);
}

public sealed record PolicyEvaluationResultDto(
    PolicyDecision Decision,
    HostCategory MatchedCategory,
    Guid? MatchedRuleId,
    string EvaluationReason);
```

## 3.2 Infrastructure Ports (Adapters)

```csharp
/// <summary>
/// ブラウザ選択・プロセス起動 Port (Infrastructure)
/// </summary>
public interface IBrowserPicker
{
    Task<BrowserPickerPromptResultDto> PromptUserChoiceAsync(UrlPreviewDto preview, CancellationToken cancellationToken);
    Task<bool> LaunchBrowserAsync(BrowserOptionDto browser, Uri targetUri, CancellationToken cancellationToken);
    IReadOnlyList<BrowserOptionDto> GetInstalledBrowsers();
}

/// <summary>
/// ユーザー定義ホストルール永続化 Port (Infrastructure Repository)
/// ※ Contracts DTO のみを受け渡し、Domain Entity を漏洩させない
/// </summary>
public interface ISocialHostRuleRepository
{
    Task<IReadOnlyList<SocialHostRuleDto>> GetActiveRulesAsync(Guid? gameProfileId, CancellationToken cancellationToken);
    Task<SocialHostRuleDto?> GetByIdAsync(Guid ruleId, CancellationToken cancellationToken);
    Task AddAsync(SocialHostRuleDto ruleDto, CancellationToken cancellationToken);
    Task UpdateAsync(UpdateSocialHostRuleDto ruleDto, CancellationToken cancellationToken);
    Task DeleteAsync(Guid ruleId, CancellationToken cancellationToken);
}

/// <summary>
/// Web Link 専用プライバシー保護監査ロガー Port
/// </summary>
public interface IWebLinkAuditLogger
{
    Task LogLaunchedAsync(string operationId, Guid? gameProfileId, BrowserType browserType, bool success, CancellationToken cancellationToken);
    Task LogBlockedAsync(string operationId, Guid? gameProfileId, PolicyEvaluationResultDto evaluation, CancellationToken cancellationToken);
    Task LogUserRejectedAsync(string operationId, Guid? gameProfileId, CancellationToken cancellationToken);
    Task LogInvalidUrlRejectedAsync(string operationId, Guid? gameProfileId, CancellationToken cancellationToken);
    Task LogEmergencyDetectedAsync(string operationId, int pid, string processName, string actionTaken, CancellationToken cancellationToken);
    Task LogRuleModifiedAsync(Guid ruleId, string action, CancellationToken cancellationToken);
}
```

---

End of Document
```

---
