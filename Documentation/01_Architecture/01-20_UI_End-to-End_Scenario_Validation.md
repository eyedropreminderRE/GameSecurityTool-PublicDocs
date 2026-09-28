# 01-20: UI End-to-End Scenario Validation

**Document ID:** GST-ARCH-UI-SCENARIOS-001  
**Version:** 0.3 (Annotated End-to-End Scenario / Visual UX Review)  
**Status:** Design Working Draft — Not an independent feature authority  
**Authority Relationship:** Subordinate to GST-BASELINE-011, GST-BASELINE-047, applicable approved module / feature specifications, and Change Control decisions  
**Parent Concept:** [01-08_UI_Information_Architecture_and_Interaction_Concept.md](01-08_UI_Information_Architecture_and_Interaction_Concept.md)  
**Historical Source Mapping:** Former Section 36 excluding Section 36.13 and Section 36.14 of the parent 01-08 UI working draft  
**Current Ownership:** This document is the maintained location for the End-to-End Scenario material after the working-draft split.
**Purpose:** Organize existing UI information-architecture and interaction design for this scope without expanding Runtime Features, Public Contracts, Domain Status, Security capabilities, or Implementation Readiness.

## Document Boundary

This document preserves the UI design material that was formerly organized as the End-to-End Scenario portion of Section 36 in the parent 01-08 UI working draft. After the working-draft split, 01-08 retains global IA / interaction principles while this document owns the End-to-End Scenario validation material.

The preserved section numbering is historical traceability only. It does not imply that this scenario material still exists in 01-08.

This document does not authorize Runtime implementation by itself. Existing formal baselines, approved module / feature specifications, Public Contracts, Security boundaries, and Change Control decisions remain authoritative where scope overlaps.

## Related Authoritative Documents

- [11_UI_UX_and_User_Interaction_Model.md](../00_Baseline/Governance/11_UI_UX_and_User_Interaction_Model.md) — Formal UI/UX baseline
- [47_User_Experience_and_Interface_Design_Governance_Model.md](../00_Baseline/Governance/47_User_Experience_and_Interface_Design_Governance_Model.md) — Formal UX governance
- [27_Quality_Assurance_and_Release_Gate_Model.md](../00_Baseline/Quality/27_Quality_Assurance_and_Release_Gate_Model.md) — QA authority
- [01-16_UI_Common_Interaction_and_Display_Design.md](01-16_UI_Common_Interaction_and_Display_Design.md) — Common interaction / display rules
- [01-17_UI_Reference_Patterns_and_Common_Component_Library.md](01-17_UI_Reference_Patterns_and_Common_Component_Library.md) — Common patterns / component boundaries
- [01-18_UI_User_Journeys_and_Review_Model.md](01-18_UI_User_Journeys_and_Review_Model.md) — Journey gates / safe exit model
- [01-19_UI_Cross-Screen_Flow_and_Transition_Model.md](01-19_UI_Cross-Screen_Flow_and_Transition_Model.md) — Cross-screen transitions / context preservation

---

# 36. End-to-End User Scenario Validation

ここまで定義したUIを、個々の画面ではなく実際のユーザー行動として通し確認する。

この章は実装テスト結果ではなく、**UI / UX設計上のシナリオレビュー**である。したがって、ここで「通過」として扱うのは、設計上の導線が成立していることを意味し、実装済み・Windows実機検証済みを意味しない。
### 36.0.1 Scenario Review Contract

各Scenarioは、`01-18` のJourney Review Contractおよび `01-19` のCross-Screen Flowに従って確認する。

- **Precondition / Start** — 前提条件と入口が明確である。
- **Understand** — 対象、現在状態、影響、既知の制約を理解できる。
- **Decide / Review** — ユーザー判断が必要な箇所と、Confirmが必要な場合を明示する。
- **Execute** — 実際の操作と、操作前に副作用が発生しない境界を明示する。
- **Verify** — 結果の確認方法を示し、`Requested ≠ Executed ≠ Verified` を維持する。
- **Safe Exit / Return** — Cancel / Back / Close / Failure時の安全な戻り先を示す。
- **Capability Gate** — Feature / Module / Contract / Build availabilityに依存する箇所は、UI設計上の導線と実際の利用可能性を分離する。

Scenarioの `Design-Coherent` は「UI上の導線が設計として矛盾しない」という意味だけを持つ。実装済み、利用可能、Windows実機検証済み、Release Readyの意味ではない。

未承認・未実装の操作は、Scenarioの存在を理由にActionableとして表示してはならない。


## 36.1 Scenario A — First Launch

目的: 初回利用者がGSTの性質と重要リスクを理解したうえで開始する。

導線:

    GST起動
      ↓
    First Launch Risk Acknowledgment
      ↓
    各確認項目を確認
      ↓
    Start GST
      ↓
    Home

成立条件:

- 実験的ソフトウェアであることが開始前に分かる。
- 重要データの保護について、外部バックアップの必要性を確認できる。
- 未確認項目がある状態で開始できない。
- 初回確認UIが通常の設定画面へ移動しただけで自動的に再実行されない。
- Start後はHomeで現在状態を理解できる。

レビュー結果: **Design-Coherent (Capability-Gated where applicable)**

## 36.2 Scenario B — Normal Game Start

目的: 日常利用で最短経路からゲームを起動する。

導線:

    Home
      ↓
    Games
      ↓
    Game A
      ↓
    Launch
      ↓
    Game A running

成立条件:

- Game AのGame Contextが維持される。
- LaunchがSettingsや削除などの低頻度操作より分かりやすい。
- Protection StatusとGame Statusが混同されない。
- 起動だけなら不必要な長いConfirmを要求しない。
- 起動前に重要な既知状態がある場合は、過度に妨げずに提示できる。

レビュー結果: **Design-Coherent (Capability-Gated where applicable)**

## 36.3 Scenario C — Immediate Backup

目的: プレイ前後などに、現在のセーブを短い導線でバックアップする。

導線:

    Games
      ↓
    Game A
      ↓
    Save Data
      ↓
    今すぐバックアップ
      ↓
    Progress
      ↓
    Result / Verification
      ↓
    Save Data

成立条件:

- 現在のセーブとBackup Generationが明確に区別される。
- Backup中の進行状況が分かる。
- 完了後に対象・保存先・結果を確認できる。
- Backup失敗時も既存世代の状態を隠さない。
- Backup作成がRestore用の高影響Confirmと混同されない。

レビュー結果: **Design-Coherent (Capability-Gated where applicable)**

## 36.4 Scenario D — Restore a Known Good Generation

目的: ゲーム内不具合などで、ユーザーが過去世代へ戻す。

導線:

    Save Data
      ↓
    Game A
      ↓
    Generation
      ↓
    Restore Review
      ↓
    Recovery Context
      ↓
    Confirm
      ↓
    Execute
      ↓
    Verify
      ↓
    Result
      ↓
    Save Data

成立条件:

- 選択世代を確認してから初めてRestoreへ進む。
- 現在のセーブがどう扱われるかをReviewで理解できる。
- RescueSnapshotによる復元前保全が説明される。
- Confirm前に実データ変更が発生しない。
- 暗号学的検証成功だけでユーザー承認済みにならない。
- Execute成功とVerify成功を区別する。
- Back / Cancelで選択対象を誤って失わない。

レビュー結果: **Design-Coherent (Capability-Gated where applicable)**

## 36.5 Scenario E — Restore Failure / Startup Recovery

目的: Restore中の失敗やプロセス終了後に、ユーザーが現在状態を把握する。

導線:

    GST Startup
      ↓
    incomplete Restore detected
      ↓
    Startup Recovery
      ↓
    available approved recovery path / recovery result
      ↓
    Verify
      ↓
    Recovery Result
      ↓
    Home / Save Data

成立条件:

- 通常の新しいRestoreを開始する前に未完了状態を確認できる。
- 現在データの状態とRecovery結果が分離される。
- RescueSnapshotが存在することだけを「復旧成功」と誤認させない。
- Recovery失敗はCriticalとして残る。
- 自動復旧できない場合に手動復旧の次の行動を示す。
- 自動復旧の表示は、実際に利用可能で承認済みのRecovery capabilityが存在する場合に限る。

レビュー結果: **Design-Coherent (Capability-Gated where applicable)**

## 36.6 Scenario E2 — Initial Safe Configuration

目的: 初回起動後、ユーザーが専門知識なしに標準的な利用状態を確認・構成する。

導線:

    First Launch
      ↓
    Standard Profile / ordinary configuration
      ↓
    Current Value / Scope / Impact
      ↓
    Apply when supported
      ↓
    Verify effective configuration
      ↓
    Home / Settings

成立条件:

- Standard / Customの意味がProtection Statusと混同されない。
- Pending変更を現在の実効状態として先行表示しない。
- Apply前に実効状態が変化したことを表示しない。
- Verifyできない場合は、適用済みと断定しない。
- Cancel / Backで未適用変更を暗黙に確定しない。

レビュー結果: **Design-Coherent (Capability-Gated)**

## 36.7 Scenario E3 — Portable Game Safe Onboarding

目的: ZIP / 7z / RAR等から導入するゲームを、安全な説明付き導線で登録する。

導線:

    Games
      ↓
    Portable Onboarding
      ↓
    Destination Review
      ↓
    Extraction / Validation
      ↓
    Executable Selection
      ↓
    Registration
      ↓
    Verify result
      ↓
    Games / Game Context

成立条件:

- 実際の対象と展開先をmaterial change前に確認できる。
- Extraction / Validation / Registrationの結果を分離する。
- 「問題を確認できなかった」を「安全である」と同一視しない。
- Cancel時に既存登録状態を不意に変更しない。
- 現行Buildで利用不可の場合は、未実装操作をActionableとして表示しない。

レビュー結果: **Design-Coherent (Capability-Gated)**

## 36.8 Scenario E4 — Strict / High-Caution Launch

目的: 承認済みのより厳格な保護モードでゲームを起動する際、何が変わるかを理解する。

導線:

    Games
      ↓
    Game A
      ↓
    Strict / High-Caution Review
      ↓
    Confirm when required
      ↓
    Existing launch operation
      ↓
    Verify active context
      ↓
    Game A running

成立条件:

- Strict LaunchをSandbox / Full Isolationと誤認させない。
- 厳格化される範囲、互換性上の注意、適用Contextを説明する。
- Confirmは適用仕様が要求する場合のみ行う。
- Active Strict Contextは権威的な状態から表示する。
- 起動失敗時に「保護が適用済み」と断定しない。

レビュー結果: **Design-Coherent (Capability-Gated)**

## 36.9 Scenario E5 — File Lock / External Tool Conflict

目的: 外部プロセスによるファイルロック等で処理できない場合、推測と確認済み情報を分けて安全に案内する。

導線:

    Operation failure
      ↓
    File Lock / Conflict detail
      ↓
    Confirmed facts / Unknown attribution
      ↓
    Retry / Wait / Manual guidance
      ↓
    Verify retry result when applicable
      ↓
    Return to original context

成立条件:

- 対象ファイル、確認できたロック情報、現在の影響を区別して表示する。
- プロセス名 / PIDを表示する場合も、確定した責務や悪意を推測しない。
- GSTが外部Security Toolを停止・終了すると誤認させない。
- Retry結果と未解決状態を区別する。
- 適用可能なRecovery pathが存在しない場合は手動対応を明示する。

レビュー結果: **Design-Coherent (Capability-Gated)**

## 36.10 Scenario E6 — Maintenance / Migration

目的: 専門的な保守・移行操作を通常利用から分離し、対象と影響を明確にして実行する。

導線:

    Settings / Advanced
      ↓
    Maintenance or Migration
      ↓
    Purpose / Scope / Impact
      ↓
    Review
      ↓
    Confirm when required
      ↓
    Execute
      ↓
    Verify
      ↓
    Result / Recovery guidance

成立条件:

- 対象、Scope、影響を実行前に確認できる。
- Migration PackageとBackup Archiveを混同しない。
- MaintenanceとFactory Resetなど高影響操作を同一Primary Actionとして扱わない。
- Verify前に完了済みと断定しない。
- 現行Buildで未提供の操作はActionableとして表示しない。

レビュー結果: **Design-Coherent (Capability-Gated)**

## 36.11 Scenario F — Firewall Block During Game

目的: 通信遮断が発生したとき、原因・影響・次の操作を理解する。

導線:

    Game A running
      ↓
    Firewall block
      ↓
    Minimal notification
      ↓
    Protection / Block detail
      ↓
    Review
      ↓
    Confirm
      ↓
    Execute
      ↓
    Verify
      ↓
    Game A contextへReturn

成立条件:

- 通知だけで内部ruleを操作しない。
- 対象ゲーム、実行ファイル、通信方向、適用ポリシー / ルール、想定影響を確認できる。
- 「今回だけ」など未定義Scopeを勝手に提示しない。
- Critical状態の場合、Smart DND中でも完全に隠れない。
- Firewallの変更後に実状態確認を行う。

レビュー結果: **Design-Coherent (Capability-Gated where applicable)**

## 36.12 Scenario G — WIPER Incident

目的: 異常なファイル変更が発生した際、ユーザーがパニックになっても安全な次の操作へ進める。

導線:

    Protected session
      ↓
    WIPER signal
      ↓
    Alert / Critical notification
      ↓
    Incident detail
      ↓
    Current state confirmation
      ↓
    Existing permitted action
      ↓
    Verify
      ↓
    Recovery guidance if needed

成立条件:

- 観測シグナルと攻撃元の断定を区別する。
- 現在対象となっているGame / Sessionを確認できる。
- Suspend / Terminate等は既存契約の範囲に限定する。
- UIから直接Infrastructure APIを呼ばない。
- Incident後に「何が止まったか」「何を再開できるか」「何を確認すべきか」を理解できる。

レビュー結果: **Design-Coherent (Capability-Gated where applicable)**

## 36.13 Scenario H — Safe Configuration Change

目的: ユーザーが設定を変更し、変更後の影響を把握する。

導線:

    Protection / Games
      ↓
    Feature detail
      ↓
    Settings
      ↓
    Current / New Value
      ↓
    Scope / Impact
      ↓
    Review
      ↓
    Confirm
      ↓
    Execute
      ↓
    Verify
      ↓
    Protection / Settings

成立条件:

- 「何を変更するのか」「どこへ効くのか」が明確。
- Custom設定になったこととProtection異常が区別される。
- 高影響設定では確認を省略しない。
- Verify失敗時に変更済みとして断定しない。
- 変更対象を失わずに元のContextへ戻れる。

レビュー結果: **Design-Coherent (Capability-Gated where applicable)**

## 36.14 Scenario I — Configuration Time Machine

目的: 過去の設定状態の一部だけを安全に戻す。

導線:

    Settings
      ↓
    Configuration Time Machine
      ↓
    Snapshot History
      ↓
    Diff Preview
      ↓
    Select changes
      ↓
    Review
      ↓
    Confirm
      ↓
    Partial Restore
      ↓
    Verify
      ↓
    Result

成立条件:

- スナップショット選択と復元実行を分ける。
- 差分がユーザー向けに理解できる。
- 選択していない対象へ変更が及ぶと誤認させない。
- アンインストール済みゲームなど既存仕様の対象外条件を説明できる。
- Restore結果をVerifyなしに成功と断定しない。

レビュー結果: **Design-Coherent (Capability-Gated where applicable)**

## 36.15 Scenario J — GST Cannot Start / Recovery Host

目的: 通常GSTが利用できない場合にも、独立復旧経路へ到達できる。

導線:

    GST startup failure
      ↓
    Recovery Host
      ↓
    Inspect
      ↓
    Available recovery paths
      ↓
    User Confirm
      ↓
    Recover
      ↓
    Verify
      ↓
    Result

成立条件:

- Recovery Hostが通常GSTとは別の復旧面であることが分かる。
- 起動しただけではシステム状態を変更しない。
- 通常SQLite DBやMainWindowが利用できなくても、許可された範囲のInspectを行える。
- Recovery Host自身の検証に失敗した場合はfail-closed。
- Available recovery pathsは、承認済み・現行Buildで利用可能な操作だけをActionableとして表示する。
- 未実装 / 未検証の復旧操作をボタンとして表示しない。

レビュー結果: **Design-Coherent (Capability-Gated where applicable)**

## 36.16 Scenario K — AI Consultation With Privacy Gate

目的: AIへ相談する場合に、送信内容を理解してから送信する。

導線:

    Settings / relevant context
      ↓
    AI consultation
      ↓
    Prepare
      ↓
    Sanitize
      ↓
    Preview
      ↓
    Optional Edit
      ↓
    Re-sanitize / Re-fingerprint
      ↓
    Re-approve
      ↓
    Send
      ↓
    Informational answer
      ↓
    User decides next action

成立条件:

- AI相談を開始しただけで外部送信しない。
- Previewで送信内容を確認できる。
- Preview後の編集で旧Approvalを再利用しない。
- AI回答だけで設定変更や復旧を自動実行しない。
- AIの助言を保証や確定診断として表示しない。

レビュー結果: **Design-Coherent (Capability-Gated where applicable)**

## 36.17 Scenario L — Uninstall / Reversion

目的: GSTを削除する際、GST本体・OS設定・ユーザー資産の扱いを混同しない。

導線:

    Settings / Maintenance
      ↓
    Uninstall / Reversion
      ↓
    Review
      ↓
    Confirm
      ↓
    Execute
      ↓
    Verify
      ↓
    Completion / Recovery guidance

成立条件:

- GST本体の削除とOS設定復元を分離して説明する。
- Backup / Save Data資産の扱いを明示する。
- 異常終了時は「完全に元へ戻った」と表示しない。
- 必要に応じてRecovery Hostへの導線を提示する。

レビュー結果: **Design-Coherent (Capability-Gated where applicable)**

## 36.18 Journey Coverage Matrix

17件のJourney InventoryとScenarioの対応を明示する。

| Journey | Primary Scenario Coverage | Coverage | Key Gate |
|---|---|---|---|
| J01 First Launch | A | Design-Mapped | Acknowledgement / Safe Exit |
| J02 Initial Configuration | E2 / H | Design-Mapped | Draft / Applied / Verified |
| J03 Register Existing Game | B | Design-Mapped | Registration outcome / Context |
| J04 Portable Onboarding | E3 | Design-Mapped | Destination / Validation / Registration |
| J05 Everyday Game Launch | B | Design-Mapped | Launch result / Active Context |
| J06 Strict Launch | E4 | Design-Mapped | Capability / compatibility / active context |
| J07 Gameplay / Runtime Status | F / G | Design-Mapped | Critical visibility / runtime evidence |
| J08 Routine Save Backup | C | Design-Mapped | Backup result / verification where defined |
| J09 Restore / Undo | D / E | Design-Mapped | Review / Confirm / Execute / Verify |
| J10 Security / Integrity Event | F / G / E5 | Design-Mapped | Known / Concerning / Unknown |
| J11 Firewall Block | F | Design-Mapped | Existing policy / confirmation / verify |
| J12 File Lock Conflict | E5 | Design-Mapped | Confirmed facts / uncertainty / manual path |
| J13 Recovery Host / Startup Failure | J / E | Design-Mapped | Independent recovery boundary |
| J14 Configuration Change / Time Machine | H / I | Design-Mapped | Scope / diff / verification |
| J15 AI Assistance | K | Design-Mapped | Sanitize / Preview / Re-approve / Send |
| J16 Maintenance / Migration | E6 | Design-Mapped | Scope / impact / verify |
| J17 Uninstall / Reversion | L | Design-Mapped | Asset separation / supported OS reversion |

This matrix establishes design-review coverage only; it does not claim implementation coverage.

## 36.19 End-to-End UX Review Result

現時点の設計レビューでは、主要な日常操作と高影響操作について、**Understand → Decide → Confirm → Execute → Verify** の原則が横断的に成立している。

17件のJourneyとScenario coverage surfaceが対応付けられている。ただし、これは設計上のEnd-to-End経路が定義されていることを意味し、各Featureの実装・Build availability・Windows実機検証を意味しない。

Capability-Dependent Scenarioでは、UIが先行して未承認機能を表示・実行可能化しないことを前提とする。

一方、これはUI設計上のレビュー結果であり、実装ReadinessやWindows実機のUX検証結果ではない。

Formal Baselineと新IAの責務マッピングは完了し、Visual Design / Design Tokens / Component Statesは [01-09_UI_Visual_Design_System.md](01-09_UI_Visual_Design_System.md) に分離した。Visual FoundationはHome / Games / Save Data / Protection / Recovery / Settingsおよび共通Interaction / Pattern / Journey / Cross-ScreenのAnnotated Working Baselineへ適用済みである。

## 36.19.1 Annotated Scenario Map — End-to-End UX Working Baseline

このScenario Mapは、6つの主要画面と17件のJourneyを跨いで、入口から結果確認までの連続性をレビューするための作業基準である。Scenarioの存在、Coverage、または表示上の経路は、対応するRuntime Feature / Contractの実装・承認を新規に発生させない。

```text
                         ┌──────────────┐
                         │ First Launch │
                         └──────┬───────┘
                                ↓
                         ┌──────────────┐
                         │     Home     │
                         └──────┬───────┘
                  ┌─────────────┼──────────────┐
                  ↓             ↓              ↓
               Games        Save Data       Protection
                  │             │              │
                  │             ├─ Backup      ├─ Status / Event
                  │             │              └─ Settings
                  │             └─ Restore          │
                  │                    ↓             ↓
                  │               Recovery ←────────┘
                  │                    │
                  ├─ Launch / Strict   │
                  │                    │
                  └────────────────────┘
                                         ↓
                                   Result / Verify
                                         ↓
                               Return / Recovery guidance

       Settings ── Configuration / Time Machine / Maintenance / Migration
          │                         │
          └────────── Review ───────┘
                     ↓
            Execute / Observe / Verify
```

### 36.19.1.1 Scenario Gate Annotations

| ID | End-to-End gate | Review focus | Safety boundary |
|---|---|---|---|
| E2E-1 | Entry | Origin / Target / current context | Entryだけで成功・実行済みとしない |
| E2E-2 | Understand | Current State / Scope / Impact / Known-Unknown | 未確認情報を確定原因へ変換しない |
| E2E-3 | Decide / Review | Proposed result / confirmation requirement | Review表示だけでApproval済みとしない |
| E2E-4 | Execute | Existing approved operation | Scenarioから新Operationを発明しない |
| E2E-5 | Observe / Reconcile | Operation result ↔ current state | Acknowledgement / progressだけでVerifiedにしない |
| E2E-6 | Verify | Final state / authoritative evidence | Execute / 100%をVerifiedへ自動昇格しない |
| E2E-7 | Return / Recovery | Result + unresolved state + next destination | 静かにOriginへ戻して問題を隠さない |

### 36.19.1.2 Scenario-to-Screen Continuity

| Scenario type | Primary surfaces | Required continuity |
|---|---|---|
| First launch | First Launch → Home | Acknowledgement result / current state |
| Everyday game | Home → Games → Game Context | Game identity / launch result |
| Backup | Games → Save Data | Game / Save Context / backup result |
| Restore | Save Data → Restore Review → Recovery | Generation / Current Save / impact / verification |
| Runtime security event | Games / Protection → Incident / Recovery | Game / Session / known-unknown / next action |
| Configuration | Protection / Games → Settings | Setting scope / current-proposed / apply / verify |
| Maintenance / Migration | Settings → Advanced flow | Asset / scope / result / recovery path |
| Uninstall / Reversion | Settings → Review → Result | Application / user assets / supported OS scope |
| AI consultation | Relevant surface → Privacy Gate | Payload / sanitize / re-approval / send |

### 36.19.1.3 Cross-Scenario State Invariants

- `Requested ≠ Executed ≠ Verified` はScenarioを跨いでも維持する。
- `Protection Status ≠ Configuration Status` を維持する。
- `Current Save ≠ Backup Generation` を維持する。
- `Generation Exists ≠ Integrity ≠ Restore Eligibility` を維持する。
- `Back ≠ Cancel ≠ Close` を維持する。
- `Dismiss / Acknowledge ≠ Resolved` を維持する。
- `Completed ≠ Verified` が適用対象Contractで必要な場合は分離表示する。
- Unavailable / Not Applicable / Failedを同じGeneric Errorとして扱わない。

### 36.19.1.4 Failure / Recovery Continuity

各Scenarioで失敗した場合、次の順序を基本とする。

```text
Failure / Incident
      ↓
Preserve current state / identify scope
      ↓
Known facts vs unknowns
      ↓
Existing authoritative recovery guidance
      ↓
Recovery / Manual path when available
      ↓
Verify
      ↓
Result + unresolved state
```

- Recovery routingはScenario UIが独自に推測しない。
- 自動Recoveryが存在しない場合、存在するように見せるplaceholder Actionを作らない。
- Critical状態は通常Notificationだけで完結させない。

### 36.19.1.5 Responsive / Accessibility Rules

- Scenarioの入口からResultまで、Target / Scope / Operationの関係が画面サイズ変更後も保持される。
- 文字拡大時にImpact、Confirmation、Verification、Recovery guidanceを削除しない。
- Scenario中の画面再配置でDestructive actionが誤ってPrimary actionへ昇格しない。
- ResultとVerificationは同じOperationに属することが視覚的に明確である。
- Long Japanese / English / Path / Generation metadataが折り返してもTarget identityを失わせない。

### 36.19.1.6 Scenario Validation Boundary

Scenario ValidationはUX設計整合性の確認であり、Runtime integration test、Windows実機検証、Feature completeness、Release readinessの代替ではない。

Scenarioが`Design-Coherent (Capability-Gated where applicable)`であっても、未実装・未検証・UnavailableのCapabilityをActionableとして表示することは許可されない。

このAnnotated Scenario Mapから、新しいFeature、Public Contract、Domain Status、Storage behavior、Security capability、Recovery capabilityを追加しない。

## 36.20 End-to-End Acceptance Criteria

- 初回起動から通常利用まで、ユーザーが現在Contextを見失わない。
- Game Start / Backup / Protection確認の導線が日常利用として過度に重くならない。
- Restore / Recovery / Firewall変更 / 高影響設定変更では安全ゲートが維持される。
- ExecuteとVerifyが必要な場面で区別される。
- Restore Failure / Startup Recovery / WIPER Incident / Recovery Hostの各経路がCritical状態を隠さない。
- Critical状態の表示・Dismiss・再表示は、問題そのものの解決・Verifyとは別に扱われる。
- AI ConsultationがPrivacy approvalをバイパスしない。
- Uninstall / ReversionでGST本体、OS設定、User Assetsが混同されない。
- 旧Formal UIと新6画面IAの不一致を仕様変更として黙って解消しない。
- UIレビュー結果を実装済み・テスト済みと誤認させない。
- Annotated Wireframeへ進む前に、Formal Baselineと新IAの責務マッピング、およびVisual Foundationの責務境界を確認できる。
- Visual Designの詳細は `01-09_UI_Visual_Design_System.md` に一元化され、`01-08` と重複しない。
- Journey / Scenario coverageを理由に新しいRuntime Feature、Public Contract、Domain status、Security capabilityを追加しない。
- Scenario上のExecute / Verifyは、適用対象の権威的なFeature / Module / Contract / Build availabilityが成立する場合にのみActionableとする。

---

## Extraction Note

The material was originally extracted from the former End-to-End Scenario scope of Section 36 in the parent UI working draft. The working-draft split changed document ownership, not product authority. Any future semantic change must be reviewed as a design change; the presence of a scenario or coverage row in this file must not be treated as a newly approved runtime requirement.

End of Document
