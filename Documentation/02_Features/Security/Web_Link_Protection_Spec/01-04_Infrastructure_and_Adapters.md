# 01-04: Infrastructure and Adapters

**Document ID:** GST-SPEC-WEBLINK-001-PART4  
**Version:** 3.2
**Parent Document:** Web Link Protection Specification v3.0  
**Category:** Infrastructure & OS Adapters  
**Status:** Approved Baseline Candidate  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Infrastructure Layer の責務

Infrastructure Layer は、Web Link Protection における**URL 正規化・Strict Query Stripping 実装、各種ブラウザプロセスの安全な起動、WMI による緊急検知、PID 再利用対策ネイティブ検証（二段階権限要求）、および SQLite へのルール永続化（`IDbWriteQueue` 直列化）**を担当する。

---

# 2. UrlSanitizer (URL 正規化 & サニタイザー実装)

`.NET Uri` クラスを基盤とし、ホモグラフ攻撃やパラメータ追跡を防ぐ厳格な正規化を実施する。

```csharp
namespace GameSecurityTool.Infrastructure.Features.WebLinkProtection.Sanitizers;

using System;
using System.Collections.Generic;
using System.Globalization;
using System.Linq;
using System.Web;
using GameSecurityTool.Contracts.Features.WebLinkProtection.DTOs;
using GameSecurityTool.Contracts.Features.WebLinkProtection.Ports;
using GameSecurityTool.Domain.Features.WebLinkProtection.Enums;

public sealed class UrlSanitizer : IUrlSanitizer
{
    public SanitizedUrlResultDto Sanitize(string rawUrl, IReadOnlyList<string> allowedQueryParams)
    {
        if (string.IsNullOrWhiteSpace(rawUrl))
            return CreateInvalidResult();

        // 1. URI Parse (Scheme 検証: http / https のみ許可)
        if (!Uri.TryCreate(rawUrl, UriKind.Absolute, out var parsedUri) ||
            (parsedUri.Scheme != Uri.UriSchemeHttp && parsedUri.Scheme != Uri.UriSchemeHttps))
        {
            return CreateInvalidResult();
        }

        // 2. UserInfo (認証情報) の除去
        var builder = new UriBuilder(parsedUri)
        {
            UserName = string.Empty,
            Password = string.Empty
        };

        // 3. IDN (国際化ドメイン) / Punycode 正規化 & 偽装フラグ判定
        var idn = new IdnMapping();
        var isPunycode = false;
        try
        {
            var rawHost = builder.Host.TrimEnd('.');
            var asciiHost = idn.GetAscii(rawHost).ToLowerInvariant();
            
            // Punycode (xn--) が含まれる場合、偽装の可能性フラグを立てる
            isPunycode = asciiHost.StartsWith("xn--", StringComparison.OrdinalIgnoreCase) ||
                         asciiHost.Contains(".xn--", StringComparison.OrdinalIgnoreCase);
            
            builder.Host = asciiHost;
        }
        catch (ArgumentException)
        {
            return CreateInvalidResult(); // 不正な IDN 表記
        }

        // 4. Strict Query Stripping & Allowlist Filter (クエリ原則完全破棄)
        var redactedCount = 0;
        if (!string.IsNullOrEmpty(builder.Query))
        {
            var queryParams = HttpUtility.ParseQueryString(builder.Query);
            var filteredParams = HttpUtility.ParseQueryString(string.Empty);

            foreach (string? key in queryParams.Keys)
            {
                if (key != null && allowedQueryParams.Contains(key, StringComparer.OrdinalIgnoreCase))
                {
                    filteredParams[key] = queryParams[key];
                }
                else
                {
                    redactedCount++;
                }
            }

            // 【末尾 '?' 残存防止】クエリが空になった場合は null を代入して '?' を完全除去
            builder.Query = filteredParams.Count > 0 ? filteredParams.ToString() : null;
        }

        var finalUri = builder.Uri;
        var maskedPath = finalUri.AbsolutePath.Length > 30 
            ? finalUri.AbsolutePath[..27] + "..." 
            : finalUri.AbsolutePath;

        return new SanitizedUrlResultDto(
            IsValid: true,
            SanitizedUri: finalUri,
            Host: finalUri.Host,
            MaskedPath: maskedPath,
            RedactedQueryCount: redactedCount,
            Category: HostCategory.Unknown,
            IsPunycodeSuspicious: isPunycode);
    }

    public SanitizedUrlResultDto SanitizeForPreview(string rawUrl) => Sanitize(rawUrl, []);

    private static SanitizedUrlResultDto CreateInvalidResult() =>
        new(false, new Uri("about:blank"), string.Empty, string.Empty, 0, HostCategory.Blocked, false);
}
```

---

# 3. Browser Adapters & プロセス起動セキュリティ

ブラウザごとのコマンドライン引数仕様の差異を吸収し、引数インジェクション（Command Injection / Flag Manipulation）を物理的に排除する。

```csharp
namespace GameSecurityTool.Infrastructure.Features.WebLinkProtection.Adapters;

using System;
using System.Diagnostics;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Features.WebLinkProtection.DTOs;

public sealed class WindowsBrowserLauncher
{
    public static async Task<bool> LaunchAsync(
        BrowserOptionDto browser,
        Uri targetUri,
        CancellationToken cancellationToken)
    {
        return await Task.Run(() =>
        {
            try
            {
                // 防衛線 1: スキームの再検証 (Defense in Depth)
                if (targetUri.Scheme != Uri.UriSchemeHttp && targetUri.Scheme != Uri.UriSchemeHttps)
                {
                    return false;
                }

                // 防衛線 2: コマンドライン引数フラグ偽装の排除 (先頭ハイフン/スラッシュ禁止)
                var urlString = targetUri.AbsoluteUri;
                if (urlString.StartsWith('-') || urlString.StartsWith('/'))
                {
                    return false;
                }

                // 防衛線 3: OS既定ブラウザ (DefaultSystemBrowser) の安全なシェル起動
                if (browser.BrowserType == BrowserType.DefaultSystemBrowser)
                {
                    var defaultStartInfo = new ProcessStartInfo
                    {
                        FileName = urlString,
                        UseShellExecute = true // サニタイズ済み URI に対して OS 既定ハンドラーを使用
                    };
                    using var defaultProc = Process.Start(defaultStartInfo);
                    return defaultProc != null;
                }

                // 防衛線 4: 個別ブラウザ実行バイナリの起動制御
                var startInfo = new ProcessStartInfo
                {
                    FileName = browser.ExecutablePath,
                    UseShellExecute = false,
                    CreateNoWindow = true
                };

                // ブラウザ種別ごとの引数構築 (Firefox 互換性担保)
                switch (browser.BrowserType)
                {
                    case BrowserType.MozillaFirefox:
                        if (browser.IsPrivateBrowsingEnabled)
                        {
                            startInfo.ArgumentList.Add("-private-window");
                        }
                        startInfo.ArgumentList.Add("-url");
                        startInfo.ArgumentList.Add(urlString);
                        break;

                    case BrowserType.GoogleChrome:
                    case BrowserType.MicrosoftEdge:
                    case BrowserType.Brave:
                    case BrowserType.Vivaldi:
                    default:
                        if (browser.IsPrivateBrowsingEnabled)
                        {
                            var privateArg = GetChromiumPrivateBrowsingSwitch(browser.BrowserType);
                            if (!string.IsNullOrEmpty(privateArg))
                            {
                                startInfo.ArgumentList.Add(privateArg);
                            }
                        }
                        // Chromium 系のオプション終端セパレータ
                        startInfo.ArgumentList.Add("--");
                        startInfo.ArgumentList.Add(urlString);
                        break;
                }

                using var process = Process.Start(startInfo);
                return process != null;
            }
            catch (Exception ex)
            {
                // documented defensive fail-safe boundary: explicit defensive fail-safe boundary.
                // Production code must emit sanitized exception observability
                // through the documented defensive fail-safe boundary global logging boundary before returning false.
                _ = ex;
                return false; // Conservative Fail-Safe
            }
        }, cancellationToken);
    }

    private static string GetChromiumPrivateBrowsingSwitch(BrowserType type) => type switch
    {
        BrowserType.MicrosoftEdge => "-inprivate",
        BrowserType.GoogleChrome  => "--incognito",
        BrowserType.Brave         => "--incognito",
        BrowserType.Vivaldi       => "--incognito",
        _                         => string.Empty
    };
}
```

---

# 4. WMI Emergency Detection & Win32 プロセス同一性検証

Managed Launch Path を迂回して起動されたブラウザプロセスに対し、**二段階権限要求（Least Privilege）および PID 再利用（PID Reuse）対策の対称時間ウィンドウ検証** を行う。

```text
[ WMI Win32_ProcessStartTrace Event (PID, ProcessName) ]
                           │
                           ▼
1. OpenProcess (PROCESS_QUERY_LIMITED_INFORMATION) で SafeProcessHandle 取得
   (※ AppContainer や別整合性レベルでもアクセス拒否を起こさずに開く)
                           │ (オープン失敗 = 既に終了または権限不一致 ➔ 安全終了)
                           ▼
2. GetProcessTimes で CreationTime を取得
   検証: eventTimeUtc - 3s <= CreationTime <= eventTimeUtc + 3s (対称許容ウィンドウ)
                           │ (ウィンドウ外 = PID Reuse / 別プロセスと判定 ➔ ハンドル解放して中止)
                           ▼
3. QueryFullProcessImageNameW で実行バイナリの正規パスを取得・照合
                           │ (パス不一致 = 中止)
                           ▼
[ 緊急アクション: User Prompt または PROCESS_TERMINATE による安全停止 ]
```

### ネイティブ Interop & PID Reuse 対称時間検証実装 (`[LibraryImport]`):

```csharp
namespace GameSecurityTool.Infrastructure.Native;

using System;
using System.IO;
using System.Runtime.InteropServices;
using Microsoft.Win32.SafeHandles;

internal static partial class ProcessNativeMethods
{
    private const uint PROCESS_QUERY_LIMITED_INFORMATION = 0x1000;
    private const uint PROCESS_TERMINATE = 0x0001;

    [LibraryImport("kernel32.dll", SetLastError = true)]
    public static partial SafeProcessHandle OpenProcess(
        uint dwDesiredAccess,
        [MarshalAs(UnmanagedType.Bool)] bool bInheritHandle,
        int dwProcessId);

    [LibraryImport("kernel32.dll", SetLastError = true, StringMarshalling = StringMarshalling.Utf16)]
    [return: MarshalAs(UnmanagedType.Bool)]
    public static partial bool QueryFullProcessImageNameW(
        SafeProcessHandle hProcess,
        uint dwFlags,
        Span<char> lpExeName,
        ref uint lpdwSize);

    [LibraryImport("kernel32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    public static partial bool GetProcessTimes(
        SafeProcessHandle hProcess,
        out long lpCreationTime,
        out long lpExitTime,
        out long lpKernelTime,
        out long lpUserTime);

    [LibraryImport("kernel32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    public static partial bool TerminateProcess(SafeProcessHandle hProcess, uint uExitCode);

    /// <summary>
    /// PID 再利用攻撃 (PID Reuse) を防ぐため、最小権限 (QUERY_LIMITED) で生成時刻と実行パスを対称検証します。
    /// </summary>
    public static bool ValidateAndOpenProcess(
        int processId,
        DateTimeOffset eventTimeUtc,
        string expectedPath,
        out SafeProcessHandle? validatedQueryHandle)
    {
        // 1. 【二段階権限要求】初期検証は制限クエリ権限のみを要求 (Access Denied を回避)
        validatedQueryHandle = OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION, false, processId);
        if (validatedQueryHandle.IsInvalid)
        {
            validatedQueryHandle.Dispose();
            validatedQueryHandle = null;
            return false;
        }

        // 2. プロセス生成時刻の対称ウィンドウ検証 (PID Reuse 対策)
        if (!GetProcessTimes(validatedQueryHandle, out var creationTimeRaw, out _, out _, out _))
        {
            validatedQueryHandle.Dispose();
            validatedQueryHandle = null;
            return false;
        }

        var creationTime = DateTimeOffset.FromFileTime(creationTimeRaw);
        var tolerance = TimeSpan.FromSeconds(3);

        // イベント受信時刻から ±3秒の許容ウィンドウ外であれば PID 再利用または別プロセスと判定
        if (creationTime < eventTimeUtc - tolerance || creationTime > eventTimeUtc + tolerance)
        {
            validatedQueryHandle.Dispose();
            validatedQueryHandle = null;
            return false;
        }

        // 3. 実行ファイルパスの厳密照合
        Span<char> buffer = stackalloc char[1024];
        uint size = (uint)buffer.Length;
        if (!QueryFullProcessImageNameW(validatedQueryHandle, 0, buffer, ref size))
        {
            validatedQueryHandle.Dispose();
            validatedQueryHandle = null;
            return false;
        }

        var imagePath = buffer[..(int)size].ToString();
        var normalizedImagePath = Path.GetFullPath(imagePath);
        var normalizedExpectedPath = Path.GetFullPath(expectedPath);

        if (!string.Equals(normalizedImagePath, normalizedExpectedPath, StringComparison.OrdinalIgnoreCase))
        {
            validatedQueryHandle.Dispose();
            validatedQueryHandle = null;
            return false;
        }

        return true;
    }

    /// <summary>
    /// 検証済みプロセスを安全に強制停止 (必要な瞬間にのみ TERMINATE 権限を要求)
    /// </summary>
    public static bool TerminateValidatedProcess(int processId)
    {
        using var termHandle = OpenProcess(PROCESS_TERMINATE, false, processId);
        if (termHandle.IsInvalid) return false;
        return TerminateProcess(termHandle, 1);
    }
}
```

---

# 5. SqliteSocialHostRuleRepository (永続化 Adapter - IDbWriteQueue 適用)

Contracts DTO を受け取り、内部の `WebHostRule` Entity へマッピングして **`IDbWriteQueue` Port** 経由で SQLite へ直列化保存する（CRIT-03 / Inviolable Guardrail ② 適用）。

```csharp
namespace GameSecurityTool.Infrastructure.Features.WebLinkProtection.Repositories;

using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Features.WebLinkProtection.DTOs;
using GameSecurityTool.Contracts.Features.WebLinkProtection.Ports;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Domain.Features.WebLinkProtection.Entities;
using GameSecurityTool.Infrastructure.Persistence;
using Microsoft.EntityFrameworkCore;

public sealed class SqliteSocialHostRuleRepository(
    IDbContextFactory<AppDbContext> dbContextFactory,
    IDbWriteQueue dbWriter) : ISocialHostRuleRepository
{
    public async Task<IReadOnlyList<SocialHostRuleDto>> GetActiveRulesAsync(
        Guid? gameProfileId,
        CancellationToken cancellationToken)
    {
        await using var context = await dbContextFactory.CreateDbContextAsync(cancellationToken);
        var now = DateTimeOffset.UtcNow;

        return await context.WebHostRules
            .AsNoTracking()
            .Where(r => (r.GameProfileId == null || r.GameProfileId == gameProfileId) && r.IsEnabled && (r.ExpiresAt == null || r.ExpiresAt > now))
            .Select(r => new SocialHostRuleDto(
                r.Id, r.GameProfileId, r.Host, r.DisplayName, r.Category,
                r.Source, r.MatchMode, r.Policy, r.IsEnabled,
                r.CreatedAt, r.ModifiedAt, r.ExpiresAt, r.Reason))
            .ToListAsync(cancellationToken);
    }

    public async Task<SocialHostRuleDto?> GetByIdAsync(Guid ruleId, CancellationToken cancellationToken)
    {
        await using var context = await dbContextFactory.CreateDbContextAsync(cancellationToken);
        var entity = await context.WebHostRules.AsNoTracking().FirstOrDefaultAsync(r => r.Id == ruleId, cancellationToken);
        if (entity == null) return null;

        return new SocialHostRuleDto(
            entity.Id, entity.GameProfileId, entity.Host, entity.DisplayName, entity.Category,
            entity.Source, entity.MatchMode, entity.Policy, entity.IsEnabled,
            entity.CreatedAt, entity.ModifiedAt, entity.ExpiresAt, entity.Reason);
    }

    public async Task AddAsync(SocialHostRuleDto ruleDto, CancellationToken cancellationToken)
    {
        var entity = new WebHostRule(
            ruleDto.Id, ruleDto.GameProfileId, ruleDto.Host, ruleDto.DisplayName,
            ruleDto.Category, ruleDto.Source, ruleDto.MatchMode, ruleDto.Policy,
            ruleDto.IsEnabled, ruleDto.CreatedAt, ruleDto.ModifiedAt, ruleDto.ExpiresAt, ruleDto.Reason);

        await dbWriter.EnqueueWriteAsync(async (CancellationToken ct) =>
        {
            await using var db = await dbContextFactory.CreateDbContextAsync(ct);
            db.WebHostRules.Add(entity);
            await db.SaveChangesAsync(ct);
        }, cancellationToken);
    }

    public async Task UpdateAsync(UpdateSocialHostRuleDto ruleDto, CancellationToken cancellationToken)
    {
        await dbWriter.EnqueueWriteAsync(async (CancellationToken ct) =>
        {
            await using var db = await dbContextFactory.CreateDbContextAsync(ct);
            var entity = await db.WebHostRules.FirstOrDefaultAsync(r => r.Id == ruleDto.Id, ct);
            if (entity != null)
            {
                entity.UpdatePolicy(ruleDto.Policy, ruleDto.MatchMode, ruleDto.Reason, DateTimeOffset.UtcNow);
                entity.SetEnabled(ruleDto.IsEnabled, DateTimeOffset.UtcNow);
                await db.SaveChangesAsync(ct);
            }
        }, cancellationToken);
    }

    public async Task DeleteAsync(Guid ruleId, CancellationToken cancellationToken)
    {
        await dbWriter.EnqueueWriteAsync(async (CancellationToken ct) =>
        {
            await using var db = await dbContextFactory.CreateDbContextAsync(ct);
            var entity = await db.WebHostRules.FirstOrDefaultAsync(r => r.Id == ruleId, ct);
            if (entity != null)
            {
                db.WebHostRules.Remove(entity);
                await db.SaveChangesAsync(ct);
            }
        }, cancellationToken);
    }
}
```

---

End of Document
```

---
