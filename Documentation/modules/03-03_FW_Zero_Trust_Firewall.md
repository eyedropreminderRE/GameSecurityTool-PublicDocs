# 03-03: FW - Zero Trust Firewall & Network Specification

**Document ID:** GST-MOD-FW-003  
**Version:** 4.7
**Status:** Approved Module Specification  
**Target Projects:**
- `GameSecurityTool.Domain`
- `GameSecurityTool.Contracts`
- `GameSecurityTool.Application`
- `GameSecurityTool.Infrastructure`

> **Lifecycle note:** This module specification describes approved design scope. Its presence does not by itself indicate that the module is implemented, Windows-verified, or released.

---

# 1. モジュール概要 ＆ 責任境界

本モジュールは、**Game Profile 単位の Default Outbound Block、Out-of-Process UAC Worker（DACL保護 Named Pipe IPC）経由の特権操作、memory-only セッション秘密と接続固有チャレンジによるIPC認証ブートストラップ、UAC プロンプト拒否バックオフ制御（H5 是正）、特権 IPC デッドロック監視（10秒ハードタイムアウト Watchdog ＆ ゾンビワーカー強制終了・オンデマンド再起動）、60秒アイドル自己終了、Authenticode 署名検証によるアンチチート安全共存、OS 実ルール取得による Drift Detection バッチ検証、Steam IPC (127.0.0.1) 自動免除、Web限定 (HTTP/80, HTTPS/443) モード ＆ カスタムポート除外、COM RCW 明示解放、および【多層監査アンカー (Layer 2) の安全な読み書き代行】** を担当する。

### Clean 5-Layer レイヤー境界:
- **Domain Layer (`GST.Domain`):** Firewall ルール整合性評価、AntiCheat 判定ルール。OS/COM 依存ゼロ。
- **Contracts Layer (`GST.Contracts`):** `IFirewallManager`, `IAntiCheatDetector`, `IElevatedWorkerClient` Port インターフェースおよび契約 DTO 群（外部依存ゼロ）。機密バッファは例外的に可変 `byte[]` を使用し、所有権契約で管理する。
- **Application Layer (`GST.Application`):** Firewall 適用ワークフロー、学習モード（Learning Mode）承認調停、Drift 通知。
- **Infrastructure Layer (`GST.Infrastructure`):** Main プロセス側 Named Pipe クライアント（`WindowsFirewallManager` - UAC バックオフ ＆ 10秒 IPC Watchdog 機能付き）、昇格ワーカー側サーバー（`PipeSecurity` DACL 構築 ＆ memory-only セッション認証ブートストラップ検証 ＆ 60秒アイドル自己終了 ＆ `INetFwPolicy2` COM API 操作・RCW明示解放）、`AntiCheatDetector`（Authenticode 署名検証）。

---

# 2. 機能要件仕様 (Functional Requirements)

## 2.1 `FN-FW-01`: 特権分離 Out-of-Process UAC Worker (DACL保護 Named Pipe & memory-only 認証ブートストラップ)
1. コマンドライン引数、環境変数、Registry、または一時ファイルによる認証秘密のbootstrapを厳禁とする。
2. Main GUI は接続単位の暗号論的256-bitセッション秘密を生成し、ディスクへ保存せずメモリ上で所有する。
3. Named Pipe 接続確立後、Worker は `GetNamedPipeClientProcessId` で実接続クライアントPIDを取得し、PIDが生存していること、Process StartTime、正規インストール先のGST GUI実行イメージパスを検証する。Authenticode検証は実行体完全性の追加確認として扱う。Peer Binding成功後、Workerは32-byteの接続固有challengeを送信し、GUIはchallenge・Worker観測PID・Worker観測StartTimeに拘束されたHMAC-SHA-256 proofと32-byte SessionTokenをbinary BootstrapFrameで返す。Workerはproofをconstant-time検証し、成功後のみ通常IPCを受理する。`ElevatedWorkerIpcRequest.SessionToken` のContracts上の正規型、caller-owned buffer所有権、およびzeroization責務はdocumented security and verification referenceで確定し、bootstrap wire contractはdocumented security and verification referenceで確定済み。
4. 昇格ワーカー側で `PipeSecurity` を構築し、アクセス権限を **Current User SID / LocalSystem / Administrators** のみに物理限定。
5. **60秒アイドル自己終了:** 接続待機中または処理完了後に 60 秒間新規リクエストがない場合、自己終了する。
6. **多重起動防止:** グローバル Mutex（`Global\GST_ElevatedWorker_Mutex`）を用いて、システム全体で常に単一の特権ワーカープロセスのみが存在することを保証する。
7. **UAC プロンプト拒否バックオフ制御 (H5 是正):** ユーザーが UAC 昇格をキャンセルした場合、またはワーカー起動に失敗した場合、最低 30 秒間は自動再プロンプトを抑制し、UAC 疲弊を防止する。

## 2.2 `FN-FW-02`: ローカル DB 突合による所有権判定 ＆ OS 実ルール Drift Detection
1. `IFirewallManager.GetOsActualRulesAsync` 経由で実 Windows Firewall 上のルールを一括取得。
2. ローカル SQLite DB（`FirewallRuleRecords`）に該当レコードが存在するかを突合。
3. DB に存在しないルールは「Foreign Rule」としてマークし、自動削除対象から除外（手動承認必須）。
4. ルールの有効状態、ポート、アドレスの外部変更を検知（Drift Detection）し、監査ログへ記録。

## 2.3 `FN-FW-03`: ルール適用範囲の `.exe` 限定フィルタリング
DLL はホストプロセス（`.exe`）のブロックで包含遮断されるため、OS ネットワークスタックの評価遅延を防ぐ目的で、ルール生成対象を `.exe` のみに厳格フィルタリングする。

## 2.4 `FN-FW-04`: Anti-Cheat Compatibility Detection & User-Controlled Exception (`AntiCheatDetector`)
1. ゲームフォルダ内の Anti-Cheat 候補（`EasyAntiCheat`, `BattlEye`, `vgc.exe` 等）を互換性シグナルとして検出する。
2. 候補ファイルは Win32 `WinVerifyTrust` により Authenticode 署名を検証する。署名検証は検出シグナルの信頼性確認であり、Firewall 変更権限を自動付与するものではない。
3. **自動免除は禁止:** Anti-Cheat の存在または署名有効性だけを理由に Outbound Block を自動免除してはならない。
4. ユーザーが明示的に Compatibility Exception を有効化した場合のみ、登録されたゲーム/実行ファイルについて **GST が作成・所有する Firewall ルールだけ**を対象に互換性例外を適用できる。Foreign Rule や第三者ルールは変更しない。
5. GST は第三者 Anti-Cheat を停止・欺瞞・無効化・回避せず、プロセス注入、DLL 注入、メモリ変更、ドライバ操作、DirectX/Vulkan フック等を行わない。
6. GST は Anti-Cheat の非検知、接続成功、誤検知回避、アカウント警告・停止・BAN 等の第三者 enforcement outcome を保証しない。
7. 製品表記では `Anti-Cheat Compatibility Exception` を使用し、`BAN 回避` / `Anti-Cheat bypass` を機能名・動作保証として使用しない。


## 2.5 `FN-FW-05`: 127.0.0.1 免除のインテリジェント制御
1. **公式ゲーム (Steam 等):** Steam クライアントとゲーム本体間の IPC および DRM 認証を阻害しないため、ルールの `RemoteAddresses` に `"1.0.0.0-126.255.255.255,128.0.0.0-255.255.255.255"` を指定し、`127.0.0.0/8` を意図的にブロック対象から除外する。
2. **フリーゲーム・同人ゲーム ＆ 厳格モード:** トンネリング攻撃やローカルプロキシ迂回を防ぐため、`127.0.0.1` もブロック対象に含め、ループバックを含む全インバウンド・アウトバウンド通信を完全に遮断する。

## 2.6 `FN-FW-06`: AI 推奨 Firewall ルールワンクリック適用 ＆ ガードレール
1. **推奨ルールの受領:** Gemini AI コンシェルジュやトラブルシュート機能から提示された推奨 FW ルール（必要なマルチプレイ用ポートのみを開放するルール等）を受領。
2. **安全ガードレールと微調整 UI:**
   - AI 推奨ルールを直接即時適用することを禁止し、必ず「自己責任免責ダイアログ」を表示する。
   - ユーザーがポート番号、プロトコル（TCP/UDP）、通信方向（Inbound/Outbound）を目視確認し、微調整できるレビューモーダルを提供する。
   - 承認後に `IFirewallManager.CreateBlockRuleAsync` または許可ルール適用を実行し、監査ログへ記録する。

## 2.7 `FN-FW-07`: 特権 IPC デッドロック防止とゾンビワーカー強制終了・オンデマンド再起動 (IPC Watchdog Guard)
1. **10秒ハードタイムアウト:** Main GUI 側から Named Pipe への接続およびトランザクション処理（`ExecuteIpcTransactionAsync`）に厳格な 10 秒のウォッチドッグ（Watchdog）タイマーを設定。
2. **ゾンビプロセス強制終了:** Windows ファイアウォール COM API 呼び出し中の OS 側デッドロック等により 10 秒を超過した場合、パイプ接続を即座に破棄・切断し、応答不能となった特権ワーカープロセス（`ElevatedWorker.exe`）を `Process.Kill()` により物理強制終了する。
3. **オンデマンド再起動 (Clean Re-spawn):** ゾンビワーカー終了後、次回 IPC リクエスト時またはリトライ時に新規ワーカーをオンデマンドで再起動・再接続し、Main GUI 全体のハングアップを防止する。

## 2.8 `FN-FW-08`: ポート除外 ＆ Web限定 (HTTP/80, HTTPS/443) 通信制御ルール
1. **Web限定モード (Web-Restricted Mode):** ゲームまたはツールの外部通信を HTTP(80) および HTTPS(443) のみに限定し、未知の独自プロトコルや非標準ポートを用いた C2 通信・情報漏洩をブロックする。
2. **カスタムポート除外 (Port Exclusion):** マルチプレイヤーゲームや専用サーバー連携に必要な特定ポート（例: 27015, 7777 等）をユーザーが明示的にホワイトリスト除外設定可能とする。
3. **一時的厳格ルール (`GST:STRICT-TEMP:`) の自動クリーンアップ:** 起動前遮断や検証フェーズで適用された一時的ルールを、ゲームセッション終了時やシステム復帰時に漏れなく一括検出・削除する。

## 2.10 FN-FW-10: End-User Block Explanation & Change Confirmation Boundary (approved security contract)

Firewallの標準ユーザー導線は、個別rule CRUDを前提とせず、遮断イベントからユーザー判断までを次の順序で扱う。

1. **Explain:** 対象GameProfile、対象`.exe`、通信方向、適用ポリシー/ルール、想定される影響、現在利用可能な変更操作を表示する。
2. **Review:** 通信制御や例外変更によって保護範囲が変わる場合、変更前の状態と変更後の想定状態を確認できるようにする。
3. **Confirm:** OS Firewall状態を変更する操作は明示的なユーザー確認後にのみ実行する。
4. **Execute:** 既存の`IFirewallManager`契約を利用して変更を適用する。未定義の「今回のみ」等のスコープを勝手に導入しない。
5. **Verify:** 適用結果と実OS状態を確認する。部分適用・失敗は既存の`FN-FW-09` Reconcile契約に従い、ユーザーへ要確認として提示する。

Anti-Cheat Compatibility Exceptionはapproved security contractの明示承認境界を維持し、検出だけで免除を適用しない。このUX項目は新規Port/DTO/Enumを導入するものではない。

## 2.9 FN-FW-09: Firewall Batch Outcome & Reconciliation Contract

`ApplyBatchRulesAsync` は部分適用を許容するが、部分適用を成功とは扱わない。既存の `Task<bool>` 契約と `ElevatedWorkerIpcResponse` を維持し、以下を必須とする。

1. **完全成功:** 要求された追加・削除操作がすべて成功し、要求件数と `AddedCount` / `RemovedCount` の結果が一致した場合のみ `Success=true` とする。
2. **部分適用・スキップ・個別失敗:** 1件でも失敗、スキップ、入力不備、または要求件数と実績件数が一致しない場合は `Success=false` とする。`AddedCount` / `RemovedCount` は実際に成功した件数を返す。
3. **自動ロールバック禁止:** 部分適用時に成功済み操作を自動的に逆操作して全件を元へ戻すことは、この契約では要求しない。
4. **Reconcile 必須:** `Success=false` を受けた呼び出し側は `GetOsActualRulesAsync` で Windows Firewall の実状態を再取得し、GST管理状態との差分を再評価する。Reconcile はバッチIPC処理の外側のワークフロー境界で実行する。
5. **監査・通知:** 部分適用またはスキップが発生した場合、実績件数・エラー情報・Reconcile結果を監査対象とし、ユーザーには部分適用／要確認として通知する。
6. **Retry:** 再試行する場合は Reconcile 後の実OS状態を基準に、未達成の操作だけを再評価する。

`Success=true` は要求バッチ全体が完了したことのみを意味し、部分適用を暗黙に成功扱いしない。

---

# 3. Contracts Layer 定義 (`GameSecurityTool.Contracts`)

```csharp
namespace GameSecurityTool.Contracts.Interfaces;

using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Common;
using GameSecurityTool.Contracts.Dtos;

public interface IFirewallManager
{
    Task<bool> CreateBlockRuleAsync(FirewallRuleDto ruleDto, CancellationToken ct = default);
    Task<bool> RemoveBlockRuleAsync(string ruleTag, CancellationToken ct = default);
    Task<bool> ApplyBatchRulesAsync(IReadOnlyList<FirewallRuleDto> rulesToAdd, IReadOnlyList<string> ruleTagsToRemove, CancellationToken ct = default);
    Task<IReadOnlyList<FirewallRuleDto>> GetActiveRulesAsync(CancellationToken ct = default);
    Task<IReadOnlyList<FirewallRuleDto>> GetOsActualRulesAsync(CancellationToken ct = default);
    Task<IReadOnlyList<FirewallRuleVerificationResultDto>> VerifyRulesIntegrityAsync(IReadOnlyList<FirewallRuleDto> rules, CancellationToken ct = default);
    Task<bool> ConfigureWebRestrictedRulesAsync(string executablePath, IReadOnlyList<int> customAllowedPorts, CancellationToken ct = default);
    Task<bool> CleanupTemporaryStrictRulesAsync(CancellationToken ct = default);
}

public interface IAntiCheatDetector
{
    (bool IsDetected, string AntiCheatType, string DetectedFile, bool IsSignatureValid) DetectAntiCheat(string gameFolderPath);
}

public interface IElevatedWorkerClient
{
    Task<bool> WriteAuditAnchorAsync(string hash, CancellationToken ct = default);
    Task<string?> ReadAuditAnchorAsync(CancellationToken ct = default);
}
```

```csharp
namespace GameSecurityTool.Contracts.Dtos;

using System.Collections.Generic;
using GameSecurityTool.Contracts.Common;

public sealed record FirewallRuleDto(
    string OperationId,
    string RuleTag,
    string ExecutablePath,
    string FileHash,
    string? SignerCertificate,
    RuleDirection Direction,
    string Protocol,
    string Port,
    string RemoteAddress
);

public sealed record FirewallRuleVerificationResultDto(
    string RuleTag,
    bool ExistsInOs,
    bool ExistsInDb,
    bool HashMatched,
    bool ScopeMatched,
    string? DriftDetails
);

public sealed record ElevatedWorkerIpcRequest(
    byte[] SessionToken,
    IReadOnlyList<FirewallRuleDto>? RulesToAdd,
    IReadOnlyList<string>? RuleTagsToRemove,
    bool RequestOsRulesList = false,
    string? AnchorHashToWrite = null,
    bool RequestReadAnchor = false
);

public sealed record ElevatedWorkerIpcResponse(
    bool Success,
    int AddedCount,
    int RemovedCount,
    IReadOnlyList<FirewallRuleDto>? OsActualRules,
    string? ReadAnchorHash,
    string? ErrorMessage
);

### 3.1 ElevatedWorker SessionToken Secret-Transit Contract (approved security contract)

- `SessionToken` のContracts上の正規型は `byte[]` であり、immutable `string` は使用しない。
- Main GUI が生成・保持する32-byte (256-bit) session secret bufferは **caller-owned** とする。`ElevatedWorkerIpcRequest` へ設定しても所有権はWorkerへ移転しない。
- Callerは認証／IPC transactionの完了後、例外経路を含めて `finally` で `CryptographicOperations.ZeroMemory` 等により自身のbufferをzeroizeする。
- Workerが受信・deserializeしたbufferはMain側bufferとは別のWorker-owned copyであり、認証失敗、Pipe切断、Worker終了、watchdog強制終了等のライフサイクル境界でzeroizeする。
- `byte[]` は意図的に可変なsecret bufferである。DTOのrecord自体がそのbufferをdeep-immutableにすることは期待しない。
- `byte[]` の採用はin-memory Contracts representationを確定するものであり、raw bytes / Base64等を含むwire serialization形式を意味しない。wire frame / serializationはdocumented security and verification referenceの範囲に残す。
```

---

# 4. Infrastructure Layer 完全実装 (`GameSecurityTool.Infrastructure`)

## 4.1 Main プロセス側 IPC クライアント ＆ UAC バックオフ (`WindowsFirewallManager.cs` - H5 是正)

> **implementation contract:** 旧 `worker.token` 読取処理、server-generated token polling、token-file bootstrapは正式に廃止する。以下のフローがapproved security contractで確定した実装契約であり、Phase 0ではまだWorker実装を作成しない。

### Main-side authentication flow
1. Named Pipeへ接続する。接続失敗時の既存UAC backoff / 10秒watchdog / retry境界は維持する。
2. Main GUIが接続単位の32-byte (256-bit) SessionTokenを `RandomNumberGenerator` で生成し、caller-owned `byte[]` としてメモリ上で保持する。
3. WorkerのChallengeFrameを受信する。ChallengeFrameにはProtocolVersion、MessageType、32-byte challenge、Worker観測Client PID、fixed-width UTC FILETIMEのProcess StartTimeが含まれる。
4. Main GUIはWorker観測値を権威情報として再解釈せず、そのchallenge・PID・StartTime・protocol versionをdomain-separated HMAC-SHA-256入力へ拘束してproofを生成する。
5. Main GUIはBootstrapFrame（ProtocolVersion、MessageType、32-byte SessionToken、32-byte HMAC proof）をbinary length-delimited形式で送信する。
6. Workerのbootstrap認証成功後のみ通常の `ElevatedWorkerIpcRequest` を送信する。
7. request/response処理の終了、認証失敗、接続失敗、watchdog abort等の全経路で、Main側caller-owned session-secret bufferを `CryptographicOperations.ZeroMemory` 等によりzeroizeする。Worker側のdeserialize済みbufferは別のWorker-owned copyとして独立にzeroizeする。

### Main-side prohibitions
- `worker.token` その他の平文認証ファイルを作成・読取・削除しない。
- Registry、環境変数、CLI引数へ認証秘密を出さない。
- 認証失敗時に通常Firewall操作を継続しない。

### Existing lifecycle controls retained
- `Global\GST_ElevatedWorker_Mutex` による単一Worker。
- 60秒アイドル自己終了。
- UAC拒否時30秒バックオフ。
- IPC 10秒watchdogとゾンビWorker終了。

## 4.2 昇格ワーカー側 Named Pipe サーバー ＆ Layer 2 監査アンカー実装 (`ElevatedFirewallService.cs`)

> **implementation contract:** Worker側も認証秘密を永続化しない。旧 `TokenDirectory` / `TokenFilePath` / `WriteTokenFileWithStrictAcl` / `CleanupTokenFile` / server-generated token pollingモデルは正式に廃止する。

### Worker-side authentication flow
1. Worker起動後、DACL保護されたNamed Pipeを作成する。
2. Pipe接続確立後、`GetNamedPipeClientProcessId` で実接続クライアントPIDを取得する。
3. Peer Bindingとして、PIDの生存、Process StartTime、正規GST GUI実行イメージパスを検証する。GUIが送るPID/StartTime/pathは権威情報として扱わない。
4. Authenticode検証を実行体完全性の追加確認として行う。期待するGST Publisher/証明書識別子は別の正式ポリシーで定義されるまでapproved security contractから追加しない。
5. Peer Binding成功後、Workerがfresh 32-byte challengeを生成し、ChallengeFrameを送信する。
6. GUIからBootstrapFrameを受信し、SessionTokenと、ProtocolVersion + Challenge + Worker観測PID + Worker観測StartTimeからなるdomain-separated HMAC-SHA-256 proofをconstant-time検証する。
7. Bootstrap認証成功後だけ通常Firewall requestを受理する。
8. Workerはdeserialize後のsession-secret bufferを必要最小限の期間だけメモリ上に保持する。これはMain側caller-owned bufferとは別のWorker-owned copyである。
9. 認証失敗、Pipe切断、Worker終了、watchdogによる強制終了では秘密バッファをzeroizeしてから破棄する。

### Layer 2 audit anchor separation
Layer 2の `audit.anchor` は監査整合性資産であり、Worker認証用session secretとは別管理とする。監査アンカーの読み書きは既存の `IElevatedWorkerClient` / `ElevatedWorkerIpcRequest` 境界を使用し、approved security contractによる認証bootstrap変更とは混同しない。

### Finalized bootstrap contract
approved security contractでは認証bootstrapの保存・生成主体・ライフタイム境界を確定し、approved security contractで `ElevatedWorkerIpcRequest.SessionToken` のContracts上の正規型を `byte[]`、caller-owned ownership、Caller／Workerそれぞれのzeroization責務として確定した。approved security contractではPeer Binding、challenge/authentication role order、ChallengeFrame / BootstrapFrame のbinary wire contractを確定した。Phase 0ではこの契約に対応する暫定production typeを新設していない。

## 4.3 アンチチート Authenticode 署名検証 (`AntiCheatDetector.cs`)

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Native;

using System;
using System.IO;
using GameSecurityTool.Contracts.Interfaces;

public sealed class AntiCheatDetector : IAntiCheatDetector
{
    private static readonly string[] AntiCheatSignatures =
    [
        "EasyAntiCheat",
        "EasyAntiCheat_x64.dll",
        "BattlEye",
        "BEService.exe",
        "vgc.exe",
        "vgc.sys",
        "Arcturus.sys",
        "nGage",
        "GameGuard",
        "Denuvo"
    ];

    public (bool IsDetected, string AntiCheatType, string DetectedFile, bool IsSignatureValid) DetectAntiCheat(string gameFolderPath)
    {
        if (!Directory.Exists(gameFolderPath))
        {
            return (false, "None", string.Empty, false);
        }

        try
        {
            var enumOptions = new EnumerationOptions { RecurseSubdirectories = true, IgnoreInaccessible = true };
            var files = Directory.EnumerateFiles(gameFolderPath, "*.*", enumOptions);
            foreach (var file in files)
            {
                string fileName = Path.GetFileName(file);
                foreach (var signature in AntiCheatSignatures)
                {
                    if (fileName.Contains(signature, StringComparison.OrdinalIgnoreCase))
                    {
                        bool isSigned = Win32SecurityNativeMethods.VerifyAuthenticodeSignature(file);
                        if (isSigned)
                        {
                            return (true, signature, fileName, true);
                        }
                    }
                }
            }
        }
        catch
        {
            return (false, "None", string.Empty, false);
        }

        return (false, "None", string.Empty, false);
    }
}
```

---

# 5. 単体テスト仕様 (`GST.UnitTests.FW`)

```csharp
#nullable enable

namespace GameSecurityTool.UnitTests.FW;

using System;
using System.IO;
using GameSecurityTool.Infrastructure.Native;
using Xunit;

public class FirewallTests
{
    [Fact]
    public void AntiCheatDetector_WhenUnsignedDummyFile_RejectsCompatibilityException()
    {
        var detector = new AntiCheatDetector();
        string tempDir = Path.Combine(Path.GetTempPath(), "FakeEACUnsigned");
        Directory.CreateDirectory(tempDir);
        
        File.WriteAllText(Path.Combine(tempDir, "EasyAntiCheat_x64.dll"), "DUMMY_UNSIGNED_CONTENT");

        var result = detector.DetectAntiCheat(tempDir);
        
        Assert.False(result.IsSignatureValid);
        Assert.False(result.IsDetected);

        Directory.Delete(tempDir, true);
    }

    [Fact]
    public void WebRestrictedRules_GeneratesCorrectOutboundBlockExcludingHttpAndHttps()
    {
        string fakeExe = @"C:\Games\IndieGame\game.exe";
        var customPorts = new List<int> { 27015 };

        string ruleTagPrefix = $"GST:WEB-RESTRICTED:{Path.GetFileNameWithoutExtension(fakeExe).ToUpperInvariant()}";
        string expectedTag = $"{ruleTagPrefix}:OUT-BLOCK-NON-WEB";

        Assert.Equal("GST:WEB-RESTRICTED:GAME:OUT-BLOCK-NON-WEB", expectedTag);
    }
}
```

---

