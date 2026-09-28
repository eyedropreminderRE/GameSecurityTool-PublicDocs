# GameSecurityTool

# Security Threat Model and Attack Surface Analysis

## Threat Identification / Attack Surface / Security Risk Analysis

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-021 |
| Version | 3.0 (Full 29 Sections Restored, Complete Attack Surfaces & Threat Master Edition) |
| Status | Formal Baseline Specification (Highest Threat Analysis Authority) |
| Category | Security Analysis |
| Authority Level | Security Architecture Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）に存在する、

- 攻撃者モデル（Threat Actor Model）
- 保護対象資産分類（Critical / Sensitive Assets）
- 各レイヤーの攻撃対象領域（Attack Surface）
- 想定される具体的脅威シナリオ（T1〜T11）
- 防御アーキテクチャおよび対抗策

を定義する。

---

GST では、

```text
Feature Security  *  Data Security  *  Environment Security  *  Recovery Resilience
(機能の安全性、データの保護、環境の非破壊、復旧処理自体の堅牢性)
```

を一体として管理する。

---

# 1. Security Threat Philosophy

## 1.1 Core Principle
GST は「攻撃されないこと」「障害が起きないこと」を前提にしない。

前提条件：
```text
Attempted Access Exists        : 不正なアクセスやプロセスの改ざん試行が存在する
Data May Be Targeted           : バックアップやセーブデータが標的になる
Environment May Be Compromised : PC 環境が既に同一権限マルウェアに感染している
Failure & Crash Will Happen    : 復旧処理中であっても突然の電源断やクラッシュが発生する
```

そのため、**Prevent（予防）➔ Detect（検知）➔ Limit（局所化）➔ Recover（安全復旧）** の 4 段階でユーザー資産を保護する。

---

# 2. Threat Actor Model (攻撃者モデル)

## 2.1 Local User (同一 PC ユーザー)
同一 PC を共有する別ユーザーによるバックアップ盗難や設定改ざん。

## 2.2 Malware (同一権限マルウェア)
ゲームやフリーソフト経由で感染した Standard User 権限プロセス（ランサムウェア、ファイル改ざんボット、情報窃取ツール）。

## 2.3 Privileged Attacker (管理者権限マルウェア)
UAC 突破等により SYSTEM または Administrators 権限を取得した悪意あるプロセス。

## 2.4 External Attacker (外部ネットワーク攻撃者)
悪意ある Web サーバー、フィッシングサイト、C2 サーバー。

---

# 3. Asset Classification (保護対象資産分類)

## 3.1 Critical Assets (最重要資産)
- セーブデータ実体 (User Asset)
- バックアップコンテナ (Standard ZIP + Argon2id)
- 隔離ファイル実体 (.qrt コンテナ ＆ DPAPI 鍵)
- 監査ハッシュチェーン (Tamper-Evident Hash Chain)

## 3.2 Sensitive Assets (重要情報)
- ユーザープロファイル・パス情報 (実名等)
- Gemini API キー (DPAPI 暗号化保管)
- Universal Rule Scope / Firewall ルール設定
- 復元前一時退避データ (RescueSnapshot)

---

# 4. Attack Surface Model (攻撃対象領域)

```text
Application UI / Dialogs (ユーザー入力・XAML)
        ↓
Application Services & UseCases (マッピング・パイプライン)
        ↓
File System & Shortcuts (.LNK, Saves, Backups, Quarantine)
        ↓
Managed Launch Broker (URL Broker, Browser Picker)
        ↓
Privilege Boundary (Named Pipe IPC, Elevated Worker)
        ↓
Operating System APIs (Win32, Registry, WMI, Windows Firewall)
```

---

# 5. UI Attack Surface

## 5.1 Risk & Threat
- **脅威:** 不正なパス入力によるパストラバーサル、URL スキーム偽装（`javascript:` 等）、引数インジェクション。
- **対策:** 厳格なスキーム検証（`http`/`https` のみ）、`IPathResolver` 実パス解像、ブラウザ起動時オプション終端セパレータ（`--`）の付与。

---

# 6. File System Attack Surface

## 6.1 Target & Threat
- **脅威:** セーブデータの不正書き換え、シンボリックリンク／ジャンクションによる境界外脱出、Zip Slip 脆弱性、Zip Bomb 攻撃。
- **対策:** `PathValidationBarrier` による境界照合、`SharpCompress` ストリーミング展開容量監視、NTFS ACL 先行適用。

---

# 7. Backup Security Threat

## 7.1 Backup Theft
- **脅威:** 攻撃者がバックアップ ZIP を取得し、セーブデータや個人情報を閲覧・窃取する。
- **対策:** Standard ZIP + Argon2id (Iterations=3, Memory=64MB) + AES-256-GCM による強力なポータブル暗号化。

## 7.2 Backup Tampering
- **脅威:** バックアップコンテナの内容が外部から改ざんされる。
- **対策:** 決定論的 `ManifestHash` 検証および 64KB チャンクごとの AEAD 認証タグ（16B）照合。

---

# 8. Migration Package Threat (完全復元)

## 8.1 Package Theft
- **脅威:** `.gstmgr` パッケージが盗難され、他環境で不正インポートされる。
- **対策:** Argon2id マスターキーによるパッケージ全体暗号化とユーザーパスフレーズ検証。

## 8.2 Password Attack
- **脅威:** 弱いパスワードに対する総当たり（Brute Force）攻撃。
- **対策:** Argon2id（メモリ 64MB / 反復 3 回）による高負荷 KDF 設計。

---

# 9. Credential Protection Threat

## 9.1 Credential Exposure
- **脅威:** 暗号鍵やパスワードが平文で保存またはメモリ上に残存し、メモリダンプ攻撃に晒される。
- **対策:** 平文保存の完全禁止、Windows DPAPI 利用、パスワード `byte[]` の `finally` での `ZeroMemory` 即時消去。

---

# 10. Database Threat

## 10.1 Database Modification
- **脅威:** SQLite ファイル（`gamesecurity.db`）が直接書き換えられ、ルールや監査ログが改ざんされる。
- **対策:** `SqliteDatabaseWriter` 単一キュー直列化、短命 `DbContext`、SHA-256 連鎖 Hash Chain ＆ 特権多層アンカーによる起動時自動検知。

---

# 11. Firewall Management Threat

## 11.1 Unauthorized Rule Change
- **脅威:** 不正プロセスが勝手に Firewall ルールを作成・変更して通信を許可する。
- **対策:** DACL 保護 Named Pipe IPC、インバンドトークンの定数時間比較、`CreatedBy = GST` 所有権追跡。

## 11.2 Rule Cleanup Risk
- **脅威:** 孤立ルール削除処理により他社ルールや正常ルールが誤爆削除される。
- **対策:** ローカル DB レコードとの突合による Foreign Rule 保護、削除前の手動承認必須化。

---

# 12. Privilege Escalation Threat

## 12.1 Risk
- **脅威:** 特権ワーカーを悪用した管理者権限の不正奪取。
- **対策:** 最小権限原則（Main GUI は Standard User）、Named Pipe DACL（CurrentUser/SYSTEM/Admin 限定）、60 秒アイドル自己終了。

---

# 13. Process Security (完全復元)

## 13.1 Process Injection Risk
- **脅威:** 実行中の GST プロセスへの DLL インジェクションやメモリ改変。
- **対策:** `SetDefaultDllDirectories(LOAD_LIBRARY_SEARCH_SYSTEM32)` による DLL ハイジャック遮断、コード完全性検証。

---

# 14. Dependency Threat (完全復元)

## 14.1 Third Party Risk
- **脅威:** サードパーティ製ライブラリの既知の脆弱性やサプライチェーン攻撃。
- **対策:** 承認済みパッケージ（`SharpCompress`, `Konscious.Security` 等）のインベントリ固定、定期的な脆弱性スキャン。

---

# 15. Update Attack (完全復元)

## 15.1 Fake Update
- **脅威:** 偽のアップデートパッケージによる不正バイナリの導入。
- **対策:** パッケージハッシュ検証、デジタル署名検証、更新失敗時の安全な自動ロールバック。

---

# 16. Logging Threat

## 16.1 Sensitive Information Leakage
- **脅威:** ログファイルやスタックトレースから本名パス、生 URL クエリ、パスワードが漏洩する。
- **対策:** `PrivacyLogSanitizer`（`[GeneratedRegex]`）による自動マスキング、機密情報のログ出力完全禁止。

---

# 17. Privacy Threat (完全復元)

## 17.1 Excessive Collection
- **脅威:** セキュリティ保護を名目としたユーザー行動やゲームプレイデータの過剰収集。
- **対策:** データ最小化原則、テレメトリ完全非搭載、外部クラウド自動送信ゼロ（Local First）。

---

# 18. Availability Threat (完全復元)

## 18.1 Data Lock
- **脅威:** 長時間の排他ロックやデッドロックによるゲーム起動妨害、復元処理の中断によるデータ利用不能。
- **対策:** Game-First I/O Guard によるスキャン即時キャンセル、`RescueSnapshot` によるアトミック自己修復。

---

# 19. Defense Layer Model (多層防御マトリクス)

```text
┌────────────────────────────────────────────────────────┐
│ Layer 1: 入力検証・バリア (PathResolver, Strict Query) │
├────────────────────────────────────────────────────────┤
│ Layer 2: ドメイン判定・確信度モデル (SecurityEngine)   │
├────────────────────────────────────────────────────────┤
│ Layer 3: アトミック変更 ＆ 資産保護 (Rescue, Standard) │
├────────────────────────────────────────────────────────┤
│ Layer 4: 特権分離 ＆ OS 遮断 (Named Pipe, Firewall)    │
├────────────────────────────────────────────────────────┤
│ Layer 5: 多層改ざん検知 ＆ 監査 (Hash Chain, Layer2)   │
└────────────────────────────────────────────────────────┘
```

---

# 20. Security Review Checklist

変更およびリリース時に確認：

☐ 平文のパスワード・暗号鍵・API キーが永続化およびログ出力されていないこと  
☐ **セーブ復元・PC 移行再封緘が一時ファイル経由でアトミックに実行されていること**  
☐ **特権アンカー未同期時および WMI 監視停止時にサイレント Fail-Open せず可視化されること**  
☐ 未署名 MOD の DLL Side-Loading 診断が外部通信を行わずローカルで機能すること  
☐ すべての特権操作が DACL 保護 Named Pipe および定数時間トークン検証を経由していること  

---

# 21. Security Evolution Rule (完全復元)

新機能追加時の必須確認事項：
- 新たな攻撃対象領域（Attack Surface）を生み出していないか
- 新たな特権や OS 権限を要求していないか
- 新たな外部通信やデータ収集を追加していないか
- 既存の信頼境界（Clean 5-Layer）を破壊していないか

---

# 22. Shortcut (.LNK) Hijack Threat

- **脅威:** ゲームショートカットの `TargetPath` や `Arguments` が書き換えられ、`cmd.exe` や `powershell.exe` を経由して不正スクリプトを起動させる LotL（Living off the Land）攻撃。
- **対策:** `ILinkInspector` による STA スレッド非ブロッキング構造解析、ベースラインスナップショット差分比較、検知理由の明示（自動削除は禁止）。

---

# 23. Web Launch Bypass Threat

- **脅威:** Managed Launch Path を迂回してゲームプロセスが直接ブラウザを起動し、追跡トークンや個人情報を外部送信する攻撃。
- **対策:** WMI 受動監視（`WmiProcessLifecycleWatcher`）、二段階権限要求（`QUERY_LIMITED`）、`eventTimeUtc ± 3秒` の対称時間ウィンドウ検証による PID Reuse 攻撃遮断。

---

# 24. Audit Layer 2 Anchor Tampering Threat

- **脅威:** 同一ユーザー権限マルウェアによるローカル DB ＋ DPAPI (Layer 1) ハッシュの一括再計算改ざん。
- **対策:** 特権管理者アンカー (`%ProgramData%\GameSecurityTool\audit.anchor`) 二重検証、未同期または不整合時はサイレント成功とせず、厳格に `false` を返却する Fail-Closed 規約（`VerifyAuditChainIntegrityAsync`）。

---

# 25. Process Monitoring Failure & Silent Fail-Open Threat (C-6 連携)

- **脅威:** WMI 障害時に保護が無効化されているのに、ユーザーが「保護中」と誤認してマルウェアを実行してしまう。
- **対策:** `WatcherFaulted` イベントによるダッシュボードへの「⚠️ プロセス監視エンジン停止」強制表示、30 秒周期自律再初期化リトライ。

---

# 26. Unsigned Mod DLL Side-Loading Threat

- **脅威:** DirectX プロキシ名（`dxgi.dll`, `version.dll` 等）に偽装した悪意あるバイナリの配置。
- **対策:** PE インポート完全解析（`ws2_32.dll`, `CreateProcess` 検知）、NTFS `Zone.Identifier` 出所特定、汎用 SHA256SUMS 突合。

---

# 27. Restoring Mutation Corruption Threat (C-3, C-4 連携)

- **脅威:** セーブ復旧（`RollbackAsync`）や PC 移行再封緘（`ImportAndResealQuarantineAsync`）実行中の電源断でデータが中途破損する。
- **対策:** 一時ファイル（`.rollback.tmp`, `.reseal.tmp`）出力 ➔ ハッシュ検証 ➔ `File.Move(overwrite: true)` によるアトミック置換。

---

# 28. Script LotL & Persistence Tampering Threat (スクリプト LotL ＆ 持続化改ざん脅威)

- **脅威:** ゲーム配布物や MOD パックに含まれる `.ini`, `.bat`, `.cmd`, `.ps1` などの設定・スクリプトファイル内に、間接コマンド実行コード（`powershell -enc`, `curl`, `certutil`, `bitsadmin` 等）を埋め込み、Living-off-the-Land (LotL) 攻撃によってマルウェアをダウンロード・常駐させる。また、ゲーム起動時にレジストリ `Run` / `RunOnce` キーや `%SystemRoot%\System32\drivers\etc\hosts` を改ざんして C2 誘導や永続化を図る。
- **対策:** `ISecurityEngine` によるスクリプト LotL 静的パターン走査、ゲーム終了時のポスト監査（`IRegistryPersistenceTracker.AuditHostsAndRunKeysAsync`）による自動検知、およびワンクリック復元機能の提供。

---

# 29. Launcher Cross-Boundary Tampering Threat (主要ランチャー越境改ざん脅威)

- **脅威:** 起動したゲームプロセスが、親プロセスまたは別ディレクトリにある公式ランチャー（Steam, Epic Games, EA 等）の本体フォルダ（`Steam.exe`, コア DLL, 設定ファイル）へアクセスし、DLL ハイジャックや不正上書きを行って横展開する。
- **対策:** `ILauncherSecurityAdapter` によるランチャー本体ディレクトリの隔離監視、コア実行ファイルのデジタル署名および SHA-256 完全性ベースライン監視、越境書き込み検知時の起動一時停止と警告。

---

# 30. Patch Update Anomaly Threat (パッチ適用時リスク急変脅威)

- **脅威:** Steam 等の自動アップデートや MOD パッチ適用によって、従来安全と判定されていたゲームに突然「未署名通信バイナリ」「既知の不審な DLL」「ドロッパー」が追加され、ユーザーが無自覚のまま脅威を実行してしまう。
- **対策:** アップデート前後のファイル差分および署名状態の比較検証（Update Anomaly 検知）。リスク急変が確認された場合、起動を一時停止してユーザーへ警告ダイアログを表示し、明示的な承認を要求する。

---

# 31. Ransomware Entropy Sudden Change Threat (ランサムウェア・エントロピー急変脅威)

- **脅威:** ランサムウェアによってゲームセーブデータが一括暗号化され、GST の自動バックアップが暗号化後の破損ファイルを「最新セーブ」として上書き保存し、健全な過去バックアップが世代管理によってパージされてしまう。
- **対策:** セーブファイルのシャノン・エントロピー急変（急激な高エントロピー化）を検知した場合、Snapshot Freeze を発動して自動パージを緊急凍結し、健全なバックアップ世代を永久保全する。

---

# 32. 【確定】Signed Infostealer & Engine Core Tampering Threat (署名付きインフォスティーラー ＆ コアバイナリ偽装脅威)

- **脅威:** 盗まれた正規コード署名証明書や不正取得された署名を悪用し、正規バイナリに見せかけてブラウザ拡張機能（MetaMask, TronLink 等の暗号資産ウォレット固有 ID）やトークンを探索・窃取する。また、`UnityPlayer.dll` 等の主要ゲームエンジン公式バイナリと同名ファイルを配置し、本来署名があるべきコアバイナリを未署名・改ざんバイナリへすり替える。
- **対策:** Behavior & Structure Over Signature（署名の有無だけで安全性を二値判定せず、ウォレット拡張 ID や生ソケット通信 API、外部コマンド実行コードを徹底走査）、WinVerifyTrust `WTD_REVOKE_WHOLECHAIN` による CRL/OCSP リアルタイム失効検証、およびエンジンコアバイナリの正規署名喪失検知。

---

# 33. 【確定】Installer Co-traveler Drop Threat (インストーラー共連れドロップ脅威)

- **脅威:** ゲームのインストーラー（setup.exe）や展開スクリプトが、ゲームインストール先フォルダの外（`%APPDATA%`, `%LOCALAPPDATA%`, `%TEMP%`, スタートアップ等）にバックドアやマイナーをこっそり配置（共連れドロップ）する。
- **対策:** 新規ゲーム配置検知時、直近 10 分間にゲームフォルダ外の重要システム領域で新規作成または更新された実行可能ファイル（`.exe`, `.dll`, `.bat`, `.ps1`）を高速監査（`RecentArtifactAuditor`）。不審ファイルを警告バナーで提示し、ワンクリックで暗号化隔離庫へ退避可能とする。

---

# 34. 【確定】In-Place High-Value Target (HVT) Tampering Threat (既存急所ファイル改ざん脅威)

- **脅威:** ゲーム実行中に `%SystemRoot%\System32\drivers\etc\hosts` やユーザーの PowerShell プロファイル（`Microsoft.PowerShell_profile.ps1`）が書き換えられ、DNS 偽装による不正認証サーバーへの誘導や、シェル起動時の悪意あるスクリプト自動実行が行われる。
- **対策:** `HighValueTargetIntegrityVerifier` によるゲーム起動前の SHA-256 ハッシュスナップショット採取、ゲーム終了時および定期検査での改ざん検知、退避ジャーナル（`hosts.baseline`）からのクリーンな状態へのアトミック復元。

---

# 35. 【確定】USB External Storage Sudden Disconnection Threat (USB 外付けストレージ物理抜け脅威)

- **脅威:** 外付け HDD や USB ドライブへのセーブバックアップ書き込みや暗号化隔離処理中に、ケーブル抜けや端子接触不良（物理抜け）が発生し、ファイルシステムの破損や不完全なゴミファイルが残留する。
- **対策:** `StorageResilienceProvider` による Win32 `WM_DEVICECHANGE` (`DBT_DEVICEREMOVECOMPLETE`) の捕捉、実行中ジョブの安全な一時中断（Suspend）、不完全な一時ファイル（`.tmp`, `.creating`）の次回起動時自動回収、およびドライブ再接続時の安全な再試行。

---

# 36. 【確定】Destructive Wiper / Mass-Deletion Sabotage Threat (破壊型ワイパー・大量消去脅威)

- **脅威:** ゲームや不正MOD等を起点として、`cmd.exe /c del`、`powershell Remove-Item`、`pwsh`、`vssadmin delete shadows`、`wbadmin`、`cipher /w` 等のLiving-off-the-Land手段を用い、ゲームフォルダ外のユーザー資産を短時間に大量削除・変更し、復元可能性を低下させる。
- **防衛境界:** `Anti-Wiper Defense & Auto-Circuit Breaker` は既定OFFのオプトイン機能とし、監視対象は明示されたユーザー資産領域に限定する。ゲーム自身のインストールディレクトリ、正規セーブ領域、ゲーム自身が生成する一時領域はバースト検知から除外する。
- **未然遮断:** `Win32_ProcessStartTrace` により監視ゲームを親とする子プロセス生成を受動検知し、既知のシェル/管理ホストと破壊的コマンドラインの組合せを検査する。条件に一致した子プロセスに対してのみ、`PROCESS_TERMINATE` を使用した終了を行う。WMIはイベント駆動であり、イベント受信より前に発生した処理を遡及して取り消すものではない。
- **バースト遮断:** `FileSystemWatcher`（OSの変更通知機構を利用）で `Deleted` / `Changed` / `Renamed` を観測し、設定された100msスライディングウィンドウのしきい値を超えた場合にCircuit Breakerへ遷移する。変更イベント自体には発生元PIDが含まれないため、バースト単体をゲームプロセス起点の証明として扱わない。
- **Canary:** `!0_gst_canary.dat` を保護領域に配置し、その変更・削除イベントを通常のバースト判定とは独立した早期トリガーとして扱う。Canary自体はGSTのスキャナー・バックアップ対象から除外する。
- **プロセス制御:** ブレーカー発動時は既定で `NtSuspendProcess` により監視対象ゲームを一時停止し、ユーザーが確認した後に再開または `TerminateProcess` による終了を選択できる。これらの操作は仮想メモリアクセスやコード注入を伴わない限定的プロセス制御として扱う。
- **VSS保護:** VSS削除を試みる子プロセスの未然遮断を補助防衛として記録する。OSの復元機能が常に利用可能であることや、先行する全ファイル変更がゼロになることは保証しない。

# 37. Wiper 防衛の攻撃面と残存リスク

- WMIプロセス開始通知、ファイル変更通知、プロセス制御はいずれもOSのユーザーモード機構に依存するため、通知遅延・キュー飽和・権限差・対象プロセスの終了競合が発生し得る。
- `FileSystemWatcher` によるイベント観測は、変更そのものを防止する仕組みではなく、観測後にブレーカーを発動する防御である。したがって「0バイトの事前防止」「全件捕捉」「1件も先行しない」といった結果は仕様上の保証対象としない。
- `Win32_ProcessStartTrace` で取得できるイベント情報とコマンドライン情報の取得経路は分離して管理し、コマンドライン解析のためにゲームプロセスのメモリを読まない。
- `NtSuspendProcess` / `TerminateProcess` の使用権限不足や競合時は、安全側の監査記録を残して通常クラッシュ復旧・ユーザー通知フローへ委譲する。

---

# 38. Relationship With Other Models (他モデルとの関係性 - 完全復元)

## 36.1 Privacy
プライバシー保護および 11 大脅威モデル：
```text
06_Privacy_and_Threat_Model.md
```

## 36.2 Security Implementation
セキュアコーディングおよびアトミック変更実装規則：
```text
14_Security_Implementation_Guideline.md
```

## 36.3 Testing Strategy
セキュリティ境界・テストベクター自動検証：
```text
15_Test_Strategy_and_Validation_Model.md
```

## 36.4 Risk Management
将来機能拡張およびリスク評価基準：
```text
19_Risk_Management_and_Future_Expansion_Model.md
```

## 36.5 Security Boundary & Audit
操作境界・監査証跡モデル：
```text
02_Security_Boundary_and_Protection_Model.md
04_Audit_and_Evidence_Model.md
```

---

# 37. Final Security Statement

GST の安全性とは、攻撃や障害を「絶対に起こらない」と仮定することではない。

---

どんな攻撃や電源断、物理障害が発生した場合でも、

```text
Never Destroy User Assets
Never Fail Silently
Always Enable Safe Recovery
```

を維持することである。

---
