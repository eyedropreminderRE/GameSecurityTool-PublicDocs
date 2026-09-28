# 01-14: UI Recovery Design

**Document ID:** GST-ARCH-UI-RECOVERY-001  
**Version:** 0.4 (Recovery Annotated Wireframe / Visual UX Review)  
**Status:** Design Working Draft — Not an independent feature authority  
**Authority Relationship:** Subordinate to GST-BASELINE-011, GST-BASELINE-047, applicable approved module / feature specifications, and Change Control decisions  
**Parent Concept:** [01-08_UI_Information_Architecture_and_Interaction_Concept.md](01-08_UI_Information_Architecture_and_Interaction_Concept.md)  
**Historical Source Mapping:** Former Section 33 of the parent 01-08 UI working draft  
**Current Ownership:** This document is the maintained location for the Recovery material after the working-draft split.
**Purpose:** Organize existing UI information-architecture and interaction design for this scope without expanding Runtime Features, Public Contracts, Domain Status, Security capabilities, or Implementation Readiness.

## Document Boundary

This document preserves UI design material that was formerly organized as Section 33 in the parent 01-08 UI working draft. After the working-draft split, 01-08 retains global IA / interaction principles while this document owns the Recovery material.

The preserved section numbering is historical traceability only. It does not imply that those sections still exist in 01-08.

This document does not authorize Runtime implementation by itself. Existing formal baselines, approved module / feature specifications, Public Contracts, Security boundaries, and Change Control decisions remain authoritative where scope overlaps.

## Related Authoritative Documents

- [09_PC_Migration_and_Recovery_Model.md](../00_Baseline/GameProtection/09_PC_Migration_and_Recovery_Model.md) — Recovery / migration baseline
- [03-04_QUAR_Quarantine_and_Restore.md](../modules/03-04_QUAR_Quarantine_and_Restore.md) — Quarantine / Restore authority
- [03-01_SYS_System_Safety_and_Lifecycle.md](../modules/03-01_SYS_System_Safety_and_Lifecycle.md) — Recovery Host / system recovery authority
- [47_User_Experience_and_Interface_Design_Governance_Model.md](../00_Baseline/Governance/47_User_Experience_and_Interface_Design_Governance_Model.md) — Formal UX governance

---

# 33. Recovery UX Design

Recoveryは「エラーを一覧する画面」ではなく、**現在の問題を安全に理解し、復旧可能な経路を選び、結果を確認するための専用コンテキスト**とする。

基本原則:

    Inspect → Decide → Confirm → Recover → Verify

通常のRecoveryと、通常GSTに依存できない場合のRecovery Hostを明確に区別する。破壊的・外部状態変更を伴う復旧操作は、確認した対象と影響範囲から逸脱して実行しない。

## 33.1 Recovery Mental Model

ユーザーが最初に理解する対象を、内部ExceptionやTransaction statusではなく次の5項目に整理する。

    Problem         ← 何が起きたか
    Current State   ← 今のデータ / 保護状態はどうなっているか
    Recovery Source ← どこから戻せるか
    Action          ← 何を行うか
    Result          ← 実行後にどうなったか

「問題を検出した」と「復旧方法が確定した」を同一視しない。また、復旧元が存在することと、現在の状態へ安全に適用できることを同一視しない。

## 33.2 Recovery Entry and Priority

Recoveryへの入口は複数あってよいが、到達後は共通のRecovery Contextへ統合する。

主な入口:

- HomeのQuick Recovery
- Save DataからのRestore Review
- Protection上の重大状態からの復旧導線
- 起動時に検出された未完了Restore
- WIPERのインシデント結果からの継続操作
- 通常GST起動不能時のRecovery Host

入口が異なっても、同じ復旧情報を重複表示して判断を分断させない。

Recoveryでは既存のRecovery Priorityに従い、最初に**現行データの追加破壊を防ぐ**ことを優先する。

## 33.3 Recovery Screen Primary Structure

通常GSTのRecovery画面は次の5領域を基本とする。

1. Current Incident / Problem Summary
2. Current State
3. Available Recovery Path
4. Recovery Progress / Result
5. Evidence / Advanced Details

概念配置:

    ┌────────────────────────────────────────────────────────────────────┐
    │ ← 復旧                                                            │
    ├────────────────────────────────────────────────────────────────────┤
    │ ⚠️ 現在の問題                                                     │
    │ 前回の復元処理が完了していません                                  │
    │                                                                     │
    │ 現在の状態                                                         │
    │ セーブデータ: 確認中                                               │
    │ 復元トランザクション: 未完了                                      │
    │                                                                     │
    │ 推奨される確認                                                     │
    │ [ 復旧状態を確認 ]                                                 │
    │                                                                     │
    │ 復旧経路                                                           │
    │ [ 復旧をレビュー ]      [ Recovery Host を確認 ]                  │
    │                                                                     │
    │ 結果 / 検証                                                        │
    │ 復旧処理: —        結果確認: —                                    │
    └────────────────────────────────────────────────────────────────────┘

具体的な復旧アクションは、そのビルドで実装・検証済みのものだけを表示する。

## 33.4 Problem / Cause / Recommended Action

Recoveryトップでは、既存UXガバナンスの形式に従い、問題・原因または既知の文脈・推奨行動を近接させる。

基本形:

    Problem
    Cause / Known Context
    Recommended Action

原因が確定していない場合は「原因」と断定せず、確認できた事実と未確認事項を分離する。

技術的な失敗コードだけを表示して、ユーザーに次の行動を推測させる構成は避ける。

## 33.5 Quick Recovery

HomeなどからのQuick Recoveryは、**一クリックで実データを書き換える機能にはしない**。

推奨:

    Home
      ↓
    Quick Recovery
      ↓
    対象・現在状態・復旧内容をInspect
      ↓
    User Confirm
      ↓
    Recover
      ↓
    Verify

「直前の状態へ戻す」という短い表現だけで対象や影響を隠さない。

## 33.6 Save Restore Recovery

Save Dataから開始したセーブ復元では、選択した世代と現在データの関係をRecovery Contextへ引き継ぐ。

最低限:

- 対象Game / Save
- 選択したBackup Generation
- 現在のセーブの扱い
- 復元前保全（RescueSnapshot）
- 復元結果の確認方法

を維持する。

Manifest認証、Containment検証などの技術的検証が成功したことだけで、ユーザー承認済みとは扱わない。

実際の置換は既存の `Review → Confirm → Execute` を維持し、完了後に `Verify` を行う。

## 33.7 RescueSnapshot and Rollback

Restore失敗時は、「失敗した」という一点だけでなく、現行データがどの状態にあるかを最優先で説明する。

概念:

    復元前データの保全
        ↓
    復元処理
        ↓
    成功 → Verify
        │
        └→ 失敗 → RescueSnapshotからの復旧を実施 / 試行

UIでは可能な範囲で、退避の確認、Rollback実施結果、現在データへの影響、次に確認すべき事項を分離する。

「必ず元通り」「100%データ損失なし」などの新しい保証文言はUIで生成しない。

## 33.8 Incomplete Restore / Startup Recovery

起動時に未完了Restoreが検出された場合、通常のGames / Save Data操作より先にRecovery状態を提示する。

概念:

    GST Startup
       ↓
    未完了Restoreを検出
       ↓
    Startup Recovery / self-healing
       ↓
    状態をVerify
       ↓
    正常化 → 結果を通知 / 履歴化
       ↓
    未解決 → Critical Recovery状態

通常の追加Restoreを許可する前に、既存の未完了状態を解消・確認する。

自己修復が成功した場合も、必要な範囲で「前回何が起き、現在どうなっているか」を確認できるようにする。

## 33.9 WIPER Incident Recovery

WIPERインシデントでは、検知シグナルと攻撃元の断定をUI上で混同しない。

表示例:

    ⚠️ 保護セッションで異常なファイル変更を検知
    観測: Canary / Burst / Child-process screening など
    現在: 対象ゲームの保護セッションは停止 / 終了状態
    [インシデント詳細]

ファイルイベントだけから攻撃元プロセスを証明できない既存境界を維持する。

既存のWIPER Suspend / Terminate結果に応じ、ユーザーが実際に利用可能な継続操作だけを提示する。UIからInfrastructure APIを直接呼び出さず、既存Application境界のCommandへ接続する。

## 33.10 Recovery Host

通常GSTが起動できない場合や通常UIに依存できない場合、独立した `GameSecurityTool.Recovery.exe` Recovery Hostを別の復旧面として扱う。

ユーザーには、通常GSTとは異なる環境であることを明示する。

基本説明:

    独立復旧ホスト
    通常GSTのMainWindow / SQLite DB / 通常設定ストアに依存しない
    起動しただけではシステム状態を変更しない

Recovery Hostで外部状態変更を行う場合は、既存Recovery Architectureの

    Inspect → User Confirm → Recover → Verify

を必須とする。

現在のビルドで特定のRecovery Host actionが未実装または未検証である場合、UI上で利用可能なアクションとして表示しない。

Recovery Host自身の検証に失敗した場合は、別の可変設定や推測した復旧ポリシーへフォールバックするようなUIを設けない。

## 33.11 Recovery Host Action Presentation

承認済みRecovery Host actionについては、目的と影響を先に示す。

Recovery Host actionのUIには、**Recovery Hostの正式なRecovery仕様で明示的に承認された操作だけ**を表示する。

UI Design Working Draftから、Recovery Hostに新しい操作一覧を推測してはならない。

各操作は、少なくとも:

- 目的
- 対象範囲
- 現在状態
- 変更される可能性があるOS / GST状態
- Review / User Confirm
- Recover後のVerify

を既存のRecovery契約に従って表示する。

未実装・未検証のRecovery Host actionはボタンとして提示せず、必要な場合は「利用可能な復旧手段がありません」等の事実ベースの状態へ変換する。


## 33.12 Manual Recovery / No Safe Automatic Path

自動復旧できない場合、Recovery画面は「失敗」だけで終わらせない。

表示する情報:

- 確認できた事実
- 復旧できない理由または未確認事項
- 現在のデータをこれ以上変更しないための注意
- 利用可能な別の復旧元
- ユーザーが手動で行う必要がある手順

帰属不明のOS設定やユーザーデータを、原因不明のままGSTが自動上書きする導線を作らない。

## 33.13 Recovery Progress

長時間のRecoveryではProgress Surfaceを使用し、現在の段階を説明する。

概念:

    Inspecting
      ↓
    Preparing Recovery
      ↓
    Recovering
      ↓
    Verifying
      ↓
    Completed / Needs Attention / Failed

内部Status enumをそのままユーザーへ公開せず、意味のある日本語文言へ変換する。

キャンセル可能な処理については、キャンセルによって何が起きるかを明示する。実データ置換の途中で安全に中断できない場合、単純なCancelボタンを表示して誤解を招かない。

## 33.14 Recovery Result and Verification

Recover完了後は「処理が終わった」だけではなく、結果の検証状態を提示する。

例:

    復旧処理: 完了
    結果確認: 完了
    現在の状態: 確認済み

検証できなかった場合は成功表示へ丸めず、

    復旧処理: 完了
    結果確認: 未完了
    [確認できなかった内容を見る]

のように分離する。

データ復旧では、指定ディレクトリへの存在、サイズ、ハッシュ、必要に応じたロード可否など、既存Recovery仕様で定義された確認結果を追跡できる構造とする。

## 33.15 Critical Recovery State

Recovery FailureやRecovery Hostの検証失敗など、データ保護へ重大な影響がある状態はCriticalとして扱う。

Critical表示では:

- ゲーム中DNDでも完全に隠さない
- 未確認状態を保持する
- 対象と現在状態を明示する
- 次に必要な確認を一つに絞る
- 危険な複数操作を同時にPrimary Actionへしない

を基本とする。

原因を推測で断定せず、確認済みの事実と未確認事項を分ける。

## 33.16 Evidence / Advanced Details

通常ユーザーには、目的・状態・影響・次の操作を先に示す。

Advanced / Expertでは必要に応じて:

- Transaction / Incident identifier
- 詳細ログ
- Manifest / integrity verification結果
- Recovery Host validation結果
- 技術的エラー情報

などを確認できるようにする。

詳細情報を開く操作だけでRecovery actionを開始しない。

## 33.17 Recovery Navigation and Context

Recoveryでは、元画面の文脈を可能な範囲で保持する。

例:

    Save Data
      → Game A
        → 世代 2026-09-28 23:10
          → Restore Review
            → Recovery

Recovery中に対象を別ゲームへ切り替える、別のRestoreを開始する等の混線を避ける。

Backは前の確認面へ戻るための操作とし、実行済みRecoverを逆に戻すUndoボタンとして扱わない。

実行後に元画面へ戻る場合も、結果・Verification状態・未解決事項を見失わせない。

## 33.17.1 Annotated Wireframe — Recovery / Visual UX Working Baseline

このAnnotated Wireframeは、Recoveryの既存設計を視覚構造へ落とし込む作業基準である。Recovery HostやRollback等の具体的CapabilityをこのWireframeから新設・承認しない。

```text
┌────────────────────────────────────────────────────────────────────────────┐
│ GST / Recovery Context                                      復旧              │
├────────────────────────────────────────────────────────────────────────────┤
│ ⚠️ 現在の問題                                                           │
│ 前回の復元処理が完了していません                                         │
│                                                                           │
│ 対象: Game A / Save Data                                                 │
│ 現在の状態: 確認中                                                       │
│ 未完了操作: Restore                                                      │
│                                                                           │
│ ┌──────────────────────────────────────────────────────────────────────┐ │
│ │ まず現在の状態を確認してください                                    │ │
│ │ 確認済みの事実: —                                                    │ │
│ │ 未確認事項: —                                                        │ │
│ │                                                                      │ │
│ │ [復旧状態を確認]                                                     │ │
│ └──────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│ 利用可能な復旧経路                                                       │
│ [復旧をレビュー]                         [Recovery Hostを確認]           │
│ ※ 実装・検証済みの既存Recovery actionのみActionable                    │
│                                                                           │
│ 復旧結果 / 検証                                                          │
│ 復旧処理: —        結果確認: —        未解決事項: —                     │
│                                                                           │
│ 詳細 / 証拠                                                               │
│ [技術情報を表示]                                                        │
└────────────────────────────────────────────────────────────────────────────┘
```

### 33.17.1.1 Region Annotations

| ID | Region | Visual / UX rule | Safety meaning |
|---|---|---|---|
| R1 | Incident Summary | Problemを最上位に表示し、対象Scopeを併記 | 問題と対象を取り違えない |
| R2 | Current State | Current Stateと未確認事項を分離 | 不確実な状態を正常扱いしない |
| R3 | First Safe Action | Inspect / 状態確認を先に配置 | 状況不明のまま変更操作へ進ませない |
| R4 | Recovery Path | 利用可能な経路だけActionableにする | UIが新しいRecovery capabilityを発明しない |
| R5 | Result / Verification | Recover結果とVerify結果を別表示 | 完了と確認済みを同一視しない |
| R6 | Critical Surface | Criticalは画面内に保持し通常Toastだけに依存しない | DismissでUnderlying stateを消さない |
| R7 | Evidence | 技術情報は段階開示 | 詳細確認と副作用を分離する |
| R8 | Context Trail | 元画面 / Game / Generation等を可能な範囲で保持 | 別対象への混線を防止する |

### 33.17.1.2 Core State Model

Recoveryの主要状態は、少なくとも次の視覚的な意味を分離する。

```text
Problem detected
      ↓
Inspecting / Current State uncertain
      ↓
Recovery path known
      ↓
Review
      ↓
User Confirm
      ↓
Recovering
      ↓
Verifying
      ↓
Completed + Verified
   ├─ Needs Attention
   └─ Failed / Recovery unresolved
```

- `Detected` は、復旧が必要だとユーザーが判断済みであることを意味しない。
- `Recovery Source exists` は、現在状態へ安全に適用できることを意味しない。
- `Completed` はOperation結果であり、適用される仕様が別Verificationを要求する場合は`Verified`と分離する。
- `Dismissed` / `Acknowledged` はUnderlying ProblemをResolvedへ変更しない。
- `Needs Attention` は成功に丸めず、残っている確認事項または影響範囲を示す。

### 33.17.1.3 State Variants

**Normal Recovery Entry / Review Ready**

- Problem、対象、Current State、Recovery Source、Impactを確認できる。
- 実行前はReview / Confirm入口だけをActionableにし、Recover Executeを直接置かない。

**Incomplete Restore / Startup Recovery**

- 通常のSave Data / Games操作より先に未完了操作を示す。
- Current Stateが確認中の場合、成功状態や通常のRestore入口を先に表示しない。
- 自動復旧が完了した場合も、必要な範囲でResultとVerificationを確認できる。

**Recovery Host**

- 通常GSTとは異なるRecovery Contextであることを明示する。
- 未実装 / 未検証 / 現在利用不能なRecovery Host actionはボタンとして表示しない。
- Recovery Hostの検証失敗から、UIが別の復旧ポリシーや可変設定を推測してはならない。

**WIPER Incident**

- 観測シグナルと攻撃元の断定を分ける。
- 現在確認できるRuntime / File stateを優先表示する。
- 既存Application境界が提供する継続操作のみをActionableとする。

**No Safe Automatic Recovery**

- 「復旧不可」だけで終わらせず、確認済みの事実、未確認事項、現在状態、手動対応の必要性を分けて表示する。
- 帰属不明のOS設定やユーザーデータを、推測した原因に基づいて自動変更する導線を作らない。

### 33.17.1.4 Recovery Action Mapping

| UI Action | Meaning | Must not imply |
|---|---|---|
| 復旧状態を確認 | Read-only inspection / reconcile準備 | Recovery Execute |
| 復旧をレビュー | Review surfaceへのentry | User approval / immediate recovery |
| Recovery Hostを確認 | Recovery Host contextへのnavigation / inspection | 未定義Actionの存在 |
| 詳細 / 証拠 | Read-only evidence | 状態変更 |
| Back | 前のContextへ戻る | 実行済みRecoverのUndo |
| Cancel | 未実行のPending operationを中止 | 既に完了した変更の巻き戻し |

高影響Recovery operationは、対象Contractで要求される`Inspect → Review → Confirm → Recover → Verify`へ接続する。実行後に元画面へ戻る場合も、Result / Verification / unresolved stateを失わせない。

### 33.17.1.5 Responsive / Accessibility Rules

- WideではIncident Summary → Current State → Recovery Path → Result / Verification → Evidenceの順を維持し、Compactでは1列へ再配置できる。
- 文字拡大時にもProblem、Target, Current State、Primary safe action、Backを優先して残す。
- Critical / Needs Attention / Unavailable / ErrorはColorだけでなくText + Icon + Groupingで区別する。
- Recovery Actionが横幅不足で折り返されても、Target / Impact / Confirm meaningを削除しない。
- Keyboard focusはProblem → Current State → Inspect / Review → Result / Evidenceの順で理解しやすい構造を基本とする。

### 33.17.1.6 Wireframe Boundary

このAnnotated WireframeはRecoveryの既存情報・操作境界を視覚化するものであり、新しいRecovery capability、Public Contract、Domain Status、Recovery Host action、外部変更権限、Rollback保証を追加しない。

Actionable presentation is capability-gated: UI上で利用可能に見える操作は、適用対象の正式なFeature / Module / Contractと、必要な実装・検証条件が成立している場合に限る。

## 33.19 Review Findings — 2026-09-29

The focused review identified one governance risk and clarified it:

| Finding | Resolution |
|---|---|
| Concrete Recovery Host operations listed in the UI draft could be mistaken for approved Recovery capabilities | Replaced the illustrative action list with an authority-bound presentation rule. Only formally approved, actually available, and appropriately verified Recovery Host actions may be surfaced. |

This clarification does not add, remove, or authorize any Recovery Host capability.

## 33.18 Recovery Acceptance Criteria

- Recovery画面を開いた時点で「何を復旧する画面か」が理解できる。
- Problem / Current State / Recovery Source / Action / Resultが区別される。
- Quick Recoveryが実データの即時上書きに直結しない。
- Save Restoreは `Review → Confirm → Execute → Verify` を維持する。
- RescueSnapshotについて、保全・Rollback・結果を誤解なく説明できる。
- 未完了Restoreが存在する場合、通常の追加Restoreより先にRecovery状態を確認できる。
- WIPERの観測シグナルと攻撃元の断定をUI上で混同しない。
- WIPERの継続操作が既存のApplication境界へ接続される。
- Recovery Hostが通常GSTとは独立した復旧面であることを理解できる。
- Recovery Hostの未実装 / 未検証機能を利用可能として表示しない。
- Recovery Hostの外部状態変更が `Inspect → User Confirm → Recover → Verify` を経る。
- 自動復旧できない場合でも、現在状態と手動復旧の次の手順を確認できる。
- 長時間Recoveryでは現在Stepと結果が表示される。
- Recovery結果とVerification結果を分離する。
- Critical Recovery状態が通常ToastやSmart DNDに埋没しない。
- 技術詳細を開いても副作用が発生しない。
- BackでRecovery Contextを壊さず、実行済みRecoverをUndoと誤認させない。
- Recovery UX追加だけを理由に、新しいRecovery capability、Public Contract、Domain status、外部変更権限を導入しない。

---

## Extraction Note

Source text was extracted mechanically from the parent UI working draft. Any future semantic change must be reviewed as a design change; an extracted statement must not be treated as a newly approved runtime requirement merely because it now has a dedicated file.

End of Document
