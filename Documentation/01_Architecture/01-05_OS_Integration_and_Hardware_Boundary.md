# 01-05: OS Integration and Hardware Boundary

**Document ID:** GST-ARCH-BASELINE-002-PART5  
**Parent Document:** Architecture & Technology Baseline v2.0 (GST-ARCH-BASELINE-002)  
**Category:** OS Integration & Hardware Baseline  
**Status:** Approved Baseline Candidate  

---

# 16. Async / Concurrency Baseline

I/O 負荷および CPU 負荷の高い処理（Scan, Backup, Hash 計算, DB 操作, Export, Timeline ロード）は完全非同期を基本とする。

## 16.1 CancellationToken の必須化
すべての非同期メソッドは `CancellationToken` を引数に取り、ユーザーキャンセルやアプリケーション終了時に速やかにリソースを解放する。
```csharp
Task ExecuteAsync(CancellationToken cancellationToken);
```

## 16.2 同時実行制御とキューイング
SSD への過度な書き込み負荷、ファイルロック競合、レースコンディションを回避するため、Save Backup や暗号化処理などの重い I/O は不要に並列化せず、単一ワーカーによるキュー直列化（`Maximum Concurrent = 1`）を実行する。

---

# 17. FileSystemWatcher Pipeline

`FileSystemWatcher` からセキュリティ判定や DB 書き込みを直接呼び出すことを禁止する。

```text
[ FileSystemWatcher (OS Event) ]
               │
               ▼
[ BoundedChannel<FileEvent> (FullMode = Wait) ]
               │
               ▼
[ ScanBackgroundWorker ] ──> [ ScanProcessor ] ──> [ Domain Security Engine ]
```

* **Back Pressure 制御:** `BoundedChannel` を使用し、バーストイベントによるメモリ枯渇を防止する。
* **Overflow Handling:** キュー溢れが発生した場合は、イベントを破棄せず安全側のフォールバックとして「差分スキャン」または「フルスキャン」へ自動移行する。

---

# 18. Background Worker Design

GST は `Microsoft.Extensions.Hosting.BackgroundService` を利用して各バックグラウンド処理を独立ワーカーとして実行する。
* **ワーカー一覧:** `ScanWorker`, `BackupWorker`, `SqliteDatabaseWriter`, `AuditWorker`, `ExportWorker`
* **ワーカーの禁止事項:** UI への直接アクセス、Domain ロジックの肥大化、Repository をバイパスした DB 直接操作。

---

# 22. Windows Platform Integration Baseline

優先順位: `.NET Standard API -> .NET Windows API -> Windows Platform API -> Native API`

* Domain / Application / UI から Win32 API, Registry, COM を直接呼び出すことを禁止し、すべて Infrastructure Layer の Adapter に閉じ込める。
* **主要統合機能:** Windows Firewall (COM), Registry (WER制御), WMI / Process Polling, AMSI, DPAPI, Shortcut (.lnk) 構造解析。

---

# 24. Native API Baseline

* **P/Invoke 規約:** レガシーな `[DllImport]` を廃止し、C# 14 / .NET 10 の **`[LibraryImport]` (Source Generated P/Invoke)** を統一使用する。
* **Native Handle 管理:** 生ポインタ（`nint` / `IntPtr`）の保持を禁止し、必ず `SafeHandle`（`SafeFileHandle`, `SafeProcessHandle` 等）でラップしてリークを物理的に防止する。

---

# 31. Quarantine Baseline (隔離基盤)

* **原則:** Non-destructive, Reversible, Evidence Preservation, User Confirmed Restore。
* **Container 構造:** Header, Version, Algorithm, Integrity Metadata, Original Metadata (Path, Name, Hash, Size, ACL, Attributes, CreationTime, Owner), Encrypted Payload。
* **暗号化:** **Windows DPAPI (`CurrentUser`) + AES-GCM (256-bit)**。端末外への流出を防ぎ、同一端末・同一ユーザーコンテキスト内でのみ復元を許可する。
* **Streaming I/O 必須:** 巨大ファイル処理時の OOM（メモリ枯渇）を防ぐため、`File.ReadAllBytes()` を厳禁とし、64KB チャンクの `FileStream` ストリーミング処理を必須とする。
* **RestoreTransaction:**
  `RestoreStarted -> Target Validation -> ACL Restore -> Attribute Restore -> Payload Restore -> Hash Verify -> Restore Completed` のトランザクション制御を行い、途中失敗（`PartialFailed`）時は自動成功扱いを禁止し、Audit に記録する。

---

# 32. Web Link Protection Baseline

* **Scope:** GST Managed Launch Path（GST から起動される URL）のみを保護対象とする。PC 全体の DNS 遮断やブラウザ常時監視は行わない。
* **URL Pipeline:** `Parse -> Normalize -> Validate -> Strict Query Stripping -> Policy Evaluation -> Browser Selection -> Launch / Confirm / Block`
* **Query String 除去:** デフォルトで Query String を完全除去（明示許可ルールのみ例外）。
* **Origin 分類:** `Game-Origin`, `User-Origin`, `External-App-Origin`, `Unknown-Origin`。
* **Host Matching Security:** `Contains` などの単純部分一致を禁止し、ホスト境界一致、完全一致、サブドメイン一致、IDN（国際化ドメイン）正規化を必須とする。

---

# 33. AMSI Baseline

* **位置付け:** AMSI は補助的な `Detection Signal + Risk Modifier` であり、Engine そのものではない。
* **初期状態:** デフォルト無効（Disabled）。ユーザーの明示的同意により有効化。
* **Failure Handling:** AMSI 利用不能時も安全側を維持し、既存の検知パイプラインを継続する。

---

# 34. Anti-Cheat Baseline

* **Architecture:** `IAntiCheatDetector`（EasyAntiCheat, Vanguard, Generic）を Strategy パターンおよび `CompositeAntiCheatDetector` で集約する。
* Anti-Cheat detectionは互換性シグナルとして扱い、検出結果だけでFirewall権限を自動付与しない。
* Application 層から個別アンチチート具象クラスへの依存を排除し、ユーザー承認済みCompatibility ExceptionはGST-owned Firewall policyの既存境界を介してのみ適用する。
* Third-party Anti-Cheat enforcementへの介入、process/DLL injection、memory/driver manipulation、graphics hookはArchitecture上の禁止事項とする。

---

# 35. Browser / Clipboard Baseline

* **Browser Adapter:** ブラウザ差異は `IBrowserAdapter`（Chrome, Edge, Firefox）で吸収。
* **Clipboard Sanitizer:** オンデマンド方式のみ。常時監視・履歴保存・無断変更を禁止。`User Paste -> Parse -> Normalize -> Query Strip -> Preview Dialog -> User Confirm`。

---

# 36. LNK Detection Baseline

ショートカット（`.lnk`）の改ざん（TargetPath, Arguments, WorkingDir, Icon, Target Identity）を検知。検知即自動削除を禁止し、リスク理由を提示してユーザー判断を仰ぐ。

---

# 37. Save Data Backup Baseline

* **基本原則:** Non-destructive, Versioned Snapshot, Incremental Processing, User Confirmed Restore, Local/Privacy First。
* **SSD 負荷抑制:** 起動ごとの全量コピーや全量ハッシュを禁止。Manifest（Size, LastWriteTimeUtc, FileId）を比較し、変更ファイルのみを増分バックアップ。
* **PC移行・災害復旧（DR）連携 (`09_PC_Migration_and_Recovery_Model.md` 準拠):**
  * Backup Archive (ZIP) は端末固定の DPAPI 単体に依存せず、単体で復元可能なパスワード保護（または手動復元）をサポートする。
  * PC 移行時は `Migration Package` 経由で新環境へ認証コンテキストを安全に移送・再封緘する。
* **Streaming I/O & 同時実行:** `File.ReadAllBytes()` 厳禁。Buffered Stream Copy を用い、最大同時実行数は 1 とする。
* **非NTFSフォールバック:** FAT32/exFAT 等で `FileId` が取得不能な場合、`FileShare.None`（排他ロック）でオープンした `SafeFileHandle` を保持したままストリーミング処理を行う。
* **Retention Policy:** `MaxGenerations`, `MaxStorageSize`, `MinimumFreeSpace` に従い、最新の有効スナップショットを保護しつつ古い世代を自動パージする。

---

# 42. Firewall Policy Baseline

* **管理対象:** 登録された Game Profile に紐づくプロセスのみを制御（PC 全体ファイアウォール管理ツールではない）。
* **モード:** Standard (Outbound), Enhanced (In+Out), Maximum (In+Out+厳格Scope)。
* **Learning Mode:** 未知の通信を自動許可せず、監視 -> 検知 -> 推奨提示 -> ユーザー承認 -> ルール作成のフローを厳守する。
* **Drift Detection & Orphan Cleaner:** 実 Windows Firewall と DB レコードの乖離を検知・Audit 記録。孤立ルールの自動削除は行わず、ユーザー確認を経て削除する。

---

# 50. Evidence Bundle Export Baseline

* **構成:** `Application: EvidenceBundleExportUseCase -> Infrastructure: BundleWriter -> FileSystem`
* **内容:** `EvidenceBundle.zip` (Audit Records, Detection Result, Evidence, Snapshot, Metadata, Hash)
* **Memory Safety:** `File.ReadAllBytes()` 厳禁。`FileStream -> ZipArchive -> Entry Stream` によるストリーミング生成を必須とする。
* **Privacy:** Export 前に個人情報・パスの確認ダイアログを表示。自動外部送信・テレメトリ送信を完全禁止。

---

# 62. System Reversion and Telemetry Isolation Components

## 62.1 Full System Reversion Service
* **Contracts Port:** `IFullSystemReversionService`
  * `Task<ReversionPreviewDto> PreviewReversionAsync(CancellationToken ct);`
  * `Task<ReversionResultDto> ExecuteFullReversionAsync(ReversionOptions options, CancellationToken ct);`
* **Infrastructure Implementation:** `FullSystemReversionService` (`GST.Infrastructure.Persistence`)
  * `SystemChangeRecord` に記録された変更を `RollbackOrder` 降順で 100% 復元消去。
  * **中断リカバリライフサイクル:** 復元途中で電源断やクラッシュが発生し `PartialFailed` となった場合、次回起動時に検知してセーフモードダイアログを表示し、ロールバックの再試行・安全な復旧を提供する。

## 62.2 Crash Report Telemetry Blocker
* **Contracts Port:** `ICrashReportTelemetryBlocker`
  * `Task EnableCrashProtectionAsync(string gameExecutablePath, CancellationToken ct);`
  * `Task DisableCrashProtectionAsync(CancellationToken ct);`
  * `Task RecoverOrphanedProtectionFlagsAsync(CancellationToken ct);`
* **Infrastructure Implementation:** `CrashReportTelemetryBlocker` (`GST.Infrastructure.Native`)
  * `HKCU\Software\Microsoft\Windows\Windows Error Reporting\ExcludedApplications` のプロセススコープ制御。Standard User 権限で安全に動作し、OS 全体を破壊しない。
```

---