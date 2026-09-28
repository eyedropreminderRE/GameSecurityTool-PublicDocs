# 01-04: Persistence and Database Architecture

**Document ID:** GST-ARCH-BASELINE-002-PART4  
**Version:** 3.3 (Repository Read-Path Boundary Clarification)
**Parent Document:** 00_Formal_Baseline_Overview.md  
**Category:** Persistence & Database Baseline  
**Status:** Approved Baseline  

---

# 19. EF Core / SQLite Baseline & Concurrency Model

GameSecurityTool（GST）では、ローカルストレージとして **SQLite + Entity Framework Core 10** を採用する。

## 19.1 データベース責務と保存対象
* **監査・追跡:** `AuditEventRecord` (Hash Chain 構造), `SystemChangeRecord` (WAL ジャーナル)
* **セキュリティポリシー:** `FirewallRuleRecord`, `AllowListEntry`, `TrustedLocationRecord`, `WebHostRuleRecord`
* **設定・プロファイル:** `SecurityProfilePreset`, `SecurityProfileInstance`, `UserSettingsRecord`
* **保護データメタデータ:** `SaveBackupSnapshotRecord`, `SaveBackupFileRecord`, `RestoreTransactionRecord`, `QuarantineEntryRecord`

## 19.2 SQLite 接続規約と WAL モード
* **接続文字列:** `Data Source=...;Mode=ReadWriteCreate;Cache=Shared;Foreign Keys=True;`
* **PRAGMA 設定:** 接続初期化時に以下を強制実行し、並行性とクラッシュ耐性を確保する。
  ```sql
  PRAGMA journal_mode = WAL;
  PRAGMA synchronous = NORMAL;
  PRAGMA busy_timeout = 5000;
  ```

---

## 19.3 読み取り並行性と書き込み直列化キュー (`IDbWriteQueue` / `SqliteDatabaseWriter`)

SQLite は単一ライター・複数リーダーモデルであるため、複数スレッドからの同時書き込みによる `SQLITE_BUSY`（database is locked）例外をアーキテクチャレベルで物理遮断する。

さらに、二段階コミット（①コンテナ作成 ➔ ②DBコミット ➔ ③元ファイル削除）の順序性を保証するため、**書き込み要求はキュー投入後、物理コミットが完了するまで呼び出し元を非同期待機（Synchronous Completion）させる**。

```text
[ 読み取り処理 (Query) ]
Application / ViewModel ──> Repository Port ──> Infrastructure Repository ──> IDbContextFactory<AppDbContext> ──> 短命 DbContext ──> SQLite (並行読込: AsNoTracking)

[ 書き込み処理 (Command / Transaction) ]
All Repositories ──> IDbWriteQueue.EnqueueWriteAsync(action, ct)
                               │ (WorkItem + TaskCompletionSource 投入)
                               ▼
                    [ Channel<WriteWorkItem> ]
                               │
                               ▼
                    [ Single Background Worker (SqliteDatabaseWriter) ]
                               │
                               ├─> context-free item.Action(ct) を1件ずつ直列実行
                               ├─> Action側のInfrastructure実装が IDbContextFactory<AppDbContext>
                               │   から短命 DbContext を生成・使用・破棄し、必要な SaveChangesAsync を実行
                               │
                               ├─(Action / コミット成功)──> tcs.SetResult() ──> 呼び出し元 await 完了
                               └─(Action / コミット失敗)──> tcs.SetException(ex) ➔ 呼び出し元へ例外伝播
```

### 19.3.1 `Contracts` Port インターフェース定義 (`GameSecurityTool.Contracts`)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System;
using System.Threading;
using System.Threading.Tasks;

public interface IDbWriteQueue
{
    /// <summary>
    /// SQLite への書き込み処理を直列化キューへ投入し、物理コミットが完了するまで非同期待機します。
    /// </summary>
    Task EnqueueWriteAsync(
        Func<CancellationToken, Task> writeAction,
        CancellationToken ct = default);
}
```

### 19.3.2 `SqliteDatabaseWriter` 完全実装仕様 (`GameSecurityTool.Infrastructure.Persistence`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Persistence;

using System;
using System.Threading;
using System.Threading.Channels;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Interfaces;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

public sealed class SqliteDatabaseWriter : BackgroundService, IDbWriteQueue
{
    private readonly ILogger<SqliteDatabaseWriter> _logger;
    private readonly Channel<WriteWorkItem> _channel;

    private sealed record WriteWorkItem(
        Func<CancellationToken, Task> Action,
        TaskCompletionSource CompletionSource,
        CancellationToken CancellationToken
    );

    public SqliteDatabaseWriter(
        ILogger<SqliteDatabaseWriter> logger)
    {
        _logger = logger;
        _channel = Channel.CreateBounded<WriteWorkItem>(new BoundedChannelOptions(500)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = true,
            SingleWriter = false
        });
    }

    /// <summary>
    /// 書き込み処理をキューへ投入し、物理コミット完了を待機 (IDbWriteQueue 実装)
    /// </summary>
    public async Task EnqueueWriteAsync(
        Func<CancellationToken, Task> writeAction,
        CancellationToken ct = default)
    {
        if (writeAction is null) throw new ArgumentNullException(nameof(writeAction));

        var tcs = new TaskCompletionSource(TaskCreationOptions.RunContinuationsAsynchronously);
        var workItem = new WriteWorkItem(writeAction, tcs, ct);

        await _channel.Writer.WriteAsync(workItem, ct);

        using (ct.Register(() => tcs.TrySetCanceled(ct)))
        {
            await tcs.Task.ConfigureAwait(false);
        }
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("SqliteDatabaseWriter 直列化キューワーカーを開始しました。");

        while (await _channel.Reader.WaitToReadAsync(stoppingToken))
        {
            while (_channel.Reader.TryRead(out var item))
            {
                if (item.CancellationToken.IsCancellationRequested)
                {
                    item.CompletionSource.TrySetCanceled(item.CancellationToken);
                    continue;
                }

                // DbContext creation/transaction lifetime is owned by the Infrastructure callback closure.
                // The queue itself only serializes execution and waits for callback completion.
                try
                {
                    await item.Action(item.CancellationToken);
                    item.CompletionSource.TrySetResult();
                }
                catch (OperationCanceledException) when (item.CancellationToken.IsCancellationRequested)
                {
                    item.CompletionSource.TrySetCanceled(item.CancellationToken);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "SQLite 直列書き込み実行エラー");
                    item.CompletionSource.TrySetException(ex);
                }
            }
        }

        _logger.LogInformation("SqliteDatabaseWriter 直列化キューワーカーを安全に停止しました。");
    }
}
```

---

# 20. Repository Pattern & レイヤー境界

Application Layer は具象 DbContext を直接触らず、Contracts 層で定義された Repository Port のみを経由してデータアクセスを行う。

### 規約:
1. **読み取り (Queries):** `IDbContextFactory<AppDbContext>` から生成された短命 `DbContext` を用いて、`AsNoTracking()` で並行実行する。
2. **書き込み (Commands):** 必ず `IDbWriteQueue` を経由して直列実行し、コミット完了を待機する。
3. **マッピング (Domain/Record Separation):** EF Core のエンティティ（`*Record`）を Domain Entity として直接扱ってはならない。必ず Repository 内部で相互変換を行う。

---

# 38. Universal Rule Scope Baseline

GST では機能ごとに個別のスコープ管理を乱立させず、共通の **Universal Rule Scope Model** を適用する。

## 38.1 スコープ階層
```text
RuleScope
├─ Global (全ゲーム・全機能共通)
│   ├─ Enforced    (絶対適用・GameProfile で解除不可)
│   └─ Overrideable(GameProfile 側で上書き可能)
└─ GameProfile (指定ゲームプロファイル専用)
```

## 38.2 評価順位と競合解決 (8段階決定表 - HIGH-03 是正)
ルール競合時は、常に安全側である **Block（拒否・遮断）を最優先（Fail-Safe）** とする。また、「Enforced」の定義に基づき、Global Enforced ルールは GameProfile ルールより上位に位置づける。

| 優先順位 | スコープ / モード | 判定動作 | 説明 |
| :---: | :--- | :--- | :--- |
| **1** | **Global Enforced Block** | 即時 Block | 全ゲーム共通の絶対遮断ルール。個別プロファイルで解除不可。 |
| **2** | **Global Enforced Allow** | 即時 Allow | 全ゲーム共通の絶対許可ルール（システム不可欠な通信等）。プロファイル側で遮断不可。 |
| **3** | **GameProfile Block** | 即時 Block | 特定ゲームプロファイルで明示された遮断ルール。 |
| **4** | **Global Overrideable Block** | Block / 上書き判定 | グローバルの既定遮断ルール。GameProfile 側の明示 Allow で上書き可能。 |
| **5** | **GameProfile Allow** | Allow | 特定ゲームプロファイルで明示された例外許可ルール。 |
| **6** | **Global Overrideable Allow** | Allow | グローバルの既定許可ルール。 |
| **7** | **Built-in Rule** | 定義に従う | システム初期登録ルール（SNS ドメイン等）。 |
| **8** | **Default Policy** | 既定判定 | 未登録対象に対する既定ポリシー（Confirm / Block）。 |

※ **競合解決の鉄則:** 同一優先度内で Allow と Block が競合した場合、必ず `Block` を採用する。

---

# 39. Universal Feature Control Policy Baseline

「ルールの適用範囲（RuleScope）」と「機能自体の ON/OFF（FeatureScope）」を厳密に分離して管理する。

```text
FeatureScope
├─ Global       (全ゲーム共通の既定状態: Enabled / Disabled)
└─ GameProfile  (特定ゲームに対する個別状態: UseGlobal / Enabled / Disabled)
```

---

# 44. Audit Chain Enhancement (改ざん検知ハッシュチェーン)

GST の監査ログは、暗号論的 **Hash Chain (SHA-256 連鎖)** 構造で SQLite に永続化する。

```text
[ Genesis Block (Hash 0) ]
            │
            ▼
[ Event 1: Scan Started ] ─── SHA256(Event1 + Hash0) ───> CurrentHash 1
            │
            ▼
[ Event 2: Rule Blocked ] ─── SHA256(Event2 + Hash1) ───> CurrentHash 2
```

最新ハッシュは管理者権限保護領域または暗号化アンカーへ二重保存し、データベースの外部改ざんを起動時に自動検知する。

---

# 45. OperationId / TransactionId Baseline

- **`OperationId`**: ユーザー操作・業務ユースケース単位の追跡 ID（例: `GST-OP-20260827-001`）。
- **`TransactionId`**: 単一処理の原子性・二段階コミット単位の追跡 ID（例: `QRT-20260827-A82F`, `RST-20260827-55F1`）。

---

End of Document
```

---