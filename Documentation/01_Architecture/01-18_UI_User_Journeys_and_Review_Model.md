# 01-18: UI User Journeys and Review Model

**Document ID:** GST-ARCH-UI-JOURNEYS-001  
**Version:** 0.3 (Annotated Journey / Review UX Working Baseline)  
**Status:** Design Working Draft — Not an independent feature authority  
**Authority Relationship:** Subordinate to GST-BASELINE-011, GST-BASELINE-047, applicable approved module / feature specifications, and Change Control decisions  
**Parent Concept:** [01-08_UI_Information_Architecture_and_Interaction_Concept.md](01-08_UI_Information_Architecture_and_Interaction_Concept.md)  
**Historical Source Mapping:** Former Section 17 + Section 18 + Section 19 + Section 20 + Section 21 of the parent 01-08 UI working draft  
**Current Ownership:** This document is the maintained location for the Journey / Review material after the working-draft split.
**Purpose:** Organize existing UI information-architecture and interaction design for this scope without expanding Runtime Features, Public Contracts, Domain Status, Security capabilities, or Implementation Readiness.

## Document Boundary

This document preserves the UI design material that was formerly organized as Sections 17 through 21 of the parent 01-08 UI working draft. After the working-draft split, 01-08 retains global IA / interaction principles while this document owns the Journey / Review material.

The preserved section numbering is historical traceability only. It does not imply that Sections 17 through 21 still exist in 01-08.

This document does not authorize Runtime implementation by itself. Existing formal baselines, approved module / feature specifications, Public Contracts, Security boundaries, and Change Control decisions remain authoritative where scope overlaps.

## Related Authoritative Documents

- [11_UI_UX_and_User_Interaction_Model.md](../00_Baseline/Governance/11_UI_UX_and_User_Interaction_Model.md) — Formal UI/UX baseline
- [47_User_Experience_and_Interface_Design_Governance_Model.md](../00_Baseline/Governance/47_User_Experience_and_Interface_Design_Governance_Model.md) — Formal UX governance
- [20_User_Manual_and_Operational_Guide.md](../00_Baseline/Governance/20_User_Manual_and_Operational_Guide.md) — User guidance authority
- [01-16_UI_Common_Interaction_and_Display_Design.md](01-16_UI_Common_Interaction_and_Display_Design.md) — Common interaction / display rules
- [01-17_UI_Reference_Patterns_and_Common_Component_Library.md](01-17_UI_Reference_Patterns_and_Common_Component_Library.md) — Common patterns / component boundaries
- [01-19_UI_Cross-Screen_Flow_and_Transition_Model.md](01-19_UI_Cross-Screen_Flow_and_Transition_Model.md) — Cross-screen transitions and context preservation
- [01-20_UI_End-to-End_Scenario_Validation.md](01-20_UI_End-to-End_Scenario_Validation.md) — End-to-end scenario validation

---

# 17. GST User Journey Inventory

This section is an application-wide UX inventory. It describes the user's goal, questions, required information, decision point, and safe exit across the complete lifecycle. It does not create new product behavior.

## 17.1 Journey Map Overview

Discover / Install
  ↓
First Run & Risk Acknowledgment
  ↓
Initial Safe Configuration
  ↓
Register / Import Game
  ↓
Choose Launch Mode
  ↓
Play & Monitor
  ↓
Backup / Protect Save Data
  ↓
Normal Problem Resolution
  ↓
Security / Integrity Event
  ↓
Recovery / Restore
  ↓
Verify Normal State
  ↓
Maintain / Customize / Migrate
  ↓
Uninstall / Revert

Two cross-cutting journeys exist at every stage: Help / Learn and privacy-sensitive AI assistance.

### 17.1.1 Common Journey Review Contract

Every journey in this inventory is reviewed using the same conceptual fields:

1. **Start / Entry** — what condition or user intent starts the journey, and from which surface it is reachable.
2. **Understand** — what the user must know before deciding.
3. **Decide** — what choice or acknowledgement the user makes, including whether the applicable authority requires Review or Confirm.
4. **Execute** — the actual user operation, when one exists. Read-only journeys may have no Execute step.
5. **Verify** — what evidence is sufficient to report the resulting state. Requested ≠ Executed ≠ Verified.
6. **Safe Exit / Return** — what happens when the user cancels, goes Back, closes the window, or when execution fails.
7. **Authority Gate** — the applicable Feature / Module / Contract / Security boundary that determines whether the operation exists and which claims the UI may make.

This contract is a design-review framework, not a new Runtime contract. A journey must not invent a missing Execute, Verification, status, or recovery capability merely to fill one of these fields.

### 17.1.2 Journey Outcome Vocabulary

Use the strongest statement supported by authoritative evidence:

- **Informational / Observed** — the UI can show what is currently known.
- **Requested** — the user expressed intent to perform an operation.
- **Executed** — the applicable operation contract confirms execution/attempt according to its defined semantics.
- **Verified** — the applicable verification contract confirms the resulting state.
- **Unavailable / Not Applicable** — the journey or operation cannot be performed in the current context; do not present it as an error unless an actual failure occurred.

Complete, Resolved, Protected, or similar user-facing claims must not be inferred from a button click, dialog closure, progress animation, or mere navigation.


## 17.2 J01 — Discover, Install, and First Launch

Goal: understand what GST is and whether it is appropriate to start using.
Required UX: product identity, experimental-software disclosure, important-risk acknowledgement, distinction between information and legal agreement, clear Exit path.
Outcome: the user reaches GST only after the required acknowledgement gate is completed.
Verify: acknowledgement state is recorded/displayed according to the authoritative first-run contract.
Safe exit: the user can leave without unintentionally accepting the acknowledgement.

## 17.3 J02 — Initial Configuration / Safe Baseline

Goal: reach a usable state without needing specialist knowledge.
Required UX: Standard Profile is understandable, ordinary settings are purpose-oriented, Standard Profile Reset is easy to locate, Protection Status and Configuration Status are independent.
Example of a valid presentation state: Protection Status = 保護中 and Configuration Status = 標準設定.
Verify: the effective configuration and resulting Protection Status are presented independently according to authoritative state.
Custom configuration must never be presented as a protection failure solely because it differs from Standard.
Safe exit: leaving the setup flow does not silently apply or discard pending material changes.

## 17.4 J03 — Register an Existing Game

Goal: add a game and understand what GST will apply to it.
Required UX: identify game/target, explain effective protection in user language, provide route to game-specific settings, expose technical paths only where useful.
Verify: the game appears in Games only when the applicable registration/onboarding operation reports the corresponding outcome; the UI does not infer registration success from navigation alone.
Safe exit: cancellation leaves pre-existing registered-game state unchanged.

## 17.5 J04 — Portable Game / Safe Onboarding

Goal: add an unfamiliar ZIP/7z/RAR-distributed game without requiring archive-security expertise.
Required UX: drop target, destination visibility before material changes, extraction progress, validation result, executable selection, safe cancellation, clear distinction between no concerning evidence and proof of safety.
Verify: extraction/validation/registration results are reported separately from the absence of concerning evidence; no safety verdict is inferred from an incomplete scan or validation.
Safe exit: cancellation does not silently commit a partially prepared registration unless the authoritative operation contract defines such a state.

## 17.6 J05 — Everyday Game Launch

Goal: start playing with minimal friction while knowing the relevant protection context.
Required UX: selected game, launch mode, current configuration summary, known limitations, no unnecessary confirmation for ordinary low-risk actions.
Verify: launch outcome and, where supported by authoritative runtime observation, active-session state are shown separately.
Safe exit: leaving the Games screen does not change launch state; failed launch routes to an appropriate explanation/retry path.

## 17.7 J06 — Strict / High-Caution Launch

Goal: deliberately start a game under an approved more restrictive protection mode.
Required UX: explain what is stricter, communication/reporting scope, what is not isolated, and compatibility/account-risk considerations where applicable.
Never imply that Strict Launch is a sandbox or full process isolation unless that is actually implemented.
Verify: the active strict policy context is shown only from authoritative runtime/configuration state.
Safe exit: cancellation before launch leaves the prior state unchanged.

## 17.8 J07 — During Gameplay / Runtime Status

Goal: keep playing without unnecessary interruption while important security conditions remain visible.
Required UX: minimal runtime indication where approved, Smart Do Not Disturb for ordinary notifications, Critical security/data-protection notifications remain discoverable, no focus theft for ordinary notices, non-invasive overlay behavior.
Verify: only states supported by the applicable runtime/notification evidence are displayed; Critical notification visibility is not treated as proof that the underlying incident is resolved.
Safe exit: dismissing a non-critical notice does not alter the protected runtime state.

## 17.9 J08 — Routine Save Backup / Protection

Goal: know whether valuable save data has a recent usable backup.
Required UX: latest backup state, storage usage, pinned saves, progress/result, capacity or known restriction warnings, distinction between backup existence and verified recoverability where relevant.
Verify: backup existence, operation result, and recoverability/verification are shown as distinct facts where the authoritative contract exposes them.
Safe exit: cancelling a backup leaves existing backup generations intact unless the operation contract explicitly defines another outcome.

## 17.10 J09 — Restore / Undo

Goal: return data or configuration to a known earlier state after an unwanted change.
Required UX flow:
Select target → review current state → review proposed result → review impact → confirm → execute → verify.
No one-click shortcut may silently bypass confirmation for actual replacement.
Verify: final data/configuration state and verification result are reported separately.
Safe exit: Cancel/Back before Execute preserves the current target state and does not consume the selected generation implicitly.

## 17.11 J10 — Security / Integrity Event

Goal: understand a concerning event without being forced to interpret security jargon.
Representative triggers: Firewall block, suspicious MOD evidence, file-lock conflict, integrity/audit warning, WIPER safety event, unexpected file change, configuration corruption.
Required information flow:
What happened? → What is affected? → What is known? → What is unknown? → What can I safely do?
Display confirmed information, concerning signals, and unconfirmed items separately. Do not turn incomplete evidence into a definitive malware or safety verdict.

## 17.12 J11 — Firewall Block / Compatibility Problem

Goal: understand why communication was blocked and decide whether an approved configuration change is appropriate.
Required information: target game, target executable, communication direction, applied policy/rule, expected impact, currently available next action.
Required flow: block explanation → Review → Confirm when required → apply an existing approved change → reconcile/verify.
Confirm is required only where the applicable authority defines the operation as requiring confirmation.
Standard users should not be forced into raw rule CRUD.

## 17.13 J12 — File Lock / External Security Tool Conflict

Goal: understand why GST cannot complete a file operation.
Required UX: plain-language explanation first; process name/PID only as useful detail; uncertainty when attribution is not established; no implication that GST will terminate another security process; clear retry/wait/manual guidance according to implemented capability.
Verify: the UI distinguishes observed file-lock facts from unconfirmed attribution.
Safe exit: Retry/Wait/Manual guidance does not imply that GST terminated or modified another process unless that capability is explicitly authoritative.

## 17.14 J13 — Recovery Host / Startup Failure

Goal: recover when normal GST cannot safely continue.
Required UX: reason for diversion, preserved/at-risk data summary, safe inspection before action, explicit confirmation where the applicable recovery contract requires it, post-recovery verification, clear return-to-normal path.
Only currently approved and implemented recovery operations may be presented as actionable; unavailable or unimplemented paths remain non-actionable and truthful.
Recovery Host must remain understandable as a recovery environment, not an unexplained second GST application.

## 17.15 J14 — Configuration Change / Time Machine

Goal: customize GST while retaining a clear route back to a known configuration.
Required UX: purpose, Global vs GameProfile scope, current/proposed state, diff preview where defined, Standard vs Custom state, progressive disclosure of specialist controls.
Verify: the applied configuration state is distinguished from the user's draft/proposed state according to the authoritative settings contract.
Safe exit: cancelling a review does not apply the pending change; if a change has already executed, its result is not presented as Verified without verification evidence.

## 17.16 J15 — AI Assistance / Help / Privacy

Goal: obtain understandable guidance without unintentionally sending sensitive information.
Required flow: Prepare → Sanitize → Preview → optional Edit → re-sanitize/re-fingerprint → re-approve → Send.
Verify: the final outbound payload corresponds to the final approved preview; sending is distinct from receiving an answer.
The UI must separate GST facts/local evidence, AI-generated interpretation, and user-approved actions. AI advice must not silently become a configuration change.
Safe exit: leaving before Send does not transmit the prepared content.

## 17.17 J16 — Maintenance / Migration / Advanced Administration

Goal: perform infrequent specialist operations without exposing them during ordinary play.
Examples: maintenance, migration, detailed audit review, advanced firewall configuration, advanced cryptographic configuration, where these capabilities are already approved and available.
Required UX: progressive disclosure, explicit purpose/scope, specialist detail on demand, safe defaults, strong confirmation where the applicable authority requires it.
Verify: specialist settings are shown as effective/applied only when authoritative state supports that claim.
Expert means more control/detail/responsibility, not more protection.

## 17.18 J17 — Uninstall / Reversion of Supported Windows State

Goal: remove GST while understanding what happens to GST-owned configuration, user-owned backups, and only those Windows settings that the authoritative uninstall/reversion contract explicitly covers.
Required UX: distinguish application data from user-owned backup assets, summarize only the OS-setting reversion actually supported by the authoritative contract, explain the recovery safety net, preserve user control over backup assets, and separate Execute from post-action verification and unresolved manual actions.

# 17.19 Journey Gate Matrix

The following matrix is the review checklist for every journey above. N/A means the journey is informational/read-only at that stage; it does not authorize an operation.

| Journey | Start / Entry | Decide / Review | Execute | Verify | Safe Exit |
|---|---|---|---|---|---|
| J01 First Launch | App startup / first-run state | Required acknowledgement | N/A (acknowledgement is the gate) | Acknowledgement state according to authority | Leave without implicit acceptance |
| J02 Safe Baseline | First-run / Settings | Current vs Standard / proposed scope | Apply only if supported | Effective configuration + Protection Status according to authority | No silent apply/discard |
| J03 Register Game | Games / onboarding | Target + intended GST scope | Existing registration operation | Registration outcome + visible context | Cancel leaves prior registration unchanged |
| J04 Portable Onboarding | Games / drop or import flow | Destination + validation evidence + executable selection | Existing extraction/registration operation | Operation result + validation result | Safe cancel per extraction contract; no invented partial state |
| J05 Everyday Launch | Games / selected game | Launch mode + known limitations | Existing launch operation | Launch result / active session only from authoritative evidence | Leave screen without changing launch state |
| J06 Strict Launch | Games / selected game | Restrictiveness + compatibility impact | Existing strict launch operation | Active strict context from authoritative state | Pre-execute cancel preserves prior state |
| J07 Gameplay / Runtime | Active session | Interpret current runtime/notification state | Usually N/A from UI; only approved actions | Runtime/notification evidence, not dismissal | Non-critical dismissal does not resolve incident |
| J08 Backup | Save Data / game context | Target + destination/status | Existing backup operation | Backup result + verification/recoverability where defined | Existing generations remain protected on cancel |
| J09 Restore | Save Data / generation | Current state + proposed result + impact | Review → Confirm → Execute as required | Final state + verification | Pre-execute cancel preserves target state |
| J10 Security / Integrity Event | Notification / Protection / Incident | Known vs concerning vs unknown | Existing permitted response only | Resulting state according to authority | Do not infer threat resolution from dismissal |
| J11 Firewall Block | Protection / notification | Applied policy + expected impact | Existing approved change only | Reconcile / verify effective policy | No unapproved rule mutation |
| J12 File Lock | Operation failure / detail | Known lock facts + uncertainty | Usually Retry/Wait/Manual; no unapproved process control | File operation result if retried | No attribution or termination claim without evidence |
| J13 Recovery Host | Startup failure / recovery entry | Inspect + currently available approved recovery path | Only approved/implemented recovery operation | Recovery verification according to authority | Abort leaves system state unchanged where contract permits |
| J14 Configuration Change | Settings / contextual settings | Current vs proposed + scope + impact | Review → Confirm → Execute as required | Applied/effective state + verification where defined | Cancel leaves pending change unapplied |
| J15 AI Assistance | Relevant context / Help | Payload preview + privacy approval | Send only after final approval | Sent payload + answer are separate states | Leaving before Send transmits nothing |
| J16 Maintenance / Advanced | Settings / specialist entry | Purpose + scope + impact | Only approved/available specialist operation | Effective state / result from authority | Exit does not imply successful administration |
| J17 Uninstall / Reversion | Settings / Maintenance | Asset handling + supported OS reversion scope | Existing uninstall/reversion operation | Completion + supported reversion verification | Unresolved manual actions remain explicit |

Any row marked with an existing operation is subject to the applicable Feature / Module / Contract / Change Control authority. The matrix is a review aid, not an implementation checklist that creates missing capabilities.

### 17.19.1 Annotated Wireframe — User Journey / Review Gate Working Baseline

このAnnotated Wireframeは、全Journeyに共通する理解・判断・実行・検証・安全終了の関係を視覚化する作業基準である。Journey Inventory / Gate Matrixはレビュー用モデルであり、未承認のRuntime capabilityやOperationを生成する実装チェックリストではない。

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ GST / Journey Context                                                    │
├──────────────────────────────────────────────────────────────────────────┤
│ ① START / ENTRY                                                         │
│ Target / Scope / Origin                                                  │
│                              ↓                                          │
│ ② UNDERSTAND                                                            │
│ Current State · Impact · Known / Unknown · Available path                │
│                              ↓                                          │
│ ③ DECIDE / REVIEW                                                       │
│ Current → Proposed Result → Impact → Confirmation requirement            │
│                              ↓                                          │
│ ④ EXECUTE                                                               │
│ Existing approved operation only                                         │
│                              ↓                                          │
│ ⑤ OBSERVE / RECONCILE                                                   │
│ Operation result ↔ current / external state                             │
│                              ↓                                          │
│ ⑥ VERIFY                                                                │
│ Verified / Needs Attention / Failed / Unavailable / Not Applicable       │
│                              ↓                                          │
│ ⑦ SAFE EXIT / RETURN                                                    │
│ Back ≠ Cancel ≠ Close · Recovery when required                          │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 17.19.1.1 Gate Annotations

| ID | Gate | UI responsibility | Boundary |
|---|---|---|---|
| JG1 | Start / Entry | Entry condition / Origin / Targetを示す | Navigation到達だけでOperation開始済みとしない |
| JG2 | Understand | Current State / Scope / Impact / Known-Unknownを説明 | 未確認事項を原因確定へ変換しない |
| JG3 | Decide / Review | Proposed Resultと確認要件を確認 | Review表示だけでApproval済みとしない |
| JG4 | Execute | 既存承認済みOperationへ接続 | Journey表から新Operationを発明しない |
| JG5 | Observe / Reconcile | 実行結果と現在状態を照合 | Click / acknowledgementをVerifiedとしない |
| JG6 | Verify | 権威的Evidenceに基づき結果を表示 | Progress 100% / Completedを自動的にVerifiedへ昇格しない |
| JG7 | Safe Exit | Back / Cancel / Close / Recoveryを意味に応じて扱う | 退出だけで成功・Undo・Resolvedを意味させない |
| JG8 | Authority Gate | Feature / Module / Contract / Security boundaryを尊重 | Journey UIがAuthorityを上書きしない |

#### 17.19.1.2 Journey Class Variants

**Read-only / Informational**

`Start → Understand → View / Navigate → Safe Exit`

Execute / Confirmを追加せず、確認できる事実だけを表示する。

**Everyday Low-Risk Operation**

`Start → Understand → Decide → Execute → Verify → Safe Exit`

既存仕様が確認不要とする操作まで一律に長いConfirmへ変換しない。ただしRequested / Executed / Verifiedの意味は保持する。

**Material / High-Impact Operation**

`Start → Understand → Review → Confirm → Execute → Observe / Reconcile → Verify`

Restore、Firewall変更、設定リセット等は適用対象Contractの確認要件を優先し、Journey UIが独自の危険度や確認要件を作らない。

**Failure / Recovery**

`Detect / Failure → Known / Unknown → Preserve Current State → Authoritative Recovery Path → Verify`

Failure検知だけでRecovery capabilityを存在すると推測しない。ResolvedはUnderlying authoritative stateの確認を必要とする。

#### 17.19.1.3 Context / Approval Continuity

- Target、Game、Generation、Incident、Settings Scope等を引き継ぐ場合、現在も有効かを再確認する。
- 対象identity、Current State、入力、Fingerprint / Integrity条件が変化した場合、旧Review / Approvalを自動再利用しない。
- Search、Notification、Contextual Action Link、Tray Quick Accessから入ったJourneyも通常のReview / Execute / Verify境界を迂回しない。
- Originを表示し、現在どのJourney / Contextにいるかを失わせない。

#### 17.19.1.4 Outcome Presentation

| Outcome | Display meaning | Do not infer from |
|---|---|---|
| Informational / Observed | 現在確認できる事実 | Navigation / visual state alone |
| Requested | 操作意図が表明された | ClickだけでExecuted扱い |
| Executed | Operation contract上の実行 / 試行結果 | Request acknowledgementだけ |
| Verified | Verification contractで結果確認済み | Progress 100% / Completed alone |
| Needs Attention | 追加確認 / 未解決事項あり | Dismiss / Close |
| Failed | Operationが契約上失敗 | Empty / Unavailable |
| Unavailable | 現在Context / Build / Environmentで利用不能 | 実処理Failure |
| Not Applicable | 現在対象に該当しない | Global system error |

#### 17.19.1.5 Responsive / Accessibility Rules

- Wideの意味順（Context → Understand → Decide → Action → Result）をCompactでも維持する。
- 文字拡大時にもTarget / Scope / Impact / Review / Primary action / Safe Exitを優先して残す。
- Review中のConfirm対象とExecute actionの対応関係をreflow後も維持する。
- OutcomeはColorだけでなくText + Icon + Placementで区別する。
- 長い日本語・英語・Technical Detailsの折り返しでUnknown / Impact / Confirmation等を削除しない。
- Destructive actionがreflowによってPrimary actionのように見えないようにする。

#### 17.19.1.6 Journey / Implementation Boundary

Journey CoverageはImplementation Coverageを意味しない。Matrixに`Execute`欄があることだけを理由に、Runtime Feature、Public Contract、Domain Status、Storage behavior、Security Capabilityを追加しない。

未実装・未検証・現在Unavailableの経路は、その事実に応じてnon-actionable / unavailableとして扱い、Journey上の空欄を埋めるために仮のOperationを作らない。


# 18. Cross-Cutting User Needs

## 18.1 Orientation
Always answer: Where am I? What am I looking at? Which game/object does this apply to?
For high-impact journeys, also show the target and operation in the active review context.

## 18.2 Impact
Before material action: What changes? What is affected? Is it reversible? Which scope is affected (global, game, object, or operation-specific)?

## 18.3 Agency
Make clear what GST can do automatically, what requires confirmation, and what can be cancelled. A UI shortcut must not bypass an authoritative confirmation gate.

## 18.4 Recovery
When something goes wrong: What happened? What is preserved? What is the safest next step? What can be verified now, and what remains unresolved?

## 18.5 Evidence
Security-related conclusions should expose their basis as Confirmed / Concerning / Unknown where applicable. The UI must not strengthen an evidence state merely to complete a journey.

# 19. Journey-Level UX Gaps Identified

These are design questions, not automatic implementation defects.

### G01 — First-level navigation
Existing specifications contain many feature-specific surfaces. The final navigation needs one coherent user-facing taxonomy so users do not have to learn GST internal feature boundaries.

### G02 — Recovery entry points
Save restore, Quick Recovery, WIPER recovery, configuration rollback, Recovery Host, and OS-setting recovery arise in different contexts. They should share one recognizable recovery language even when their implementations differ.

### G03 — Normal vs unavailable vs failed
Planned, optional, disabled, unsupported-in-build, and failed states must not collapse into one generic disabled control.

### G04 — Protection information architecture
Runtime, network, file/data, audit/integrity, and optional protections have different meanings. The UI should expose purpose first and technical subsystem second.

### G05 — Error-to-action continuity
Every critical notification needs a deterministic route from notification → context → explanation → safe action → verification.

### G06 — User model vs implementation model
The UI must not mirror Domains, Ports, repositories, SQLite, workers, or infrastructure components. Those are architectural boundaries, not user navigation.

### G07 — Current-build truthfulness
Planned/manual feature descriptions must not cause the application to display unimplemented capability as available. Current-build availability must be explicit.

# 20. UX Review Priority

P0 — Main Window + First Run
P0 — Normal Game Journey
P0 — Failure / Notification / Recovery Journey
P1 — Save Backup / Restore Journey
P1 — Protection / Firewall Journey
P1 — Settings / Configuration Journey
P2 — AI / Privacy Journey
P2 — Advanced / Expert Journey
P2 — Visual Design System

The highest-priority UX work is the continuity between state → explanation → decision → action → recovery, not cosmetic styling.

# 21. Journey Inventory Acceptance Criteria

The UX model is ready for detailed screen/wireframe design when:
- the first-launch-to-gameplay journey is coherent;
- backup and recovery are reachable without knowledge of internal architecture;
- every critical failure has a defined user-facing entry point;
- every major journey has an explicit Start / Decide / Execute / Verify / Safe Exit review state, including N/A where no operation occurs;
- restore and other material changes share a recognizable confirmation model;
- Protection Status and Configuration Status remain independent;
- Normal / Advanced / Expert disclosure is consistent;
- privacy-sensitive operations show what will be sent before approval;
- current-build availability is truthful;
- terminology is consistent across screens;
- every major journey has a safe completion state and safe cancel/exit state.

---

## Extraction Note

The material was originally extracted from the former Section 17–21 scope of the parent UI working draft. The working-draft split changed document ownership, not product authority. Any future semantic change must be reviewed as a design change; the presence of a journey or matrix row in this file must not be treated as a newly approved runtime requirement.

End of Document
