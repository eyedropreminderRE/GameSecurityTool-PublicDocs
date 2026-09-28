# 01-07: Cross-Cutting Concerns and Lifecycle

**Document ID:** GST-ARCH-BASELINE-002-PART7  
**Parent Document:** Architecture & Technology Baseline v2.0 (GST-ARCH-BASELINE-002)  
**Category:** Governance, Quality & Lifecycle  
**Status:** Approved Baseline Candidate  

---

# 52. Logging Baseline & Privacy Protection

GST では `Microsoft.Extensions.Logging` を採用し、構造化ログ（Structured Logging）を出力する。

## 52.1 目的と出力先
* **目的:** デバッグ、操作追跡、エラー分析、セキュリティ監査の補助
* **出力先:** ローカルファイル（`%LocalAppData%\GameSecurityTool\Logs`）および SQLite ログテーブルのみ（外部送信ゼロ）

## 52.2 Privacy Logging Rule (個人情報保護の絶対規約)
以下の機密情報をログファイルへ出力・永続化することを厳禁とする：
* パスワード、パスフレーズ、PIN
* 暗号鍵、ソルト、セッショントークン、API Key
* クリップボードの全文
* URL の Query String 全文
* ユーザープロファイル名を含む個人を特定可能な絶対パス（`C:\Users\<UserName>\...`）

## 52.3 Sanitization パイプライン
すべてのログ出力は、共通サニタイザー（`PrivacyLogSanitizer`）を通過させる。
```text
Raw Log Message ──> [ PrivacyLogSanitizer ] ──> [ Masked Output ] ──> File / DB
```
* **パスのマスキング例:** `C:\Users\JohnDoe\Saved Games\GameA` -> `C:\Users\***\Saved Games\GameA`
* **URL のマスキング例:** `https://example.com/login?token=secret123` -> `https://example.com/login?[QUERY_STRIPPED]`

---

# 54. Testing Baseline & Quality Assurance

品質保証は Architecture・Security・Function の 3層で実施する。

## 54.1 Unit Test (単体テスト)
* **対象:** Rule Evaluation, Risk/Confidence Calculation, URL Normalization, Host Matching, Backup Manifest 差分計算, Policy Resolution。
* **要件:** OS ネイティブ API や Windows ハンドルに依存せず、CI 環境（Linux/Windows）で高速に実行可能であること。

## 54.2 Integration Test (統合テスト)
* **対象:** SQLite (WAL + EF Core), DPAPI 暗号化/復号, Windows Firewall COM Adapter, WMI Process Tracker, File System Adapter。
* **要件:** 実環境に近い Windows テスト環境で、DB 書き込み直列化キュー（`SqliteDatabaseWriter`）のロック耐性を検証する。

## 54.3 Security Test (必須セキュリティテスト項目)
以下の脅威シナリオに対する自動/手動テストを必須とする：
* **TOCTOU / Hardlink 置換攻撃:** 検証と実処理の間に別ファイルへ差し替えられた場合の検知。
* **Reparse Point ループ / ジャンクション脱出:** シンボリックリンクによる保護領域外アクセスの防止。
* **PID Reuse 攻撃:** プロセス終了後の同一 PID 再利用時の誤認検知防止。
* **Path Traversal:** 相対パス（`..\..\`）を含む不正ファイルパスの拒否。
* **URL Host Spoofing / Subdomain 偽装:** 部分一致による誤許可（`evil-domain.com`）の防止。
* **UAC Denied:** 特権昇格ダイアログが拒否された場合の安全なフォールバック。
* **Disk Full / OOM 耐性:** ストレージ容量枯渇や大容量ファイル処理時の安全停止。
* **Cancellation:** 処理中の `CancellationToken` 発火によるリソース完全解放。

---

# 55. Build / CI Baseline

## Local Build 手順
```powershell
dotnet restore
dotnet build --configuration Release
dotnet test --configuration Release
```

## 55.1 CI Environment
* GitHub Actions / Windows Runner / .NET 10 SDK

## 55.2 Build Quality Gate
以下の条件を 1 つでも満たさない場合、マージおよびビルドを失敗（Build Failure）とする：
1. Compile Error = 0
2. Nullable Warning = 0 (`<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`)
3. Code Analyzer 重大問題 = 0
4. XAML 構文および Binding 整合性の検証成功
5. プロジェクト間依存関係ルール（Clean Architecture 違反なし）の検証成功

---

# 56. AI Implementation Rules Baseline

AI によるコード生成・実装時は、以下の規約を厳格に適用する。

## 56.1 AI 禁止事項
* **Source of Truth の無視:** 実在しない仮想クラスや廃止されたインターフェースを捏造して実装すること。
* **未完成コードの提示:** `// TODO:`, `/* 後で実装 */`, 擬似コードの残置。
* **大規模破壊的変更:** 関連プロジェクトの合意なしにインターフェースシグネチャを一括破壊すること。

## 56.2 Implementation Flow
```text
実ファイル・既存設計の調査
        ↓
影響範囲と依存関係の確認
        ↓
小単位でのインターフェース・DTO・クラス実装
        ↓
Build & Unit Test による検証
        ↓
完了報告
```

---

# 58. Technical Debt & Lifecycle Management

## 58.1 スナップショット（RescueSnapshots）のライフサイクル管理
セーブデータ復元時の一時退避スナップショット（`RescueSnapshots`）は、以下のルールで自動クリーンアップする：
* **24時間経過:** 正常完了した復元操作の一時スナップショットは、次回アプリケーション起動時に自動パージ。
* **7日間経過:** 異常終了や警告（`PartialFailed`）が残った一時スナップショットは、警告表示の上で 7日経過後に物理削除。
* **実行優先度:** クリーンアップ処理はバックグラウンドの最低優先度（`TaskPriority.Lowest`）で実行し、ゲーム起動を阻害しない。

## 58.2 起動時整合性検証 & セーフモードリカバリ (指摘6連携)
アプリケーション起動時、`SystemChangeRecord` を走査して前回の未完了トランザクションを検証する。
* **異常検知 (`ReversionStatus == PartialFailed / InProgress`):**
  前回のロールバックがクラッシュ等で中断していた場合、自動的にセーフモードリカバリダイアログを起動し、ロールバックの再実行または安全なフォールバックをユーザーに提供する。

## 58.3 Technical Debt（将来拡張候補）の管理
以下の項目は仕様書へ混入させず、Technical Debt バックログとして段階的に評価・導入する：
* USN Journal によるファイル変更検知の高速化
* SQLCipher による SQLite データベース全体の暗号化
* 複数検出器間での Shared Scan Context（共有スキャンコンテキスト）

---

# 59. Baseline Update Rule

本書群（`01-00` 〜 `01-07`）を変更する場合は、以下の変更記録を必須とする：
* `Version`
* `ChangedAt`
* `ChangeSummary`
* `Reason`
* `AffectedProjects`
* `MigrationRequired`

特に、DB プロバイダ変更、IPC プロトコル変更、暗号方式変更などの重大なアーキテクチャ変更は、チーム全体のセキュリティレビューを経てから改訂する。
```

---