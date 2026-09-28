# 01-09: UI Visual Design System

**Document ID:** GST-ARCH-UI-VISUAL-001  
**Version:** 0.2 (Configuration State Semantics / Cross-Document Alignment Review)  
**Status:** Design Working Draft — **Not an independent feature authority**  
**Authority Relationship:** Subordinate to GST-BASELINE-011, GST-BASELINE-047, approved module specifications, and Change Control decisions  
**Related Document:** [01-08_UI_Information_Architecture_and_Interaction_Concept.md](01-08_UI_Information_Architecture_and_Interaction_Concept.md)  
**Purpose:** Define GST-wide visual language, semantic design tokens, common component states, and visual safety/acceptance rules without creating or changing Runtime Features, Public Contracts, Domain Status, or Security capabilities.

---

# 0. Document Boundary and Traceability

This document is the extracted Visual Design working layer from `GST-ARCH-UI-CONCEPT-001`.

Extraction mapping from `01-08_UI_Information_Architecture_and_Interaction_Concept.md`:

| Previous section | Current section |
|---|---|
| 37. Visual Design Foundation | 1. Visual Design Foundation |
| 38. GST Design Token Baseline | 2. GST Design Token Baseline |
| 39. GST Common Component State Model | 3. GST Common Component State Model |
| 40. Visual Design Safety Rules | 4. Visual Design Safety Rules |
| 41. Visual Design Acceptance Criteria | 5. Visual Design Acceptance Criteria |

The extraction is organizational only. The substantive visual-design rules are preserved from the source document unless later changed through normal design review and change-control rules.

This document is a design companion to the formal UX baselines and the Presentation architecture. It does not authorize implementation by itself.

---

# 1. Visual Design Foundation

この章は、6画面IAに共通するVisual Languageを定義する。目的は装飾ではなく、**状態理解・文脈保持・誤操作防止・情報密度制御**である。

## 1.1 Visual Priority

```text
Page Context
  ↓
Primary State
  ↓
Primary Action
  ↓
Explanation / Impact
  ↓
Secondary Actions
  ↓
Technical Details
```

正常状態は落ち着いて表示し、注意状態だけを必要な範囲で強調する。常時警告画面化しない。

## 1.2 Surface Hierarchy

```text
Application Background
  └ Page Surface
      ├ Section Surface
      └ Status / Content
  └ Flyout / Dialog
```

強いshadow、枠線、背景色を重ねすぎない。Mica / Acrylicは可読性と状態認識を損なわない場合のみ採用候補とする。

## 1.3 Status Semantics

| Semantic | Meaning | Non-color cue |
|---|---|---|
| Success | 権威的に正常 / 完了と確認された結果の視覚表現 | text + icon |
| Attention | 一部制限 / 要確認 | text + icon |
| Critical | 保護停止 / 重大未解決 | text + icon + prominent placement |
| Info | 説明 / 補助情報 | text + icon |
| Neutral | 中立 / 未設定 | text |

色は補助情報であり、色だけで状態を判定させない。Protection Statusの正式語彙は既存Baselineを優先する。

## 1.4 Typography

```text
Page Title
  ↓
Section Heading
  ↓
Primary State
  ↓
Body / Explanation
  ↓
Label / Caption
  ↓
Technical Detail
```

日本語と英語の文字列長差を前提とし、固定幅や本文の過度な縮小で帳尻を合わせない。

## 1.5 Spacing / Radius

Spacingは4の倍数を基礎とし、主要間隔は8の倍数を優先する。

```text
4  micro
8  compact
12 related
16 standard
24 section
32 major section
48 major separation
```

RadiusはSmall / Medium / Large / Pillの意味Tokenへ分ける。PillはBadge等の小要素に限定する。

## 1.6 Iconography and Motion

- 同じ操作には同じIcon familyを使用する。
- Icon-only controlにはaccessible nameと必要な補助説明を持たせる。
- Critical / Warning / Infoはshapeとtextでも区別する。
- Motionは状態変化の理解を補助するために使い、装飾を目的にしない。
- Criticalの過度な点滅、常時loopする警告、Verify完了と誤認させるanimationを避ける。

# 2. GST Design Token Baseline

以下はsemantic design tokenの基準であり、Runtime ContractやPublic Contractの追加ではない。具体値はVisual QA / Accessibility QAで再検証できる。

## 2.1 Reference Color Tokens

### Light

| Token | Reference |
|---|---|
| Surface.Background | `#F7F7F7` |
| Surface.Primary | `#FFFFFF` |
| Surface.Secondary | `#F1F1F1` |
| Text.Primary | `#1F1F1F` |
| Text.Secondary | `#5F6368` |
| Border.Subtle | `#D6D6D6` |
| State.Success | `#107C10` |
| State.Attention | `#9A6700` |
| State.Critical | `#C42B1C` |
| State.Info | `#0067C5` |

### Dark

| Token | Reference |
|---|---|
| Surface.Background | `#1F1F1F` |
| Surface.Primary | `#292929` |
| Surface.Secondary | `#303030` |
| Text.Primary | `#F5F5F5` |
| Text.Secondary | `#C8C8C8` |
| Border.Subtle | `#4A4A4A` |
| State.Success | `#6CCB5F` |
| State.Attention | `#F2C94C` |
| State.Critical | `#FF6B5F` |
| State.Info | `#5EB6FF` |

色値は大面積の塗りつぶしへ無条件適用するものではなく、text / icon / border等の意味に応じて使用する。

## 2.2 Typography Tokens

```text
Type.PageTitle
Type.SectionHeading
Type.PrimaryState
Type.Body
Type.BodyStrong
Type.Label
Type.Caption
Type.Technical
```

Font familyとfallback chainはWindows実機とアクセシビリティ条件を確認してから実装値を確定する。

## 2.3 Layout Tokens

```text
Space.1 = 4
Space.2 = 8
Space.3 = 12
Space.4 = 16
Space.5 = 20
Space.6 = 24
Space.7 = 32
Space.8 = 40
Space.9 = 48

Radius.Small
Radius.Medium
Radius.Large
Radius.Pill

ControlHeight.Compact
ControlHeight.Standard
ControlHeight.Large

Layout.PagePadding
Layout.ContentMaxWidth
Layout.SectionGap
Layout.CardGap
Layout.DialogPadding
```

実装上の具体寸法は、Windows表示スケール・大きい文字・最小Windowでの実機確認後に変換する。

## 2.4 Motion Tokens

```text
Motion.Fast
Motion.Standard
Motion.Slow
Motion.Reduced
```

Motion tokenとRuntime timeout / retry / cooldownは別物であり、UI animation値からRuntime timingを決めない。

## 2.5 Token Ownership Rule

Design Tokenの変更によってProtection Status、Configuration Status、Review / Confirm / Execute / Verify、Back / Cancel / Close、Recovery safety gate、Requested / Executed / Verifiedの意味を変更しない。

Visual Tokenは意味そのものではなく、意味の表現手段である。

# 3. GST Common Component State Model

## 3.1 Global Interactive States

```text
Default → Hover → Focus → Pressed → Disabled
                         └→ Loading / Busy
```

状態はvisualだけでなくkeyboard interactionとaccessible stateでも一致させる。

## 3.2 Navigation Item

`Default` / `Hover` / `Focus` / `Selected` / `Disabled` を共通状態とする。Selectedは現在表示中を意味し、Security Statusを意味しない。

## 3.3 Button

Roles: `Primary` / `Secondary` / `Destructive` / `Subtle`

Busy中は二重実行を防止する。Destructive styleは安全確認を省略してよいことを意味しない。

## 3.4 Status Card

`Normal` / `Attention` / `Critical` / `Unavailable` / `Not Applicable` / `Loading` / `Error` を表現できる。UnavailableとError、Not ApplicableとFailureを同一視しない。

## 3.5 Needs Attention Card

`Present` / `Resolving` / `Resolved` / `Critical`。AcknowledgedやDismissはUnderlying stateを自動的にResolvedへ変更しない。

## 3.6 Search

`Empty` / `Focused` / `Query Entered` / `Searching` / `Results` / `No Results` / `Error`。SearchはDiscovery / Navigationであり、High-impact Executeを直接開始しない。

## 3.7 Settings Control

`Standard` / `Customized` / `Draft Changed` / `Pending Review` / `Applying` / `Applied` / `Verified` / `Unavailable`。`Customized` は実効設定がStandardと異なる状態、`Draft Changed` は未適用の提案変更を示す。`Applied` は適用完了を示すが、Verificationが別途必要な場合はVerifiedを意味しない。高影響設定では表示上の変更と実際の適用を分離する。

## 3.8 Confirmation Dialog

`Review` / `Confirm-ready` / `Executing` / `Verification` / `Completed` / `Failed` / `Cancelled`。

Executing中にConfirmを再実行しない。CompletedとVerifiedを必要に応じて別表示する。Cancelは確認/操作フローの中止であり、完了済み変更のUndoではない。

## 3.9 Progress / Notification

ProgressはCurrent Step / Progress State / Cancellation / Result / Verificationを扱う。100%は自動的にSuccessを意味しない。

NotificationはCritical / Important / Informationalを使用し、AcknowledgedとResolvedを分ける。Critical notificationを閉じてもUnderlying Critical stateは消えない。

## 3.10 Data and Page States

`Empty` / `Loading` / `Partial` / `Error` / `Unavailable` を共通Page Stateとする。

Emptyはデータなし、Loadingは取得中、Partialは一部取得、Errorは処理失敗、Unavailableは現在のBuild / Environment / Contextで利用不能を意味する。

# 4. Visual Design Safety Rules

1. 色・Icon・Animationだけで新しいSecurity Statusを発明しない。
2. Visual componentからInfrastructure / OS / DBへ直接アクセスしない。
3. responsive reflowを理由にTarget / Impact / Confirmation情報を削除しない。
4. compact layoutを理由にReview / Confirm / Verifyを短縮しない。
5. Advanced / Expert stylingを「より安全なモード」と誤認させない。
6. 未実装 / 未検証機能をplaceholderだけで利用可能に見せない。
7. Design Token変更をRuntime Contract変更へ自動変換しない。

# 5. Visual Design Acceptance Criteria

- Light / Dark双方で主要状態の意味が維持される。
- 色、Icon、Textの複数チャネルで状態を理解できる。
- 文字拡大・Windows表示スケール変更で重要本文とConfirm / Cancelが失われない。
- Normal / Attention / Critical / Unavailable / Errorが視覚的に区別される。
- Selected / Modified / Protectedの意味が混同されない。
- Progress完了とVerification Successを混同しない。
- Draft Changed / Applied / Verifiedを、同じ意味の状態として表示しない。
- Critical状態が通常通知へ縮退しない。
- Audit / Firewall / Recoveryの詳細が文字縮小だけで詰め込まれない。
- Keyboard focusが確認できる。
- UI設計だけを理由に新しいRuntime Feature / Public Contract / Domain Status / Security capabilityが追加されていない。

---

# 6. Design Continuation

The Visual Design System is now the common visual foundation for the subordinate UI Design Working-Draft set:

- 01-08_UI_Information_Architecture_and_Interaction_Concept.md — global IA / interaction model
- 01-10_UI_Home_and_Dashboard_Design.md — Home / Dashboard
- 01-11_UI_Games_Design.md — Games
- 01-12_UI_Save_Data_Design.md — Save Data
- 01-13_UI_Protection_Design.md — Protection
- 01-14_UI_Recovery_Design.md — Recovery
- 01-15_UI_Settings_Design.md — Settings
- 01-16_UI_Common_Interaction_and_Display_Design.md — common interaction / display
- 01-17_UI_Reference_Patterns_and_Common_Component_Library.md — common UI patterns
- 01-18_UI_User_Journeys_and_Review_Model.md — journeys / review model
- 01-19_UI_Cross-Screen_Flow_and_Transition_Model.md — cross-screen flow / mapping
- 01-20_UI_End-to-End_Scenario_Validation.md — scenario validation

These are subordinate design documents. They remain descriptive and must not be interpreted as authorization to add Runtime Features, Public Contracts, Domain Status, or Security capabilities.

End of Document