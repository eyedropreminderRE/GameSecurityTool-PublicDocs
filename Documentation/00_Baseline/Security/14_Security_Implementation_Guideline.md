# GameSecurityTool

# Security Implementation Guideline

## Secure Coding / System Access / Protection Implementation Rules

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-014 |
| Version | 3.2 (Secret Transit & Global Log Sanitization Boundary Edition) |
| Status | Formal Baseline Specification (Highest Secure Coding Authority) |
| Category | Security Implementation |
| Authority Level | Coding Security Standard |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- セキュアコーディング基準
- ファイル操作およびアトミック変更原則
- レジストリおよび OS アクセス規約
- Windows 権限モデルと特権分離 IPC
- Firewall 実装規則
- DPAPI および認証情報・暗号鍵のメモリ保護（Zeroization）
- 暗号化ルール（Argon2id + AES-256-GCM）
- 外部プロセス起動および引数インジェクション防止
- 一時データおよびアップデートセキュリティ
- AI 生成コードのセキュリティ審査

の実装規則を定義する。

---

GST では、

```text
Correct Function  *  Safe Execution  *  Zero Memory Leakage  *  Atomic Mutation
(正しい機能、安全な実行、メモリ上の秘密情報漏洩ゼロ、完全なアトミック変更)
```

を同時に満たすことを要求する。

---

# 1. Security Implementation Philosophy

## 1.1 Core Principle
実装では、便利さや開発速度よりも安全性を絶対優先する。

基本原則：
```text
Never Trust External Input (入力を無条件に信頼しない)
Never Assume Permissions   (権限を勝手に仮定しない)
Never Hide Change          (変更を隠蔽しない)
Never Mutate Files In-Place Directly (直接上書きを行わない)
```

---

# 2. Secure Coding Rules

## 2.1 Input Validation (入力検証)
すべての外部入力（ファイルパス、プロセス名、URL、JSON、アーカイブエントリ）は無条件に信頼せず、必ず正規化と境界検証を行う。

## 2.2 Null and Invalid State Handling (無効状態の処理)
予期せぬ Null や不正な状態を検知した場合、処理を強行せず、検証 ➔ ハンドリング ➔ ユーザー通知の安全フローを実行する。

## 2.3 Exception Handling (例外処理規約)
1. **空 catch の厳禁:** `catch (Exception) { }` で例外を握りつぶすことを厳禁とする。必ず分類・ログ記録・安全なフォールバックを行う。
2. **Result パターンの採用:** 業務ユースケースにおいて、予期される失敗（ファイルロック、ハッシュ不一致、容量不足等）は例外ではなく `Result<T>` または結果 DTO で返却する。
3. **例外メッセージの機密性:** 例外メッセージやスタックトレースに生パスワード、トークン、未加工 URL を含めてはならない。

---

# 3. File System Security & Atomic Mutation

## 3.1 File Access Principle
ファイル操作は、UI や Domain から直接 API を呼ばず、必ず Application Service / Contracts Port 経由で実行する。

## 3.2 Path Validation & TOCTOU 防御
1. パス操作直前に `IPathResolver` 経由で Win32 `GetFinalPathNameByHandle` を呼び出し、ジャンクションやシンボリックリンクによる境界外脱出を遮断する。
2. Check と Use の間にパスが差し替えられる脆弱な実装を禁止し、同一ハンドル上での検証・操作（Same Handle Operation）を徹底する。

## 3.3 Path Traversal & Zip Slip 防御
ユーザー入力パスの正規化を行い、境界外アクセスを遮断する。アーカイブ展開時は相対パス（`../`）を含むエントリを構造的に検知し即時拒否する。

## 3.4 File Deletion & Atomic Mutation Rule (C-3, C-4 規約)
削除処理は最高リスク操作として扱い、必ず事前確認・監査ログ記録を伴う。  
また、セーブ復旧（`RollbackAsync`）や PC 移行再封緘（`ImportAndResealQuarantineAsync`）において、既存ファイルを直接開いてインプレース上書きすることを厳禁とし、**「一時ファイル（`.tmp`）出力 ➔ 検証 ➔ `File.Move(overwrite: true)` によるアトミック置換」** を必須実装パターンとする。

## 3.5 【確定】Win32 `SetThreadExecutionState` によるトランザクション中スリープ抑止パターン
セーブデータ復元や暗号化隔離などの破壊不可能なトランザクション実行中、OS がスリープやスタンバイに突入してディスク I/O が中途半端に中断・破損するのを防止する：
```csharp
[LibraryImport("kernel32.dll", SetLastError = true)]
private static partial uint SetThreadExecutionState(uint esFlags);

private const uint ES_CONTINUOUS = 0x80000000;
private const uint ES_SYSTEM_REQUIRED = 0x00000001;

// トランザクション保護パターン
SetThreadExecutionState(ES_SYSTEM_REQUIRED | ES_CONTINUOUS);
try
{
    // 不可分ファイル I/O または暗号化トランザクション実行
}
finally
{
    // 完了時に確実に抑止解除
    SetThreadExecutionState(ES_CONTINUOUS);
}
```

## 3.6 【確定】Win32 `RestartManager` によるファイルロック競合プロセスの特定 (`IFileLockInspector`)
セーブ復元や暗号化コンテナ操作時にファイル共有違反（`ERROR_SHARING_VIOLATION: 0x80070020`）が発生した場合、単なるエラー終了とせず、Win32 `Restart Manager API`（`RmStartSession`, `RmRegisterResources`, `RmGetList`, `RmEndSession`）を呼び出してロックを握っているプロセスを特定する：
1. **他社セキュリティソフト干渉の識別:** ロックプロセス名がウイルス対策ソフト（`mcshield.exe`, `bdservicehost.exe`, `avp.exe` 等）であるかを判定。
2. **原因の透明な提示:** 「⚠️ 外部セキュリティツールによりファイルが一時ロックされています」とユーザーへ理由を明示し、数秒間の待機後に自動再試行する。

---

# 4. Save Data Protection (完全復元)

## 4.1 Ownership Rule
セーブデータはユーザーの完全な資産である。GST は保護、バックアップ、復元支援のみを担当し、所有権を取得しない。

## 4.2 Modification Restriction
以下を厳格に禁止する：
- セーブデータの自動編集・改変
- セーブデータの無断自動クリーンアップ
- 未知のセーブファイルの勝手な削除

---

# 5. Registry Access Rules (完全復元)

## 5.1 Registry Principle
レジストリ走査は必要最小限とし、OS 全域のフルスキャンを禁止する。

## 5.2 Registry Modification
レジストリを変更する際は、以下を義務付ける：
1. 目的の明示
2. 変更前の状態（BeforeState）のディスクジャーナル（`WerBeforeState.json` 等）への事前退避
3. 監査ログ記録
4. 終了時およびクラッシュ再起動時における元値への確実な復元

## 5.3 Registry Ownership
GST 管理外のレジストリキーはすべて Read-Only として扱う。

---

# 6. Windows Permission Model & Privilege Isolation

## 6.1 Least Privilege Principle
通常起動コンテキストは常に **Standard User（標準ユーザー権限）** で動作し、常時管理者権限を要求しない。

## 6.2 Elevation Rule & IPC Security
管理者権限が必要な操作は、ユーザーの明示的承認を経て独立した昇格ワーカー（`ElevatedWorker.exe`）へ委譲する。
- `PipeSecurity` DACL による CurrentUser / SYSTEM / Admin 限定
- Main GUI が生成する 32-byte (256-bit) SessionToken の memory-only bootstrap
- Workerによる `GetNamedPipeClientProcessId` → PID / Process StartTime / canonical GUI pathのPeer Binding（PID単独を信用しない）
- Authenticodeによる実行体完全性の追加確認。未定義のGST Publisher / certificate identifierを独自追加しない
- Peer Binding成功後のみ発行する接続固有32-byte challenge
- binary length-delimited ChallengeFrame / BootstrapFrame と、SessionTokenに拘束されたHMAC-SHA-256 proof
- 認証プルーフの定数時間比較（`CryptographicOperations.FixedTimeEquals` 相当）
- `worker.token` 等のファイル・Registry・環境変数・CLI bootstrap を禁止
- JSON / text encodingによるbootstrap frameを禁止し、Wire契約の固定長secret/challenge/proofを維持
- 60 秒アイドル自己終了
- グローバル Mutex（`Global\GST_ElevatedWorker_Mutex`）による多重起動防止
- UAC 昇格キャンセル時の 30 秒間プロンプト抑制バックオフ

## 6.3 Elevated Operation Logging
特権ワーカーが実行したすべての操作（Firewall ルール変更、保護領域アクセス）は必ず監査ログに記録する。

---

# 7. Firewall Implementation Rules (完全復元)

## 7.1 Firewall Ownership
GST が作成したルールのみを管理対象とし、`CreatedBy = GST`、`RuleTag`、およびローカル DB レコードとの突合により識別する。

## 7.2 Firewall Modification
ルールの追加・削除は、ゲームの存在検証 ➔ ルール検証 ➔ ユーザー承認 ➔ UAC 委譲 ➔ 監査ログコミットの順序で実行する。

## 7.3 External Rule Protection
他社ソフトウェアや手動で作成された外部ルールの自動変更・自動削除を厳格に禁止する。

---

# 8. Memory Safety & Secret Lifecycle (秘密情報のメモリ保護)

## 8.0 Gemini API key transport boundary
`IAiConsentRepository.SaveConsentAndApiKeyAsync` receives the API key as caller-owned `byte[]`. The caller zeroizes its buffer in `finally` after completion. Infrastructure zeroizes only internal temporary secret buffers that it creates. A transient textual representation required for final HTTPS header construction is confined to the Infrastructure transport boundary and must never cross Contracts, persist, log, or enter UI state.

## 8.1 `string` 型パスワード保持の完全禁止
.NET の `string` はイミュータブルで GC 回収までヒープに残存するため、パスワードや暗号鍵を `string` 型で保持・受け渡しすることを厳禁とする。

## 8.2 `byte[]` 受領と即時消去 (Zeroization)
1. パスワード等の秘密情報は必ず `byte[]`（または `SecureString`）として扱う。
2. 鍵導出や暗号処理の完了直後に `finally` ブロックで `CryptographicOperations.ZeroMemory(passwordBytes)` または `Array.Clear(passwordBytes, 0, passwordBytes.Length)` を実行して物理消去する。
3. **Caller-Owns 所有権契約 (H-2 規約):** マイグレーションマスターキー等の複数 Port に跨る鍵は、呼び出し元（Caller: `MigrationCoordinator`）が所有権を持ち、全処理完了後に Caller が `finally` で `ZeroMemory` する。各 Port メソッド（Callee）は自身が内部生成した一時鍵のみを消去する。

## 8.3 Windows DPAPI の適用境界
Windows DPAPI (`CurrentUser`) は、GST 内部のローカルメタデータ、隔離（Quarantine）コンテナの鍵、ローカル監査アンカーの保護にのみ限定して使用する。

---

# 9. Cryptography Baseline (製品統一暗号仕様)

暗号アルゴリズムの独自実装を厳禁とし、OS および .NET 10 標準プロバイダを使用する。

| 用途 | アルゴリズム / 方式 | 統一パラメータ仕様 |
| :--- | :--- | :--- |
| **Password KDF** | **Argon2id** (`Konscious.Security`) | **Iterations = 3, Memory = 64 MB (65536 KB), Parallelism = 4, Salt = 16 Bytes** |
| **Payload 暗号化** | **AES-256-GCM (Chunked AEAD)** | **KeySize = 256-bit, Nonce = 12 Bytes (チャンク別一意乱数), Tag = 16 Bytes, ChunkSize = 64 KB** |
| **Local Secret 保護** | **Windows DPAPI** | **Scope = DataProtectionScope.CurrentUser (端末固定メタデータ・隔離鍵のみ)** |
| **完全性 / 改ざん検知** | **SHA-256** | **256-bit Hash, 秒精度正規化 (UnixTimeSeconds)** |

---

# 10. Archive and Backup Security

## 10.1 Archive Validation
アーカイブの完全性、ヘッダー構造、マニフェストハッシュを検証する。

## 10.2 External Extraction Compatibility (No Vendor Lock-In)
バックアップ ZIP は標準形式（Standard ZIP + Argon2id + AES-256）を採用し、GST なしでも外部ファイル圧縮ソフトウェア等で自力復元可能とする。

## 10.3 Backup Password
GST はバックドアやパスワード復旧機能を持たない（ユーザー自身による厳格な自己管理）。

---

# 11. External Process Handling ＆ 外部ツール安全呼出指針

外部プロセス（外部ブラウザ、外部ファイル圧縮ソフトウェア、外部セキュリティスキャナ等）を呼び出す際は、OS レベルの引数インジェクションや権限昇格攻撃を防止するため、以下の実装規則を厳格に適用する。

## 11.1 Process Launch ＆ 実行ファイル検証
1. **パスのホワイトリストおよび正規化検証:**  
   呼び出し対象のバイナリパス（外部ファイル圧縮ソフト、Windows 標準セキュリティ `MpCmdRun.exe` 等）は、事前にホワイトリスト化された信頼済みディレクトリ（`C:\Program Files`, `C:\Program Files (x86)`, `C:\Windows\System32` 等）内に存在することを `IPathResolver` で正規化検証する。
2. **デジタル署名・完全性検証:**  
   実行前に Win32 `WinVerifyTrust` または SHA-256 ハッシュを検証し、未署名または改ざんされた偽装バイナリの起動を遮断する。

## 11.2 Command & Argument Injection Prevention
1. **文字列連結による引数生成の厳禁:**  
   `ProcessStartInfo.Arguments` に文字列連結（`exe + " " + userInput`）で引数を構築することを厳禁とする。
2. **`ArgumentList` の必須採用:**  
   必ず `ProcessStartInfo.ArgumentList.Add(...)` を使用し、各パラメータを独立したエントリとして OS に渡すことで、空白や記号（`&`, `|`, `;`, `"`, `'` 等）によるコマンド連結・引数インジェクションを物理遮断する。
3. **オプション終端セパレータ (`--`):**  
   外部ブラウザ等への URL 渡しでは、オプション終端セパレータ（`--`）または専用パラメータフラグ（`-url`）を明示的に付与する。
4. **プロセス実行環境の隔離:**  
   `ProcessStartInfo.UseShellExecute = false`、`CreateNoWindow = true` を設定し、意図しないシェル機能の呼び出しや環境変数インジェクション（`PATH`, `PATHEXT` 等の汚染）を排除する。

## 11.3 【確定】安全起動ショートカット生成 (`ISafeShortcutGenerator`) ＆ 外部スキャナ安全呼出
1. **安全起動ショートカット (`.lnk`) の安全生成:**
   Windows Script Host（`WScript.Shell`）COM 経由でショートカットを生成する際、引数は固定書式（`--launch {GameProfileId} --mode strict`）のみを許可し、ユーザー入力文字列を引数へ直接展開しない。
2. **外部セキュリティスキャナ呼出のスケジューリング:**
   Windows 標準セキュリティ（`MpCmdRun.exe`）や外部スキャンツールを呼び出す際は、全画面ゲームプレイ中の実行を延期し、PC アイドル時または明示的な手動要求時のみに限定する。

---

# 12. Logging Security

1. **機密情報の完全排除:** パスワード、暗号鍵、Gemini API キー、生 URL クエリ、クリップボード本文のログ出力を禁止する。
2. **Global Sanitizing Logger Provider:** すべての通常 `ILogger` / `LoggerMessage` 出力は、物理Sinkへ到達する前にGlobal Sanitizing Logger Provider / sink gatewayを必ず通過させる。Message、structured state、scope、Exception情報をsanitization対象とし、raw `Exception` objectおよびraw structured stateを下流Sinkへ渡さない。
3. **自動マスキング:** Global Sanitizing Logger Provider内で `PrivacyLogSanitizer` / `ILogSanitizer` を使用し、実名ユーザー名および個人パスを `C:\Users\***\...` へ自動伏字化する。
4. **Secret prohibition:** パスワード、暗号鍵、Gemini APIキー、生URL query、clipboard本文等は、sanitizerの存在にかかわらずログへ渡すこと自体を禁止する。Sanitizationはdefense-in-depthであり、秘密loggingの許可ではない。
5. **Sink consistency:** すべての通常Sinkはsanitized recordだけを受領し、新しいSink追加時も直接unsanitized provider pathを作らない。

---

# 13. Temporary Data Handling (完全復元)

## 13.1 Temporary Files
一時ファイル（`.tmp`, `.rollback.tmp`, `.reseal.tmp`）は制御されたディレクトリ（`%LocalAppData%\GameSecurityTool\`）に出力し、処理完了後またはキャンセル時に `finally` で即座に安全消去する。

## 13.2 Temporary Secret Data
メモリ上の一時秘密データは不要になった瞬間に `ZeroMemory` し、ディスク上へ平文のまま放置することを厳禁とする。

---

# 14. Update Security (完全復元)

## 14.1 Update Validation
アップデートパッケージの取得元、ハッシュ、デジタル署名を検証する。

## 14.2 Update Failure
更新失敗時は直前の安定版へ安全にロールバックし、ユーザーのセーブデータや設定を一切破壊しない。

---

# 15. Secure Development Checklist

実装レビュー時確認：

☐ すべての入力パスが `IPathResolver` で検証され、TOCTOU が排除されていること  
☐ **セーブ復元・PC 移行再封緘が一時ファイル経由でアトミックに実行されていること**  
☐ **パスワードや暗号鍵が `string` 型で保持されず、`byte[]` かつ `finally` で `ZeroMemory` されていること**  
☐ 暗号化パラメータが Argon2id (Iterations=3, Memory=64MB) / AES-256-GCM (64KB Chunked) に統一されていること  
☐ 特権ワーカー通信が DACL 保護 Named Pipe および定数時間トークン検証を行っていること  
☐ 通常ログがGlobal Sanitizing Logger Providerを経由し、例外・structured state・scopeを含めて `ILogSanitizer` で安全化されていること
☐ raw `Exception` object / raw structured stateがphysical Sinkへ渡っていないこと  
☐ データベース書き込みがすべて `IDbWriteQueue` 経由で行われていること  

---

# 16. AI Generated Code Security Review (完全復元)

AI 生成コードのレビュー時確認：
- ファイルアクセスが安全な抽象レイヤー（Port）を経由しているか
- 不要な管理者権限や OS 具象 API を要求していないか
- 秘密情報が平文で保存またはメモリ残存していないか
- Clean 5-Layer レイヤー境界をバイパスしていないか

---

# 17. Security Testing (完全復元)

必須セキュリティテスト項目：
- 権限昇格テスト（Named Pipe DACL・トークン検証）
- 障害耐性テスト（復元・再封緘中の中途クラッシュ時のアトミック性検証）
- 改ざん検知テスト（Layer 1 / Layer 2 アンカー検証）
- マイグレーション再封緘テスト
- OOM 耐性テスト（巨大ファイル処理時メモリ < 120MB）

---

# 18. Relationship With Other Models (他モデルとの関係性 - 完全復元)

## 18.1 Architecture Governance
開発ガバナンスと Clean 5-Layer 結合規則：
```text
07_Architecture_and_Development_Governance.md
```

## 18.2 Service & Interface Model
Contracts Port インターフェース定義：
```text
13_API_and_Service_Interface_Model.md
```

## 18.3 Privacy Model
プライバシー保護および 11 大脅威モデル：
```text
06_Privacy_and_Threat_Model.md
```

## 18.4 PC Migration & Recovery
認証移行とアトミック再封緘：
```text
09_PC_Migration_and_Recovery_Model.md
```

## 18.5 Technology Stack & Package Standards
全社統一暗号化基準および外部パッケージ：
```text
01_Architecture/01-02_Technology_Stack_and_Runtimes.md §57
```

---

# 19. Final Security Statement

GST の実装は、機能を実現するだけでは不十分である。

---

安全な実装とは、

```text
Expected Behavior  +  Strict Memory Safety  +  Atomic Mutation  +  Recoverable Failure
(期待通りの動作、厳格なメモリ安全性、アトミックな変更、復旧可能な障害耐性)
```

を満たすことである。

---

Final Principle:

```text
Protect Data Always
Zeroize In-Memory Secrets
Mutate Files Atomically
Never Fail Silently
```

---

