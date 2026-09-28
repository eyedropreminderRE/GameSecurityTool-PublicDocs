# GameSecurityTool Features Documentation

GameSecurityTool（GST）のエンドユーザー向け機能仕様書、設計ドキュメント、機能別仕様書群（物理32ファイル / 仕様31ファイル）の公式インデックスです。

> **Lifecycle note:** The feature documents in this directory describe approved design/specification scope. Their presence does not by itself indicate implementation, Windows verification, or release.

---

## 機能仕様書体系一覧

### 1. セキュリティ保護機能 (`Security/`)

#### 1.1 セーブデータ安全自動バックアップ (`Save_Data_Secure_Auto_Backup/` 全11ファイル)
セーブデータの非破壊・増分・64KB Chunked AEAD・タイムトラベル（ピン留め・タグ）・ローカル完結型バックアップ仕様群。
* **[02-00_Save_Backup_Index.md](Security/Save_Data_Secure_Auto_Backup/02-00_Save_Backup_Index.md)** — 【統括】バックアップ機能仕様インデックス
* **[02-01_Save_Backup_Core_and_Feature_Control.md](Security/Save_Data_Secure_Auto_Backup/02-01_Save_Backup_Core_and_Feature_Control.md)** — 設計思想・Universal Feature Control・Policy
* **[02-02_Save_Backup_Domain_and_Snapshot_Model.md](Security/Save_Data_Secure_Auto_Backup/02-02_Save_Backup_Domain_and_Snapshot_Model.md)** — Snapshot/Manifestモデル・秒精度正規化・タグ/ピン留めモデル
* **[02-03_Save_Backup_Pipeline_and_Execution.md](Security/Save_Data_Secure_Auto_Backup/02-03_Save_Backup_Pipeline_and_Execution.md)** — 2段階Debounceキュー・直列実行・抽象ストリーミング
* **[02-04_Save_Backup_Storage_and_Encryption.md](Security/Save_Data_Secure_Auto_Backup/02-04_Save_Backup_Storage_and_Encryption.md)** — Standard ZIP・64KB Chunked AEAD・Storage Safety Guard
* **[02-05_Save_Backup_Restore_Transaction.md](Security/Save_Data_Secure_Auto_Backup/02-05_Save_Backup_Restore_Transaction.md)** — 復元Transaction・RescueSnapshot自動退避・TOCTOU防御
* **[02-06_Save_Backup_Retention_and_Maintenance.md](Security/Save_Data_Secure_Auto_Backup/02-06_Save_Backup_Retention_and_Maintenance.md)** — ピン留め保護除外・孤立コンテナ回収GC・ストレージ引越し
* **[02-07_Save_Backup_Contracts_and_Architecture.md](Security/Save_Data_Secure_Auto_Backup/02-07_Save_Backup_Contracts_and_Architecture.md)** — `ISaveBackupService`・Port・不変DTO・型規約
* **[02-08_Save_Backup_Database_and_Audit.md](Security/Save_Data_Secure_Auto_Backup/02-08_Save_Backup_Database_and_Audit.md)** — SQLite スキーマ・Restrict削除制約・ピン留め/タグ永続化
* **[02-09_Save_Backup_UI_and_UX.md](Security/Save_Data_Secure_Auto_Backup/02-09_Save_Backup_UI_and_UX.md)** — WPF/MVVM・タグ編集・容量推移グラフ・ブランチ複製・引越しUI
* **[02-10_Save_Backup_Test_and_Verification.md](Security/Save_Data_Secure_Auto_Backup/02-10_Save_Backup_Test_and_Verification.md)** — 単体・結合・セキュリティ検証マトリクス

---

#### 1.2 Web Link Protection (`Web_Link_Protection_Spec/` 全7ファイル)
ゲーム内からの Web リンク起動を制御・サニタイズ・多種ブラウザで安全起動する仕様群。
* **[01-00_Overview_and_Scope.md](Security/Web_Link_Protection_Spec/01-00_Overview_and_Scope.md)** — 【統括】概要・保証境界・非目標
* **[01-01_Domain_Model_and_Policies.md](Security/Web_Link_Protection_Spec/01-01_Domain_Model_and_Policies.md)** — ホストルール・8段階決定表・正規化
* **[01-02_Application_UseCases_and_Pipeline.md](Security/Web_Link_Protection_Spec/01-02_Application_UseCases_and_Pipeline.md)** — 起動ユースケース・7段階パイプライン
* **[01-03_Contracts_and_DTOs.md](Security/Web_Link_Protection_Spec/01-03_Contracts_and_DTOs.md)** — Port (Interface)・不変DTO定義
* **[01-04_Infrastructure_and_Adapters.md](Security/Web_Link_Protection_Spec/01-04_Infrastructure_and_Adapters.md)** — URLサニタイザー・Firefox/既定ブラウザ対応・二段階PID検証
* **[01-05_Presentation_and_Dialogs.md](Security/Web_Link_Protection_Spec/01-05_Presentation_and_Dialogs.md)** — Browser Picker・確認ダイアログ・VM
* **[01-06_Security_Audit_and_Edge_Cases.md](Security/Web_Link_Protection_Spec/01-06_Security_Audit_and_Edge_Cases.md)** — 監査ログ規約・テスト・Fail-Safe

---

#### 1.3 Trust Enhancement / 説明エンジン (`Trust_Enhancement_Spec/` 全7ファイル)
検知結果・根拠・設定状態をユーザーへ説明・可視化し、安全な AI 相談を提供する仕様群。
* **[04-00_Overview_and_Principles.md](Security/Trust_Enhancement_Spec/04-00_Overview_and_Principles.md)** — 【統括】Trust Enhancement 概要・設計原則
* **[04-01_Domain_Confidence_and_Signals.md](Security/Trust_Enhancement_Spec/04-01_Domain_Confidence_and_Signals.md)** — 確信度モデル・シグナル分離 (Pure C#)
* **[04-02_Application_Explanation_Engine.md](Security/Trust_Enhancement_Spec/04-02_Application_Explanation_Engine.md)** — Explanation Engine（説明生成エンジン）
* **[04-03_Configuration_Snapshot_Model.md](Security/Trust_Enhancement_Spec/04-03_Configuration_Snapshot_Model.md)** — 設定スナップショットモデル・後方互換JSON
* **[04-04_Evidence_Bundle_and_Export.md](Security/Trust_Enhancement_Spec/04-04_Evidence_Bundle_and_Export.md)** — エビデンスバンドル出力・Audit Chain事前整合性検証
* **[04-05_Timeline_View_and_Contracts.md](Security/Trust_Enhancement_Spec/04-05_Timeline_View_and_Contracts.md)** — タイムライン表示・Contracts DTO
* **[04-06_Gemini_Ai_Integration_and_Consent_Spec.md](Security/Trust_Enhancement_Spec/04-06_Gemini_Ai_Integration_and_Consent_Spec.md)** — 【新設】BYOK Gemini API連携・事前説明・プライバシー同意確認

---

#### 1.4 高度セキュリティ ＆ ゲーマー保護仕様
* **[Advanced_User_Protection_Master_Spec.md](Security/Advanced_User_Protection_Master_Spec.md)** — 高度ユーザー保護機能 統合マスター仕様書 (v3.1)
* **[LNK_Hijack_Detection_Spec.md](Security/LNK_Hijack_Detection_Spec.md)** — `.LNK` 改ざん検知仕様書 (非ブロッキング COM 解析)
* **[Mod_Security_and_Provenance_Spec.md](Security/Mod_Security_and_Provenance_Spec.md)** — 【新設】MOD 安全性診断 ＆ 出所追跡仕様書 (3層ハイブリッド)
* **[InGame_Overlay_HUD_Spec.md](Security/InGame_Overlay_HUD_Spec.md)** — 【新設】インゲーム・オーバーレイ HUD 仕様書 (デュアルトリガー・3大パッド配列適応)

---

### 2. プライバシー保護機能 (`Privacy/` 全2ファイル)
* **[03_OnDemand_Clipboard_URL_Sanitizer_Spec.md](Privacy/03_OnDemand_Clipboard_URL_Sanitizer_Spec.md)** — オンデマンド クリップボードURLサニタイザー仕様書 (STAスレッド安全性保証)
* **[04_PrivacySafe_Crash_Diagnostics_Report_Spec.md](Privacy/04_PrivacySafe_Crash_Diagnostics_Report_Spec.md)** — 【新設】個人情報完全保護型 クラッシュ相談レポート生成仕様書 (Discord/Reddit用伏字化)

---

## 関連ベースラインおよびコアモジュール

機能仕様は、以下のアーキテクチャおよびベースライン設計に準拠します。
* **[00_Baseline/](../00_Baseline/)** — 最高権限ベースライン・不可侵原則・セキュリティ境界 (全58ファイル)
* **[01_Architecture/](../01_Architecture/)** — Clean Architecture 5層構造・技術基盤標準 (物理10ファイル / 仕様9ファイル)
* **[modules/](../modules/)** — 6大コア実行モジュール仕様 (全7ファイル)

---

End of Document
```

---