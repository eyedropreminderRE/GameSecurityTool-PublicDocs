# GameSecurityTool

# Security Boundary and Protection Model

## Security Architecture & Boundary Definition

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-002 |
| Version | 3.0 (Full Requirements Restored, Safe Mutation Pipeline & Complete Boundaries Master Edition) |
| Status | Formal Baseline Specification (Highest Security Boundary Authority) |
| Category | Security Boundary Definition |
| Authority Level | Core Security Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- 保護対象と資産分類
- 3大操作境界（Observation / Decision / Mutation）
- 読取と変更の厳格な分離
- Protected Path（システム保護領域）保護
- 特権昇格および IPC 境界
- ゲーム環境・データ所有権境界
- Safe Mutation Pipeline（変更操作の安全パイプライン）
- Security Failure Handling（障害時の安全停止）
- Web / クリップボード / オーバーレイ / 厳格起動境界

を定義する。

---

GST の基本方針：

```text
Observe More
Modify Less
(観察・理解を深め、勝手な改変を最小化する)
```

である。

GST は環境を理解する能力を持つが、環境を自由に改変・支配する権限を持つことを目的としない。

---

# 1. Security Boundary Principle

## 1.1 3大境界モデル (Three Boundary Model)

GST は以下の境界をアーキテクチャレベルで物理分離する。

```text
┌────────────────────────────────────────────────────────┐
│               Observation Boundary (観測境界)          │
│ - ファイルメタデータ・ハッシュ・署名の受動的読み取り    │
│ - WMI 受動プロセストレース (CPU 0% 待機)               │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│                Decision Boundary (判定境界)            │
│ - SecurityEngine による客観的シグナル評価 (Pure C#)    │
│ - Universal Rule Scope 照合 ＆ 確信度 (Confidence) 算出│
│ - ユーザーへの理由・証拠 (Evidence) の可視化           │
└──────────────────────────┬─────────────────────────────┘
                           │ (※ 自動実行の禁止)
                           ▼
┌────────────────────────────────────────────────────────┐
│                Mutation Boundary (変更境界)            │
│ - ユーザーの明示的同意 (Explicit Consent) が必須        │
│ - 事前退避 (RescueSnapshot / ChangeJournal) の義務化   │
│ - アトミックロールバック ＆ 安全復元トランザクション   │
└────────────────────────────────────────────────────────┘
```

---

# 2. Read / Mutation Separation (読取と変更の分離)

## 2.1 Core Rule
GST では、読み取り操作と変更操作を完全に分離して扱う。

```text
Read Operation  !=  Mutation Operation
```

* **情報取得フロー:**
  `Read ➔ Analyze ➔ Display (画面表示のみ・副作用ゼロ)`
* **変更フロー:**
  `Request ➔ Validation ➔ User Approval ➔ Audit ➔ Atomic Execute`

## 2.2 直接変更の完全禁止
UI や分析ロジックから、以下の物理操作を直接呼び出すことをビルドレベルで禁止する。

```csharp
// 【禁止】UI または Domain からの直接呼出
File.Delete();
Directory.Delete();
Directory.Move();
Registry.SetValue();
```

必ず `Presentation ➔ Application UseCase ➔ Contracts Port ➔ Infrastructure Adapter` のパイプラインを経由して実行する。

---

# 3. Protected Path Mutation Guard

## 3.1 対象保護領域
```text
C:\Windows
C:\Windows\System32
Program Files / Program Files (x86)
WindowsApps
Game Installation Directory
Game Save Directory
User Profile Configuration Directory
```

## 3.2 保護ルール
Protected Path に対するすべての変更操作は、必ず `Path Classification ➔ Permission Check ➔ User Confirmation ➔ Audit ➔ Atomic Execute` を通過する。

## 3.3 自動変更の絶対禁止
検知結果（マルウェア疑い等）のみを理由として、GST が保護領域内のファイルを自動削除・自動改変することを厳禁とする。必ず証拠を提示し、ユーザーの明示的同意を得てから隔離・変更を行う。

---

# 4. Administrator Privilege Model (特権分離境界)

## 4.1 最小権限原則 (Least Privilege)
Main GUI プロセスは常に **Standard User（標準ユーザー権限）** で起動・動作する。マニフェストでの `requireAdministrator` の常時要求を禁止する。

## 4.2 特権操作の分離 (Out-of-Process UAC Worker)
管理者権限が必要な操作（Windows Firewall ルール制御等）は、DACL 保護された Named Pipe 経由で独立した昇格ワーカー（`ElevatedWorker.exe`）へ委譲し、60秒間のアイドルタイムアウトで自動自己終了させる。

---

# 5. Game Environment Protection Boundary

## 5.1 データ所有権
ゲーム本体、セーブデータ、設定ファイル、ユーザー作成 MOD はすべてユーザーの所有物であり、GST は所有権を取得しない。

## 5.2 非破壊性の原則
ユーザーが明示的に復元（Restore）を承認した場合を除き、セーブデータやゲームバイナリを勝手に上書き・編集・消去してはならない。

---

# 6. Firewall Protection Boundary

1. **所有権の明示:** GST が作成したルールには `CreatedBy = GST` および `RuleTag` を付与し、ローカル DB に登録されたレコードと突合して管理する。
2. **Foreign Rule の保護:** GST が作成していない外部ルールや他社ルールの自動削除を禁止する（手動承認必須）。
3. **127.0.0.1 免除のインテリジェント制御:** Steam 等の公式ランチャー管理下にあるゲームではローカル IPC や実績同期を維持するため `127.0.0.1` を免除するが、フリーゲーム・同人ゲーム実行時および「厳格モード」有効時は、トンネリング攻撃やローカルプロキシ迂回を防ぐため `127.0.0.1` も含めて完全に遮断する。

---

# 7. Storage Protection Boundary

GST 内部データとゲームデータを物理的に分離する。

```text
GST Storage (%LocalAppData%\GameSecurityTool\)
├ gamesecurity.db (SQLite メイン DB)
├ IpcTokens\ (特権 IPC 用トークン)
├ RescueSnapshots\ (復元前一時退避データ)
└ Quarantine\ (64KB Chunked AEAD 暗号化隔離コンテナ)

Game Storage (ユーザー管理領域)
├ Game Files
└ Save Data
```

---

# 8. Configuration Protection Boundary

設定変更は必ずバリデーションおよび監査ログ（Audit Log）記録を伴い、サイレントなバックグラウンド設定変更を禁止する。

---

# 9. Backup and Recovery Boundary (No Vendor Lock-In)

## 9.1 Backup の位置付け
バックアップは GST 内部の閉域データではなく、**ユーザー資産の安全な複製（Safety Copy）** である。

## 9.2 ポータブル暗号化と DPAPI の境界分離
* **バックアップ ZIP (ユーザー資産):**  
  GST が存在しない環境でも外部ファイル圧縮ソフトウェア等で自力復元できるよう、**Standard ZIP 互換形式 + ユーザー指定パスワード（Argon2id + AES-256-GCM）** を採用する（端末固定 DPAPI 単体への依存を排除）。
* **Windows DPAPI (`CurrentUser`):**  
  同一端末内でのみ使用される GST 内部メタデータ、隔離（Quarantine）コンテナの AES 鍵、ローカル監査アンカー（Layer 1）の保護に厳格に限定して使用する。

---

# 10. Safe Mutation Pipeline ＆ 二層整合性モデル

すべての環境変更・データ更新操作は、以下のパイプラインおよび二層整合性モデルを厳格に遵守しなければならない。

```text
User Request (ユーザー操作要求)
       ↓
Operation Classification (操作の分類・リスク評価)
       ↓
Security Policy Validation (Universal Rule Scope 照合)
       ↓
Permission Validation (権限・TOCTOU バリア検証)
       ↓
User Confirmation (破壊的変更時の明示的承認)
       ↓
Audit Creation (OperationId 付番・事前ログ記録)
       ↓
Two-Tier Atomic Execution (二層整合性アトミック実行)
       ↓
Result Verification (結果の整合性検証・確定)
```

### 二層整合性モデル (Two-Tier Consistency Model)
複数ファイル操作（セーブ復元、隔離ファイル一括操作等）において、中途半端なクラッシュや電源断によるデータ破損を防ぐため、以下の二層で安全性を担保する：
1. **第一層（物理アトミック置換）:**  
   各ファイルへの書き込みは直接上書きを厳禁とし、同一ボリューム上の一時ファイル（`.tmp`）へストリーミング出力・完全性ハッシュ照合を行った後、`File.Move(..., overwrite: true)` によりファイル単位で物理的にアトミック置換する。
2. **第二層（冪等な再実行・全体整合性保証）:**  
   複数ファイル処理の中途で電源断やクラッシュが発生した場合、次回起動時のポスト・クラッシュ監査によって中途状態（`.creating`, `.rollback.tmp` 等の残存）を自動検知する。トランザクションジャーナルに基づいて安全なロールバックまたは冪等な再実行を行い、全体としての完全な整合性を回復する。

---

# 11. Security Failure Handling (完全復元)

## 11.1 Failure Principle
安全判断が不能な場合、または例外が発生した場合、**「安全側へ停止（Stop Safely / Fail-Safe）」** を絶対優先する。

禁止事項：
```text
Unknown State (未定義・不明な状態) ──> Force Continue (強制継続・無理な変更)
```

## 11.2 Failure Response (異常時の標準対応シーケンス)
1. 操作の即時安全中断（Operation Pause / Abort）
2. 監査ログへの異常記録（Audit Record）
3. ユーザーへの平易な理由通知（User Notification）
4. 安全なロールバック選択肢の提示（Recovery Option）

---

# 12. Web Link and Launch Boundary (Web リンク起動境界)

## 12.1 Managed Launch Path
GST の URL Broker を経由して起動される管理対象経路において、以下の保護を保証する。

```text
Game / URL Request
        ↓
Parse & Scheme Validation (http / https のみ許可)
        ↓
Strict Query Stripping (クエリパラメータを原則完全破棄)
        ↓
Policy Evaluation (Allow / Confirm / Block / SilentReject)
        ↓
Browser Picker (用途別ブラウザ選択 & 引数インジェクション遮断起動)
```

## 12.2 ブラウザ制御境界
OS 既定ブラウザの強制変更や `UserChoice` レジストリの直接改変、および全 DNS 通信の完全遮断の過剰保証を禁止する。

---

# 13. Clipboard Access Boundary (クリップボード境界)

* **オンデマンド原則:** バックグラウンドでのクリップボード常時監視や自動改変、履歴保存を完全禁止する。
* ユーザーが明示的に「URLを安全化」ボタンを押下した時のみ、STA スレッド上で安全にサニタイズを実行する。

---

# 14. In-Game Overlay HUD Boundary (インゲーム HUD 境界)

* **完全非干渉 (Hookless):** DirectX / Vulkan へのグラフィックフック（プロセス注入）を一切行わず、OS レベルの最前面透過ウィンドウ（`Topmost` ＆ `WS_EX_NOACTIVATE`）として描画する。
* **アンチチート非干渉・互換性境界:** EAC / BattlEye / Vanguard 等を停止・欺瞞・無効化・回避せず、プロセス/DLL注入・メモリ変更・ドライバ操作・グラフィックフックを行わない。Anti-Cheat detection alone は Firewall authorization にならず、互換性例外はユーザーが明示的に有効化した場合に限り GST 所有ルールへ適用する。第三者Anti-Cheatの非検知、接続、誤検知回避、BAN等の enforcement outcome は保証しない。

---

# 15. Strict Safe Launch Boundary (厳格モード起動境界)

未知のインディーゲームや同人ゲームを実行する際、セッション限定で以下のサンドボックス境界を適用する：
1. **In/Out 完全遮断:** 対象プロセスの一時的なインバウンド・アウトバウンド通信強制ブロック。
2. **WER テレメトリ強制抑止:** クラッシュダンプの外部送信を完全ミュート。
3. **残留ルール自己修復:** クラッシュ終了時でも、次回起動時ポスト監査で一時ルールを自動除去。

---

# 16. Final Security Statement

GST は、強制的に環境を支配するツールではない。

GST のセキュリティモデル：

```text
Observe Safely
Control Boundaries
Require Consent
Preserve Ownership
Mutate Atomically
Stop Safely on Failure
Recover Always
```

である。

---

