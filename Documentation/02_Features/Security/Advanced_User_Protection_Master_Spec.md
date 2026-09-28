# GameSecurityTool 高度ユーザー保護機能 統合マスター仕様書

**文書ID:** GST-ADV-MASTER-001  
**版:** 3.1 (Mod Provenance, In-Game HUD, Crash Report & Gemini AI Integrated)  
**状態:** 統合マスター仕様 (Approved Master Baseline)  
**対象:** Windows 10 / Windows 11 / .NET 10 / C# 14 / WPF  

> **Lifecycle note:** This specification describes approved integrated feature scope. Approval does not by itself indicate that every integrated capability is implemented, Windows-verified, or released.

---

# 1. 目的

本書は、GameSecurityTool（GST）における高度ユーザー保護機能群を一元統合・調停するためのマスター仕様である。

### 対象機能群：
1. **Web Link Protection** (ゲーム内リンク起動制御・トラッキング除去・多種ブラウザ適合 Browser Picker)
2. **`.LNK` Hijack Detection** (ショートカット改ざん・LotL攻撃検知・非ブロッキング COM 解析)
3. **Save Data Secure Auto-Backup & Time Travel** (非破壊・64KB Chunked AEAD・ピン留め永久保護・ストレージ引越し)
4. **On-Demand Clipboard URL Sanitizer** (ユーザー主導のURL安全化・STA スレッド安全性保証)
5. **Mod Security & Provenance Analysis** (未署名MOD安全性診断・3層ハイブリッド出所追跡)
6. **In-Game Overlay HUD** (Hookless 非干渉最前面描画・デュアルトリガー・3大パッド配列適応)
7. **Privacy-Safe Crash Diagnostics Report** (Discord/Reddit相談用 4大診断ブロック伏字化レポート生成)
8. **Opt-in Gemini AI Integration** (BYOK Gemini API連携・事前説明・プライバシー同意確認)

本書は「各機能が何を保証し、どのレイヤーに属し、どの共通安全基盤を再利用するか」を定義する。

---

# 2. 文書体系と優先順位

## 2.1 製品全体との関係
製品全体の機能一覧、共通ルール、Feature Control Policy などの上位統制は、GST本体のSource of Truth側で管理されます。公開候補の本書は、その上位定義と整合する統合マスター仕様として位置付けます。

## 2.2 本書
- **`Documentation/02_Features/Security/Advanced_User_Protection_Master_Spec.md`**  
  高度ユーザー保護機能群の統合マスター仕様。

## 2.3 個別詳細仕様書への公式リンク一覧
- **Web Link Protection:** [`Web_Link_Protection_Spec/01-00_Overview_and_Scope.md`](Web_Link_Protection_Spec/01-00_Overview_and_Scope.md) (他 01-01〜01-06 全7ファイル)
- **`.LNK` 改ざん検知:** [`LNK_Hijack_Detection_Spec.md`](LNK_Hijack_Detection_Spec.md)
- **Save Backup & Time Travel:** [`Save_Data_Secure_Auto_Backup/02-00_Save_Backup_Index.md`](Save_Data_Secure_Auto_Backup/02-00_Save_Backup_Index.md) (他 02-01〜02-10 全11ファイル)
- **Clipboard Sanitizer:** [`../Privacy/03_OnDemand_Clipboard_URL_Sanitizer_Spec.md`](../Privacy/03_OnDemand_Clipboard_URL_Sanitizer_Spec.md)
- **MOD 安全性診断 ＆ 出所追跡:** [`Mod_Security_and_Provenance_Spec.md`](Mod_Security_and_Provenance_Spec.md)
- **インゲーム・オーバーレイ HUD:** [`InGame_Overlay_HUD_Spec.md`](InGame_Overlay_HUD_Spec.md)
- **クラッシュ相談レポート生成:** [`../Privacy/04_PrivacySafe_Crash_Diagnostics_Report_Spec.md`](../Privacy/04_PrivacySafe_Crash_Diagnostics_Report_Spec.md)
- **Trust Enhancement & Gemini AI:** [`Trust_Enhancement_Spec/04-00_Overview_and_Principles.md`](Trust_Enhancement_Spec/04-00_Overview_and_Principles.md) (他 04-01〜04-06 全7ファイル)
- **機能統合差分仕様:** [`../../01_Architecture/Feature_Integration_Update_v1.0.md`](../../01_Architecture/Feature_Integration_Update_v1.0.md)

### 優先順位の絶対規則:
1. `00_Baseline/` (製品最高権限ベースライン: Tier 0)
2. 製品全体の上位ガバナンス定義
3. 本書 (`Advanced_User_Protection_Master_Spec.md`)
4. 個別詳細仕様書群
5. 実装ソースコード

---

# 3. 共通アーキテクチャ原則

## 3.1 Clean 5-Layer & Hexagonal Architecture
```text
       [ Presentation Layer (GST.Presentation / Overlay) ]
             │               │
             │ (ViewModel)   ▼ (Ports / DTOs)
             │         [ Contracts Layer (GST.Contracts) ]
             ▼               ▲
       [ Application Layer ] ┘ (Implements UseCases)
             │
             ▼
       [ Domain Layer (GST.Domain) ] (Pure C# Business Rules: Contracts参照ゼロ)
             ▲
             │ (Implements Ports)
       [ Infrastructure Layer (GST.Infrastructure) ] (Win32, EF Core, Storage, AI)
```

## 3.2 開発およびセキュリティの禁止事項
- Domain 層から Windows API, Win32 ハンドル, `System.IO`, DB, **Contracts DTO/Port** を直接参照・呼び出さない。
- Application 層から Infrastructure 具象クラス（`WindowsFirewallManager` 等）や `DbContext` を直接インスタンス化・操作しない。
- UI / ViewModel から DB, Firewall, ファイル I/O を直接操作しない（必ず `IDispatcherService` で同期）。
- 常時管理者権限（`requireAdministrator`）での起動を禁止する（Standard User 起動厳守）。
- Kernel Driver, 常駐 SYSTEM サービス, プロセス注入, DirectX フックを導入しない。
- テレメトリ送信、外部クラウドへの自動アップロードを完全排除する（Local First / Privacy First / オプトイン BYOK のみ例外）。

## 3.3 共通追跡識別子
- **`OperationId`**: ユーザー操作・業務ユースケース単位の追跡ID（例: `GST-OP-20260826-A1B2`）。
- **`TransactionId`**: 単一処理の原子性・二段階コミット単位の追跡ID（例: `RST-20260826-55F1`, `QRT-20260826-A82F`）。

---

# 4. 共通 Rule Scope & Feature Control Policy

すべての高度保護機能は、製品共通の **Universal Rule Scope** および **Universal Feature Control Policy** に従って管理・評価される。

```text
RuleScope
├─ Global
│   ├─ Enforced    (全ゲーム強制適用・上書き不可)
│   └─ Overrideable(GameProfile側で上書き可能)
└─ GameProfile     (特定ゲーム専用ルール)

FeatureScope
├─ Global          (機能自体の既定 ON/OFF)
└─ GameProfile     (UseGlobal / Enabled / Disabled)
```

### 競合解決の Fail-Safe 原則:
同一優先度内で Allow と Block が競合した場合、常に **Block（遮断・保護維持）を最優先** とする。

---

# 5. 高度保護機能群の責任境界と保証範囲

```text
                                [ GameSecurityTool ]
                                          │
       ┌──────────────────────────────────┼──────────────────────────────────┐
       ▼                                  ▼                                  ▼
[ Web ＆ オーバーレイ ]           [ セキュリティ ＆ MOD 診断 ]        [ ゲーム保護 ＆ レポート ]
・Managed Launch Path             ・Adaptive Security Pipeline       ・Save Backup & Time Travel
・Strict Query Stripping          ・PE プロキシ構造診断              ・64KB Chunked AEAD
・Multi-Browser Picker            ・Zone.Identifier 出所追跡         ・Restrict 削除制約 ＆ GC
・Hookless インゲーム HUD         ・.LNK Hijack (Non-blocking)       ・100% 匿名化クラッシュ相談
・3大コントローラー配列適応       ・Wabbajack マニフェスト直読       ・BYOK Gemini AI 連携 (同意必須)
       │                                  │                                  │
       ▼                                  ▼                                  ▼
[ クリップボード保護 ]            [ 信頼ベースライン ＆ 防御 ]       [ 特権分離 ＆ 回復 ]
・On-Demand 伏字化                ・初回信頼 (TOFU) スナップショット ・Named Pipe IPC (In-band Token)
・STA スレッドディスパッチ        ・Zero Trust Firewall 外部通信遮断 ・RescueSnapshot 自動ロールバック
```

---

# 6. 共通 Audit Event & Threat Reason

全モジュールで `GameSecurityTool.Contracts.Common` の統一列挙型を使用する（明示的数値固定済み）。

### 6.1 AuditEventType (新設イベント統合)
- **Save Backup & Time Travel:** `SaveBackupCreated = 40`, `SaveBackupFailed = 41`, `SaveBackupDeleted = 42`, `SaveRestoreStarted = 43`, `SaveRestoreCompleted = 44`, `SaveRestoreFailed = 45`, `SaveBackupPolicyChanged = 46`
- **Web Link & Overlay:** `WebLinkReceived = 50`, `WebLinkSanitized = 51`, `WebLinkBlocked = 52`, `WebLinkConfirmed = 53`, `WebBrowserSelected = 54`, `WebLinkEmergencyDetected = 55`, `WebBrowserEmergencyStopped = 56`, `ShortcutChangedDetected = 57`
- **Clipboard & Privacy:** `ClipboardSanitized = 58`, `ClipboardSanitizationFailed = 59`
- **Mod Provenance:** `ModInspected = 60`, `ModProvenanceVerified = 61`, `ModManifestImported = 62`
- **Crash Report & AI:** `CommunityReportGenerated = 63`, `AiConsentGranted = 64`, `AiConsentRevoked = 65`, `AiExplanationRequested = 66`

### 6.2 ThreatReasonCode
- `LnkTargetChanged` / `LnkArgumentsChanged` / `LnkSuspiciousCommand` / `LnkUnexpectedTarget` / `LnkUnsignedTarget`
- `SaveBackupIntegrityFailed` / `SaveRestorePartialFailed`
- `ClipboardUrlInvalidScheme` / `ClipboardUrlNormalizationFailed`
- `ExternalBrowserDetected` / `PunycodeDomainSuspicious`
- `DROPPED_OUTSIDE_GAME_FOLDER` / `UNSIGNED_BINARY` / `NEW_FILE_DETECTED` / `KNOWN_HIJACK_DLL_NAME`
- `CONTAINS_SUSPICIOUS_NETWORK_API` / `CONTAINS_PROCESS_SPAWNING_API`

---

# 7. 共通 Port (Interface) 一覧

すべての Port は `GameSecurityTool.Contracts` に定義し、Infrastructure が Adapter として実装する。

### Web, Clipboard & Overlay Ports:
* `IUrlSanitizer`
* `IWebLinkPolicyEvaluator`
* `ILaunchManagedUrlUseCase`
* `IBrowserPicker`
* `ISocialHostRuleRepository`
* `IWebLinkAuditLogger`
* `IClipboardUrlService`
* `IOverlayManager`
* `IGamepadDeviceDetector`
* `IGlobalHotkeyService`

### LNK, Save, Mod & Diagnostics Ports:
* `ILinkInspector`
* `ILinkBaselineRepository`
* `ISaveBackupService` (Application Facade)
* `ISaveBackupStorage`
* `ISaveBackupRepository`
* `ISaveDataPathScanner`
* `IProxyDllInspector`
* `IZoneIdentifierReader`
* `IArchiveIntegrityMatcher`
* `IModManifestImporter`
* `ICrashDiagnosticsService`
* `IHardwareDiagnosticsProvider`
* `ICrashEventLogReader`
* `IAiExplanationProvider`
* `IAiConsentRepository`

---

# 8. 永続化および並行性制御ルール

1. **書き込み直列化と完了同期保証 (`SqliteDatabaseWriter`):**
   すべての Insert, Update, Delete は `SqliteDatabaseWriter.EnqueueWriteAsync` 経由で実行し、`TaskCompletionSource` により物理コミット完了を待機して二段階コミットの順序性を保証する。
2. **先行ディレクトリ ACL 制御:**
   親ディレクトリ作成時点で継承切断およびアクセス制限を先行適用し、初期生成時の露出ウィンドウをゼロ化する。
3. **読み取りの独立性:**
   クエリは `IDbContextFactory<AppDbContext>` から生成された短命な `DbContext` を用いて、`AsNoTracking()` で並行実行する。

---

# 9. 実装優先度 & Definition of Done

```text
P1 (MVP Alpha / Beta)
├─ Web Link Protection (Managed Launch, Strict Query, Multi-Browser Picker)
├─ .LNK Hijack Detection (Baseline Diff, LotL 検知, SLR 非ブロッキング)
├─ On-Demand Clipboard URL Sanitizer (On-Demand Clean, STA 安全性)
├─ MOD 安全性診断 ＆ 出所追跡 (PE プロキシ構造診断 ＆ Zone.Identifier 出所追跡)
└─ インゲーム・オーバーレイ HUD (Hookless 最前面描画 ＆ デュアルトリガー ＆ 3大パッド配列適応)

P2 (v1.0 Core)
├─ Save Data Secure Auto-Backup & Time Travel (差分検知, Standard ZIP, Restrict 削除制約, コンテナ GC, ピン留め)
├─ 個人情報完全保護型 クラッシュ相談レポート (4大診断ブロック ＆ 外部ログ全行伏字化)
└─ BYOK Gemini AI 連携 ＆ 事前プライバシー同意 (完全オプトイン ＆ DPAPI 鍵保管 ＆ 送信前完全マスキング)
```

### Definition of Done (完了基準):
- Clean 5-Layer 依存方向違反ゼロ（NetArchTest で検証）。
- Nullable 警告ゼロ、コンパイルエラーゼロ。
- 巨大ファイル処理時の OOM ゼロ（64KB チャンクストリーミング I/O 検証）。
- 監査ログおよび外部送信データに生パスワード、生 URL Query、クリップボード本文、本名パスを出力しない。
- 単体テスト・結合テスト・セキュリティテストの定義済み品質ゲートをすべて満たすこと。

---

# 10. 最終原則

高度ユーザー保護機能群は、利用者の通常操作やゲームパフォーマンスを妨害することなく、ゲーム環境由来の危険な挙動を局所的に検知・抑制・復旧することを目的とする。

```text
Observe Safely ──> Explain Clearly ──> Preserve Carefully ──> Change Only With Consent
```

---

End of Document
```

---

