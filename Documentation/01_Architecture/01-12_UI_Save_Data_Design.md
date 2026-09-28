# 01-12: UI Save Data Design

**Document ID:** GST-ARCH-UI-SAVEDATA-001  
**Version:** 0.4 (Save Data Annotated Wireframe / Visual UX Review)  
**Status:** Design Working Draft — Not an independent feature authority  
**Authority Relationship:** Subordinate to GST-BASELINE-011, GST-BASELINE-047, applicable approved module / feature specifications, and Change Control decisions  
**Parent Concept:** [01-08_UI_Information_Architecture_and_Interaction_Concept.md](01-08_UI_Information_Architecture_and_Interaction_Concept.md)  
**Historical Source Mapping:** Former Section 31 of the parent 01-08 UI working draft  
**Current Ownership:** This document is the maintained location for the Save Data material after the working-draft split.
**Purpose:** Organize existing UI information-architecture and interaction design for this scope without expanding Runtime Features, Public Contracts, Domain Status, Security capabilities, or Implementation Readiness.

## Document Boundary

This document preserves UI design material that was formerly organized as Section 31 in the parent 01-08 UI working draft. After the working-draft split, 01-08 retains global IA / interaction principles while this document owns the Save Data material.

The preserved section numbering is historical traceability only. It does not imply that those sections still exist in 01-08.

This document does not authorize Runtime implementation by itself. Existing formal baselines, approved module / feature specifications, Public Contracts, Security boundaries, and Change Control decisions remain authoritative where scope overlaps.

## Related Authoritative Documents

- [02-09_Save_Backup_UI_and_UX.md](../02_Features/Security/Save_Data_Secure_Auto_Backup/02-09_Save_Backup_UI_and_UX.md) — Save Backup UI authority
- [05_Save_Backup_and_Data_Ownership_Model.md](../00_Baseline/GameProtection/05_Save_Backup_and_Data_Ownership_Model.md) — Save/data ownership baseline
- [47_User_Experience_and_Interface_Design_Governance_Model.md](../00_Baseline/Governance/47_User_Experience_and_Interface_Design_Governance_Model.md) — Formal UX governance

---

# 31. Save Data UX Design

Save Dataは、「どのゲームの、どのセーブ世代を守っているか」と「問題が起きたときに、どの世代へ安全に戻せるか」を理解するための画面とする。

Save DataとRecoveryの役割は分ける。

- **Save Data:** 何を保存しているか、どの世代があるか、容量をどれだけ使っているか、どの操作を始めるかを理解する。
- **Recovery:** 事故や失敗が発生した後に、現在の問題をどう安全に解消するかを理解する。

Save DataからRestoreを開始することは可能だが、Restoreそのものの承認責務は共通のRecovery / Confirmation flowへ引き渡す。

## 31.1 Save Data Mental Model

ユーザーが最初に理解する対象を、内部のBackup RepositoryやTransactionではなく次の4つに整理する。

    現在のセーブ       ← PC上で今使われているデータ
    バックアップ世代   ← 過去の時点へ戻るための保存
    バックアップ保全   ← 整合性・利用可能性を確認できる状態
    復元               ← 選んだ世代を現在のデータへ戻す操作

特に「バックアップが存在する」と「そのバックアップを安全に復元できることを確認済み」は同じ意味として扱わない。

UIでは可能な範囲で、

    存在するか → いつの世代か → 何を含むか → 整合性状態 → 何が起きるか

の順で説明する。

## 31.2 Save Data Screen Primary Structure

Save Data画面は、次の4領域を基本とする。

1. Save Data Overview
2. Game / Save Selection
3. Generation History
4. Actions / Storage Context

概念配置:

    ┌──────────────────────────────────────────────────────────────────────┐
    │ ←  セーブデータ                                      [🔎 検索]       │
    ├──────────────────────────┬───────────────────────────────────────────┤
    │ ゲーム / セーブ対象       │ 🎮 Game A                                │
    │                          │ 最新セーブ: 確認中 / あり / なし          │
    │ ● Game A                 │ バックアップ: 3 世代                     │
    │ ● Game B                 │                                           │
    │ ● Game C                 │ [ 今すぐバックアップ ]                   │
    │                          │                                           │
    │ フィルター               │ ┌─────────────────────────────────────┐ │
    │ [ピン留め] [最近]         │ │ 最新世代  2026-09-28 23:10          │ │
    │ [履歴あり]                │ │ 整合性: 確認済み                     │ │
    │                          │ │ タグ: MOD導入前  📌                  │ │
    │                          │ │ [詳細]   [この世代から復元をレビュー] │ │
    │                          │ └─────────────────────────────────────┘ │
    │                          │                                           │
    │                          │ ストレージ使用量 / 保存先                │
    │                          │ [管理]                                    │
    └──────────────────────────┴───────────────────────────────────────────┘

これは構造検討用Wireframeであり、最終的な寸法、色、カード形状、アイコンはVisual Design段階で決定する。

## 31.3 Save Data Overview

画面上部では、技術情報を先に並べず、ユーザーが現在の状況を判断できる順序とする。

推奨順:

1. Selected Game / Save Context
2. 現在のセーブ状態
3. 最新バックアップ世代
4. バックアップの保全状態
5. 利用可能な主要操作
6. ストレージ状況

「バックアップあり」だけを緑色にして、保存世代の内容や整合性を説明しない構成は避ける。

ストレージについても、空き容量不足とバックアップ異常を同じ警告として扱わない。

### 31.3.1 Save Context Scope

The Save Data page should make the currently selected **Game / Save target** explicit whenever more than one game or save target is available.

Selection is UI context only. It does not prove:

- that the game is currently running,
- that the current save is protected at runtime,
- that the selected generation is safe to restore,
- or that a restore operation has been approved.

Where a state is game-scoped, the game identity should remain visible near that state rather than relying only on the navigation highlight.

## 31.4 Current Save vs Backup State

現在PC上にあるセーブとバックアップ世代は、明確に分離する。

    現在のセーブ
    Game A / PC上の現在データ
    最終確認: たった今

    バックアップ
    3世代
    最新: 2026-09-28 23:10
    最古: 2026-09-20 19:42

ユーザーがバックアップ世代を選択しても、その時点では現在のセーブを書き換えない。

選択は読み取り操作であり、Restore開始とは明確に区別する。

## 31.5 Generation History

世代一覧は「古い順のファイル一覧」ではなく、「どの時点へ戻れるか」を中心に表示する。

各世代について、必要な範囲で:

- 保存日時
- Game / Save context
- 世代の主要メタデータ
- 整合性確認状態
- Pin / Tag
- 概算サイズ
- 復元可能性に関する注意
- 詳細への導線

を表示する。

内部のTransactionId、Repository名、Storage Port名などは通常表示しない。

世代の選択だけでは復元を開始しない。

### 31.5.1 Generation State vs Restore Eligibility

A generation may have several distinct presentation attributes:

- **Exists:** the generation record is present.
- **Integrity state:** the stored generation has the applicable integrity / validation result.
- **Restore eligibility:** the current product state permits the generation to enter the approved Restore Review flow.

These meanings must not be collapsed into a single green "safe" indicator.

A generation marked as existing or integrity-verified must not be presented as if the user has already approved its restoration.

## 31.6 Backup Action

通常の単一ゲームバックアップは、日常操作として短い導線で開始できる。

推奨導線:

    Gameを選択
      ↓
    今すぐバックアップ
      ↓
    進行状況
      ↓
    完了 / 失敗
      ↓
    結果・保存先・世代情報を確認

バックアップ作成はユーザーの既存セーブを置換するRestoreとは異なるため、毎回長い危険操作確認を要求しない。

ただし、長時間化する可能性、保存先容量不足、対象を確定できない状態などがある場合は、開始前にその事実を表示する。

完了表示では少なくとも「何を保存したか」「どこへ保存されたか」「結果」を確認できるようにする。

## 31.7 All-Games Bulk Backup

全ゲーム一括バックアップは便利さを優先しつつ、対象と負荷を見えないまま実行しない。

一括操作の開始前に、可能な範囲で:

- 対象ゲーム数
- バックアップ対象の概数
- 推定書き込み量
- 利用可能な空き容量
- 対象外 / スキップ予定がある場合の理由

を表示する。

このPreflightは「何が起きるか」を理解するための確認であり、個々のゲームのバックアップ成功を保証するものではない。

一括処理中は、全体進捗と現在処理中の対象を表示し、個別失敗が発生しても全体結果から隠さない。

## 31.8 Restore Entry Point

Save Data画面からRestoreを開始する場合、Primary Actionとして「復元」を無条件に目立たせるのではなく、**選択した世代へ戻すためのReview導線**として設計する。

推奨:

    [世代を選択]
        ↓
    [復元をレビュー]
        ↓
    Review
      - どのゲーム / セーブか
      - 現在のデータに何が起きるか
      - 選択したバックアップ世代
      - 復元後に期待される状態
      - RescueSnapshotによる復元前退避
      - 実行を続ける場合の注意
        ↓
    Confirm
        ↓
    Execute
        ↓
    Verify

Restoreの暗号学的検証やContainment検証が成功したこと自体を、ユーザーの上書き承認とは扱わない。

### 31.8.1 Restore Review Boundary

The Save Data surface owns the decision to **enter** Restore Review; the destructive restore execution remains governed by the common Recovery / Confirmation path and its authoritative feature contract.

When a generation is not currently eligible for restore, the UI should explain the reason or route the user to the appropriate details surface rather than exposing an executable restore control that is expected to fail immediately.

## 31.9 Restore Review Surface

Review画面では、技術的な実装詳細を最初に示すのではなく、ユーザーの判断に必要な差分を中心にする。

必須の説明項目:

- **何を戻すか**
- **現在の何が置き換わるか**
- **復元元はどの世代か**
- **復元後に何が変わるか**
- **復元前に現在データをRescueSnapshotへ退避すること**
- **失敗時の復旧経路**
- **追加の注意事項**

必要な詳細情報はDetails Expanderへ分離する。

「暗号化済み」「署名済み」などの抽象語だけで安全性を説明せず、ユーザーが実際に知る必要がある影響を併記する。

## 31.10 RescueSnapshot UX

RescueSnapshotは、ユーザーに見えない内部処理として完全に隠すのではなく、「復元前に現在のデータを保全する」という意味で説明する。

推奨表示:

    復元前の現在データを退避
    ✓ 完了
    復元処理を開始できます

    復元中に問題が発生した場合は、
    退避した状態からの復旧を試みます。

ただし、UIが「データ損失を絶対に防ぐ」「必ず元通りになる」などの保証表現を新たに追加してはならない。

復元前退避が完了したことを確認できない場合、ユーザーに通常の成功状態として見せたまま復元を続行する構成は避ける。

## 31.11 Restore Progress and Result

復元中は「処理中です」だけではなく、現在の段階を見えるようにする。

概念ステップ:

    検証中
      ↓
    現在データの保全中
      ↓
    復元中
      ↓
    検証・整合性確認
      ↓
    完了

UI上の表示は、内部のPersistence / Domain state名称をそのまま公開するのではなく、ユーザーが意味を理解できる文言へ変換する。

失敗した場合は、単に「Restore Failed」とせず、

- 何が完了したか
- 何が未完了か
- 現在のデータが置き換わったかどうか
- 復旧処理が実施されたか
- 次にユーザーが確認すべきこと

を区別して表示する。

## 31.12 Startup Recovery / Incomplete Restore

前回起動中にRestoreが未完了だった場合、通常のバックアップ一覧より先に、Recovery状態をユーザーへ知らせる必要がある。

表示例:

    ⚠️ 前回の復元処理が完了していません

    現在の状態を確認しています。
    安全確認が完了するまで、通常の復元操作は開始できません。

    [復旧状態を確認]

起動時自己修復が成功した場合も、単に無かったことにせず、必要な範囲で「何が起きたか」「現在どうなっているか」を履歴へ残す。

自己修復失敗時はCritical Data Protection状態として扱い、通常のDND通知に埋没させない。

## 31.13 Pinning and Tags

Pin / Tagは、世代を選びやすくするための意味づけとして扱う。

例:

- 📌 MOD導入前
- 📌 ボス戦直前
- ローカライズ変更前
- 大型アップデート前

Pinは「この世代が重要」というユーザー意思を表すものであり、世代の暗号学的完全性や復元成功を保証する表示には使わない。

タグ編集は世代本体の復元や削除と混同しない。

## 31.14 Storage Usage and Capacity

ストレージ使用量は「何GB使っているか」だけでなく、必要な判断ができる単位に分解する。

可能な表示:

    Save Data Backups
    現在使用量: 18.4 GB
    空き容量: 72.1 GB
    保存先: D:\\GST\\Backups

    Game A  8.1 GB
    Game B  6.4 GB
    Game C  3.9 GB

大容量項目から詳細へ遷移できる構成は採用候補とするが、グラフや視覚効果だけを主導線にしない。

保存先の変更・移行など、失敗時影響が高い操作は既存の確認ルールへ接続する。

## 31.15 Smart Cleanup

Smart Cleanupは、ユーザーの保存資産を削除する操作であるため、バックアップ作成とは扱いを分ける。

最初の画面では:

- 削除候補
- なぜ候補なのか
- Pinされた世代 / 重要世代がどう扱われるか
- 削除後に残る世代
- 解放見込み容量

をReviewできるようにする。

実削除は、既存のMaterial / destructive operationに対する確認ルールに従い、一覧上の「候補表示」と実際の削除を同一操作にしない。

## 31.16 Export / External Storage

外部保管・移送は、通常のバックアップ一覧と同じ場所からアクセスできるが、「外部へコピーする操作」と「GSTがローカルで復元できるバックアップ」を区別する。

エクスポート時には可能な範囲で:

- 出力先
- 対象世代
- 出力形式
- 暗号化の有無
- GST互換性に関する注意

を表示する。

汎用アーカイバで外観上展開できることと、GSTの暗号化payloadを復元できることを同一視しない。

## 31.17 No Backup / Stale / Problem States

Save Data画面は、正常な世代一覧だけを設計して終わらせない。

### No Backup

    バックアップはまだありません

    このゲームの現在のセーブデータを
    最初のバックアップとして保存できます。

    [ 今すぐバックアップ ]

### Backup Exists but Requires Attention

    バックアップはあります
    一部の世代について確認が必要です

    [詳細を確認]

### Backup Operation Failed

    バックアップを完了できませんでした

    原因と保存できた範囲を確認してください。
    既存のバックアップ世代はそのまま保持されています。

    [結果を見る]

これらの表示は具体的なDomain statusを新設するものではなく、既存状態をユーザー向けに意味変換するためのPresentation patternとする。

## 31.18 Historical / Uninstalled Games

Games画面と同様に、アンインストール済みゲームのバックアップは、現在プレイ中のゲームとは別の状態として扱う。

例:

    Game A
    現在: インストール済み

    Game B
    現在: 未インストール
    バックアップ: 5世代

未インストールであること自体をGST異常と表示しない。

過去のバックアップを保持していることと、現在のゲーム環境を保護していることを同一視しない。

## 31.19 Navigation to Recovery

Save Dataで「復元をレビュー」を押した後、ユーザーが「どこへ来たのか分からない」状態を作らない。

推奨Context:

    セーブデータ
      → Game A
        → 世代 2026-09-28 23:10
          → 復元レビュー

Review後に実行する場合はRecovery contextへ自然に移行し、BackでSave Dataの選択世代・検索・フィルターを可能な範囲で保持する。

Recovery中にSave Data一覧の再編集へ飛ばすなど、危険操作と並行してContextを増やす構成は避ける。

## 31.20 Save Data Action Efficiency

日常利用で頻度が高い:

    Game選択 → 最新バックアップ確認
    Game選択 → 今すぐバックアップ
    Game選択 → 世代選択 → 復元レビュー

は短い導線で到達できるようにする。

一方、

- 世代削除
- Smart Cleanup
- 保存先変更 / 移行
- 外部エクスポート
- 大量世代の管理

は、視覚的に明確な別グループとして扱い、バックアップや復元のPrimary Actionと誤クリックしにくい配置とする。

## 31.21 Save Data Search / Filtering

検索・フィルターは、Game Listだけでなく世代管理にも適用できる構造を検討する。

候補:

- Game Name
- Tag
- Pin
- Date range
- Backup availability

ただし、検索フィルターの追加によって内部の保存形式やDomainモデルを先行変更しない。

検索結果から世代を選択しても自動復元は開始しない。

## 31.22 Save Data Wireframe — Working Baseline

    ┌────────────────────────────────────────────────────────────────────┐
    │ ← セーブデータ                                      [🔎 検索]      │
    ├────────────────┬───────────────────────────────────────────────────┤
    │ ゲーム         │ 🎮 Game A                                         │
    │                │                                                   │
    │ ● Game A       │ 現在のセーブ: 確認済み                           │
    │ ● Game B       │ バックアップ: 3世代                             │
    │ ● Game C       │                                                   │
    │                │ [ 今すぐバックアップ ]                           │
    │ [📌 ピン留め] │                                                   │
    │                │ 最新世代                                         │
    │                │ 2026-09-28 23:10                                  │
    │                │ 整合性: 確認済み                                 │
    │                │ 📌 MOD導入前                                      │
    │                │ [詳細]   [復元をレビュー]                        │
    │                │                                                   │
    │                │ 履歴                                              │
    │                │ ├ 2026-09-27 21:40  [詳細]                       │
    │                │ └ 2026-09-20 19:42  [詳細]                       │
    │                │                                                   │
    │                │ ストレージ                                       │
    │                │ 18.4 GB / 空き 72.1 GB                           │
    │                │ [ストレージ管理]                                  │
    └────────────────┴───────────────────────────────────────────────────┘

### 31.24 Review Findings — 2026-09-29

The focused review against the formal Save Backup UI specification, Save/Data Ownership baseline, and UX governance identified three presentation ambiguities and clarified them:

| Finding | Resolution |
|---|---|
| Selected Game / Save context could be visually separated from the state it describes | Added explicit Save Context Scope guidance. |
| Generation existence, integrity, and restore eligibility could be read as one "safe" state | Added explicit Generation State vs Restore Eligibility semantics. |
| A Restore control could imply execution rather than entry into the controlled Review flow | Added a Restore Review Boundary and unavailable-action guidance. |

No backup storage contract, Restore transaction contract, Public Contract, Domain Status, or Security capability was changed.

## 31.23.1 Annotated Wireframe — Save Data / Visual UX Working Baseline,,このAnnotated Wireframeは、`01-09_UI_Visual_Design_System.md`、`01-17` の共通Pattern、`01-18` のJourney Gate、`01-19` のContext PreservationをSave Dataへ適用した作業基準である。最終Pixel値、WPF resource、Icon assetはVisual QA / implementation段階で確定する。,,```text,┌────────────────────────────────────────────────────────────────────────────┐,│ GST / Save Context                         セーブデータ      [🔎 検索]    │,├──────────────────────┬─────────────────────────────────────────────────────┤,│ 対象ゲーム             │ 🎮 Game A — Save Context                           │,│                       │                                                     │,│ ● Game A              │ 現在のセーブ                                       │,│ ○ Game B              │ 状態: 確認済み / 確認中 / 要確認                  │,│ ○ Game C              │ 最終確認: 直近                                    │,│                       │                                                     │,│ フィルター             │ バックアップ: 3 世代                              │,│ [📌 ピン留め]         │ 最新: 2026-09-28 23:10                            │,│ [最近] [履歴あり]     │                                                     │,│                       │ [ 今すぐバックアップ ]                             │,│                       │                                                     │,│                       │ ┌────────────────────────────────────────────────┐ │,│                       │ │ 世代 2026-09-28 23:10                           │ │,│                       │ │ 存在: あり  整合性: 確認済み                    │ │,│                       │ │ 復元適格性: Reviewへ進めます                   │ │,│                       │ │ 📌 MOD導入前                                   │ │,│                       │ │ [詳細]            [復元をレビュー]             │ │,│                       │ └────────────────────────────────────────────────┘ │,│                       │                                                     │,│                       │ 履歴                                               │,│                       │ ├ 2026-09-27 21:40  [詳細]                       │,│                       │ └ 2026-09-20 19:42  [詳細]                       │,│                       │                                                     │,│                       │ ストレージ                                         │,│                       │ 使用 18.4 GB / 空き 72.1 GB                       │,│                       │ [ストレージ管理]                                  │,├──────────────────────┴─────────────────────────────────────────────────────┤,│ Selected Game / Save Context · 最終更新                                     │,└────────────────────────────────────────────────────────────────────────────┘,```,,### 31.23.2 Region Annotations,,| ID | Region | Visual / UX rule | Safety meaning |,|---|---|---|---|,| S1 | Save Context Header | Game + Save targetを明示 | Prevent wrong-target operations |,| S2 | Current Save | Current dataをBackup Generationから分離 | Current state ≠ historical backup |,| S3 | Backup Summary | 世代数・最新世代などを要約 | Existence ≠ recoverability |,| S4 | Generation Card | Exists / Integrity / Restore Eligibilityを別表示 | Prevent single green 'safe' inference |,| S5 | Restore Review action | Review entryであることを明示 | Selection ≠ Execute |,| S6 | Storage | Usage / free space / destinationを簡潔に提示 | Capacity ≠ backup health |,| S7 | History | 重要な世代へ迅速に到達 | History selection has no write side effect |,| S8 | Critical Recovery notice | Incomplete restore等を通常Historyへ埋没させない | Critical state remains actionable |,,### 31.23.3 Generation / Action State Mapping,,- 世代選択はRead-only Contextであり、Restore approvalやExecuteを意味しない。,- `Exists` は世代が存在することのみを示す。,- `Integrity` は適用されるIntegrity / Validation結果を示す。,- `Restore Eligibility` は現在の製品状態でRestore Reviewへ進める条件を満たすことを示すだけであり、ユーザー承認済みを意味しない。,- `復元をレビュー` はReview入口であり、実データの置換を開始しない。,- `今すぐバックアップ` は既存Backup operationが利用可能な場合だけActionableとする。,- BackupのProgress 100%はVerified Successを自動的に意味しない。,- Smart Cleanup / Delete等はBackup Primary Actionと別グループに置き、誤操作を避ける。,,### 31.23.4 Restore Review / Recovery Boundary,,```text,Save Data,   ↓,Game A / Generation 2026-09-28 23:10,   ↓,Restore Review,   ├ Current Save,   ├ Proposed Result,   ├ Impact,   ├ RescueSnapshot / preservation status,   └ Failure / Recovery path,   ↓,Confirm when required,   ↓,Execute,   ↓,Observe / Reconcile,   ↓,Verify,   ↓,Result,```,,Review画面から離れた場合は、旧Approvalを自動再利用しない。対象世代、現在データ、必要なIntegrity / Fingerprint条件が変化した場合は再Review / 再Approveが必要になる。,,### 31.23.5 Responsive / Accessibility Rules,,- WideではGame / Generationを2領域で表示し、Compactでは一覧と詳細を1列へ再配置できる。,- Game Name、Current Save、Generation timestamp、Restore Review、Backを文字拡大で失わない。,- Exists / Integrity / Restore Eligibilityは色だけでなくText + Iconで区別する。,- `復元をレビュー` と `今すぐバックアップ` を近接しすぎた同一Primary action群に置かない。,- Storage詳細が画面下へ押し出されても、現在のSave Contextと主要Operation入口を優先する。,- Keyboard focusはGame selection → Current Save → Backup → Generation → Restore Reviewの順で理解できる構造を基本とする。,,### 31.23.6 Wireframe Boundary,,このAnnotated WireframeはSave Dataの情報・視覚構造を定義するものであり、Backup、Restore、Recovery、Storage、Export等の新しいRuntime FeatureやPublic Contractを追加するものではない。Wireframe上のActionは、適用対象の既存Capabilityが成立した場合にのみActionableとする。,## 31.23 Save Data Acceptance Criteria

- Current SaveとBackup Generationが明確に区別される。
- GenerationのExists / Integrity / Restore Eligibilityが独立して理解できる。
- 最新世代、履歴、Pin / Tag、整合性状態の意味が理解できる。
- 世代を選択しただけでRestoreが開始されない。
- Restoreは必ず `Review → Confirm → Execute → Verify` に接続される。
- Reviewで「何が変わるか」「復元前に何を保全するか」「失敗時にどう扱われるか」を理解できる。
- RescueSnapshotの存在を、誤解を招かない表現でユーザーへ説明できる。
- 起動時に未完了Restoreがある場合、通常操作より先に必要なRecovery情報が提示される。
- Criticalな復旧失敗が通常通知やゲーム中DNDに埋没しない。
- 一括バックアップでは対象と概算負荷を事前に把握できる。
- Smart Cleanupなどの削除操作がバックアップ作成と混同されない。
- 外部ExportでGST互換性と暗号化payloadの制約を誤認させない。
- 未インストールゲームのバックアップをGST異常と誤表示しない。
- Back / Search / FilterでSave Data Contextを可能な範囲で保持する。
- 画面サイズや文字サイズ変更で主要なBackup / Review / Back操作が消えない。
- UI概念の追加だけを理由に、新しいRuntime Feature、Public Contract、Domain status、Storage behaviorを導入しない。

---

## Extraction Note

Source text was extracted mechanically from the parent UI working draft. Any future semantic change must be reviewed as a design change; an extracted statement must not be treated as a newly approved runtime requirement merely because it now has a dedicated file.

End of Document
