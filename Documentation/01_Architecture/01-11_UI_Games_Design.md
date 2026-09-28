# 01-11: UI Games Design

**Document ID:** GST-ARCH-UI-GAMES-001  
**Version:** 0.4 (Games Annotated Wireframe / Visual UX Review)  
**Status:** Design Working Draft — Not an independent feature authority  
**Authority Relationship:** Subordinate to GST-BASELINE-011, GST-BASELINE-047, applicable approved module / feature specifications, and Change Control decisions  
**Parent Concept:** [01-08_UI_Information_Architecture_and_Interaction_Concept.md](01-08_UI_Information_Architecture_and_Interaction_Concept.md)  
**Historical Source Mapping:** Former Section 24 + Section 30 of the parent 01-08 UI working draft  
**Current Ownership:** This document is the maintained location for the Games material after the working-draft split.
**Purpose:** Organize existing UI information-architecture and interaction design for this scope without expanding Runtime Features, Public Contracts, Domain Status, Security capabilities, or Implementation Readiness.

## Document Boundary

This document preserves UI design material that was formerly organized as Section 24 + Section 30 in the parent 01-08 UI working draft. After the working-draft split, 01-08 retains global IA / interaction principles while this document owns the Games material.

The preserved section numbering is historical traceability only. It does not imply that those sections still exist in 01-08.

This document does not authorize Runtime implementation by itself. Existing formal baselines, approved module / feature specifications, Public Contracts, Security boundaries, and Change Control decisions remain authoritative where scope overlaps.

## Related Authoritative Documents

- [01-06_Presentation_and_UI_Architecture.md](01-06_Presentation_and_UI_Architecture.md) — Presentation architecture
- [11_UI_UX_and_User_Interaction_Model.md](../00_Baseline/Governance/11_UI_UX_and_User_Interaction_Model.md) — Formal UI/UX baseline
- [47_User_Experience_and_Interface_Design_Governance_Model.md](../00_Baseline/Governance/47_User_Experience_and_Interface_Design_Governance_Model.md) — Formal UX governance
- [10_Game_Lifecycle_and_Management_Model.md](../00_Baseline/GameProtection/10_Game_Lifecycle_and_Management_Model.md) — Game lifecycle / launch-state authority
- [03_Game_Environment_Protection_Model.md](../00_Baseline/GameProtection/03_Game_Environment_Protection_Model.md) — Game protection scope authority

---

# 24. Next UX Design Target — Games

Homeが「今どうなっているか」を担当するのに対して、次はGamesを「何を遊ぶか・何を登録するか・どう起動するか」の中心として設計する。

Gamesでは通常起動、Strict Launch、ゲーム登録、Portable Safe Onboarding、ゲーム単位のProtection/Configuration表示の関係を整理する。

# 30. Games UX Design

GamesはGSTの日常利用における主要画面であり、「ゲームを登録する」「ゲームを選ぶ」「起動する」「現在の保護状態を理解する」「ゲーム固有設定を調整する」という一連の行動を一つのゲームContextで扱う。

## 30.1 Game-Centric Mental Model

ユーザーの基本認識は「GSTを操作している」ではなく、

Games → Game A → いまGame Aをどう扱うか

とする。

したがって、ゲームを選択した後は可能な限り同じGame Contextを維持し、毎回Games一覧へ戻って再選択させない。

## 30.2 Games Screen Primary Structure

Games画面は次の3領域を基本とする。

1. Game List / Selection
2. Selected Game Overview
3. Contextual Actions

概念配置:

Game List | Selected Game
Game A   | Game A
Game B   | Status
Game C   | Launch
         | Protection summary
         | Save Data summary
         | Settings

ゲーム数が少ない場合でも、一覧と詳細の役割は混ぜない。

## 30.3 Game List

一覧では、ユーザーが「どのゲームを選ぶか」を短時間で判断できることを優先する。

各ゲーム項目には必要に応じて:

- Game Name
- current operational state
- Protection Status
- Configuration Status
- relevant warning / attention marker

を表示する。

内部DB ID、Repository名、Worker状態などを一覧の主要情報にしない。

### 30.3.1 Game List Information Budget

The Game List should remain scannable as the number of registered games grows.

- Primary identity is the user-facing Game Name.
- Status indicators are compact summaries, not full policy descriptions.
- Detailed Protection / Configuration information remains in Selected Game Overview or the relevant destination.
- The list must remain compatible with the existing Presentation / QA virtualization and pagination expectations where applicable; this design does not authorize a new data-loading contract.

## 30.4 Game Selection and Context

選択されたゲームは明確にハイライトし、Selected Game Overviewへ即時反映する。

選択状態を表すだけの操作と、ゲームの起動など副作用のある操作を混同しない。

ゲームを選択しただけではFirewall、Backup、設定変更などを開始しない。

### 30.4.1 Selected vs Active Runtime Context

Games uses two related but distinct concepts:

- **Selected Game:** the game currently being inspected or configured in the UI.
- **Active Game / Session:** a game process that is actually running and bound to a current runtime session.

Selection must not be treated as proof of execution. A selected game may be stopped, historical, or unavailable.

## 30.5 Selected Game Overview

ゲーム詳細画面では、最初に:

Game Name
Current Game Status
Protection Status
Configuration Status

を表示する。

その後に目的別のカードを配置する。

Recommended order:
Launch → Protection → Save Data → Activity / Details

細かな設定はOverviewの下位階層またはDetailsへ置き、画面上部へすべての設定項目を並べない。

### 30.5.1 Status Scope Cue

Protection Status and Configuration Status shown in Selected Game Overview are explicitly scoped to the selected game where that scope is supported by the underlying state.

The UI must not present a game-scoped status as a GST-wide guarantee. Conversely, a GST-wide Security Foundation state must not be relabeled as if it were proof that the selected game's Runtime Protection is currently active.

## 30.6 Launch Area

LaunchはGames画面で最重要のPrimary Actionとする。

通常起動とStrict Launchが現在のビルドで利用可能な場合は、両者の意味を明確に区別する。

例:
▶ 通常起動 — 現在のGameProfile / Effective Policyで起動
🛡 Strict Launch — 承認されたより厳格な通信・報告・監視条件で起動

Strict Launchについて、Sandboxや完全なOS隔離を提供すると誤認させない。

### 30.6.1 Strict Launch Presentation Boundary

Strict Launch may be presented only according to an approved feature specification and the actual capability of the current build.

The UI must not describe Strict Launch as sandboxing, OS isolation, anti-cheat bypass, or guaranteed compatibility unless an authoritative specification explicitly defines that behavior.

## 30.7 Launch Readiness

起動前に既知の重要状態がある場合、起動ボタンの近くで知らせる。

例:
Protection Status: 一部制限あり
Backup: 最新バックアップあり
Action required: なし

これは起動を不必要に阻害するためではなく、ユーザーが起動前の状況を理解するためである。

起動そのものが重要設定変更を伴わない場合、毎回の長い確認ダイアログは避ける。

## 30.8 Game-Specific Protection Summary

ゲーム固有の保護情報は、設定値の羅列ではなく:

何を保護するか → 現在どうなっているか → 影響 → 詳細

の順で表示する。

例:
ネットワーク保護
アウトバウンド通信を制御中
詳細 →

設定値の変更は既存のMaterial Change / Review → Confirm → Executeルールに従う。

## 30.9 Game-Specific Settings

Game-specific Settingsは「このゲームについてだけ変更する」ことが明確になる構造とする。

各設定では可能な範囲で:
Current value
Standard / Global reference
Scope
Purpose
Impact

を確認できるようにする。

Global設定に従う、GameProfile個別設定など、スコープの違いを視覚的に明確にする。

## 30.10 Portable Safe Onboarding

Portable Gameの追加では、ドラッグ&ドロップの簡単さを維持しながら、ユーザーが「何が起きるか」を事前に理解できることを優先する。

基本流れ:
Drop → Inspect → Show destination / detected contents → Execute extraction where supported → Verify → Register

抽出前に対象・保存先・重要な検証結果を確認できることを基本とする。

アーカイブの安全性を確認したことと、そのゲーム自体が安全であることを同一視しない。

## 30.11 In-Game / Background Discovery

GSTが実行中の未管理ゲームを検出し、承認済み仕様に基づくGameIngestionDialogを表示する場合、現在のユーザー作業を不必要に中断しない。

提示内容:
検出対象 → なぜ表示されたか → 選択可能な対応 → 現在の作業へ戻る

Criticalな状態と通常のオンボーディングを同じ優先度で扱わない。

## 30.12 Game State vs GST State

Gameの状態とGST全体の状態を分離する。

例:
GST Protection Status = 保護中
Game A = 実行中
Game B = 未実行

Game Bが未実行であることをGSTの異常と表示しない。

## 30.13 Uninstalled / Historical Game

ゲーム本体がPCから削除された場合でも、承認済み仕様によりGSTに過去のゲーム情報や関連バックアップが残る場合がある。

この状態は:
Current / Installed
Historical / Not Installed

などユーザーが意味を理解できる形で区別する。

過去データを持っていることと、現在ゲームが保護対象として実行中であることを混同しない。

## 30.14 Game Search and Filtering

登録ゲーム数が増えた場合は、Game List内検索や既存のGlobal Searchからゲームを素早く見つけられる設計を採用候補とする。

検索中でも現在のゲームContextを失わない。

フィルター状態はBackや詳細画面から戻った際に安全な範囲で維持する。

## 30.15 Game Action Efficiency

日常利用で頻度が高い:
ゲーム選択 → 起動
ゲーム選択 → セーブ確認
ゲーム選択 → 保護状態確認

は短い導線で到達できるようにする。

ゲーム起動のPrimary Actionを、詳細設定や削除などの低頻度・高影響操作より視覚的に優先する。

## 30.16 Games Wireframe — Working Baseline

```text
┌────────────────────────────────────────────────────────────────────┐
│ ←  Games                                      [🔎 ゲームを検索]  │
├──────────────────┬─────────────────────────────────────────────────┤
│ 登録ゲーム       │ 🎮 Game A                                      │
│                  │                                                 │
│ 🟢 Game A        │ 🟢 保護中     カスタム設定                    │
│ 🟢 Game B        │ 実行状態: 停止                                 │
│ 🟡 Game C        │                                                 │
│                  │ [ ▶ 通常起動 ]  [ 🛡 Strict Launch ]          │
│ [ゲームを登録]   │                                                 │
│ [安全導入]       │ ┌─────────────┐ ┌─────────────┐               │
│                  │ │🛡 保護       │ │💾 セーブ     │               │
│                  │ │現在の状態    │ │最新状態      │               │
│                  │ │[詳細]        │ │[開く]        │               │
│                  │ └─────────────┘ └─────────────┘               │
│                  │                                                 │
│                  │ ⚙ このゲームの設定                             │
│                  │ [詳細設定を表示]                               │
│                  │                                                 │
│                  │ 最近の重要な活動                               │
└──────────────────┴─────────────────────────────────────────────────┘
```

このWireframeは構造検討用であり、最終的な色、寸法、カード形状、アイコンは未確定。

### 30.16.1 Annotated Wireframe — Games / Visual UX Working Baseline

このAnnotated Wireframeは、`01-09_UI_Visual_Design_System.md` のVisual Foundationと、`01-18` / `01-19` のJourney・Context規則をGamesへ適用した作業基準である。最終Pixel値、WPF resource、Icon assetはVisual QA / implementation段階で確定する。

#### Normal / Selected Game

```text
┌────────────────────────────────────────────────────────────────────────────┐
│ GST / 現在の文脈             Games                     [🔎 ゲームを検索]   │
├──────────────────────┬─────────────────────────────────────────────────────┤
│ 登録ゲーム             │ 🎮 Game A — Selected Game                         │
│                       │                                                     │
│ ● Game A              │ Game Status: 未実行                                │
│ ○ Game B              │ Protection Status: 🟢 保護中                       │
│ ○ Game C              │ Configuration Status: カスタム設定                │
│                       │                                                     │
│ [ゲームを登録]        │ ▶ 通常起動                                         │
│ [安全導入]            │ 🛡 Strict Launch  （利用可能時のみ）                │
│                       │                                                     │
│                       │ ┌───────────────────┐ ┌──────────────────────────┐ │
│                       │ │ 🛡 保護            │ │ 💾 セーブデータ          │ │
│                       │ │ 現在の状態         │ │ 最新状態 / 詳細入口       │ │
│                       │ │ [詳細を見る]       │ │ [開く]                   │ │
│                       │ └───────────────────┘ └──────────────────────────┘ │
│                       │                                                     │
│                       │ ⚙ このゲームの設定                                │
│                       │ Scope: Game A                                      │
│                       │ [詳細設定を表示]                                   │
│                       │                                                     │
│                       │ 最近の重要な活動                                  │
├───────────────────────┴─────────────────────────────────────────────────────┤
│ Current Game Context / notification state / version                          │
└────────────────────────────────────────────────────────────────────────────┘
```

#### Active Game Variant

```text
🎮 Current Game: Game A
Game Status: 実行中
Active Session: Game A
Protection Status: 🟢 保護中
Configuration Status: カスタム設定

[ ゲームの詳細 ]    [ 保護を見る ]    [ セーブを見る ]
```

Active Gameは実際のRuntime Sessionに基づく情報として表示し、Selected Gameだけでは「実行中」としない。

#### Handoff / Navigation Variant

Protection / Save Data / Settingsへ移動した場合でも、画面上部または同等の階層で `Game A` のContextを確認できる。

    Games → Game A → Protection
    Games → Game A → Save Data
    Games → Game A → Settings

戻り先では可能な範囲でGame Listの検索・Filter・選択状態を維持する。ただし対象が外部で変更・削除された可能性がある場合は、保持したContextをそのままExecute対象に再利用しない。

### 30.16.2 Region Annotations

| ID | Region | Visual / UX rule | Safety meaning |
|---|---|---|---|
| G1 | Game List | Identity first; compact state cues only | Avoid policy detail overload |
| G2 | Selection | Selectedを視覚的に強調するがSecurity Statusとは別 | Selection ≠ execution |
| G3 | Selected Game Header | Game Name + Game Status + scoped Protection / Configuration | Prevent scope confusion |
| G4 | Launch Area | Normal LaunchをPrimary、Strictを明確な別操作として表現 | Capability / contract governs availability |
| G5 | Contextual Actions | Protection / Save Data / SettingsへCurrent Gameを保持して移動 | Prevent wrong-target operation |
| G6 | Game Settings | ScopeをGame単位で明示 | Global settingsを誤変更しない |
| G7 | Background / Discovery | Non-invasive notification surface | Do not interrupt ordinary play unnecessarily |
| G8 | Historical Game | Current / Historicalを明示 | Historical ≠ active session |

### 30.16.3 Action-State Rules

- Game selectionは副作用を持たず、選択だけでLaunch / Backup / Protection変更を開始しない。
- `通常起動` は、利用可能な既存Launch capabilityに接続する場合のみActionableとする。
- `Strict Launch` は承認済みfeature capabilityが現行Buildで利用可能な場合だけ表示・有効化し、Sandbox / Full Isolation等の未定義保証を付けない。
- Protection / Save Data / SettingsへのカードはReview / Details / Destinationへの入口であり、クリック自体をExecute済みと表示しない。
- Destructive / material actionはLaunchのPrimary操作と近接しすぎない位置へ分離する。
- Busy / Disabled / Unavailableは`01-09` / `01-17`の共通State Modelに従う。

### 30.16.4 State Variants

**No Games Registered**

一覧を空白のままにせず、Games登録という目的を明示する。これはErrorではない。

**Historical / Not Installed**

ゲーム本体が存在しない場合でも、許可された履歴・Backup情報が残るなら、現在実行可能なGameと明確に区別する。

**Unavailable**

登録情報が存在することと、現行BuildでそのLaunch / Onboarding操作が利用可能であることを分離する。

**Attention / Critical**

Game-specific warningは対象Gameとともに表示する。Critical状態は通常のGame List markerだけに縮退させず、承認済みのCritical Notification / Incident surfaceへ接続する。

### 30.16.5 Responsive / Accessibility Rules

- WideではGame List + Selected Gameを2領域で維持し、Compactでは詳細カードを1列へreflowできる。
- Game Name、Game Status、Launch、Protection / Configuration Statusが文字拡大で欠落しない。
- List stateは色だけでなくSelection、Text、Iconで区別する。
- Primary / Strict / Settings / Delete等の操作がフォント拡大で接触しないよう再配置する。
- Keyboard focus orderはGame List → Selected Game → Launch → contextual actions → settingsの順で理解できる構造を基本とする。

### 30.16.6 Wireframe Boundary

このAnnotated WireframeはGamesの情報・視覚構造を定義するものであり、Launch、Strict Launch、Onboarding、Protection等の新しいRuntime FeatureやPublic Contractを追加するものではない。Wireframe上のActionは、適用対象の既存Capabilityが成立した場合にのみActionableとする。

### 30.18 Review Findings — 2026-09-29

The first focused review identified four presentation risks and clarified them without changing runtime behavior:

| Finding | Resolution |
|---|---|
| Game List could accumulate too much per-game status detail | Added an information-budget rule and retained detail in the selected-game surface. |
| Selected Game could be mistaken for an active session | Added explicit Selected vs Active Runtime Context semantics. |
| Game-scoped and GST-wide status could be visually mixed | Added explicit status-scope guidance. |
| Strict Launch wording could imply stronger guarantees than the underlying feature provides | Added an authoritative-capability presentation boundary. |

These clarifications do not add a new Runtime Feature, Public Contract, Domain Status, Security capability, or implementation phase.

## 30.17 Games Acceptance Criteria

- ゲーム一覧と選択中ゲームの詳細が明確に分かれる。
- Game Contextを維持したまま関連画面へ移動できる。
- 起動操作が最も分かりやすい。
- 通常起動とStrict Launchの違いが明確。
- Game StatusとGST Protection Statusを混同しない。
- Game-specific Settingsのスコープが明確。
- Portable Safe Onboardingで対象と保存先を理解できる。
- 未管理ゲームの検出通知が通常操作を過度に妨げない。
- 未インストールゲームと実行中ゲームを混同しない。
- 検索・Back・FilterでGame Contextを失わない。
- 危険操作が起動操作の近くで誤クリックされにくい。
- 表示幅や文字サイズの変更で主要操作が消えない。

---

## Extraction Note

Source text was extracted mechanically from the parent UI working draft. Any future semantic change must be reviewed as a design change; an extracted statement must not be treated as a newly approved runtime requirement merely because it now has a dedicated file.

End of Document
