# GameSecurityTool

# Quality Assurance and Release Gate Model

## Quality Assurance / Validation / Release Decision Specification

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-027 |
| Version | 3.1 (ThreatReasonCodes SSOT Architecture-Test Alignment Edition) |
| Status | Formal Baseline Specification (Highest Quality Gate Authority) |
| Category | Quality Assurance |
| Authority Level | Release Governance Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- 品質保証（Quality Assurance）方針
- 段階的品質ゲート概念
- リリース分類（Major, Minor, Patch）
- 全25章にわたる個別品質ゲート（Code, Functional, Backup, Restore, Migration, Security, Firewall, Regression, Performance, Compatibility, Documentation, User Impact, RC, Emergency）
- リリース承認基準、証跡管理、公開後監視、およびロールバック規約

を定義する。

---

目的：

```text
Prevent Defective Release
*
Protect Existing Users & Assets
*
Guarantee Zero Corruption on Failure
*
Maintain Absolute Product Trust
```

---

# 1. Quality Philosophy

## 1.1 Core Principle
GST では、「新機能の実装完了」をリリース条件としない。

正式リリース条件：
```text
All Functions Work Correctly
*
User Assets Are Safely Protected (No Lock-In)
*
Security Boundaries Are Strictly Enforced
*
Zero Silent Fail-Open Guaranteed
*
Atomic Recovery & Rollback Are Proven
```

---

# 2. Quality Gate Concept (段階的品質ゲート概念)

リリース前には以下の段階的確認を厳格に通過する。

```text
Development Complete (完全実装・TODOゼロ)
       ↓
Code Review & Static Analysis (Nullable警告ゼロ)
       ↓
Automated NetArchTest & Unit/Integration Tests (All Pass)
       ↓
Security & Resilience Review (Fault Injection / OOM 耐性検証)
       ↓
Release Candidate Validation (新規・移行実証)
       ↓
Formal Release Approval (三者承認)
```

---

# 3. Release Classification (リリース分類)

## 3.1 Major Release (メジャーリリース: X.0.0)
- **対象:** アーキテクチャ刷新, データ互換性変更, セキュリティ境界変更。
- **要件:** 全 25 品質ゲートの通過 ＆ アーキテクト署名。

## 3.2 Minor Release (マイナーリリース: 0.X.0)
- **対象:** 新機能追加, UI 改善, パフォーマンス向上。
- **要件:** Gate 1〜22 の通過 ＆ リードエンジニア承認。

## 3.3 Patch Release (パッチリリース: 0.0.X)
- **対象:** バグ修正, 緊急セキュリティパッチ。
- **要件:** 修正の局所化と回帰テストの全件合格。

---

# 4. Code Quality Gate (コード品質ゲート - 完全復元)

必須確認項目：
- [ ] Visual Studio / CI において Compile Error = 0
- [ ] Nullable Warning = 0 (`<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`)
- [ ] 未実装コメント（`// TODO:`, `/* 後で実装 */`, 擬似コード, 省略記号）が 0 件であること
- [ ] **`NetArchTest.Rules` 1〜6（Domain/Application/Presentation/SQLite Writer/Infrastructure 型結合規約）および `TC-ARCH-SSOT-01`（Contracts ThreatReasonCodes SSOT テスト）が全件パスしていること**

---

# 5. Functional Quality Gate (機能品質ゲート - 完全復元)

確認対象：
- ゲーム登録, スキャン, 隔離, 復元, バックアップ, Firewall 制御, WebLink 保護, LNK 改ざん検知, MOD 出所診断
- すべての機能が仕様書通りに動作することを実証。

---

# 6. Backup Quality Gate (バックアップ品質ゲート - 完全復元)

必須確認項目：
- [ ] バックアップ形式が Standard ZIP 互換であり、GST なしでも外部ファイル圧縮ソフトウェア等で自力展開できること（No Vendor Lock-in）
- [ ] パスワード暗号化（Argon2id + AES-256-GCM）が正しく動作し、自動バックアップ時に平文パスワードがメモリ常駐しないこと
- [ ] 差分検知（Fast Metadata Check）により、無変更ファイルの再ハッシュ・再コピーがスキップされること
- [ ] データ完全性（決定論的 `ManifestHash`）が確実に保証されること
- [ ] `ISaveBackupRepository.GetReferencedSourceBackupIdsAsync` によるバッチクエリが実行され、N+1 クエリが解消されていること
- [ ] シャノン・エントロピー急変を検知した際、Snapshot Freeze が発動して自動パージが緊急凍結されること

---

# 7. Restore Quality Gate (復元 ＆ アトミックロールバックゲート - C-3 連携)

必須確認項目：
- [ ] 復元開始前に `RescueSnapshot` への一時退避（`manifest.json` 検証含む）がアトミックに完了すること
- [ ] **二層整合性モデルに基づき、復元失敗時はユーザーファイルを直接上書きせず、`.rollback.tmp` 経由のアトミック置換で元の状態へ安全ロールバックされること**
- [ ] 復元処理中の障害注入（Fault Injection）テストにおいて、ユーザーセーブデータが破損しないこと（データ損失防止保証）

---

# 8. Migration Quality Gate (PC 移行品質ゲート - C-4, H-2 連携)

必須確認項目：
- [ ] 移行パッケージ（`.gstmgr`）のエクスポート、移送、インポート、DPAPI 再封緘の全工程が完了すること
- [ ] **PC 移行時の隔離コンテナ再封緘において、既存ファイルを直接上書きせず、`.reseal.tmp` 経由のアトミック置換で新 DPAPI 鍵が再封緘されること**
- [ ] **`migrationMasterKey`（`byte[]`）の所有権が Caller にあり、全処理完了後に `finally` で `ZeroMemory` されること**
- [ ] 旧 PC ➔ 新 PC シナリオでの完全な環境再現が実証されていること

---

# 9. Migration Compatibility Gate (移行互換性ゲート - 完全復元)

必須確認項目：
- [ ] 旧バージョンで作成されたバックアップ ZIP が現行バージョンで正常に復元できること
- [ ] 移行パッケージのフォーマットバージョン非互換（`MigrationPackageVersionMismatch`）時、強制終了せず安全中断して Standard ZIP 自力復元へフォールバックできること

---

# 10. Security Quality Gate (セキュリティ ＆ Fail-Closed 徹底ゲート)

必須確認項目：
- [ ] **WMI プロセス監視ウォッチャーが停止した際、ログ警告のみで放置されず、UI へ「⚠️ プロセス監視エンジン停止」が即時伝播されること**
- [ ] **監査チェーン検証（`VerifyAuditChainIntegrityAsync`）において、Layer 2 特権アンカー未同期時または不整合時はサイレント成功とせず、厳格に `false` を返す Fail-Closed 動作が実証されていること**
- [ ] 特権操作が DACL 保護 Named Pipe および定数時間トークン検証を経由し、UAC 30秒バックオフが機能すること
- [ ] WMI 緊急検知における PID Reuse 検証が `eventTimeUtc ± 3秒` の対称ウィンドウで動作すること
- [ ] ログ出力およびスタックトレースに生パスワード、暗号鍵、生 URL クエリ、本名パス、各種機密トークンが出力されないこと（Universal Privacy Shield `ILogSanitizer` の確実な適用）
- [ ] 既知の重大なセキュリティ脆弱性（CVE）が存在しないこと

---

# 11. Firewall Safety Gate (Firewall 安全ゲート - 完全復元)

必須確認項目：
- [ ] ルール作成・削除・所有権判定（`CreatedBy = GST` ＆ ローカル DB 突合）が正確に機能すること
- [ ] **GST が他社ソフトウェアや手動作成された未知のルール（Foreign Rule）を決して自動削除しないこと**
- [ ] 127.0.0.1 免除のインテリジェント制御が正しく動作し、公式ゲームではローカル通信が免除され、フリー/同人ゲームおよび厳格モード時には確実に完全遮断されること

---

# 12. Regression Prevention Gate (回帰防止ゲート - 完全復元)

必須確認項目：
- [ ] 新機能追加によって既存のセーブバックアップ、復元、隔離機能が一切破壊されていないこと
- [ ] 過去に修正された不具合・脆弱性が再発（Regression）していないこと
- [ ] 新たなデータ損失リスクやサイレント Fail-Open が持ち込まれていないこと

---

# 13. Performance Quality Gate (性能品質ゲート - 完全復元)

必須確認項目：
- [ ] ゲーム未起動時のバックグラウンド監視 CPU 使用率が 0.00% を維持すること
- [ ] 4GB 超の巨大ファイル処理時でも、対象workloadに定義された承認済みresource gateを満たすこと。Save Backup single-file payload workloadは承認済みのmatched-control resource gate（4GB peak - 1-byte control peak <=16MB）を適用し、その他のworkloadは既存基準を維持する。
- [ ] 大量ゲーム登録時（数百件）でも UI 描画やリストスクロールが遅延しないこと

---

# 14. Compatibility Gate (環境互換性ゲート - 完全復元)

確認対象：
- [ ] Windows 10 (22H2 以降) および Windows 11 (23H2 / 24H2) での正常動作
- [ ] .NET 10 LTS ランタイム環境での完全互換性
- [ ] FAT32 / exFAT などの非 NTFS ボリュームにおける安全なフォールバック動作

---

# 15. Documentation Gate (ドキュメント品質ゲート - 完全復元)

必須確認項目：
- [ ] 現行の公式127仕様書スコープ（`00_Baseline/`, `01_Architecture/`, `modules/`, `02_Features/` 等）の整合性が確認されていること
- [ ] ユーザーマニュアル（`User_Manual.md`）およびリリースノートが最新仕様と完全一致していること
- [ ] 仕様書の更新なしにコードの実装が先行変更されていないこと

---

# 16. User Impact Assessment (ユーザー影響評価 - 完全復元)

確認項目：
- 既存ユーザーのセーブデータ原本、設定ファイル、バックアップ ZIP、移行パッケージへの影響を事前評価し、データ損失リスクがゼロであることを証明する。

---

# 17. Release Candidate Validation (RC 検証 - 完全復元)

正式公開前に、クリーン環境での検証を実施する：
- クリーンインストール検証
- 旧バージョンからのアップグレード検証
- PC 移行パッケージのインポート・再封緘実証
- 復元中断時における自己修復ロールバック実証

---

# 18. Emergency Release Gate (緊急リリースゲート - 完全復元)

セキュリティパッチや緊急修正版の条件：
- 該当する脆弱性・重大バグが完全に解決されていること
- 修正が最小限であり、回帰リスクが完全に審査・テストされていること

---

# 19. Release Approval Model (リリース承認モデル)

正式公開には、以下の三者承認を必須とする：
1. **Technical Approval (技術承認):** ビルド・アーキテクチャ・テスト完全性
2. **Security Approval (セキュリティ承認):** 権限境界・暗号・Fail-Open 根絶
3. **Quality Approval (品質承認):** データ保護・回帰防止・ドキュメント整合

---

# 20. Release Evidence (リリース証跡管理 - 完全復元)

保存対象：
- ビルド情報、コミットハッシュ、NetArchTest ログ、全テスト実行ログ、セキュリティレビュー記録、正式配布パッケージの SHA-256 チェックサム。

---

# 21. Post Release Monitoring (公開後監視 - 完全復元)

公開後の監視項目：
- クラッシュ報告、例外発生推移、ユーザーフィードバック、Windows Update 適合性。

---

# 22. Rollback Policy (公開停止 ＆ ロールバック規約 - 完全復元)

万が一、公開後にデータ損失リスクやセキュリティ境界突破が確認された場合：
1. 配布パッケージの即時公開停止
2. 影響範囲の分析とユーザーへの緊急通知
3. 旧安定版への安全な復旧パスおよび修正版の提供

---

# 23. Quality Checklist (リリース直前確認)

☐ Gate 1: Code & Architecture Gate (NetArchTest ルール 1〜6 および `TC-ARCH-SSOT-01` 合格)
☐ Gate 2: Functional Quality Gate  
☐ Gate 3: Backup Quality Gate (Standard ZIP / Argon2id / N+1バッチ解消 / Snapshot Freeze)  
☐ **Gate 4: Restore Quality Gate (二層整合性モデル / .rollback.tmp アトミック置換)**  
☐ **Gate 5: Migration Quality Gate (.reseal.tmp / Caller-Owns ZeroMemory)**  
☐ **Gate 6: Security & Anti-Fail-Open Gate (WMI 監視障害可視化 / Layer 2 アンカー未同期時 Fail-Closed 検証)**  
☐ Gate 7: Firewall Safety Gate (Foreign Rule 保護 / 127.0.0.1 インテリジェント制御)  
☐ Gate 8: Regression Prevention Gate  
☐ Gate 9: Performance Quality Gate (workload-specific approved resource gate / CPU 0.00%; Save Backup single-file payload workload uses its approved matched-control resource gate)  
☐ Gate 10: Compatibility Gate (Windows 10/11)  
☐ Gate 11: Documentation Gate (現行の公式127仕様書スコープ完全同期)  

---

# 24. Relationship With Other Models (他モデルとの関係性 - 完全復元)

## 24.1 Test Strategy
テスト戦略および自動検証モデル：
```text
15_Test_Strategy_and_Validation_Model.md
```

## 24.2 Deployment Model
デプロイ・インストールおよび初回免責同意：
```text
17_Deployment_and_Release_Operation_Model.md
```

## 24.3 Maintenance & Support
長期運用保守およびクリーンアンインストール：
```text
18_Operational_Maintenance_and_Support_Model.md
```

## 24.4 Disaster Recovery
災害復旧および No Vendor Lock-in 原則：
```text
26_Disaster_Recovery_and_Business_Continuity_Model.md
```

---

# 25. Final Quality Statement

GST の品質とは、単にエラーが少ないことではない。

---

**どんな最悪の障害・クラッシュ・攻撃が発生しても、ユーザーのデータを確実に守り抜き、安心して更新・移行・復旧できる状態を永続的に維持することである。**

---

Final Principle:

```text
Test Completely
Enforce All 25 Quality Gates
Never Compromise on Recovery Atomicity
Maintain Absolute User Trust
```

---
