# GameSecurityTool On-Demand Clipboard URL Sanitizer 仕様書

**文書ID:** GST-FEAT-CLIP-001  
**版:** 3.0 (STA Thread Safety, Retry Logic & Infrastructure Implementation Hardened)  
**状態:** 採用確定（P1）  
**対象:** Windows 10 / Windows 11 / .NET 10 / C# 14 / WPF  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

## 1. 目的

ユーザーが外部コミュニティ（チャット等）、ブラウザ等からコピーした URL を、**ユーザーの明示操作時のみ安全化（サニタイズ）** する。

Web Link Protection の Game-Origin 保護とは分離し、User-Origin の操作主権を尊重した補助機能として提供する。

---

## 2. 基本方針

- **常時クリップボード監視（Background Polling）の完全禁止**
- **クリップボード本文の自動書き換え禁止**
- ユーザーが明示的に「URLをクリーニング」「安全化して開く」を押下した場合のみ処理
- 外部通信ゼロ・完全ローカル完結
- ブラウザで開く前にサニタイズ後のプレビューを必ず表示する

---

## 3. UI 機能

### 3.1 Clean URL 画面

入力:
- URL テキストボックス（手動入力または「クリップボードから貼り付け」ボタン）
- 「安全化」ボタン

出力:
- Original URL（画面上の一時プレビュー表示のみ。ログや DB に保存しない）
- Sanitized URL
- 削除したパラメータ数
- 判定結果（危険スキーム警告、Punycode 偽装警告）
- 「コピー」ボタン
- 「安全にブラウザで開く」ボタン

---

## 4. URL 処理パイプライン

```text
[ Clipboard / Manual Input ]
             │
             ▼
[ STA Thread Dispatch (IDispatcherService) ]
             │
             ▼
[ Parse & Scheme Validation (http/https のみ許可) ]
             │
             ▼
[ Host Canonicalization & Punycode 検査 ]
             │
             ▼
[ Strict Query Stripping (クエリ原則全削除) ]
             │
             ▼
[ Risk & Category Evaluation ]
             │
             ▼
[ Preview Display (Privacy-Safe) ]
             │
             ▼
[ User Action (Copy / Open) ]
```

---

## 5. Strict Query Stripping

デフォルトでは `?` 以降のクエリ文字列をすべて削除する。

例:
```text
https://example.com/page?session_token=abc&utm_source=game&user_id=12345
↓
https://example.com/page
```

理由:
- `utm_*` 等の既知追跡タグだけでは未知のトラッキングパラメータを網羅できない
- セッション ID や認証トークンの漏洩面を構造的に最小化する
- デフォルト全削除の方がユーザーにとって説明しやすく直感的である

※ 例外パラメータの許可（Query Allowlist）は将来拡張とし、初期実装では提供しない。

---

## 6. URL 正規化規約

処理順序を固定する。

`Parse ➔ Normalize ➔ Validate ➔ Sanitize`

考慮する事項:
- URL エンコード / 二重エンコードの正規化
- Host の小文字化および末尾ドット除去
- Unicode / Punycode（`xn--`）の偽装フラグ判定
- ユーザー情報部分（`user:pass@`）の完全除去
- IP アドレス直打ち表記の検知
- 予期しない非標準ポートの検知

許可スキーム:
- `http`
- `https`

※ それ以外のスキーム（`javascript:`, `file:`, `data:` 等）は即時拒否（Block）。

---

## 7. OS クリップボード API ＆ STA スレッド安全性

- **STA スレッド境界の保証:**
  Win32 / WPF のクリップボード API は STA（Single-Threaded Apartment）スレッドでのみ動作するため、`IClipboardUrlService` は必ず `IDispatcherService` を経由して UI スレッド上でクリップボードへアクセスする。
- **クリップボード排他ロック（`CLIPBRD_E_CANT_OPEN: 0x800401D0`）の自動リトライ:**
  他プロセス（クリップボード履歴ツール等）がクリップボードを開いている最中の競合エラーに対し、最大 3 回の指数バックオフ（50ms ➔ 100ms ➔ 200ms）でリトライを実行する。
- **安全なブラウザ起動:**
  ブラウザ起動時は `WindowsBrowserLauncher`（Managed Launch Path）を再利用し、引数インジェクションを遮断する。

---

## 8. Game-Origin との関係

本機能は Game-Origin の URL 起動を直接捕捉・フックするものではない。

- Game-Origin: Web Link Protection の Managed Launch Path 側で処理。
- User-Origin: 本機能（On-Demand Clipboard URL Sanitizer）でユーザー主導処理。

---

## 9. プライバシー保護規約 ＆ Universal Privacy Shield

### 9.1 保存禁止（Strict Prohibitions）
- Original URL
- Query String
- トークン・認証情報
- クリップボードの履歴データ

### 9.2 Universal Privacy Shield マスキング規約
クリップボードから取得したテキストや URL 内に含まれる以下の機密情報は、表示・ログ出力・外部連携前に `ILogSanitizer`（C# 14 `[GeneratedRegex]`）によって自動的かつ完全にマスキングされる：
- **Discord Webhook URL:** `https://discord.com/api/webhooks/...` ➔ `[REDACTED_WEBHOOK]`
- **Twitch OAuth トークン:** `oauth:...` ➔ `[REDACTED_TWITCH_TOKEN]`
- **Authorization Bearer トークン:** `Bearer ...` ➔ `[REDACTED_BEARER_TOKEN]`
- **個人認証・識別情報:** メールアドレス (`***@***.***`)、MAC アドレス (`**:**:**:**:**:**`)、ローカル IP アドレス、ホスト名・ユーザー名パス (`C:\Users\***\...`)

### 9.3 監査ログ（Audit Log）の最小情報記録
監査ログに保存可能なのは以下の最小情報のみとする:
- `OperationId`
- `RedactedCount` (削除したパラメータ数)
- `HostCategory` (匿名カテゴリ)
- `Result` (Success / InvalidScheme / Blocked)

---

## 10. エラーハンドリング

- **URL 形式不正:** ユーザーへ修正案内を表示。
- **危険スキーム:** ブラウザを起動せず即時終了・警告表示。
- **Sanitization 失敗:** 原文をそのまま開かず、安全側に停止。
- **クリップボードアクセス失敗（リトライ上限超過）:** 「クリップボードが他のアプリで使用中です。手動で貼り付けてください」と案内。

---

## 11. Contracts Port 定義 (`GameSecurityTool.Contracts`)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System.Threading;
using System.Threading.Tasks;

public interface IClipboardUrlService
{
    /// <summary>
    /// クリップボードからテキストを安全に取得 (STA スレッドディスパッチ ＆ ロックリトライ適用)
    /// </summary>
    Task<string?> GetClipboardTextAsync(CancellationToken ct = default);

    /// <summary>
    /// サニタイズ済み URL をクリップボードへ安全に書き戻し
    /// </summary>
    Task<bool> SetClipboardTextAsync(string text, CancellationToken ct = default);
}
```

---

## 12. Infrastructure Layer 完全実装 (`WindowsClipboardUrlService.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Native;

using System;
using System.Runtime.InteropServices;
using System.Threading;
using System.Threading.Tasks;
using System.Windows;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.Extensions.Logging;

public sealed class WindowsClipboardUrlService(
    IDispatcherService dispatcherService,
    ILogger<WindowsClipboardUrlService> logger) : IClipboardUrlService
{
    private const int MaxRetries = 3;
    private const int CantOpenClipboardErrorCode = unchecked((int)0x800401D0); // CLIPBRD_E_CANT_OPEN

    public async Task<string?> GetClipboardTextAsync(CancellationToken ct = default)
    {
        for (int i = 0; i < MaxRetries; i++)
        {
            ct.ThrowIfCancellationRequested();
            try
            {
                string? result = null;
                await dispatcherService.InvokeAsync(() =>
                {
                    if (Clipboard.ContainsText())
                    {
                        result = Clipboard.GetText();
                    }
                });
                return result;
            }
            catch (COMException ex) when (ex.ErrorCode == CantOpenClipboardErrorCode)
            {
                if (i == MaxRetries - 1)
                {
                    logger.LogWarning(ex, "クリップボードのオープンに失敗しました (リトライ上限到達)。");
                    return null;
                }
                await Task.Delay(50 * (int)Math.Pow(2, i), ct); // 50ms, 100ms, 200ms
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "クリップボードからのテキスト取得中に予期せぬ例外が発生しました。");
                return null;
            }
        }

        return null;
    }

    public async Task<bool> SetClipboardTextAsync(string text, CancellationToken ct = default)
    {
        if (text == null) return false;

        for (int i = 0; i < MaxRetries; i++)
        {
            ct.ThrowIfCancellationRequested();
            try
            {
                await dispatcherService.InvokeAsync(() =>
                {
                    Clipboard.SetText(text);
                });
                return true;
            }
            catch (COMException ex) when (ex.ErrorCode == CantOpenClipboardErrorCode)
            {
                if (i == MaxRetries - 1)
                {
                    logger.LogWarning(ex, "クリップボードへの書き戻しに失敗しました (リトライ上限到達)。");
                    return false;
                }
                await Task.Delay(50 * (int)Math.Pow(2, i), ct);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "クリップボードへのテキスト設定中に予期せぬ例外が発生しました。");
                return false;
            }
        }

        return false;
    }
}
```

---

## 13. 単体テスト仕様 (`GST.UnitTests.Privacy.Clipboard`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.Privacy.Clipboard;

using System;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Infrastructure.Native;
using Microsoft.Extensions.Logging.Abstractions;
using Moq;
using Xunit;

public class ClipboardUrlServiceTests
{
    [Fact]
    public async Task GetClipboardTextAsync_DispatchesToDispatcherService()
    {
        var mockDispatcher = new Mock<IDispatcherService>();
        mockDispatcher.Setup(d => d.InvokeAsync(It.IsAny<Func<Task>>()))
                      .Returns(Task.CompletedTask);

        var service = new WindowsClipboardUrlService(mockDispatcher.Object, NullLogger<WindowsClipboardUrlService>.Instance);

        // ※ 単体テスト環境での STA 呼び出し検証
        var text = await service.GetClipboardTextAsync();

        mockDispatcher.Verify(d => d.InvokeAsync(It.IsAny<Action>()), Times.Once);
    }
}
```

---

## 14. テスト要件

- UTM パラメータの完全除去
- session/token パラメータの完全除去
- 二重エンコード URL の正規化
- Unicode / Punycode Host の偽装フラグ検証
- 不正 Scheme（`javascript:` 等）の即時拒否
- 空 URL / 長大 URL の安全処理
- **バックグラウンドスレッドからの呼び出し時に STA クラッシュが発生しないこと**
- **クリップボードロック競合時に自動リトライが機能すること**
- ユーザーキャンセル時の安全停止

---

## 15. ロードマップ

**P1 / MVP 重要 UX 機能** として実装する。

既存の `StrictQueryUrlSanitizer`、`WebLinkPolicy`、`PrivacyLogSanitizer`、`WindowsBrowserLauncher` を再利用すること。

---