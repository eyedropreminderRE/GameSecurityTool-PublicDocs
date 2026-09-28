# GameSecurityTool

# Regression Prevention and Final Baseline Checklist

## Quality Assurance / Security Regression / Release Validation Definition

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-008 |
| Version | 3.1 (Contracts ThreatReasonCodes SSOT Architecture-Test Edition) |
| Status | Formal Baseline Specification (Highest Quality Authority) |
| Category | Regression Prevention and Release Governance |
| Authority Level | Final Validation Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- 回帰防止（Regression Prevention）
- 既存安全機能の保護
- Security Regression 防止
- Clean 5-Layer アーキテクチャ整合性検証
- リリース前最終ゲート基準

を定義する。

---

新機能追加によって既存の安全性・復旧可能性・パフォーマンスを低下させてはならない。

基本原則：

```text
New Feature must not Break Existing Protection
(新機能は既存の保護を破壊してはならない)
```

---

# 1. Regression Prevention Philosophy

## 1.1 Core Principle

GST の品質保証方針：

```text
Preserve Existing Behavior Before Adding New Capability
(新機能の追加前に既存の振る舞いを保護する)
```

機能追加時における既存ルールの暗黙的な無効化・緩和・サイレントな Fail-Open を厳禁とする。

---

# 2. Architecture & Layer Regression Checklist

## 2.1 NetArchTest 自動検証 (CI 必須ゲート)
- [ ] `Domain` 層が `System.IO`, `Microsoft.Win32`, `System.Management`, `EF Core`, `WPF` を参照していないこと（Pure C#）。
- [ ] `Domain` 層が `GameSecurityTool.Contracts` プロジェクト（DTO / Enum / Port）を参照していないこと。
- [ ] `Application` 層が OS 固有具象 API（WMI, COM, Registry 等）や `System.IO` 物理操作を直接行わず、Contracts の Port 経由で調停していること。
- [ ] `Presentation` (UI) 層から `Infrastructure` への直接参照・直接呼出が存在しないこと。
- [ ] `SqliteDatabaseWriter` 以外のクラスから `DbContext.SaveChangesAsync()` が直接呼ばれていないこと。
- [ ] **【L-1 是正規約】`GST.Infrastructure` 内のクラスが、Application 層のマッピング Port（`IRiskAssessmentService` 等）を経由せず、`GameSecurityTool.Domain.Models.*` サブ名前空間の型を直接インスタンス化・参照してマッピングを肩代わりしていないこと。**
- [ ] **【SSOT 整合規約】`TC-ARCH-SSOT-01` により、`GameSecurityTool.Contracts.Common.ThreatReasonCodes` が唯一の Threat Reason Code レジストリとして維持され、公開文字列定数が空・重複せず、`ThreatReasonDto.ThreatReasonCode` が `string` を使用し、Domain / Contracts に競合する `ThreatReasonCode` enum が存在しないこと。**

---

# 3. Security & Permission Regression Checklist

## 3.1 権限モデルと特権分離
- [ ] Main GUI プロセスが常に Standard User 権限で起動・動作していること。
- [ ] 管理者権限を要する Firewall 操作等が、DACL 保護された Named Pipe 経由で Out-of-Process UAC Worker へ正しく委譲されていること。
- [ ] ユーザーが UAC 昇格をキャンセルした場合に、30秒間のバックオフタイマーが作動し、プロンプトの連発が発生しないこと。
- [ ] Firewall ルール所有権判定において、プレフィックス文字列だけでなくローカル SQLite DB との突合が必須化されていること（Foreign Rule の誤爆削除防止）。
- [ ] 127.0.0.1 免除において、公式ゲームでは通信を許容し、フリー/同人ゲームおよび厳格モード時には確実に完全遮断する分岐が機能していること。

## 3.2 ライフサイクル・監視可用性と Fail-Open 根絶
- [ ] **【C-6 是正規約】WMI プロセス監視ウォッチャーが障害停止した際、ログ警告のみで放置されず、ダッシュボードおよびオーバーレイへ「⚠️ プロセス監視エンジン停止（保護無効）」が即時伝播されること。**
- [ ] プロセス作成時刻検証が、`eventTimeUtc ± 3秒` の対称許容ウィンドウで正しく機能し、正常プロセスの誤遮断および PID 使い回しプロセスの誤認が防止されていること。
- [ ] プロセスオープンから検証・終了まで同一の `SafeProcessHandle` 上で実行され、TOCTOU が排除されていること。
- [ ] ゲーム終了時のポスト監査により、レジストリ `Run` キーへの不正登録および `hosts` ファイルの改ざんが検知され、ワンクリック復元が機能すること。
- [ ] 主要ランチャー本体フォルダに対する越境改ざん監視（`ILauncherSecurityAdapter`）が機能し、コアファイルの完全性が保護されていること。

---

# 4. Data Protection & Recovery Regression Checklist

## 4.1 Quarantine (暗号化隔離 ＆ PC 移行再封緘)
- [ ] 巨大ファイル隔離時に一括メモリ展開（`new byte[fileInfo.Length]`）を行わず、64KB チャンク分割ストリーミング AEAD が機能していること（OOM 防止保証）。
- [ ] 隔離コミット順序が「①一時コンテナ出力 ➔ ②アトミック移動 ➔ ③DB コミット ➔ ④元ファイル安全消去」を遵守し、DB コミット失敗時に一時コンテナが即座にロールバック消去されること。
- [ ] 復元時に `RestoreTransaction`（二段階コミット）が機能し、ReadOnly 属性による例外を防ぐ順序（ACL ➔ タイムスタンプ ➔ 属性）で復元され、失敗時に `PartialFailed` が正しく記録されること。
- [ ] **【C-4 是正規約】PC 移行時の隔離コンテナ再封緘（`ImportAndResealQuarantineAsync`）において、既存コンテナを直接上書きせず、`.reseal.tmp` 経由のアトミック置換で新 DPAPI 鍵が再封緘されること。**
- [ ] **【H-2 是正規約】マイグレーションマスターキー（`byte[]`）の所有権が Caller にあり、全処理完了後に `finally` で `ZeroMemory` されること。**

## 4.2 Save Backup (セーブデータバックアップ ＆ タイムトラベル)
- [ ] バックアップ形式が Standard ZIP 互換であり、GST なしでも外部ファイル圧縮ソフトウェア等の標準ツールで展開可能であること（No Vendor Lock-in）。
- [ ] パスワード保護がポータブル（Argon2id + AES-256）であり、端末固定 DPAPI 単体に依存していないこと。
- [ ] 自動バックアップはパスワードなし（平文 ZIP / ACL 保護）で実行され、メモリ上に平文パスワードが常時常駐しないこと。
- [ ] Debounce バッファ内でリクエストが上書き置換された際、旧パスワード配列が即座に `ZeroMemory` されること。
- [ ] 差分検知（Fast Metadata Check）により、変更のないファイルに対する再コピー・フルハッシュ計算がスキップされていること。
- [ ] **【N+1 解消是正規約】`SaveBackupMaintenanceCoordinator` が `ISaveBackupRepository.GetReferencedSourceBackupIdsAsync` によるバッチクエリを使用し、ループ内個別 DB クエリが根絶されていること。**
- [ ] **【エントロピー検知】セーブデータのシャノン・エントロピー急変を検知した際、Snapshot Freeze が発動してバックアップの自動パージが緊急凍結されること。**
- [ ] **【二層整合性規約】複数ファイル操作において、ファイル単位の物理アトミック置換（.tmp -> File.Move）と起動時再実行による全体整合性保証が機能していること。**
- [ ] **【C-3 是正規約】セーブデータ復元ロールバック（`RollbackAsync`）において、ユーザーファイルを直接上書きせず、`.rollback.tmp` 経由のアトミック移動で元の状態へ安全復元されること。**
- [ ] **【H-6 是正規約】退避途中でクラッシュした未完成の `{transactionId}.creating` フォルダが、起動時 GC により安全に自動パージされること。**
- [ ] `IsPinned == true` のスナップショット、および他スナップショットから差分参照されているコンテナが自動削除から確実に除外されていること。
- [ ] ドライブ空き容量が不足している場合（`MinimumFreeSpace` 未満）、新規書き込みを安全中断して既存スナップショットを保護すること。

---

# 5. Privacy & Audit Regression Checklist

## 5.1 監査ログと改ざん検知多層アンカー
- [ ] 創世記ハッシュを起点とする SHA-256 ハッシュチェーン連鎖が完全に検証されること。
- [ ] **【Layer 2 Fail-Closed 是正規約】`TamperEvidentAuditLogger.VerifyAuditChainIntegrityAsync` において、Layer 2（特権管理者アンカー）が未同期または不整合の場合、不確定な承認を排して確実に `false`（Fail-Closed）を返すこと。**
- [ ] 全ログ書き込みロック（`_chainLock`）内で同期的な UAC 呼び出しを行わず、Layer 2 アンカーがバックグラウンドチャネルで非同期同期されること。
- [ ] Universal Privacy Shield（`ILogSanitizer`）により、各種 Webhook、OAuth/Bearer トークン、PC 名、MAC アドレス、メールアドレスが自動完全マスキングされること。
- [ ] AI 送信前プロンプト確認（Prompt Preview）により、マスキング済み送信プロンプトの事前確認およびキャンセルが可能であること。

## 5.2 Web Link & クリップボード
- [ ] Strict Query Stripping により、`?` 以降のクエリパラメータがデフォルトで完全破棄されていること（末尾 `?` 残存ゼロ）。
- [ ] ホスト判定において部分一致（`Contains`）が排除され、完全一致またはドット境界サブドメイン一致が保証されていること。
- [ ] 公式サポートドメイン厳格照合により、主要ゲームプラットフォームの公式ドメインが保護され、Punycode 偽装ドメインが即時 Block されること。
- [ ] 生 IP アドレスリテラル（`192.168.1.1` 等）の登録と評価が完全に対称一致すること。
- [ ] クリップボードの常時監視・自動置換・履歴保存が一切行われていないこと（On-Demand 厳守）。
- [ ] 監査ログに生の URL、Query String、認証トークン、クリップボード本文が出力されていないこと。

## 5.3 OS 設定保護 (WER テレメトリ遮断 ＆ アンインストール)
- [ ] `DontSendAdditionalData = 1` 適用前に変更前のレジストリ値（BeforeState）がディスクジャーナルへ保存され、終了時に元の状態へ正確に書き戻されていること。
- [ ] 完全アンインストール時、未復元の隔離ファイルが救済され、`SqliteConnection.ClearAllPools()` により接続プールが完全切断された後に内部データが消去されること。
- [ ] OS 自動起動オプションが既定 OFF（オプトイン）であり、アンインストール時に `Run` キー登録が確実に削除されること。

---

# 6. Release Final Quality Gate (リリース判定チェックリスト)

リリースビルド作成前に以下をすべて確認・クリアする。

### 1. ビルド・静的解析
- [ ] Visual Studio / CI において Compile Error = 0
- [ ] Nullable Warning = 0 (`<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`)
- [ ] `GameSecurityTool.ArchitectureTests` (NetArchTest ルール 1〜6 および `TC-ARCH-SSOT-01`) が全件パス

### 2. テスト検証
- [ ] すべての Unit Tests, Integration Tests, Security Tests が全件パス
- [ ] Layer 2 未同期時の Fail-Closed 動作テストがパス
- [ ] 巨大ファイル（4GB超）処理時のプライベートメモリ資源基準を各ワークロードの承認済みresource gateに従って満たすこと（Save Backup single-file payload workloadは承認済みのmatched-control resource gate <=16MBを適用。その他のworkloadは既存基準を維持）
- [ ] SQLite マルチスレッド同時書き込み時のロック競合（`SQLITE_BUSY`）ゼロ検証
- [ ] ロールバック中クラッシュ耐性（`TV-SEC-BAK-03`）および再封緘中クラッシュ耐性（`TV-SEC-QRT-05`）の検証成功

### 3. ドキュメント整合
- [ ] 仕様変更に伴う `00_Baseline/`, `01_Architecture/`, `modules/`, `02_Features/` の完全同期

---

# 7. Final Quality Statement

GST の品質基準：

```text
Secure By Design
Zero OOM & Zero Lock-in
Explainable & Non-Destructive
Recoverable Always
```

---
