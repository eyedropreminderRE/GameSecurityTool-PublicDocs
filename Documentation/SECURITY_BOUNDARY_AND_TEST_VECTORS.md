# GameSecurityTool Security Boundary & Test Vectors Specification

**Document ID:** GST-SPEC-VECTORS-001  
**Version:** 5.5 (11-Boundary Model + WIPER Recovery Remediation)
**Target Projects:** `GameSecurityTool.SecurityTests`, `GameSecurityTool.ArchitectureTests`, `GameSecurityTool.Infrastructure`  

> **Lifecycle note:** This document describes approved design/specification scope. Its presence does not by itself indicate that the described behavior is implemented, Windows-verified, or released.

---

# 1. 11大セキュリティ境界モデル (Security Boundaries)

GST は、単一の防衛線に依存せず、以下の 11 個の独立したセキュリティ境界（Security Boundaries）を維持する。Boundary 1–8 は Core Security Boundaries、Boundary 9–11 はその拡張として扱う。機能追加によってこれらの境界をバイパスすることは許されない。

```text
┌────────────────────────────────────────────────────────────────────────┐
│ Boundary 1: Privilege Boundary (Standard User vs Out-of-Process UAC)   │
├────────────────────────────────────────────────────────────────────────┤
│ Boundary 2: Web & Browser Boundary (Managed Path vs WMI Emergency)     │
├────────────────────────────────────────────────────────────────────────┤
│ Boundary 3: Quarantine Isolation Boundary (Chunked AEAD + ZeroMemory)  │
├────────────────────────────────────────────────────────────────────────┤
│ Boundary 4: Save Data Asset Boundary (Standard ZIP + Argon2id + Rescue)│
├────────────────────────────────────────────────────────────────────────┤
│ Boundary 5: Persistence Concurrency Boundary (IDbWriteQueue 直列化)    │
├────────────────────────────────────────────────────────────────────────┤
│ Boundary 6: OS Configuration Safety Boundary (WER Restore & ClearPools)│
├────────────────────────────────────────────────────────────────────────┤
│ Boundary 7: Archive Safety Boundary (Zip Slip & Zip Bomb Protection)   │
├────────────────────────────────────────────────────────────────────────┤
│ Boundary 8: Strict Safe Launch Boundary (In/Out Block & WER Isolation) │
├────────────────────────────────────────────────────────────────────────┤
│ Boundary 9: COM / LNK Analysis Boundary (STA & LotL Detection)         │
├────────────────────────────────────────────────────────────────────────┤
│ Boundary 10: Anti-Wiper Emergency Containment Boundary                │
├────────────────────────────────────────────────────────────────────────┤
│ Boundary 11: External AI Outbound Privacy & Consent Boundary           │
└────────────────────────────────────────────────────────────────────────┘
```

---

# 2. 攻撃シナリオ ＆ テストベクター詳細仕様

各テストベクターは `GameSecurityTool.SecurityTests` プロジェクトにおいて、決定論的な自動テストとして実装されなければならない。

## 2.1 【Boundary 1】特権境界 ＆ IPC 通信ベクター

### approved security decision bootstrap verification contract

Boundary 1のIPCベクターは、DACLに加えてapproved security decisionのPeer Bindingとbootstrap frame契約を検証対象とする。Worker観測PID / Process StartTime / canonical GUI pathを権威情報とし、GUIが送るidentity metadataを信用しない。Peer Binding成功後のみfresh 32-byte challengeを発行し、binary BootstrapFrameの32-byte SessionToken + 32-byte HMAC-SHA-256 proofをconstant-time検証する。認証成功前の通常Firewall request処理は禁止する。

### 脅威シナリオ:
非特権プロセスが Named Pipe に不正接続し、権限昇格を悪用して任意の Firewall ルールを作成・削除する。マルウェアが DPAPI アンカーと SQLite DB を一括改ざんして監査証跡を消去する。特権ワーカーが短時間に連発起動されて UAC 疲れを招く。

### テストベクター:
| Vector ID | テスト入力 / 攻撃手法 | 期待される安全動作 (Expected Result) |
| :--- | :--- | :--- |
| **`TV-SEC-IPC-01`** | 別ユーザーセッションからの Named Pipe 接続試行 | DACL（`PipeSecurity`）によりアクセス拒否（Access Denied）。 |
| **`TV-SEC-IPC-02`** | Peer Binding済みクライアントから不一致SessionToken / challenge proofを送信 | BootstrapFrameのHMAC-SHA-256 proofをconstant-time検証し、失敗時は通常リクエストを処理せず即時切断。 |
| **`TV-SEC-IPC-06`** | `worker.token` が存在する、またはToken bootstrapを `%TEMP%` / Registry / 環境変数 / CLI から取得しようとする | 認証ブートストラップは成立せず、秘密情報をディスク等へ保存しない。 |
| **`TV-SEC-IPC-07`** | 直前の接続のChallenge/Proofを別のNamed Pipe接続で再送 | 接続固有challengeに対する検証に失敗し、再利用を拒否。 |
| **`TV-SEC-IPC-08`** | Named Pipe client PIDは取得できるが、Process StartTimeが接続時検証値と一致しないPID再利用ケース | Peer Bindingを失敗としてchallengeを発行せず、接続を拒否。PID単独で認証を継続しない。 |
| **`TV-SEC-IPC-09`** | Pipe client PIDの実行イメージパスがcanonical GST GUI pathと異なる | Peer Bindingを失敗としてchallengeを発行せず、接続を拒否。GUIが送信したpathは権威情報として扱わない。 |
| **`TV-SEC-IPC-10`** | GST GUIと異なる実行体からの接続、またはAuthenticode完全性検証に失敗するGUI実行体 | Peer Binding / integrity checkに失敗し、challengeを発行せず通常IPCを受理しない。approved security decisionで未定義のPublisher/証明書識別子を新規前提にしない。 |
| **`TV-SEC-IPC-11`** | ChallengeFrame / BootstrapFrameをJSONやtext encodingで送信、またはframe契約に違反する形式を送信 | binary length-delimited bootstrap protocolとして拒否し、通常IPCへ遷移しない。 |
| **`TV-SEC-IPC-03`** | Main プロセス内からの `INetFwPolicy2` COM 直接実行 | Standard User 権限のため例外捕捉され、Main 側での特権実行を遮断。 |
| **`TV-SEC-IPC-04`** | 特権ワーカー起動後 60 秒間無通信待機 | アイドルタイムアウトが発火し、ワーカープロセスが安全に自動終了。 |
| **`TV-SEC-IPC-05`** | 特権ワーカー起動中に別スレッドから `LaunchElevatedWorker` を重複呼出 | グローバル Mutex（`Global\GST_ElevatedWorker_Mutex`）により新規起動が安全にスキップされ、パイプ再接続が行われること (M1 保証)。 |
| **`TV-SEC-approved audit reference`** | SQLite DB および Layer 1 (DPAPI) のハッシュを改ざん | **Layer 2 (特権 IPC 経由の `%ProgramData%` アンカー)** との不一致を検知し、改ざんアラートを正しく発報すること (CRIT-04 保証)。 |

---

## 2.2 【Boundary 2】Web Link & WMI 緊急検知ベクター

### 脅威シナリオ:
ゲーム内リンクによる機密トークン外部送信、Punycode ホモグラフ偽装、コマンドライン引数インジェクション、および PID 再利用（PID Reuse）や WQL インジェクションによる監視のサイレント無効化。

### テストベクター:
| Vector ID | テスト入力 / 攻撃手法 | 期待される安全動作 (Expected Result) |
| :--- | :--- | :--- |
| **`TV-SEC-WEB-01`** | `https://example.com/p?token=secret123&utm=game` | Strict Query Stripping により `https://example.com/p` のみ渡される。 |
| **`TV-SEC-WEB-02`** | 国際化ドメイン偽装 (`https://xn--pple-43d.com/`) | IDN 正規化され、Punycode 警告フラグ（`IsPunycodeSuspicious = true`）が立つ。 |
| **`TV-SEC-WEB-03`** | コマンドライン先頭引数偽装 (`-private --remote-debugging-port`) | 先頭ハイフンを検知し、ブラウザ起動を即時ブロック。 |
| **`TV-SEC-WEB-04`** | Unicode ホストで Block ルール登録後、同一ドメインの URL を評価 | 登録時と評価時で完全に対称な IDN 正規化が行われ、確実に Block されること (H2 保証)。 |
| **`TV-SEC-PID-01`** | 同一 PID でイベント時刻より 5秒後に起動した別プロセス | `ValidateAndOpenProcess` が時間対称ウィンドウ外として拒否、誤終了を防止。 |
| **`TV-SEC-PID-02`** | PID 一致だが実行バイナリパスが異なるプロセス | `QueryFullProcessImageNameW` 検証で不一致を検知し、即時ハンドル解放。 |
| **`TV-SEC-WMI-01`** | 監視対象外のプロセス（`cmd.exe` 等）を 10,000 回ループ起動 | 動的 WQL フィルタにより `WmiPrvSE.exe` 側にイベントが流れない（CPU 0% 維持）こと (HIGH-06 保証)。 |
| **`TV-SEC-WMI-02`** | 実行ファイル名にシングルクォートを含むゲーム（`Player's Game.exe`）の監視 | WQL クエリが構文エラーにならず、エスケープされて安全に監視が継続すること (H1 保証)。 |

---

## 2.3 【Boundary 3】暗号化隔離 ＆ OOM 防御ベクター

### 脅威シナリオ:
大容量不審ファイルによるメモリ枯渇（OOM クラッシュ）、DB コミット前の元ファイル消失、暗号鍵のメモリ残存、二段階コミット失敗による巨大孤児ファイルの放置、ReadOnly 属性による復元失敗。

### テストベクター:
| Vector ID | テスト入力 / 攻撃手法 | 期待される安全動作 (Expected Result) |
| :--- | :--- | :--- |
| **`TV-SEC-QRT-01`** | 5GB 超のダミーファイル暗号化隔離 | 64KB チャンク分割ストリーミングにより、プロセス使用メモリが 120MB 以下を維持。 |
| **`TV-SEC-QRT-02`** | コンテナ書き込み中に強制例外注入 (Fault Injection) | DB 未コミットのため元ファイルが削除されず、破損一時コンテナのみ安全消去。 |
| **`TV-SEC-QRT-03`** | 隔離完了後のマネージドヒープ・スタック走査 | `ZeroMemory` によりメモリ上に平文 AES-256 鍵が残存しないこと。 |
| **`TV-SEC-QRT-04`** | `FileAttributes.ReadOnly` を持つ隔離ファイルの復元 | 属性復元を最後に行うシーケンスにより、ACL/タイムスタンプ設定が拒否されず成功。 |
| **`TV-SEC-QRT-05`** | 隔離 DB コミット処理（`EnqueueWriteAsync`）で例外発生 | **キャッチブロック内で確定コンテナ `.qrt` が物理的に消去され、孤児ファイル化しないこと** (HIGH-07 保証)。 |

---

## 2.4 【Boundary 4】セーブバックアップ ＆ ポータブル復旧ベクター

### 脅威シナリオ:
TOCTOU / ジャンクションによる保護外ディレクトリへの不正展開、復元失敗時や退避中クラッシュ時の既存セーブデータ消失、メモリダンプからのパスワード漏洩、GC による新規 ZIP の誤爆削除、参照コンテナの誤削除、およびパージ処理による失敗証跡の消失。

### テストベクター:
| Vector ID | テスト入力 / 攻撃手法 | 期待される安全動作 (Expected Result) |
| :--- | :--- | :--- |
| **`TV-SEC-BAK-01`** | 別 PC 環境への ZIP バックアップ移送と復元 | ユーザー指定パスワード（Argon2id + AES-256）で自力復元可能（No Lock-in）。 |
| **`TV-SEC-BAK-02`** | 復元先ディレクトリ内に外部（`C:\Windows` 等）を指すジャンクションを作成 | `PathValidationBarrier` が境界脱出を検知し、`SecurityException` で即時拒否。 |
| **`TV-SEC-BAK-03`** | 復元ストリーム書き込み中に強制プロセス終了 | 次回起動時、`RescueSnapshot` のマニフェスト完全性を検証した上で直前状態へ自動ロールバック（自己修復）。 |
| **`TV-SEC-BAK-04`** | 他スナップショットから実体として参照されているコンテナの削除操作 | DB スキーマの **`DeleteBehavior.Restrict`** 制約およびリポジトリ事前検証により削除が物理的に拒否されること (CRIT-05 保証)。 |
| **`TV-SEC-BAK-05`** | 復元前退避（`PreRestoreBackingUp`）の書き込み途中でプロセス強制終了 | **未確定の一時ディレクトリ（`.creating`）が残るのみで正式確定せず、次回起動時にロールバック処理が誤爆して正常な現行セーブデータを破壊しないこと** (C2 保証)。 |
| **`TV-SEC-BAK-06`** | ZIP パスワード処理直後のマネージドヒープ走査 | `string` の使用排除と Storage 層での `Array.Clear` により、ヒープ上に平文パスワードが一切残存しないこと (CRIT-06 / M2 保証)。 |
| **`TV-SEC-BAK-07`** | 復元失敗（`RestoreStatus == Failed`）となった RescueSnapshot のパージ実行 | 24 時間経過時点では削除されず、**7 日間確実に保護・保持されること** (C3 保証)。 |
| **`TV-SEC-BAK-08`** | 差分参照元コンテナが存在する状態での保持世代評価 | `RetentionEvaluator` が参照されているコンテナを削除候補から除外し、ストレージ容量の会計を正しく行うこと (H3 保証)。 |

---

## 2.5 【Boundary 5】SQLite 永続化 ＆ 並行性ベクター

### 脅威シナリオ:
並列スキャナー、バックグラウンドセーブバックアップ、隔離処理が同時に DB 書き込みを行い、`SQLITE_BUSY` ロック競合でトランザクションが脱落する。

### テストベクター:
| Vector ID | テスト入力 / 攻撃手法 | 期待される安全動作 (Expected Result) |
| :--- | :--- | :--- |
| **`TV-SEC-DB-01`** | 100 スレッドから同時に `SaveSnapshotAsync` / `QuarantineFileAsync` 実行 | `IDbWriteQueue` によりデッドロック・ロック例外なしで完了。 |
| **`TV-SEC-DB-02`** | 大量書き込み実行中に並行して `TimelineViewModel` から検索クエリ発行 | 短命 DbContext + WAL モードにより、UI 読み取りクエリがブロックされず即時完了。 |
| **`TV-SEC-DB-03`** | 直列書き込みアクション内で例外発生 | 当該アクションのみ失敗し、短命 DbContext 破棄により後続書き込みが正常継続。 |

---

## 2.6 【Boundary 6】OS 設定保護 ＆ BeforeState 復元ベクター

### 脅威シナリオ:
ユーザーが元々設定していた OS セキュリティ設定（WER 無効化等）が、ゲーム終了時に GST によって勝手に消去・初期化される。アンインストール時に DB ファイルがロックされて削除失敗する。

### テストベクター:
| Vector ID | テスト入力 / 攻撃手法 | 期待される安全動作 (Expected Result) |
| :--- | :--- | :--- |
| **`TV-SEC-WER-01`** | 事前に `DontSendAdditionalData = 1`、`Logging = 1` 等の非既定値を設定してゲーム起動 ➔ 終了 | GST は記録済みの元値を正確に復元し、Windows既定値への置換を行わない。 |
| **`TV-SEC-WER-02`** | 事前に対象キーが存在しなかった環境でゲーム起動 ➔ 終了 | Journal が `Present=false` を明示している対象値のみ削除し、存在しない値を推測して消去しない。 |
| **`TV-SEC-WER-03`** | Primary journal が破損／JSON構造不正、Recovery journal は有効 | Recovery journal の検証済み状態から元値を復元し、復元後に両 artifact を削除する。 |
| **`TV-SEC-WER-04`** | Primary journal が欠損、Recovery journal は有効 | Recovery journal の検証済み状態から元値を復元し、Windows既定値へフォールバックしない。 |
| **`TV-SEC-WER-05`** | Journal が partial / required field 欠落 / fingerprint 不一致 | 復元を fail-closed で停止し、WER の現在値と recovery artifact を保持する。 |
| **`TV-SEC-WER-06`** | Primary / Recovery がともに有効だが payload fingerprint が不一致 | 復元元を曖昧として扱い、WER を変更せず両 artifact を保持する。 |
| **`TV-SEC-WER-07`** | Primary / Recovery がともに存在し、どちらも有効な復元元を持たない | `RecoveryBlocked` とし、Windows既定値へ変更せず artifact を保持する。 |
| **`TV-SEC-WER-08`** | Primary / Recovery がともに存在しない | `NoOutstandingJournal` とし、現在の WER 値を変更しない。 |
| **`TV-SEC-WER-09`** | 復元成功後に同一復元を再実行 | 2回目は安全に `NoOutstandingJournal` となり、不要なレジストリ変更を行わない。 |
| **`TV-SEC-WER-10`** | WER 復元処理が失敗した状態で完全アンインストール継続 | `FullSystemReversionService` は complete success を報告せず、`%LocalAppData%\\GameSecurityTool` の recovery artifact を削除しない。 |
| **`TV-SEC-WER-11`** | recovery artifact cleanup の途中で失敗し、後から uninstall/recovery を再実行 | WER 元値は再利用可能な journal から同一状態へ再確認でき、復旧処理は idempotent に再試行できる。 |

---

## 2.7 【Boundary 7】アーカイブ安全展開 ＆ Zip Slip 防御ベクター

### 脅威シナリオ:
悪意あるフリーゲームアーカイブによるディレクトリトラバーサル（Zip Slip）攻撃や、解凍時にテラバイト規模へ膨張する Zip Bomb 攻撃。

### テストベクター:
| Vector ID | テスト入力 / 攻撃手法 | 期待される安全動作 (Expected Result) |
| :--- | :--- | :--- |
| **`TV-SEC-ARCH-01`** | `../../Windows/System32/evil.dll` を含む Zip Slip アーカイブ展開試行 | `PathValidationBarrier` が管理フォルダ外への書き出しを検知し、展開を即時拒否。 |
| **`TV-SEC-ARCH-02`** | 展開後サイズがディスク容量を超える Zip Bomb アーカイブ展開試行 | ストリーミング容量監視が閾値超過を検知し、ディスク枯渇前に安全中断。 |
| **`TV-SEC-ARCH-03`** | ヘッダー破損または未知アルゴリズムの 7z/RAR アーカイブ展開試行 | 例外を安全に捕捉し、一時展開フォルダを自動消去してユーザーへ案内。 |

---

## 2.8 【Boundary 8】厳格モード起動 (Strict Safe Launch) 遮断ベクター

### 脅威シナリオ:
素性の知れないインディーゲーム・フリーゲームが、バックグラウンドで不正な C2 サーバーへの通信や個人情報の外部送信を試みる。セッション中に強制終了した場合に通信遮断ルールが OS に永続残存する。

### テストベクター:
| Vector ID | テスト入力 / 攻撃手法 | 期待される安全動作 (Expected Result) |
| :--- | :--- | :--- |
| **`TV-SEC-STRICT-01`** | 厳格モード起動中のゲームプロセスからの外部アウトバウンド TCP/UDP 送信 | 一時ブロックルールによりパケットが物理遮断されること。 |
| **`TV-SEC-STRICT-02`** | 厳格モード起動中のゲームプロセスに対する外部インバウンド接続試行 | 一時ブロックルールにより外部からの接続要求がすべて拒否されること。 |
| **`TV-SEC-STRICT-03`** | 厳格モード起動中に OS を強制電源断 ➔ 次回起動 | **起動時ポスト・クラッシュ監査（`ExecutePostCrashAuditAsync`）により、OS 上に取り残された一時ルール（`GST:STRICT-TEMP:`）が自動検出・除去され、通信機能が正常回復すること** (H4 保証)。 |

### approved security decision Anti-Cheat Compatibility Contract Vectors

| Vector ID | テスト入力 / 攻撃手法 | 期待される安全動作 (Expected Result) |
| :--- | :--- | :--- |
| **`TV-SEC-ANTICHEAT-01`** | Valid Authenticode署名を持つAnti-Cheat候補を検出するが、Compatibility Exceptionが無効 | 検出結果は互換性シグナルとして記録されるだけで、Firewallルールは変更されない。 |
| **`TV-SEC-ANTICHEAT-02`** | ユーザーが登録ゲームに対してCompatibility Exceptionを明示承認 | 選択されたゲーム/実行ファイルに対するGST-owned Firewall ruleだけが例外対象となり、Foreign Ruleは変更されない。 |
| **`TV-SEC-ANTICHEAT-03`** | Foreign Ruleを含む環境でCompatibility Exceptionを適用 | 外部作成Firewall ruleを削除・変更・上書きせず、GST-managed scopeだけを評価する。 |
| **`TV-SEC-ANTICHEAT-04`** | Unsigned / invalidly signed Anti-Cheat candidate | Compatibility Exceptionは検出だけでは発動せず、ユーザー承認なしのFirewall mutationも発生しない。 |
| **`TV-SEC-ANTICHEAT-05`** | Anti-Cheat compatibility path実行中のprocess/memory/DLL/driver/graphics hook操作 | 非侵入境界外の操作経路を持たず、第三者Anti-Cheat enforcementへ干渉しない。 |
| **`TV-SEC-ANTICHEAT-06`** | Compatibility Exceptionを解除 | GSTの通常Firewall policyへ戻し、Foreign Ruleを変更せず、監査イベントをCompatibility Exception semanticsで記録する。 |


---

## 2.9 【Boundary 9】COM 解析 ＆ LNK ハイジャックベクター

### 脅威シナリオ:
ショートカットファイルの解析中に COM のスレッドアパートメント（STA/MTA）が衝突してアプリがハングアップ、または無限ループトラバーサルに陥る。

### テストベクター:
| Vector ID | テスト入力 / 攻撃手法 | 期待される安全動作 (Expected Result) |
| :--- | :--- | :--- |
| **`TV-SEC-LNK-01`** | `Arguments` に `powershell.exe -enc ...` を仕込んだショートカット | リスク理由 `LnkSuspiciousCommand` を正確に抽出してベースラインと比較する。 |
| **`TV-SEC-LNK-02`** | ネットワークドライブや未マウントドライブを指す破損ショートカットの解析 | `SLR_NO_UI` フラグにより、ダイアログ表示でハングアップせず直ちにタイムアウトする。 |
| **`TV-SEC-LNK-03`** | 複数スレッド (MTA) から同時に `.lnk` 解析メソッドを呼び出し | 内部の **STA 専用スレッドディスパッチ** によって `InvalidCastException` やハングアップが発生せず、正常に解析が完了すること (HIGH-08 保証)。 |

---

## 2.11 【Boundary 11】External AI Outbound Privacy & Consent Vector

### 脅威シナリオ
未サニタイズのユーザー入力・クラッシュ情報・設定情報等がAI ProviderからGeminiへ到達する、API key存在だけで同意なし送信が発生する、Prompt Preview後の編集で旧承認がバイパスされる、または同意撤回後のstale authorizationで新規HTTP要求が開始される。

### テストベクター
| Vector ID | テスト入力 / 攻撃手法 | 期待される安全動作 (Expected Result) |
| :--- | :--- | :--- |
| **`TV-SEC-AI-01`** | raw / unsanitized `userMessage` を `IAiExplanationProvider` に直接渡す | AI ProviderはPrepared AI Request以外を受理せず、HTTP要求は発生しない。 |
| **`TV-SEC-AI-02`** | outbound text fieldの1項目をsanitizationせずPrepared Requestを生成 | Prepared Request生成を拒否するか、AI send gateでfail-closedし、HTTP要求は発生しない。 |
| **`TV-SEC-AI-03`** | Default OFF / unconsented state + 有効API key | API key存在を理由に送信せず、HTTP `SendAsync` は呼び出されない。 |
| **`TV-SEC-AI-04`** | consent撤回後に旧Prepared Requestを再送 | ConsentRevisionのstaleを検知し、HTTP要求を開始しない。 |
| **`TV-SEC-AI-05`** | Liability Guard未承認でAI送信を試行 | fail-closedで拒否し、HTTP要求を開始しない。 |
| **`TV-SEC-AI-06`** | Prompt Preview後にユーザーがpayloadを編集し、旧approvalを再利用 | 旧approvalを無効化し、再sanitization / re-fingerprint / re-approvalを要求する。 |
| **`TV-SEC-AI-07`** | 承認済みpayload fingerprintとcanonical再サニタイズ後のpayload fingerprintが不一致 | final Infrastructure send gateが拒否し、HTTP要求を開始しない。 |
| **`TV-SEC-AI-08`** | `ILogSanitizer` または `IExternalLogSanitizer` のみを通過した値をAIへ送信 | AI outbound authorizationを得られず、HTTP要求は発生しない。 |
| **`TV-SEC-AI-09`** | 有効Consent + Liability Guard + Prompt Approval + matching fingerprint + sanitized payload | 正常なAI送信経路のみがHTTP要求へ進む。 |
| **`TV-SEC-AI-10`** | ConsentRevisionとAPI key存在を分離した状態でsend gateを呼出 | API key存在のみでauthorizationが成立しないことを確認する。 |

# 3. セキュリティテスト自動化実装規約 (`GST.SecurityTests`)

セキュリティテストは、単なるモックテストにとどまらず、**実際の Win32 ハンドル、ストリーミング I/O、SQLite メモリ DB、ZIP/7z アーカイブ** を用いて決定論的に検証しなければならない。

```csharp
namespace GameSecurityTool.SecurityTests;

using System;
using System.IO;
using System.Security;
using System.Threading.Tasks;
using Xunit;

public class ArchiveSecurityBoundaryTests
{
    [Fact]
    public async Task ExtractArchive_WhenZipSlipPathDetected_ThrowsSecurityExceptionAndAborts()
    {
        // TV-SEC-ARCH-01: Zip Slip パストラバーサル攻撃の遮断を検証
    }

    [Fact]
    public async Task StrictLaunch_AppliesInboundAndOutboundBlockRules()
    {
        // TV-SEC-STRICT-01 & 02: 厳格モード起動時の In/Out 完全遮断を検証
    }
}
```

---

# 3.1 【Boundary 10】Anti-Wiper Emergency Containment Vector

Anti-Wiper は既定OFFの明示オプトイン機能であり、通常のGST機能が持つ「ユーザーデータを勝手に変更・削除しない」境界の限定例外として、監視対象プロセスの一時停止または終了を行う。ファイルイベント単体から発生元ゲームを断定せず、単一ActiveProtectionSession、PID世代一致、明示設定、限定されたOS操作をすべて満たした場合だけプロセス制御を許可する。

| Vector ID | テスト入力 / 攻撃手法 | 期待される安全動作 |
|---|---|---|
| TV-WIPER-01 | Canary既存同名ファイル | GSTは既存ファイルをCanary採用せず、上書き・削除しない。 |
| TV-WIPER-02 | Canary新規配備 | GST作成直後のCanaryを所有対象へ登録し、Watcher有効化前の自己配備イベントでブレーカーを発動しない。 |
| TV-WIPER-03 | 単一ファイル重複Changedイベント | 同一パスへの重複通知だけではBurstしきい値を満たさない。 |
| TV-WIPER-04 | PID再利用 | 登録時開始時刻と現在のProcess.StartTimeが一致しない場合、Suspend/Terminateを実行しない。 |
| TV-WIPER-05 | `del` を含む通常ファイル名 | コマンド文脈でない文字列だけでは子プロセスを終了しない。 |
| TV-WIPER-06 | Watcher Error / WMI Stopped | `IsOperational=false` とし、正常稼働表示を継続しない。 |
| TV-WIPER-07 | Application登録片側失敗 | Child GuardまたはCircuit Breakerの登録失敗時、Coordinatorは保護セッションを成立させず、成功側の登録を解除する。 |
| TV-WIPER-08 | 個人rootの無承認監視 | Anti-WiperがOFF、またはrootがユーザーに明示選択されていない場合、Desktop/Documents/Pictures等を監視対象にせず、Canaryも配置しない。 |
| TV-WIPER-09 | 個人ファイル内容へのアクセス試行 | 保護rootに含まれる個人ファイルについて、本文・画像・メール・文書・セーブデータ内容を読み取り、解析、収集、保存する経路を持たない。 |
| TV-WIPER-10 | Personal-Asset Protection Exceptionの最小権限 | 明示選択された個人rootではファイルシステム変更通知と保護判定に必要な最小限のメタデータのみを扱い、raw user path等を通常ログへ未処理で出力しない。 |
| TV-WIPER-11 | Canary既存ファイル衝突 | 選択root直下に同名のユーザーファイルが存在する場合、GSTは既存ファイルを上書き・削除・Canary化せず、配備を失敗/スキップとして扱う。 |
| TV-WIPER-12 | 既存同名Canary衝突後のOperational判定 | `IsOperational=false` とし、ファイルイベント起点の自動プロセス制御を実行しない。 |
| TV-WIPER-13 | 設定root欠落・Watcher未成立 | 全設定rootを監視できない場合は `IsOperational=false` とし、正常稼働を表示しない。 |
| TV-WIPER-14 | Watcher Error発生中の残存Watcherとin-flightイベント | 残存Watcherを停止し、`IsOperational=false` とする。以後のイベントおよび障害後に到達したin-flight評価はプロセス制御を実行しない。 |
| TV-WIPER-15 | Manual Terminate 1回目 / 2回目 / 期限切れ | 1回目はOSプロセス制御を行わず確認待ちとし、10秒以内の2回目だけ既存PID + StartTime検証済みTerminateへ進む。期限切れ後は新しい1回目から再確認する。 |
| TV-WIPER-16 | Watcher fault automatic recovery | 障害時は `IsOperational=false` のまま残存Watcherを停止し、2秒→5秒→15秒→30秒のbounded backoffで既存初期化のみを再試行する。全root/Canary成立後のみOperationalへ戻る。 |
| TV-WIPER-17 | Recovery cancellation | WIPER無効化・設定変更・Dispose後に旧recovery loopがWatcherを再作成しない。 |
| TV-WIPER-18 | Canary replacement ownership | GSTが以前所有していたCanaryパスがユーザー/第三者ファイルへ置換された場合、復旧/撤収処理はそのファイルを削除せず、Canary collisionとしてOperationalを拒否する。 |
| TV-WIPER-19 | Recovery concurrency | 自動復旧、`UpdateConfig`、Watcher fault teardownが同時発生してもWatcher lifecycleの並行再構成を起こさず、stale recoveryは無効化される。 |

---


## Cross-Boundary Verification — Uninstall Crash / Power-Loss Durable Recovery

| Vector | Scenario | Expected Result |
|---|---|---|
| TV-SEC-UNINSTALL-01 | Power loss immediately after `Prepared` | Recovery journal remains valid; no guessed mutation state; rerun can resume safely. |
| TV-SEC-UNINSTALL-02 | Crash after QuarantineDrain mutation but before step commit | Recovery reconciles quarantine state and completes/reconciles the step idempotently. |
| TV-SEC-UNINSTALL-03 | Crash after WER mutation but before `WERRestore` durable commit | approved security decision exact-original-state recovery remains authoritative; uninstall does not default/fallback. |
| TV-SEC-UNINSTALL-04 | Crash after Firewall rule removal but before `FirewallCleanup` commit | Recovery observes actual rule absence and does not perform unsafe compensating restoration. |
| TV-SEC-UNINSTALL-05 | Crash after ConnectionDrain but before durable step commit | Recovery safely rechecks the connection state and continues. |
| TV-SEC-UNINSTALL-06 | Crash during `InternalCleanup` | Later recovery does not require the deleted GST DB; external-step progress remains derivable from the uninstall journal and OS state. |
| TV-SEC-UNINSTALL-07 | Durable `Committed` written, then crash before journal deletion | Next recovery treats transaction as committed and retries cleanup only. |
| TV-SEC-UNINSTALL-08 | Journal deletion failure after `Committed` | Transaction remains logically committed; failure to delete the artifact does not reopen uninstall steps. |
| TV-SEC-UNINSTALL-09 | Partial/malformed journal write | Recovery fails closed and preserves the journal; no step state is guessed. |
| TV-SEC-UNINSTALL-10 | Unknown schema or step identifier | Recovery fails closed and preserves artifacts; no silent compatibility guess. |
| TV-SEC-UNINSTALL-11 | Rerun after every injected interruption point | Repeated recovery converges to `Committed` or an explicitly preserved blocked/failed state without unsafe reversal. |
| TV-SEC-UNINSTALL-12 | External uninstaller attempts early recovery-journal deletion | Deletion is rejected/blocked until durable `Committed`; full reversion cannot be reported early. |
| TV-SEC-UNINSTALL-13 | Power loss between every external mutation and its completion record | No GST-created Firewall/WER state remains permanently orphaned solely due to the interruption; supported forward recovery remains available. |

These are verification requirements. Executable uninstall failure-injection tests remain gated by the implementation-readiness gate.
## Cross-Boundary Verification — Independent Safe Mode Recovery Host (approved security decision)

| Vector | Scenario | Expected Result |
|---|---|---|
| TV-SEC-RECOVERY-01 | Normal GST startup health check fails because the SQLite DB is corrupt/unavailable | Normal MainWindow startup is not continued; Recovery Host handling is selected. |
| TV-SEC-RECOVERY-02 | Normal GST self-integrity verification fails | Normal startup is aborted and Recovery Host handling is selected; no normal feature initialization proceeds. |
| TV-SEC-RECOVERY-03 | User directly launches Recovery Host while GameSecurityTool.exe cannot start | Recovery Host starts without requiring successful normal-app startup. |
| TV-SEC-RECOVERY-04 | Recovery Host starts while normal SQLite DB is unavailable | Recovery Host remains operable for approved recovery actions without opening/depending on the normal DB. |
| TV-SEC-RECOVERY-05 | Recovery Host starts while normal configuration store is unavailable | Recovery Host remains operable using only its approved independent recovery configuration/state. |
| TV-SEC-RECOVERY-06 | Embedded RecoveryConfig.json is missing/unreadable | Recovery Host fails closed; no external mutable fallback or inferred policy is used. |
| TV-SEC-RECOVERY-07 | Embedded RecoveryConfig.json contains invalid JSON/schema | Recovery Host fails closed and performs no recovery mutation. |
| TV-SEC-RECOVERY-08 | Embedded RecoveryConfig.json integrity validation fails | Recovery Host fails closed and performs no recovery mutation. |
| TV-SEC-RECOVERY-09 | User launches a recovery action without explicit confirmation | No externally visible or destructive recovery action is performed. |
| TV-SEC-RECOVERY-10 | Recovery action completes but post-action validation fails | Result is not reported as successful recovery; required evidence/state is preserved according to the applicable recovery contract. |
| TV-SEC-RECOVERY-11 | Recovery Host own integrity/startup validation fails | Recovery operation is blocked fail-closed; no guessed recovery policy is executed. |
| TV-SEC-RECOVERY-12 | Recovery is attempted from a distribution whose trust/integrity is not established | approved security decision provides no claim that recovery from a fully untrusted/compromised distribution artifact is safe or guaranteed. |

These are verification requirements. Executable Recovery Host/startup failure-injection tests remain gated by the implementation-readiness gate.

## Cross-Cutting Verification — Error Handling / Fail-Safe Observability (approved security decision)

| Vector | Scenario | Expected Result |
|---|---|---|
| TV-SEC-ERROR-01 | Empty `catch { }` or `catch (Exception) { }` with no observable handling | Prohibited; implementation/design review fails. |
| TV-SEC-ERROR-02 | Expected failure handled through broad catch instead of canonical `Result<T>` / result DTO | Prohibited; expected failure contract remains explicit. |
| TV-SEC-ERROR-03 | Secondary temporary-artifact cleanup fails after a primary failure | Primary failure remains authoritative; cleanup handling does not manufacture success. |
| TV-SEC-ERROR-04 | Cleanup failure can affect transaction, recovery, security-boundary, or protected-asset correctness | Cleanup failure is not suppressible; failure remains observable and participates in the applicable contract. |
| TV-SEC-ERROR-05 | Explicit defensive broad-catch boundary returns conservative fail-safe value | Sanitized observability is emitted through the global logging boundary and the conservative failure result is preserved. |
| TV-SEC-ERROR-06 | Broad fail-safe catch returns a success/authorization/completed state | Prohibited; fail-safe conversion must not masquerade as success. |
| TV-SEC-ERROR-07 | Exception observation contains raw sensitive data at a physical sink | Prohibited; approved security decision sanitization boundary remains mandatory. |

These are verification requirements. Executable conformance tests remain subject to the applicable implementation-readiness gate.


# 4. Final Security Boundary Statement

GST のセキュリティ境界は、「破られないこと」を過信しない。

```
Verify Every Operation
Limit Every Privilege
Preserve Every Evidence
Recover Every Asset
Never Leave Orphaned Rules
```

---

End of Document
```

---

