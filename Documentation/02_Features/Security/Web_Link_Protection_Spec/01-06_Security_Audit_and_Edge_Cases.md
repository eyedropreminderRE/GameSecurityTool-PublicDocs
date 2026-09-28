# 01-06: Security Audit and Edge Cases

**Document ID:** GST-SPEC-WEBLINK-001-PART6  
**Version:** 3.1 (IP Literal Symmetry Tests & Edge-Case Hardened Edition)
**Parent Document:** Web Link Protection Specification v3.0  
**Category:** Security, Audit & Testing Baseline  
**Status:** Approved Baseline Candidate  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Audit Logging & Privacy Baseline (監査ログ規約)

Web Link Protection に関するすべての操作・判定・緊急検知は、GST の改ざん検知ハッシュチェーン（Audit Hash Chain）に記録する。

## 1.1 監査イベント一覧 (`AuditEventType` 連携)
| イベント種別 | 整数値 | 発火タイミング | 記録目的 |
| :--- | :---: | :--- | :--- |
| `WebLinkReceived` | `50` | URL 起動要求の受信 | 操作開始の追跡 (`OperationId` 発行) |
| `WebLinkSanitized` | `51` | Strict Query Stripping 完了 | パラメータ破棄数の記録 |
| `WebLinkBlocked` | `52` | ポリシー判定による遮断 | ブロック理由および該当ルールの監査 |
| `WebLinkConfirmed` | `53` | ユーザー確認プロンプト表示 | ユーザー同意フローの追跡 |
| `WebBrowserSelected` | `54` | ブラウザ選択 & 起動成功 | 起動先ブラウザと起動成否の監査 |
| `WebLinkEmergencyDetected` | `55` | WMI による迂回ブラウザ起動検知 | 未管理起動の異常検知記録 |
| `WebBrowserEmergencyStopped` | `56` | 迂回プロセスの緊急停止実行 | 被害抑止アクションの監査 |
| `ShortcutChangedDetected` | `57` | ショートカット改ざん検知 | リンク先改ざんの追跡 |

## 1.2 Privacy-First 記録ルール
個人情報漏洩およびトラッキングを防止するため、監査ログへの記録項目を厳格に制限する。

### 保存を許可する属性:
* `OperationId` (例: `GST-URL-20260813-A1B2C3D4`)
* `TimestampUtc` (UTC)
* `GameProfileId`
* `Action` (Blocked / Launched / Prompted / Terminated)
* `Result` (Success / Failed / UserRejected)
* `HostCategory` (Social / Community / Trusted / Unknown)
* `RedactedQueryCount` (破棄したパラメータ数)
* `BrowserType` (起動したブラウザ種別)

### 保存を厳禁とする属性 (Strict Prohibitions):
* **完全な URL 文字列 (Raw URL)**
* **Query String 全文 (パラメータ名および値)**
* **セッショントークン / OAuth コード / 認証情報**
* **ユーザー名を含むプロファイル絶対パス**

---

# 2. Testing Requirements & Verification Matrix

## 2.1 Unit Tests (単体テストマトリクス)
以下のロジックが OS や外部環境に依存せず、すべての定義済み 決定論的にパスすることを検証する：

1. **Strict Query Stripping:**
   * クエリ全削除（`https://example.com/p?a=1&b=2` ➔ `https://example.com/p`、末尾 `?` 残存ゼロ）
   * Allowlist 指定パラメータのみ維持（`?lang=ja&token=secret` ➔ `?lang=ja`）
   * フラグメント（`#hash`）の安全な取り扱い
2. **Host Canonicalization & Matching:**
   * IDN / Punycode 正規化（`xn--...` の正確な相互変換）
   * 大文字小文字の区別なし（`EXAMPLE.COM` == `example.com`）
   * ドット境界サブドメイン一致（`api.discord.com` は `discord.com` にマッチ、`fake-discord.com` はマッチしない）
   * ポート番号付きホスト（`example.com:8080`）の正確なホスト分離
3. **生 IP アドレスリテラルの登録・評価対称性 (L-H2 是正):**
   * IPv4 リテラル（`192.168.1.1`）での登録と `http://192.168.1.1/login` 評価の完全一致
   * ポート付き IPv4（`127.0.0.1:8080`）からのホスト抽出一致
   * 前後空白混入入力（`  discord.com  `）のトリム・正規化一致
4. **Policy Priority & Conflict (8段階決定表):**
   * 優先順位（`Global Enforced Block > GameProfile Block > Global Overrideable Block > ...`）
   * 競合時の Block 優先（`Block > Allow`）
   * ルール有効期限（`ExpiresAt`）切れの TOCTOU リアルタイム無効化

---

# 3. 単体テストコード仕様 (`GST.UnitTests.WebLink.Security`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.WebLink.Security;

using System;
using System.Collections.Generic;
using GameSecurityTool.Contracts.Features.WebLinkProtection.DTOs;
using GameSecurityTool.Domain.Features.WebLinkProtection.Enums;
using GameSecurityTool.Infrastructure.Features.WebLinkProtection.Sanitizers;
using Xunit;

public class WebLinkSecurityTests
{
    private readonly UrlSanitizer _sanitizer = new();

    [Theory]
    [InlineData("https://example.com/login?token=secret123&utm_source=game", "https://example.com/login", 2)]
    [InlineData("https://example.com:8080/view?session=abc#section1", "https://example.com:8080/view#section1", 1)]
    [InlineData("http://192.168.1.50/dashboard?admin=true", "http://192.168.1.50/dashboard", 1)]
    public void Sanitize_StrictQueryStripping_RemovesAllQueriesWithoutTrailingQuestionMark(
        string rawUrl, string expectedUrl, int expectedRedactedCount)
    {
        var result = _sanitizer.Sanitize(rawUrl, []);

        Assert.True(result.IsValid);
        Assert.Equal(expectedUrl, result.SanitizedUri.AbsoluteUri);
        Assert.Equal(expectedRedactedCount, result.RedactedQueryCount);
        Assert.False(result.SanitizedUri.AbsoluteUri.EndsWith('?'));
    }

    [Theory]
    [InlineData("https://admin:password123@example.com/dashboard", "https://example.com/dashboard")]
    [InlineData("http://user@127.0.0.1/api", "http://127.0.0.1/api")]
    public void Sanitize_UserInfoPresent_StripsCredentialsCompletely(string rawUrl, string expectedUrl)
    {
        var result = _sanitizer.Sanitize(rawUrl, []);

        Assert.True(result.IsValid);
        Assert.Equal(expectedUrl, result.SanitizedUri.AbsoluteUri);
        Assert.Empty(result.SanitizedUri.UserInfo);
    }

    [Fact]
    public void Sanitize_PunycodeHost_DetectsSuspiciousFlag()
    {
        // 偽装リンゴドメイン (apple.com 酷似の Punycode)
        string spoofedUrl = "https://xn--pple-43d.com/login";
        var result = _sanitizer.Sanitize(spoofedUrl, []);

        Assert.True(result.IsValid);
        Assert.True(result.IsPunycodeSuspicious);
        Assert.Equal("xn--pple-43d.com", result.Host);
    }

    [Theory]
    [InlineData("javascript:alert(1)")]
    [InlineData("file:///C:/Windows/System32/calc.exe")]
    [InlineData("data:text/html,<h1>Malicious</h1>")]
    [InlineData("custom-proto://execute")]
    public void Sanitize_InvalidOrDangerousSchemes_RejectsAsInvalid(string invalidUrl)
    {
        var result = _sanitizer.Sanitize(invalidUrl, []);

        Assert.False(result.IsValid);
        Assert.Equal(HostCategory.Blocked, result.Category);
    }

    [Theory]
    [InlineData("192.168.1.1", "http://192.168.1.1/test", true)]
    [InlineData("  discord.com  ", "https://discord.com/invite/game", true)]
    [InlineData("127.0.0.1", "http://127.0.0.1:8080/status", true)]
    public void HostMatching_IpLiteralsAndTrimmedHosts_MatchesSymmetrically(
        string registeredHost, string targetUrl, bool expectedMatch)
    {
        // 登録時サニタイズ
        var regResult = _sanitizer.SanitizeForPreview($"https://{registeredHost.Trim()}/");
        Assert.True(regResult.IsValid);
        string normalizedRegistered = regResult.Host;

        // 評価時サニタイズ
        var evalResult = _sanitizer.Sanitize(targetUrl, []);
        Assert.True(evalResult.IsValid);
        string normalizedTarget = evalResult.Host;

        bool isMatched = string.Equals(normalizedRegistered, normalizedTarget, StringComparison.OrdinalIgnoreCase);
        Assert.Equal(expectedMatch, isMatched);
    }
}
```

---

# 4. エッジケース仕様 & Fail-Safe 動作規約

| エッジケース | 想定シナリオ | GST の Fail-Safe 挙動 |
| :--- | :--- | :--- |
| **不正な URI スキーム** | `javascript:...`, `file://...`, `custom://...` | 即時 Block。HTTP / HTTPS 以外のスキームは一切起動しない。 |
| **Punycode 不正文字列** | デコード不能な不正 IDN 表記 | 即時 Block（`InvalidUrlRejected` を監査ログに記録）。 |
| **生 IPv6 アドレス** | `http://[::1]:8080/path` | ブラケット表記を保持して正規化し、安全にホスト抽出。 |
| **指定ブラウザの未インストール** | 選択されたブラウザの実行バイナリが存在しない | エラー通知の上、Browser Picker を再表示（勝手に既定ブラウザで開かない）。 |
| **WMI サービス停止 / クラッシュ** | OS の WMI サービスが停止している | WMI 緊急検知を Degraded 状態とし、通常の Managed Launch Path のみで保護を継続。 |
| **ネットワーク障害 / オフライン** | 完全オフライン環境でのゲーム起動 | 全判定がローカル完結しているため、通常通り全機能が正常動作する。 |

---

End of Document
```

---
