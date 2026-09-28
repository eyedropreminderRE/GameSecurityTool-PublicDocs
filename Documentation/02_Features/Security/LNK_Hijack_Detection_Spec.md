# GameSecurityTool .LNK 改ざん検知仕様書

**Document ID:** GST-FEAT-LNK-001  
**Version:** 3.1 (STA Thread Dispatch, Anti-Hang & Sample-Code Boundary Clarified)
**状態:** 採用確定（P1）  
**対象:** Windows 10 / Windows 11 / .NET 10 / C# 14 / WPF  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

## 1. 目的

ゲームのショートカット（`.lnk`）が改ざんされ、ゲーム本体の代わりに `cmd.exe`、`powershell.exe`、`wscript.exe` 等を経由して不正な処理を起動するショートカットハイジャック（Living off the Land: LotL 攻撃）を検知する。

本機能は「削除」や「自動修復」を目的とせず、**検知・説明・ユーザー主権の意思決定支援**を目的とする。

---

## 2. 適用範囲

対象は以下に限定する。

- 登録済み Game Profile に関連するゲームショートカット
- デスクトップ、スタートメニュー、ユーザーのスタートアップフォルダ等でユーザーが監査対象として指定した `.lnk`
- スキャン時に発見したゲーム関連 `.lnk`

※ PC全体の全ショートカットを無差別に変更・削除しない。

---

## 3. 取得情報

最低限以下を取得する。

- `ShortcutPath` (ショートカット自体の絶対パス)
- `TargetPath` (リンク先実行ファイルパス)
- `Arguments` (起動引数)
- `WorkingDirectory` (作業ディレクトリ)
- `IconLocation` (アイコンパスおよびインデックス)
- `HotKey` (ショートカットキー設定)
- `ShowCommand` (通常/最大化/最小化 起動状態)
- `LastWriteTimeUtc` (更新日時)
- `FileId` (取得可能な場合)
- `TargetFileId` (対象ファイルへ安全にアクセスできる場合)
- `TargetHash` (対象ファイルへ安全にアクセスできる場合)
- `SignatureStatus` (対象が実行可能ファイルの場合の Authenticode 署名状態)

---

## 4. 判定ロジック

### 4.1 高リスク条件 (Threat Conditions)

以下を単独または複合で評価する。

- `TargetPath` が `cmd.exe`, `powershell.exe`, `pwsh.exe`, `wscript.exe`, `cscript.exe`, `mshta.exe`, `rundll32.exe`, `regsvr32.exe`, `reg.exe` 等のシステムスクリプト実行ホストを指している
- `Arguments` に不審なコマンド連結（`&`, `|`, `&&`, `||`）、外部 URL（`http://`, `https://`）、難読化文字列（`%COMSPEC%`, `-enc` 等）が含まれる
- 既知の正常ベースライン状態から `TargetPath` が変更された
- 既知の正常ベースライン状態から `Arguments` が変更された
- `TargetPath` が Game Profile の許可された実行ファイル集合から外れている
- `TargetPath` が Reparse Point 経由で予期しない場所へ解決される

### 4.2 ThreatReasonCode

- `LnkTargetChanged` (深刻度: Danger)
- `LnkArgumentsChanged` (深刻度: Danger)
- `LnkSuspiciousCommand` (深刻度: Danger)
- `LnkUnexpectedTarget` (深刻度: Warning)
- `LnkExternalTargetResolved` (深刻度: Warning)
- `LnkUnsignedTarget` (深刻度: Warning)

---

## 5. 信頼ベースライン (Baseline Snapshot)

正常なショートカットの状態をスナップショットとして保存し、後続スキャンと比較する。

ベースラインには最低限以下を含める。

- `ShortcutPath`
- `TargetPath`
- `Arguments`
- `WorkingDirectory`
- `IconLocation`
- `BaselineHash` (SHA256)
- `CreatedAt`
- `ApplicationVersion`

※ ベースラインはユーザー承認なしに自動更新しない。

---

## 6. 安全性要件 ＆ COM 非ブロッキング仕様

### 6.1 Clean Architecture 隔離
`.lnk` 解析時に OS API や COM を Domain/Application から直接呼び出さず、Contracts の `ILinkInspector` Port を介して Infrastructure 層へ委譲する。

### 6.2 STA スレッドディスパッチの強制 (HIGH-08 是正)
バックグラウンドの並列スキャナ（`Task` / MTA スレッドプール）から直接 COM（`IShellLinkW` / `IPersistFile`）を呼び出すと、スレッドアパートメントの不一致により `InvalidCastException` が発生したり内部でハングアップする。
これを防ぐため、Infrastructure の `ILinkInspector` 具象実装においては、COM 呼び出しを必ず **STA が明示的に設定された専用のバックグラウンドスレッド** へディスパッチして実行しなければならない。

> **実装境界:** 以下のコードブロックは **仕様説明用の未完成サンプル／擬似コード** であり、現行製品の完成実装ではない。`IShellLinkW` / `IPersistFile` の実際の COM 相互運用定義、実ファイルからのターゲット取得、エラー処理、COM オブジェクト解放等を含む本番実装は別途実装・検証する必要がある。このブロックの固定パスやダミー値を本番コードとして使用してはならない。

```csharp
#nullable enable

namespace GameSecurityTool.Infrastructure.Native;

using System;
using System.Runtime.InteropServices;
using System.Threading;
using System.Threading.Tasks;
using GameSecurityTool.Contracts.Dtos;
using GameSecurityTool.Contracts.Interfaces;

public sealed class ShellLinkInspector : ILinkInspector
{
    public Task<ShortcutInspectionDto?> InspectShortcutAsync(string lnkFilePath, CancellationToken ct = default)
    {
        var tcs = new TaskCompletionSource<ShortcutInspectionDto?>(TaskCreationOptions.RunContinuationsAsynchronously);

        var thread = new Thread(() =>
        {
            try
            {
                ct.ThrowIfCancellationRequested();

                // COM オブジェクトの生成と IShellLinkW / IPersistFile へのキャスト
                Type? shellLinkType = Type.GetTypeFromProgID("WScript.Shell");
                // NOTE: ここから下は仕様説明用の擬似コード。
                // 本番実装では実際の IShellLinkW / IPersistFile 相互運用コードを使用する。
                // COM オブジェクトのライフサイクル管理・解放も本番実装で必須。
                
                // Dummy result for specification illustration only.
                var result = new ShortcutInspectionDto(lnkFilePath, "<resolved-target>", "<arguments>", "<working-directory>", "<description>");
                
                tcs.TrySetResult(result);
            }
            catch (OperationCanceledException)
            {
                tcs.TrySetCanceled(ct);
            }
            catch (Exception ex)
            {
                tcs.TrySetException(ex);
            }
            finally
            {
                // COM RCW 明示解放 (Marshal.ReleaseComObject) を必ず実行する
            }
        });

        // 【HIGH-08 是正】COM アクセスのために STA (Single-Threaded Apartment) を強制
        thread.SetApartmentState(ApartmentState.STA);
        thread.IsBackground = true;
        thread.Start();

        return tcs.Task;
    }
}
```

### 6.3 非ブロッキング Link Resolve (Anti-Hang)
破損ショートカットや未マウントドライブを指す `.lnk` の解析時にスレッドがハングすることを防ぐため、`IShellLinkW::Resolve` 呼び出し時は必ず以下のフラグを適用する。
```csharp
// SLR_NO_UI (0x1) | SLR_NOSEARCH (0x10) | SLR_NOTRACK (0x20) | SLR_NOLINKINFO (0x40) = 0x0071
const uint SLR_NO_FLAGS = 0x0071;
shellLink.Resolve(nint.Zero, SLR_NO_FLAGS);
```

### 6.4 RCW (Runtime Callable Wrapper) 明示解放
`IShellLinkW` および `IPersistFile` の COM オブジェクトは `finally` 節で `Marshal.ReleaseComObject` を確実に呼び出して即座に解放し、メモリリークを物理的に排除する。

### 6.5 Reparse Point 境界解決
ショートカットのパス文字列だけを信用せず、`PathValidationBarrier` 経由でリンク先の物理実パスを解像する。

### 6.6 非破壊原則
検知のみを行い、自動削除・自動修復は厳禁とする。

---

## 7. UI 設計

検知カードには以下を表示する。

- 「⚠️ ゲームショートカットのリンク先が変更されています」
- 変更前 TargetPath
- 変更後 TargetPath
- Arguments 差分（不審引数の強調表示）
- リスク理由（ThreatReasonCode）
- 「詳細を見る」
- 「この変更を信頼してベースライン更新」
- ※「変更を修復」機能は将来拡張とし、初期実装では提供しない（非破壊性優先）

---

## 8. 監査ログ (Audit Record)

`AuditEventType = ShortcutChangedDetected` として最小限の監査情報を記録する。

- `OperationId`
- `Timestamp`
- `Target`: 匿名化されたショートカット識別子
- `Result`
- `RiskLevel`
- `ThreatReasonCode`

※ URL やユーザー名等の不要な個人情報は `PrivacyLogSanitizer` でマスクする。

---

## 9. テスト要件

- 正常なゲームショートカットを誤検知しない
- TargetPath 変更を検知する
- Arguments 変更を検知する
- `cmd.exe /c` 等を高リスク化する
- Reparse Point 経由の異常 Target を検知する
- **【追加】STA スレッド境界テスト:** 複数スレッド（MTA）から同時に `InspectShortcutAsync` を呼び出した際に `InvalidCastException` やハングアップが発生せず、正常に完了すること。
- **壊れた `.lnk` や未マウントドライブ参照時にスレッドがハングしないこと (SLR フラグ検証)**
- **COM リソースが完全に解放されること (RCW リークゼロ)**
- Standard User 権限で安全に動作すること

---

## 10. ロードマップ

**P1 / MVP 重要保護機能** として実装する。

既存の `SecurityEngine`、`Snapshot Diff`、`ThreatReason`、`AuditLog`、`PathValidationBarrier` を再利用すること。

---

End of Document
```

---