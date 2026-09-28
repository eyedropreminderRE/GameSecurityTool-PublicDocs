# GameSecurityTool インゲーム・オーバーレイ HUD 仕様書

**文書ID:** GST-FEAT-OVERLAY-001  
**版:** 3.1 (Dynamic Process Tracking, Resume Re-probe & Resilience Edition)
**状態:** 採用確定 (Approved Feature Specification)  
**カテゴリ:** Security / Gamer UX  
**親文書:** `00_Formal_Baseline_Overview.md` / `Advanced_User_Protection_Master_Spec.md`  
**対象:** Windows 10 / Windows 11 / .NET 10 / C# 14 / WPF  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 0. Purpose ＆ 設計思想

## 0.1 目的
フルスクリーンゲームプレイ中であっても、画面最小化（Alt+Tab）やフォーカス喪失を一切起こさずに、**「リアルタイムの通信ブロック状態確認」「ゲーム内 Web リンクの安全なブラウザ選択（Browser Picker）」「ワンタップ即時バックアップ ＆ ピン留め」** をゲーム画面最前面で行える軽量 HUD（Heads-Up Display）を提供する。

## 0.2 設計原則
1. **Hookless & Non-Invasive (アンチチート安全保証):**  
   DirectX / Vulkan のグラフィックフック（`Present` フック）やゲームプロセスへの DLL 注入を一切行わず、OS レベルの最前面透過ウィンドウ（`Topmost` ＆ `WS_EX_NOACTIVATE`）として描画し、EAC / BattlEye / Vanguard での誤 BAN リスクを極小化する。
2. **Store 非依存 ＆ スタンドアロン動作:**  
   Microsoft Store や Xbox Game Bar SDK に依存せず、通常の Win32 / WPF アプリケーション単体で動作する。
3. **Dual-Trigger ＆ Zero Friction (キーボード ＆ パッド同時待機):**  
   キーボード（`Shift + F12` 等）とゲームパッドのボタン（Xbox ボタン / PS ボタン / Switch Pro ボタン / 背面パドル）の両方を常時同時待機し、設定変更なしでその時手に持っているデバイスから瞬時にオーバーレイを開ける。
4. **Adaptive Button Glyphs (3 大コントローラー配列自動適応):**  
   接続されたデバイス（Xbox / PlayStation / Nintendo Switch Pro）の VID を完全ローカル自動識別し、画面上のボタン案内アイコン（A/B/X/Y, ✕/◯/⬜/△, B/A/Y/X）を最適切り替え表示する。
5. **Dynamic Target Process Tracking (動的フォアグラウンドプロセス追跡):**
   Win32 `GetForegroundWindow` を照会し、オーバーレイ展開時に最前面にあるアクティブなゲームプロセスを動的に特定・バインドする。
6. **Resume ＆ Hot-Plug Controller Resilience (コントローラー再検出レジリエンス):**
   PC スリープ復帰時や USB コントローラー抜き差し時（`WM_DEVICECHANGE`）に HID デバイス列挙を自動再実行し、コントローラー入力を即座に復帰させる。

---

# 1. 画面レイアウト ＆ マルチコントローラー操作体系

## 1.1 インゲーム HUD メイン画面 (`OverlayHudView`)

```text
+--------------------------------------------------------+
| 🛡️ GameSecurityTool HUD              [ (B/◯) 閉じる ]  |
+--------------------------------------------------------+
| 🎮 実行中: Cyberpunk 2077 (保護中: 🟢 正常)            |
|                                                        |
| 【セーブ保護クイック操作】                             |
| ・最終バックアップ: 2分前 (18ファイル / 420MB)         |
|                                                        |
| [ (Y/△/X) ⚡ 今すぐバックアップ ] [ (X/⬜/Y) 📌 ピン留め ]|
+--------------------------------------------------------+
| 【セキュリティ ＆ リアルタイム通信状態】               |
| ・外部通信ブロック: 0 件 (Default Outbound Deny 稼働中)|
| ・読み込み中 MOD:   dxgi.dll (ReShade プロキシ正常 ✅) |
+--------------------------------------------------------+
```

## 1.2 Smart Game IME Lock 状態表示 (`OverlayImeStatusBar`)

```text
+--------------------------------------------------------+
| ⌨ Smart Game IME Lock                                  |
| 状態: 🟢 英数固定中                                    |
| 日本語入力: 🔒 長押しで有効化                         |
| [ IME 誤操作防止を一時停止 ]                            |
+--------------------------------------------------------+
```

### 1.2.1 表示状態

- **英数固定中:** ゲームセッションが保護状態にあり、日本語入力要求を長押しだけ受理する状態。
- **日本語有効:** ユーザーの明示操作により日本語入力を許可している状態。脱出トリガーの説明を必要に応じて表示する。
- **保護一時停止:** ユーザーがHUDから明示的に一時停止した状態。対象は現在のゲームセッションに限定し、Windowsの恒久設定は変更しない。
- `Hidden` バッジ設定時はこの状態バー自体を表示せず、HUDのデータモデルでは状態を保持する。

### 1.2.2 HUDトグルの責務境界

HUDはIME状態の表示と「一時停止」要求の発行のみを担当する。IME制御自体は `ISmartImeCoordinator` 等のContracts Portを介してApplicationへ委譲し、HUDからWin32 APIを直接呼び出さない。

- 一時停止操作では既存のIME入力監視・状態遷移を停止し、進行中の非同期要求がある場合はLifecycle Mutation Gateにより古い結果を破棄する。
- 再開時は現在の前面ゲーム、Thread/PID、対象HKLを再確認してから監視を復帰する。
- HUD操作によってゲーム入力を注入、消費、再送することは禁止する。

## 1.3 ゲーム内 Web リンク承認 ＆ Browser Picker (`OverlayBrowserPickerView`)

```text
+--------------------------------------------------------+
| 🌐 ゲームからWebリンクが開かれようとしています         |
+--------------------------------------------------------+
| 接続先:  discord.com (コミュニティリンク)              |
| 🛡️ 安全化: 追跡パラメータ・セッショントークン削除済み  |
|                                                        |
| 起動するブラウザを選択してください:                    |
| ┌────────────────────────────────────────────────────┐ │
| │ (●) 外部ブラウザ (InPrivate / 匿名モード)          │ │
| │ ( ) 既定のブラウザ                                 │ │
| │ ( ) サブブラウザ                                   │ │
| └────────────────────────────────────────────────────┘ │
|                                                        |
|    [ (A/✕/A) 🌐 開く (Enter) ]   [ (B/◯/B) 🚫 拒否 ]   |
+--------------------------------------------------------+
```

---

# 2. 3大コントローラー配列マッピング ＆ VID 自動判別仕様

Windows 標準の Raw Input / HID API を使用し、接続されたコントローラーの **Vendor ID (VID)** を完全ローカルで自動判定する。

```text
[ コントローラー接続検知 (USB / Bluetooth) ]
                       │
                       ▼
┌────────────────────────────────────────────────────────┐
│      DeviceIdentificationAdapter (Infrastructure)      │
│  - Vendor ID (VID) の自動照合:                         │
│    ・Microsoft (Xbox)         ➔ VID: 0x045E            │
│    ・Sony (PlayStation DS4/5) ➔ VID: 0x054C            │
│    ・Nintendo (Switch Pro)    ➔ VID: 0x057E            │
└──────────────────────┬─────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────┐
│         OverlayHudViewModel (Presentation)             │
│  - ActiveGlyphSet を切り替え (UI ガイダンス更新)        │
└────────────────────────────────────────────────────────┘
```

### 2.1 コントローラー別ボタン割り当て対照表

| 操作アクション | ① Xbox コントローラー<br>(VID: `0x045E`) | ② PlayStation (DualSense/DS4)<br>(VID: `0x054C`) | ③ Nintendo Switch Pro<br>(VID: `0x057E`) | キーボード / マウス |
| :--- | :---: | :---: | :---: | :---: |
| **決定 / ブラウザを開く** | **`(A)`** (下) | **`(✕)`** (下) / *(◯:日本式)* | **`(A)`** (右) | `Enter` / 左クリック |
| **閉じる / 拒否** | **`(B)`** (右) | **`(◯)`** (右) / *(✕:日本式)* | **`(B)`** (下) | `Esc` / [閉じる] |
| **📌 直前セーブにピン留め** | **`(X)`** (左) | **`(⬜)`** (左) | **`(Y)`** (左) | `P` キー |
| **⚡ 今すぐバックアップ** | **`(Y)`** (上) | **`(△)`** (上) | **`(X)`** (上) | `B` キー |
| **フォーカス移動** | 左スティック / 十字キー | 左スティック / 方向キー | 左スティック / 十字ボタン | 矢印キー / マウス移動 |

※ 設定画面で `ButtonGlyphStyle: [ 自動判別 (既定) | Xbox | PlayStation | Nintendo ]` の手動固定および、PS 用の「◯決定 / ✕戻る」反転トグルを提供。

---

# 3. Dual-Trigger (デュアル・トリガー) 入力待機設計

ゲームによって操作デバイスを使い分けるゲーマーのため、設定切り替えなしで両方の入力からオーバーレイを起動可能とする。

```text
┌────────────────────────────────────────────────────────┐
│             GST Overlay Trigger Coordinator            │
├──────────────────────────┬─────────────────────────────┤
│  [ キーボード待機ルート ] │      [ ゲームパッド待機ルート ]│
│  Win32 RegisterHotKey    │      Raw Input / HID リスナー│
│  (既定: Shift + F12)     │      (Xbox / PS / Switch Pro)│
└─────────────┬────────────┴──────────────┬──────────────┘
              │                           │
              └─────────────┬─────────────┘
                            ▼
      【どちらが押されても即座にオーバーレイを開く！】
```

- **CPU 負荷 0.0% (イベント駆動):** ポーリング（常時監視ループ）を行わず、OS の入力メッセージ駆動で待機。
- **ゲーム操作との非干渉:** オーバーレイが非表示の間はコントローラー入力をゲームへ完全にパススルー。オーバーレイ表示中のみナビゲーションキーを処理。

## 3.1 動的フォアグラウンドプロセス追跡 (Dynamic Foreground Process Tracking)
- **課題:** ユーザーがマルチウィンドウゲームや複数ゲームを同時起動している場合、または Alt+Tab でウィンドウを切り替えた場合、HUD が過去の固定ターゲットに結びついていると誤ったゲームのステータスを表示するリスクがある。
- **仕様:**
  1. ホットキーまたはパッドボタン押下時に Win32 `GetForegroundWindow` および `GetWindowThreadProcessId` を呼び出し、最前面ウィンドウのプロセス ID を動的に取得。
  2. 登録済み `GameProfile` のプロセスと照合し、一致した場合は当該ゲームのステータス（通信遮断数、MOD 読み込み状態、最新セーブ情報）を HUD に動的バインド。
  3. 未登録ゲームの場合は「未管理ゲーム検出」バッジを表示し、ワンクリックで GST 保護下への途中吸収（`GameIngestionDialog`）を促す。

## 3.2 スリープ復帰 ＆ ホットプラグ時のコントローラー自動再プローブ (Resume & Hot-Plug Re-probe)
- **課題:** PC がスリープや休止状態から復帰した際、または USB / Bluetooth コントローラーが再接続された際に、Raw Input ハンドルが無効化してコントローラー入力が効かなくなる問題。
- **仕様:**
  1. `WM_DEVICECHANGE`（`DBT_DEVICEARRIVAL` / `DBT_DEVICEREMOVECOMPLETE`）および OS 電源復帰イベント（`PBT_APMRESUMEAUTOMATIC`）を監視。
  2. イベント検知時に `HidGamepadDeviceDetector.ReenumerateDevices()` をトリガーし、Raw Input HID デバイス一覧を即座に再取得。
  3. 接続されたゲームパッドの VID/PID を再照合し、HUD 画面上のグリフ表示（A/B/X/Y, ✕/◯/⬜/△）を瞬時に最新状態へ自動同期する。

---

# 4. Clean 5-Layer アーキテクチャ責務境界

```text
[ Presentation Layer (GST.Presentation.Overlay) ]
  - OverlayHudWindow.xaml (Mica/Acrylic 半透明 Topmost ウィンドウ)
  - OverlayHudViewModel.cs (CommunityToolkit.Mvvm / Adaptive Glyphs バインディング)
  - OverlayBrowserPickerViewModel.cs
        │
        ▼ (calls UseCases)
[ Application Layer (GST.Application) ]
  - OverlayCoordinatorUseCase (表示状態 ＆ WebLink プロンプト調停)
  - QuickBackupUseCase / PinLatestSnapshotUseCase
        │
        ├─────────────────────────────┐
        ▼ (uses Domain Rules)         ▼ (calls Port Interfaces)
[ Domain Layer (GST.Domain) ]       [ Contracts Layer (GST.Contracts) ]
  - GamepadType (Enum)               - IOverlayManager (Port)
  - GamepadGlyphMapping (Model)      - IGamepadDeviceDetector (Port)
  - OverlayDisplayMode (Enum)        - IGlobalHotkeyService (Port)
                                     - OverlayStatusDto (DTO)
                                      ▲
                                      │ (implements)
                                    [ Infrastructure Layer (GST.Infrastructure) ]
                                      - Win32GlobalHotkeyAdapter (RegisterHotKey)
                                      - HidGamepadDeviceDetector (VID/PID 判定)
                                      - Win32WindowControlAdapter (WS_EX_NOACTIVATE)
```

---

# 5. Domain Layer 定義 (`GameSecurityTool.Domain`)

```csharp
namespace GameSecurityTool.Domain.Models.Overlay;

public enum GamepadType
{
    Unknown = 0,
    Xbox = 1,
    PlayStation = 2,
    NintendoSwitchPro = 3
}

public enum GamepadGlyphStyle
{
    AutoDetect = 0,
    Xbox = 1,
    PlayStation = 2,
    Nintendo = 3
}

public sealed record ControllerGlyphSet(
    string ConfirmGlyph,
    string CancelGlyph,
    string PinGlyph,
    string BackupGlyph
);
```

## 5.1 グリフ解決ドメインサービス (`GamepadGlyphResolver.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Domain.Services;

using GameSecurityTool.Domain.Models.Overlay;

public static class GamepadGlyphResolver
{
    public static ControllerGlyphSet ResolveGlyphSet(
        GamepadType detectedType,
        GamepadGlyphStyle userPreference,
        bool psClassicJapanLayout = false)
    {
        var effectiveType = userPreference switch
        {
            GamepadGlyphStyle.Xbox => GamepadType.Xbox,
            GamepadGlyphStyle.PlayStation => GamepadType.PlayStation,
            GamepadGlyphStyle.Nintendo => GamepadType.NintendoSwitchPro,
            _ => detectedType != GamepadType.Unknown ? detectedType : GamepadType.Xbox
        };

        return effectiveType switch
        {
            GamepadType.PlayStation => psClassicJapanLayout
                ? new ControllerGlyphSet("◯", "✕", "⬜", "△")
                : new ControllerGlyphSet("✕", "◯", "⬜", "△"),

            GamepadType.NintendoSwitchPro => new ControllerGlyphSet("A", "B", "Y", "X"),

            GamepadType.Xbox or _ => new ControllerGlyphSet("A", "B", "X", "Y")
        };
    }
}
```

---

# 6. Contracts Layer 定義 (`GameSecurityTool.Contracts`)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;

public interface IOverlayManager
{
    void ShowHud();
    void HideHud();
    void ToggleHud();
    Task<BrowserPickerPromptResultDto> PromptBrowserPickerInOverlayAsync(UrlPreviewDto preview, CancellationToken ct = default);
}

public interface IGamepadDeviceDetector : IDisposable
{
    event Action<GamepadDeviceInfoDto>? GamepadConnected;
    GamepadDeviceInfoDto GetCurrentActiveGamepad();
    void ReenumerateDevices();
}

public interface IGlobalHotkeyService : IDisposable
{
    event Action? HotkeyPressed;
    bool RegisterKeyboardHotkey(uint modifiers, uint virtualKey);
    bool RegisterGamepadBinding(uint buttonMask);
    void UnregisterAll();
}
```

```csharp
namespace GameSecurityTool.Contracts.Dtos;

using System;
using GameSecurityTool.Domain.Models.Overlay;

public sealed record GamepadDeviceInfoDto(
    GamepadType DeviceType,
    int VendorId,
    int ProductId,
    string DeviceDisplayName
);

public sealed record OverlayStatusDto(
    bool IsGameRunning,
    string? RunningGameName,
    DateTimeOffset? LastBackupTimeUtc,
    int LastBackupFileCount,
    long LastBackupSizeBytes,
    int BlockedOutboundCount,
    string? LoadedProxyDllName,
    bool IsProxyDllVerified
);
```

---

# 7. Infrastructure Layer 実装 (`GameSecurityTool.Infrastructure`)

## 7.1 HID コントローラー VID 判別 Adapter (`HidGamepadDeviceDetector.cs` - L2 是正)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Native;

using System;
using System.Runtime.InteropServices;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Domain.Models.Overlay;
using Microsoft.Extensions.Logging;

public sealed partial class HidGamepadDeviceDetector : IGamepadDeviceDetector
{
    private const int VidMicrosoft = 0x045E;
    private const int VidSony = 0x054C;
    private const int VidNintendo = 0x057E;

    private const uint RIM_TYPEHID = 2;
    private const uint RIDI_DEVICEINFO = 0x2000000b;

    [StructLayout(LayoutKind.Sequential)]
    private struct RAWINPUTDEVICELIST
    {
        public IntPtr hDevice;
        public uint dwType;
    }

    [StructLayout(LayoutKind.Explicit)]
    private struct RID_DEVICE_INFO
    {
        [FieldOffset(0)] public uint cbSize;
        [FieldOffset(4)] public uint dwType;
        [FieldOffset(8)] public RID_DEVICE_INFO_HID hid;
    }

    [StructLayout(LayoutKind.Sequential)]
    private struct RID_DEVICE_INFO_HID
    {
        public uint dwVendorId;
        public uint dwProductId;
        public uint dwVersionNumber;
        public ushort usUsagePage;
        public ushort usUsage;
    }

    [LibraryImport("user32.dll", SetLastError = true)]
    private static partial uint GetRawInputDeviceList(
        [Out] RAWINPUTDEVICELIST[]? pRawInputDeviceList,
        ref uint puiNumDevices,
        uint cbSize);

    [LibraryImport("user32.dll", SetLastError = true)]
    private static partial uint GetRawInputDeviceInfoW(
        IntPtr hDevice,
        uint uiCommand,
        ref RID_DEVICE_INFO pData,
        ref uint pcbSize);

    private readonly ILogger<HidGamepadDeviceDetector> _logger;
    public event Action<GamepadDeviceInfoDto>? GamepadConnected;

    public HidGamepadDeviceDetector(ILogger<HidGamepadDeviceDetector> logger)
    {
        _logger = logger;
    }

    /// <summary>
    /// Win32 Raw Input API を用いて、接続されたゲームパッドの Vendor ID (VID) を動的に完全ローカル判定します。
    /// </summary>
    public GamepadDeviceInfoDto GetCurrentActiveGamepad()
    {
        try
        {
            int detectedVid = DetectConnectedVendorId();

            var (type, name) = detectedVid switch
            {
                VidSony => (GamepadType.PlayStation, "PlayStation DualSense / DualShock"),
                VidNintendo => (GamepadType.NintendoSwitchPro, "Nintendo Switch Pro Controller"),
                VidMicrosoft => (GamepadType.Xbox, "Xbox Wireless / Compatible Controller"),
                _ => (GamepadType.Xbox, "Standard Controller (Xbox Compatible)")
            };

            return new GamepadDeviceInfoDto(type, detectedVid, 0, name);
        }
        catch (Exception ex)
        {
            _logger.LogTrace(ex, "コントローラー自動判別フォールバック (Xbox)");
            return new GamepadDeviceInfoDto(GamepadType.Xbox, VidMicrosoft, 0, "Xbox Controller");
        }
    }

    /// <summary>
    /// 【L2 是正】スタブを排除し、実際に OS から HID デバイスを列挙してゲームパッドの VID を抽出する
    /// </summary>
    private static int DetectConnectedVendorId()
    {
        uint deviceCount = 0;
        uint listStructSize = (uint)Marshal.SizeOf<RAWINPUTDEVICELIST>();
        
        if (GetRawInputDeviceList(null, ref deviceCount, listStructSize) != 0)
        {
            return VidMicrosoft; // 取得失敗時 Fallback
        }

        if (deviceCount == 0) return VidMicrosoft;

        var deviceList = new RAWINPUTDEVICELIST[deviceCount];
        if (GetRawInputDeviceList(deviceList, ref deviceCount, listStructSize) == unchecked((uint)-1))
        {
            return VidMicrosoft;
        }

        foreach (var device in deviceList)
        {
            if (device.dwType == RIM_TYPEHID)
            {
                uint infoSize = (uint)Marshal.SizeOf<RID_DEVICE_INFO>();
                var info = new RID_DEVICE_INFO { cbSize = infoSize };
                
                if (GetRawInputDeviceInfoW(device.hDevice, RIDI_DEVICEINFO, ref info, ref infoSize) != unchecked((uint)-1))
                {
                    // UsagePage 1 (Generic Desktop), Usage 5 (Game Pad) or 4 (Joystick)
                    if (info.hid.usUsagePage == 1 && (info.hid.usUsage == 5 || info.hid.usUsage == 4))
                    {
                        return (int)info.hid.dwVendorId;
                    }
                }
            }
        }

        return VidMicrosoft; // 見つからなかった場合の Fallback
    }

    public void ReenumerateDevices()
    {
        _logger.LogInformation("コントローラーデバイス一覧を再列挙します (WM_DEVICECHANGE / スリープ復帰)...");
        var active = GetCurrentActiveGamepad();
        GamepadConnected?.Invoke(active);
    }

    public void Dispose() { }
}
```

---

# 8. Presentation Layer 実装 (`GameSecurityTool.Presentation.Overlay`)

## 8.1 Overlay ViewModel (`OverlayHudViewModel.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Presentation.ViewModels;

using System;
using System.Threading;
using System.Threading.Tasks;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;
using GameSecurityTool.Domain.Models.Overlay;
using GameSecurityTool.Domain.Services;

public sealed partial class OverlayHudViewModel : ObservableObject
{
    private readonly ISaveBackupService _backupService;
    private readonly IGamepadDeviceDetector _gamepadDetector;
    private readonly IDispatcherService _dispatcherService;

    [ObservableProperty]
    private OverlayStatusDto? _status;

    [ObservableProperty]
    private ControllerGlyphSet _glyphs = GamepadGlyphResolver.ResolveGlyphSet(GamepadType.Xbox, GamepadGlyphStyle.AutoDetect);

    [ObservableProperty]
    private bool _isGamepadMode;

    public OverlayHudViewModel(
        ISaveBackupService backupService,
        IGamepadDeviceDetector gamepadDetector,
        IDispatcherService dispatcherService)
    {
        _backupService = backupService;
        _gamepadDetector = gamepadDetector;
        _dispatcherService = dispatcherService;

        RefreshGlyphs();
        _gamepadDetector.GamepadConnected += _ => RefreshGlyphs();
    }

    private void RefreshGlyphs()
    {
        var device = _gamepadDetector.GetCurrentActiveGamepad();
        _dispatcherService.Invoke(() =>
        {
            Glyphs = GamepadGlyphResolver.ResolveGlyphSet(device.DeviceType, GamepadGlyphStyle.AutoDetect);
        });
    }

    [RelayCommand]
    private async Task QuickBackupAsync(Guid gameProfileId, CancellationToken ct)
    {
        await _backupService.ExecuteManualBackupAsync(gameProfileId, null, ct);
    }
}
```

---

# 9. 単体テスト仕様 (`GST.UnitTests.Overlay.Glyphs`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.Overlay.Glyphs;

using GameSecurityTool.Domain.Models.Overlay;
using GameSecurityTool.Domain.Services;
using Xunit;

public class GamepadGlyphResolverTests
{
    [Fact]
    public void ResolveGlyphSet_WhenNintendoSwitchPro_ResolvesBAYXCorrectly()
    {
        var glyphs = GamepadGlyphResolver.ResolveGlyphSet(
            detectedType: GamepadType.NintendoSwitchPro,
            userPreference: GamepadGlyphStyle.AutoDetect
        );

        Assert.Equal("A", glyphs.ConfirmGlyph); // 右ボタン (決定)
        Assert.Equal("B", glyphs.CancelGlyph);  // 下ボタン (戻る)
        Assert.Equal("Y", glyphs.PinGlyph);     // 左ボタン (ピン留め)
        Assert.Equal("X", glyphs.BackupGlyph);  // 上ボタン (バックアップ)
    }

    [Fact]
    public void ResolveGlyphSet_WhenPlayStationWithClassicJapan_ResolvesCircleConfirm()
    {
        var glyphs = GamepadGlyphResolver.ResolveGlyphSet(
            detectedType: GamepadType.PlayStation,
            userPreference: GamepadGlyphStyle.AutoDetect,
            psClassicJapanLayout: true
        );

        Assert.Equal("◯", glyphs.ConfirmGlyph);
        Assert.Equal("✕", glyphs.CancelGlyph);
    }
}
```

---

# 10. ロードマップ反映

- **MVP Beta (Phase 3):** Win32 ホットキー ＆ 最前面透過 HUD（`InGame_Overlay_HUD`）を配備。
- **v1.0 Core (Phase 4):** Dual-Trigger（キーボード ＆ パッド同時待機）および 3 大コントローラー配列（Xbox / PS / Nintendo）アダプティブ・グリフ切り替え（Raw Input 動的取得）を統合。

---

End of Document

---
