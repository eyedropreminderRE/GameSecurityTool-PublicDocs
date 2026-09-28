# GameSecurityTool

# Test Strategy and Validation Model

## Quality Assurance / Test Architecture / Validation Process Definition

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-015 |
| Version | 3.4 (Contracts ThreatReasonCodes SSOT Architecture-Test Edition) |
| Status | Formal Baseline Specification |
| Category | Testing and Validation |
| Authority Level | Quality Assurance Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- テスト戦略・アーキテクチャ
- 品質保証基準
- レイヤー別テスト責務
- セキュリティ・障害耐性検証マトリクス
- CI パイプラインおよびリリースゲート

を定義する。

---

GST におけるテストの目的：

```text
Confirm Correct Function
Verify Security Boundaries
Guarantee Safe Recovery
Prevent Architectural Decay
```

---

# 1. Test Layer Model (Test Pyramid)

```text
┌────────────────────────────────────────────────────────┐
│               System & End-to-End Tests                │ (GUI, Complete Workflows)
├────────────────────────────────────────────────────────┤
│                 Security Attack Tests                  │ (TOCTOU, PID Reuse, OOM, Nonce, Reseal)
├────────────────────────────────────────────────────────┤
│                   Integration Tests                    │ (SQLite Queue, Named Pipe, Storage GC)
├────────────────────────────────────────────────────────┤
│                   Domain Unit Tests                    │ (Risk Logic, Manifest Diff, Retention)
├────────────────────────────────────────────────────────┤
│               Architecture Tests (CI Gate)             │ (NetArchTest: 5-Layer Dependency Rules + Composition Root Exception)
└────────────────────────────────────────────────────────┘
```

---

# 2. テストプロジェクト構成 (`Tests/`)

```text
Tests/
├── GameSecurityTool.ArchitectureTests  <-- NetArchTest によるレイヤー依存方向・型直結自動検証
├── GameSecurityTool.UnitTests          <-- Domain / Application ロジック単体テスト
├── GameSecurityTool.IntegrationTests   <-- SQLite, Standard ZIP, Named Pipe IPC 結合テスト
└── GameSecurityTool.SecurityTests      <-- 脆弱性・攻撃耐性・障害復旧・OOM 耐性テスト
```

---

# 3. Architecture Test Strategy (`GST.ArchitectureTests` - L-1 是正)

「プロースで正しいことを謳いながらコードで破られる」事故を機械的に遮断するため、CI ビルド時に `NetArchTest.Rules` を実行する。

### 必須自動検証ルール:
1. **Domain Isolation:** `GST.Domain` は他の全プロジェクト（`Application`, `Infrastructure`, `Contracts`, `UI`）および外部 I/O ライブラリに一切依存しないこと。
2. **Application OS Independence:** `GST.Application` は Windows OS 固有 API（`System.Management`, `Microsoft.Win32`, `System.Runtime.InteropServices`）に直接依存しないこと。
3. **Infrastructure Direct Access Prohibited:** `GST.Presentation` の UI / View / ViewModel は `GST.Infrastructure` を参照しないこと。`App.xaml.cs` の Composition Root における DI 登録目的の project reference のみ許可する。
4. **Single Writer Queue Enforced:** `SqliteDatabaseWriter` 以外のクラスが `DbContext.SaveChangesAsync()` を直接呼び出していないこと。
5. **Infrastructure Domain Direct Binding Prohibited (L-1 是正):** `GST.Infrastructure` 内のクラスが、Application 層のマッピング Port（`IRiskAssessmentService` 等）を経由せず、`GameSecurityTool.Domain.Models.*` サブ名前空間の型を直接インスタンス化・参照してマッピングを肩代わりしていないこと。
6. **SSOT Alignment (`TC-ARCH-SSOT-01`):** `GameSecurityTool.Contracts.Common.ThreatReasonCodes` を唯一の Threat Reason Code レジストリとして扱い、公開された文字列定数が空・重複していないこと、`ThreatReasonDto.ThreatReasonCode` が `string` 境界を使用すること、Domain / Contracts 内に競合する `ThreatReasonCode` enum を定義していないことを CI で機械検証すること。

---

# 4. Unit Test Strategy (`GST.UnitTests`)

OS や外部環境に依存せず、CI 環境（Linux / Windows）でミリ秒単位で高速実行される不変ロジックの検証。

* **Domain Risk Evaluation:** `SecurityEngineEvaluator` によるスコア加算、未署名・ハイジャック警戒名判定、アセット Mod コンテンツ種別不一致判定、スクリプト LotL 走査。
* **Rule Scope & Feature Control:** Global Enforced / Overrideable / GameProfile ルール優先順位（8段階決定表）の評価。
* **URL Normalization & Sanitization:** IDN / Punycode 変換、UserInfo 除去、Strict Query Stripping（クエリ完全除去）、公式サポートドメイン厳格照合、生 IP アドレスリテラルの登録・評価対称性。
* **Manifest & Change Detection:** `ManifestHashCalculator` の決定論的ハッシュ算出、Fast Metadata Check 差分判定、パッチ適用時リスク急変 (Update Anomaly) 検知。
* **Retention & Rescue Snapshot Policy:** `RetentionEvaluator` による差分参照保護、`RescueSnapshotRetentionEvaluator` による 24時間 / 7日間期限切れ判定、シャノン・エントロピー急変による Snapshot Freeze（パージ凍結）。
* **Allow List TOCTOU Guard:** 有効期限切れ（`ExpiresAt`）ルールのリアルタイム無効化、`TamperedAllowList` なりすまし拒否。

---

# 5. Integration Test Strategy (`GST.IntegrationTests`)

実コンポーネントを用いた結合動作および並行性の検証。

* **SQLite Single Writer Queue:** マルチスレッドから大量の書き込み要求が集中した際、`SqliteDatabaseWriter` により `SQLITE_BUSY` なしで直列コミットされること。
* **SaveBackupRepository バッチクエリ:** `GetReferencedSourceBackupIdsAsync` により N+1 回の SQL 発行を排除し、単一バッチクエリで依存関係が解決されること。
* **Standard ZIP Container:** `ZipSaveBackupStorage` による ZIP 作成、マニフェスト書き出し、ストリーミング展開、ハッシュ検証の結合動作。
* **Named Pipe IPC & UAC Backoff:** Main プロセスと `ElevatedWorker` 間でのワンタイムトークン認証、DACL 保護、UAC 拒否時バックオフ、バッチルール適用の疎通確認。
* **Tamper-Evident Audit Chain:** SQLite 内の `PreviousHash` / `CurrentHash` 連鎖と Layer 1 (DPAPI) / Layer 2 (特権アンカー) の詳細検証（`VerifyAuditChainDetailedAsync`）結合動作。
* **Layer 2 未同期 Fail-Closed:** `TamperEvidentAuditLogger.VerifyAuditChainIntegrityAsync` において、Layer 2 アンカーが未同期または不整合の場合に確実に `false` を返却すること。
* **Container GC & Grace Period:** 作成後 1 時間以内の新規コンテナが誤爆削除されず、参照数 0 の孤立コンテナのみが回収されること。

---

# 6. Security & Failure Test Strategy (`GST.SecurityTests`)

セキュリティ製品としての堅牢性・フェイルセーフを保証する必須テストマトリクス。

| テスト分類 | 検証項目 | 期待される安全動作 |
| :--- | :--- | :--- |
| **OOM 耐性** | 5GB 超のダミーファイルを隔離 / バックアップ | プロセス使用メモリが 120MB 以下を維持（ストリーミング検証）。 |
| **PID Reuse 防御** | 同一 PID で起動時刻が 3秒超乖離したプロセス | `ValidateAndOpenProcess` が拒否し、`TerminateProcess` を誤爆させない。 |
| **TOCTOU / ジャンクション** | 復元先・スキャンパスを外部領域へシンボリックリンク置換 | `PathValidationBarrier` が検知し、即座に例外中断。 |
| **アトミックロールバック** | ロールバック処理（`RollbackAsync`）中に強制電源断/例外注入 | `.rollback.tmp` 経由のアトミック移動により、ユーザーの元ファイルが中途破損しないこと（二層整合性モデル）。 |
| **アトミック再封緘** | PC 移行再封緘（`ImportAndReseal`）中に強制例外注入 | `.reseal.tmp` 経由のアトミック移動により、元の隔離コンテナが破損せず復号不能にならないこと。 |
| **Nonce 重複排除** | 隔離コンテナおよびバックアップ内の全エントリ | チャンク別 Nonce (12B) がすべてユニークに乱数生成されていること。 |
| **WER 元値復元** | ユーザーが事前に設定していたレジストリ値 | ゲーム終了後、削除ではなく変更前の元の値へ正確に書き戻されること。 |
| **Fail-Open 根絶** | WMI 監視ウォッチャーの停止 / 障害 | サイレントに放置されず、`WatcherFaulted` が UI に劣化警告を即時伝播し再初期化を試行すること。 |
| **WIPER-01** | Canary所有権境界 | 既存同名ファイルをCanaryとして採用せず、GSTが新規作成したファイルだけを所有対象として追跡・撤収すること。 |
| **WIPER-02** | PID世代境界 | WIPERのプロセス制御が登録時のPIDだけでなく開始時刻一致を確認し、再利用PIDを対象にしないこと。 |
| **WIPER-03** | Command Token負誤検知 | 既知のコマンド文脈だけを検出し、ファイル名等に破壊語が偶然含まれるだけでは終了処理を実行しないこと。 |
| **WIPER-04** | Watcher障害可視化 | FileSystemWatcher / WMI Watcherの障害時に `IsOperational=false` となり、正常稼働扱いを継続しないこと。 |
| **WIPER-05** | Applicationセッション原子性 | Child Guard / Circuit Breakerの片側登録失敗時に成功側の登録を解除し、中途状態を残さないこと。 |
| **WIPER-06** | Feature無効化時のセッション終了 | Anti-Wiperを無効化した際、Application Coordinatorが先に保護セッションを終了し、各Infrastructure登録を解除すること。 |
| **容量枯渇中断** | ドライブ空き容量が 2GB 未満の状態で書き込み | `Storage Safety Guard` が即座に中断し、既存スナップショットを保護。 |
| **二層整合性・再実行** | 複数ファイル復元トランザクション中に強制終了 | 次回起動時の整合性検査により中途状態が検知され、冪等な再実行またはクリーンロールバックが行われること。 |

---

# 7. CI / CD Pipeline & Release Gates

```text
[ Git Commit / Pull Request ]
              │
              ▼
1. Build (`dotnet build` - TreatWarningsAsErrors = true, Nullable = 0)
              │
              ▼
2. Architecture Tests (`NetArchTest` 5層依存, L-1 型直結検査, TC-ARCH-SSOT-01)
              │
              ▼
3. Unit & Integration Tests (全件パス必須)
              │
              ▼
4. Security & Failure Tests (OOM / TOCTOU / Rollback / Reseal / Fail-Closed 検証)
              │
              ▼
[ Release Gate Approval ] ──> Release Candidate Package
```

---

# 8. Final Testing Statement

GST のテストは、単なる「正常動作の確認」ではない。

```text
Test The Failure
Test The Boundary
Test The Recovery
Protect The User Assets
```

---

