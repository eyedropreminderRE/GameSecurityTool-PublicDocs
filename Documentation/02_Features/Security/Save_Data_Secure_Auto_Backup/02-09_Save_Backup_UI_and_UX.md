# GameSecurityTool Save Backup UI & UX Specification

**Document ID:** GST-FEAT-SAVE-UI-009  
**Version:** 4.0 (Save Data Hub & Versatile Export Edition)
**Status:** Approved Presentation Specification  
**Target Layer:** Presentation Layer (`GameSecurityTool.Presentation`)  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. UI/UX Design Philosophy

Save Backup のユーザーインターフェースは、**「安全性の可視化」「誤操作の構造的防止」「ゲームプレイへの完全不干渉」** を原則として設計する。

### 必須原則
1. **Explain Before Execute**:
   復元や削除、ストレージ移動など、データを書き換える操作では「何が変更され、どう退避され、どう戻せるか」を必ず事前に明示する。
2. **Zero Game Interruption**:
   フルスクリーンゲーム実行中は通知ポップアップを完全自動抑制（サイレント化）し、画面最小化や FPS 低下を起こさない。
3. **Dispatcher Thread Affinity**:
   バックグラウンドワーカーからの状態通知や進捗更新は、必ず `IDispatcherService` を経由して安全に UI コレクション（`ObservableCollection<T>`）へ同期する。

---

# 2. 画面設計 ＆ ワイヤーフレーム (ASCII Layout)

## 2.1 セーブデータ統合管理ハブ (`SaveDataHubView`)

すべてのゲームのバックアップ状況とストレージ消費を俯瞰し、一括操作および個別画面へのハブとなるメイン画面。

```text
+-----------------------------------------------------------------------------------------+
| 💾 セーブデータ統合管理ハブ (Save Data Hub)                                             |
+-----------------------------------------------------------------------------------------+
| 【全体ストレージ使用量: 4.8 GB / 20.0 GB (空き容量十分 ✅)】                            |
| 5.0GB ┤ ▆▆▆ (Cyberpunk 2077)                                    ← クリックで詳細へ      |
| 2.5GB ┤ ▆▆ (Skyrim SE)                                          ← クリックで詳細へ      |
| 1.0GB ┤ ▆ (ELDEN RING)                                          ← クリックで詳細へ      |
|                                                                                         |
| [ ⚡ 全ゲームの最新状態を一括バックアップ ]   [ 🧹 安全な一括容量お掃除 (古い世代の整理) ] |
+-----------------------------------------------------------------------------------------+
| 【保護中のゲーム一覧】                                                                  |
| 🎮 Cyberpunk 2077       [最終: 20分前] [世代: 3 / 📌2] [3.1 GB]                         |
|    [ ⚡ バックアップ ] [ ⚙️ 詳細・復元 ] [ 🗑️ ゲーム単位のバックアップ全削除 ]             |
| 🎮 Skyrim Special Edition [最終: 昨日]   [世代: 5 / 📌5] [1.2 GB]                         |
|    [ ⚡ バックアップ ] [ ⚙️ 詳細・復元 ] [ 🗑️ ゲーム単位のバックアップ全削除 ]             |
| 🎮 ELDEN RING           [最終: 3日前]  [世代: 1 / 📌0] [0.5 GB]                         |
|    [ ⚡ バックアップ ] [ ⚙️ 詳細・復元 ] [ 🗑️ ゲーム単位のバックアップ全削除 ]             |
+-----------------------------------------------------------------------------------------+
```

## 2.2 ゲーム別詳細 ＆ タイムトラベル管理 (`GameDetailBackupView`)

`SaveDataHubView` のグラフやリストから遷移する、ゲーム個別の管理画面。

```text
+-----------------------------------------------------------------------------------------+
| 🎮 Cyberpunk 2077 - セーブデータ詳細・復元・タイムトラベル                              |
+-----------------------------------------------------------------------------------------+
| [🛡️ 自動バックアップ: 有効 (標準)]   [ 保持設定: 最新5世代 / 5GB ]   [ ⚡ 今すぐバックアップ ]   |
|                                                                                         |
| ・最終バックアップ: 2026/08/27 20:30 (正常完了)                                          |
| ・保存先設定: [ D:\GameBackups\Cyberpunk2077\ ] [ 参照 ]        [ 📦 過去分のお引越し ]  |
| ・ストレージ使用量: 3.1 GB / 5.0 GB (通常: 3世代 / 📌 ピン留め: 2世代)                    |
|                                                                                         |
| [ 📤 指定したセーブデータを汎用 ZIP として外部エクスポート... ]                           |
|                                                                                         |
| 【セーブデータ容量推移グラフ (Save Bloat 監視)】                                        |
| 500MB ┤             ┌──● 420MB (現在: 安定)                                             |
| 400MB ┤       ┌─────┘                                                                   |
| 300MB ├──●────┘ 310MB                                                                   |
+-----------------------------------------------------------------------------------------+
| 【スナップショット履歴】                                                                |
| 保護 | 作成日時         | サイズ | タグ / メモ                      | 操作                 |
|------+------------------+--------+----------------------------------+----------------------|
| 🟢   | 2026/08/27 20:30 | 420 MB | [最新自動セーブ]                 | [↩️ 復元] [🗑️ 削除]  |
| 📌   | 2026/08/27 18:15 | 415 MB | [⚔️ ボス戦直前] [オダ戦前]        | [↩️ 復元] [🏷️ 編集]   |
| 📌   | 2026/08/26 22:00 | 380 MB | [🛠️ MOD導入前] [CyberEngine v2] | [↩️ 復元] [🗑️ 削除]   |
+-----------------------------------------------------------------------------------------+
```

## 2.3 汎用 ZIP エクスポートダイアログ (`SnapshotExportDialog`)

クラウドへの手動退避、別 PC やポータブル機への移動を目的としたエクスポート画面。用途を限定せず汎用的に利用できる。

```text
+-----------------------------------------------------------------------------------------+
| 📤 バックアップの外部エクスポート (汎用 ZIP ファイル出力)                               |
+-----------------------------------------------------------------------------------------+
| 指定したスナップショットを、他のPCや外部ストレージで扱える汎用的な ZIP に書き出します。  |
|                                                                                         |
| 出力モードの選択:                                                                       |
| (●) 🟢 最新のセーブデータのみ (別PCでのプレイ継続・一時退避用)                           |
| ( ) 📌 ピン留めされたセーブのみ (重要なデータだけの厳選保管用)                           |
| ( ) 🗄️ ゲームの全バックアップ履歴 (マニフェストを含む完全なアーカイブ)                    |
|                                                                                         |
| 🔒 セキュリティ設定:                                                                    |
| [✔] パスワードで暗号化する (Argon2id + AES-256)                                         |
|     パスワード: [ ******************** ]                                                |
|     ※ 安全のため、クラウドや外部ドライブにデータを保管する場合は暗号化を強く推奨します。 |
|                                                                                         |
| 出力先: [ C:\Users\***\Desktop\Cyberpunk_Saves.zip                     ] [ 参照 ]       |
|                                                                                         |
|                                     [ 📤 ZIP をエクスポート ]   [ キャンセル ]          |
+-----------------------------------------------------------------------------------------+
```

## 2.4 保存先ストレージ引越しダイアログ (`StorageMigrationDialog`)

設定画面で「新しい保存先」を指定したのち、過去のバックアップをそこへ移送するための画面。

```text
+-----------------------------------------------------------------------------------------+
| 📦 バックアップ保存先ストレージのお引越し                                               |
+-----------------------------------------------------------------------------------------+
| 過去に作成したすべてのバックアップファイルを、新しい保存先ドライブへ安全に移動します。  |
|                                                                                         |
| 現在の保存先: C:\Users\***\AppData\Local\GameSecurityTool\Backups\ (SSD)                |
| 新しい保存先: D:\GameBackups\Cyberpunk2077\                                             |
|                                                                                         |
| ・移動対象ファイル数: 5 件                                                              |
| ・移動データ総容量:   3.1 GB                                                            |
| ・移動先空き容量:     840.5 GB (空き容量十分 ✅)                                        |
|                                                                                         |
| 💡 【安心安全保証】                                                                     |
| コピー完了後に SHA256 ハッシュ突合を行い、完全一致を確認した後にのみ旧ファイルを消去   |
| します。移動中にエラーが発生した場合でもデータは失われません。                          |
|                                                                                         |
|                                                     [ 引越しを開始する ]  [ キャンセル ]|
+-----------------------------------------------------------------------------------------+
```

---

# 3. MVVM 設計 ＆ UI スレッド同期モデル

## 3.1 統合ハブ ViewModel (`SaveDataHubViewModel.cs`)

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

public sealed partial class SaveDataHubViewModel : ObservableObject
{
    private readonly ISaveBackupService _backupService;
    private readonly IDispatcherService _dispatcherService;

    [ObservableProperty]
    private long _totalStorageUsageBytes;

    public ObservableCollection<GameBackupSummaryDto> GameSummaries { get; } = [];

    public SaveDataHubViewModel(ISaveBackupService backupService, IDispatcherService dispatcherService)
    {
        _backupService = backupService;
        _dispatcherService = dispatcherService;
    }

    [RelayCommand]
    private async Task LoadHubDataAsync(CancellationToken ct)
    {
        var summaries = await _backupService.GetAllGameBackupSummariesAsync(ct);
        
        _dispatcherService.Invoke(() =>
        {
            GameSummaries.Clear();
            long total = 0;
            foreach (var s in summaries)
            {
                GameSummaries.Add(s);
                total += s.TotalStorageUsageBytes;
            }
            TotalStorageUsageBytes = total;
        });
    }

    [RelayCommand]
    private async Task BackupAllGamesAsync(CancellationToken ct)
    {
        // 登録されている全ゲームの最新状態を一括バックアップ (直列実行)
        await _backupService.ExecuteBulkBackupAsync(ct);
        await LoadHubDataAsync(ct);
    }
}
```

## 3.2 個別詳細 ViewModel (`GameDetailBackupViewModel.cs` 抜粋)

```csharp
    [RelayCommand]
    private async Task ExportAsZipAsync(ExportMode mode, string destinationPath, string? password, CancellationToken ct)
    {
        var request = new ExportBackupRequestDto(GameProfileId, mode, destinationPath, password);
        await _backupService.ExportToStandardZipAsync(request, ct);
    }

    [RelayCommand]
    private async Task UpdateStorageLocationAsync(string newPath, CancellationToken ct)
    {
        // 単純な保存先設定の変更
        await _backupService.UpdateBackupLocationSettingAsync(GameProfileId, newPath, ct);
        
        // 過去分がある場合は引越しを提案する処理へ繋ぐ
        if (StatusSummary?.TotalSnapshotCount > 0)
        {
            // 引越しダイアログの表示処理...
        }
    }
```

---

# 4. 通知ポリシー ＆ フルスクリーン保護 (`Smart Do Not Disturb`)

ゲーム体験を阻害しないため、通知の表示は OS の画面状態（フルスクリーン判定）と連動させる。

```text
[バックアップ完了 / 容量警告イベント]
                 │
                 ▼
┌────────────────────────────────────────────────────────┐
│ UserPresenceService.IsSafeToShowNotification()         │
│ (SHQueryUserNotificationState Win32 API 照会)          │
└─────────────────────────┬──────────────────────────────┘
                          │
          ┌───────────────┴───────────────┐
  (通常デスクトップ状態)          (D3D フルスクリーンゲーム中)
          │                               │
          ▼                               ▼
  [トースト通知を表示]            [ポップアップを完全自動抑制]
                                  - 画面最小化ゼロ
                                  - トレイアイコンのバッジ点灯のみ
```

---

End of Document
```

---