# GameSecurityTool

# Data Integrity and Forensics Model

## Data Validation / Evidence Preservation / Recovery Investigation Specification

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-043 |
| Version | 4.0 (Full 28 Sections Restored, Referential Integrity, Atomic Recovery & Multi-Layer Audit Forensics Master Edition) |
| Status | Formal Baseline Specification (Highest Data Integrity Authority) |
| Category | Data Security Architecture |
| Authority Level | Integrity Governance |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- データ完全性（Data Integrity）管理基準
- 差分参照整合性およびスキーマ保護制約（`DeleteBehavior.Restrict`）
- 決定論的マニフェストハッシュ計算（秒精度正規化）
- 改変検知、分類、フォレンジック証跡保全（Multi-Layer Audit Chain）
- 全28章にわたるデータ保護規約（移行完全性、外部改変検知、DB完全性、暗号完全性、Zero Trust、AI開発規約）
- 障害復旧・アトミックロールバック時の完全性保証

を定義する。

---

目的：

```text
Protect Important Data
*
Detect Unexpected Changes
*
Guarantee Zero Corruption on Rollback
*
Support Reliable Recovery & Forensics
```

---

# 1. Data Integrity Philosophy

## 1.1 Core Principle
GST では、単にデータを保存するだけでなく、保存された状態の信頼性・完全性・復元可能性を永続的に維持する。

基本原則：
```text
Valid Data        : 破損や不正改変のない真正なデータ
Traceable History : OperationId / TransactionId による追跡可能な履歴
Recoverable State : 障害時にも 100% 直前状態へ戻せるアトミック復旧性
```

---

# 2. Integrity Model (完全性評価階層)

データ信頼性は以下の 5 段階で厳格に評価する。

```text
1. Existence    : 物理ファイル・コンテナが存在すること
2. Structure    : Standard ZIP / QRTv03 ヘッダー構造が正当であること
3. Consistency  : スキーマ・外部キー制約 (Restrict) および参照グラフが整合していること
4. Authenticity : SHA-256 ハッシュおよびマニフェストハッシュが完全一致すること
5. Usability    : 権限 (ACL) および属性が適用され、ゲームから正常にロード可能であること
```

---

# 3. Protected Data Classification (保護データ分類)

```text
1. Game Save Data    : ユーザーの最重要資産 (原本保護 ＆ RescueSnapshot 退避)
2. Backup Containers : Standard ZIP + Argon2id + 64KB Chunked AEAD
3. Quarantine Files  : QRTv03 コンテナ ＆ DPAPI 暗号化鍵 (アトミック再封緘)
4. Audit Records     : SHA-256 連鎖 Hash Chain ＆ 特権多層アンカー
5. Migration Package : .gstmgr パッケージ ＆ フォーマットバージョン整合
6. Security Evidence : PE 構造解析シグナル, Zone.Identifier 出所データ
```

---

# 4. Data Ownership & Non-Destructive Principle

* **所有権:** セーブデータおよびバックアップはユーザーの完全な所有物である。
* **非破壊性:** 完全性検証を理由として、破損が疑われるセーブデータを勝手に自動削除・上書きしてはならない。

---

# 5. Integrity Validation Pipeline (完全性照合パイプライン)

重要データの検証項目：
1. **File Existence & Size:** 物理ファイルの存在とバイト数の確認。
2. **Format Validity:** Standard ZIP / QRTv03 ヘッダーマジックバイトの検証。
3. **Checksum & AEAD Tag:** 64KB チャンクごとの AEAD 認証タグ（16B）および SHA-256 ハッシュの照合。
4. **Metadata Consistency:** タイムスタンプ、ファイル数、所有権メタデータの整合性確認。

---

# 6. Hash Verification Model (ハッシュ検証モデル)

重要データは SHA-256 ハッシュにより改ざんおよび破損を検知する。
- **対象:** バックアップコンテナ, 隔離コンテナ, PC 移行パッケージ, 監査レコード, 重要設定ファイル

---

# 7. Metadata Integrity (メタデータ完全性 - 完全復元)

管理情報の検証項目：
- 作成日時（`CreatedAt`）および最終更新日時（`LastWriteTimeUtc`）の秒精度正規化（`ToUnixTimeSeconds()`）
- スキーマバージョン番号および生成元識別子（`SourceContainerBackupId`）
- パス区切り文字のスラッシュ（`/`）への統一による決定論的 `ManifestHash` の算出

---

# 8. Backup Integrity Model (バックアップ完全性 ＆ 差分参照保護 - CRIT-05 連携)

バックアップの成功条件は、単なるファイルコピーの完了ではない。

必須条件：
```text
Copy Success (完全なストリーミング出力)
*
Validation Success (決定論的 ManifestHash の完全一致)
*
Restore Possibility (Argon2id パスワードによる復号可能性の保証)
```

### 差分参照保護規約 (CRIT-05 連携):
- 差分バックアップにおいて、他スナップショットから参照されているコンテナのメタデータ削除を `DeleteBehavior.Restrict` 制約により RDBMS レベルで物理拒否する。
- 物理コンテナ GC において、作成後 1 時間以内の新規コンテナをグレースピリオドにより誤爆削除から保護する。

---

# 9. Restore Verification (復元前検証)

復元実行前に以下を必ず確認する：
1. バックアップアーカイブおよび `GST_Manifest.json` の正当性
2. 復元先ディレクトリの TOCTOU / Reparse Point 境界
3. 保存先ドライブの空き容量（`Storage Safety Guard` 2GB 閾値）
4. 復元による影響範囲のユーザー事前プレビュー

---

# 10. Migration Integrity (PC 移行完全性 - 完全復元 ＆ C-4, H-2 連携)

移行処理の完全性保証：
1. **移行前検証:** 旧 PC データの完全性とフォーマットバージョン互換性照合。
2. **中間暗号化完全性:** Argon2id マスターキーによる AES-256-GCM 暗号化の検証。
3. **アトミック再封緘 (C-4 是正):** 新 PC での隔離コンテナ再封緘時、既存ファイルを直接上書きせず、一時ファイル（`.reseal.tmp`）経由のアトミック置換（`File.Move`）を実行。
4. **鍵消去の徹底 (H-2 是正):** `MigrationCoordinator` が移行完了後に `migrationMasterKey` を `ZeroMemory` 消去。

---

# 11. Migration Evidence (移行証跡記録 - 完全復元)

PC 移行時に以下を監査ログに記録する：
- 元バージョンおよび対象バージョン
- 移行結果サマリー（プロファイル数、バックアップ数、隔離項目数）
- 障害発生時の失敗ステップ (`FailedStep`)

---

# 12. Data Change Detection (データ変更検知 - 完全復元)

検出対象：
- 予期せぬファイル更新・差し替え
- 外部ツールや手動操作による変更
- ファイル破損および中途半端な更新
- 未署名バイナリの新規配置

---

# 13. Change Classification (変更分類 - 完全復元)

検知された変更の分類：
1. **Expected Change:** ゲーム実行による通常のセーブデータ保存
2. **User Change:** ユーザーによる手動設定変更や MOD 導入
3. **Application Change:** GST によるアトミックバックアップや復元
4. **Unknown Change:** 発信元不明の不審なバイナリ書き出し（アラート対象）

---

# 14. Forensic Evidence Model (フォレンジック証拠保全)

調査・説明のために保存する証跡データ：
- タイムスタンプ (UTC 秒精度)
- 実行された操作および操作主体 (`Actor`)
- 判定理由コード (`ThreatReasonCode`)
- PE プロキシ構造解析シグナルおよび `Zone.Identifier` 出所情報
- イベント発生時の `ConfigurationSnapshot`

---

# 15. Evidence Preservation Principle (現状保全の原則 - 完全復元)

証拠データは改変しない。

原則：
```text
Preserve Before Analyze
(分析・修復を開始する前に、原本証拠の現状を完全に保全する)
```

---

# 16. Incident Investigation Flow (障害調査標準フロー)

```text
Detect Issue (異常・クラッシュ検知)
       ↓
Collect Evidence (サニタイズ済み証拠データの収集)
       ↓
Analyze Root Cause (根本原因分析 ＆ ユーザーへの平易な解説)
       ↓
Recover State (RescueSnapshot / バックアップからのアトミック復旧)
       ↓
Prevent Recurrence (回帰防止テスト追加 ＆ ベースライン強化)
```

---

# 17. Corruption Handling & Atomic Rollback (破損処理 ＆ アトミック復元 - C-3 連携)

セーブデータやバックアップの破損検知時：
1. 安全でない操作（破損データの上書き等）を直ちに停止。
2. 原本セーブデータを `RescueSnapshot` へ一時退避。
3. **アトミックロールバック:** 復元失敗時、ユーザーファイルを直接上書きせず、必ず一時ファイル（`.rollback.tmp`）経由のアトミック置換（`File.Move`）で直前状態へ巻き戻す。

禁止事項：
```text
Overwrite Before Backup
(バックアップ・一時退避を行わずに既存データを上書きすることを厳禁とする)
```

---

# 18. Recovery Priority (復旧優先順位 - 完全復元)

意思決定の優先順位：
```text
1. Preserve User Data (現行のユーザー資産をこれ以上破壊しない)
2. Restore Safe State (安全な直前状態へアトミックにロールバックする)
3. Improve Prevention (再発防止策をテストピラミッドへ組み込む)
```

---

# 19. Secure Deletion Principle (安全削除原則 - 完全復元)

削除処理は最高リスク操作として扱い、必ず以下を義務付ける：
- 削除対象の一覧プレビュー表示
- ユーザーの明示的承認ダイアログ
- 監査ログ記録 (`AuditEventType.SaveBackupDeleted` 等)
- 差分参照依存性（`DeleteBehavior.Restrict`）の事前検証

---

# 20. External Modification Detection (外部改変検知 - 完全復元)

第三者ツール、手動エディタ、システムプロセスによるゲームフォルダやレジストリの変更をゲーム終了時ポスト監査で検知し、改ざんを可視化する。

---

# 21. Database Integrity (データベース完全性 - 完全復元)

SQLite データベースの整合性維持：
- `PRAGMA journal_mode = WAL;` および `PRAGMA synchronous = NORMAL;` の強制
- `IDbWriteQueue` による単一ライター直列化コミット
- Enum 追記専用原則（Append-Only Rule）の遵守

---

# 22. Encryption and Integrity (暗号化と完全性の分離管理 - 完全復元)

暗号化利用時、以下の 3 要素を明確に分離して管理する：
```text
Confidentiality (機密性: Argon2id + AES-256-GCM による暗号化)
Integrity       (完全性: 64KB チャンク別 AEAD AuthTag ＆ SHA-256 マニフェスト)
Key Management  (鍵管理: ZeroMemory 即時消去 ＆ Caller-Owns 契約)
```

---

# 23. Audit Integration & Multi-Layer Anchor (監査統合 - C-5 連携)

フォレンジック調査のための改ざん検知多層検証：
1. **SQLite 内 Hash Chain 連鎖検証 (GenesisHash 起点)**
2. **Layer 1 (ローカル DPAPI `audit.dpapi`) ハッシュ照合**
3. **Layer 2 (特権管理者 `audit.anchor`) ハッシュ照合**
   - ※ 未同期時はサイレント成功（Fail-Open）とせず、`AuditVerificationResultDto` を通じて劣化状態を可視化する。

---

# 24. Zero Trust Integration (Zero Trust 統合 - 完全復元)

外部データおよび内部データファイルを無条件に信用しない。
- 発信元（`Origin`）、完全性（`Integrity`）、実行コンテキスト（`Context`）を常に検証してから処理する。

---

# 25. AI Development Rule (AI 開発規約 - 完全復元)

AI コーディングエージェントがデータ処理コードを変更する際の必須確認：
- ユーザーデータの消失リスクがゼロか
- アトミック変更パターン（`.tmp` ➔ `File.Move`）が遵守されているか
- 破壊的データ操作を直接行っていないか

---

# 26. Testing Requirement (テスト要件)

データ完全性テストマトリクス：

☐ `TC-INT-01`: チャンク別 Nonce (12B) の一意性検証（Nonce Reuse ゼロ）  
☐ `TV-SEC-BAK-03`: `RollbackAsync` 中の電源断時における元ファイル無傷検証（アトミック性）  
☐ `TV-SEC-QRT-05`: `ImportAndResealQuarantineAsync` 中の例外発生時におけるコンテナ無傷検証  
☐ `TV-SEC-AUD-02`: Layer 1 / Layer 2 アンカー不一致および未同期時の改ざん・劣化検知検証  
☐ 差分参照元コンテナの誤削除拒否（`DeleteBehavior.Restrict`）検証  

---

# 27. Relationship With Other Models (他モデルとの関係性 - 完全復元)

## 27.1 Backup & Data Ownership
データ所有権およびポータブル暗号化：
```text
05_Save_Backup_and_Data_Ownership_Model.md
```

## 27.2 Disaster Recovery & Continuity
災害復旧および No Vendor Lock-in 原則：
```text
Operations/26_Disaster_Recovery_and_Business_Continuity_Model.md
```

## 27.3 PC Migration & Recovery
PC 移行とアトミック再封緘：
```text
09_PC_Migration_and_Recovery_Model.md
```

## 27.4 Database Migration & Schema
DB スキーマおよび Enum 追記専用規約：
```text
Architecture/12_Data_Model_and_Database_Schema.md
Architecture/29_Database_Migration_and_Schema_Evolution_Model.md
```

## 27.5 Audit & Forensics
監査証跡およびエビデンス保全：
```text
04_Audit_and_Evidence_Model.md
```

## 27.6 Zero Trust Security
Zero Trust 境界および操作検証：
```text
Security/42_Zero_Trust_Security_Model.md
```

---

# 28. Final Data Integrity Statement

GST におけるデータ保護とは、単にチェックサムを計算することではない。

---

**どんな障害・クラッシュ・攻撃が発生した場合でも、ユーザーの資産を絶対に破損させず、確実に元の状態へ復旧できる安全基盤を提供することである。**

---

Final Principle:

```text
Verify Every Hash Reliably
Mutate Files Atomically Always
Enforce Referential Integrity by Restrict
Preserve Forensics Evidence Untampered
```

---

End of Document
```

---
