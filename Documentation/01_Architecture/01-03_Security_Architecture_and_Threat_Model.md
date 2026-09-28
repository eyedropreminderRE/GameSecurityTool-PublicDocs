# 01-03: Security Architecture and Threat Model

**Document ID:** GST-ARCH-BASELINE-002-PART3  
**Version:** 2.4 (Elevated Worker IPC Bootstrap Synchronization)
**Parent Document:** 00_Formal_Baseline_Overview.md  
**Category:** Security Architecture & Governance  
**Status:** Approved Baseline Candidate  

---

# 25. Security Baseline

## 25.1 Security Principles
GST は以下のセキュリティ原則を全設計・実装において維持する。

* **Least Privilege (最小権限):** 必要最小限の権限で動作し、常時管理者権限を要求しない。
* **Secure By Design:** 脅威モデルに基づき、アーキテクチャレベルで攻撃面を排除する。
* **Defense In Depth (多層防御):** 単一の検知・防御層に依存せず、複数の検証層を配置する。
* **Fail Safe (安全側への倒し込み):** 異常系・未定義状態では、許可ではなくブロックまたは保護継続を選択する。
* **Explicit Consent (明示的同意):** ファイル削除・隔離・Firewall変更等の破壊的変更はユーザーの明示的確認を必須とする。
* **Reversible Change (可逆性の担保):** システム変更・隔離・セーブデータ復元は、変更前のスナップショット退避を行い、常に元の状態へロールバック可能とする。

## 25.2 Least Privilege Model
* **通常起動コンテキスト:** **Standard User（標準ユーザー権限）** で動作する。
* **常時管理者権限の禁止:** アプリケーションマニフェストにおける `requireAdministrator` の常時要求を厳禁とする。
* **理由:** 攻撃対象領域（Attack Surface）を最小化し、誤操作やマルウェアによる不正利用時の被害拡大を抑止する。

## 25.3 Elevated Operation (オンデマンド特権昇格)
管理者権限が必要な操作のみ、独立した昇格ワーカープロセス（Out-of-Process UAC Worker）を `runas` 起動する。
* **特権処理の対象:**
  * Windows Firewall ルールの追加・変更・削除
  * システム保護領域（`Program Files` 等）内のファイル操作
  * 復旧処理（System Reversion の一部特権操作）
* **昇格前の UI 説明要求:** UAC ダイアログ表示前に、Presentation 層で必ず以下を明示する：
  * `Operation` (実行する操作名)
  * `Reason` (昇格が必要な理由)
  * `Impact` (システムに与える影響範囲)

---

# 26. Privilege Boundary & Secure IPC

特権境界（Standard User GUI と Elevated Worker）間の通信は物理的に分離し、安全な IPC プロトコルを採用する。

## 26.1 禁止通信方式
* **一時ファイル経由の通信禁止:** レースコンディション（TOCTOU）、改ざん、情報漏洩のリスクがあるため禁止。
* **コマンドライン引数による機密情報受け渡し禁止:** プロセス一覧、WMI、Sysmon / Windows イベントログから引数が平文可視化されるため、機密 DTO、パスワード、認証トークンのコマンドライン引数渡しを厳禁とする。

## 26.2 推奨通信方式とセキュリティ記述子 (DACL)
特権ワーカーとの通信には、DACL（Discretionary Access Control List）で厳格に保護された **Named Pipe** を使用する。

### Bootstrap role/order contract

The authoritative bootstrap sequence is:

```text
GUI generates SessionToken (32 bytes, memory-only)
        ↓
ElevatedWorker.exe launched via approved UAC runas path
        ↓
Named Pipe connection
        ↓
Worker observes client PID
        ↓
Peer Binding: PID + Process StartTime + canonical GUI path
        ↓
Authenticode integrity check (defense-in-depth)
        ↓
Worker sends fresh ChallengeFrame
        ↓
GUI sends BootstrapFrame (SessionToken + HMAC-SHA-256 proof)
        ↓
Worker constant-time proof validation
        ↓
Authenticated IPC session
        ↓
Normal ElevatedWorkerIpcRequest accepted
```

**ChallengeFrame**
- ProtocolVersion
- MessageType = Challenge
- Challenge = 32 bytes
- ObservedClientPid = UInt32
- ObservedClientStartTime = fixed-width UTC FILETIME

**BootstrapFrame**
- ProtocolVersion
- MessageType = Bootstrap
- SessionToken = exactly 32 bytes
- Proof = exactly 32 bytes (HMAC-SHA-256)

The GUI-supplied identity values are not authoritative. Worker-side observed process identity is the only Peer Binding input.

The challenge-response is not described as proof of a secret already pre-shared with the Worker. Peer Binding is the client authorization gate; the challenge binds the bootstrap to the current connection; SessionToken is the transient secret used by the existing authenticated request boundary.

### Named Pipe 通信規約 ＆ DACL 実装:
1. **DACL によるアクセス制限 (`PipeSecurity`):**
   アクセス権限を **Current User SID**, **Local System**, **Builtin Administrators** のみに限定し、同一 PC 上の別ユーザーや非特権マルウェアからの不正接続を物理遮断する。
2. **インバンド・セッショントークン認証（メモリ内ブートストラップ）:**
   親プロセス（GUI）が暗号論的乱数（`RandomNumberGenerator`）で 256-bit のセッション秘密を生成し、ワーカー起動前からメモリ上で所有する。ワーカーは Named Pipe 接続後、まず `GetNamedPipeClientProcessId` で実接続クライアントPIDを取得し、PIDが生存していること、Process StartTime、正規インストール先のGST GUI実行イメージパスを順に検証する。Authenticode検証は追加の実行体完全性確認として扱うが、このBootstrap設計では、未定義の新規Publisher／証明書識別子を導入しない。Peer Binding成功後にのみ、Workerは接続固有の32-byteランダムchallengeを送信する。

   GUIはそのchallengeとWorker観測値（PID + Process StartTime）に拘束されたHMAC-SHA-256 proof、および32-byte SessionTokenを、binary length-delimited BootstrapFrameとして返す。Workerはproofをconstant-time検証し、成功後のみ通常の `ElevatedWorkerIpcRequest` を受理する。BootstrapFrameはJSON/textではなくbinary frameとし、SessionTokenは既存の `byte[]` ownership/zeroization 契約に従う。

   認証用秘密そのものはファイル、Registry、環境変数、コマンドラインへ保存・渡送しない。Worker側のSessionToken copyは認証失敗、Pipe切断、Worker終了、watchdog強制終了等でzeroizeする。

```csharp
// 昇格ワーカー側の Named Pipe DACL 構築実装
var pipeSecurity = new PipeSecurity();

var currentUser = WindowsIdentity.GetCurrent().User;
if (currentUser != null)
{
    pipeSecurity.AddAccessRule(new PipeAccessRule(
        currentUser, PipeAccessRights.ReadWrite, AccessControlType.Allow));
}

var adminSid = new SecurityIdentifier(WellKnownSidType.BuiltinAdministratorsSid, null);
pipeSecurity.AddAccessRule(new PipeAccessRule(
    adminSid, PipeAccessRights.ReadWrite, AccessControlType.Allow));

var systemSid = new SecurityIdentifier(WellKnownSidType.LocalSystemSid, null);
pipeSecurity.AddAccessRule(new PipeAccessRule(
    systemSid, PipeAccessRights.ReadWrite, AccessControlType.Allow));

await using var server = NamedPipeServerStreamAcl.Create(
    "GST_ElevatedWorker_Pipe",
    PipeDirection.InOut,
    NamedPipeServerStream.MaxAllowedServerInstances,
    PipeTransmissionMode.Byte,
    PipeOptions.Asynchronous,
    inBufferSize: 8192,
    outBufferSize: 8192,
    pipeSecurity);
```

---

# 27. File Security & TOCTOU Prevention

ファイル操作において、単なるファイルパス文字列（Path String）のみを信頼した処理を禁止する。

## 27.1 File Operation Security Flow
```text
Resolve Path (正規化・Reparse Point解決)
        ↓
Open Handle (SafeFileHandle で排他オープン)
        ↓
Get File Identity (FileIdInfo / VolumeSerialNumber + FileIndex)
        ↓
Verify Identity & Hash (検証)
        ↓
Execute Operation (オープン済み Handle / 同一検証コンテキストで実行)
```

## 27.2 TOCTOU (Time-of-Check to Time-of-Use) 対策
`Check -> 別処理 -> Path再取得 -> Operate` のような、検証と実処理の間にパスが差し替えられる脆弱な実装を禁止する。

## 27.3 Controlled File Operation & 非NTFSフォールバック規約
* **NTFS 環境:**
  `GetFileInformationByHandleEx` (`FileIdInfo` / 128-bit File ID) を取得して一意性を検証する。
* **非NTFS環境（FAT32 / exFAT / 外付けストレージ等）のフォールバック:**
  `FileId` が取得不能（または 0 を返却）なボリュームでは、**`FileShare.None`（排他ロック）でオープンした `SafeFileHandle` を保持したまま直接ストリーミング処理（Same Handle Operation）** を行う。ハンドルを一度閉じて再オープンすることを禁止する。

---

# 28. Reparse Point Security

Symbolic Link、Junction、Mount Point などの Reparse Point はセキュリティ境界として扱い、パス偽装攻撃（Junction Traversal 等）を防止する。

## 28.1 保持メタデータ
* `OriginalPath`: ユーザーまたはシステムが指定した入力パス
* `ResolvedPath`: シンボリックリンク解決後の物理絶対パス
* `FileId`: ボリューム内一意ファイル識別子
* `IsReparsePoint`: Reparse Point フラグ
* `ValidatedAt`: 検証日時 (UTC)
* `Expiration`: キャッシュ有効期限

## 28.2 Cache Policy
Reparse Point 解決結果の無期限キャッシュ（Infinite Cache）を禁止する。有効期限（TTL）を設け、重要処理（隔離・バックアップ復元）前には必ず再検証（Re-validate）を実施する。

---

# 29. Path Validation Baseline

重要ファイル操作（Scan, Backup, Restore, Quarantine, Trusted Location, Allow List）の実行前に、共通の Path Validation パイプラインを通過させる。

## 29.1 Validation 項目
```text
Path Format Check (無効文字・予約名チェック)
        ↓
Path Exists Check
        ↓
Access Check (現在の権限でアクセス可能か)
        ↓
Reparse Point Check (リンク走査とループ検知)
        ↓
Identity Check (FileId / Volume 一意性検証)
        ↓
Permission / ACL Check (所有者・書き込み権限の検証)
```

## 29.2 Trusted Location のセキュリティ定義
Trusted Location（信頼済みフォルダ）は**「検査スキップ（Skip Scan）」ではなく「Risk Modifier（スコア補正値）」** としてのみ機能する。
* **理由:** 信頼済みフォルダ内に配置されたファイルであっても、DLL Side-Loading や外部からの不正改ざんのリスクが存在するため。

---

# 30. Cryptography Baseline & Key Management

GST では暗号アルゴリズムの独自実装を厳禁とし、OS 提供および .NET 10 標準の暗号化プロバイダを使用する。

## 30.1 暗号技術スタック

| 用途 | 採用技術 | 特性・規約 |
| :--- | :--- | :--- |
| **Quarantine（端末内隔離）** | Windows DPAPI (`CurrentUser`) + AES-256-GCM (64KBチャンクAEAD) | 端末外への流出防止。ローカル端末内でのみ復元可能。 |
| **Save Data Backup（ユーザー資産）** | **Standard ZIP + Argon2id + AES-256-GCM** | **No Vendor Lock-in。PC移行および GST なしの災害復旧に完全対応。** |
| **Migration Package（PC移行）** | ユーザー指定パスワード (Argon2id) + AES-256-GCM | `09_PC_Migration_and_Recovery_Model.md` 準拠の暗号化移送。 |
| **Integrity / Hash** | SHA-256 | 改ざん検知およびファイル同一性検証。 |
| **Audit Chain** | Hash Chain (SHA-256 連鎖) + DPAPI 二重保護 | 監査ログの改ざん・削除・挿入検知。 |

## 30.2 暗号実装における禁止事項
* 独自暗号アルゴリズムの実装
* ハードコードされた固定暗号鍵・ソルトの使用
* 平文での機密データ（Secret / パスワード / 生鍵）の永続化
* MD5 / SHA-1 等の脆弱なハッシュアルゴリズムの利用

## 30.3 Secret Lifecycle & メモリ保護 (CRIT-06 是正規約)
暗号鍵や生パスワードを保持するバッファの管理について、以下の絶対規約を設ける。

* **`string` 型パスワード保持の完全禁止:** .NET の `string` はイミュータブルであり、GC に回収されるまでメモリ上に平文パスワードが残存し続ける。Contracts 境界の DTO や Interface メソッドでパスワードを `string` 型で受け渡すことを厳格に禁止する。
* **安全な型への置換:** パスワードは必ず `byte[]`（または `Memory<byte>`, `SecureString`）として受け渡すこと。UI で入力を受け取った瞬間に配列化し、以降の層に引き回す。
* **メモリの即時破棄:** 鍵導出（Argon2id等）や復号処理が完了した後は、必ず `finally` ブロック内で `Array.Clear(passwordBytes)` または `CryptographicOperations.ZeroMemory(passwordBytes)` を呼び出し、メモリダンプによる平文パスワード流出を物理的に防ぐこと。

---

# 40. Allow List Baseline

Allow List はユーザーが明示的に信頼した対象を管理する機能である。Allow List 登録によるセキュリティエンジンの完全スキップを禁止し、Risk Modifier として扱う。

## 40.1 Allow List Scope
共通 Rule Scope Model に従い、`Global` および `GameProfile` の2階層で管理する。

## 40.2 識別強度の優先順位
```text
1. Hash Allow (SHA-256 による完全一致・最強)
2. Signature Allow (証明書署名者による検証)
3. Path Allow (特定絶対パス配下の許可・注意対象)
4. Name Allow (ファイル名のみによる許可・最弱)
```

## 40.3 Allow List Security Rules
* **UI 警告の義務化:** `Path Allow` または `Name Allow` を登録する際、ファイル差し替えによるなりすましリスクがある旨を UI で警告する。
* **自動登録の禁止:** ユーザーの明示的確認なしに自動で Allow List へ追加することを禁止する。
* **Temporary Allow (一時許可):** `Permanent`, `1 Hour`, `24 Hours`, `Until Next Startup` をサポートし、期限切れ時は物理削除ではなく履歴保持の上で無効化（Disabled）する。

---

# 41. Trusted Location Baseline

* **Trust Modifier の動作例:**
  * 未知のパス（Unknown Path）: `Risk Score +30`
  * ユーザー承認済み場所（Trusted Location）: `Risk Score -10`
* **Critical Signal による無効化:** マルウェアシグネチャや改ざんが検知された場合（Critical Signal）、Trusted Location によるスコア減算は完全に無視（バイパス）される。

---

# 46. Evidence Based Security Baseline

セキュリティ検知は単なる判定結果（True/False）だけでなく、改変不能な証拠データ（Evidence）を伴って記録する。

### 構成要素:
```text
Detection Event
  ├─ File Metadata (Size, Path, Timestamps, FileId)
  ├─ Hash & Signature Information
  ├─ Configuration Snapshot (検知時の設定スナップショット)
  └─ Audit Hash Chain Reference (監査レコードへの参照リンク)
```

---

# 47. Explanation Engine Baseline

Explanation Engine は **Application Layer** に配置し、Domain 層の純粋性を保護する。

* **Domain 層の責務:** `RiskLevel`、`RiskScore`、`Confidence`、`EvidenceCollection`、`DetectionSignals` を保持するのみ（UI 文言・言語リソースを一切持たない）。
* **Application 層 (Explanation Engine) の責務:** Domain の判定結果とシグナルを解釈し、多言語リソースを適用して `RiskExplanationDto` を生成する。

---

# 48. Confidence Model Baseline

セキュリティ判定を「安全 / 危険」の2値に単純化せず、確信度（Confidence Score: 0.0〜1.0）と危険度（RiskLevel: Low/Medium/High/Critical）を組み合わせて評価する。

### 確信度モデルの運用規約:
* Confidence Score は UI での推奨アクション提示およびユーザー判断支援に利用する。
* 低い確信度（例: Confidence 30% の High Risk）を理由に、独断での自動隔離・自動削除を行うことを厳禁とする。
```

---
