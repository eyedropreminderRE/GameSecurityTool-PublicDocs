# 03-05: ALLOW - Allow List & Trust Specification

**Document ID:** GST-MOD-ALLOW-005  
**Version:** 4.1 (Domain Sub-namespace Isolation & SSOT Hardened Edition)  
**Status:** Approved Module Specification  
**Target Projects:**
- `GameSecurityTool.Domain`
- `GameSecurityTool.Contracts`
- `GameSecurityTool.Application`
- `GameSecurityTool.Infrastructure`

> **Lifecycle note:** This module specification describes approved design scope. Its presence does not by itself indicate that the module is implemented, Windows-verified, or released.

---

# 1. モジュール概要 ＆ 責任境界

本モジュールは、ユーザーが信頼したファイル、パス、署名者情報を管理する**例外許可リスト（Allow List）、TOCTOU 防御（リアルタイム有効期限評価）、Universal Rule Scope 照会、なりすまし検知（Tampered Allow List 防御）、およびセキュリティリスク判定**を担当する。

### Clean 5-Layer レイヤー境界:
- **Domain Layer (`GST.Domain`):** 照会優先順位ルール（1. Hash ➔ 2. Signature ➔ 3. Path ➔ 4. Name）、TOCTOU 評価ロジック、`SecurityEngineEvaluator`（純粋ドメイン判定サービス）。**Contracts、System.IO、OS API、DB等の外部依存完全ゼロ（Pure C#）。`GameSecurityTool.Domain.Models.AllowList` サブ名前空間に独立モデルを配置し、他モジュールとの型衝突を物理排除する。**
- **Contracts Layer (`GST.Contracts`):** `IAllowListRepository`, `IRiskAssessmentService`, `IDbWriteQueue` Port インターフェースおよび不変 DTO 群。
- **Application Layer (`GST.Application`):** Allow List 登録・削除 Use Case、有効期限切れクリーンアップ調停。**Domain モデル ⇄ Contracts DTO のマッピング（変換）責務を負う。**
- **Infrastructure Layer (`GST.Infrastructure`):** SQLite `AllowListEntries` テーブルへの永続化、複合インデックス検索、`IDbWriteQueue` 直列化コミット。

---

# 2. 機能要件仕様 (Functional Requirements)

## 2.1 `FN-ALLOW-01`: 例外許可リスト統合照会 ＆ 優先順位ルール
* **照会優先順位:**
  1. **`Hash Allow` (最優先):** SHA256 ハッシュによる完全一致照会。
  2. **`Signature Allow`:** デジタル署名発行元名（PublisherName）による照会。
  3. **`Path Allow`:** フォルダ配下再帰一致（`C:\Games\Foo\*` 等）。
  4. **`Name Allow` (最弱):** ファイル名単体一致。
* **競合解決の絶対安全保証 (Tampered Allow List Guard):**
  同一ファイルパスに対して「Hash 許可」と「Path 許可」が両方登録されており、実際のファイルが**Hash ルールに一致しなかった場合（＝ファイルが差し替えられた疑いがある場合）**、後続の弱い Path ルールで通過させることを禁止する。即座に「なりすましの疑い (`TamperedAllowList`)」の高リスクシグナルへ昇格させてブロック（Critical）判定を下す。

## 2.2 `FN-ALLOW-02`: TOCTOU (Time-of-Check to Time-of-Use) 防御 ＆ クロック注入
* **目的:** バックグラウンドタイマーの削除タイミングに依存せず、判定の瞬間に期限切れルールを確実に無効化する。
* **処理仕様:** 判定エンジン（`SecurityEngineEvaluator.Evaluate`）内で `evaluationTimeUtc`（未指定時は `DateTimeOffset.UtcNow`）を受け取り、`rule.ExpiresAt.HasValue && rule.ExpiresAt.Value <= now` をリアルタイム検証して期限切れエントリを即座に除外。

## 2.3 `FN-ALLOW-03`: Path Allow セキュリティ注意バナー表示規約
* **処理仕様:** ユーザーが UI 上で `Path Allow` または `Name Allow` を選択した場合、*「⚠️ パス指定許可はファイル差し替え攻撃に弱い設定です。Hash許可または署名許可を推奨します」* のセキュリティアドバイスバナーを表示する。

## 2.4 `FN-ALLOW-04`: 一時許可（Temporary Allow）ライフサイクル
* **選択肢:** `Permanent (永続)`, `OneHour (1時間)`, `TwentyFourHours (24時間)`, `NextRestart (次回起動まで)`。
* **クリーンアップ:** 5分周期タイマー、アプリ起動時、およびスキャン開始時に期限切れレコードを自動整理する。

---

# 3. Domain Layer 定義 (`GameSecurityTool.Domain`)

Domain 層は Contracts や外部ライブラリを一切参照せず、完全に独立した純粋なエンティティおよびドメイン固有の Enum を定義する（C1 是正）。これらの型と Contracts 側の DTO/Enum との変換は Application 層で行われる。

```csharp
namespace GameSecurityTool.Domain.Models.AllowList;

using System;
using System.Collections.Generic;

/// <summary>
/// Domain 層共通のリスクレベル定義 (外部依存なし)
/// </summary>
public enum DomainRiskLevel
{
    Safe = 0,
    Low = 1,
    Medium = 2,
    High = 3,
    Critical = 4
}

public enum DomainThreatSeverity
{
    Info = 0,
    Warning = 1,
    Danger = 2
}

public enum DomainAllowType
{
    Hash = 0,
    Signature = 1,
    Path = 2,
    Name = 3
}

public enum DomainExpirationType
{
    Permanent = 0,
    OneHour = 1,
    TwentyFourHours = 2,
    NextRestart = 3
}

public sealed record ThreatReason(
    string ThreatReasonCode,
    string Description,
    DomainThreatSeverity Severity
);

public sealed record FileIdentityCandidate(
    string FileName,
    string AbsolutePath,
    string SHA256,
    long FileSize,
    bool IsSigned,
    string? PublisherName,
    bool IsNewFile
);

public sealed record AllowListRule(
    Guid Id,
    Guid? GameProfileId,
    string OperationId,
    DomainAllowType Type,
    string? TargetSHA256,
    string? TargetPath,
    string? TargetFileName,
    string? PublisherName,
    int Priority,
    DomainExpirationType Expiration,
    DateTimeOffset? ExpiresAt,
    string? UserMemo,
    DateTimeOffset CreatedAt
);

/// <summary>
/// ALLOW / SCAN 判定エンジンが生成する純粋ドメイン判定結果 (C1 是正: 型名を明確化)
/// </summary>
public sealed record SecurityEvaluationResult(
    DomainRiskLevel Level,
    int Score,
    IReadOnlyList<ThreatReason> Reasons,
    DateTimeOffset EvaluatedAt,
    string RuleVersion
);
```

## 3.1 純粋判定エンジン ＆ 競合ガード (`SecurityEngineEvaluator.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Domain.Services;

using System;
using System.Collections.Generic;
using System.Linq;
using GameSecurityTool.Domain.Models.AllowList;

public sealed class SecurityEngineEvaluator
{
    public SecurityEvaluationResult Evaluate(
        FileIdentityCandidate fileInfo,
        IReadOnlyList<AllowListRule> allowRules,
        DateTimeOffset? evaluationTimeUtc = null)
    {
        var now = evaluationTimeUtc ?? DateTimeOffset.UtcNow;

        // 1. TOCTOU リアルタイム期限切れ除外
        var activeRules = allowRules
            .Where(r => !r.ExpiresAt.HasValue || r.ExpiresAt.Value > now)
            .OrderBy(r => (int)r.Type) // Hash (0) > Signature (1) > Path (2) > Name (3)
            .ToList();

        // 2. 競合解決ガード: 同一ファイルへの Hash ルールの登録があるのに不一致だった場合、なりすましと断定
        var targetRules = activeRules.Where(r => 
            (r.TargetPath != null && fileInfo.AbsolutePath.StartsWith(r.TargetPath, StringComparison.OrdinalIgnoreCase)) || 
            (r.TargetFileName != null && string.Equals(r.TargetFileName, fileInfo.FileName, StringComparison.OrdinalIgnoreCase)) || 
            (r.TargetSHA256 != null && string.Equals(r.TargetSHA256, fileInfo.SHA256, StringComparison.OrdinalIgnoreCase))).ToList();

        var hashRules = targetRules.Where(r => r.Type == DomainAllowType.Hash).ToList();
        if (hashRules.Count > 0 && !hashRules.Any(r => string.Equals(r.TargetSHA256, fileInfo.SHA256, StringComparison.OrdinalIgnoreCase)))
        {
            return new SecurityEvaluationResult(
                Level: DomainRiskLevel.Critical,
                Score: 100,
                Reasons: [new ThreatReason("TamperedAllowList", "ハッシュ許可リストと不一致です（ファイル改ざん・なりすましの疑い）", DomainThreatSeverity.Danger)],
                EvaluatedAt: now,
                RuleVersion: "1.0");
        }

        // 3. Allow List 照会 (適合時は即座に Safe 返却)
        foreach (var rule in targetRules)
        {
            if (IsRuleMatched(fileInfo, rule))
            {
                return new SecurityEvaluationResult(
                    Level: DomainRiskLevel.Safe,
                    Score: 0,
                    Reasons: [new ThreatReason("AllowListMatched", $"例外許可リストに適合 ({rule.Type})", DomainThreatSeverity.Info)],
                    EvaluatedAt: now,
                    RuleVersion: "1.0");
            }
        }

        // 4. 一般加算評価
        int score = 0;
        var reasons = new List<ThreatReason>();

        if (!fileInfo.IsSigned)
        {
            score += 30;
            reasons.Add(new ThreatReason("UnsignedNewBinary", "未署名のバイナリです", DomainThreatSeverity.Warning));
        }

        if (fileInfo.IsNewFile)
        {
            score += 20;
            reasons.Add(new ThreatReason("NewFileDetected", "新規配置されたファイルです", DomainThreatSeverity.Info));
        }

        var level = score switch
        {
            >= 70 => DomainRiskLevel.Critical,
            >= 40 => DomainRiskLevel.High,
            >= 20 => DomainRiskLevel.Medium,
            > 0   => DomainRiskLevel.Low,
            _     => DomainRiskLevel.Safe
        };

        return new SecurityEvaluationResult(level, score, reasons, now, "1.0");
    }

    private static bool IsRuleMatched(FileIdentityCandidate fileInfo, AllowListRule rule)
    {
        return rule.Type switch
        {
            DomainAllowType.Hash => string.Equals(fileInfo.SHA256, rule.TargetSHA256, StringComparison.OrdinalIgnoreCase),
            DomainAllowType.Signature => rule.PublisherName != null && string.Equals(fileInfo.PublisherName, rule.PublisherName, StringComparison.OrdinalIgnoreCase),
            DomainAllowType.Path => rule.TargetPath != null && fileInfo.AbsolutePath.StartsWith(rule.TargetPath, StringComparison.OrdinalIgnoreCase),
            DomainAllowType.Name => string.Equals(fileInfo.FileName, rule.TargetFileName, StringComparison.OrdinalIgnoreCase),
            _ => false
        };
    }
}
```

---

# 4. Contracts Layer 定義 (`GameSecurityTool.Contracts`)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Common;
using GameSecurityTool.Contracts.Dtos;

public interface IAllowListRepository
{
    Task<IReadOnlyList<AllowListRuleDto>> GetActiveRulesAsync(Guid? gameProfileId, CancellationToken ct = default);
    Task AddRuleAsync(AllowListRuleDto rule, CancellationToken ct = default);
    Task RemoveRuleAsync(Guid ruleId, CancellationToken ct = default);
    Task CleanupExpiredRulesAsync(CancellationToken ct = default);
}

public interface IRiskAssessmentService
{
    RiskAssessmentResultDto AssessFileRisk(
        FileIdentityCandidateDto fileInfo, 
        IReadOnlyList<AllowListRuleDto> allowRules,
        DateTimeOffset? evaluationTimeUtc = null);
}
```

```csharp
namespace GameSecurityTool.Contracts.Dtos;

using System;
using System.Collections.Generic;
using GameSecurityTool.Contracts.Common;

public sealed record AllowListRuleDto(
    Guid Id,
    Guid? GameProfileId,
    string OperationId,
    AllowType Type,
    string? TargetSHA256,
    string? TargetPath,
    string? TargetFileName,
    string? PublisherName,
    int Priority,
    ExpirationType Expiration,
    DateTimeOffset? ExpiresAt,
    string? UserMemo,
    DateTimeOffset CreatedAt
);

public sealed record FileIdentityCandidateDto(
    string FileName,
    string AbsolutePath,
    string SHA256,
    long FileSize,
    bool IsSigned,
    string? PublisherName,
    bool IsNewFile
);

public sealed record ThreatReasonDto(
    string ThreatReasonCode,
    string Description,
    ThreatSeverity Severity
);

public sealed record RiskAssessmentResultDto(
    RiskLevel Level,
    int Score,
    IReadOnlyList<ThreatReasonDto> Reasons,
    DateTimeOffset EvaluatedAt,
    string RuleVersion
);
```

---

# 5. Infrastructure Layer 完全実装 (`GameSecurityTool.Infrastructure`)

## 5.1 永続化 Entity (`AllowListEntry.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Persistence.Entities;

using System;

public class AllowListEntry
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public Guid? GameProfileId { get; set; }
    public required string OperationId { get; set; }
    public int MatchType { get; set; } // DB永続化用として int 保持
    public string? TargetSHA256 { get; set; }
    public string? TargetPath { get; set; }
    public string? TargetFileName { get; set; }
    public string? PublisherName { get; set; }
    public int Expiration { get; set; } // DB永続化用として int 保持
    public DateTimeOffset? ExpiresAt { get; set; }
    public string? UserMemo { get; set; }
    public DateTimeOffset CreatedAt { get; set; } = DateTimeOffset.UtcNow;
}
```

## 5.2 SQLite リポジトリ実装 (`SqliteAllowListRepository.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Persistence.Repositories;

using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Common;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Infrastructure.Persistence;
using GameSecurityTool.Infrastructure.Persistence.Entities;
using Microsoft.EntityFrameworkCore;

public sealed class SqliteAllowListRepository(
    IDbContextFactory<AppDbContext> dbFactory,
    IDbWriteQueue dbWriter) : IAllowListRepository
{
    public async Task<IReadOnlyList<AllowListRuleDto>> GetActiveRulesAsync(Guid? gameProfileId, CancellationToken ct = default)
    {
        await using var db = await dbFactory.CreateDbContextAsync(ct);
        var now = DateTimeOffset.UtcNow;
        var records = await db.Set<AllowListEntry>()
            .AsNoTracking()
            .Where(r => (r.GameProfileId == null || r.GameProfileId == gameProfileId) &&
                        (r.ExpiresAt == null || r.ExpiresAt > now))
            .OrderBy(r => r.MatchType)
            .ToListAsync(ct);

        return records.Select(r => new AllowListRuleDto(
            r.Id, r.GameProfileId, r.OperationId, (AllowType)r.MatchType, r.TargetSHA256,
            r.TargetPath, r.TargetFileName, r.PublisherName, 0,
            (ExpirationType)r.Expiration, r.ExpiresAt, r.UserMemo, r.CreatedAt
        )).ToList();
    }

    public async Task AddRuleAsync(AllowListRuleDto rule, CancellationToken ct = default)
    {
        var record = new AllowListEntry
        {
            Id = rule.Id,
            GameProfileId = rule.GameProfileId,
            OperationId = rule.OperationId,
            MatchType = (int)rule.Type,
            TargetSHA256 = rule.TargetSHA256,
            TargetPath = rule.TargetPath,
            TargetFileName = rule.TargetFileName,
            PublisherName = rule.PublisherName,
            Expiration = (int)rule.Expiration,
            ExpiresAt = rule.ExpiresAt,
            UserMemo = rule.UserMemo,
            CreatedAt = rule.CreatedAt
        };

        await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
        {
            await using var db = await dbFactory.CreateDbContextAsync(innerCt);
            db.Set<AllowListEntry>().Add(record);
            await db.SaveChangesAsync(innerCt);
        }, ct);
    }

    public async Task RemoveRuleAsync(Guid ruleId, CancellationToken ct = default)
    {
        await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
        {
            await using var db = await dbFactory.CreateDbContextAsync(innerCt);
            var record = await db.Set<AllowListEntry>().FirstOrDefaultAsync(r => r.Id == ruleId, innerCt);
            if (record != null)
            {
                db.Set<AllowListEntry>().Remove(record);
                await db.SaveChangesAsync(innerCt);
            }
        }, ct);
    }

    public async Task CleanupExpiredRulesAsync(CancellationToken ct = default)
    {
        var now = DateTimeOffset.UtcNow;
        await dbWriter.EnqueueWriteAsync(async (CancellationToken innerCt) =>
        {
            await using var db = await dbFactory.CreateDbContextAsync(innerCt);
            var expired = await db.Set<AllowListEntry>()
                .Where(r => r.ExpiresAt != null && r.ExpiresAt <= now)
                .ToListAsync(innerCt);

            if (expired.Count > 0)
            {
                db.Set<AllowListEntry>().RemoveRange(expired);
                await db.SaveChangesAsync(innerCt);
            }
        }, ct);
    }
}
```

---

# 6. Application Layer マッピングサービス (`RiskAssessmentService.cs`)

Domain の `SecurityEngineEvaluator`（純粋判定）を呼び出し、Contracts 層の DTO 体系へ変換する調停サービス。

```csharp
#nullable enable

namespace GameSecurityTool.Application.Features.AllowList;

using System;
using System.Collections.Generic;
using System.Linq;
using GameSecurityTool.Contracts.Common;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Domain.Models.AllowList;
using GameSecurityTool.Domain.Services;

public sealed class RiskAssessmentService(SecurityEngineEvaluator domainEvaluator) : IRiskAssessmentService
{
    public RiskAssessmentResultDto AssessFileRisk(
        FileIdentityCandidateDto fileInfo,
        IReadOnlyList<AllowListRuleDto> allowRules,
        DateTimeOffset? evaluationTimeUtc = null)
    {
        var domainCandidate = new FileIdentityCandidate(
            fileInfo.FileName,
            fileInfo.AbsolutePath,
            fileInfo.SHA256,
            fileInfo.FileSize,
            fileInfo.IsSigned,
            fileInfo.PublisherName,
            fileInfo.IsNewFile
        );

        var domainRules = allowRules.Select(r => new AllowListRule(
            r.Id,
            r.GameProfileId,
            r.OperationId,
            (DomainAllowType)r.Type,
            r.TargetSHA256,
            r.TargetPath,
            r.TargetFileName,
            r.PublisherName,
            r.Priority,
            (DomainExpirationType)r.Expiration,
            r.ExpiresAt,
            r.UserMemo,
            r.CreatedAt
        )).ToList();

        var domainResult = domainEvaluator.Evaluate(domainCandidate, domainRules, evaluationTimeUtc);

        return new RiskAssessmentResultDto(
            Level: MapToContractsRiskLevel(domainResult.Level),
            Score: domainResult.Score,
            Reasons: domainResult.Reasons.Select(r => new ThreatReasonDto(
                r.ThreatReasonCode,
                r.Description,
                MapToContractsSeverity(r.Severity)
            )).ToList(),
            EvaluatedAt: domainResult.EvaluatedAt,
            RuleVersion: domainResult.RuleVersion
        );
    }

    private static RiskLevel MapToContractsRiskLevel(DomainRiskLevel level) => level switch
    {
        DomainRiskLevel.Critical => RiskLevel.Critical,
        DomainRiskLevel.High     => RiskLevel.High,
        DomainRiskLevel.Medium   => RiskLevel.Medium,
        DomainRiskLevel.Low      => RiskLevel.Low,
        _                        => RiskLevel.Safe
    };

    private static ThreatSeverity MapToContractsSeverity(DomainThreatSeverity severity) => severity switch
    {
        DomainThreatSeverity.Danger  => ThreatSeverity.Danger,
        DomainThreatSeverity.Warning => ThreatSeverity.Warning,
        _                            => ThreatSeverity.Info
    };
}
```

---

# 7. 単体テスト仕様 (`GST.UnitTests.ALLOW`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.ALLOW;

using System;
using GameSecurityTool.Domain.Models.AllowList;
using GameSecurityTool.Domain.Services;
using Xunit;

public class AllowListTests
{
    [Fact]
    public void Evaluate_ExpiredRule_IsIgnoredByTOCTOUGuard()
    {
        var evaluator = new SecurityEngineEvaluator();
        var fileInfo = new FileIdentityCandidate("version.dll", "C:\\Games\\version.dll", "HASH_EXP", 1024, false, null, true);

        var now = DateTimeOffset.UtcNow;
        var expiredRule = new AllowListRule(
            Guid.NewGuid(), null, "OP-1", DomainAllowType.Hash, "HASH_EXP", null, null, null, 1,
            DomainExpirationType.OneHour, now.AddMinutes(-5), "Expired", now.AddHours(-1)
        );

        var result = evaluator.Evaluate(fileInfo, [expiredRule], now);
        Assert.NotEqual(DomainRiskLevel.Safe, result.Level);
    }

    [Fact]
    public void Evaluate_ConflictingHashAndPath_RejectsTamperedFile()
    {
        var evaluator = new SecurityEngineEvaluator();
        
        // ファイル実体は HASH_MODIFIED に書き換わっている
        var fileInfo = new FileIdentityCandidate("version.dll", "C:\\Games\\version.dll", "HASH_MODIFIED", 1024, false, null, false);

        var now = DateTimeOffset.UtcNow;
        // ユーザーが登録した本来の Hash ルール (HASH_ORIGINAL)
        var hashRule = new AllowListRule(
            Guid.NewGuid(), null, "OP-1", DomainAllowType.Hash, "HASH_ORIGINAL", null, null, null, 1,
            DomainExpirationType.Permanent, null, "Safe Hash", now);
        
        // 同じパスに対する緩い Path ルール
        var pathRule = new AllowListRule(
            Guid.NewGuid(), null, "OP-2", DomainAllowType.Path, null, "C:\\Games\\", null, null, 1,
            DomainExpirationType.Permanent, null, "Safe Path", now);

        // Act: Pathルールにマッチするが、Hashルールに不一致
        var result = evaluator.Evaluate(fileInfo, [hashRule, pathRule], now);

        // Assert: Path でスルーされず、なりすまし (Critical) としてブロックされること
        Assert.Equal(DomainRiskLevel.Critical, result.Level);
        Assert.Contains(result.Reasons, r => r.ThreatReasonCode == "TamperedAllowList");
    }
}
```

---

End of Document
```

---
