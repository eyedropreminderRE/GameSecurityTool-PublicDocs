# GameSecurityTool

# External Integration and Platform Compatibility Model

## External Service Integration / Platform Compatibility / Dependency Control Specification

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-030 |
| Version | 2.1 (Full Relationship Links, SharpCompress, Steam Loopback & BYOK AI Edition) |
| Status | Formal Baseline Specification (Highest Integration Authority) |
| Category | External Integration |
| Authority Level | Compatibility Governance |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- 外部ランチャー連携（Steam, Epic, GOG 等）
- マルチアーカイブ展開連携（`SharpCompress`）
- 外部生成 AI 連携（BYOK Google Gemini API）
- プラットフォーム・ネットワーク互換性（Steam 127.0.0.1 免除）
- Offline First 原則および外部依存リスクの隔離

を定義する。

---

目的：

```text
Support External Gaming Environments
*
Reduce Dependency & Cloud Risks
*
Preserve Offline Core Functionality
```

---

# 1. Integration Philosophy

## 1.1 Core Principle
GST は、外部クラウドサービスに依存して基本動作するアプリケーションではない。

基本方針：
```text
External Integration Enhances GST (外部連携は利便性を向上させる)
External Service Does Not Control GST (外部サービスに GST を支配・停止させない)
Offline First Always (ネットワーク切断時でも全コア機能が確実に稼働する)
```

---

# 2. Integration Priority Model (連携の優先度階層)

## 2.1 Core Functions (完全ローカル完結・外部依存ゼロ)
外部通信なしで GST 単体で動作する不可侵機能：
- 並列ファイルスキャン ＆ PE ヘッダー検証
- Standard ZIP + Argon2id によるセーブバックアップ ＆ アトミック復元
- 64KB Chunked AEAD による暗号化隔離 ＆ アトミック再封緘
- Windows Firewall ルール制御 ＆ 監査ハッシュチェーン

## 2.2 Optional Local Integrations (ローカル外部ツール・ランチャー連携)
- Steam / Epic / GOG のローカルインストールパス検出
- `SharpCompress` によるポータブルゲーム・本体アーカイブ（`.7z`/`.rar`/`.zip`）の安全展開
- 外部ファイル圧縮ソフトウェアおよび外部セキュリティスキャナとの安全連携

## 2.3 User-Opt-In Cloud Integrations (ユーザー完全同意型 クラウド連携)
- BYOK 方式による Google Gemini API 連携（中間サーバーゼロ直接 HTTPS 通信）

---

# 3. External Dependency Rules

禁止事項：
```text
Required External Login       : クラウドログインの強制
Required Internet Connection  : インターネット常時接続の強制
Required Third-Party Account  : 第三者アカウント作成の強制
```

原則：  
完全オフライン環境であっても、UI 起動、スキャン、バックアップ、復元、Firewall 制御のすべてが正常に動作しなければならない。

---

# 4. Game Launcher & Platform Integration

## 4.1 Supported Launcher Concept
Steam、Epic Games Launcher、GOG Galaxy、Xbox App 等のゲームライブラリをサポートする。

## 4.2 127.0.0.1 免除のインテリジェント制御
Steam クライアント等の公式ゲームとクライアント間の IPC および DRM 認証を阻害しないため、公式ゲームでは `127.0.0.0/8`（Localhost）をブロック対象から自動免除する。一方、フリーゲームや同人ゲーム、および「厳格モード」有効時は、ローカルポートを利用したトンネリングやプロキシ迂回を防ぐため、`127.0.0.1` も含めて完全に遮断する。

## 4.3 Launcher Failure Handling
ランチャーが停止・オフライン・アンインストールされた場合でも、GST はローカル DB の登録情報に基づいて保護およびバックアップを継続する。

---

# 5. Multi-Archive & External Tool Integration

## 5.1 Multi-Archive Extraction (`SharpCompress`)
フリーゲームの安全展開（`SafeGameOnboarding`）および本体アーカイブ復元（`GameArchiveRestored`）における外部ライブラリ統制：
1. **採用ライブラリ:** **`SharpCompress`** (公式承認 / MIT ライセンス)。
2. **対応フォーマット:** `.7z` (LZMA/LZMA2), `.rar`, `.zip`。
3. **Zip Slip 防御バリア:** アーカイブ展開時、相対パス（`../`）による管理フォルダ外脱出を構造的に検知し即座に展開を拒否する。
4. **ストリーミング容量監視:** 展開後サイズがディスク容量を超える Zip Bomb 攻撃を展開途中で安全中断する。

## 5.2 外部アーカイバ・外部スキャナ連携規約
1. **外部ファイル圧縮ソフトウェア連携 (`IExternalArchiverAdapter`):**  
   ゲーム本体フォルダのバックアップ時、外部ファイル圧縮ソフトウェアを呼び出して高速アーカイブ化を委譲する。圧縮パラメータは GST から強制せず、ユーザー設定のプリセットを尊重する。誤操作防止のため「圧縮開始前の詳細設定ダイアログ表示」は **既定 ON** とする。
2. **外部セキュリティスキャナ連携 (`IExternalSecurityScannerAdapter`):**  
   Windows 標準セキュリティ機能（`MpCmdRun.exe`）や外部スキャンツールを呼び出し、ゲームや MOD フォルダの検査を実施可能とする。定期自動スキャンは「既定 OFF（オプトイン）」とし、ゲーム実行中は負荷防止のため「絶対延期」とする。
3. **外部ツール安全呼出および引数インジェクション排除:**  
   外部ツールおよび外部ブラウザ起動時は、文字列連結を厳禁とし、必ず `ProcessStartInfo.ArgumentList` を用いてパラメータを独立渡しする。パスはホワイトリスト検証と正規化を行い、信頼されたディレクトリ内の正規バイナリのみを起動する。

---

# 6. BYOK Generative AI Integration & Guardrails (Gemini API 連携規約)

ユーザーが任意で有効化する外部 AI 相談機能のセキュリティ境界とガードレール：

```text
[ ユーザーの AI 相談要求 ]
            │
            ▼
1. [クライアントサイド事前マスキング] (Universal Privacy Shield ILogSanitizer による伏字化)
            │
            ▼
2. [送信前プレビュー (Prompt Preview)] (ユーザーが送信ペイロードを目視確認・承認)
            │
            ▼
3. [ダイレクト HTTPS 送信] (中間サーバー完全ゼロ)
   - 送信先: https://generativelanguage.googleapis.com/v1beta/models/...
   - 認証方式: x-goog-api-key HTTP ヘッダー (URL クエリ露出の完全禁止)
            │
            ▼
4. [回答の受信 ＆ 画面表示] (API キーは DPAPI 暗号化保管)
```

### AI ガードレール仕様
- **分析限界と免責:** AI の回答は GST がローカルで収集した客観的ログのみに基づく推測であり、安全性を完全に保証するものではない旨を UI 上に明記する。
- **送信前プロンプト確認 (Prompt Preview):** クラウドへ送信される生プロンプト（マスキング適用後）をユーザーが事前に確認できる確認モーダルを提供する。
- **推奨 FW ルール適用の安全化:** AI が推奨した Firewall ルールを適用する際は、ワンクリックで適用できる導線を提供するが、必ず「自己責任免責ダイアログ」と「ポート・プロトコルの微調整 UI」を経由させる。
- **オフライン・障害フォールバック:** API 制限到達時やオフライン時は、クラッシュさせずにローカルの静的診断結果（`RiskExplanationDto`）へ安全にフォールバックする。

---

# 7. File System & Cloud Drive Compatibility

- **NTFS / 非NTFS 互換性:** NTFS 以外のファイルシステム（FAT32, exFAT, ネットワークドライブ等）では、ACL 適用を安全にスキップし、`FileShare.None` によるストリーミング処理で安全性を担保する。
- **クラウドストレージ共存:** OneDrive, Google Drive 等の同期フォルダ内にセーブデータが存在する場合でも、排他ロックリトライにより安全に読み取りバックアップを実行する。

---

# 8. Integration Module Architecture (Hexagonal Isolation)

外部連携による OS や外部 API の仕様変更の影響を局所化するため、必ず Contracts Port を経由して接続する。

```text
GST.Application (UseCase)
        │
        ▼ (Port Interface)
GST.Contracts (IArchiveExtractor, IAiExplanationProvider, IFirewallManager)
        │
        ▼ (implements)
GST.Infrastructure (SharpCompressAdapter, GeminiApiClientAdapter, WindowsFirewallManager)
```

---

# 9. Compatibility & Security Checklist

外部連携機能の実装・リリース前に確認：

☐ インターネット未接続（完全オフライン）環境で全コア機能が動作すること  
☐ **127.0.0.1 免除のインテリジェント制御が正しく動作し、公式ゲーム免除・厳格モード遮断が徹底されていること**  
☐ **`SharpCompress` によるアーカイブ展開で Zip Slip / Zip Bomb 防御が機能すること**  
☐ **外部アーカイバ呼出時の詳細設定ダイアログが既定 ON であり、引数インジェクションが排除されていること**  
☐ **外部スキャナ連携が既定 OFF であり、ゲーム実行中は絶対延期されること**  
☐ **Gemini API キーが URL クエリではなく `x-goog-api-key` ヘッダーで送信されていること**  
☐ 外部 API への送信直前に個人情報・各種認証トークンが `ILogSanitizer` で確実に伏字化され、Prompt Preview が表示されること  
☐ 外部ランチャーのアカウント認証情報やパスワードを GST が保存していないこと  

---

# 10. Relationship With Other Models (他モデルとの関係性 - 完全復元)

## 10.1 Source Structure & Modules
アーキテクチャおよび Clean 5-Layer 結合規則：
```text
28_Source_Code_Structure_and_Module_Architecture_Model.md
```

## 10.2 Security Implementation
セキュアコーディングおよび外部 API 規約：
```text
14_Security_Implementation_Guideline.md
```

## 10.3 Threat Analysis
外部連携における脅威モデルおよび攻撃面：
```text
21_Security_Threat_Model_and_Attack_Surface_Analysis.md
```

## 10.4 Configuration Management
外部連携設定および出荷時 Standard プロファイル：
```text
23_Configuration_Management_and_Environment_Model.md
```

## 10.5 Game Environment Protection & AI Integration
ゲーム保護および BYOK AI 連携仕様：
```text
GameProtection/03_Game_Environment_Protection_Model.md
02_Features/Security/Trust_Enhancement_Spec/04-06_Gemini_Ai_Integration_and_Consent_Spec.md
```

---

# 11. Final Integration Statement

GST の外部連携設計とは、外部サービスに依存することではない。

---

**外部環境を安全・柔軟に受け入れながら、どんな状況でもローカル完結の保護機能を確実に維持することである。**

---

Final Principle:

```text
Offline First Always
Isolate External Dependencies by Ports
Intelligent Loopback Control
Sanitize Before External AI Calls
```

---
