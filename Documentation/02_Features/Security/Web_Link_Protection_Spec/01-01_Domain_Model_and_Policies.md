# 01-01: Domain Model and Policies

**Document ID:** GST-SPEC-WEBLINK-001-PART1  
**Version:** 3.0 (SilentReject Policy & 8-Stage Precedence Synchronized)
**Parent Document:** Web Link Protection Specification  
**Category:** Domain Logic & Policy Baseline  
**Status:** Approved Baseline Candidate  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Domain Layer の責務と設計原則

Web Link Protection における Domain Layer は、URL 起動に関連する**ビジネスルール・セキュリティポリシー・ホスト評価ロジック**を集約する。

### Domain 規約:
* **Pure C# の徹底:** WPF, EF Core, SQLite, Win32 API, HTTP 通信ライブラリへの依存を厳禁とする。
* **UI 文言の非保持:** 判定結果として自然言語文字列を保持せず、列挙型（`PolicyDecision` 等）およびドメインシグナルのみを生成・返却する。
* **決定論的評価:** 入力パラメータに対し、副作用なく決定論的にポリシー評価を下す。

---

# 2. 列挙型 & 値オブジェクト (Domain Enums & Value Objects)

```csharp
namespace GameSecurityTool.Domain.Features.WebLinkProtection.Enums;

public enum PolicyDecision
{
    Allow,         // 承認済みブラウザで即時起動
    Confirm,       // ユーザー確認 (Browser Picker / Prompt) を要求
    Block,         // ブラウザ起動を遮断 (通知ダイアログ表示)
    SilentReject   // ゲームプレイを最優先し、画面遷移や通知を出さずに静かに却下
}

public enum HostCategory
{
    Trusted,      // 公式・信頼済みドメイン
    Social,       // SNS・ソーシャルメディア
    Community,    // フォーラム・掲示板・コミュニティ
    Marketplace,  // ストア・決済
    Unknown,      // 未分類・未知のホスト
    Blocked       // 明示的ブロック対象
}

public enum MatchMode
{
    ExactHost,          // 完全一致 (例: discord.com)
    IncludeSubdomains   // サブドメイン一致 (例: *.discord.com)
}

public enum RuleSource
{
    BuiltIn,      // システム既定
    UserDefined   // ユーザー登録
}

public enum QueryPolicyMode
{
    StripAll,         // 全 Query String を破棄
    AllowlistedOnly   // 明示許可キーのみ維持
}

public enum OriginClassification
{
    GameOrigin,         // 監視対象ゲームプロセスからの起動
    UserOrigin,         // GST GUI からの明示操作
    ExternalAppOrigin,  // 外部プロセスからの起動
    UnknownOrigin       // 発信元不明
}
```

---

# 3. Domain Entities (`WebHostRule`)

```csharp
namespace GameSecurityTool.Domain.Features.WebLinkProtection.Entities;

using System;
using GameSecurityTool.Domain.Features.WebLinkProtection.Enums;

public sealed class WebHostRule
{
    public Guid Id { get; init; }
    public Guid? GameProfileId { get; init; } // null = Global ルール
    public string Host { get; private set; } = string.Empty;
    public string DisplayName { get; private set; } = string.Empty;
    public HostCategory Category { get; private set; }
    public RuleSource Source { get; init; }
    public MatchMode MatchMode { get; private set; }
    public PolicyDecision Policy { get; private set; }
    public bool IsEnforced { get; init; }     // true = Global Enforced (上書き不可)
    public bool IsEnabled { get; private set; }
    public DateTimeOffset CreatedAt { get; init; }
    public DateTimeOffset ModifiedAt { get; private set; }
    public DateTimeOffset? ExpiresAt { get; private set; }
    public string? Reason { get; private set; }

    public bool IsActive(DateTimeOffset utcNow)
    {
        if (!IsEnabled) return false;
        if (ExpiresAt.HasValue && ExpiresAt.Value <= utcNow) return false;
        return true;
    }

    public void UpdatePolicy(PolicyDecision newPolicy, MatchMode newMode, string? reason, DateTimeOffset utcNow)
    {
        Policy = newPolicy;
        MatchMode = newMode;
        Reason = reason;
        ModifiedAt = utcNow;
    }

    public void SetEnabled(bool enabled, DateTimeOffset utcNow)
    {
        IsEnabled = enabled;
        ModifiedAt = utcNow;
    }
}
```

---

# 4. ポリシー評価アルゴリズム ＆ 8段階決定表マッピング (HIGH-03 是正維持)

正規化済み Host に対し、製品共通の **Universal Rule Scope 8段階決定表** に従って厳密に評価する。

```text
[ 正規化済み Host の評価フロー ]
                │
                ▼
1. Global Enforced Block 照合 (最優先・上書き不可)
                │ (非該当)
                ▼
2. Global Enforced Allow 照合 (上書き不可・システム必須通信等)
                │ (非該当)
                ▼
3. GameProfile Block / SilentReject 照合 (個別設定)
                │ (非該当)
                ▼
4. Global Overrideable Block 照合
                │ (非該当)
                ▼
5. GameProfile Allow 照合 (Exact ➔ Subdomain)
                │ (非該当)
                ▼
6. Global Overrideable Allow 照合
                │ (非該当)
                ▼
7. Built-in SNS / Host Rule 照合
                │ (非該当)
                ▼
8. Default Policy (UnknownHostPolicy: Confirm / Block / SilentReject)
```

### 競合解決の絶対規則:
* **Global Enforced の絶対性:** Global Enforced Block に該当するドメイン（悪意あるドメイン等）は、GameProfile 側に UserDefined Allow が存在しても **絶対に解除できない**（即時 Block）。
* **Block / SilentReject 優先 (Fail-Safe):** 同一優先度内で Allow と Block / SilentReject が競合した場合、安全側である Block または SilentReject を採用する。

---

# 5. ホスト境界マッチングのセキュリティ規約

* **ExactHost (完全一致):** `string.Equals(targetHost, ruleHost, StringComparison.OrdinalIgnoreCase)`
* **IncludeSubdomains (サブドメイン一致):** `targetHost == ruleHost || targetHost.EndsWith("." + ruleHost, StringComparison.OrdinalIgnoreCase)`
* **禁止:** `targetHost.Contains("example.com")` や `EndsWith("example.com")` のような簡易文字列比較（`evil-example.com` の誤許可を防止）。
```

---
