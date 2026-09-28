# GameSecurityTool Save Backup Core & Feature Control Specification

**Document ID:** GST-FEAT-SAVE-CTRL-001  
**Version:** 2.2 (Anti-Wiper Canary Exclusion Boundary Edition)
**Status:** Approved Design Specification  
**Target Layer:** Application / Domain  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Core Philosophy & Design Principles

Save Data Secure Auto Backup は、以下の製品哲学に厳格に従って動作する。

1. **User Control First**:
   バックアップの有効化、モード選択、復元の実行はすべてユーザーが完全に制御する。ツールの独断によるセーブデータの自動復元や勝手な上書きを絶対に行わない。
2. **Privacy First & Local First**:
   セーブデータは完全にローカルストレージ内でのみ複製・保護される。いかなる外部サーバーへの通信も行わない。
3. **Non-Destructive (非破壊性)**:
   バックアップ処理によって元のセーブデータファイルを破損・変更・ロックさせてはならない。
4. **SSD Low Impact**:
   SSDの寿命およびゲームパフォーマンスへの影響を最小化するため、無変更ファイルの再コピーや不要なフルハッシュ計算を排除する。

---

# 2. Universal Feature Control Policy 統合

## 2.1 Feature 分類と識別子
Save Backup は「ユーザー向け保護機能（User Controlled Feature）」として定義し、製品共通の Universal Feature Control Policy に従って管理する。

- **FeatureId**: `SaveDataSecureBackup`
- **Category**: `UserProtectionFeature`

```text
┌────────────────────────────────────────────────────────────────────────┐
│                   Mandatory Security Foundation                        │
│  (Path Validation, Secure Storage, Audit Integrity, Crash Recovery)   │
│  ※ ユーザーによる無効化不可 (常時有効)                                   │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ (基盤上で実行)
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│                      User Controlled Features                          │
│  - SaveDataSecureBackup  <── [Enabled / Disabled 制御可能]             │
│  - WebLinkProtection                                                   │
│  - LnkHijackDetection                                                  │
└────────────────────────────────────────────────────────────────────────┘
2.2 Security Foundation との分離
ユーザーが Save Backup 機能を Disabled（無効）に設定した場合でも、製品自身の安全基盤（パス検証、セキュアストレージ、監査チェーン整合性保護）は一切無効化されない。
3. FeatureScope & 状態解決モデル
3.1 Scope 定義
機能の有効状態は Global および GameProfile の2階層で管理する。
code
Text
FeatureScope
├─ Global       (全ゲーム共通の既定状態)
└─ GameProfile  (特定ゲームに対する個別状態)
3.2 選択可能な状態値 (FeatureState)
Game Profile では以下の3つの状態から選択する。
UseGlobal: Global の設定状態を自動継承する（デフォルト）。
Enabled: 当該ゲームのみバックアップ機能を強制的に有効化する。
Disabled: 当該ゲームのみバックアップ機能を強制的に無効化する。
3.3 有効状態の評価解決フロー (Effective State Resolution)
code
Text
[ゲームセッション開始 / トリガー検知]
                │
                ▼
      [GameProfile 設定確認]
                │
        ┌───────┴───────┐
 (Overrideあり)    (UseGlobal)
        │               │
        ▼               ▼
 [指定設定を適用]   [Global設定を継承]
        │               │
        └───────┬───────┘
                │
                ▼
   [Effective Feature State]
   ├─ Enabled  ──> バックアップパイプラインを実行
   └─ Disabled ──> 処理を安全にスキップ (Audit記録のみ)
4. バックアップ対象パスの境界モデル (Target Data Boundary)
4.1 管理対象パス
バックアップ対象は、ユーザーが Game Profile に登録したセーブデータパス（または自動検出された候補パス）のみに限定する。
代表例：
%USERPROFILE%\Saved Games\<GameName>
%USERPROFILE%\Documents\My Games\<GameName>
<SteamPath>\userdata\<UserId>\<AppId>
ユーザーが明示的に指定したカスタムパス

**Anti-Wiper Canary 境界:** `CanaryTrapManager.IsCanaryPath(filePath) == true` のパスは、バックアップ対象として列挙・スナップショット化・ハッシュ算出・コンテナ保存してはならない。これはCanaryを「安全なファイル」と判定するためではなく、GST自身の防衛資産をユーザーデータ保護処理から分離するための除外契約である。同名の第三者ファイルを名前だけで除外してはならない。
4.2 ドメイン非依存ルール
既知のゲームフォルダ構造（Steam, My Games 等）の文字列判定を Domain 層にハードコードしてはならない。パスの収集および供給は Application 層の SaveDataPathProvider を通じて行う。
4.3 Trusted Location との関係
セーブデータフォルダが Trusted Location に登録されている場合でも、それはスキャンをスキップするフラグではなく、リスクスコアの減点補正（Risk Modifier） としてのみ扱う。
5. バックアップ実行トリガー (Backup Triggers)
バックアップの実行要求は以下のイベントから発生する。
1. Automatic Triggers (自動)
Game Start: ゲーム起動検知時、起動前スナップショットを確認。
Game End: ゲーム終了検知時（プロセス消滅後の Grace Period 経過後）、変更されたセーブデータを保存。
Save Change Detected: ファイル監視によってセーブフォルダ内の更新を検知した保護タイミング。
2. Manual Trigger (手動)
User Requested: 画面上の「今すぐバックアップ」ボタン押下。
※トリガーが発生した場合でも、後述する Manifest 比較により「差分なし」と判定された場合は実際のディスク書き込みを行わない。
6. バックアップ動作モード (Backup Modes)
ユーザーは負荷と保護のバランスに応じて以下の3モードから選択できる。
モード名	動作特性	適用ユースケース
Standard (標準)	差分検出時のみ変更ファイルをバックアップ。不要な書き込みを抑制。	通常のゲームプレイ環境（推奨）
SSD Low Impact (超低負荷)	連続変更を一定時間集約（Debounce）し、同一セッション内の世代生成頻度を抑制。重複スナップショットを排除。	SSDの書き込み寿命を最重要視する環境
Maximum Protection (最大保護)	変更検知時およびゲーム終了時に確実にスナップショットを生成。	頻繁にMODの導入・設定変更を行う環境
7. 監査・構成変更追跡要件
Feature State の変更操作はすべて OperationId を付番し、改ざん検知ログ（Audit Log）へ記録する。
記録必須イベント：
SaveBackupFeatureEnabled
SaveBackupFeatureDisabled
SaveBackupPolicyChanged
SaveBackupScopeOverridden
保存情報：
OperationId, FeatureId, Scope, GameProfileId, OldState, NewState, ModifiedBy, Timestamp