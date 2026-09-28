# 01-05: Presentation and Dialogs

**Document ID:** GST-SPEC-WEBLINK-001-PART5  
**Parent Document:** Web Link Protection Specification v2.1  
**Category:** Presentation & UX Baseline  
**Status:** Approved Baseline Candidate  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Presentation Layer の責務

Presentation Layer は、Web Link Protection における**ユーザー確認ダイアログ（Browser Picker / Confirm Prompt）、遮断通知ダイアログ（Block Notification）、およびホストルール管理画面**を担当する。

### 規約:
* **Infrastructure 直接参照の禁止:** View / ViewModel から OS API、レジストリ、DB を直接呼び出すことを禁止する。
* **Privacy-Safe Display の強制:** ダイアログや設定画面に生の Query String や UserInfo を含む未加工 URL をそのまま描画することを厳禁とする。
* **MVVM 疎結合調停:** ViewModel は View のインスタンスを保持・操作せず、`TaskCompletionSource` または `IDialogService` を介して非同期にダイアログ結果を受け渡す。

---

# 2. Privacy-Safe Display & 偽装ドメイン警告規約

ダイアログに表示される URL は、視認性・プライバシー・セキュリティを両立するために以下のフォーマットを適用する。

```text
[ 元の入力 URL (悪意あるホモグラフ偽装例) ]
https://admin:pass123@xn--pple-43d.com/login?token=secret123&utm=game#frag

[ UI 表示用フォーマット (Privacy-Safe & Warning) ]
┌────────────────────────────────────────────────────────┐
│ ⚠️ 警告: 国際化ドメイン(Punycode)による偽装の可能性   │
│ 接続先ホスト: xn--pple-43d.com (apple.com に酷似)       │
│ パス:         /login                                   │
│ 安全化処置:   URL Queryを既定ポリシーに従って破棄しました         │
└────────────────────────────────────────────────────────┘
```

1. **ホストの強調:** ドメイン偽装を看破できるよう、`Host` 部分を太字・独立表示する。
2. **Punycode / IDN 警告の明示:** `IsPunycodeSuspicious == true` の場合、黄色/赤色の警告バナーを強制表示し、Punycode 文字列を明示する。
3. **Query String の完全非表示:** `?` 以降の値やキーは画面上に描画しない。代わりに、Queryを破棄した事実と理由を安全なメタデータで説明する。正常リンクが機能しない場合は、既存のTrusted-host Query Allowlistへ進む回復導線を提示できる。
4. **UserInfo の完全除去:** `user:password@` 形式の認証情報は表示前に破棄する。
5. **長大パスの切り詰め:** 画面崩れを防ぐため、30文字を超えるパスは末尾を `...` で省略する。

---

# 3. UX / ダイアログ設計

## 3.1 WebLinkConfirmDialog (Confirm 時の確認 & Browser Picker)
ポリシー判定結果が `Confirm` の場合、または起動ブラウザの再選択時にポップアップ表示される。

### ダイアログ表示仕様:
```text
┌────────────────────────────────────────────────────────────┐
│ 🌐 ゲームからWebリンクが開かれようとしています            │
├────────────────────────────────────────────────────────────┤
│ 接続先:  example.com (コミュニティリンク)                  │
│                                                            │
│ 🛡️ 安全化処置:                                            │
│ ・Queryは既定のPrivacy/Security policyに従って破棄されました。│
│                                                            │
│ [⚠️ 国際化ドメイン(Punycode)による偽装の可能性があります]   │
│                                                            │
│ 起動するブラウザを選択してください:                        │
│ ┌────────────────────────────────────────────────────────┐ │
│ │ (●) Google Chrome (InPrivate / 匿名モード)             │ │
│ │ ( ) Microsoft Edge                                     │ │
│ │ ( ) Mozilla Firefox                                    │ │
│ └────────────────────────────────────────────────────────┘ │
│                                                            │
│ [ ] このゲームでは次回からこのブラウザで開く               │
│                                                            │
│      [ 開く (Enter) ]        [ 拒否して閉じる (Esc) ]      │
└────────────────────────────────────────────────────────────┘
```

## 3.2 WebLinkBlockDialog (Block 時の遮断通知)
ポリシー判定結果が `Block` の場合に表示され、なぜ開けなかったのかの根拠を提示する。

### ダイアログ表示仕様:
```text
┌────────────────────────────────────────────────────────────┐
│ 🚫 リンクの起動がブロックされました                        │
├────────────────────────────────────────────────────────────┤
│ 接続先:  discord.com                                       │
│                                                            │
│ 理由:                                                      │
│ 現在のゲームプロファイル設定により、SNS への自動遷移は     │
│ 遮断されています。                                         │
│                                                            │
│ 次の操作:                                                  │
│ ゲーム設定の「Webリンク保護」から許可リストへ追加できます。│
│                                                            │
│                     [ 閉じる (Esc) ]                       │
└────────────────────────────────────────────────────────────┘
```

---

# 4. ViewModels 実装設計

## 4.1 WebLinkPromptViewModel (疎結合ダイアログ調停)
`TaskCompletionSource<BrowserPickerPromptResultDto>` を保持し、ユーザー操作結果を非同期で呼び出し元へ返却する。

```csharp
namespace GameSecurityTool.Presentation.Features.WebLinkProtection.ViewModels;

public sealed partial class WebLinkPromptViewModel : ObservableObject
{
    private TaskCompletionSource<BrowserPickerPromptResultDto>? _tcs;

    [ObservableProperty]
    private string _displayHost = string.Empty;

    [ObservableProperty]
    private string _maskedPath = string.Empty;

    [ObservableProperty]
    private string _categoryDisplayName = string.Empty;

    [ObservableProperty]
    private bool _isPunycodeSuspicious;

    [ObservableProperty]
    private ObservableCollection<BrowserOptionDto> _availableBrowsers = [];

    [ObservableProperty]
    private BrowserOptionDto? _selectedBrowser;

    [ObservableProperty]
    private bool _rememberChoiceForProfile;

    public Task<BrowserPickerPromptResultDto> WaitForUserChoiceAsync(
        UrlPreviewDto preview,
        IReadOnlyList<BrowserOptionDto> browsers,
        CancellationToken cancellationToken)
    {
        _tcs = new TaskCompletionSource<BrowserPickerPromptResultDto>();

        DisplayHost = preview.DisplayHost;
        MaskedPath = preview.MaskedPath;
        CategoryDisplayName = preview.Category.ToString();
        IsPunycodeSuspicious = preview.IsPunycodeSuspicious;
        AvailableBrowsers = [..browsers];
        SelectedBrowser = AvailableBrowsers.FirstOrDefault();
        RememberChoiceForProfile = false;

        cancellationToken.Register(() => _tcs.TrySetCanceled(cancellationToken));
        return _tcs.Task;
    }

    [RelayCommand]
    private void Confirm()
    {
        if (SelectedBrowser == null) return;

        _tcs?.TrySetResult(new BrowserPickerPromptResultDto(
            IsConfirmed: true,
            SelectedBrowser: SelectedBrowser,
            RememberChoiceForProfile: RememberChoiceForProfile));
    }

    [RelayCommand]
    private void Reject()
    {
        _tcs?.TrySetResult(new BrowserPickerPromptResultDto(
            IsConfirmed: false,
            SelectedBrowser: SelectedBrowser ?? AvailableBrowsers.First(),
            RememberChoiceForProfile: false));
    }
}
```

## 4.2 SocialRuleManagementViewModel (ルール管理画面)
ユーザー定義ホストルールの追加・一覧・削除を管理する。

```csharp
public sealed partial class SocialRuleManagementViewModel(
    ISocialHostRuleRepository ruleRepository,
    IDispatcherService dispatcherService) : ObservableObject
{
    [ObservableProperty]
    private ObservableCollection<SocialHostRuleDto> _rules = [];

    [ObservableProperty]
    private string _inputHost = string.Empty;

    [ObservableProperty]
    private string _inputDisplayName = string.Empty;

    [ObservableProperty]
    private MatchMode _selectedMatchMode = MatchMode.ExactHost;

    [ObservableProperty]
    private PolicyDecision _selectedPolicy = PolicyDecision.Block;

    [RelayCommand]
    private async Task LoadRulesAsync(Guid? gameProfileId, CancellationToken cancellationToken)
    {
        var activeRules = await ruleRepository.GetActiveRulesAsync(gameProfileId, cancellationToken);
        Rules = [..activeRules];
    }

    [RelayCommand]
    private async Task AddRuleAsync(Guid? gameProfileId, CancellationToken cancellationToken)
    {
        if (string.IsNullOrWhiteSpace(InputHost)) return;

        var ruleId = Guid.NewGuid();
        var newRule = new SocialHostRuleDto(
            ruleId,
            gameProfileId,
            InputHost.Trim().ToLowerInvariant(),
            string.IsNullOrWhiteSpace(InputDisplayName) ? InputHost : InputDisplayName,
            HostCategory.Social,
            RuleSource.UserDefined,
            SelectedMatchMode,
            SelectedPolicy,
            true,
            DateTimeOffset.UtcNow,
            DateTimeOffset.UtcNow,
            null,
            null);

        await ruleRepository.AddAsync(newRule, cancellationToken);
        await LoadRulesAsync(gameProfileId, cancellationToken);

        InputHost = string.Empty;
        InputDisplayName = string.Empty;
    }

    [RelayCommand]
    private async Task DeleteRuleAsync(Guid ruleId, Guid? gameProfileId, CancellationToken cancellationToken)
    {
        await ruleRepository.DeleteAsync(ruleId, cancellationToken);
        await LoadRulesAsync(gameProfileId, cancellationToken);
    }
}
```
```

---