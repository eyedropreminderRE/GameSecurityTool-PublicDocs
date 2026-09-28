# GameSecurityTool

# Privacy and Threat Model

## Privacy Protection / Threat Analysis / Security Assumption Definition

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-006 |
| Version | 3.3 External AI Outbound Consent & Payload Boundary — Final Send Re-sanitization) |
| Status | Formal Baseline Specification (Highest Privacy Authority) |
| Category | Privacy and Threat Management |
| Authority Level | Security Policy Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- Privacy Protection（プライバシー保護方針）
- Data Handling Policy（データ分類と境界）
- Credential & Secret Protection（認証情報・暗号鍵の保護）
- Local First & Zero Telemetry Policy
- Privacy Boundary Rules（アクセス許可／アクセス禁止データ一覧）
- Security Event Response（自動処罰の禁止とユーザー主権）
- UI Transparency（透明性の保証と隠蔽操作の禁止）
- Threat Model（脅威モデル T1〜T11）
- 生成 AI（Gemini API）およびクラッシュ相談レポートのプライバシー規約

を定義する。

---

GST の Privacy 基本原則：

```text
Collect Less
Process Locally
Expose Clearly
Control Explicitly
(最小限しか扱わず、ローカルで処理し、透明に説明し、ユーザーが完全に制御する)
```

である。

---

# 1. Privacy First Principle

## 1.1 Core Privacy Philosophy
GST は Privacy First 設計を採用する。 
GST の目的は、**ユーザーのゲーム環境を保護することであり、ユーザー情報を収集・監視・プロファイリングすることではない。**

禁止事項：
```text
Protection (保護) ──> Mass Data Collection (大量データ収集へのすり替え)
```

## 1.2 Data Minimization (データ最小化原則)
GST はセキュリティ判定および復元支援に必要な最小限の技術データのみを扱い、無関係な個人ファイルを走査・収集しない。

---

# 2. Data Classification Model (データ分類)

## Category A: GST Metadata
- **内容:** Game Profile, Configuration, Policy, Audit Hash Chain, Snapshot Manifests
- **管理:** GST 内部管理データ（NTFS ACL および DPAPI で保護）。

## Category B: User Game Data
- **内容:** セーブデータ, 設定ファイル (.ini, .cfg, .json), ユーザー作成 MOD
- **所有権:** **ユーザーの完全な資産**（GST は所有権を取得しない）。

## Category C: Security Evidence
- **内容:** ファイルハッシュ (SHA256), デジタル署名状態, PE プロキシ構造シグナル, イベントログ
- **用途:** セキュリティ判断およびユーザーへの説明（Explanation）生成のみ。

## Category D: Credential & Secret Data
- **内容:** パスワード byte[], DPAPI 暗号化マスターキー, Gemini API キー
- **保護水準:** 最高レベル保護対象（メモリ上即時消去・平文永続化ゼロ）。

---

# 3. Local First Architecture

## 3.1 Processing Principle
GST は Local First を採用する。すべてのスキャン、ハッシュ計算、バックアップ、復元はローカル端末内で完結する。

## 3.2 External Communication Policy
許可される外部通信は、**ユーザーが明示的に開始した操作（User Initiated Action）のみ** とする。
- ユーザーによる明示的なエクスポート
- ユーザーによる手動更新確認
- オプトインでの BYOK Gemini AI 相談

## 3.3 Silent Communication Prohibition (サイレント通信の禁止)
以下を製品全体で厳格に禁止する：
- バックグラウンドでの自動アップロード
- サイレントなテレメトリ送信
- ユーザー行動トラッキング

---

# 4. Telemetry Policy

## 4.1 Telemetry Philosophy
GST は、利用状況収集を基本機能としない。
- アプリケーション利用状況の追跡禁止
- ゲームアクティビティの追跡禁止
- ユーザー行動パターンの分析禁止

## 4.2 Optional Diagnostic Export
診断情報が必要な場合は、ユーザーが明示的に要求・内容レビューを行い、手動でエクスポートして共有する形態のみを許可する（自動送信は完全禁止）。

---

# 5. Credential Protection Model

## 5.1 Credential Principle
認証情報・暗号鍵は最重要保護対象として扱う。平文ファイル保存、平文ログ出力、監査ログへの平文記録を厳禁とする。

## 5.2 Credential Storage
Windows DPAPI (`CurrentUser`) および Protected Local Storage を使用する。

## 5.3 Credential Boundary
資格情報はユーザーコンテキストにバインドされるため、単純なファイルコピーによる別 PC での直接再利用を禁止する。

## 5.4 Migration Credential Handling
PC 移行時は、ユーザー承認のもとで Argon2id 中間暗号化を用いて移送し、新 PC 上で新 DPAPI コンテキストをアトミックに再構築する（`09_PC_Migration_and_Recovery_Model.md` 準拠）。

## 5.5 【確定】DPAPI 障害救済用パスワード暗号化リカバリーパッケージ (`.gstmgr`)
Windows パスワード強制変更や OS プロファイル破損により Windows DPAPI が使用不能になった場合、隔離庫マスターキーや重要設定が開けなくなるリスクに対処する：
1. **生キーメモの禁止:** ユーザーに平文の暗号鍵をメモさせる危険な運用を廃止し、Argon2id + AES-256-GCM で保護された「パスワード暗号化リカバリーパッケージ (`.gstmgr`)」を作成・保管させる。
2. **UAC セキュアデスクトップ昇格:** パッケージのエクスポート時は必ず UAC 昇格（セキュアデスクトップ）を要求し、情報窃取マルウェアやリモートデスクトップによる不正エクスポートを物理遮断する。
3. **高リスクシークレットの除外:** リカバリーパッケージ内には Gemini API キーやセーブデータの生パスワード等の外部認証情報は含めず、GST 内部の隔離庫鍵と基本構成のみを安全に格納する。

---

# 6. Threat Model Overview (11大脅威体系)

```text
T1: Unauthorized Access (第三者による GST 管理情報への不正アクセス)
T2: Data Theft (バックアップやセーブデータの不正取得・盗難)
T3: Configuration Tampering (設定改ざんによる保護の無効化)
T4: Audit Tampering (監査ハッシュチェーンの改ざん・証拠隠滅)
T5: Credential Exposure (パスワード・暗号鍵のメモリダンプ露出)
T6: Privilege Abuse (常時管理者権限要求による権限濫用)
T7: Migration Abuse (PC 移行パッケージ悪用による不正インポート)
T8: Web Link Telemetry & Token Leakage (URL クエリを通じたトークン漏洩)
T9: Clipboard Privacy Exposure (クリップボード常時監視による意図しない情報取得)
T10: Generative AI Data Leakage (外部 AI 連携時の個人情報・実名パス漏洩)
T11: Crash Diagnostics Report Data Leakage (コミュニティ相談時の身バレ・環境漏洩)
```

---

# 7. 個別脅威対策仕様 (T1〜T7)

## T1: 不正アクセス対策
- 親ディレクトリ段階での NTFS ACL 先行適用（CurrentUser / SYSTEM のみフルコントロール）。

## T2: データ盗難対策
- Standard ZIP + Versioned Authenticated Envelope + Argon2id + AES-256-GCM によるポータブル暗号化。
- Secure Backupは非空パスワードを必須とし、`GST_Manifest.json.enc` にユーザー固有/file-specific metadataを暗号化・認証して格納する。DBの`ExpectedManifestHash`は追加の外部anchorであり、単独の真正性根拠ではない。
- 復元時はarchive由来`RelativePath`を信用せず、root containment + TOCTOU + Reparse Point検証を適用する。

## T3: 設定改ざん対策
- Universal Rule Scope 8段階決定表による Global Enforced ルールの優先適用。

## T4: 監査改ざん対策
- SHA-256 Hash Chain 連鎖 ＋ ローカル DPAPI (Layer 1) ＆ 特権管理者アンカー (Layer 2) の多層検証（未同期時の可視化含む）。

## T5: 秘密情報露出対策
- パスワード `byte[]` 引き回し ＆ `ZeroMemory` 消去。

## T6: 権限濫用対策
- Standard User 起動厳守 ＆ DACL 保護 Named Pipe IPC によるオンデマンド昇格。

## T7: 移行パッケージ不正対策
- Argon2id によるパッケージ全体暗号化 ＆ パスフレーズ検証。

---

# 8. WER テレメトリ遮断ポリシー (Crash Report Protection)

## 8.1 Windows Error Reporting 漏洩脅威
ゲームクラッシュ時に OS が自動起動する `WerFault.exe` は、メモリダンプやゲーム実行パス、環境変数等のセンシティブ情報を Microsoft サーバーへ外部送信するプライバシーリスクを持つ。

## 8.2 プライバシー保護統制
- 保護対象ゲーム実行中、`HKCU\Software\Microsoft\Windows\Windows Error Reporting` の `DontSendAdditionalData = 1` および `Logging = 0` を一時適用する。
- 適用前の元値を `WerBeforeState.json`（primary）と `WerBeforeState.recovery.json`（recovery copy）の二重ジャーナルへ退避する。
- ジャーナルは値の「存在／不在」を明示的に保持し、schema・必須フィールド・fingerprint を検証する。片方が壊れていても、もう片方が有効なら元値復元に使用できる。
- 両方が壊れている、内容が不一致、または artifact が存在するのに有効な復元元がない場合は fail-closed とし、Windows 既定値への置換、現在値の推測変更、ジャーナル削除を行わない。
- 両ジャーナルが存在しない場合は `NoOutstandingJournal` として WER を変更せず、ジャーナルライフサイクル不変条件の範囲で「未処理のGST WER変更なし」と扱う。
- 復元成功後にのみジャーナルを削除し、削除失敗時は artifact を残して再試行可能にする。

---

# 9. Web リンク ＆ クリップボード プライバシー (T8, T9)

## 9.1 T8: Strict Query Stripping
URL の `?` 以降のクエリパラメータはデフォルトで完全破棄し、セッショントークンや追跡タグの外部送信を構造的に遮断する（明示許可された Allowlist のみ例外維持）。

## 9.2 T9: オンデマンド・クリップボード規約
バックグラウンドでのクリップボード監視ループを完全禁止し、ユーザーが明示的に「安全化」を要求した時のみ処理する（クリップボード本文の履歴保存・ログ記録は禁止）。

---

# 10. 生成 AI (Gemini API) プライバシー規約 (T10) ＆ Universal Privacy Shield

ユーザーが自身の Google API キー（BYOK）を用いて AI 診断を利用する際のプライバシー統制：

1. **完全オプトイン (Default OFF):** 初期状態では外部通信は完全ゼロとし、ユーザーの明示的な AI 利用同意が有効でない限り、外部 AI 通信を開始しない。
2. **専用 AI Outbound Sanitization Boundary:** Gemini へ到達するすべてのテキスト入力は、通常ログ用の `ILogSanitizer` や外部ログ取込用の `IExternalLogSanitizer` とは別の、AI outbound 専用 sanitization policy を通過しなければならない。対象には、ユーザー入力、クラッシュレポート、診断詳細、provenance、ゲーム／画面コンテキスト、マニュアル／コンテキスト抜粋、設定スナップショット等を含む。
3. **不変な送信準備境界:** AI Provider へ渡せるのは、AI outbound sanitization 済みの内容、送信payload fingerprint、および有効な consent / approval revision を束ねた immutable prepared request のみとする。raw text を Provider が直接受け取る経路を設けない。
4. **Prompt Preview と最終承認:** Prompt Preview は情報表示だけでなく送信承認境界とする。ユーザーがPreview後に編集した場合、以前の準備済みpayloadと承認は無効化し、編集内容を再サニタイズして新しいfingerprintを作成し、その最終内容について再承認を得る。
5. **明示同意 + Liability Guard:** API key の存在を同意の代用とせず、現在有効な外部AI利用同意と当該利用時のLiability Guard承認をともに満たした場合のみ送信可能とする。
6. **同意revision拘束:** 送信準備物は、それを承認した consent revision に束縛する。撤回によって未送信の承認は無効化され、stale authorization は fail-closed として新規ネットワーク要求を開始してはならない。
7. **送信直前の最終ゲート:** Infrastructure は準備済みoutbound text fieldsをcanonicalなAI outbound sanitization policyで再処理し、current consent / authorizationを再検証し、stale ConsentRevisionを拒否し、再サニタイズ後のpayload fingerprintが承認済みfingerprintと一致することを確認する。いずれかが不一致ならネットワーク要求を開始しない。
8. **Privacy guaranteeの範囲:** GST は AI outbound sanitization policy で定義された機密パターンの除去と、承認済みsanitized payloadのみが送信境界を越えることを保証する。未知の個人情報を含む任意の自由入力について「絶対に個人情報が送信されない」とは表現しない。
9. **Header-Only Authentication:** API キーを URL クエリ文字列（`?key=...`）に含めることを厳禁とし、`x-goog-api-key` HTTP ヘッダーで送信する。
10. **中間サーバー完全ゼロ:** ユーザー PC から Google 公式 API エンドポイントへ直接 HTTPS 通信し、第三者中継サーバーを介在させない。
11. **API キーの保護:** Windows DPAPI で暗号化保存し、画面上では `************` で常時マスク表示する。API key のbytes/zeroization境界はを正本とし、で変更しない。

# 11. クラッシュ相談レポート型安全プライバシー規約 (T11 - 新設)

外部コミュニティ（フォーラム等）で質問・共有するためのレポート生成におけるプライバシー統制：

1. **Universal Privacy Shield の完全適用:** `CommunityReportFormatter` により、実名パスや IP アドレスだけでなく、各種 Webhook、OAuth/Bearer トークン、PC 名、MAC アドレスを自動的に `[REDACTED_***]` または `***` に伏字化。
2. **型安全サニタイズパイプライン:** 外部ログ（`crash.log` 等）を取り込む際、生の `string` を直接格納することを禁止し、必ず `IExternalLogSanitizer` を通過して生成された `SanitizedLogTextDto` のみを添付可能とする。
3. **4 大診断ブロックの限定開示:** ハードウェア仕様、WER 例外コード、MOD 差分、伏字化ログのみを出力し、個人ドキュメントやゲーム内セーブデータ本文を含めない。

---

# 12. Privacy Boundary Rules (プライバシー境界規則 - 完全復元)

## 12.1 Allowed Data Access (アクセス許可データ)
- 対象ゲームのインストールパス・実行ファイル情報
- セキュリティ判定に必要な客観的証拠（ハッシュ、署名、PE 構造）
- ユーザーが明示的に要求したセーブデータバックアップ
- **Anti-Wiper の明示的オプトインで選択された保護rootに対する、保護判定に必要な最小限のファイルシステム変更通知・メタデータ処理**
- **同じ明示的オプトインで選択された保護root直下へのGST Canaryファイルの新規配置・状態確認・撤去**。既存ユーザーファイルをCanaryとして再利用・上書きしてはならない。

### 12.1A Anti-Wiper Personal-Asset Protection Exception
Anti-Wiper は、ユーザー自身が明示的に保護対象として選択し有効化した個人資産root（例: Desktop / Documents / Pictures）について、ワイパー攻撃による削除・変更を検知する目的に限り、**ファイルシステムイベントおよび保護判定に必要な最小限のメタデータを一時的に処理できる**。

この例外は「個人データの内容」へのアクセス許可ではない。GSTは、Anti-Wiperの保護判定のために個人ファイルの本文・画像・メール本文・文書本文・セーブデータ本文等を読み取り、解析、収集、保存してはならない。また、保護対象rootを明示的に選択していない限り、当該個人rootを監視対象としてはならない。通常の既定状態ではAnti-WiperはOFFであり、保護rootは空集合とする。

Anti-Wiperのイベント処理で得られたユーザー固有パス等のメタデータは、診断・監査・UI表示に必要な場合も含め、既存のログサニタイズ境界およびデータ最小化原則に従い、平文のまま通常ログへ出力してはならない。

## 12.2 Forbidden Access (アクセス絶対禁止データ)
以下へのアクセス・走査・収集を厳格に禁止する：
- ブラウザの履歴、Cookie、保存されたパスワード
- ユーザーの個人ドキュメント（写真、文書、メール等）の**内容**
- 個人ファイルの内容を取得するための再帰的ファイルスキャン、全文解析、サムネイル生成等
- クリップボードの履歴データ
- ゲーム環境と無関係なシステムファイル
- ユーザーの行動分析・操作ログ
- 明示的Opt-inなしの個人ディレクトリ監視、Canary配置、保護判定

---

# 13. Security Event Response (自動処罰の禁止 - 完全復元)

## 13.1 Detection Rule
脅威検知時の基本シーケンス：
```text
Detect ➔ Record Evidence ➔ Notify User ➔ Provide Action Options
(検知 ➔ 証拠記録 ➔ ユーザー通知 ➔ 対処オプションの提示)
```

## 13.2 Automatic Punishment Prohibition (短絡的自動処罰の禁止)
検知結果のみを理由として、以下を自動実行することを厳禁とする：
- ファイルの自動削除（Auto Delete）
- 通信の無断強制ブロック（Silent Block）
- セーブデータの自動書き換え（Auto Repair）

理由：誤検知によるゲーム環境およびユーザー資産の破壊を防ぐため。

---

# 14. Privacy and UI Transparency (UI 透明性要件 - 完全復元)

## 14.1 Explanation Requirement (説明責任)
GST は、重要操作において以下をユーザーへ明確に説明しなければならない：
- `What Data` (どのデータにアクセスするか)
- `Why Accessed` (なぜ必要なのか)
- `How Used` (どのように使用・処理されるか)
- `Where Stored` (どこに保存されるか)

## 14.2 Hidden Operation Prohibition (隠蔽操作の完全禁止)
以下を製品全体で厳格に禁止する：
- ユーザーに隠れて行うバックグラウンドデータ収集
- ユーザーの認知なしに行う設定のサイレント変更
- ユーザーの同意なしに行う外部アップロード

---

# 15. Relationship With Other Models (他モデルとの関係性 - 完全復元)

## 15.1 Security Boundary
操作境界と権限モデル：
```text
02_Security_Boundary_and_Protection_Model.md
```

## 15.2 Audit & Evidence
証跡と改ざん検知：
```text
04_Audit_and_Evidence_Model.md
```

## 15.3 Backup & Data Ownership
データ所有権とポータブル暗号化：
```text
05_Save_Backup_and_Data_Ownership_Model.md
```

## 15.4 PC Migration & Recovery
認証移行とアトミック再封緘：
```text
09_PC_Migration_and_Recovery_Model.md
```

## 15.5 AI Integration & Consent
BYOK Gemini AI 連携仕様：
```text
02_Features/Security/Trust_Enhancement_Spec/04-06_Gemini_Ai_Integration_and_Consent_Spec.md
```

---

# 16. Final Privacy Statement

GST は、ユーザーの環境と資産を守るために存在する。 
しかし、ユーザーの個人情報を収集・プロファイリングするために存在することは決してない。

---

Final Principle:

```text
Local By Default
Private By Design
Sanitized Before Sharing
Never Punish Automatically
Transparent to Users Always
```

---

