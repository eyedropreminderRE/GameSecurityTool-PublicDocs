# 01-06: Presentation and UI Architecture

**Document ID:** GST-ARCH-BASELINE-002-PART6  
**Version:** 3.2 (WIPER Presentation State Boundary Fixed)
**Parent Document:** Architecture & Technology Baseline v3.0 (GST-ARCH-BASELINE-002)  
**Category:** Presentation & UI Baseline  
**Status:** Approved Baseline Candidate  

---

# 13. WPF / MVVM Baseline

GST の Presentation Layer は **WPF (Windows Presentation Foundation) + MVVM (Model-View-ViewModel)** パターンを採用する。

## 13.1 MVVM Structure & 依存方向
```text
[ View (XAML / Code-behind) ]
             │
             ▼
[ ViewModel (ObservableObject / Commands) ]
             │
             ▼
[ Application Services / UseCases ]
             │
             ▼
[ Contracts (DTOs / Port Interfaces) ]
```

* **厳禁事項:** View または ViewModel から Infrastructure Layer（DB, Firewall, Win32 Native 等）への直接依存・直接呼び出しをビルドレベルで禁止する。

## 13.2 View の責務と禁止事項
* **View の責務:**
  * XAML レイアウト描画およびコントロール定義
  * Data Binding および Visual State（表示状態）の管理
  * Fluent Design / Mica・Acrylic 背景エフェクトの適用
  * UI アニメーションおよび Styles / Templates の適用
* **View の禁止事項:**
  * Database 直接操作
  * Firewall / Registry 操作
  * ファイル I/O 操作
  * セキュリティ判断・リスク評価ロジックの実装
  * 例: View コードビハインドから `FirewallManager.Block()` や `File.Delete()` を直接呼び出すことを厳禁とする。

## 13.3 ViewModel の責務と禁止事項
* **ViewModel の責務:**
  * View に対する表示用プロパティ（State）の保持と公開
  * ユーザー操作（RelayCommand / AsyncRelayCommand）の受付
  * Application UseCase の非同期呼び出し
  * Application から返却された Contracts DTO の受け取りと UI 向けコレクションへの展開
  * 確認ダイアログ（Confirmation Dialog）等のインタラクション制御
* **ViewModel の禁止事項:**
  * Security Engine インスタンスそのものの保持
  * `CalculateRiskScore()` や `EvaluateFirewallRules()` 等の Domain ロジック直接実装

## 13.4 CommunityToolkit.Mvvm の採用 ＆ Dispatcher 同期実装 (MED-08 解決)
Boilerplate コードを排除し、安全性と保守性を向上させるため `CommunityToolkit.Mvvm` を統一採用する。
* `[ObservableProperty]` によるプロパティ変更通知（`INotifyPropertyChanged`）の自動生成
* `[RelayCommand]` / `[AsyncRelayCommand]` による非同期コマンド生成
* `ObservableObject` を ViewModel の基底クラスとして利用

```csharp
namespace GameSecurityTool.Presentation.ViewModels;

using System;
using System.Collections.ObjectModel;
using System.Threading;
using System.Threading.Tasks;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;

public sealed partial class TimelineViewModel(
    ITimelineQueryService timelineService,
    IDispatcherService dispatcherService) : ObservableObject
{
    [ObservableProperty]
    private bool _isLoading;

    public ObservableCollection<TimelineItemDto> TimelineEntries { get; } = [];

    [RelayCommand]
    private async Task LoadTimelineAsync(TimelineFilterDto filter, CancellationToken cancellationToken)
    {
        IsLoading = true;
        try
        {
            var pagination = new PaginationParams(PageNumber: 1, PageSize: 50);
            var result = await timelineService.GetTimelineAsync(filter, pagination, cancellationToken);

            // 【UI Thread 同期保証 (MED-08 解決)】
            // バックグラウンドスレッドからの更新を IDispatcherService で UI スレッドへ同期
            dispatcherService.Invoke(() =>
            {
                TimelineEntries.Clear();
                foreach (var item in result.Items)
                {
                    TimelineEntries.Add(item);
                }
            });
        }
        finally
        {
            IsLoading = false;
        }
    }
}
```

## 13.5 UI Thread Boundary (Dispatcher 境界)
バックグラウンドワーカーや非同期スレッドから ViewModel のプロパティや UI コレクションを直接更新することを禁止する。
* 必ず `IDispatcherService`（Port）を経由し、WPF の `DispatcherQueue` / `Dispatcher` 上で安全に UI 同期・プロパティ更新を実行する。

## 13.7 Dispatcher Cancellation Contract

`IDispatcherService.InvokeAsync` accepts an optional `CancellationToken`. The token has a deliberately narrow meaning: it cancels the dispatcher operation while the delegate is still queued and has not begun execution.

- A pre-canceled token must prevent delegate execution.
- For a cross-thread dispatch, WPF `Dispatcher.InvokeAsync` receives the token so a queued operation can be canceled before it starts.
- Once the delegate has begun execution, the dispatcher contract does not implicitly cancel the delegate's own asynchronous work. Delegate-internal cancellation remains the responsibility of the delegate's existing asynchronous dependencies and their own cancellation tokens.
- `InvokeAsync` does not change `Invoke(Action)` and does not introduce an adapter-specific overload.

This is a synchronization/cancellation contract at the Presentation UI-thread boundary. It does not alter Phase 0/Implementation Readiness gating.

## 13.6 WIPER State Projection and Session-Bound Commands
WIPERのPresentation stateは `MainWindowViewModel` が保持し、`WiperBreakerTriggeredEventDto` / `WiperChildProcessControlResultDto` をUI向け状態へ投影する。WIPER操作は `IWiperDefenseCoordinator` をApplication境界の窓口として使用し、View/ViewModelからInfrastructure adapterを直接呼び出してはならない。

- Breaker alert stateは表示可否、PID、ProcessName、Reason、TriggeredAt、AffectedPaths、操作中状態、操作結果メッセージを保持する。
- `WiperBreakerTriggeredEventDto` はPIDと `OffendingProcessStartTimeUtc` をaction bindingとして持ち、再開/終了コマンドは両方をApplicationへ渡す。
- Applicationは現在の保護セッションのPID＋StartTimeを再確認し、不一致の操作を拒否する。Infrastructureも同じStartTimeを再確認する。
- background eventは `IDispatcherService` 経由でのみViewModel状態へ反映する。
- 新しいalertが旧alertを置き換えた後、旧コマンドが完了しても旧結果で現在のalertを隠蔽・上書きしてはならない。
- ViewModelはDispose時にWIPERイベント購読を解除する。


---

# 51. Timeline View Baseline

Timeline View は、GST が検知・保護・実行した全セキュリティイベントを時系列で俯瞰・分析するための UI コンポーネントである。

## 51.1 アーキテクチャフロー
```text
[ TimelineView (XAML) ]
           │
           ▼
[ TimelineViewModel ]
           │
           ▼
[ ITimelineQueryService (Contracts Port) ] ──> [ TimelineQueryUseCase (Application) ]
                                                            │
                                                            ▼
                                                [ IAuditRepository (Port) ]
                                                            │
                                                            ▼
                                                [ SqliteAuditRepository (Infra) ] ──> SQLite (PRAGMA WAL)
```

## 51.2 大容量レコード表示とパフォーマンス要件
長期間の運用により監査レコードが数十万件に達することを前提とし、以下の要件を必須とする：
* **全件取得の禁止:** `SELECT * FROM AuditRecords;` による一括ロードを厳禁とする。
* **Pagination (ページネーション):** ページ単位（**デフォルト 50件**）での遅延読み込み。`pageSize` は明示指定可能だが、Timeline Presentation の標準既定値は 50 とする。
* **Date Range Filter (日付範囲絞り込み):** 直近 24時間 / 7日間 / 30日間 / カスタム範囲のインデックス付きクエリ。
* **Virtualization (UI仮想化):** WPF `VirtualizingStackPanel` による UI コントロールの仮想化レンダリング。

## 51.3 Application 境界コントラクト
```csharp
namespace GameSecurityTool.Contracts.Dtos;

using System;
using GameSecurityTool.Contracts.Common;

public sealed record TimelineFilterDto(
    DateTime? StartDateUtc,
    DateTime? EndDateUtc,
    string? GameProfileId,
    string? EventCategory,
    string? SearchKeyword);

public sealed record PaginationParams(
    int PageNumber,
    int PageSize);

public sealed record TimelineItemDto(
    long EventId,
    string OperationId,
    DateTime TimestampUtc,
    string EventType,
    string Severity,
    string Title,
    string Description,
    string? AssociatedPath,
    bool HasSnapshot);
```

## 51.4 Database Independence
Timeline UI は SQLite のテーブル名、列名、暗号化ハッシュチェーン内部構造を一切意識しない。すべてのデータは Contracts の DTO を経由して完全に抽象化された状態で受け取る。

---

End of Document
```

---
