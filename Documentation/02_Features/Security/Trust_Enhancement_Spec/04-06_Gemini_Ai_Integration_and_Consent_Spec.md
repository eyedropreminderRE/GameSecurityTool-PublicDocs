# 04-06: Gemini AI Integration & Privacy Consent Specification

**Document ID:** GST-FEAT-TRUST-006  
**版:** 4.2
**状態:** 採用確定 (Approved Feature Master Specification)  
**カテゴリ:** Trust Enhancement / Cloud AI Assistant (Opt-in)  
**親文書:** `04-00_Overview_and_Principles.md`  
**対象:** Windows 10 / Windows 11 / .NET 10 / C# 14 / WPF  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 0. Purpose ＆ 基本方針

## 0.1 目的
専門知識のないゲーマーであっても、検知された不審ファイル、ゲームクラッシュの原因、および複雑なセキュリティ設定について、Google Gemini API（BYOK: Bring Your Own Key 方式）を活用して **GST 画面内で直接平易な自然言語解説・即時原因分析・設定相談（AIコンシェルジュ）** を受け取れるようにする。

## 0.2 不可侵セキュリティ ＆ プライバシー原則
1. **完全オプトイン（Default Disabled）:**
   初期状態では外部通信は完全ゼロ（0.0%）。ユーザーが明示的にAI利用同意を有効化し、当該利用時のLiability Guardを承認した場合のみ外部AI送信を許可する。
2. **専用 AI Outbound Sanitization Boundary:**
   Geminiへ到達し得るすべてのテキストは専用のAI outbound sanitization policyを通過する。対象にはユーザー入力、クラッシュレポート、診断詳細、provenance、ゲーム／画面コンテキスト、マニュアル／コンテキスト抜粋、設定スナップショット等を含む。通常ログ用の`ILogSanitizer`と外部ログ取込用の`IExternalLogSanitizer`は、この境界の代替にはならない。
3. **Immutable Prepared AI Request:**
   サニタイズ済みテキスト、payload fingerprint、ConsentRevision、PromptApprovalId、およびLiability Guard承認状態を束ねたimmutable `AiPreparedRequestDto` のみがAI Providerへ渡される。raw `userMessage` その他の未サニタイズ値をProviderが直接受け取る経路は禁止する。
4. **Prompt Previewは送信承認境界:**
   Previewは表示だけでなく、実送信payloadのユーザー承認を表す。Preview後に編集した場合、以前のprepared requestとapprovalは無効化され、編集内容を再サニタイズ・再fingerprint・再承認する。
5. **最終Infrastructure Send Gate:**
   HTTP送信直前に、Infrastructureが準備済みoutbound text fieldsをcanonicalなAI outbound sanitization policyで再処理し、current consent/authorizationを再検証し、stale ConsentRevisionを拒否し、再サニタイズ後のpayload fingerprintと承認済みfingerprintの一致を確認する。いずれかに失敗した場合は新規network requestを開始しない。
6. **API key存在と同意を分離:**
   API keyが保存されていること、AI画面が開いていること、以前に別payloadを承認したことのいずれも、現在のAI送信同意の代用にはならない。
7. **Privacy guaranteeの範囲:**
   GSTはAI outbound sanitization policyで定義された機密パターンの除去と、承認済みsanitized payloadのみが送信境界を越えることを保証する。未知の個人情報を含む任意の自由入力について「絶対に個人情報が送信されない」とは表現しない。
8. **Header-Only Authentication:**
   APIキーをURLクエリへ含めず、`x-goog-api-key` HTTPヘッダーで送信する。
9. **中間サーバー完全ゼロ:**
   ユーザーPCからGoogle公式APIエンドポイントへ直接HTTPS通信し、第三者中継サーバーを介在させない。
10. **APIキー保護:**
   APIキーのbyte[] secret transit、caller ownership、zeroization、DPAPI persistence、および最終transport-boundaryのtextual representationは、文書化された秘密情報管理・送信境界契約を正本とし、この仕様単独では変更しない。
## 0.3 法的表記および商標クレジット (Legal Disclosure)
1. 本ツールは独立したサードパーティ製ソフトウェアであり、Google LLC との提携、後援、または公認を受けた公式製品ではありません。
2. Gemini API のご利用には、Google の利用規約（Google APIs Terms of Service および Gemini API Additional Terms of Service）が適用されます。
3. Google、Gemini、および関連するロゴは Google LLC の商標です。

---

# 1. AI 連携 8 大ユースケース体系

```text
┌────────────────────────────────────────────────────────────────────────┐
│  ユースケース 1: 【未署名 MOD レントゲン診断】                         │
│  - 検知された DLL のインポート/エクスポート・出所シグナルから平易な日本語解説を生成 │
├────────────────────────────────────────────────────────────────────────┤
│  ユースケース 2: 【クラッシュレポート即時原因診断】                    │
│  - WER 例外コード (0xC0000005 等) や直前 MOD 差分から原因と対処法を即答  │
├────────────────────────────────────────────────────────────────────────┤
│  ユースケース 3: 【設定画面 AI コンシェルジュ】                        │
│  - ユーザーが開いている画面のマニュアルと設定 JSON を自動注入してチャット │
├────────────────────────────────────────────────────────────────────────┤
│  ユースケース 4: 【コミュニティ脅威 OSINT 調査】 (新設)                │
│  - 外部コミュニティで報告された悪性 MOD や不審ハッシュの脅威情報を調査  │
├────────────────────────────────────────────────────────────────────────┤
│  ユースケース 5: 【マルチプレイ接続トラブルシュート】 (新設)           │
│  - ゲームの接続障害時に FW ブロック・NAT 状況を診断し推奨ルールを提示   │
├────────────────────────────────────────────────────────────────────────┤
│  ユースケース 6: 【セーブ肥大化・異常診断】 (新設)                     │
│  - セーブ容量急増・シャノンエントロピー急変時の破損や攻撃要因を分析    │
├────────────────────────────────────────────────────────────────────────┤
│  ユースケース 7: 【週間セキュリティ要約】 (新設)                       │
│  - 直近 7 日間の防御イベント、遮断通信、バックアップ実績を要約レポート  │
├────────────────────────────────────────────────────────────────────────┤
│  ユースケース 8: 【MOD 起動不能・前提不足相談】 (新設)                 │
│  - 前提ライブラリ欠落や依存関係エラーの特定と安全な解決手順を案内      │
└────────────────────────────────────────────────────────────────────────┘
```

---

# 2. モデル選択アーキテクチャ (Flash / Pro ＆ カスタムモデル指定)

Google のモデル世代交代や提供終了（EOL）に柔軟に対応するため、既定プリセットに加えて**ユーザーによるカスタムモデル名の自由指定**をサポートする。

## 2.1 モデルの使い分け基準
* **⚡ Flash 系 (既定: `gemini-3.7-flash`):** 高速・軽量。無料枠の制限が緩いため、日常の設定相談や MOD 簡易診断に最適 [1]。
* **🧠 Pro 系 (既定: `gemini-3.1-pro-preview`):** 深い推論力。無料枠の制限が厳しいが、複雑なクラッシュログや MOD 競合の深層分析に最適 [1, 2]。

## 2.2 Domain バリデーション (URL インジェクション防御)
ユーザーが自由入力したモデル名に対し、不正な文字（`/`, `?`, `&` 等）を遮断する厳格なバリデーション（`AiModelNameValidator`）を Domain 層で適用する。

---

# 3. コンテキスト注入パイプライン (Context Injection)

AI コンシェルジュ相談時、ハルシネーションを防ぐため、裏側で以下のコンテキストを自動結合して送信する。

1. **画面連動マニュアル抜粋:** `20_User_Manual_and_Operational_Guide.md` から、現在画面（`ViewContext`）に対応するセクションを抽出。
2. **サニタイズ済み設定 JSON:** `ConfigurationSnapshot` を `ILogSanitizer` で伏字化した状態で添付。

---

# 4. 同意確認 UI ＆ 安心ガイド仕様

## 4.1 API 有効化ダイアログ (`AiConsentDialog`)

```text
+-----------------------------------------------------------------------------------------+
| 🤖 Gemini AI 連携機能の設定 ＆ 同意確認                                                 |
+-----------------------------------------------------------------------------------------+
| 【API キー入力】 [ ************************************************** ] [ 👁️ 表示 ]     |
|                                                                                         |
| 💡 【ご利用前の安心ガイド ＆ 重要注意事項】                                              |
|                                                                                         |
| 🔒 [1. 料金について]                                                                    |
|    Google AI Studio の無料キーを使用している限り、勝手に課金されることは一切ありません。|
|    上限に達した場合は一時的に利用できなくなるだけです。                                 |
|                                                                                         |
| ⚡ [2. モデルの制限と使い分け]                                                           |
|    ・Flash (高速): 制限が緩く普段の相談に最適です。                                     |
|    ・Pro (高精度): 賢いですが無料枠の制限が厳しいため、複雑なクラッシュ分析向きです。   |
|                                                                                         |
| 🛡️ [3. プライバシーと Google 規約]                                                      |
|    Google API 送信直前に、GST が本名や個人パスを自動で伏字 (***) に変換します。          |
|    個人が特定されるデータが送信されることはありません。                                 |
|                                                                                         |
| ⚠️ [4. API キーの管理]                                                                  |
|    キーはパスワードと同じです。他人に教えたり配信に映さないよう大切に保管してください。  |
|                                                                                         |
| [✔] 上記の注意事項、免責事項、および法的表記を理解し、AI 連携を有効化します。           |
|                                                                                         |
|                                     [ 🤖 同意して保存する ]   [ キャンセル ]            |
+-----------------------------------------------------------------------------------------+
```

## 4.2 チャット相談 ＆ クラッシュ分析時の免責同意 (`Liability Guard`)

チャット画面やクラッシュ AI 診断を開いた際、**最初の実行前に** 以下の同意を必須とする。

```text
+-------------------------------------------------------------------------+
| ⚠️ 【注意事項および免責事項】                                           |
| ・AIのアドバイスは公式マニュアルに基づく参考情報です。                  |
| ・AIの提案による設定変更や操作で発生した不具合等は自己責任となります。  |
|                                                                         |
| [✔] 自己責任でAIに相談することに同意します。  [ 診断・相談を開始する ]   |
+-------------------------------------------------------------------------+
```

## 4.3 AI ガードレール UX (Prompt Preview ＆ 免責 ＆ 微調整)
- **送信前プロンプト確認 (Prompt Preview モーダル):** AI 相談や診断の実行直前、クラウドへ送信する最終payloadを表示し、ユーザーが目視確認・編集・承認できるモーダルを提供する。
- **編集後の再処理:** Preview後にユーザーが編集した場合、その編集は新しいraw inputとして扱い、再サニタイズ、payload fingerprint再生成、ConsentRevisionへの再bind、および再承認を必須とする。
- **Approval binding:** 実際の送信payloadのfingerprintがユーザー承認済みfingerprintと一致しない場合、送信を開始しない。
- **最終送信ゲート:** InfrastructureはHTTP送信直前に現在のconsent/authorizationを再確認し、stale authorizationをfail-closedで拒否する。
- **分析限界と免責の常時提示:** 「AIの回答はGSTローカルログ等の提供された情報に基づく参考情報であり、安全性や完全性を保証しません」という注意書きを常時明示する。
- **推奨FWルール適用の安全化:** AIが推奨したFirewallルールを適用する際は、従来どおり自己責任免責ダイアログとポート・プロトコルの手動微調整UIを経由させる。
---


### 4.4 送信承認不変条件

- Prompt Preview は情報表示ではなく外部送信の承認境界とする。
- Preview後にユーザーが編集した場合、以前のPrepared RequestとApprovalは失効し、編集内容を再サニタイズ・再fingerprint・再承認する。
- Prepared Requestには、sanitized text fields、payload fingerprint、ConsentRevision、PromptApprovalId、Liability Guard承認状態を保持する。
- HTTP送信直前にInfrastructure側でcanonical AI outbound sanitizationを再実行したうえで、現在のconsent/authorizationを再確認し、stale consent revisionを拒否する。
- 実送信payloadのfingerprintが承認済みfingerprintと一致しない場合、送信を開始しない。
- 同意撤回後の未送信authorizationは失効し、API keyの存在だけでは送信を許可しない。
- AI outbound sanitizationは「定義済みの機密パターンを除去する」ことを保証するものであり、任意の自由入力に未知の個人情報が存在しないことまで保証するものではない。

# 5. Clean 5-Layer アーキテクチャ責務境界

```text
[ Presentation Layer (GST.Presentation) ]
  - AiConsentDialog.xaml (設定・安心ガイド・法的開示)
  - CommunityCrashReportView.xaml (「AI に原因を相談する」ボタン)
  - ConciergeChatPanel.xaml (免責同意・チャット UI ＆ Prompt Preview)
        │
        ▼ (calls UseCase)
[ Application Layer (GST.Application) ]
  - RequestAiExplanationUseCase (MOD レントゲン解説)
  - AnalyzeCrashReportWithAiUseCase (クラッシュレポート即時診断)
  - RequestAiConciergeUseCase (設定チャット相談)
  - InvestigateOsintThreatUseCase (コミュニティ脅威 OSINT 調査)
  - TroubleshootMultiplayerUseCase (マルチプレイ接続トラブルシュート)
  - DiagnoseSaveBloatUseCase (セーブ肥大化・異常診断)
  - GenerateWeeklySecuritySummaryUseCase (週間セキュリティ要約)
  - ConsultModPrerequisitesUseCase (MOD 起動不能・前提不足相談)
  - AiPromptBuilder (プロンプト構築 ＆ コンテキスト注入)
  - AI Outbound Preparation (全text fieldのsanitization → PreparedAiRequest生成)
        │
        ├─────────────────────────────┐
        ▼ (uses Domain Rules)         ▼ (calls Port Interfaces)
[ Domain Layer (GST.Domain) ]       [ Contracts Layer (GST.Contracts) ]
  - AiModelNameValidator (Pure C#)   - IAiExplanationProvider (Port)
  - ConfigurationSnapshot            - IAiConsentRepository (Port)
                                     - ILogSanitizer (Port)
                                      ▲
                                      │ (implements)
                                    [ Infrastructure Layer (GST.Infrastructure) ]
                                      - GeminiApiClientAdapter (動的エンドポイント ＆ x-goog-api-key)
                                      - DpapiApiKeyStorage (Windows DPAPI)
```

---

# 6. Domain Layer 完全実装 (`GameSecurityTool.Domain.Services`)

```csharp
#nullable enable

namespace GameSecurityTool.Domain.Services;

using System.Text.RegularExpressions;

public static partial class AiModelNameValidator
{
    // 英数字、ハイフン、アンダースコア、ドットのみ (1〜64文字)
    [GeneratedRegex(@"^[a-zA-Z0-9\-\._]{1,64}$", RegexOptions.CultureInvariant)]
    private static partial Regex ValidModelNameRegex();

    public static bool IsValidModelName(string? modelName)
    {
        if (string.IsNullOrWhiteSpace(modelName)) return false;
        return ValidModelNameRegex().IsMatch(modelName.Trim());
    }

    public static string SanitizeOrFallback(string? customModelName, string defaultFallback)
    {
        if (string.IsNullOrWhiteSpace(customModelName) || !IsValidModelName(customModelName))
        {
            return defaultFallback;
        }
        return customModelName.Trim();
    }
}
```

---

# 7. Contracts Layer 定義 (`GameSecurityTool.Contracts`)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;

public interface IAiOutboundSanitizer
{
    Task<AiOutboundSanitizedPayloadDto> SanitizeOutboundPayloadAsync(
        IReadOnlyDictionary<string, string> rawTextFields,
        CancellationToken ct = default);
}

public interface IAiExplanationProvider
{
    bool IsConfigured { get; }

    Task<AiExplanationResultDto> GenerateExplanationAsync(
        AiPreparedRequestDto request,
        CancellationToken ct = default);

    Task<AiExplanationResultDto> SendChatMessageAsync(
        AiPreparedRequestDto request,
        CancellationToken ct = default);
}

public interface IAiConsentRepository
{
    Task<AiConsentStateDto> GetConsentStateAsync(CancellationToken ct = default);
    Task SaveConsentAndApiKeyAsync(bool isConsented, byte[] apiKey, string flashModelName, string proModelName, string appVersion, CancellationToken ct = default);
    Task RevokeConsentAndClearKeyAsync(CancellationToken ct = default);
}
```

```csharp
namespace GameSecurityTool.Contracts.Dtos;

using System;

public sealed record AiOutboundSanitizedPayloadDto(
    System.Collections.Immutable.ImmutableDictionary<string, string> SanitizedTextFields,
    string PayloadFingerprintSha256
);

public sealed record AiPreparedRequestDto(
    AiOutboundSanitizedPayloadDto Payload,
    string ModelName,
    Guid ConsentRevisionId,
    Guid PromptApprovalId,
    bool LiabilityGuardApproved
);

public sealed record AiConsentStateDto(
    bool IsConsented,
    DateTimeOffset? ConsentedAtUtc,
    string? ConsentedAppVersion,
    Guid ConsentRevisionId,
    bool HasApiKeyConfigured,
    string FlashModelName,
    string ProModelName
);

public sealed record AiExplanationResultDto(
    bool Success,
    string MarkdownExplanation,
    string ModelName,
    string? ErrorMessage,
    bool FallbackToLocal
);
```

---

# 8. Application Layer 完全実装（設計参照契約）

Applicationは、AIに送る全text fieldを明示的に列挙し、専用`IAiOutboundSanitizer`へ渡す。

sanitization完了後、Applicationは次の情報を束ねた`AiPreparedRequestDto`を生成する。
- sanitized text fields
- payload fingerprint
- current ConsentRevision
- PromptApprovalId
- Liability Guard approval

Prompt Preview後のユーザー編集は既存のPrepared Requestを必ず失効させる。編集後の値を再sanitization → re-fingerprint → approvalしない限り、Provider呼出しへ進んではならない。

Application use caseはraw `userMessage`、raw crash report、raw path、raw configuration JSON等を`IAiExplanationProvider`へ直接渡してはならない。
# 9. Infrastructure Layer 完全実装（設計参照契約）

`GeminiApiClientAdapter`は`IAiExplanationProvider`を実装し、`AiPreparedRequestDto`のみを受け取る。

HTTP送信直前の必須順序:
1. current consent stateを取得する。
2. requestのConsentRevisionがcurrent stateと一致することを確認する。
3. Liability Guard approvalとPromptApprovalIdの存在を確認する。
4. prepared payloadの全text fieldをcanonical AI outbound sanitizationで再処理し、その結果をcanonical orderで再fingerprintしてrequest fingerprintと一致することを確認する。
5. API keyを文書化されたcaller-owned/zeroization契約に従って取得する。
6. `x-goog-api-key` headerをtransport boundary内だけで生成する。
7. 上記すべてのgateが成功した場合のみHTTP requestを開始する。

いずれかのgateが失敗した場合はfail-closedで終了し、新規network requestを開始しない。

Consent撤回とsend gateは同じconsent revisionを共有する。撤回によって既存authorizationはstaleとなり、未送信のrequestは拒否される。

この節は実装開始を意味せず、Phase 0で確定した実装契約を示す参照仕様である。
# 10. 監査ログ規約 (`AuditEventType`)

- `AiConsentGranted = 64`: Gemini API 連携の事前説明確認 ＆ ユーザー同意完了
- `AiConsentRevoked = 65`: 同意撤回 ＆ API キーの完全消去完了
- `AiExplanationRequested = 66`: 伏字化プロンプトによる AI 解説生成完了

---

## 10.1 必須回帰テスト

1. raw/unsanitized `userMessage`をAI Providerへ渡す経路が存在しない。
2. outbound text fieldのいずれかが未sanitizedの場合、Prepared Requestを生成できず、最終send gateでも再sanitization後のfingerprint不一致として拒否される。
3. Default OFF / unconsented stateではHTTP `SendAsync`が呼ばれない。
4. API keyが存在するだけでconsentが無効な場合、HTTP `SendAsync`が呼ばれない。
5. Consent revisionがstaleまたはrevokedの場合、HTTP `SendAsync`が呼ばれない。
6. Liability Guard未承認の場合、HTTP `SendAsync`が呼ばれない。
7. Prompt Preview後にpayloadを編集した場合、旧approvalが無効となり、再sanitization / re-fingerprint / re-approvalが必要になる。
8. 実送信payloadのfingerprintが承認済みfingerprintと不一致の場合、HTTP `SendAsync`が呼ばれない。
9. `ILogSanitizer`または`IExternalLogSanitizer`だけを通過した値ではAI送信authorizationを得られない。
10. approved sanitized payload、valid consent revision、Liability Guard approval、およびmatching fingerprintがそろった場合のみ通常のHTTP送信経路へ進む。
# 11. 単体テスト仕様 (`GST.UnitTests.Trust.AiConsent`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.Trust.AiConsent;

using GameSecurityTool.Application.Features.TrustEnhancement;
using GameSecurityTool.Domain.Services;
using GameSecurityTool.Infrastructure.Logging;
using Xunit;

public class AiConsentAndModelTests
{
    [Theory]
    [InlineData("gemini-2.5-flash", true)]
    [InlineData("gemini-3.7-flash", true)]
    [InlineData("gemini-3.1-pro-preview", true)]
    [InlineData("invalid/model?name=1", false)] // インジェクション遮断
    [InlineData("", false)]
    public void AiModelNameValidator_ValidatesModelNamesCorrectly(string input, bool expected)
    {
        bool actual = AiModelNameValidator.IsValidModelName(input);
        Assert.Equal(expected, actual);
    }

    [Fact]
    public void BuildCrashAnalysisPrompt_AssemblesCorrectStructure()
    {
        var sanitizer = new PrivacyLogSanitizer();
        var builder = new AiPromptBuilder(sanitizer);

        string report = "### クラッシュレポート\n例外: 0xC0000005\nモジュール: dxgi.dll";
        string prompt = builder.BuildCrashAnalysisPrompt(report);

        Assert.Contains("0xC0000005", prompt);
        Assert.Contains("dxgi.dll", prompt);
        Assert.Contains("トラブルシューティング", prompt);
    }
}
```

---
