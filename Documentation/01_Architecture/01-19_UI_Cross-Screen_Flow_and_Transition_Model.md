# 01-19: UI Cross-Screen Flow and Transition Model

**Document ID:** GST-ARCH-UI-CROSSSCREEN-001  
**Version:** 0.3 (Annotated Cross-Screen Flow / Transition UX Review)  
**Status:** Design Working Draft — Not an independent feature authority  
**Authority Relationship:** Subordinate to GST-BASELINE-011, GST-BASELINE-047, applicable approved module / feature specifications, and Change Control decisions  
**Parent Concept:** [01-08_UI_Information_Architecture_and_Interaction_Concept.md](01-08_UI_Information_Architecture_and_Interaction_Concept.md)  
**Historical Source Mapping:** Former Section 35 + Section 36.13 + Section 36.14 of the parent 01-08 UI working draft  
**Current Ownership:** This document is the maintained location for the Cross-Screen Flow / Transition material after the working-draft split.
**Purpose:** Organize existing UI information-architecture and interaction design for this scope without expanding Runtime Features, Public Contracts, Domain Status, Security capabilities, or Implementation Readiness.

## Document Boundary

This document preserves the UI design material that was formerly organized as Section 35 and the cross-scenario/mapping material under Section 36.13 and 36.14 of the parent 01-08 UI working draft. After the working-draft split, 01-08 retains global IA / interaction principles while this document owns the cross-screen flow and transition material.

The preserved section numbering is historical traceability only. It does not imply that these sections still exist in 01-08.

This document does not authorize Runtime implementation by itself. Existing formal baselines, approved module / feature specifications, Public Contracts, Security boundaries, and Change Control decisions remain authoritative where scope overlaps.

## Related Authoritative Documents

- [11_UI_UX_and_User_Interaction_Model.md](../00_Baseline/Governance/11_UI_UX_and_User_Interaction_Model.md) — Formal UI/UX baseline
- [47_User_Experience_and_Interface_Design_Governance_Model.md](../00_Baseline/Governance/47_User_Experience_and_Interface_Design_Governance_Model.md) — Formal UX governance
- [01-06_Presentation_and_UI_Architecture.md](01-06_Presentation_and_UI_Architecture.md) — Presentation / Timeline authority
- [03-03_FW_Zero_Trust_Firewall.md](../modules/03-03_FW_Zero_Trust_Firewall.md) — Firewall authority
- [01-16_UI_Common_Interaction_and_Display_Design.md](01-16_UI_Common_Interaction_and_Display_Design.md) — Common interaction / display rules
- [01-17_UI_Reference_Patterns_and_Common_Component_Library.md](01-17_UI_Reference_Patterns_and_Common_Component_Library.md) — Common patterns / component boundaries
- [01-18_UI_User_Journeys_and_Review_Model.md](01-18_UI_User_Journeys_and_Review_Model.md) — Journey gates / safe exit model
- [01-20_UI_End-to-End_Scenario_Validation.md](01-20_UI_End-to-End_Scenario_Validation.md) — End-to-end scenario validation

---

# 35. Cross-Screen User Flow and Transition Model

6つの主要画面が個別に整っていても、画面間の遷移で「どこへ進んだのか」「何を変更したのか」「確認が必要なのか」が失われると、GST全体の安全性と使いやすさが低下する。

そのため、画面横断では次の共通モデルを採用する。

    Discover → Understand → Review → Confirm → Execute → Verify → Return

ただし、すべての操作が全ステップを必要とするわけではない。読み取り専用操作と低影響設定変更まで過剰な確認を要求せず、**操作の影響に応じて必要なゲートだけを適用する**。

## 35.1 Global Navigation Model

主要ナビゲーション:

    Home
    Games
    Save Data
    Protection
    Recovery
    Settings

各画面には1つの主目的を持たせ、別画面の責務を大きく取り込まない。

これはUI Working Draft上のNavigation Modelであり、Public Contractの追加・変更を意味しない。

例:

    Games      → ゲームを選ぶ / 起動する / ゲームContextを見る
    Save Data  → セーブ世代・バックアップを扱う
    Protection → 保護状態・影響を理解する
    Recovery   → 問題から安全に戻る
    Settings   → 構成を変更する

クロスリンクは許可するが、遷移先で新しい責務を二重表示して「どちらの画面を操作しているのか」を曖昧にしない。

## 35.2 Context Preservation

画面間を移動した場合、可能な範囲で以下を保持する。

- Selected Game
- Selected Backup Generation
- Search Query
- Filter
- Category
- Scroll Position（安全に復元でき、表示内容の意味が変わっていない場合のみ）
- Review対象
- Origin Page

ただし、危険操作中に状態を引き継ぎすぎて、別の対象へ誤って操作できる状態は作らない。
対象が外部で変更・削除された可能性がある場合は、保存されたContextをそのまま操作対象として再利用せず、現在状態を再確認する。

RestoreやRecoveryのような高影響Contextでは、対象Game / Generation / Recovery Incidentを固定表示し、Operation対象のidentityを再確認できる構造とする。

## 35.3 Context Header

詳細画面では、現在の対象をヘッダーまたはBreadcrumb付近に明示する。

例:

    セーブデータ
      → Game A
        → 世代 2026-09-28 23:10
          → 復元レビュー

Protectionの場合:

    保護
      → Game A
        → ネットワーク保護

Settingsの場合:

    設定
      → ゲーム保護
        → Smart Game IME Lock

Contextを表示することは、内部IDやRepository情報を通常UIへ露出することを意味しない。

## 35.4 Standard Read-Only Flow

読み取り専用操作では、Review / Confirm / Executeを追加しない。

例:

    Home → Protection
    Games → Game details
    Save Data → Backup details
    Settings → Setting details

主な操作:

    Select → View → Back

Refreshもこの読み取り専用系に含め、状態変更を開始しない。

## 35.5 Everyday Low-Risk Flow

日常的で低影響の操作は短い導線を維持する。

例:

    Games
      → Game A
        → Launch

    Games
      → Game A
        → Save Data
          → Backup

    Home
      → Status detail
        → Back

頻度の高い操作のために、安全確認を削除しない。単に「確認不要」とするのではなく、既存仕様で確認が要求されない操作だけを短縮する。

## 35.6 Material / High-Risk Flow

ユーザーのファイル、Windows設定、Firewall、設定全体などに実質的な変更を与える操作は、**適用対象のFeature / Module / Contractが定義する影響分類と確認要件に従って**共通の安全ゲートへ接続する。UI側で新しい危険度分類を独自生成しない。

基本:

    Discover
      ↓
    Review
      ↓
    Confirm
      ↓
    Execute
      ↓
    Verify
      ↓
    Result

代表例:

    Save Data → Restore
    Protection → Firewall change
    Settings → High-impact setting change
    Settings → Standard Profile Reset
    Settings → Factory Reset
    Recovery → External / destructive recovery

「詳細を開く」と「実行する」を同じボタンにしない。

## 35.7 Back / Cancel / Close Distinction

3つを同じ意味として扱わない。

### Back

前の画面・前のContextへ戻る。

    Back = navigation

Backで戻すContextは、現在も有効な対象だけを再利用する。外部状態の変化が疑われる場合は、一覧・対象・Review状態を再確認してから操作可能とする。

### Cancel

現在進行中のReview / Dialog / 入力を取り消す。

    Cancel = abandon current pending operation

「開始前の状態へ戻す」は、まだExecuteされていないPending Operationについての意味とする。すでにExecuteされた変更をCancelだけでUndoしたことにはしない。Undo / Rollbackが必要な場合は、その操作の既存Recovery契約を使用する。

### Close

画面やDialogを閉じる操作。

Closeしても、すでにConfirm済みの操作や実行済みのRecoverをUndoする意味にはしない。

特に高影響操作では、ユーザーが「閉じたから変更されていない」と理解できるよう、Execute前後の状態を明確にする。

## 35.8 Pending Review State

Review画面を開いただけでは、変更を予約・適用しない。

ユーザーがReviewから離れた場合:

    Review pending
        ↓
    leave / close
        ↓
    no Execute

再度同じ操作を開いた場合は、旧ReviewのApprovalを自動再利用せず、対象identity、現在状態、操作内容、および必要なFingerprint / Integrity条件を再評価する構造とする。

特にAI送信やRestoreなど、内容のFingerprint / Integrityに依存する承認は、入力・対象・関連状態のいずれかが変化した場合、旧Approvalを失効させて再Review / 再Approveを要求する。

Review状態そのものは、画面を再表示したことだけを根拠にApprovedへ戻してはならない。

## 35.9 Execute-to-Verify Boundary

Executeが成功したことを、そのまま最終成功表示にしない。

基本:

    Execute / Attempt
      ↓
    Observe / Reconcile
      ↓
    Verify
      ↓
    Verified / Partially Verified / Failed / Needs Attention

Execute / Attemptの表現は、実際のOperation Contractが区別を提供する場合のみUI上でも区別する。Contractが定義する以上に「完了」を強く表現しない。

Windows外部状態へ変更する操作では、必要に応じて実状態の確認結果をユーザーへ提示する。

「要求を送信しました」と「実際に状態が変わったことを確認しました」を区別する。

## 35.10 Error-to-Recovery Routing

エラー発生時に、すべてをGeneric Error画面へ送らない。

分類:

    Recoverable in current context
      → 現在画面で原因 / 次の操作を表示

    Requires dedicated Recovery
      → Recoveryへ遷移

    Requires independent Recovery Host
      → Recovery Hostへの導線

    No safe automatic recovery
      → 現在状態を保全し、手動復旧手順を表示

画面遷移先はUI側の独自推測ではなく、適用対象のOperation / Featureの権威的な結果分類・Recovery guidanceに基づいて決定する。エラーコードは補助情報として扱い、コード単独から新しいRecovery capabilityや状態をUI側で推測しない。

## 35.11 Home-to-Recovery Flow

HomeからのQuick Recoveryは、最短でもReviewを省略しない。

    Home
      ↓
    Quick Recovery
      ↓
    Inspect
      ↓
    Review
      ↓
    Confirm
      ↓
    Recover
      ↓
    Verify
      ↓
    Home / Recovery Result

実データを変更する復旧を、Home上のワンクリックで完了させない。

## 35.12 Save Data-to-Recovery Flow

セーブ復元では、Save Dataの「世代選択」とRecoveryの「復旧実行」を分ける。

    Save Data
      ↓
    Game A
      ↓
    Generation
      ↓
    Restore Review
      ↓
    Recovery
      ↓
    Confirm
      ↓
    Execute
      ↓
    Verify
      ↓
    Result

実行後にSave Dataへ戻る場合は、選択していたGenerationと結果を保持し、実行前の一覧へ静かに戻して「何が起きたか」を失わせない。

## 35.13 Protection-to-Settings Flow

Protectionから設定変更へ遷移する場合、現在の状態と変更対象を引き継ぐ。

    Protection
      ↓
    Problem / Feature detail
      ↓
    Settings
      ↓
    Current Value / New Value / Scope / Impact
      ↓
    Review → Confirm → Execute → Verify

Settingsへ移動しただけで対象設定を変更しない。

変更後にProtectionへ戻る場合は、変更結果とVerification状態を表示し、単に「設定画面へ戻った」だけにしない。

## 35.14 Games-to-Save Data Flow

GamesからSave Dataへ移る場合はGame Contextを維持する。

    Games
      → Game A
        → Save Data
          → Game A selected

別ゲームのデータを誤って扱わないよう、Save Data側でもSelected Gameを再確認できる状態とする。

BackでGamesへ戻る場合は、Game Listの検索・Filter・選択位置を可能な範囲で保持する。

## 35.15 Games-to-Protection Flow

GamesからProtectionへ移る場合も、Game-specific stateとGST-wide stateを混同しない。

例:

    Games
      → Game A
        → Protection

表示:

    Game Aの保護状態
    +
    必要に応じたGST全体のProtection Context

Game Aが停止していることを、GST Protection failureと自動解釈しない。

## 35.16 Settings-to-Recovery Flow

設定変更によって問題が発生した場合は、Settingsへ留まり続けて原因不明の変更を繰り返すのではなく、必要に応じてRecoveryへ接続する。

    Settings
      ↓
    Apply / Verify
      ↓
    Problem detected
      ↓
    Recovery guidance

ただし、設定変更失敗だけを理由に自動Resetや自動Rollbackを追加しない。利用可能なRecovery pathが既存仕様にある場合のみ、その導線を提示する。


### 35.16.1 Runtime Protection Path State Reconciliation
Protection → Firewall / WER transitions retain the active-session Scope. Their displayed three-state value must come from the authoritative normalized path state defined by the applicable normalized path-state contract. The transition layer does not derive state from visible rule counts, toggle state, operation-request results, or generic SystemHealth. WER recovery failures after session end route through Recovery/Critical handling instead of being silently rewritten as the current Runtime Protection state.

## 35.17 Global Search Navigation Safety

Global Searchは、承認済みの検索能力が存在する場合に高速な入口として利用するが、検索結果から安全ゲートをバイパスしてはならない。Search UI自体が未承認のDeep Link capabilityを新設しない。

検索結果が:

    Information
    Setting
    Game
    Backup Generation
    Recovery item

のいずれであっても、選択しただけでは副作用を起こさない。

また、Expert設定へのDeep Linkがあっても、必要なReview / Confirmや権限確認を飛ばして実行可能状態にしない。

## 35.18 Notification-to-Page Flow

通知は、通知そのものを最終回答にせず、必要なContextへ接続できるようにする。

例:

    Backup failed
      → Save Data / Result

    Firewall blocked
      → Protection / Block detail

    Restore failed
      → Recovery / Incident

    Critical WIPER incident
      → Recovery / Incident detail

通知を閉じても未解決Critical状態そのものを消去したと扱わない。通知から詳細画面へ移動した場合は、外部状態が変化し得る対象について必要な再取得 / 再評価を行い、古い通知Contextだけを根拠にExecute可能状態へしない。

## 35.19 Cross-Screen State Vocabulary

画面を跨いでも同じ意味の状態は同じ言葉で表示する。

特に:

    Protection Status
      🟢 保護中
      🟡 一部制限あり
      🔴 保護停止

    Configuration Status
      標準設定
      カスタム設定

は共通語彙とする。

一方、Backupの存在、Restoreの結果、Recoveryの状態などは、それぞれの意味を持つため、Protection Statusへ無理に変換しない。
Requested / Executed / Verifiedも、必要な操作では共通の意味で扱うが、個別Contractが露出する情報以上の状態をUI側で生成しない。

## 35.20 Transition Anti-Patterns

以下を避ける。

- 同じ操作のために毎回Homeへ戻ってから再選択させる深いPogo-sticking
- Search結果から高影響操作を即時実行するDeep Link
- Dialogを閉じただけで実行済み操作をUndoしたように見せる表現
- Execute成功だけを最終成功として表示する
- 画面遷移でSelected Game / Generationを失い、別対象を誤操作させる
- Critical状態を通常Toastへ変換して終わる
- エラーコードだけを表示し、次の安全な行動を示さない
- 画面ごとに異なるBack / Confirm / Cancelの意味を与える

## 35.21 Cross-Screen Working Flow Map

全体の概念フロー:

    ┌──────────────┐
    │    Home      │
    └──────┬───────┘
           │
    ┌──────┼────────────┬─────────────┬────────────┬────────────┐
    ▼      ▼            ▼             ▼            ▼            ▼
  Games  Save Data  Protection    Recovery     Settings    Search/Help
    │       │           │             │            │
    │       │           │             │            │
    ├───────┼───────────┼─────────────┼────────────┤
    │       │           │             │            │
    │       └──Restore──┘             │            │
    │                  └──────────────▶│            │
    │                                  │            │
    └────Protection / Save────────────────────────▶│
                     │                             │
                     └────High-impact change──────┘
                                   │
                           Review → Confirm
                                   │
                                Execute
                                   │
                                 Verify
                                   │
                                Result
                                   │
                         Context-preserving Return

この図は画面遷移の概念モデルであり、個別機能の新規追加やRuntime capabilityの定義ではない。

## 35.21.1 Annotated Wireframe — Cross-Screen Flow / Transition Working Baseline

このAnnotated Wireframeは、画面間遷移に伴うContext、Review / Approval、Operation結果、Verification、Safe Returnの意味を視覚化する作業基準である。遷移モデルから新しいRuntime FeatureやContractを生成しない。

```text
┌────────────────────────────────────────────────────────────────────────────┐
│ Origin: Save Data                                                        │
│ Target: Game A / Generation 2026-09-28 23:10                             │
│ Operation: Restore                                                     │
├────────────────────────────────────────────────────────────────────────────┤
│                        NAVIGATION / CONTEXT                              │
│ Save Data → Game A → Generation → Restore Review                        │
│      │          │          │             │                               │
│      │          │          │             └─ Current / Proposed / Impact │
│      │          │          │                Confirmation state            │
│      │          │          └─ selected generation                         │
│      │          └─ selected game                                           │
│      └─ origin page                                                       │
├────────────────────────────────────────────────────────────────────────────┤
│ APPROVAL BOUNDARY                                                         │
│ Review → Confirm (when required) → Execute                                │
│                 ▲                                                         │
│                 └─ target / state / fingerprint must still match          │
├────────────────────────────────────────────────────────────────────────────┤
│ RESULT BOUNDARY                                                           │
│ Execute / Attempt → Observe / Reconcile → Verify → Result                │
│                                      │                                    │
│                                      ├─ Verified                          │
│                                      ├─ Needs Attention                   │
│                                      └─ Failed / Unresolved               │
├────────────────────────────────────────────────────────────────────────────┤
│ RETURN / RECOVERY                                                         │
│ Result + Verification + unresolved state                                │
│        ↓                                                                  │
│ Return to Origin / Recovery / appropriate destination                    │
│        │                                                                  │
│        └─ stale Context is revalidated before another material action    │
└────────────────────────────────────────────────────────────────────────────┘
```

### 35.21.1.1 Region Annotations

| ID | Region | Visual / UX rule | Safety meaning |
|---|---|---|---|
| X1 | Origin | Origin Pageを保持し、遷移先で現在位置を明示 | どこから来たかを失わせない |
| X2 | Context | Target / Scope / selected objectを遷移先でも再確認可能にする | 別Game / Generationへの誤操作を防止 |
| X3 | Review | Current / Proposed / Impact / Confirmation requirementを保持 | 遷移だけでApproval済みとしない |
| X4 | Approval freshness | identity / state / fingerprint等の再評価条件を保持 | 古いApprovalの誤再利用を防止 |
| X5 | Execute boundary | 高影響操作の入口をReview / Confirmと分離 | NavigationがExecuteを意味しない |
| X6 | Result | Requested / Executed / Verifiedを必要に応じて分離 | 実行完了をVerifiedと誤認させない |
| X7 | Recovery routing | Result / authoritative guidanceに応じて遷移 | UI独自推測でRecovery pathを生成しない |
| X8 | Safe Return | Result / Verification / unresolved stateを保持して戻る | 静かにOriginへ戻して重要結果を失わせない |

### 35.21.1.2 Transition Classes

**Read-only Navigation**

`Select → View → Back`

- Search、Contextual Link、Details、Refresh等は副作用なしで遷移する。
- 遷移しただけでTargetの状態やSecurity Statusを変更しない。

**Everyday Low-Risk Flow**

`Origin → Context → Existing Operation → Result → Return`

- 既存仕様でConfirm不要の操作は過剰に長い確認画面へ強制しない。
- ただしResultが重要な場合はRequested / Executed / Verifiedを適用可能な範囲で区別する。

**Material / High-Impact Cross-Screen Flow**

`Origin → Context → Review → Confirm → Execute → Observe / Reconcile → Verify → Result`

- Restore、Firewall変更、High-impact Settings、Recovery等は適用対象Contractを優先する。
- Deep Link、Search、Notification、Trayから入っても安全ゲートを迂回しない。

**Error / Recovery Flow**

`Origin → Failure / Incident → Current State → Authoritative Recovery Guidance → Recovery → Verify → Result`

- Error codeや画面遷移だけから新Recovery capabilityを推測しない。
- SafetyのためCurrent Stateが不明な場合は、先にInspect / Reconcileを行える構造を基本とする。

### 35.21.1.3 Context / Approval Invalidation

遷移先へ持ち越せるContextと、持ち越してはいけないApprovalを明確に分ける。

| Condition | Context handling | Approval handling |
|---|---|---|
| Same target / relevant state unchanged | Continue current Context | Reuse only where applicable authority permits |
| Target identity changed | Rebind / reselect | Invalidate |
| Current state changed externally | Reacquire / revalidate | Invalidate when decision basis changed |
| Review input changed | Update proposed state | Re-Review / re-approve |
| Fingerprint / integrity basis changed | Reacquire evidence | Invalidate |
| Notification is stale | Reopen current authoritative context | Do not Execute from stale notification alone |

`Back`で戻ったContextも、外部変更の可能性がある場合はそのままMaterial actionへ使用せず再評価する。

### 35.21.1.4 Notification / Search / Deep-Link Safety

- NotificationはDestination hintであり、notification itselfがExecution authorityではない。
- Search resultはInformation / Navigation entryとして扱い、High-impact operationへ直接Executeしない。
- Deep LinkでExpert SettingsやRestore Reviewへ到達しても、必要なReview / Confirm / Verificationを短絡しない。
- Tray Quick Accessは状態確認・Navigation中心とし、高リスクOperationの直接実行を提供しない。
- Stale notification / search resultからMaterial actionへ進む場合、現在状態を再取得し、古いContextとの差分を確認する。

### 35.21.1.5 Result / Return Mapping

| Result state | Return behavior | Must not imply |
|---|---|---|
| Verified | Return may show verified effective result | Return itself caused verification |
| Executed / Verification Pending | Preserve pending / verification state | Verified success |
| Needs Attention | Return exposes unresolved item / appropriate destination | Problem resolved by navigation |
| Failed | Keep failure context and recovery guidance | Failure automatically recovered |
| Unavailable / Not Applicable | Explain current applicability | Actual operation failure |

### 35.21.1.6 Responsive / Accessibility Rules

- Compact reflowでもOrigin / Target / Operation / Current State / primary actionの関係を維持する。
- Breadcrumbが折りたたまれても、現在TargetとOriginの関係を別のContext Header等で確認できる。
- Review → Confirm → Executeの対応関係をレスポンシブ再配置で逆転させない。
- ResultとVerificationは同一Operationのまとまりとして読み取れるよう保持する。
- 長い日本語 / 英語 / Game Name / Generation timestamp / Path等が折り返してもTarget identityを省略しない。
- Keyboard focusはNavigation → Context → Review / Action → Resultの意味順を維持する。

### 35.21.1.7 Wireframe Boundary

このAnnotated Wireframeは既存のCross-Screen Navigation / Transition semanticsを視覚化するものであり、新しいNavigation capability、Deep Link capability、Public Contract、Domain Status、Security capability、Recovery capabilityを追加しない。

遷移先にActionable controlが存在するかどうかは、適用対象の正式なFeature / Module / Contractおよび現在のBuild / verification availabilityに従う。UIの到達経路だけで実装・承認済みとは扱わない。
## 35.22 Cross-Screen Acceptance Criteria

- 主要6画面の目的と責務が遷移後も混同されない。
- Selected Game / Generation / Incident等の重要Contextが可能な範囲で保持される。
- 読み取り操作と高影響操作の遷移が区別される。
- High-impact operationは既存の安全ゲートをバイパスしない。
- Back / Cancel / Closeの意味が共通している。
- Reviewを開いただけで変更が予約・実行されない。
- Execute成功とVerify成功が区別される。
- エラーは安全な次の行動へ接続される。
- Quick Recoveryが実データの即時上書きに直結しない。
- Save Data → Restore → Recoveryで対象世代を失わない。
- Protection → Settingsで変更対象とScopeが維持される。
- Games → Save Data / ProtectionでGame Contextが維持される。
- Global SearchがReview / Confirm等の安全ゲートを迂回しない。
- Notificationから適切な詳細画面へ移動できる。
- Critical notificationを閉じてもCritical状態そのものが消えたことにならない。
- 主要状態語彙が画面間で一貫する。
- 外部状態が変化した可能性のあるContextを無条件に再利用しない。
- 遷移先やRecovery routingをUI独自の推測で決定しない。
- UI設計だけを理由に新しいRuntime Feature、Public Contract、Domain state、Security capabilityを追加しない。

## 36.13 Cross-Scenario Findings

シナリオ横断で、現時点のUI概念に次の共通境界が成立している。

### Safety Boundary

    Information
        ↓
    Review
        ↓
    Confirm
        ↓
    Execute
        ↓
    Verify

高影響操作だけがこの境界を使用し、読み取り専用操作まで過剰に重くしない。

### Context Boundary

    Origin
      +
    Target
      +
    Operation
      +
    Return Context

を意識して保持する。

特にRestore、Recovery、WIPER Incident、Settings Changeでは、対象を別のものへすり替えない。

### Result Boundary

    Requested
      ≠
    Executed
      ≠
    Verified

ユーザー向けUIでもこの3段階を必要に応じて区別する。

## 36.14 Existing Formal UI vs New Information Architecture

既存のFormal UX仕様には、Firewall、Web Protection、Audit Timelineなど、**機能・実装コンポーネント単位で分けられた旧来の画面構成**が残っている。

一方、本UI Conceptでは、ユーザー目的を軸に次の6画面を第一階層とする。

```text
Home
Games
Save Data
Protection
Recovery
Settings
```

ここで行っているのは、まず**情報アーキテクチャとNavigationの再編**であり、Formal Baseline、Feature Specification、Contracts、既存Runtime capabilityを自動的に変更することではない。

### 36.14.1 照合した正本と確認結果

今回の責務マッピングでは、少なくとも以下の現行 `main` を確認した。

| Source | 確認した事実 |
|---|---|
| `GST-BASELINE-011` / `Documentation/00_Baseline/Governance/11_UI_UX_and_User_Interaction_Model.md` | 旧UI Architecture Modelとして `Dashboard / Games / Save Data Hub / Web Protection / Firewall / Audit Timeline / Settings` を明示。Firewall遮断時の `Review → Confirm → Execute`、Quick Recovery、WIPER/Recovery関連UIも定義。 |
| `GST-BASELINE-047` / `Documentation/00_Baseline/Governance/47_User_Experience_and_Interface_Design_Governance_Model.md` | UXの正式原則は旧画面名に依存せず、危険操作・確認・透明性・状態表示・Recoveryを横断的に要求。 |
| `GST-ARCH-BASELINE-002-PART6` / `Documentation/01_Architecture/01-06_Presentation_and_UI_Architecture.md` | WPF/MVVM/Dispatcher境界を定義し、`Timeline View` を独立したPresentation componentとして定義。Timeline QueryはContracts Portを介し、Pagination 50件既定・Filter・Virtualizationを要求。 |
| `GST-BASELINE-020` / `Documentation/00_Baseline/Governance/20_User_Manual_and_Operational_Guide.md` | User Manualは `Firewall`、`Save Data Hub`、`Web リンク保護`、`隔離と復元`、`Settings` を独立した利用説明章として保持。 |
| `GST-MOD-FW-003` / `Documentation/modules/03-03_FW_Zero_Trust_Firewall.md` | FirewallはGame Profile単位の通信制御、Drift Detection、Anti-Cheat Compatibility Exception、ユーザー承認を含む既存機能責務として定義。 |
| `GST-SPEC-WEBLINK-001-PART5` / `Documentation/02_Features/Security/Web_Link_Protection_Spec/01-05_Presentation_and_Dialogs.md` | Web LinkのConfirm / Block DialogとHost Rule Managementは既存Presentation責務として定義。 |
| `GST-BASELINE-04` / `Documentation/00_Baseline/GameProtection/04_Audit_and_Evidence_Model.md` | Auditは証跡・完全性・Export・Retentionを含む独立した証拠モデルであり、UI名称を変更しても責務は維持されるべき対象。 |
| `GST-BASELINE-05` / `Documentation/00_Baseline/GameProtection/05_Save_Backup_and_Data_Ownership_Model.md` | Backup UIはGame Profile、Backup Status、Storage、Snapshot History、Restore / Exportを中心に定義。 |
| `GST-BASELINE-09` / `Documentation/00_Baseline/GameProtection/09_PC_Migration_and_Recovery_Model.md` | Migration / Manual Recovery / Firewall Rule Rebuildは既存Recovery責務として定義されているが、Recovery Hostを通常UIへ吸収する設計ではない。 |

これは**設計照合結果**であり、各Featureが現行ビルドで実装・利用可能であることを示すものではない。

### 36.14.2 旧画面 → 新6画面の責務マッピング

| 旧Formal UI / Surface | 新6画面での主配置 | 責務として保持する内容 | 分類 |
|---|---|---|---|
| Dashboard | **Home** | Protection Status、Configuration Status、Current Game、Important Events、Quick Recovery、Backup概要 | Navigation再編 |
| Games / Game Library & Launcher | **Games** | Game Library、Launch、Safe Onboarding、Selected Game、Game-specific protection/settings | Navigation再編 |
| Save Data Hub | **Save Data** | Backup Status、Storage、Snapshot History、Pin/Tag、Restore/Export、Save Bloat | Navigation再編 |
| Web Protection | **Protection** + **Games** + **Settings** | 保護状態・遮断通知はProtection、Game-specific Link PolicyはGames、Global / Advanced Rule ManagementはSettings | Navigation再編。ただしPolicy/Contract変更は別Scope |
| Firewall | **Protection** + **Games** + **Settings** | 現在の保護状態・Block detailはProtection、Game-specific Effective PolicyはGames、Rule CRUD / 詳細設定はSettingsのAdvanced/Expert | Navigation再編。ただし通信Policy変更は別Scope |
| Audit Timeline | **Protection** の Integrity / Audit領域 + **Advanced/Expert** | 最近の重要イベントはProtection、既存TimelineのFilter / Pagination / 詳細証跡はAdvanced/Expertへ保持。必要に応じGame/RecoveryからContext付きで参照 | Navigation再編。機能削減はScope変更 |
| Quarantine & Recovery | **Recovery** | 隔離復元、Restore Failure、Startup Recovery、WIPER Recovery、Manual Recovery、Recovery Host入口 | Navigation再編 |
| Settings / Maintenance / Migration / AI | **Settings** | Standard Profile、通知、AI、Migration、Maintenance、Factory Reset、Uninstall/Reversion等 | Navigation再編 |
| In-Game Overlay / Resident Surface | **独立した第一階層にはしない**。Active Game Contextに従う | ゲーム中の状態・Quick Action・通知等の既存Overlay責務 | Surface再編。入力/ランタイム挙動変更は別Scope |
| Smart Game IME Lock / Gamer Defenseの設定UI | **Settings**。Runtime状態はGames/ProtectionへContextual表示 | 既存設定・状態表示・ユーザー制御 | Navigation再編 |
| Configuration Time Machine | **Settings**で設定履歴・Diffを確認し、復元実行/結果は**Recovery**へ接続 | Snapshot History、Diff Review、Review → Confirm → Execute → Verify | Navigation再編。復元契約変更は別Scope |

### 36.14.3 Web Protection / Firewall / Audit Timeline の「分散配置」ルール

旧画面を新6画面へ移す際、単純に同じ内容を別ページへコピーするのではなく、**状態・対象・設定・証跡**の4種類に分ける。

```text
Protection
  ├ 現在何が保護されているか
  ├ なぜBlock / Warningが発生したか
  └ 重要なAudit / Integrity状態

Games
  ├ 選択中GameのEffective Policy
  ├ Game-specific Web / Firewall設定
  └ Game Contextを保持した詳細表示

Settings
  ├ Global / Advanced Configuration
  ├ Firewall Rule Management
  └ Web Link Host Rule Management

Recovery
  ├ BlockやIncidentの結果から安全に戻る導線
  └ Restore / Recovery / Manual Recovery
```

この分散配置によって、ユーザーは「Firewallという実装名のためにFirewall画面を探す」のではなく、**現在の状態を理解する → 対象を選ぶ → 設定を変える → 必要なら復旧する**という目的ベースの経路を利用できる。

ただし、分散配置は**同一機能を複数箇所へ複製することを意味しない**。Canonicalな設定・状態の所有場所を一つに保ち、他画面からはContext-preserving navigationまたはDetails/Deep Linkで参照する。

### 36.14.4 Audit Timeline の特別扱い

`01-06_Presentation_and_UI_Architecture.md` は `TimelineView`、`TimelineViewModel`、`ITimelineQueryService` を明示し、Timelineを単なる「最近の通知」ではなく、Pagination / Date Range / Game Profile / Event Category / Search Keywordを持つ詳細Query surfaceとして定義している。

したがって新IAでは、

- Homeの「最近のイベント」だけでTimelineを代替しない。
- Protectionに重要なIntegrity / Audit状態を表示する。
- 詳細Timeline QueryはProtectionのAdvanced/Expert詳細へ収容する。
- RecoveryやGamesからAudit Detailへ遷移する場合は、対象Game / Incident等のContextを可能な範囲で保持する。
- 既存のPagination、Filter、Virtualization、Contracts境界をUI再編だけを理由に削除しない。

**Timelineの詳細Query能力を削除する、Audit Eventの意味を変更する、Retention / Export責務を縮小する等はNavigation変更ではなくFeature / Contract Scopeの変更として扱う。**

### 36.14.5 現行Presentation / Recovery実装との境界

現行のPresentation / Recovery境界では、Main UIと独立したRecovery UIの責務を分離する。

- Main UIは通常のPresentation責務を担い、既存のApplication境界を介して機能へ接続する。
- 独立Recovery UIは通常の操作経路とは分離されたRecovery境界として扱う。
- Recovery境界は、公開済み仕様で定義された非破壊的なInspectや安全確認を中心に扱い、未定義・未実装のRecovery能力を新たに示さない。

内部の実装進捗管理や具体的なSourceファイル構成は本公開候補には含めない。

したがって、新6画面IAへの再編は、**既存Recovery HostをMainWindowへ吸収することでも、未実装Recovery操作を新しいボタンとして追加することでもない**。

現時点で確認できたPresentation実体とFormal UI仕様の間には粒度差があるため、旧画面名称を理由に既存Runtime implementationを削除・移設してはならない。

### 36.14.6 User Manual / Search / Index 参照の扱い

現行User Manualには、旧来の `Firewall`、`Web リンク保護`、`Save Data Hub`、`隔離と復元`、`Settings` 等が独立章として残っている。これは**直ちにManualを削除する理由ではない**。

扱いは次のように分ける。

| 参照 | 現時点の扱い |
|---|---|
| Feature名・モジュール名としての「Firewall」「Web Link Protection」「Audit」 | 維持。これらはDomain / Feature責務の名称であり、Navigation Labelと同一である必要はない。 |
| ユーザー向け第一階層としての旧「Firewall」等 | 将来のManual整合化対象。新6画面IAへ誘導する説明へ再編する。 |
| `Documentation/README.md` の `01-06 ... Timeline` 等 | Architecture componentへの参照なので現段階で変更不要。 |
| 統合検索の検索対象としての旧画面名 | 現行の正式な検索Index実装・Indexファイルをこのレビューで確定できていないため、**クリーンと判定しない**。実装復帰前のGovernance taskとして実検索対象とDeep Linkを監査する。 |
| Search / Deep Link | 検索結果から直接High-impact Executeへ到達させず、対象ページのReview / Confirm等の既存安全ゲートへ必ず接続する。 |

この時点では、検索Indexを推測で編集したり、旧名称を一括置換したりしない。

### 36.14.7 「Navigation再編」と「実質的Scope変更」の判定規則

以下は**Navigation / IA再編として扱える**。

- 旧独立ページを6画面の内部Section / Details / Advanced領域へ移す。
- 同一の既存Contract / Use Case / Security capabilityを別Navigation位置から呼び出す。
- Context-preserving Deep LinkやBreadcrumbを追加する。
- 既存状態をHome / Protection / Gamesなどへ要約表示し、詳細へ遷移させる。
- TimelineをProtection配下の詳細surfaceへ移す。ただし既存Query能力を保持する。

以下は**Feature / Contract / Security scope変更として別途Change Controlが必要**。

- FirewallのDefault Policy、Loopback扱い、Anti-Cheat Compatibility Exception等の意味を変更する。
- Web LinkのQuery stripping、Allowlist、Browser Picker、Block判定等の既存契約を変更する。
- TimelineのEvent Type、Filter、Pagination、Retention、Export、Integrity meaningを削除・変更する。
- Quarantine / Restore / Recovery Hostの操作対象や安全ゲートを変更する。
- 新6画面化を理由に既存Runtime capability、Public Contract、Domain Stateを削除・追加する。
- 「画面を移動しただけ」という名目で、既存Featureを使えなくする。

判定に迷う場合は「UI変更」ではなく、**既存Contract / Feature Specification / Change Controlに対する影響分析が必要な変更**として扱う。

### 36.14.8 今後のGovernance task

本Mapping完了時点で、次の作業は**別工程**として登録する。

1. Formal UX Baselineの旧Navigation Modelを、新6画面IAへ正式整合化するかChange Controlで判断する。
2. User Manualの章構成・画面リンクを新IAへ段階的に整合化する。
3. 統合Search Index / Deep Linkの実装範囲を確認し、旧画面名が単なるFeature検索語なのか旧Navigationとして残存しているのかを区別する。
4. Web Protection / Firewall / Audit Timelineの既存ContractsとPresentation implementationの実体を、Implementation Readiness再開時に改めて照合する。
5. 上記4点を完了するまでは、6画面IAを根拠に既存Runtime implementationを削除・移設しない。

この工程は、現在のDesign-First Pause中に**設計・Governance上の整合性を確保するための作業**であり、新しいRuntime featureの追加許可ではない。

### 36.14.9 Mapping Acceptance Criteria

- 旧Formal UIの各主要責務が、新6画面のいずれか、または明示されたCross-Screen Surfaceへ一意に説明できる。
- Web Protection / FirewallのConfigurationとRuntime Statusが混同されない。
- Audit Timelineの詳細Query能力を「最近のイベント」へ誤って縮退させない。
- Recovery Hostを通常GST UIの単なる別ページとして扱わない。
- User Manualの旧画面名が残っていることを、未承認の仕様不整合修正として自動削除しない。
- Search Indexの未確認状態を「存在しない」と誤認しない。
- Navigation再編とFeature / Contract / Security scope変更の境界が文書化されている。
- 既存Runtime implementationを新IAだけを理由に削除・移設しない。
- UI Conceptの責務マッピングは、実装済み・テスト済み・Windows実機確認済みを意味しない。

---

## Extraction Note

The material was originally extracted from the former Section 35 and Section 36.13/36.14 scope of the parent UI working draft. The working-draft split changed document ownership, not product authority. Any future semantic change must be reviewed as a design change; the presence of a flow or transition rule in this file must not be treated as a newly approved runtime requirement.

End of Document
