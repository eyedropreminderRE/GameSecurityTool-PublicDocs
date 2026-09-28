# 01-02: Application Use Cases and Pipeline

**Document ID:** GST-SPEC-WEBLINK-001-PART2  
**Version:** 3.1 (WMI Port Abstraction & Symmetric Host Normalization Hardened)
**Parent Document:** Web Link Protection Specification v2.1  
**Category:** Application Workflow & Pipeline  
**Status:** Approved Baseline Candidate  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Application Layer の責務

Application Layer は、Web Link Protection における**URL 処理パイプラインのオーケストレーション、ユースケース制御、Explanation 生成、トランザクション調停**を担当する。

### 主な責務:
* URL の受信からブラウザ起動までのシーケンス制御
* `IUrlSanitizer`、`IWebLinkPolicyEvaluator`、`IBrowserPicker` 等の Port 連携
* `OperationId` の発行と監査ログ（Audit Chain）への記録
* Explanation Engine を用いたユーザー向け説明 DTO（`UrlPreviewDto` 等）の生成
* WMI Emergency Detection からの異常検知イベントの調停（`IProcessLifecycleWatcher` 経由）
* ブラウザ起動失敗時の安全な再選択・リカバリ制御

---

# 2. URL Processing Pipeline (URL 処理パイプライン)

URL が渡されてからブラウザが起動されるまで、以下の 7 ステップを厳格な順序で実行する。

```text
[ Raw URL Input ]
        │
        ▼
1. Parse & Scheme Validation (http / https のみ許可、UserInfo 除去)
        │
        ▼
2. Canonicalization (IDN / Punycode 正規化、Path 正規化)
        │
        ▼
3. Strict Query Stripping (Query String 破棄 / Allowlist 抽出)
        │
        ▼
4. Domain Policy Evaluation (Host / Category / Profile 評価)
        │
        ▼
5. User Consent & Browser Selection (Confirm 時の UI 表示)
        │
        ▼
6. Audit Logging (OperationId 付与、サニタイズ済みログ記録)
        │
        ▼
7. Safe Browser Launch (許可されたブラウザのプロセス起動)
```

---

# 3. Strict Query Stripping の詳細仕様

## 3.1 デフォルト破棄ポリシー
個人識別情報（PII）、セッション ID、認証トークン、トラッキングパラメータの漏洩を構造的に遮断するため、**Query String（`?` 以降）はデフォルトで完全破棄**する。

```text
入力:
https://example.com/login?session_token=secret123&utm_source=game&user_id=9876

Strict Query Stripping 適用後:
https://example.com/login
```

### 3.1.1 Query破棄のユーザー説明と回復導線 (documented security boundary)

Strict Query Strippingの既定値は維持する。Queryが破棄された場合、ユーザーにはRaw Query値を表示せず、破棄が発生した事実と理由を安全なメタデータとして説明する。元のリンクが機能しない場合は、Trusted hostに対して既存のQuery AllowlistへReview → Confirmで進む導線を提供する。

- Query値そのものをUIへ再表示しない。
- 未知Queryを推測して自動保持しない。
- 認証・セッションQueryの自動保持を行わない。
- Allowlistは明示登録されたキーのみを対象とする既存契約を維持する。

## 3.2 Query Allowlist (明示的例外維持)
特定の信頼済みホストにおいて、動作上不可欠なパラメータが存在する場合に限り、明示的に登録されたキーのみを維持する。

```csharp
// Allowlist: ["lang", "page"]
// 入力: https://example.com/view?lang=ja&session=secret&page=2
// 出力: https://example.com/view?lang=ja&page=2
```
* **規約:** 未知のパラメータは自動許可せず破棄する。ブラックリスト方式（推測による個別除去）は採用しない。

---

# 4. Use Cases 実装仕様

## 4.1 LaunchManagedUrlUseCase (URL 起動ユースケース)
ゲームまたは UI からの URL 起動要求を処理するメインユースケース。

```csharp
#nullable enable

namespace GameSecurityTool.Application.Features.WebLinkProtection.UseCases;

using System;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Features.WebLinkProtection.DTOs;
using GameSecurityTool.Contracts.Features.WebLinkProtection.Ports;
using GameSecurityTool.Domain.Features.WebLinkProtection.Enums;
using Microsoft.Extensions.Logging;

public sealed class LaunchManagedUrlUseCase(
    IUrlSanitizer urlSanitizer,
    IWebLinkPolicyEvaluator policyEvaluator,
    IBrowserPicker browserPicker,
    ISocialHostRuleRepository ruleRepository,
    IWebLinkAuditLogger auditLogger,
    ILogger<LaunchManagedUrlUseCase> logger) : ILaunchManagedUrlUseCase
{
    public async Task<UrlLaunchResultDto> ExecuteAsync(
        UrlLaunchRequestDto request,
        CancellationToken cancellationToken)
    {
        var operationId = $"GST-URL-{DateTimeOffset.UtcNow:yyyyMMdd}-{Guid.NewGuid():N[..8]}";

        // 1. Parse & Canonicalize & Strict Query Strip
        var sanitizeResult = urlSanitizer.Sanitize(request.RawUrl, request.AllowedQueryParams);
        if (!sanitizeResult.IsValid)
        {
            await auditLogger.LogInvalidUrlRejectedAsync(operationId, request.GameProfileId, cancellationToken);
            return new UrlLaunchResultDto(operationId, PolicyDecision.Block, false, "無効なURL形式です。");
        }

        // 2. Load Active Rules & Evaluate Domain Policy
        var rules = await ruleRepository.GetActiveRulesAsync(request.GameProfileId, cancellationToken);
        var evaluation = policyEvaluator.Evaluate(sanitizeResult.SanitizedUri, request.PolicySettings, rules);

        // 3. Handle Decision
        switch (evaluation.Decision)
        {
            case PolicyDecision.Block:
                await auditLogger.LogBlockedAsync(operationId, request.GameProfileId, evaluation, cancellationToken);
                return new UrlLaunchResultDto(operationId, PolicyDecision.Block, false, "リンクの起動はブロックされました。");

            case PolicyDecision.Confirm:
                return await HandleConfirmFlowAsync(operationId, sanitizeResult, evaluation, request.GameProfileId, cancellationToken);

            case PolicyDecision.Allow:
            default:
                // 登録済みブラウザでの直接起動を試行
                var launched = await LaunchBrowserSafelyAsync(operationId, request.DefaultBrowser, sanitizeResult.SanitizedUri, request.GameProfileId, cancellationToken);
                if (!launched)
                {
                    // 起動失敗時: 勝手に既定ブラウザへフォールバックせず、Browser Picker を起動して再選択を仰ぐ
                    logger.LogWarning("登録ブラウザの起動に失敗したため、Browser Picker へフォールバックします。");
                    return await HandleConfirmFlowAsync(operationId, sanitizeResult, evaluation, request.GameProfileId, cancellationToken);
                }
                return new UrlLaunchResultDto(operationId, PolicyDecision.Allow, true, "起動成功");
        }
    }

    private async Task<UrlLaunchResultDto> HandleConfirmFlowAsync(
        string operationId,
        SanitizedUrlResultDto sanitizeResult,
        PolicyEvaluationResultDto evaluation,
        Guid? gameProfileId,
        CancellationToken cancellationToken)
    {
        var preview = new UrlPreviewDto(
            operationId,
            sanitizeResult.Host,
            sanitizeResult.MaskedPath,
            evaluation.MatchedCategory,
            sanitizeResult.IsPunycodeSuspicious);

        var pickerResult = await browserPicker.PromptUserChoiceAsync(preview, cancellationToken);
        if (!pickerResult.IsConfirmed)
        {
            await auditLogger.LogUserRejectedAsync(operationId, gameProfileId, cancellationToken);
            return new UrlLaunchResultDto(operationId, PolicyDecision.Confirm, false, "ユーザーによりキャンセルされました。");
        }

        var success = await LaunchBrowserSafelyAsync(operationId, pickerResult.SelectedBrowser, sanitizeResult.SanitizedUri, gameProfileId, cancellationToken);
        return new UrlLaunchResultDto(
            operationId,
            PolicyDecision.Allow,
            success,
            success ? "起動成功" : "ブラウザの起動に失敗しました。インストール状態を確認してください。");
    }

    private async Task<bool> LaunchBrowserSafelyAsync(
        string operationId,
        BrowserOptionDto browser,
        Uri targetUri,
        Guid? gameProfileId,
        CancellationToken cancellationToken)
    {
        var success = await browserPicker.LaunchBrowserAsync(browser, targetUri, cancellationToken);
        await auditLogger.LogLaunchedAsync(operationId, gameProfileId, browser.BrowserType, success, cancellationToken);
        return success;
    }
}
```

## 4.2 PreviewUrlUseCase (表示用サニタイズプレビュー)
ダイアログ表示用に、機密情報を除去した安全なプレビュー DTO を生成する。

```csharp
#nullable enable

namespace GameSecurityTool.Application.Features.WebLinkProtection.UseCases;

using GameSecurityTool.Contracts.Features.WebLinkProtection.DTOs;
using GameSecurityTool.Contracts.Features.WebLinkProtection.Ports;

public sealed class PreviewUrlUseCase(IUrlSanitizer urlSanitizer)
{
    public UrlPreviewDto Execute(string rawUrl)
    {
        var result = urlSanitizer.SanitizeForPreview(rawUrl);
        return new UrlPreviewDto(
            OperationId: string.Empty,
            DisplayHost: result.Host,
            MaskedPath: result.MaskedPath,
            Category: result.Category,
            IsPunycodeSuspicious: result.IsPunycodeSuspicious);
    }
}
```

## 4.3 ManageSocialRulesUseCase (ユーザー定義ルール管理 - H2 是正)
ユーザー定義ホストルールの追加・編集・削除・有効化切替を行う。登録時にも評価時と完全に対称な IDN/Punycode 正規化を適用し、偽装ドメインの登録・評価のすれ違いを物理的に防ぐ。

```csharp
#nullable enable

namespace GameSecurityTool.Application.Features.WebLinkProtection.UseCases;

using System;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Features.WebLinkProtection.DTOs;
using GameSecurityTool.Contracts.Features.WebLinkProtection.Ports;
using GameSecurityTool.Domain.Features.WebLinkProtection.Enums;

public sealed class ManageSocialRulesUseCase(
    ISocialHostRuleRepository ruleRepository,
    IUrlSanitizer urlSanitizer, // H2 是正: 登録時も同一の正規化パイプラインを通す
    IWebLinkAuditLogger auditLogger)
{
    public async Task AddRuleAsync(CreateSocialHostRuleDto dto, CancellationToken cancellationToken)
    {
        // 入力された Host 文字列を、ダミーの URI に組み込んで IUrlSanitizer に正規化 (IDN変換、小文字化、末尾ドット除去等) させる
        var dummyUrl = $"https://{dto.Host}/";
        var sanitizeResult = urlSanitizer.SanitizeForPreview(dummyUrl);

        if (!sanitizeResult.IsValid || string.IsNullOrWhiteSpace(sanitizeResult.Host))
        {
            throw new ArgumentException("無効なホスト名、または解析不可能な文字列です。");
        }

        string normalizedHost = sanitizeResult.Host;

        var ruleId = Guid.NewGuid();
        var ruleDto = new SocialHostRuleDto(
            ruleId,
            dto.GameProfileId,
            normalizedHost, // 正規化済み (ASCII/Punycode) ホストを保存
            dto.DisplayName,
            dto.Category,
            RuleSource.UserDefined,
            dto.MatchMode,
            dto.Policy,
            true,
            DateTimeOffset.UtcNow,
            DateTimeOffset.UtcNow,
            dto.ExpiresAt,
            dto.Reason);

        await ruleRepository.AddAsync(ruleDto, cancellationToken);
        await auditLogger.LogRuleModifiedAsync(ruleId, "RuleAdded", cancellationToken);
    }
}
```

---

# 5. WMI Emergency Detection ワークフロー (H1 是正)

Managed Launch Path を迂回して起動されたブラウザプロセスを検知した際のオーケストレーション。
Application 層は WMI や OS API を直接扱わず、Contracts 層の `IProcessLifecycleWatcher` Port からの抽象化されたイベント駆動で動作し、Clean Architecture を維持する。

```text
[ IProcessLifecycleWatcher.ProcessStarted イベント受信 (Contracts Port) ]
                      │
                      ▼
[ WmiEmergencyOrchestrator (Application) ]
                      │
                      ├─> 1. Process Identity Revalidation (SafeProcessHandle + StartTime + FileId)
                      ├─> 2. Game Process & Launch Context 照合
                      │
                      ▼ (迂回起動と判定)
[ Action: User Prompt / Emergency Terminate ]
                      │
                      ▼
[ Audit Hash Chain Record (WebLinkEmergencyDetected) ]
```

---

# 6. パフォーマンスと軽量性 (Performance Baseline)

* **同期・非同期の適切な分離:** URI Parse / Canonicalize / Query Strip / Host Matching はすべてインメモリ計算（In-Memory CPU 処理）とし、同期でマイクロ秒単位で完了させる。
* **重スキャンの排除:** URL 判定処理の都度、ディスク全体のフルスキャンや重いファイル I/O を実行することを厳禁とする。
```

---
