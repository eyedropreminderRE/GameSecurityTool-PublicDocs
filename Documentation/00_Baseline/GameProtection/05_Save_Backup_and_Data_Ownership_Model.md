# GameSecurityTool

# Save Backup and Data Ownership Model

## User Data Preservation / Backup / Restore Definition

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-005 |
| Version | 3.1 (Full 17 Sections Restored, Inviolable Retention Rules & Complete Dual-Fusion Master Edition) |
| Status | Formal Baseline Specification (Highest Data Ownership Authority) |
| Category | Data Ownership and Backup Management |
| Authority Level | Core Data Protection Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- セーブデータ保護哲学
- データ所有権原則（No Vendor Lock-In）
- Backup / Restore 設計境界
- ポータブル暗号化（Argon2id + AES-256）と DPAPI ローカル保護の境界分離
- タイムトラベル（ピン留め・タグ）および不可侵保持ルール（4大原則）
- 復元安全性（RescueSnapshot アトミックロールバック）
- 手動復元（Manual Restore）、クリーンアップ、UI モデル
- PC 移行（Migration）および災害復旧（DR）連携

を定義する。

---

GST の基本思想：

```text
Protect User Data
without Taking Ownership
(ユーザーデータを保護するが、所有権は決して奪わない)
```

である。

---

# 1. Data Ownership Principle

## 1.1 User Data Ownership
GST が扱うゲーム関連データは、原則としてユーザーの完全な所有物（資産）である。

対象：
```text
Save Data
Game Configuration (.ini, .cfg, .json)
User Created Content & Mods
Backup Archive (Standard ZIP)
```

GST は、これらの所有権を取得・独占・囲い込み（Lock-in）しない。

---

## 1.2 GST Responsibility
GST の責務：
```text
Locate (保存場所の特定)
Identify (安全性の検証)
Backup Assist (非破壊・増分バックアップ支援)
Restore Assist (RescueSnapshot による安全な復元支援)
Preserve Reference (メタデータ・履歴の保護)
```

GST の非責務（禁止事項）：
```text
Own Data (データの私有化)
Control Data (無断でのデータ改変・自動削除)
Upload Data (クラウドへの自動送信)
Lock Data (GST 専用形式への閉じ込め)
```

---

# 2. Backup Philosophy

## 2.1 Backup Definition
GST Backup とは、ユーザーデータの安全な複製および復旧支援である。

Backup は：
```text
Safety Copy (安全な予備データ)
```
であり、
```text
GST Internal Database (ツールの内部閉域データ)
```
ではない。

---

## 2.2 Backup Independence Principle (No Vendor Lock-In)
バックアップは、**GSTが起動できない環境やアンインストールされた後でも、公開されたバックアップ形式に対応する独立復旧手段によってユーザー自身がアクセス・復元可能でなければならない。** 汎用ZIPツールによる外側コンテナの展開可能性と、暗号化payloadの復号可能性は別の性質として扱う。

必須要件：
```text
Standard ZIP-Compatible Container
*
Portable Password Protection (Argon2id + AES-256-GCM-CHUNKED)
*
Documented Independent Recovery Format
*
User Accessible Storage
```

復旧境界：
```text
Backup ZIP
  ├─ Generic ZIP tool → outer container / unencrypted metadata
  └─ GST-compatible independent recovery implementation → encrypted payload plaintext
```

7-Zip / Explorer等の汎用ZIPツール単独で暗号化payloadを復号できることは保証しない。

---

# 3. Backup Package Model

## 3.1 Backup Package Definition
GST Backup Package は、Standard ZIP互換コンテナとしての可搬性と、暗号化payloadの公開形式仕様に基づく独立復旧可能性を備えた構造とする。

構成：
```text
Backup_<GameName>_<Timestamp>_<Id>.zip
├── GST_Header.json                   <-- 平文: ゲーム名, BackupId, 暗号化有無 (判別用)
├── GST_Integrity.json                <-- Argon2id KDF パラメータ (Salt, Iterations, Memory)
├── GST_Manifest.json                 <-- 全ファイルメタデータ & 決定論的 ManifestHash
└── Payload/                          <-- 暗号化/平文ファイルストリーム (64KB Chunked AEAD)
    ├── SaveData.dat
    └── Config.ini
```

---

## 3.2 GST Metadata Separation
GST 固有の管理情報は、アーカイブ内で明確に分離する。

理由：  
GSTを介さず外側ZIPコンテナを確認・展開する際、ユーザーが実セーブファイルとGST固有メタデータを区別して扱えるようにするため。暗号化payloadの復号には定義されたGST互換の独立復旧実装が必要であり、7-Zip / Explorer単独による直接復号は保証しない。

---

# 4. Standard ZIP Compatibility

## 4.1 Archive Format
バックアップアーカイブは、世界標準の **ZIP 互換形式** を採用する。

理由：
- OS 標準エクスプローラーでの展開対応
- 外部ファイル圧縮ソフトウェア等の汎用ツールでの自力展開可能性
- 10年以上の長期的保存性
- ベンダーロックインの完全排除

---

## 4.2 External Extraction Support
ユーザーは、必要に応じて外部ツールでStandard ZIPコンテナを検査・展開できる。暗号化payloadの平文復元には、公開形式に対応したGST互換の独立復旧実装を使用する。

```text
[ GST 本体に依存しない手動復旧フロー ]
バックアップ ZIP を取得
         │
         ▼
外側ZIPを汎用ZIPツール等で検査・展開
         │
         ▼
暗号化payloadの場合は、公開形式に対応したGST互換の独立復旧実装で Argon2id → AES-256-GCM-CHUNKED 復号
         │
         ▼
Payload/ 配下のセーブファイルをゲーム保存先へ配置 ➔ 復旧完了
```

---

## 4.3 外部アーカイバ連携 (Game Backup)
GST から外部ファイル圧縮ソフトウェアを呼び出してゲーム本体フォルダを安全にバックアップする。圧縮パラメータは GST から強制せずユーザーのプリセットを尊重する。誤操作を防止するため、「圧縮開始前の詳細設定ダイアログ表示」は **既定 ON** とする。

---

# 5. Backup Password Protection Model (ポータブル暗号化)

## 5.1 Purpose
バックアップアーカイブを外部ドライブやクラウドへ保管する際の機密性保護。

## 5.2 暗号化アルゴリズム仕様
端末に縛られないポータブル復元を実現するため、以下の標準アルゴリズムを採用する：
- **Key Derivation Function (KDF):** **Argon2id** (Iterations = 3, Memory = 64MB, Parallelism = 4, Salt = 16B)
- **Payload Cipher:** **AES-256-GCM-CHUNKED (64KB 固定長 Chunked AEAD)**
- **Nonce 管理:** チャンクごとに暗号論的乱数で 12B Nonce を生成（Nonce Reuse を物理遮断）

## 5.3 Password Ownership
Password はユーザーの完全な自己管理とする。  
GST はバックドア、マスターキー、パスワード復旧機能を提供しない（暗号論的強度の維持）。

---

# 6. Windows DPAPI Protection Boundary (ローカルメタデータ保護)

## 6.1 DPAPI の適用範囲
Windows DPAPI（`DataProtectionScope.CurrentUser`）は、**同一端末・同一ユーザーコンテキスト内でのみ復号可能なローカル秘密情報の保護** に限定して適用する。

適用対象：
```text
GST 内部データベース接続・整合性マーカー
ローカル監査ハッシュアンカー (Layer 1: audit.dpapi)
端末内隔離ファイル (Quarantine) の AES-256 鍵
ローカル保存された認証トークン・設定シークレット
```

## 6.2 バックアップと DPAPI の境界分離
* **禁止事項:** バックアップ ZIP の暗号化に DPAPI 単体を使用すること（他 PC で復号不能になり、No Lock-in 原則に違反するため）。
* **正しい構造:** バックアップ ZIP の暗号化は **Argon2id パスワード** を使用し、DPAPI はローカル設定の保護にのみ使用する。

---

# 7. Migration Relationship (移行連携 ＆ 再封緘)

## 7.1 Separation Model
```text
Migration Package (.gstmgr)  ──(references)──>  Backup Archive (Standard ZIP)
   [GST 環境の移送・再封緘]                         [ユーザー資産の長期保全]
```

## 7.2 独立性の保証
Migration Package 内に大容量セーブデータ実体を必ずしも内包させず、参照ポインタとして管理することでパッケージの肥大化を防ぐ。

## 7.3 PC 移行時の再封緘 (Re-sealing Pipeline - 完全復元)
新 PC への移行時は、`Migration Package` 経由で Argon2id 中間暗号化を用いて秘密情報を安全に移送し、新 PC のローカル DPAPI コンテキストでアトミックに再封緘する（`09_PC_Migration_and_Recovery_Model.md` 準拠）。

---

# 8. Migration Success Flow (正常移行シーケンス - 完全復元)

正常時の移行フロー：
```text
旧 PC で Migration Package (.gstmgr) をエクスポート (Argon2id 中間暗号化)
         │
         ▼
新 PC で GST をインストール ➔ パッケージをインポート
         │
         ▼
新 PC 上でバックアップ参照を自動再リンク ＆ DPAPI アトミック再封緘 (.reseal.tmp ➔ File.Move)
         │
         ▼
GST 環境再現完了 (ユーザー操作はパスフレーズ入力のみで最小限)
```

---

# 9. Recovery Flow (障害復旧フロー - 完全復元)

## 9.1 Migration Failure Recovery (移行失敗時復旧)
Migration Package が破損・紛失した場合でも、Standard ZIP バックアップとパスワードがあれば、GST へ手動再登録して使用を継続できる。

## 9.2 Password Required Case (パスワード要求ケース)
以下の場合、ユーザーにパスワード入力を求める：
- 手動バックアップインポート時
- Migration Package 紛失時
- 別 PC への手動バックアップ移送時

## 9.3 Password Lost Case (パスワード紛失時の規約)
パスワードを紛失した場合、GST は暗号のアンロックやクラックを行わない（暗号強度の保証）。ユーザーへ事実を通知し、セキュリティモデルを堅持する。

---

# 10. Manual Restore Model (GST本体非依存の手動復旧モデル)

## 10.1 Purpose
GST が動作不能な環境、OS再インストール時、PC故障時でも、Standard ZIPコンテナと公開形式仕様に基づく独立復旧手段を用いた自力復旧経路を保証する。汎用ZIPツール単独で暗号化payloadを復号することは保証しない。

## 10.2 Manual Restore Information (手動復旧に必要な情報)
UI およびマニュアルで以下を明示する：
- 元のゲームセーブフォルダ配置場所
- 復元に必要なファイル一覧
- GST 固有の除外ファイル一覧

## 10.3 GST Excluded Data (手動復元時除外データ)
手動復元時、以下の GST 固有ファイルはゲームディレクトリへ配置してはならない：
- GST Cache, GST Temporary Data, GST Internal Metadata (`GST_Header.json`, `GST_Integrity.json`, `GST_Manifest.json`)

---

# 11. Restore Safety Model (復元安全性 ＆ アトミックロールバック - 完全復元)

## 11.1 Restore Operation (アトミック復元パイプライン ＆ 二層整合性モデル)
復元は既存データを書き換える最高リスク操作であるため、二層整合性モデル（ファイル単位の物理アトミック置換と起動時再実行による全体整合性保証）を適用し、以下のパイプラインを厳格に強制する：

```text
[ ユーザーの明示的承認 (Explicit Consent) ]
                   │
                   ▼
1. [TOCTOU ＆ Reparse Point 境界検証]
                   │
                   ▼
2. [復元前一時退避 (RescueSnapshot)] ➔ アトミック作成 ＆ manifest.json 検証
                   │ (退避完全完了)
                   ▼
3. [64KB チャンクストリーミング復元 ＆ ハッシュ照合]
                   │
         ┌─────────┴─────────┐
      (成功)               (失敗)
         │                   │
         ▼                   ▼
    [正常完了]     [RescueSnapshot から直前状態へアトミックロールバック]
                   (一時ファイル .rollback.tmp 経由の差し替え)
```

## 11.2 Restore Conflict (既存データ競合解決)
復元先に既存データが存在する場合、サイレントな自動上書きを禁止する。ユーザーに確認ダイアログを表示し、明示的な承認を得て、現行データを `RescueSnapshot` へ一時退避した上で実行する。

---

# 12. Backup Validation Model (バックアップ検証モデル - 完全復元)

## 12.1 Integrity Check (完全性照合)
バックアップ生成時および復元前に、以下を検証する：
- アーカイブ構造の正当性
- `GST_Manifest.json` の決定論的ハッシュ突合 (秒精度正規化)
- ファイル数および合計バイト数の整合性

## 12.2 Validation Failure
検証失敗時、破損したバックアップからの自動復元を禁止し、ユーザーへ警告通知を表示する。

---

# 13. Backup Cleanup & Retention Policy (保持 ＆ クリーンアップ規約 - 完全復元)

## 13.1 Cleanup Principle
バックアップの削除は、ユーザー操作または明示的な保持ポリシー（Retention Policy）に基づいて実行する。
有効なバックアップをツールの独断でサイレント自動削除することを厳禁とする。

## 13.2 不可侵保持ルール (Non-Destructive Retention Rules - 完全復元)
いかなる自動パージ処理においても、以下の 5 大原則を絶対遵守する：
1. **最新世代の絶対保護:** いかなる容量制限・世代制限に達しても最新スナップショットは自動削除しない。
2. **ピン留め保護 (`IsPinned == true`):** ユーザーがピン留めしたスナップショットは自動削除から確実に除外（永久保持）。
3. **差分参照元保護:** 他スナップショットから参照されているコンテナは削除候補から除外（`DeleteBehavior.Restrict`）。
4. **1時間グレースピリオド:** 作成後 1 時間以内の新規コンテナは GC による誤爆削除から保護。
5. **ランサムウェア・エントロピー検知 (Snapshot Freeze):** セーブデータのシャノン・エントロピー急変（暗号化マルウェア等による急激な無秩序化）を検知した場合、健全なバックアップが世代管理によって上書き・削除されるのを防ぐため、自動パージ処理を緊急凍結（Freeze）する。

## 13.3 Cleanup Confirmation
手動削除時は、対象バックアップの保存場所、作成日時、サイズ、関連ゲーム名を明示して確認を求める。

## 13.4 N+1 クエリ解消による保持評価バッチ化
`SaveBackupMaintenanceCoordinator` は、スナップショット間の差分参照関係を評価する際、個別クエリをループ実行するのではなく、`ISaveBackupRepository.GetReferencedSourceBackupIdsAsync` を呼び出して一括取得（バッチ化）する。これにより DB 往復オーバーヘッドを排除し、数千世代のスナップショットが存在する場合でも高速かつ安全に世代判定を行う。

---

# 14. Backup UI Model (推奨 UI 表示モデル - 完全復元)

ゲーム単位の推奨表示構造：
```text
Game Profile
├ Backup Status (有効 / 無効 / SSD超低負荷モード)
├ Last Backup Time & Size (最終バックアップ詳細)
├ Storage Usage (通常世代 / 📌ピン留め世代の容量内訳)
├ Save Bloat Graph (セーブ容量推移グラフ)
├ Snapshot History (タグ・メモ・ピン留め一覧)
└ Restore & Export Options (↩️ 復元 / 📤 汎用 ZIP エクスポート)
```

---

# 15. Data Recovery Priority (データ復旧優先順位 - 完全復元)

復旧時の意思決定優先順位：
```text
1. Preserve Existing User Data (現行のユーザーデータを破壊しない)
2. Avoid Destructive In-Place Mutation (アトミック置換を徹底する)
3. Provide Self-Recovery Path (GST 非依存の自力展開手段を保証する)
4. Record Audit Evidence (すべての復元・ロールバック証跡を記録する)
```

---

# 16. Relationship With Other Models (他モデルとの関係性 - 完全復元)

## 16.1 Security Boundary
操作境界と権限モデル：
```text
02_Security_Boundary_and_Protection_Model.md
```

## 16.2 Game Environment Protection
ゲーム環境保護モデル：
```text
03_Game_Environment_Protection_Model.md
```

## 16.3 Audit & Evidence
監査証跡と改ざん検知：
```text
04_Audit_and_Evidence_Model.md
```

## 16.4 PC Migration & Recovery
PC 移行とアトミック再封緘：
```text
09_PC_Migration_and_Recovery_Model.md
```

## 16.5 Data Architecture
データモデルおよびスキーマ保護制約 (Restrict)：
```text
Architecture/12_Data_Model_and_Database_Schema.md
```

---

# 17. Final Backup Statement

GST のバックアップ機能は、ユーザーのセーブデータを守るために存在する。  
しかし、GST がユーザーデータの所有者になることは決してない。

---

Final Principle:

```text
User Owns Data Always
GST Protects Data Non-Destructively
Standard ZIP & Argon2id Guarantee Portability
Never Lock-in User Assets
```

---

