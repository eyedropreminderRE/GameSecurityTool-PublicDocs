# GameSecurityTool

# Audit and Evidence Model

## Security Evidence / Activity Tracking / Integrity Definition

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-004 |
| Version | 3.1 (Full 15 Sections Restored, Multi-Layer Anchor Protection & Complete Dual-Fusion Master Edition) |
| Status | Formal Baseline Specification (Highest Audit Authority) |
| Category | Audit and Evidence Management |
| Authority Level | Security Evidence Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- 監査ログ（Audit Hash Chain）
- 証拠保全（Evidence Management）
- セキュリティイベント追跡（全47種 AuditEventType）
- 操作履歴および改ざん防止（Layer 1 DPAPI ＆ Layer 2 特権アンカー）
- 復元・PC移行・エラー記録規則
- 保存範囲およびエビデンスバンドル出力規約

を定義する。

---

GST の Audit 目的は、ユーザー監視ではなく、

```text
Explainability  *  Accountability  *  Recovery Support  *  Tamper-Evidence
(説明可能性、説明責任、復旧支援、改ざん耐性の確立)
```

を提供することである。

---

# 1. Audit Philosophy

## 1.1 Core Principle
GST の Audit は、「システムおよびセキュリティ境界で何が起きたか」を客観的に記録する。

禁止事項（ユーザー監視の排除）：
```text
User Behavior Monitoring       : ユーザーの通常行動の監視
Personal Activity Tracking     : 個人ファイルやブラウジングの追跡
External Surveillance          : 外部サーバーへの監視ログ送信
```

## 1.2 Audit Definition
Audit とは、**セキュリティ上重要な操作・判定・変更の改変不能な記録（Security Relevant Event Record）** である。

記録対象：
- GST 操作（スキャン、バックアップ、復元、隔離）
- セキュリティ判断およびポリシー適用
- 設定変更およびプロファイル切り替え
- PC 移行および再封緘トランザクション
- Firewall ルール変更および WMI 監視イベント

---

# 2. Audit Boundary

## 2.1 GST Data and Game Data Separation
Audit の対象は「GST の活動履歴」であり、「ゲーム本文データ」ではない。

記録禁止事項:
- セーブデータ本文の内容
- ゲーム画面のスクリーンショット
- 個人ドキュメントやファイル本文
- キーボード等の入力履歴

## 2.2 ログ保存の厳禁事項 (Strict Prohibitions - 完全復元)
以下を監査ログおよびエビデンス内に記録することを厳禁とする：
- パスワード、パスフレーズ、暗号鍵、API キーの生文字列
- 生 URL Query String（`?token=...` 等）
- クリップボード本文
- ユーザー名を含むプロファイル絶対パス（必ず `ILogSanitizer` で伏字化）

## 2.3 Audit Storage Ownership
Audit Storage は GST が管理する内部データであるが、**所有権はユーザーに帰属する。**  
ユーザーは自らの意思で監査ログをエクスポート、バックアップ、または手動削除できる。

---

# 3. Audit Event Classification

## 3.1 Event Level Model (重要度分類)
Audit Event はその重要度に応じて 4 段階で分類する。

- **Level 0: Informational (通常情報):** `ScanStarted`, `ScanCompleted`, `FileChanged`, `SafeGameOnboarded`, `GameArchiveRestored`
- **Level 1: Context (状況確認情報):** `AllowListAdded`, `AllowListRemoved`, `SaveBackupPolicyChanged`, `WebBrowserSelected`
- **Level 2: Security Relevant (セキュリティ判断情報):** `FirewallRuleCreated`, `FirewallRuleRemoved`, `FirewallRuleDriftDetected`, `WebLinkBlocked`, `WebLinkSanitized`, `ShortcutChangedDetected`, `ModInspected`, `StrictLaunchActivated`
- **Level 3: Critical Evidence (重要証跡):** `IntegrityCheckFailed`, `QuarantineExecuted`, `RestoreFailed`, `RollbackCompleted`, `SaveRestoreFailed`, `WebLinkEmergencyDetected`, `AuditLayer2AnchorMismatch`

## 3.2 全47種 AuditEventType 体系 (数値固定・Append-Only Rule - 完全復元)
監査イベントは `GameSecurityTool.Contracts.Common.AuditEventType` で一元管理し、明示的な整数値を固定（追記専用規約）する。

```text
1. System & Lifecycle (0〜9)   : ScanStarted=0, ScanCompleted=1, FileChanged=2, IntegrityCheckFailed=3, CrashProtectionActivated=4, FullReversionExecuted=5
2. Firewall & Network (10〜19) : FirewallRuleCreated=10, FirewallRuleRemoved=11, FirewallRuleDriftDetected=12, OrphanedRuleRemoved=13, AntiCheatCompatibilityExceptionApplied=14 (legacy `AntiCheatBypassActivated` name retired)
3. Allow List (20〜29)         : AllowListAdded=20, AllowListRemoved=21, AllowListExpired=22
4. Quarantine & Recovery (30〜39): QuarantineExecuted=30, RestoreStarted=31, RestoreCompleted=32, RestoreFailed=33, RollbackStarted=34, RollbackCompleted=35, RollbackFailed=36
5. Save Backup (40〜49)        : SaveBackupCreated=40, SaveBackupFailed=41, SaveBackupDeleted=42, SaveRestoreStarted=43, SaveRestoreCompleted=44, SaveRestoreFailed=45, SaveBackupPolicyChanged=46
6. Web, LNK & Privacy (50〜59) : WebLinkReceived=50, WebLinkSanitized=51, WebLinkBlocked=52, WebLinkConfirmed=53, WebBrowserSelected=54, WebLinkEmergencyDetected=55, WebBrowserEmergencyStopped=56, ShortcutChangedDetected=57, ClipboardSanitized=58, ClipboardSanitizationFailed=59
7. Mod, AI & Launch (60〜69)   : ModInspected=60, ModProvenanceVerified=61, ModManifestImported=62, CommunityReportGenerated=63, AiConsentGranted=64, AiConsentRevoked=65, AiExplanationRequested=66, SafeGameOnboarded=67, GameArchiveRestored=68, StrictLaunchActivated=69
```

---

# 4. Audit Record Structure

## 4.1 Required Fields & SHA-256 Hash Chain Structure (完全復元)
すべての監査レコードは、創世記ハッシュ（`GenesisHash`）を起点とする暗号論的 **SHA-256 Hash Chain** 構造で SQLite に永続化する。

```text
Event ID               : SequenceNumber (連番 long) [PK]
Timestamp              : TimestampUtc (秒精度正規化 UnixTimeSeconds)
Operation ID           : OperationId (業務追跡 ID)
Event Type             : EventType (数値固定 Enum 0〜69)
Actor                  : 実行主体 (GST.Worker, User, ElevatedWorker 等)
Target                 : 対象パス・識別子 (ILogSanitizer 伏字化済み)
Result                 : 処理結果 (Success, Failed, UserRejected 等)
Previous Hash          : 直前レコードの CurrentHash (GenesisHash 起点)
Current Hash           : SHA256(Seq + PrevHash + OpId + EventType + Actor + Target + Result + TimestampUtc)
```

## 4.2 Event Identity
各イベントには一意の `OperationId`（業務単位）および `TransactionId`（原子処理単位）を付与し、複数テーブルを跨いだ追跡を可能にする。

---

# 5. Evidence Model

## 5.1 Evidence (証拠データ) の定義 (完全復元)
Evidence（証拠）とは、セキュリティ判断および説明（Explanation）の根拠となる客観的事実情報である。
- ファイルハッシュ (SHA256), Authenticode デジタル署名情報
- PE エクスポート／インポート構造解析シグナル (DirectXプロキシ構造, 外部通信API不在証明)
- NTFS `Zone.Identifier` 出所メタデータ (ダウンロード元URL, ZoneId)
- イベント発生時の `ConfigurationSnapshot`

## 5.2 Evidence Limitation
証拠収集はセキュリティ判断に必要なものに限定し、「念のため」という理由で無関係なファイルを過剰収集することを厳禁とする（Privacy First）。

---

# 6. Evidence Confidence Model

## 6.1 Confidence Principle
Evidence は確信度（Confidence Score: 0.0〜1.0）の算出根拠となる。

## 6.2 確信度連動 ＆ 自動処罰の禁止 (完全復元)
Evidence は確信度（0.0〜1.0）の算出根拠となるが、**高確信度であってもツールの独断による自動削除・自動隔離を行わず、必ず証拠を提示してユーザー同意を要求する**。

---

# 7. Audit Integrity Protection (多層アンカー Hash Chain - C-5 是正)

## 7.1 Hash Chain 連鎖
創世記ハッシュ（`GenesisHash`）を起点として、すべての監査レコードを SHA-256 で連鎖（Hash Chain）させ、途中の改ざん・削除・挿入を数学的に検知する。

## 7.2 多層アンカー二重保護 ＆ Fail-Closed 規約
同一ユーザー権限マルウェアによる DB 改ざんに対抗するため、多層アンカーを適用する：
1. **Layer 1 (ローカル DPAPI `audit.dpapi`):** ログ追記時に即時同期。
2. **Layer 2 (特権管理者アンカー `audit.anchor`):** `%ProgramData%\GameSecurityTool\` に非同期保存。
3. **Layer 2 未同期時 Fail-Closed 規約:** `TamperEvidentAuditLogger.VerifyAuditChainIntegrityAsync` において、Layer 2 特権アンカーが未同期（ファイル不在または空）あるいは不整合の場合、保護レベル低下をサイレントに容認せず、厳格に `false`（検証不合格・改ざんの疑い）を返却する。また、詳細検証 `VerifyAuditChainDetailedAsync` においても `AuditVerificationResultDto.IsLayer2Synchronized = false` をダッシュボードへ明示伝播し、セキュリティリスクを即時警告する。

---

# 8. Audit Operation Rules (最新操作種別・記録義務表 - 完全復元)

## 8.1 Required Audit Operations (記録必須操作一覧)

| 操作種別 | Audit 記録義務 | 備考 |
| :--- | :---: | :--- |
| **Backup / Restore / Rollback** | **必須** | RescueSnapshot 退避・アトミック復元結果を記録 |
| **Quarantine / Reseal** | **必須** | 64KB AEAD 隔離・PC 移行再封緘トランザクションを記録 |
| **Firewall / WER Change** | **必須** | ルールタグ・BeforeState 退避・復元結果を記録 |
| **Allow List / Consent** | **必須** | 例外登録・Gemini AI 同意/撤回を記録 |
| **Launch Modes & Mod Inspection** | **必須** | `StrictLaunchActivated`, `SafeGameOnboarded`, `ModInspected` |
| **通常画面表示 / 参照** | **対象外** | 不要なユーザー行動追跡を防止するため記録しない |

## 8.2 Read Operations
通常の参照操作（ゲーム一覧表示、ダッシュボード閲覧、ステータス確認）は監査ログの記録対象外とする（不要なユーザー行動監視の防止）。

---

# 9. Restore Audit Model (復元監査モデル - 完全復元)

セーブデータおよび隔離ファイルの復元時は、必ず以下を監査レコードとして記録する：
- 復元元バックアップ / 隔離コンテナ ID (`BackupId` / `TransactionId`)
- 復元先ルート絶対パス (`TargetRootPath` - `ILogSanitizer` 伏字化済み)
- ユーザー明示承認の事実
- 処理成否および失敗ステップ (`FailedStep`)
- タイムスタンプ (UTC)

復元操作をログに記録せずに実行することを厳禁とする。

---

# 10. Migration Audit Model (移行監査モデル - 完全復元)

PC 移行（Migration）では以下を記録する：
- 移行パッケージ作成 (`Export Created`)
- インポート開始 ＆ フォーマットバージョン照合結果
- DPAPI 再封緘トランザクション結果 (`Quarantine Re-sealed`)
- 失敗時のエラー情報およびフォールバック誘導

### 移行プライバシー規約:
移行監査ログ内に、ユーザーパスフレーズ、暗号鍵、平文認証情報を含めてはならない。

---

# 11. Audit Export Model (エビデンス出力 - 04-04 準拠)

## 11.1 User Export
ユーザーはサポート提出やフォレンジック調査のために監査ログおよびエビデンス（`EvidenceBundle.zip`）をエクスポートできる。

## 11.2 Export Format & エビデンス出力 3大規約 (完全復元)
1. **事前整合性検証:** エクスポート実行前に必ず Hash Chain 連鎖および多層アンカーの完全性を検証し、改ざん有無（`IntegrityStatus`）をマニフェストに明記。
2. **完全ストリーミング I/O:** メモリ一括展開を禁止し、64KB チャンクの `FileStream` ➔ `ZipArchive` ストリーミング出力により OOM を防止。
3. **個人情報完全伏字化:** プロファイルパスや個人情報は `PrivacyLogSanitizer` で確実にマスキング。

---

# 12. Audit Retention Model (保持・クリーンアップ規約 - 完全復元)

## 12.1 Default Policy
監査ログは、過去のセキュリティ判断および復元証跡を証明するため、原則として保持される。

## 12.2 User Controlled Cleanup
監査ログの削除は、**ユーザーの明示的な要求と確認ダイアログによる承認があった場合のみ実行可能** とする。
ツールの独断による監査ログの自動サイレント削除を厳格に禁止する。

---

# 13. Error and Failure Logging (失敗・障害記録規約 - 完全復元)

## 13.1 Failure Rule
すべての処理失敗（バックアップ失敗、復元失敗、ロールバック発生、権限昇格拒否、WMI 監視停止）は、必ず監査イベントまたはエラーログとして記録する。

## 13.2 Sensitive Information Exclusion
エラーログおよびスタックトレース内に、パスワード、API キー、生暗号鍵、生 URL Query を含めることを厳禁とする。

---

# 14. Relationship With Other Models (他モデルとの関係性 - 完全復元)

## 14.1 Security Boundary
操作境界と権限モデル：
```text
02_Security_Boundary_and_Protection_Model.md
```

## 14.2 Game Environment Protection
ゲーム保護および起動モード：
```text
03_Game_Environment_Protection_Model.md
```

## 14.3 Backup & Migration
データ所有権および PC 移行モデル：
```text
05_Save_Backup_and_Data_Ownership_Model.md
09_PC_Migration_and_Recovery_Model.md
```

## 14.4 Implementation & Evidence Export
監査ロガー実装およびエビデンス出力仕様：
```text
modules/03-06_CFG_Configuration_and_Gamer_UX.md
02_Features/Security/Trust_Enhancement_Spec/04-04_Evidence_Bundle_and_Export.md
```

---

# 15. Final Audit Statement

GST の Audit は、ユーザーを監視するために存在するのではない。

---

Audit の目的：

```text
Remember Safely
Explain Clearly
Verify Tamper-Evidence Reliably
Never Fail Silently
Preserve User Privacy Always
```

である。

---

