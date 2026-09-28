# 01-13: UI Protection Design

**Document ID:** GST-ARCH-UI-PROTECTION-001  
**Version:** 0.4 (Protection Annotated Wireframe / Visual UX Review)  
**Status:** Design Working Draft — Not an independent feature authority  
**Authority Relationship:** Subordinate to GST-BASELINE-011, GST-BASELINE-047, applicable approved module / feature specifications, and Change Control decisions  
**Parent Concept:** [01-08_UI_Information_Architecture_and_Interaction_Concept.md](01-08_UI_Information_Architecture_and_Interaction_Concept.md)  
**Historical Source Mapping:** Former Section 32 of the parent 01-08 UI working draft  
**Current Ownership:** This document is the maintained location for the Protection material after the working-draft split.
**Purpose:** Organize existing UI information-architecture and interaction design for this scope without expanding Runtime Features, Public Contracts, Domain Status, Security capabilities, or Implementation Readiness.

## Document Boundary

This document preserves UI design material that was formerly organized as Section 32 in the parent 01-08 UI working draft. After the working-draft split, 01-08 retains global IA / interaction principles while this document owns the Protection material.

The preserved section numbering is historical traceability only. It does not imply that those sections still exist in 01-08.

This document does not authorize Runtime implementation by itself. Existing formal baselines, approved module / feature specifications, Public Contracts, Security boundaries, and Change Control decisions remain authoritative where scope overlaps.

## Related Authoritative Documents

- [03-03_FW_Zero_Trust_Firewall.md](../modules/03-03_FW_Zero_Trust_Firewall.md) — Firewall authority
- [01-05_Presentation_and_Dialogs.md](../02_Features/Security/Web_Link_Protection_Spec/01-05_Presentation_and_Dialogs.md) — Web Link presentation authority
- [04_Audit_and_Evidence_Model.md](../00_Baseline/GameProtection/04_Audit_and_Evidence_Model.md) — Audit evidence authority
- [03-01_SYS_System_Safety_and_Lifecycle.md](../modules/03-01_SYS_System_Safety_and_Lifecycle.md) — Runtime lifecycle / WIPER operational-state authority
- [47_User_Experience_and_Interface_Design_Governance_Model.md](../00_Baseline/Governance/47_User_Experience_and_Interface_Design_Governance_Model.md) — Formal UX governance

---

# 32. Protection UX Design

Protectionは、「GSTがどの機能を持っているか」の一覧ではなく、**現在何を、どの範囲で、どの状態で保護しているか**を理解するための画面とする。

ユーザーが最初に知るべき情報を、内部コンポーネント名や実装方式より先に置く。

基本順序:

    何を保護するか
      ↓
    現在どうなっているか
      ↓
    何が影響を受けるか
      ↓
    問題なら何を確認 / 実行するか
      ↓
    詳細な証拠・設定

Protection画面はSecurity FoundationとRuntime Protectionを明確に分離し、GSTが停止している場合にRuntime Protectionが継続していると誤認させない。

## 32.1 Protection Mental Model

ユーザーがProtection画面で理解する対象を、次の4つに整理する。

    Asset      ← ゲーム、セーブデータ、通信、設定など何を守るか
    Scope      ← どのゲーム / 実行環境 / 資産が対象か
    State      ← 現在利用できる保護経路の状態
    Action     ← 問題時に確認・変更・復旧するための導線

「機能が存在する」ことと「現在その保護が利用できる」ことを同一視しない。

また、設定状態と保護状態を混ぜない。ユーザーがカスタム設定にしたことだけを理由に、保護状態を自動的に悪化表示してはならない。

## 32.2 Protection Screen Primary Structure

Protection画面は次の4領域を基本とする。

1. Protection Status Summary
2. Protection by Purpose
3. Attention / Impact / Recommended Action
4. Details / Advanced Controls

概念配置:

    ┌────────────────────────────────────────────────────────────────────┐
    │ ← 保護                                                         │
    ├────────────────────────────────────────────────────────────────────┤
    │ 🛡️ 保護状態: 🟢 保護中                 設定: 標準設定             │
    │ 対象: GST / 現在の保護セッション                                   │
    ├──────────────────────────┬─────────────────────────────────────────┤
    │ 保護するもの              │ 現在の状態                              │
    │                           │                                          │
    │ 🎮 ゲーム / 実行環境      │ 🟢 利用可能な保護経路を稼働中             │
    │ 🌐 ネットワーク           │ 状態 →                                 │
    │ 💾 ファイル / データ      │ 状態 →                                 │
    │ 🧾 完全性 / 監査           │ 状態 →                                 │
    │ 🛡️ Foundation             │ 常時保持される安全基盤                  │
    │                           │                                          │
    └──────────────────────────┴─────────────────────────────────────────┘

これは構造検討用Wireframeであり、最終的な寸法、色、カード形状、アイコンはVisual Design段階で決定する。

## 32.3 Top Status and Scope

画面最上部のProtection Statusは、既存UXベースラインの状態語彙を使用する。

- 🟢 **保護中**
- 🟡 **一部制限あり**
- 🔴 **保護停止**

同時に、**どの範囲の保護状態なのか**を明示する。

例:

    🟢 保護中
    対象: Game A / GST Runtime Protection

また、Configuration Statusは独立した軸として表示する。

    保護状態: 🟢 保護中
    設定状態: カスタム設定

単一の「総合保護度」「100%保護」などのスコアを追加しない。

### 32.3.1 Aggregate Status Derivation Boundary

**Aggregate Protection Status rule:** Aggregate Protection Status is produced within an explicit Scope by Application-owned canonical aggregation. Presentation must render the supplied result and must not derive it from visual state, feature counts, toggle state, or raw subsystem fields.

Protection Status is a semantic summary of the authoritative underlying protection state for the stated scope. The UI must not derive it by counting green cards, enabled toggles, or available feature tiles.

Where a required protection path is unavailable or constrained, the aggregate state must follow the existing Protection Status contract rather than a visual component heuristic.

## 32.4 Security Foundation vs Runtime Protection

Protection画面の最上位で次の2つを分離する。

### Security Foundation

Recovery Host、Recovery Journal、Secure Storage、Integrity Verification等の、無効化を前提としない安全基盤について、ユーザーが「何を支えているか」を理解できるようにする。

Foundationの存在は、GSTが終了していてもFirewallやゲーム監視が動き続けることを意味しない。

### Runtime Protection

GST実行中に提供されるゲーム監視、Firewall等を、現在のRuntime Contextと対象範囲とともに表示する。

GSTが実行中でない場合は、「保護中」とだけ表示してRuntime Protectionが動作中であるかのような誤解を与えない。

## 32.5 Protection by Purpose

Protectionの分類は内部Architectureではなく、ユーザーが守りたい目的で整理する。

Runtime Protectionの基本分類:

- **ゲーム / 実行環境** — ゲーム実行中の対象保護、関連状態、必要なGame Context
- **ネットワーク / Firewall** — 通信制御と、遮断時の影響説明
- **ファイル / データ** — セーブデータや対象ファイルの保全
- **完全性 / 監査** — GSTが確認した完全性・監査状態

Security FoundationはこれらのRuntime Protection分類とは別の領域として表示する。Foundationの存在をRuntime Protectionの一カテゴリとして合算したり、Runtime Protectionの正常性をFoundationの存在だけで推定したりしない。

機能が未実装、利用条件を満たさない、または現在のビルドで利用できない場合は、利用可能な機能として誤表示しない。

### 32.5.1 Foundation / Runtime Visual Separation

Protection画面では、Security FoundationをRuntime Protectionの一覧へ混在させず、視覚的にも別のSection / Surfaceとして扱う。

Runtime Protection
  ├ Game / Execution
  ├ Network / Firewall
  ├ File / Data
  └ Integrity / Audit

Security Foundation
  ├ Recovery support
  ├ Secure storage
  └ Integrity foundation

この分離は、FoundationがGST未起動中にも保持され得る一方、Runtime ProtectionはGST実行中のセッション等に依存することを誤解させないためのもの。

## 32.6 Protection Card Pattern

各保護カードは、可能な範囲で次の順で表示する。

    Purpose
    Current State
    Scope
    Impact
    Next Action
    Details

例:

    ネットワーク保護
    Game A のアウトバウンド通信を現在のポリシーで制御中
    対象: Game A
    影響: 通信先によってはオンライン機能が利用できない場合があります
    [遮断の詳細を見る]

設定値だけを羅列する構成や、「有効 / 無効」だけで意味を終える構成を避ける。

## 32.7 Firewall State and Blocked Communication UX

Firewall遮断が発生した場合、ユーザーは少なくとも次を確認できるようにする。

- 対象ゲーム
- 対象実行ファイル
- 通信方向
- 適用ポリシーまたはルール
- 想定される影響
- 現在利用可能な次の操作

通常ユーザーを個別Firewall rule CRUDへ直接誘導しない。

設定変更や例外変更が必要な場合は、

    状況確認 → Review → Confirm → Execute → 実状態確認

の流れを維持する。

UIには、仕様で定義されていない「今回だけ許可」「この実行ファイルだけ許可」などのスコープを先行表示しない。

## 32.8 WIPER Protection State

WIPERのProtection表示は、単なる「機能ON/OFF」ではなく、実際の監視WatcherとCanaryの成立状態を反映する。

必須構成を満たせない場合は、既存UXベースラインに従い **🔴 保護停止** として表示する。

監視障害時は、残存Watcherがあるという理由だけで「正常稼働」と表示しない。

WIPERのアラートや操作では、UIから内部Infrastructure APIを直接呼び出さず、既存のApplication境界を通じて実行する。

ユーザー向け表示では、PIDやStartTime等の内部識別子を通常画面の主要情報にしない。必要な詳細証拠はDetailsで確認できる構成とする。


### 32.8.1 Firewall / WER Path Normalization
The approved design uses conservative owner-defined normalization for active-session Runtime Protection:

- Firewall: Not Applicable / Disabled is excluded; fully verified required protection is 保護中; residual effective protection with missing/drifted/partial required paths is 一部制限あり; required protection not effectively established or safely unverifiable is 保護停止.
- WER: Not Applicable / Disabled is excluded; active protection with verified BeforeState durability is 保護中; a bounded partial state may be shown as 一部制限あり only when the WER authority explicitly exposes that state; required protection unavailable or authoritatively unknown/unsafe during an active session is 保護停止.
- Post-session WER recovery outcomes remain Recovery/Critical evidence and are not automatically converted into current Runtime Protection Status.
- Presentation renders the authoritative path state and never derives it from card counts, toggles, requested rule counts, or unrelated health fields.
- This semantic decision does not authorize a new Contract; WER active-state observability requires a later contract-boundary review.

## 32.9 Integrity / Audit Protection

完全性・監査系のProtection表示では、「監査ログがある」ことと「すべてが正常である」ことを同一視しない。

表示順は:

    何を確認しているか
      ↓
    最後に確認できた状態
      ↓
    問題 / 未確認の範囲
      ↓
    詳細証拠

とする。

内部ログの大量表示をProtectionトップへ置かず、必要な証拠だけをDetails / Auditへ開く。

判定不能な状態を、緑色の正常状態へ丸めない。

## 32.10 Data Protection Link to Save Data

Protection画面からSave Dataへ遷移できるが、Save Dataの世代管理とProtectionの全体状態を混同しない。

例:

    ファイル / データ保護
    セーブデータのバックアップ状態を確認
    [セーブデータを開く]

ここでは「バックアップがある」ことを、セーブデータが完全に安全であるという総合判定へ拡張しない。

Restoreの詳細判断はSave Data → Restore Review → Recoveryへ引き渡す。

## 32.11 Attention and Critical Conditions

Protection画面では、問題そのものと、その問題に対する行動を隣接させる。

基本形式:

    Problem
    Cause / Known Context
    Recommended Action

例:

    ⚠️ 一部の保護経路が利用できません
    対象: Game A / Runtime Protection
    影響: 該当する保護機能を利用できません
    [詳細を確認]

保護停止、重大な脅威検知、復旧失敗、重大なユーザー確認要求などのCritical状態は、通常のToastだけに依存せず、未確認状態を残す。

Smart DND中もCritical状態を完全に隠さない。

## 32.12 Disabled / Unavailable / Not Applicable

「無効」「利用不可」「対象外」を同じ意味として扱わない。

- **ユーザー設定で無効:** ユーザーが明示的に設定した結果
- **利用不可:** 現在の環境・状態・実装条件では利用できない
- **対象外:** 現在のゲーム / 資産 / セッションには該当しない

この区別により、設定上のカスタマイズを保護異常へ誤変換しない。

対象外の機能を、GST全体の異常として黄色や赤で強調しない。

## 32.13 Action Safety

Protection画面でのPrimary Actionは、現在の問題を理解したあとに次の行動へ進むものとする。

安全な情報操作:

    詳細を見る / 更新 / 該当画面を開く

影響の大きい操作:

    設定変更 / 例外追加 / Firewall変更 / 保護停止 / 復旧

は既存の `Review → Confirm → Execute` へ接続する。

特に「保護を一時停止」「例外を追加」など、意味が大きい操作を単純なトグルだけで実行しない。

## 32.14 Refresh and Runtime Context

Protectionの状態はRuntime中に変化するため、Refreshは読み取り専用として扱う。

Refresh実行でFirewall変更、ルール適用、サービス再起動、保護状態変更などを開始しない。

Auto-Refreshを利用する場合も、現在のユーザー編集を上書きしない。

Runtime Contextが変化した場合は、

    対象ゲーム / セッション → 現在状態 → 更新時刻

を可能な範囲で表示し、「前のゲームの状態」を現在のゲームの状態と誤認させない。

## 32.15 Advanced / Expert Details

詳細Firewall rule、KDF、監査証拠、Migration、診断情報などはAdvanced / Expertへ段階的に開示する。

ただし、専門機能であることだけを理由に、ユーザーに必要な説明を隠さない。

Normal画面:

    目的 / 状態 / 影響 / 次の操作

Advanced / Expert:

    詳細設定 / 証拠 / 技術情報 / 専門操作

という役割分担を基本とする。

## 32.16 Protection Wireframe — Working Baseline

    ┌────────────────────────────────────────────────────────────────────┐
    │ ← 保護                                                          │
    ├────────────────────────────────────────────────────────────────────┤
    │ 🛡️ 保護状態  🟢 保護中          設定状態  標準設定               │
    │ 対象: GST Runtime Protection / 現在の保護セッション               │
    ├───────────────────────┬────────────────────────────────────────────┤
    │ 保護するもの           │ 現在の状態                                 │
    │                        │                                             │
    │ 🎮 ゲーム / 実行環境   │ 🟢 現在利用可能な保護経路を稼働中            │
    │ 🌐 ネットワーク        │ Firewall / 通信状態 →                       │
    │ 💾 ファイル / データ   │ セーブデータ保護 →                          │
    │ 🧾 完全性 / 監査       │ 確認状態 →                                  │
    ├───────────────────────┼────────────────────────────────────────────┤
    │ Security Foundation    │ 復旧・安全基盤 →                             │
    │                        │                                             │
    ├───────────────────────┴────────────────────────────────────────────┤
    │ ⚠️ 注意事項                                                        │
    │ 一部の保護経路に制限があります。                                   │
    │ [何が影響するかを見る]                                             │
    └────────────────────────────────────────────────────────────────────┘

## 32.16.1 Annotated Wireframe — Protection / Visual UX Working Baseline

このAnnotated Wireframeは、Protection画面の既存意味論を視覚構造として固定するための作業基準である。特に、Aggregate Protection Statusをカード数・色・トグル状態から推定させないこと、Security FoundationとRuntime Protectionを別面として扱うことを優先する。

```text
┌────────────────────────────────────────────────────────────────────────────┐
│ GST / 現在の保護範囲                                      保護              │
├────────────────────────────────────────────────────────────────────────────┤
│ Protection Status                         Configuration Status              │
│ 🟢 保護中                                 標準設定                           │
│ 対象: Game A / GST Runtime Protection                                      │
│ 状態の基準: 現在の保護セッション                                             │
├──────────────────────────────────┬─────────────────────────────────────────┤
│ Runtime Protection               │ 状態 / 影響 / 次の操作                  │
│                                  │                                         │
│ 🎮 ゲーム / 実行環境             │ 🟢 現在利用可能な保護経路を稼働中            │
│    Game A                        │ 対象: Game A                            │
│                                  │ 影響: 現在のRuntime Contextで適用        │
│ 🌐 ネットワーク / Firewall       │ [通信状態を見る]                        │
│    通信制御                      │                                         │
│                                  │ 🟡 一部制限あり                         │
│ 💾 ファイル / データ             │ 対象: Save Data / 現在のContext        │
│    セーブデータ等                │ [何が影響するかを見る]                 │
│                                  │                                         │
│ 🧾 完全性 / 監査                 │ 未確認 / 判定不能時は状態を明示          │
│                                  │ [確認結果を見る]                        │
├──────────────────────────────────┴─────────────────────────────────────────┤
│ Security Foundation                                                        │
│ 復旧・安全基盤 / Secure Storage / Integrity Foundation                   │
│ Runtime Protectionとは別の基盤として表示                                  │
│ [詳細を見る]                                                              │
├────────────────────────────────────────────────────────────────────────────┤
│ ⚠️ 注意事項                                                               │
│ 一部の保護経路に制限があります。対象と影響を確認してください。             │
│ [詳細を確認]                                                              │
└────────────────────────────────────────────────────────────────────────────┘
```

### 32.16.1.1 Region Annotations

| ID | Region | Visual / UX rule | Safety meaning |
|---|---|---|---|
| P1 | Protection Status | ScopeをStatusと同じ視認レベルで表示 | Statusを別ゲーム / 別セッションへ誤適用しない |
| P2 | Configuration Status | Protection Statusと別軸で配置 | Custom設定を自動的にProtection異常と解釈しない |
| P3 | Runtime Protection | Purpose → State → Scope → Impact → Actionの順 | 機能名だけで稼働を推測させない |
| P4 | Individual Protection State | StateはText + Iconで表現 | 色だけで正常 / 注意 / 停止を判断させない |
| P5 | Security Foundation | Runtime Protectionと別Section / visual group | Foundationの存在をRuntime稼働と誤認しない |
| P6 | Attention | Problem / Context / Impact / Next Actionを隣接 | 通知を閉じてもUnderlying stateを消さない |
| P7 | Details | 技術証拠を段階開示 | PID等の内部識別子を主要UIへ持ち込まない |
| P8 | Navigation | Save Data / Recovery / Settings等はdestinationとして表示 | Details / NavigationとExecuteを混同しない |

### 32.16.1.2 Aggregate Status and State Mapping

- Aggregate Protection Statusは、個々のカードの色、件数、トグル状態、Availabilityの単純集計から生成される視覚値ではない。
- `🟢 保護中` は、表示Scopeについて権威的なProtection Stateがそう判断される場合だけ表示する。
- `Unavailable`、`Not Applicable`、`Disabled`、`Error`、`Critical` は、それぞれ既存の意味を保持する。
- Runtime Contextが存在しない場合、Foundationが存在することだけを根拠にRuntime Protectionを「保護中」と表示しない。
- 個別カードが`Unavailable`でも、それだけからAggregate Statusの別の値をUIが独自算出しない。適用対象のProtection Status authorityに従う。
- `Refresh` はRead-onlyであり、状態取得以外の副作用を開始しない。
- Protection画面を開いたこと、詳細を見たこと、通知をDismissしたことはProtection Stateを変えない。

### 32.16.1.3 State Variants

**Normal / Active Runtime**

- Protection StatusとScopeが明示される。
- Runtime Protectionの各Purposeに現在状態と影響が表示される。
- Security Foundationは別Sectionとして表示される。

**Runtime Not Active / No Active Game**

- Runtime Protectionを「保護中」と断定しない。
- 現在Scopeがない場合は、その事実を明示する。
- Foundationの状態とRuntimeの状態を別々に説明する。

**Attention / Partial Restriction**

- 影響範囲を明示し、詳細確認への導線を近接させる。
- 「一部制限あり」をカード数から視覚的に演算しない。

**Critical / Protection Stopped**

- Critical状態を通常Toastだけに依存せず、画面内に残す。
- Smart DND中でも必要なCritical情報を通常通知より上位のPersistent surfaceで確認できる構造を基本とする。
- 重大状態の解消はDismissではなく、権威的な状態確認 / Operation結果によって判断する。

**Unavailable / Not Applicable**

- 利用不可と対象外を区別する。
- 対象外のProtectionをGST全体の異常として赤く集計しない。
- Actionable controlは既存Capabilityが成立している場合だけ表示する。

### 32.16.1.4 Action-State Mapping

| UI Action | Meaning | Must not imply |
|---|---|---|
| 詳細を見る | Read-only inspection / navigation | 状態変更 |
| 更新 / Refresh | 最新状態の再取得 | Protection再起動・設定適用 |
| Save Dataを開く | Save Data surfaceへのnavigation | Backup / Restore Execute |
| Recoveryを開く | Recovery surfaceへのnavigation | Recovery Execute |
| 設定を開く | Settings surfaceへのnavigation | Configuration Apply |
| 保護変更 / 例外変更 | 高影響Operationのentry | 即時実行・承認済み |

高影響Operationは、適用対象の既存Journey / Contractが要求する`Review → Confirm → Execute → Verify`へ接続する。

### 32.16.1.5 Responsive / Accessibility Rules

- WideではSummary → Runtime Protection → Security Foundation → Attentionの順を維持し、Compactでは1列へ再配置できる。
- 文字拡大時にもProtection Status、Scope、Primary detail/action、Backを優先して残す。
- Runtime ProtectionとSecurity Foundationの見出しは、色だけに依存せずText / Icon / Groupingで区別する。
- Individual stateとAggregate statusの意味を、色相だけで伝達しない。
- Keyboard focusはStatus説明 → Runtime Protection → Attention → Navigationの順で意味を理解しやすい構造を基本とする。
- Firewall、WIPER、Integrityなどの詳細情報が横幅不足で切り詰められても、Purpose / State / Scope / Impactの順で可読性を保つ。

### 32.16.1.6 Wireframe Boundary

このAnnotated Wireframeは既存Protection UXの視覚化であり、新しいProtection Capability、Aggregate Status、Runtime behavior、Public Contract、Domain Status、Security Boundaryを追加しない。

Wireframe上でActionableに見える要素は、適用対象の既存Feature / Module / Contract / implementation readinessが成立している場合にのみActionableとする。未成立の機能をUI都合で実在化しない。
## 32.18 Review Findings — 2026-09-29

The focused review identified two presentation-boundary issues and clarified them:

| Finding | Resolution |
|---|---|
| Security Foundation and Runtime Protection could appear as sibling protection categories | Foundation is now a separate visual / semantic surface. |
| Aggregate Protection Status could be inferred from UI card colors or feature availability | Added an explicit rule that aggregate status follows authoritative underlying state, not visual component counting. |

No new Protection capability, aggregate status, Contract, or Runtime behavior is introduced by these clarifications.

## 32.17 Protection Acceptance Criteria

- Protection画面を開いた時点で「何を守る画面か」が理解できる。
- Protection StatusとConfiguration Statusが混同されない。
- Protection Statusの対象範囲が明示される。
- Security FoundationとRuntime Protectionが明確に分離される。
- GST停止中にRuntime Protectionが継続していると誤認させない。
- 保護機能が「存在する」ことと「現在利用可能」であることを混同しない。
- 各保護カードで目的、状態、範囲、影響、次の操作を理解できる。
- Firewall遮断時に対象ゲーム、実行ファイル、方向、適用ルール/ポリシー、影響、次の操作を確認できる。
- Firewall変更などの高影響操作が即時トグルで実行されない。
- WIPERの表示が実際のWatcher / Canary成立状態と矛盾しない。
- 完全性 / 監査の判定不能状態が正常状態へ丸められない。
- Critical状態が通常ToastやSmart DNDに埋没しない。
- Disabled / Unavailable / Not Applicableが区別される。
- Refreshが副作用を持たない。
- Runtime Context変更時に古いGame Contextを現在状態と誤認させない。
- Advanced / Expertに詳細を開示しても、Normalユーザーに必要な目的・状態・影響を隠さない。
- Protection UX追加だけを理由に、新しいRuntime Feature、Public Contract、Domain status、Security capabilityを導入しない。

---

## Extraction Note

Source text was extracted mechanically from the parent UI working draft. Any future semantic change must be reviewed as a design change; an extracted statement must not be treated as a newly approved runtime requirement merely because it now has a dedicated file.

End of Document
