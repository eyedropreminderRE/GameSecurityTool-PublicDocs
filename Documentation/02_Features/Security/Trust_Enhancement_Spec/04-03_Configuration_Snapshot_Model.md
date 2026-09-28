# 04-03: Configuration Snapshot Model

**Document ID:** GST-FEAT-TRUST-003  
**Version:** 2.0 (Schema Versioning & Backward Compatibility Hardened)
**Parent Document:** 04-00_Overview_and_Principles.md  
**Category:** Domain & Data Specification  
**Status:** Approved Feature Specification  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. 目的

セキュリティイベント（検知、隔離、ブロック、URLサニタイズ）が発生した瞬間の **「完全な設定状態・ポリシー状態」をスナップショットとして固定保存** する。

後からの調査・監査タイムライン表示において、
* ユーザー設定が不足していたのか
* Global Enforced ルールが適用されたのか
* GameProfile による意図的な Override だったのか
* 検知エンジンのシグナルによるものだったのか
を すべての定義済み 確実に追跡・再現できるようにする。

---

# 2. スナップショット構造 (`ConfigurationSnapshot`)

Universal Rule Scope および Universal Feature Control Policy と完全統合する。

```csharp
namespace GameSecurityTool.Domain.Models;

using System;
using System.Collections.Generic;

public sealed record FeatureStateSnapshotEntry(
    string FeatureId,
    string FeatureScope, // "Global" / "GameProfile"
    bool GlobalEnabled,
    bool? ProfileOverrideEnabled,
    bool EffectiveEnabled
);

public sealed record RuleStateSnapshotEntry(
    string RuleId,
    string RuleType,     // "WebHost" / "Firewall" / "AllowList" 等
    string Scope,        // "Global" / "GameProfile"
    string Mode,         // "Enforced" / "Overrideable"
    string EffectiveDecision
);

public sealed record ConfigurationSnapshot(
    Guid SnapshotId,
    string SchemaVersion, // スキーマバージョン (初期: "1.0")
    string OperationId,
    DateTimeOffset CreatedAtUtc,
    string AppVersion,
    string SecurityProfileVersion,
    IReadOnlyList<FeatureStateSnapshotEntry> Features,
    IReadOnlyList<RuleStateSnapshotEntry> Rules,
    bool FirewallBlockActive,
    bool CrashProtectionActive
);
```

---

# 3. 永続化 ＆ デシリアライズ後方互換性仕様 (MED-06 解決)

1. **記録タイミング:** セキュリティ判定時、隔離実行時、URLアクセス判定時。
2. **ストレージ形式:** SQLite `AuditRecords` テーブルの `SnapshotData` カラム（JSON文字列）に保存。
3. **データ整合性:** 親となる `AuditEventRecord` と同一トランザクション内でコミットし、孤立レコードの発生を防止。
4. **後方互換性デシリアライズ規約:**
   将来のバージョンアップで新フィールドが追加された場合でも過去ログの復元に失敗しないよう、Infrastructure 層での JSON 読み書きには以下のオプションを強制する。

```csharp
namespace GameSecurityTool.Infrastructure.Persistence.Serialization;

using System.Text.Json;
using System.Text.Json.Serialization;

public static class SnapshotJsonSerializerOptions
{
    public static readonly JsonSerializerOptions Default = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        WriteIndented = false,
        // 未知のプロパティが存在しても安全にスキップして読み込む (後方互換性保証)
        UnmappedMemberHandling = JsonUnmappedMemberHandling.Skip,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
    };
}
```

---

# 4. 単体テスト仕様 (`GST.UnitTests.Trust.Snapshot`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.Trust.Snapshot;

using System;
using System.Collections.Generic;
using System.Text.Json;
using GameSecurityTool.Domain.Models;
using GameSecurityTool.Infrastructure.Persistence.Serialization;
using Xunit;

public class ConfigurationSnapshotTests
{
    [Fact]
    public void Deserialize_WhenJsonContainsUnknownFutureFields_DeserializesWithoutException()
    {
        // 将来のバージョンで追加された未知フィールド "futureGuardActive" を含む JSON
        string futureJson = """
        {
          "snapshotId": "a82f91c0-0000-0000-0000-000000000001",
          "schemaVersion": "1.1",
          "operationId": "OP-TEST-01",
          "createdAtUtc": "2026-08-27T12:00:00+00:00",
          "appVersion": "2.0.0",
          "securityProfileVersion": "Standard",
          "features": [],
          "rules": [],
          "firewallBlockActive": true,
          "crashProtectionActive": true,
          "futureGuardActive": true
        }
        """;

        var snapshot = JsonSerializer.Deserialize<ConfigurationSnapshot>(futureJson, SnapshotJsonSerializerOptions.Default);

        Assert.NotNull(snapshot);
        Assert.Equal("1.1", snapshot.SchemaVersion);
        Assert.True(snapshot.FirewallBlockActive);
    }
}
```

---

End of Document
```

