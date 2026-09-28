# 01-10: UI Home and Dashboard Design

**Document ID:** GST-ARCH-UI-HOME-001  
**Version:** 0.4 (Home Annotated Wireframe / Visual UX Review)  
**Status:** Design Working Draft — Not an independent feature authority  
**Authority Relationship:** Subordinate to GST-BASELINE-011, GST-BASELINE-047, applicable approved module / feature specifications, and Change Control decisions  
**Parent Concept:** [01-08_UI_Information_Architecture_and_Interaction_Concept.md](01-08_UI_Information_Architecture_and_Interaction_Concept.md)  
**Historical Source Mapping:** Former Section 4 + Section 23 of the parent 01-08 UI working draft  
**Current Ownership:** This document is the maintained location for the Home / Dashboard material after the working-draft split.
**Purpose:** Organize existing UI information-architecture and interaction design for this scope without expanding Runtime Features, Public Contracts, Domain Status, Security capabilities, or Implementation Readiness.

## Document Boundary

This document preserves UI design material that was formerly organized as Section 4 + Section 23 in the parent 01-08 UI working draft. After the working-draft split, 01-08 retains global IA / interaction principles while this document owns the Home / Dashboard material.

The preserved section numbering is historical traceability only. It does not imply that those sections still exist in 01-08.

This document does not authorize Runtime implementation by itself. Existing formal baselines, approved module / feature specifications, Public Contracts, Security boundaries, and Change Control decisions remain authoritative where scope overlaps.

## Related Authoritative Documents

- [01-06_Presentation_and_UI_Architecture.md](01-06_Presentation_and_UI_Architecture.md) — Presentation / WPF / MVVM / Dispatcher
- [11_UI_UX_and_User_Interaction_Model.md](../00_Baseline/Governance/11_UI_UX_and_User_Interaction_Model.md) — Formal UI/UX baseline
- [47_User_Experience_and_Interface_Design_Governance_Model.md](../00_Baseline/Governance/47_User_Experience_and_Interface_Design_Governance_Model.md) — Formal UX governance

---

# 4. Home / Dashboard Information Design

The Dashboard is not a "scoreboard". It is a **decision surface**.

## 4.1 Top status block

**Protection Status aggregation rule:** Protection Status uses the adopted scope-first model. Home's status is a scope-labeled GST-wide Runtime Protection summary; selected-game and active-session protection states remain separately scoped. Application owns the canonical aggregation semantics; Presentation renders the result only.

The first visual block presents two independent axes:

```
Protection Status       🟢 保護中
Configuration Status    標準設定
```

The implementation may combine them into one compact header row, but they remain semantically independent.

Allowed protection vocabulary:

- 🟢 **保護中**
- 🟡 **一部制限あり**
- 🔴 **保護停止**

Allowed configuration vocabulary:

- **標準設定**
- **カスタム設定**

No single aggregate "100% protected", "security score", or similar composite rating is introduced by this concept.

### 4.1.1 Status Scope Disclosure

Protection Status and Configuration Status must never be presented without enough context to identify what they describe.

The Home surface should provide a compact scope cue whenever the underlying state is not self-evidently global. Depending on the actual source state, the cue may distinguish:

- application / GST-wide state
- selected registered game
- currently active game / session

The UI must not imply that a game-specific status is a PC-wide guarantee, or that a GST-wide foundation state means Runtime Protection is active while GST is not running.

Scope text is a presentation aid only. It does not create a new Domain Status or Runtime capability.

## 4.2 Runtime boundary explanation

When relevant, the Dashboard should make the distinction between:

- **Security Foundation**
- **Runtime Protection**

understandable without presenting internal architecture terms first.

Example user-facing wording pattern:

```
安全基盤
復旧情報・安全な保存・完全性確認を維持

実行中の保護
GST起動中にゲーム監視・通信保護などを提供
```

The exact features shown depend on the current implementation and release scope.

## 4.3 Current game context

When a registered game is active or recently selected, the Dashboard may show:

```
🎮 現在のゲーム
[ Game Name ]

保護状態       🟢 保護中
設定状態       カスタム設定
起動状態       実行中 / 停止
```

The Dashboard should not duplicate every game setting. It should provide a summary and a clear route to Games.

## 4.4 Important unresolved state

Only issues requiring user awareness belong in the prominent area.

Examples:

- protection stopped
- restore failed
- important recovery action required
- unresolved critical security/data-protection event

Routine informational events remain in activity/history surfaces.

## 4.5 Quick Recovery

"Quick Recovery" is a fast entry into the recovery workflow, not an unconditional one-click overwrite.

```
[ ↩ クイックリカバリ ]
        ↓
Recovery Review
        ↓
Confirm
        ↓
Execute
        ↓
Verify
```

The Dashboard should make this distinction visually obvious.

## 4.6 Backup / Save Data summary

A compact dashboard card may show:

- most recent backup state
- number of protected games with current backup state known
- storage pressure or retention warnings
- pending recovery state

Detailed management remains under Save Data.

---

# 23. Home / Dashboard UX Design

Homeは情報集約画面ではなく、ユーザーの判断起点とする。第一画面だけで「今は問題なく使えるか」「何か対応が必要か」「必要ならどこへ進むか」を判断できることを目的とする。

## 23.1 Home Priority Order

表示優先順位は次の通りとする。

1. Protection Status
2. Configuration Status
3. Needs Attention / Critical state
4. Current Game Context
5. Primary actions: Games / Save Data / Recovery
6. Recent important activity

重要な問題が存在する場合、Needs AttentionはCurrent Game Contextより前に表示できる。ただしCritical状態を通常のカードと同列に埋没させない。

## 23.2 Normal Home State

```
┌───────────────────────────────────────────────────────────────┐
│ ホーム                                                       │
│                                                               │
│ ┌─────────────────────┐  ┌─────────────────────┐             │
│ │ 保護状態            │  │ 設定状態            │             │
│ │ 🟢 保護中           │  │ 標準設定            │             │
│ │ 詳細を見る →        │  │ 設定を見る →        │             │
│ └─────────────────────┘  └─────────────────────┘             │
│                                                               │
│ 🎮 現在のゲーム                                               │
│ ┌───────────────────────────────────────────────────────────┐ │
│ │ ゲーム未実行                                              │ │
│ │ 次に遊ぶゲームをGamesから選択できます                    │ │
│ │                                   [ ゲームを選ぶ ]        │ │
│ └───────────────────────────────────────────────────────────┘ │
│                                                               │
│ ┌─────────────────────┐  ┌─────────────────────┐             │
│ │ 💾 セーブデータ      │  │ ↩ 復旧              │             │
│ │ 最新状態を確認       │  │ 問題がある場合に確認 │             │
│ │ [バックアップを見る] │  │ [復旧を確認]         │             │
│ └─────────────────────┘  └─────────────────────┘             │
│                                                               │
│ 最近の重要な活動                                             │
│ 09:30  バックアップ完了                                      │
│ 08:52  ゲーム登録                                            │
└───────────────────────────────────────────────────────────────┘
```

## 23.3 Active Game Home State

ゲーム実行中は、Current Game Contextの情報量を増やし、設定詳細はGamesへ残す。

```
🎮 現在のゲーム
[ Game Name ]
実行中
保護状態   🟢 保護中
設定状態   カスタム設定
[ ゲームの詳細 ]
```

Homeではゲームの個別ルールやFirewall rule一覧を展開しない。

### 23.3.1 Active Session vs Selected Game

Home must distinguish a **currently active game/session** from a **selected or recently viewed registered game**.

- Active game/session: may be summarized as the current runtime context.
- Selected/recent game: may be used for navigation continuity, but must not be labeled as currently running.
- No active game: show an explicit non-running state rather than retaining stale runtime wording.

When the user changes selection while no game is running, the selected game may remain visible as navigation context, but Protection Status must not be silently re-scoped to that game unless the underlying state actually supports that scope.

## 23.4 Needs Attention State

問題がない場合はNeeds Attention領域そのものを表示しないか、高さを極小化する。

対応が必要な場合:

```
⚠ 要対応
ゲームの復旧に確認が必要です。
対象: Game Name
影響: 現在のゲームデータに変更を加える前に確認が必要です。
[ 詳細を確認 ]
```

この領域は「警告一覧」ではなく、現在ユーザーが行うべき判断を示す。

## 23.5 Critical State

Critical security/data-protection状態では、Homeの通常カード構造を優先して維持するのではなく、最上部へCritical stateを投影する。

必要情報:

- 何が起きたか
- 何が影響を受けるか
- 現在の保護状態
- ユーザーが今できること
- Recovery / Detailsへの入口

ゲーム中のSmart Do Not Disturb設定によってCritical通知を完全に隠してはならない。

## 23.6 No Game Registered State

ゲーム未登録の場合、空白のDashboardにせず、最初の目的を明示する。

```
🎮 まだゲームが登録されていません
GSTで守るゲームを登録すると、ゲーム別の保護状態やバックアップを確認できます。
[ ゲームを登録 ]
```

これはエラーではない。Protection Statusなどのシステム状態と混同しない。

## 23.7 No Backup Yet State

バックアップ未作成は、保護停止とは別の状態である。

例:
```
💾 セーブデータ
まだバックアップがありません
大切なセーブを保護するには、最初のバックアップを作成してください。
[ バックアップを作成 ]
```

「バックアップなし」を自動的にProtection Status = 保護停止と同義にしない。実際の製品仕様で定義された保護経路の状態を優先する。

## 23.8 Home Action Budget

Homeに表示する主要アクションは、通常状態では数個に抑える。複数の「同じ結果へ到達するボタン」を同時に並べない。

推奨:

- ゲームを選ぶ / Gamesへ
- バックアップを見る / Save Dataへ
- 対応が必要なら確認する / Recoveryまたは対象画面へ
- 保護詳細を見る / Protectionへ

Expert操作をHomeの主要アクションとして表示しない。

### 23.8.1 Action Eligibility

Home actions should reflect the actual availability of their underlying operation.

- A recovery entry should not imply that a recoverable item exists when no eligible recovery candidate is known.
- A backup action should distinguish "no backup yet" from an unavailable or failed backup operation.
- An unavailable operation must not be represented as an executable action merely because the destination screen exists.

This is a presentation rule: it does not authorize a new operation or change the underlying Contract.

## 23.9 Home Must Not Do

Homeでは次を行わない。

- Firewall rule CRUDの直接表示
- SQLite / Repository / Workerなど内部構造の表示
- 単一の総合Security Scoreの算出
- ユーザーが意図しない設定変更
- Critical stateをRecent Activityの中へだけ隠す
- 未実装機能を利用可能なボタンとして見せる

## 23.10 Home Decision Questions

Homeの設計レビューでは、最低限次の質問に答えられることを確認する。

1. このPCでGSTは今、通常利用できる状態か。
2. Protection Statusはどうなっているか。
3. 設定はStandardかCustomか。
4. 今、ユーザーが対応すべきことはあるか。
5. 今どのゲームが関係しているか。
6. 問題がある場合、次にどこへ行けばよいか。

### 23.12 Review Findings — 2026-09-29

The first focused review against GST-BASELINE-011, GST-BASELINE-047, and the Presentation Architecture identified three presentation risks and resolved them in this working draft:

| Finding | Resolution |
|---|---|
| Protection / Configuration status could be read without an explicit scope | Added Status Scope Disclosure guidance. |
| "Current Game" could blur active runtime context and selected/recent game context | Added explicit Active Session vs Selected Game distinction. |
| Destination buttons could imply an executable operation even when its underlying operation is unavailable | Added Action Eligibility guidance. |

These are presentation clarifications only. No Runtime Feature, Public Contract, Domain Status, Security capability, or implementation phase is changed by this review.

## 23.11 Home Wireframe Acceptance Criteria

- 画面上部だけでProtection StatusとConfiguration Statusを区別できる。
- 通常状態と要対応状態の差が視覚的に明確。
- ゲーム未登録・ゲーム未実行・バックアップ未作成を障害と誤認させない。
- Critical stateを通常通知と同列に埋没させない。
- 主要アクションが目的ベースで理解できる。
- HomeからRecoveryへ迷わず進める。
- Homeだけで詳細な専門設定を強制的に理解する必要がない。
- 現在の対象ゲームを失わない。

---

## 23.13 Annotated Wireframe — Home / Visual UX Working Baseline

このAnnotated Wireframeは、`01-09_UI_Visual_Design_System.md` のVisual FoundationをHomeへ適用した情報配置・状態表現の作業基準である。ピクセル寸法、WPF resource名、最終色、Icon assetは後続のVisual QA / implementation段階で確定する。

### 23.13.1 Normal State

```text
┌────────────────────────────────────────────────────────────────────────────┐
│ GST                         現在の文脈        [🔎 検索]  [? ヘルプ]          │
├────────────────┬───────────────────────────────────────────────────────────┤
│ 🏠 ホーム      │ ホーム                                                     │
│ 🎮 ゲーム      │ 「今どうなっているか」                                     │
│ 💾 セーブ      │                                                           │
│ 🛡️ 保護       │ ┌───────────────────┐  ┌───────────────────┐              │
│ ↩ 復旧        │ │ 保護状態          │  │ 設定状態          │              │
│ ⚙ 設定        │ │ 🟢 保護中         │  │ 標準設定          │              │
│                │ │ 対象: GST Runtime │  │ 実効状態           │              │
│                │ │ [詳細を見る]      │  │ [設定を見る]       │              │
│                │ └───────────────────┘  └───────────────────┘              │
│                │                                                           │
│                │ 🎮 現在のゲーム                                           │
│                │ ┌───────────────────────────────────────────────────────┐ │
│                │ │ ゲーム未実行                                            │ │
│                │ │ 現在のActive Sessionなし                               │ │
│                │ │ [ゲームを選ぶ]                                         │ │
│                │ └───────────────────────────────────────────────────────┘ │
│                │                                                           │
│                │ ┌────────────────────────┐ ┌────────────────────────────┐ │
│                │ │ 💾 セーブデータ        │ │ ↩ 復旧                     │ │
│                │ │ 最新状態を確認         │ │ 要対応項目がある場合       │ │
│                │ │ [バックアップを見る]   │ │ [復旧を確認]               │ │
│                │ └────────────────────────┘ └────────────────────────────┘ │
│                │                                                           │
│                │ 最近の重要な活動                                           │
│                │ 09:30 バックアップ完了   08:52 ゲーム登録                  │
├────────────────┴───────────────────────────────────────────────────────────┤
│ Runtime / Protection Context · 通知状態 · Version                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### 23.13.2 Region Annotations

| ID | Region | Visual / UX rule | Safety meaning |
|---|---|---|---|
| H1 | Page Header | Page title + one-line purpose; avoid decorative density | Orientation only |
| H2 | Protection Status | State text + icon + explicit scope; no score / card-count inference | Canonical Protection Status only |
| H3 | Configuration Status | Show effective Standard / Custom state independently | Configuration is not Protection Status |
| H4 | Current Game | Explicitly label Active Session vs non-running / selected context | Prevent stale runtime interpretation |
| H5 | Primary Context Actions | Destination/action name states what the user will review or open | Availability follows underlying capability |
| H6 | Save Data Summary | Summary only; detailed generations remain in Save Data | No implied recoverability verdict |
| H7 | Recovery Entry | Review-oriented label; not direct destructive execution | Recovery gate preserved |
| H8 | Recent Activity | Important state changes only; never the sole Critical surface | History does not replace current state |
| H9 | Footer / Context Strip | Compact secondary information only | Critical state never hidden here |

### 23.13.3 Action-State Mapping

- `詳細を見る` / `設定を見る` / `ゲームを選ぶ` はNavigation / Read-only entryとして扱い、クリック自体をOperation executionと表示しない。
- `バックアップを見る` はSave Dataへの入口であり、未確認のBackup存在やRecoverabilityを新規に断定しない。
- `復旧を確認` はRecovery Reviewへの入口であり、実データ変更を直接開始しない。
- underlying operationが利用不可の場合、Destinationが存在することだけを理由にExecutable actionとして表示しない。
- ActionのBusy / Disabled表現は`01-09`および`01-17`の共通State Modelに従う。

### 23.13.4 State Variants

**Attention**

Protection / Configuration cardsの下に、Problem → Impact → Next Actionを置く。Current Gameより上に配置してよいが、Criticalでない情報を過度に画面全体へ拡張しない。

**Critical**

Critical状態はPage Header直下または同等の最上位領域へ投影する。Recent Activityだけへ隠さない。DismissはUnderlying Critical stateの解決を意味しない。

**No Game Registered**

空のDashboardをエラー風にせず、Gamesへの入口を目的として示す。

**No Active Game**

Selected GameやRecent Gameが残っていても「実行中」と表記しない。

**Unavailable / Error**

Destination存在、データEmpty、Operation Errorを同一表示にしない。Unavailableは現行Build / Environment / Context上の利用不能、Errorは本来利用可能な処理の失敗として扱う。

### 23.13.5 Responsive / Accessibility Rules

- WideではStatus cardsを2列、Compactでは1列へ再flowできる。
- Body / Explanationの可読性を優先し、文字を極端に縮小して収納しない。
- 日本語長文化でButton labelが切れない。必要なら折り返し・再配置する。
- Keyboard focusはH1〜H8の主要操作順を追える。
- Critical / Attention / Normalは色だけでなくText + Icon + Placementで区別する。
- 125%等のWindows表示倍率、大きい文字、狭いWindowでもProtection Status、Configuration Status、主要Action、Recovery入口が画面外へ押し出されない。

### 23.13.6 Wireframe Boundary

このAnnotated WireframeはHomeの情報・視覚構造を定義するものであり、Homeに新しいRuntime Feature、Public Contract、Domain Status、Security capabilityを追加するものではない。Wireframe上のActionは、実際の利用可能性が成立する既存Capabilityへ接続する場合のみActionableとする。

---

## Extraction Note

Source text was extracted mechanically from the parent UI working draft. Any future semantic change must be reviewed as a design change; an extracted statement must not be treated as a newly approved runtime requirement merely because it now has a dedicated file.

End of Document
