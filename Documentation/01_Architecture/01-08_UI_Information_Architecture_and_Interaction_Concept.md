# 01-08: UI Information Architecture and Interaction Concept

**Document ID:** GST-ARCH-UI-CONCEPT-001  
**Version:** 0.7 (UI Working-Draft Final Consistency Gate)
**Status:** Design Working Draft — Not an independent feature authority  
**Authority Relationship:** Subordinate to GST-BASELINE-011, GST-BASELINE-047, approved module specifications, approved feature specifications, and Change Control decisions  
**Related Documents:**  
- [01-09_UI_Visual_Design_System.md](01-09_UI_Visual_Design_System.md)  
- [01-10_UI_Home_and_Dashboard_Design.md](01-10_UI_Home_and_Dashboard_Design.md)  
- [01-11_UI_Games_Design.md](01-11_UI_Games_Design.md)  
- [01-12_UI_Save_Data_Design.md](01-12_UI_Save_Data_Design.md)  
- [01-13_UI_Protection_Design.md](01-13_UI_Protection_Design.md)  
- [01-14_UI_Recovery_Design.md](01-14_UI_Recovery_Design.md)  
- [01-15_UI_Settings_Design.md](01-15_UI_Settings_Design.md)  
- [01-16_UI_Common_Interaction_and_Display_Design.md](01-16_UI_Common_Interaction_and_Display_Design.md)  
- [01-17_UI_Reference_Patterns_and_Common_Component_Library.md](01-17_UI_Reference_Patterns_and_Common_Component_Library.md)  
- [01-18_UI_User_Journeys_and_Review_Model.md](01-18_UI_User_Journeys_and_Review_Model.md)  
- [01-19_UI_Cross-Screen_Flow_and_Transition_Model.md](01-19_UI_Cross-Screen_Flow_and_Transition_Model.md)  
- [01-20_UI_End-to-End_Scenario_Validation.md](01-20_UI_End-to-End_Scenario_Validation.md)  
**Purpose:** Define GST-wide information architecture and interaction model. Screen-specific organization, common interaction/display, reusable patterns, journeys, cross-screen flows, and end-to-end scenario validation are maintained in the subordinate 01-10〜01-20 working-draft set. Visual-system details remain centralized in 01-09.

---

# 0. Design-First Pause Context

2026-09-28、Production runtime implementation is temporarily paused by explicit user direction so that the GST user experience can be designed as a coherent system before additional Presentation implementation.

During this design pause:

- UI information architecture, navigation, terminology, state presentation, interaction patterns, recovery flows, accessibility, and visual-design principles may be refined.
- Existing product behavior is not silently changed.
- New runtime capabilities, new public Contracts, or feature expansion must not be introduced merely to realize a visual idea.
- Existing UX and security decisions remain authoritative.
- This document is an **interpretation and composition layer**, not a replacement for those specifications.

---

## 0.1 Working-Draft Document Set Structure

The UI Design Working Draft is intentionally distributed by responsibility so that an implementation agent can retrieve only the relevant scope:

| Document | Responsibility |
|---|---|
| 01-08 | Global IA / interaction principles and application-wide structure |
| 01-09 | Visual Foundation / Design Tokens / common visual states |
| 01-10 | Home / Dashboard |
| 01-11 | Games |
| 01-12 | Save Data |
| 01-13 | Protection |
| 01-14 | Recovery |
| 01-15 | Settings |
| 01-16 | Common interaction / display |
| 01-17 | Reference patterns / common component library |
| 01-18 | User journeys / review model |
| 01-19 | Cross-screen flow / transition / responsibility mapping |
| 01-20 | End-to-end scenario validation |

The split is organizational only. Authority remains with the existing Formal Baselines, approved Feature / Module Specifications, Contracts, Security boundaries, and Change Control.

---

# 1. UI Product Identity

GST should feel like a **game security control center that helps the user make safe decisions**, not like a conventional antivirus dashboard and not like a developer diagnostic console.

The user-facing experience should answer four questions in order:

1. **今どうなっている？**
2. **自分のゲームやデータに何が関係している？**
3. **問題があるなら、何をすればよい？**
4. **その操作で何が変わり、どう戻せる？**

Core interaction principles:

```
Understand → Decide → Confirm → Execute → Verify
```

For normal daily use, the UI should minimize technical terminology while retaining a deliberate path to detailed evidence and expert controls.

---

# 2. Application Shell

The Main Window is organized into five persistent regions.

```
┌──────────────────────────────────────────────────────────────────────┐
│ GST / Current Context                  Global Search   Help           │
├───────────────┬──────────────────────────────────────────────────────┤
│               │                                                      │
│  HOME         │                 PAGE CONTENT                         │
│  GAMES        │                                                      │
│  SAVE DATA    │                                                      │
│  PROTECTION   │                                                      │
│  RECOVERY     │                                                      │
│               │                                                      │
│  SETTINGS     │                                                      │
│               │                                                      │
│  ───────────  │                                                      │
│  Advanced     │                                                      │
│  / Expert     │                                                      │
│               │                                                      │
├───────────────┴──────────────────────────────────────────────────────┤
│ Runtime / Protection context · notifications · version / status      │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.1 Region responsibilities

**Header**

- GST identity
- current context / selected game when applicable
- Global Search
- Help / manual access
- non-critical application-level actions when applicable

**Primary Navigation**

The navigation is task-oriented rather than API-oriented:

- **Home**
- **Games**
- **Save Data**
- **Protection**
- **Recovery**
- **Settings**

The navigation must not expose internal implementation concepts such as SQLite, Ports, repositories, workers, or Win32 components.

**Advanced / Expert access**

Advanced / Expert controls are reached through explicit progressive disclosure. They do not become a second, competing application shell.

**Content region**

The content region shows the selected task and its current state. It must not require the user to understand internal subsystem boundaries before taking ordinary actions.

**Status footer / context strip**

This area is for compact contextual information only. Critical security/data-protection conditions must use the notification rules defined by the UX baseline and must not be hidden in a footer.

---

# 3. Main Navigation Model

## 3.1 Home

**Purpose:** answer "Is GST currently functioning as expected, and what matters right now?"

Home is the default landing page.

Primary content hierarchy:

```
1. Protection Status
2. Configuration Status
3. Current / Recent Game Context
4. Important unresolved events
5. Quick Recovery
6. Backup / Save Data health
7. Recent security activity
```

Home must not become a numerical "overall security score" screen.

## 3.2 Games

**Purpose:** manage registered games and launch modes.

Primary actions:

- select a game
- launch using an available, implemented launch mode
- view the game's protection configuration
- inspect game-specific status
- begin safe onboarding where supported by the current build

Game-specific security state belongs here; system-wide security state remains on Home / Protection.

## 3.3 Save Data

**Purpose:** protect, inspect, and recover user save data.

Primary areas:

- backup history
- storage use
- pinned important saves
- restore history
- time-machine style configuration recovery when applicable
- external storage/export operations where implemented

Every destructive or replacement-capable restore follows the established `Review → Confirm → Execute` contract.

## 3.4 Protection

**Purpose:** explain what protection mechanisms are active and what they are currently doing.

This page should organize by user-relevant protection purpose rather than implementation component.

Suggested groups:

- Game / runtime protection
- Network / Firewall protection
- File and data protection
- Integrity / audit protection
- optional advanced protections

The page must distinguish **Security Foundation** from **Runtime Protection** and must never imply runtime protection continues while GST is not running when it does not.

## 3.5 Recovery

**Purpose:** provide a single understandable place for recovery work.

Examples:

- quick recovery entry
- incomplete restore recovery state
- WIPER recovery state where applicable
- Recovery Host handoff
- failed operation inspection
- manual recovery guidance

Recovery is not an error archive. The page should lead with the action that safely resolves the present condition.

## 3.6 Settings

**Purpose:** configure user-owned behavior and expose advanced controls through progressive disclosure.

The first screen should favor:

- Standard Profile
- notifications
- UI preferences
- ordinary game/security preferences
- backup policy

Advanced / Expert sections may expose specialist controls such as detailed firewall configuration, migration, diagnostics, KDF-related settings, and audit detail where already specified.

---

# 5. State Presentation Model

GST should consistently use a three-layer presentation model.

```
State
  ↓
Explanation
  ↓
Available Action
```

For a degraded or failed condition:

```
What happened?
Why does it matter?
What can I do now?
What will change?
Can I recover?
```

Technical diagnostics may be available in a secondary "Details" area, but the primary screen should not require the user to decode raw exception messages, HRESULT values, process IDs, or database states.

---

# 6. Action Hierarchy

All UI actions fall into four practical classes.

| Class | Example | UX pattern |
|---|---|---|
| Informational | View event details | immediate |
| Reversible / low-risk | Change a display preference | direct apply where defined |
| Material change | Change protection policy | Review → Confirm → Execute |
| High-risk / destructive | Restore, delete, OS/security change | explicit impact review → Confirm → Execute → Verify |

The classification must follow the authoritative feature specification for each operation.

---

# 7. Normal / Advanced / Expert Disclosure

## Normal

The user should be able to operate GST for ordinary daily tasks without opening specialist controls.

```
Games
Save Data
Protection Status
Recovery
Recommended Settings
```

## Advanced

Detailed configuration and evidence are exposed when a user intentionally requests deeper control.

```
Game-specific security settings
Detailed backup policy
Protection behavior settings
Expanded event details
```

## Expert

Specialist operations that can materially affect system behavior or require domain knowledge.

```
Detailed Firewall rule management
Migration / maintenance
Deep diagnostics
Detailed audit information
Advanced cryptographic configuration
```

"Expert" must never mean "more protected". It means "more control / more detail / more responsibility".

---

# 8. Notification Hierarchy

Notifications are grouped into:

```
Critical
Important
Informational
```

Critical security/data-protection notifications must not be fully suppressed by gameplay Do Not Disturb.

Normal informational notifications may be condensed during gameplay according to the approved Smart Do Not Disturb rules.

The notification itself should be actionable or explain where the user can review it.

---

# 9. Recovery Interaction Pattern

Recovery UI must maintain one recognizable language across:

- Save restore
- Quarantine restore
- Configuration rollback
- OS setting restoration
- other future approved recovery actions

Common pattern:

```
[1] Identify target
[2] Show current state
[3] Show proposed result
[4] Show important impact
[5] User confirms
[6] Execute
[7] Verify result
[8] Report final state
```

A "Cancel" path must leave the protected current state unchanged unless the authoritative operation contract explicitly defines another outcome.

---

# 10. Privacy Presentation

Privacy-sensitive screens must prefer **purpose-first disclosure**.

Do:

```
送信対象
マスキング済み
個人情報を含む可能性のある項目は伏せています
[内容を確認]
```

Do not expose raw secrets, tokens, or unnecessary personal paths simply because an internal diagnostic exists.

For AI-related flows:

```
Prepare
→ Sanitize
→ Preview
→ User Edit (optional)
→ Re-sanitize / re-fingerprint
→ Re-approve
→ Send
```

---

# 11. Accessibility Baseline

The application-wide UI concept assumes:

- keyboard navigation for every primary operation
- visible focus indication
- readable text at user-selected scaling
- sufficient contrast and non-color-only state communication
- meaningful accessible names for controls
- status conveyed through text/icons, not color alone
- dialogs must remain understandable without animation
- important recovery and security actions must be reachable without precision pointing

Detailed accessibility acceptance criteria remain a later UX validation task.

---

# 12. Visual Design Direction — Separated Working Layer

Detailed Visual Foundation, Design Tokens, Common Component States, Visual Safety Rules, and Visual Acceptance Criteria are maintained in the companion document [01-09_UI_Visual_Design_System.md](01-09_UI_Visual_Design_System.md).

This document retains only the **IA / Interaction boundary** for visual design: visual treatment must support the established information hierarchy, interaction safety, accessibility, and state semantics without changing their underlying meaning.

Final concrete WPF resource values, theme application, icon selection, animation timing, and Windows accessibility validation remain implementation/QA concerns governed by the companion document and applicable baselines.

---

# 13. Implementation Boundary During Design Pause

During this design-first period, a design decision does **not** automatically authorize implementation.

Before runtime implementation resumes, each approved UI decision should be mapped to:

```
UI Surface
   ↓
User State
   ↓
Existing Contract / Use Case
   ↓
Required DTO / ViewModel state
   ↓
Security / UX validation
   ↓
Implementation task
```

If a proposed UI needs a capability that does not exist in the approved product contracts, that gap must be handled as a design / requirements issue first rather than silently inventing an interface.

---

# 14. Initial Main Window Wireframe

```
┌───────────────────────────────────────────────────────────────────────────────┐
│ 🛡️ GameSecurityTool                [ 🔎 検索… ]   [? ヘルプ]                 │
├────────────────┬──────────────────────────────────────────────────────────────┤
│                │  ホーム                                                       │
│ 🏠 ホーム      │                                                               │
│ 🎮 ゲーム      │  ┌──────────────────────────┐  ┌──────────────────────────┐ │
│ 💾 セーブ      │  │ 保護状態                 │  │ 設定状態                 │ │
│ 🛡️ 保護       │  │ 🟢 保護中                │  │ 標準設定                 │ │
│ ↩ 復旧        │  │                          │  │                          │ │
│ ⚙ 設定        │  └──────────────────────────┘  └──────────────────────────┘ │
│                │                                                               │
│ ───────────── │  🎮 現在のゲーム                                               │
│ 詳細設定       │  ┌────────────────────────────────────────────────────────┐ │
│                │  │ ゲーム名 / 状態 / 現在適用中の保護                       │ │
│ Advanced       │  │                                      [ゲームを表示]       │ │
│ Expert         │  └────────────────────────────────────────────────────────┘ │
│                │                                                               │
│                │  ⚠ 重要なお知らせ / 要対応事項                              │
│                │  ┌────────────────────────────────────────────────────────┐ │
│                │  │ 復旧が必要な項目がある場合だけ表示                       │ │
│                │  │                                      [確認する]          │ │
│                │  └────────────────────────────────────────────────────────┘ │
│                │                                                               │
│                │  ↩ クイックリカバリ      💾 セーブデータ                    │
│                │  [復旧レビューへ]         [バックアップ状態を見る]            │
│                │                                                               │
│                │  最近の保護イベント / 状態変化                               │
│                │  ────────────────────────────────────────────────────────   │
├────────────────┴──────────────────────────────────────────────────────────────┤
│ GST実行状態 / 保護コンテキスト / 通知状態 / バージョン                        │
└───────────────────────────────────────────────────────────────────────────────┘
```

This wireframe is an information-architecture reference, not a final visual specification.

---

# 15. Design Questions Reserved for the Next UI Review

The earlier open question about whether **Recovery** should be a first-level navigation item has been resolved by the current working navigation decision in Section 22: **Recovery remains a first-level screen**.

The remaining questions are:

1. Whether **Protection** should be one page with purpose-based groups or separate subpages for Network / File / Runtime / Integrity.
2. How **current game context** behaves when no game is running.
3. How to distinguish **"nothing to do"** from **"not currently available / not implemented in this build"** without confusing users.
4. Whether the left navigation remains persistent at compact window sizes.
5. The exact wording and visual treatment of the Security Foundation / Runtime Protection boundary.

These remain design questions only; resolving them does not authorize runtime implementation.


---

# 16. Acceptance Criteria for This Concept Layer

Before the UI concept is considered sufficiently stable for Presentation implementation:

- every first-level screen has a clear user purpose;
- Home answers the user's current protection/state question without an aggregate security score;
- Protection Status and Configuration Status remain independent;
- recovery actions use a consistent confirmation model;
- critical notifications cannot be hidden by ordinary gameplay DND rules;
- Normal / Advanced / Expert disclosure is understandable;
- privacy-sensitive information is never exposed merely for technical convenience;
- current-build availability can be represented without pretending planned features are implemented;
- the design does not require undocumented Public Contracts;
- the visual layer can later change without changing the underlying safety workflow.

---

# 22. Main Navigation Design Decision — Working Baseline

この章は17章のユーザージャーニーをMain Windowの情報構造へ変換するための作業上の基準案である。最終Visual DesignやPresentation実装を直接承認するものではない。

## 22.1 First-Level Navigation

第1階層は次の6項目を基本構成とする。

🏠 ホーム
🎮 ゲーム
💾 セーブデータ
🛡️ 保護
↩ 復旧
⚙ 設定

理由:

- 6項目はGSTの主要なユーザー目的を直接表現できる。
- 「復旧」を独立項目にすることで、異常発生時に「どこへ行けば戻せるか」をユーザーが学習しやすい。
- 「保護」を独立項目にすることで、現在の保護機能を確認する場所と、個別ゲーム操作の場所を分離できる。
- 「設定」を最後に置くことで、日常操作を設定画面中心にしない。
- Advanced / Expert は第1階層の別アプリとして扱わず、各画面のProgressive DisclosureとSettings内の専門領域として扱う。

### 22.1.1 Navigation Rule

第1階層の項目名は内部コンポーネント名ではなくユーザーの目的を表す。

禁止例: Database / Workers / Firewall Engine / Audit Repository / WIPER Core / Runtime Services

これらはユーザー向けナビゲーション名称に使用しない。

## 22.2 Home — 「今どうなっているか」

HomeはGSTの判断起点である。

ユーザーがHomeを開いたとき、最初に理解できるものは、1) GSTの現在状態、2) Protection Status、3) Configuration Status、4) 現在のゲーム文脈、5) 要対応事項、6) Recoveryへの入口、7) 最近の重要な状態変化とする。

Homeはすべての情報を集約する場所ではなく、「今見るべきもの」を集約する場所とする。

## 22.3 Games — 「何を遊ぶか」

Gamesはゲーム単位の操作の中心とする。

Games → Registered Games / Add-Onboard / Launch / Game Status / Game-specific Settings

ゲーム固有の設定や起動方式はGamesに置く。Windows全体に関係する保護状態はProtectionへ誘導する。

## 22.4 Save Data — 「データを守る」

Save Dataはセーブデータ保護を一つの目的としてまとめる。

Save Data → Backup / History・Generations / Storage / Pin・Tag / Restore / Export

セーブデータ復元はSave Dataから開始できるが、実際の危険操作は共通のRecovery Interaction Patternを使用する。

「Save Data」は何のデータを扱うか、「Recovery」は問題を安全にどう解決するか、という役割分担とする。

## 22.5 Protection — 「何が守られているか」

Protectionは機能一覧ではなく保護目的の一覧として設計する。

Protection → Runtime・Game / Network / File・Data / Integrity・Audit / Optional・Advanced

各項目では原則として、目的 → 現在状態 → 対象 → 説明 → 必要なら詳細、の順に表示する。

実装用語を初期表示へ持ち込まない。

## 22.6 Recovery — 「問題から安全に戻る」

RecoveryはGST固有の重要領域として第1階層に残す。

理由:
1. 異常時には通常画面を探す余裕がない。
2. 復元・ロールバックはGSTの主要価値の一つである。
3. 複数機能からRecoveryが発生しても、ユーザーが同じ概念として理解できる。
4. Recovery Hostへの引き継ぎを含め、「正常状態へ戻る」ための共通入口を提供できる。

Recovery内では Needs Attention → Inspect → Review → Confirm → Execute → Verify → Resolved という共通言語を用いる。ここで `Resolved` は、通知を閉じたこと、画面遷移したこと、Executeが完了したことではなく、適用対象の権威的な状態・検証結果が解決を支持する場合に限って表示する。

Recoveryは単なるエラー一覧やAudit Timelineの代替にはしない。

## 22.7 Settings — 「GSTを自分に合わせる」

Settingsは通常利用の中心にしない。

第1画面ではRecommended・Standard、Notifications、UI Preferences、Backup Preferences、Ordinary Protection Preferencesを優先する。

詳細Firewall rule、Migration、Maintenance、Diagnostics、Cryptographic configuration等は、承認済み仕様に従ってAdvanced / Expertへ段階的に開示する。

## 22.8 Advanced / Expert is a Disclosure Layer

Advanced / Expertは第1階層の独立ナビゲーションではない。

Normal → Advanced → Expert は、同じ目的に対して表示する情報量と制御量を増やす仕組みとする。

Advanced ≠ more protected
Expert ≠ more secure

Expertは「より高い保護レベル」ではなく「より多くの制御・詳細・責任」を意味する。

## 22.9 Navigation by User Situation

| User Situation | Primary Entry | Secondary Destination |
|---|---|---|
| GSTを開いて状態を確認 | Home | Protection |
| ゲームを遊びたい | Games | Home |
| 新しいゲームを追加 | Games | Onboarding |
| セーブを守りたい | Save Data | Home |
| 何かおかしい | Home | Recovery / Protection |
| 通信が止まった | Home notification | Protection |
| データを戻したい | Save Data / Home | Recovery |
| 設定を変更したい | Settings | relevant feature page |
| 高度な調査をしたい | relevant page | Advanced / Expert |
| GSTが正常起動できない | Recovery path | Recovery Host |

## 22.10 Contextual Navigation Rule

第1階層ナビゲーションに加え、現在の対象を維持するContextual Navigationを設計原則とする。

例: Games → Game A → Protection → Network → Review

この場合、ユーザーがどのゲームについて操作しているかを失わない。

ゲーム、セーブ、復旧、Firewall等のサブ画面では、可能な限りCurrent Game / Current Object / Current Operationを画面上部または同等の視覚階層で示す。

## 22.11 Global Search and Help

Global SearchとHelpは横断機能としてHeaderに置き、第1階層の主要目的には含めない。

検索結果は必要に応じてSetting / Feature / Manual / Recovery Guidanceを同時に提示する。

## 22.12 Navigation Safety Rules

1. 危険操作へ直接ジャンプするショートカットを作っても、認証・確認フローを短絡させない。
2. Quick RecoveryはRecovery Reviewへの入口であり、直接Executeではない。
3. Critical状態を通常ナビゲーションの奥へ隠さない。
4. 現在の対象ゲーム・対象データ・対象操作を失わせない。
5. 戻る操作によって未確認の重要な変更を暗黙に確定しない。
6. 画面遷移だけで設定やセキュリティ状態を変更しない。
7. 未実装・未対応機能を通常ナビゲーション上で利用可能と誤認させない。

## 22.13 Proposed Main Window Structure

Header: GST identity / Current Context / Global Search / Help
Primary Navigation: Home / Games / Save Data / Protection / Recovery / Settings
Disclosure: Advanced / Expert controls exposed contextually rather than as a mandatory second shell
Content: current task and state
Footer/context strip: compact runtime/context information only; critical alerts use the approved notification path

## 22.14 Navigation Decision Review Result

現時点のWorking Baselineでは、Home / Games / Save Data / Protection / Recovery / Settingsの6項目を維持する。

役割:
- Home = 今どうなっているか
- Games = 何を遊ぶか
- Save Data = 何を守るか
- Protection = 何が守られているか
- Recovery = 問題からどう戻るか
- Settings = GSTをどう構成するか

この役割分担が崩れない限り、将来の機能追加でナビゲーションが肥大化しても、既存のユーザーmental modelを維持しやすい。

## 22.15 Next UX Design Target

次に整理すべき対象はHome画面そのものとする。

Home → Protection Status / Configuration Status → Current Game Context → Needs Attention → Quick Recovery → Save Data Summary → Recent Activity

この順序を検討・確定した後、Games / Save Data / Protection / Recovery / Settingsへ展開する。

# 37. UI Design Working-Draft Set Boundary

The detailed UI design material is now distributed across the subordinate Design Working-Draft set 01-08 through 01-20.

- 01-08 — global Information Architecture and Interaction Concept.
- 01-09 — Visual Design System.
- 01-10〜01-15 — screen-specific design.
- 01-16〜01-17 — common interaction/display and reusable UI patterns.
- 01-18 — User Journeys and Review Model.
- 01-19 — Cross-Screen Flow, Transition, and responsibility mapping.
- 01-20 — End-to-End Scenario Validation.

These documents remain subordinate design artifacts. They do not replace Formal Baselines, Feature Specifications, Contracts, Security boundaries, or Change Control decisions.

# 42. UI Design Working-Draft Set Continuation

The six-screen UI design work is now maintained as a document set rather than a single Annotated Wireframe file.

1. 01-08_UI_Information_Architecture_and_Interaction_Concept.md
2. 01-09_UI_Visual_Design_System.md
3. 01-10_UI_Home_and_Dashboard_Design.md
4. 01-11_UI_Games_Design.md
5. 01-12_UI_Save_Data_Design.md
6. 01-13_UI_Protection_Design.md
7. 01-14_UI_Recovery_Design.md
8. 01-15_UI_Settings_Design.md
9. 01-16_UI_Common_Interaction_and_Display_Design.md
10. 01-17_UI_Reference_Patterns_and_Common_Component_Library.md
11. 01-18_UI_User_Journeys_and_Review_Model.md
12. 01-19_UI_Cross-Screen_Flow_and_Transition_Model.md
13. 01-20_UI_End-to-End_Scenario_Validation.md

The set supports future Annotated Wireframe / presentation-design work while preserving the rule that design artifacts do not, by themselves, approve Runtime implementation.

End of Document
