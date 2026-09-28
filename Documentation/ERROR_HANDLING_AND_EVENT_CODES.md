# GameSecurityTool Error Handling & Event Codes Specification

**Document ID:** GST-SPEC-ERROR-001  
**Version:** 4.5 (Bounded Fail-Safe / Cleanup Exception Policy Edition)
**Status:** Approved Master Specification  
**Target Projects:** `GameSecurityTool.Contracts`, `GameSecurityTool.Application`, `GameSecurityTool.Infrastructure`  

> **Lifecycle note:** This document describes approved design/specification scope. Its presence does not by itself indicate that the described behavior is implemented, Windows-verified, or released.

---

# 1. エラーハンドリング哲学 ＆ Result パターン

GST では、業務ユースケースにおいて例外（`Exception`）を制御フローとして乱用することを禁止し、明示的な **`Result<T>` パターン** を採用する。

### 規約:
1. **予期される失敗 (Expected Failures):**
   「ファイルロック中」「ハッシュ不一致」「空き容量不足」「ユーザーキャンセル」「未署名ファイルの検出」「Zip Slip 検知」「マイグレーションバージョン非互換」「なりすまし許可リスト検知」等は例外をスローせず、`Result<T>.Failure(ErrorCode, Message)` または適切な結果 DTO を返却する。
2. **例外的な障害 (Unexpected Disasters):**
   「DB ファイル破損」「物理ディスク障害」等の致命的障害のみ例外を送出し、パイプライン最上位で安全に捕捉・ログ記録してセーフモードまたはロールバックへ移行する。
3. **空 catch の厳禁:**
   `catch (Exception) { }` で例外を握りつぶすことを厳禁とする。

---

## 1.1 Canonical Contracts Result / Error Contract

The canonical `Result<T>`, `ResultError`, and `ErrorCode` types are owned by `GameSecurityTool.Contracts.Common` in `ResultContracts.cs`.

- `Result<T>.Success(value)` represents an expected successful operation.
- `Result<T>.Failure(code, message)` represents an expected failure without using exceptions as normal control flow.
- `Result<T>.Value` is populated only on success.
- `Result<T>.Error` is populated only on failure.
- `ResultError` contains the structured `ErrorCode` and failure message.
- `ErrorCode` is an immutable value type owned by Contracts; feature projects must not define competing error-code types.
- Failure messages must already satisfy the privacy/sanitization requirements before crossing presentation, logging, or external-AI boundaries.
- Unexpected catastrophic failures remain exceptional and are handled at the safe top-level boundary.
- `Result<T>.Success` and `Result<T>.Failure` are the single approved factory members. Feature-local Result/Error replacements are prohibited.

## 1.2 approved error-handling policy Bounded Fail-Safe / Cleanup Exception Policy

The following exception-handling policy is authoritative across Tier 0–3 implementation guidance:

- **Empty catch is always prohibited.** `catch { }`, `catch (Exception) { }`, and equivalent exception suppression that produces no observable handling are forbidden.
- **Expected failures** must continue to use `Result<T>` / result DTOs where the failure is part of the normal contract. A broad catch must not hide a known expected failure behind a generic fallback.
- **Secondary cleanup failures** may be handled independently only when the primary operation has already failed or completed, the cleanup is limited to non-authoritative intermediate artifacts, and the cleanup failure cannot affect user-asset safety, security-boundary state, transaction correctness, or recovery correctness. The primary failure/result must remain authoritative; cleanup failure must never manufacture success.
- **Recovery-, transaction-, security-, or asset-critical cleanup is not a suppressible cleanup-only failure.** Its failure must remain observable and must participate in the applicable Result/error/recovery contract.
- **Explicit defensive fail-safe conversion** such as `catch (Exception) { return false; }` may be used only at a documented defensive boundary where the fallback is demonstrably conservative/fail-closed. Such handling must provide sanitized observability through the global logging boundary before or while returning the safe fallback.
- The `ILogSanitizer` / Global Sanitizing Logger Provider / sink gateway must govern any exception observation emitted by the above paths. Raw exception/state must not cross the physical sink boundary.
- Fail-safe conversion must not turn an operational failure into a success state, authorization, completed transaction, or verified recovery result.
- Tier 3 code blocks remain implementation guidance; every example must conform to these rules when translated into production code. This policy does not authorize production implementation by itself.

# 2. 共通 AuditEventType 一覧 (全50種 / 明示的数値固定)

全モジュールで `GameSecurityTool.Contracts.Common.AuditEventType` を統一使用する。IME-004 / WIPER-005 を含む追加イベントも本一覧をSSOTとし、既存数値を変更せず追記する。
SQLite への `HasConversion<int>()` 永続化における意味的完全性を担保するため、**追記専用原則（Append-Only Rule）** を厳守する。

| カテゴリ | AuditEventType | 整数値 | 発火タイミング |
| :--- | :--- | :---: | :--- |
| **System & Lifecycle** | `ScanStarted` | `0` | スキャン開始 |
| | `ScanCompleted` | `1` | スキャン完了 |
| | `FileChanged` | `2` | ファイル更新検知 |
| | `IntegrityCheckFailed` | `3` | アプリ本体の署名/ハッシュ検証失敗 |
| | `CrashProtectionActivated` | `4` | WER テレメトリ遮断適用 |
| | `FullReversionExecuted` | `5` | 完全クリーンアンインストール実行 |
| **Firewall & Network** | `FirewallRuleCreated` | `10` | ブロックルール作成 |
| | `FirewallRuleRemoved` | `11` | ブロックルール削除 |
| | `FirewallRuleDriftDetected` | `12` | OS 上の Firewall 設定乖離検知 |
| | `OrphanedRuleRemoved` | `13` | 孤立ルールのユーザー承認削除 |
| | `AntiCheatCompatibilityExceptionApplied` | `14` | ユーザー承認済みAnti-Cheat互換性例外の適用 |
| **Allow List** | `AllowListAdded` | `20` | 例外許可ルール追加 |
| | `AllowListRemoved` | `21` | 例外許可ルール手動削除 |
| | `AllowListExpired` | `22` | 一時許可ルールの期限切れ無効化 |
| **Quarantine & Restore** | `QuarantineExecuted` | `30` | 64KB チャンク暗号化隔離完了 (QRTv03) |
| | `RestoreStarted` | `31` | 隔離ファイル復元開始 |
| | `RestoreCompleted` | `32` | 隔離ファイル復元完了 |
| | `RestoreFailed` | `33` | 隔離ファイル復元失敗 (`PartialFailed`) |
| | `RollbackStarted` | `34` | OS 変更 / セーブデータロールバック開始 |
| | `RollbackCompleted` | `35` | ロールバック完了 |
| | `RollbackFailed` | `36` | ロールバック失敗 |
| **Save Backup & Time Travel** | `SaveBackupCreated` | `40` | セーブデータ ZIP 作成完了 |
| | `SaveBackupFailed` | `41` | バックアップ失敗（容量不足等） |
| | `SaveBackupDeleted` | `42` | 古い世代 / 孤立コンテナの安全削除完了 |
| | `SaveRestoreStarted` | `43` | セーブデータ復元開始 (RescueSnapshot 退避) |
| | `SaveRestoreCompleted` | `44` | セーブデータ復元完了 |
| | `SaveRestoreFailed` | `45` | セーブデータ復元失敗 (直前状態へロールバック) |
| | `SaveBackupPolicyChanged` | `46` | 保持世代数・容量上限等の設定変更 |
| **Web Link, Clipboard & Overlay** | `WebLinkReceived` | `50` | Web リンク起動要求受信 |
| | `WebLinkSanitized` | `51` | Strict Query Stripping 完了 |
| | `WebLinkBlocked` | `52` | ポリシー判定による遮断 |
| | `WebLinkConfirmed` | `53` | ユーザー確認プロンプト表示 |
| | `WebBrowserSelected` | `54` | ブラウザ選択 & 安全起動 |
| | `WebLinkEmergencyDetected` | `55` | WMI による迂回ブラウザ起動検知 |
| | `WebBrowserEmergencyStopped`| `56` | 迂回プロセスの安全停止実行 |
| | `ShortcutChangedDetected` | `57` | `.LNK` ショートカット改ざん検知 |
| | `ClipboardSanitized` | `58` | クリップボード URL の安全化完了 |
| | `ClipboardSanitizationFailed`| `59` | クリップボード URL の形式不正/拒否 |
| **Mod, Diagnostics & AI** | `ModInspected` | `60` | 未署名 MOD の静的レントゲン診断完了 |
| | `ModProvenanceVerified` | `61` | Zone.Identifier / 元 ZIP 突合による出所証明完了 |
| | `ModManifestImported` | `62` | Wabbajack / Collections 定義の一括登録完了 |
| | `CommunityReportGenerated` | `63` | 個人情報マスキング済み相談レポート生成完了 |
| | `AiConsentGranted` | `64` | Gemini AI 連携の事前説明確認 ＆ ユーザー同意完了 |
| | `AiConsentRevoked` | `65` | Gemini AI 同意撤回 ＆ API キー完全消去完了 |
| | `AiExplanationRequested` | `66` | 伏字化プロンプトによる AI 解説・クラッシュ診断完了 |
| **Safe Onboarding & Launch Modes** | `SafeGameOnboarded` | `67` | ZIP/7z からの安全展開・プロファイル登録完了 |
| | `GameArchiveRestored` | `68` | ゲーム本体アーカイブ (.7z/.zip/.rar) からの直接復元完了 |
| | `StrictLaunchActivated` | `69` | 厳格モード (In/Out 遮断 ＆ WER 抑止) による起動完了 |
| **Anti-Wiper Defense** | `WiperThreatBlocked` | `70` | ゲーム起点の破壊的子プロセスを未然遮断 |
| | `WiperBreakerTriggered` | `71` | ファイル変更バースト / Canary検知によるブレーカー発動 |

---

# 3. 共通 ThreatReasonCode 一覧 (PascalCase 統一 ＆ 完全網羅版 - H-1 是正)

検知理由および説明生成で利用する標準コード体系。`Contracts.Common.ThreatReasonCodes` 定数クラスと 1:1 で完全同期する。

| ThreatReasonCode | 深刻度 (ThreatSeverity) | 説明 |
| :--- | :---: | :--- |
| `DroppedOutsideGameFolder` | Danger | ゲーム実行中に外部領域へ配置された不審バイナリ |
| `UnsignedNewBinary` | Warning | ゲームフォルダ内に追加された未署名実行ファイル |
| `NewFileDetected` | Info | 直前のゲームプレイまたは外部ツールによって新しく配置されたファイル (H-1 追加) |
| `AllowListMatched` | Info | 例外許可リストに適合したファイル (H-1 追加) |
| `KnownHijackDllName` | Danger | `version.dll`, `dxgi.dll` 等の既知ハイジャック警戒名 |
| `ContainsSuspiciousNetworkApi` | Danger | 未署名 DLL 内に外部通信 API（`ws2_32.dll` 等）を検出 |
| `ContainsProcessSpawningApi` | Danger | 未署名 DLL 内に外部プロセス起動コード（`CreateProcess` 等）を検出 |
| `TamperedAllowList` | Danger | ハッシュ許可リストと不一致のファイルが検出された（ファイル改ざん・なりすましの疑い） |
| `LnkTargetChanged` | Danger | ゲームショートカットのリンク先が変更された |
| `LnkArgumentsChanged` | Danger | ショートカットの引数に不審なコマンド連結を検知 |
| `LnkSuspiciousCommand` | Danger | `cmd.exe` や `powershell.exe` を経由する起動 |
| `LnkUnexpectedTarget` | Warning | 許可された実行ファイル以外を指すショートカット |
| `LnkUnsignedTarget` | Warning | リンク先が未署名バイナリ |
| `SaveBackupIntegrityFailed` | Danger | バックアップコンテナのハッシュ不一致または破損 |
| `SaveRestorePartialFailed` | Danger | セーブデータ復元が中途失敗 (RescueSnapshot から復旧) |
| `SaveBloatDetected` | Warning | セーブデータ容量の急激な異常肥大化（+50%以上）を検知 |
| `RescueSnapshotCreationFailed` | Danger | 復元前退避が失敗し、ロールバック保護が有効にならず安全停止 |
| `ClipboardUrlInvalidScheme` | Warning | `http`/`https` 以外の危険スキーム (`javascript:` 等) |
| `ClipboardUrlNormalizationFailed`| Warning | デコード不能または不正な URL 表記 |
| `ExternalBrowserDetected` | Warning | Managed Path を迂回して起動されたブラウザ |
| `PunycodeDomainSuspicious` | Warning | 国際化ドメイン (`xn--`) によるホモグラフ偽装の可能性 |
| `ZipSlipTraversalDetected` | Danger | アーカイブ内に管理フォルダ外へ脱出する相対パス (`../`) を検知 |
| `ZipBombDecompressionAborted` | Danger | 展開サイズが異常（容量制限超過）のため Zip Bomb として処理中断 |
| `AuditLayer2AnchorMismatch` | Danger | 特権アンカー (Layer 2) と DPAPI アンカーのハッシュ不一致 (同一権限マルウェアによる改ざん疑い) |
| `MigrationPackageVersionMismatch` | Warning | PC 移行パッケージの Format Version が非互換のためインポートを安全中断 |
| `WiperChildProcessDetected` | Danger | ゲーム起点のシェル/管理ツールが破壊的コマンドを実行しようとしたことを検知 |
| `WiperCanaryTriggered` | Danger | Anti-Wiper Canary の削除・変更を検知 |
| `WiperBurstDeletionDetected` | Danger | 短時間の大量削除イベントを検知 |
| `WiperBurstModificationDetected` | Danger | 短時間の大量変更イベントを検知 |

---

# 4. プライバシー保護エラー表示規約 (`ILogSanitizer` 連携 ＆ メモリ安全性)

ユーザーに提示するエラーメッセージ、ログ出力、および外部 AI プロンプト生成において、以下の機密情報を一切含めてはならない。

1. **生 URL Query String:** `https://example.com/p?token=xxx` ➔ `https://example.com/p` のみ表示。
2. **生パスワード / 暗号鍵 / API キー:** ログや画面にパスワードおよび Gemini API キーの平文を出力しない。
   - **メモリ安全性規約:** 例外メッセージ等に `string` 型で保持されたパスワード情報がスタックトレース等を通じてヒープ上に残存することを完全に防ぐため、これらを取り扱う変数は必ず `byte[]`（または `SecureString`）で受領し、利用後直ちに `Array.Clear` または `CryptographicOperations.ZeroMemory` で消去すること。
3. **ユーザー名を含むプロファイル絶対パス:** `C:\Users\<UserName>\...` ➔ `ILogSanitizer` により `C:\Users\***\...` へ自動マスク。
4. **クリップボード本文:** クリップボードの生テキストを監査ログに保存しない。

---

End of Document
```

---
