# 04-05: Timeline View & Contracts Specification

**Document ID:** GST-FEAT-TRUST-005  
**Parent Document:** 04-00_Overview_and_Principles.md  
**Category:** Contracts & Presentation Specification  
**Status:** Approved Feature Specification  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. タイムライン表示 (Timeline View)

セキュリティイベント（検知、隔離、復元、URLサニタイズ、設定変更、ポリシー上書き）を時系列で俯瞰できる UI 画面を提供する。

### 記録・表示対象イベント:
* `Detection`: 不審バイナリ・LNK乗っ取りの検知
* `Quarantine`: 暗号化隔離庫への退避
* `Restore`: ユーザー同意による隔離からの安全復元
* `WebLink`: URLクエリ除去、ブラウザ転送、ブロック
* `PolicyChange`: Global / GameProfile 設定の変更
* `SystemSafety`: クラッシュレポート遮断の適用・解除

---

# 2. パフォーマンス要件 (Pagination & Lazy Loading)

データベースの全件ロード（`SELECT * FROM AuditRecords`）を厳禁とし、カーソルベースまたはページネーションによる遅延読み込みを必須とする。

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Common;
using GameSecurityTool.Contracts.Dtos;

public interface ITimelineQueryService
{
    Task<PagedResultDto<TimelineItemDto>> GetTimelineAsync(
        TimelineFilterDto filter,
        PaginationParams pagination,
        CancellationToken ct = default
    );
}
```

---

# 3. Contracts DTO 定義 (`GameSecurityTool.Contracts`)

```csharp
namespace GameSecurityTool.Contracts.Dtos;

using System;
using System.Collections.Generic;

public sealed record RiskExplanationDto(
    string TargetPath,
    string DisplayTitle,
    string SummaryMessage,
    string RiskLevelText,
    double ConfidencePercentage,
    IReadOnlyList<string> TranslatedReasons,
    IReadOnlyList<string> RecommendedActions,
    string OperationId
);

public sealed record TimelineFilterDto(
    DateTime? StartDateUtc,
    DateTime? EndDateUtc,
    string? GameProfileId,
    string? EventCategory,
    string? SearchKeyword
);

public sealed record TimelineItemDto(
    long EventId,
    string OperationId,
    DateTime TimestampUtc,
    string EventType,
    string Severity,
    string Title,
    string Description,
    string? AssociatedPath,
    bool HasSnapshot
);

public sealed record EvidenceExportResultDto(
    bool Success,
    string ExportedZipPath,
    long FileSizeBytes,
    string Sha256Hash,
    string? ErrorMessage
);
```
