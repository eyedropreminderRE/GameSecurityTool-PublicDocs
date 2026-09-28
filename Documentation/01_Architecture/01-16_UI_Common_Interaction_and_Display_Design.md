# 01-16: UI Common Interaction and Display Design

**Document ID:** GST-ARCH-UI-COMMON-001  
**Version:** 0.4 (Common Interaction Annotated Wireframe / Visual UX Review)  
**Status:** Design Working Draft — Not an independent feature authority  
**Authority Relationship:** Subordinate to GST-BASELINE-011, GST-BASELINE-047, applicable approved module / feature specifications, and Change Control decisions  
**Parent Concept:** [01-08_UI_Information_Architecture_and_Interaction_Concept.md](01-08_UI_Information_Architecture_and_Interaction_Concept.md)  
**Historical Source Mapping:** Former Section 25 + Section 26 + Section 27 of the parent 01-08 UI working draft  
**Current Ownership:** This document is the maintained location for the Common Interaction / Display material after the working-draft split.
**Purpose:** Organize existing UI information-architecture and interaction design for this scope without expanding Runtime Features, Public Contracts, Domain Status, Security capabilities, or Implementation Readiness.

## Document Boundary

This document preserves UI design material that was formerly organized as Section 25 + Section 26 + Section 27 in the parent 01-08 UI working draft. After the working-draft split, 01-08 retains global IA / interaction principles while this document owns the Common Interaction / Display material.

The preserved section numbering is historical traceability only. It does not imply that those sections still exist in 01-08.

This document does not authorize Runtime implementation by itself. Existing formal baselines, approved module / feature specifications, Public Contracts, Security boundaries, and Change Control decisions remain authoritative where scope overlaps.

## Related Authoritative Documents

- [01-06_Presentation_and_UI_Architecture.md](01-06_Presentation_and_UI_Architecture.md) — Presentation architecture
- [47_User_Experience_and_Interface_Design_Governance_Model.md](../00_Baseline/Governance/47_User_Experience_and_Interface_Design_Governance_Model.md) — Formal UX governance
- [27_Quality_Assurance_and_Release_Gate_Model.md](../00_Baseline/Quality/27_Quality_Assurance_and_Release_Gate_Model.md) — UI QA authority

---

# 25. Interaction Efficiency and Navigation Usability

GSTは安全性だけでなく、日常利用時の操作負担も設計対象とする。クリック数、マウス移動、画面更新、戻る操作を「便利機能」ではなく、ユーザーが状態を把握しやすくするための基本UXとして扱う。

## 25.1 Click Budget

クリック数は一律最小化せず、操作の危険度に応じて扱う。

| 操作種別 | 目標操作量 | 原則 |
|---|---:|---|
| 状態確認 | 1〜2クリック | すぐ見られる |
| 一般的な日常操作 | 1〜3クリック | 主要導線から直接到達 |
| 詳細確認 | 2〜4クリック程度 | Contextを維持 |
| 重要な設定変更 | 追加確認を許容 | 安全性を優先 |
| データ上書き・削除・OS/Firewall変更 | 必要な確認段階を省略しない | クリック数より誤操作防止を優先 |

「クリック数を減らす」ことを理由に、Review / Confirm / Verifyを省略してはならない。

## 25.2 Mouse Travel / Fitts-Aware Placement

頻繁に使用する主要操作は、視線移動とマウス移動が少なくなるよう配置する。

原則:
- Primary actionはページ内で一貫した位置に置く。
- 相互に関連する操作は近接配置する。
- 破壊的操作と通常操作は近づけすぎない。
- 画面端・隅の固定領域は、意図せずクリックしやすい操作を置かない。
- ダイアログのConfirmとCancelは毎回位置関係を大きく変えない。
- キーボード操作でも同じ主要導線を完結できるようにする。

具体的なピクセル距離や座標はVisual Design / Accessibility検証時に実測して決定する。

## 25.3 Back Navigation

すべての主要ページは、ユーザーが直前の文脈へ戻れる明確なBack操作を持つ。

Backは単なるMain Windowの再描画ではなく、直前のユーザー文脈を可能な限り復元する。

保持対象の例:
- 選択中のゲーム
- 選択中のバックアップ / 復旧対象
- 前回の一覧フィルター・スクロール位置が安全に保持できる場合
- 現在の検索文脈

未保存の編集がある場合、Backによって変更を暗黙に確定してはならない。必要に応じてDiscard / Continue Editingの確認を行う。

BackとCancelは同義とは限らない。
- Back = 前の画面へ戻る
- Cancel = 現在の操作を取り消す

## 25.4 Refresh / 最新情報への更新

データを扱うページには、必要に応じて明示的なRefreshを提供する。

Refreshの基本契約:
```
Refresh
  ↓
最新の読み取りデータを再取得
  ↓
現在の表示を更新
```

Refreshだけで設定変更、バックアップ、削除、復旧、Firewall変更などの副作用を発生させてはならない。

更新中は「更新しています」を示し、同一操作の多重実行を安全に扱う。

最後に更新した時刻を表示できる画面では、例として「最終更新: 12:43」のように表示し、ユーザーが表示データの鮮度を判断できるようにする。

## 25.5 Auto-Refresh

Runtime Status、Protection Status、Recovery状態など時間変化が重要な画面では、明示Refreshに加えて限定的なAuto-Refreshを検討する。

Auto-Refreshは:
- CPU / I/O負荷を過度に増やさない。
- ユーザーが入力中の編集内容を上書きしない。
- 画面遷移を発生させない。
- 重要状態の変化を見逃さない。

Auto-Refreshの具体的な間隔は各機能仕様と実測で決定し、このUIコンセプトでは固定しない。

## 25.6 Stale Data Awareness

一覧や詳細画面が古い情報である可能性がある場合、ユーザーに判断材料を提供する。

表示例:
```
最終更新: 12:43
[ ↻ 更新 ]
```

外部状態が変化し得るページでは、読み取り時点を曖昧にしない。

## 25.7 Context Preservation

画面遷移によってユーザーが「今どのゲームについて見ていたのか」を失わない。

例:
```
Games
  → Game A
    → Protection
      → Network
        → Details
          → Back
        ← Network
      ← Protection
    ← Game A
  ← Games
```

詳細画面からBackした際に別ゲームや別項目へ突然移動することを避ける。

## 25.8 Search / Filter Persistence

大量のゲーム、バックアップ、イベントを扱う一覧では、Backや詳細表示から戻った際に安全な範囲で検索・フィルター状態を維持する。

ただし、セキュリティ上の理由やデータ更新により結果が変わった場合は、現在表示が最新でない可能性を明示する。

## 25.9 Long-Running Operation UX

Backup、Restore、Scan、Migrationなど時間のかかる処理では、クリック数を減らすことより「今何をしているか」を優先する。

必要表示:
- Current step
- Progress / state
- Cancellation availability
- Result / verification status

処理完了後、ユーザーが同じ操作を意図せず二重実行しないよう、完了状態を明確にする。

### 25.9.1 Requested / Executed / Verified Separation

For operations whose result matters to security, data integrity, OS state, or configuration, the UI must distinguish:

- **Requested:** the user or UI has expressed the intention to perform an operation.
- **Executed:** the operation was actually attempted or completed according to the applicable operation contract.
- **Verified:** the resulting state was checked according to the applicable verification contract.

A click, command acknowledgement, progress completion, or dialog closure must not by itself be presented as Verified.

Where the underlying feature contract exposes fewer explicit states, the Presentation layer must not invent stronger claims. It should display the strongest state that is actually supported by authoritative evidence.

## 25.10 Efficiency Acceptance Criteria

- 日常の状態確認は短い導線で到達できる。
- 日常操作は主要ページからおおむね1〜3クリックで開始できる。
- 危険操作についてはクリック削減を理由に安全確認を省略しない。
- Backで現在の対象と文脈を可能な限り維持する。
- CancelとBackの意味を混同しない。
- 最新情報が重要な画面には適切なRefreshまたはAuto-Refreshがある。
- Refreshは読み取り更新だけを行い、意図しない副作用を発生させない。
- 更新時刻やデータ鮮度を判断できる。
- キーボードのみでも主要な日常導線を完結できる。
- 具体的なマウス移動距離・ボタン寸法は後続の実機UX検証で測定する。

# 26. Window Lifecycle, Minimize, Close, and Resident Behavior

GSTはゲーム利用中のバックグラウンド保護・通知・状態監視を想定する常駐型アプリケーションであるため、Main Windowの「最小化」「閉じる」「アプリケーション終了」を別の操作として定義する。

## 26.1 Minimize

タイトルバーの最小化ボタンは、通常のWindowsアプリと同様に**ウィンドウをタスクバーへ最小化する**。

MinimizeではGSTプロセスを終了しない。Runtime Protection、監視、必要なバックグラウンド処理、通知処理は、各機能仕様の範囲で継続する。

ユーザーはブラウザ、ゲーム、その他のアプリケーションを操作した後、タスクバーからGSTウィンドウをすぐ再表示できる。

## 26.2 Close Button

Main Window右上のClose（X）は、通常の「アプリケーション終了」ではなく**Main Windowを閉じて通知領域へ退避する操作**として扱う。

基本動作:

`X` / Alt+F4
  ↓
Main Windowを非表示
  ↓
GSTは常駐継続
  ↓
通知領域（system tray）アイコンを維持

これにより、ユーザーは設定を詰めるためにブラウザ等へ移動したり、ゲームを開始したりする際、GSTを終了させずに画面だけを閉じられる。

「CloseしたのにGSTが終了していない」ことがユーザーの意図に反する可能性を考慮し、初回利用時または必要なヘルプ表示で「閉じる = 常駐継続 / 実終了 = トレイメニュー」と明示できるようにする。

## 26.3 Explicit Application Exit

GSTのプロセス自体を終了する操作は、通知領域メニューなどの**明示的な終了操作**に限定する。

例:

`GSTトレイアイコン` → `GSTを終了`

「ウィンドウを閉じる」だけでプロセス終了してはならない。

## 26.4 Exit Safety

実終了が要求された場合でも、進行中の重要処理を単純に打ち切らない。

少なくとも、Restore、Backup、Migration、Recovery、重要な設定適用などの処理が進行中なら、現在の処理状態を確認してから終了可否を決める。

終了時に安全に中断できない処理がある場合は、

```
現在、重要な処理が実行中です。
今すぐ終了すると処理を中断する可能性があります。
[処理を確認] [終了を続行] [キャンセル]
```

のように、影響を明示してユーザーに判断させる。

ただし、個別機能のtransaction/recovery仕様が別途定義されている場合は、その仕様を優先する。

## 26.5 Reopen Behavior

通知領域アイコンのダブルクリック、またはコンテキストメニューの「GSTを開く」でMain Windowを再表示する。

再表示時は、可能な範囲で直前のユーザー文脈を保持する。

保持候補:
- 最後に表示していたページ
- 選択中のゲーム
- 検索 / フィルター状態
- スクロール位置が安全に復元可能な場合

ただし、状態が外部で変化している可能性がある画面では再表示時に最新情報を再取得し、更新時刻を更新する。

## 26.6 Window Close and Unsaved Changes

Closeが「常駐退避」であることと、編集中の未保存データを破棄することは別問題である。

未保存のMaterial Editがある場合:

```
変更が保存されていません。
[保存] [変更を破棄] [キャンセル]
```

のように、編集内容の扱いを明示する。

単なる閲覧画面では、Close / Backで余計な確認を表示しない。

## 26.7 Taskbar vs Tray Model

GSTの画面状態は次のように整理する。

| 操作 | Main Window | Taskbar | Tray | GST Process |
|---|---|---|---|---|
| Minimize | 非表示 | 残る | 残る | 継続 |
| Close (X / Alt+F4) | 非表示 | 原則非表示 | 残る | 継続 |
| Tray → Open | 表示 | 表示 | 残る | 継続 |
| Tray → Exit | 終了 | 消える | 消える | 終了 |

「Minimize」と「Close」はどちらもGSTの終了ではない。

## 26.8 Tray UX

通知領域アイコンは、GSTが常駐していることをユーザーが確認できる場所として扱う。

基本メニュー候補:

- GSTを開く
- 現在の保護状態を確認
- 要対応事項がある場合は確認画面へ
- 設定を開く
- GSTを終了

Critical状態がある場合、単なるトレイアイコンの色変更だけで済ませず、承認済みのCritical Notification経路を使用する。

## 26.9 Interaction Efficiency

ユーザーが「ちょっと設定を確認したい」場合に、GSTを終了→再起動する必要がないことを基本とする。

典型例:

GSTをタスクバーから最小化 → ブラウザで調査 → タスクバーまたはトレイからGSTへ復帰

または:

GSTのX → ブラウザ / ゲーム → トレイからGSTを再表示

これらを低摩擦な日常導線として扱う。

## 26.10 Window Lifecycle Acceptance Criteria

- MinimizeでGSTプロセスを終了しない。
- Close / Alt+F4でGSTプロセスを終了しない。
- 実終了はユーザーが明示的に選択できる別操作とする。
- Main Window再表示時に安全な範囲で直前の文脈を維持する。
- 未保存変更をCloseやBackで暗黙に確定・破棄しない。
- 重要処理中の実終了では影響を明示する。
- TaskbarとTrayの役割をユーザーが混同しにくい。
- GSTを一度閉じても再起動せず素早く再表示できる。
- 本設計は既存のタスクトレイ常駐仕様と矛盾しない。

# 27. Responsive Layout, Font Scaling, Wrapping, and Display Integrity

GSTはウィンドウサイズ変更、Windows表示スケール、ユーザー指定の文字拡大、長い日本語文、専門用語、将来の多言語化によってUIが崩れないことを共通UX要件とする。

## 27.1 Fundamental Rule

ウィンドウサイズが変化したとき、基本動作は「文字を縮めて押し込む」ことではなく、

Width changes → Reflow → Wrap → Resize controls → Scroll when necessary

の順で吸収する。

本文フォントをウィンドウ幅に合わせて連続的に小さくすることは禁止する。読みやすさを損なうためである。

ユーザーが明示的に文字サイズ・表示倍率を変更した場合は、その設定を優先し、レイアウト側が追従する。

## 27.2 Window Size Tiers

Main Windowは少なくとも次の概念的なレイアウト段階を持つ。

| Tier | 状態 | 方針 |
|---|---|---|
| Wide | 十分な横幅 | 2列カード・詳細情報を表示 |
| Normal | 通常利用 | 標準レイアウト |
| Compact | 横幅が限られる | 1列化・カード再配置・補助情報の折りたたみ |
| Minimum | これ以上縮小すると操作性を維持できない | 最小ウィンドウサイズで下限を保持 |

具体的なピクセル境界は後続のVisual Design / 実機UX検証で決定する。

## 27.3 Text Wrapping

ユーザー向け文章は原則として自然な改行を許可する。

特に対象:
- 警告文
- 復旧説明
- Firewallの影響説明
- MOD / Integrity evidence
- AI Privacy説明
- 初回リスク確認
- ボタンの補助説明

固定高さのTextBlockに長文を押し込む設計を避ける。

長文を表示するコンテナは、内容量に応じて高さが伸びるか、明示的なScrollViewerにより全文へ到達できることを保証する。

## 27.4 Japanese Text and Language Expansion

日本語では英語と文字幅・改行位置・情報密度が異なるため、英語文を前提に固定幅を決めない。

多言語化を考慮し、
- 翻訳によって文字列が長くなること
- 日付 / 数値 / 単位表現が変わること
- ボタン文言が長くなること
- エラーメッセージが複数行になること

を前提とする。

重要なボタンに「英語なら収まる」という理由で固定幅を設定しない。

## 27.5 Typography Hierarchy

文字サイズは意味に応じた階層を持つ。

```
Page Title
  ↓
Section Heading
  ↓
Primary State
  ↓
Body / Explanation
  ↓
Secondary Detail
  ↓
Technical Detail
```

狭いウィンドウでは、詳細情報を折りたたんだり再配置したりすることを優先し、Body / Explanationの可読性を犠牲にしない。

## 27.6 User Font / Display Scaling

GSTはWindowsの表示スケールやユーザーのアクセシビリティ設定を前提として設計する。

ユーザーが文字を大きくした場合:
1. Textは折り返す。
2. コンテナは必要に応じて高さを増やす。
3. 2列レイアウトは1列へ移行できる。
4. 補助情報を折りたたむ。
5. 必要ならスクロール可能にする。

禁止:
- 重要本文だけを自動的に極端に縮小する。
- ボタン文字を切り捨てる。
- 警告文を途中で省略して意味を変える。
- 文字サイズ変更を理由にConfirmボタンを画面外へ押し出す。

## 27.7 Responsive Cards

Dashboard等のカードは固定幅を前提にせず、利用可能幅に応じて:

Wide → 2列
Normal → 2列または1列
Compact → 1列

へ再配置できる構造とする。

カード内の重要な状態は、カード幅が変わっても最初に読める位置へ残す。

## 27.8 Controls and Buttons

ボタンは文字の増減に耐えることを優先する。

原則:
- ボタンの文字を省略記号で切らない。
- 危険操作の名称を短くするために意味を削らない。
- Primary / Secondary / Destructiveの視覚的役割を維持する。
- Confirm / Cancelの位置関係を各ダイアログで可能な限り統一する。
- キーボードフォーカスでも全操作へ到達できる。

## 27.9 Tables and Long Technical Values

Audit、File Lock、Firewall、Migration等で長い値を扱う場合、表示崩れを防ぐために列を固定しすぎない。

長い値の扱い:
- 重要な表示名は可能な範囲で折り返す。
- パス、ハッシュ、GUID等は表示領域に応じて省略表示できるが、詳細表示で完全値を確認できること。
- PID等の短い値を不必要に折り返さない。
- コピー可能な値は、コピー操作を提供できる場合に明示する。

## 27.10 No-Data, Loading, and Error Layout

データ件数が0、読み込み中、読み込み失敗の各状態でもレイアウトの骨格を維持する。

No data → 目的説明 + 次の操作
Loading → 現在ページの枠を維持 + Loading表示
Error → 問題 + 原因/制限 + 次の操作

状態変化のたびにコントロールが大きく移動し、意図しないクリックを誘発する設計を避ける。

## 27.11 Scroll Policy

スクロールは「表示崩れを隠すため」ではなく、情報量が物理的に画面へ収まらない場合の正式なUI手段とする。

原則:
- ページ全体Scrollとカード内Scrollを必要以上に入れ子にしない。
- 重要操作を見えない位置へ隠し続けない。
- Review / Confirm画面では影響範囲とConfirm操作を同じ視野で確認できることを優先する。
- 長い詳細情報は折りたたみ + 詳細Viewを利用できる。

## 27.12 Dialog Resize and Safety

重要ダイアログはウィンドウサイズや文字サイズが変化しても、Target / Action / Impact / Result expectation / Confirm / Cancelへ到達できることを保証する。

Confirmが画面外へ消える、説明文が重なって読めない、Cancelが隠れる、といった状態を許容しない。

## 27.13 Window Minimum Size

Main Windowおよび重要ダイアログには、主要操作を維持できる最小サイズを設ける。

最小サイズへ到達した後はさらに縮小させるのではなく、OS標準のサイズ変更制約により下限を維持する。

最小サイズの数値は、最終UI実装と実機アクセシビリティ検証で決定する。

## 27.14 Reflow Test Matrix

UI受入時には、少なくとも次の組み合わせを確認する。

| Condition | Required Check |
|---|---|
| Wide window | 通常レイアウト |
| Normal window | 標準操作 |
| Compact window | 1列化 / 折り返し |
| Minimum window | 主要操作が隠れない |
| 大きいWindows表示スケール | 文字切れなし |
| ユーザー文字拡大 | 説明文・警告文が読める |
| 長い日本語 | 自然な改行 |
| 長い英語 | ボタン・ラベル崩れなし |
| Long path / hash / GUID | 詳細確認可能 |
| Loading | レイアウトの急変なし |
| Error | 次の操作へ到達可能 |

具体的な表示スケール値やピクセル寸法は、後続の実機テストで測定・確定する。

### 27.16 Display Integrity and Action-State Coupling

Responsive layout behavior must preserve not only visibility of controls but also the semantic relationship between controls and their current action state.

When wrapping, reflow, or resizing occurs:

- a disabled or busy action must not visually resemble an executable action;
- Confirm must remain associated with the operation being reviewed;
- progress and verification results must remain visually attached to the same operation;
- a moved or collapsed control must retain its accessible name and state;
- a responsive transformation must not reorder controls in a way that makes destructive actions appear to be the default action.

### 27.14.1 Annotated Wireframe — Common Interaction / Display Integrity Working Baseline

このAnnotated Wireframeは、画面固有の機能を追加せず、GST共通のInteraction / Display安全境界を視覚的に確認するための作業基準である。

```text
┌────────────────────────────────────────────────────────────────────────────┐
│ GST / Current Context                                      画面タイトル       │
│ [← Back]                                               [Help / Details]      │
├────────────────────────────────────────────────────────────────────────────┤
│ Primary Context / Target                                                     │
│ Game A · Save Data · Generation 2026-09-28 23:10                            │
│                                                                           │
│ Current State                                                               │
│ 🟡 注意 / 現在の状態を確認してください                                   │
│ Scope: このContext                                                        │
│                                                                           │
├────────────────────────────────────────────────────────────────────────────┤
│ Primary Action Area                                                         │
│ [詳細を見る]    [Review]    [Execute / Apply]                             │
│                                                                           │
│ ※ Actionable / Disabled / Busyは状態を視覚的・アクセシブルに区別する        │
├────────────────────────────────────────────────────────────────────────────┤
│ Operation / Progress                                                        │
│ Step 2 / 4   現在: 保全中                                                   │
│ [████████████░░░░░░]                                                       │
│ Cancel: 利用可能 / 不可                                                     │
│                                                                           │
│ Result: 完了                                                               │
│ Verification: 確認中                                                      │
├────────────────────────────────────────────────────────────────────────────┤
│ Attention / Error                                                          │
│ ⚠️ 確認できていない項目があります。                                       │
│ [確認する]                                                                 │
├────────────────────────────────────────────────────────────────────────────┤
│ Safe Exit: Back / Cancel / Close はそれぞれ異なる意味を保持                │
└────────────────────────────────────────────────────────────────────────────┘
```

#### 27.14.1.1 Region Annotations

| ID | Region | Visual / UX rule | Safety meaning |
|---|---|---|---|
| C1 | Context Header | Target / Scope / Originを明示 | 状態と対象の取り違えを防止 |
| C2 | Current State | StateとScopeを近接表示 | 古い / 別Contextの状態を誤認しない |
| C3 | Action Area | Review / Execute / Disabled / Busyを明確に区別 | 状態表示が実行可能性を偽装しない |
| C4 | Progress | Current Step / progress / cancellationを同一Operationへ結合 | 別Operationとの混線を防止 |
| C5 | Result / Verification | ResultとVerificationを分離 | 100% / CompletedをVerifiedと誤認しない |
| C6 | Attention / Error | Problem / Impact / Next Actionを隣接 | エラーを閉じただけで解消したと誤認しない |
| C7 | Safe Exit | Back / Cancel / Closeを別ラベル・別意味で表示 | navigationとpending-operation中止を混同しない |
| C8 | Details / Help | Read-only entryとして扱う | 詳細表示が副作用を開始しない |

#### 27.14.1.2 Common Action Lifecycle

```text
User intent
    ↓
Requested
    ↓
Review / Confirm when required
    ↓
Executed / Attempted
    ↓
Observe / Reconcile
    ↓
Verified / Needs Attention / Failed
```

- Click、keyboard activation、dialog close、progress 100%だけをVerifiedとして表示しない。
- Underlying Contractが明示しない状態をUIだけで新設しない。
- `Requested ≠ Executed ≠ Verified` をすべての高影響Operationで維持する。
- Verify不能な場合は、完了表示をVerification完了へ丸めない。

#### 27.14.1.3 Back / Cancel / Close Mapping

| Control | Meaning | Must not imply |
|---|---|---|
| Back | 前の画面 / Contextへ移動 | Execute済みOperationのUndo |
| Cancel | 未実行のPending operation / Dialog / Reviewを中止 | Execute済み変更のRollback |
| Close | Window / Dialogを閉じる | Operationの自動取消し・Undo |

高影響OperationのExecution後にBack / Closeを行っても、Result / Verification / unresolved stateを失わせない。Rollback / Recoveryが存在する場合は、その正式な操作導線を利用する。

#### 27.14.1.4 Responsive State Preservation

- WideではContext → State → Action → Progress / Result → Attentionの意味順を維持する。
- Compactでは1列へ再配置できるが、Confirm対象とActionの対応関係を崩さない。
- 文字拡大時にPrimary Actionだけを残してImpact / Confirmationを削除しない。
- Busy / Disabled actionは、reflow後も実行可能なPrimary Actionと視覚的に同じ表現へ変化しない。
- ProgressとVerificationは折り返しやカード移動後も同一Operationへ属することが視覚的に分かる。
- 長い日本語・英語・Path / Hash / GUIDは、重要な安全情報を隠さない範囲で折り返しまたはDetailsへ退避する。
- Window minimum sizeでもBack / Cancel / Closeと必要なPrimary safety controlへ到達できる。

#### 27.14.1.5 Notification / Dismiss Boundary

- NotificationはUnderlying stateの要約・導線であり、Dismissは状態解消を意味しない。
- Critical notificationを閉じても、対応が必要なUnderlying Critical stateは画面またはPersistent surfaceで確認できる。
- Toastの成功表示だけでResult / Verification stateを上書きしない。

#### 27.14.1.6 Wireframe Boundary

このAnnotated Wireframeは共通Interaction / Display patternを視覚化するものであり、新しいFeature、Public Contract、Domain Status、Security Capability、Window Lifecycle behaviorを追加しない。

各画面の具体的Actionは、その画面の既存Feature / Module / ContractとCapability条件に従う。共通Patternが存在することだけを理由に、UIから新しいExecute / Verify / Recovery操作を生成しない。
## 27.15 Display Integrity Acceptance Criteria

- ウィンドウを縮小しても重要本文が読める。
- 本文フォントを幅に合わせて極端に縮小しない。
- 長文は自然に改行される。
- 固定高さによる文章の切断がない。
- 文字拡大時にボタンやConfirm/Cancelが画面外へ消えない。
- Compact layoutで重要情報の優先順位を維持する。
- 長いパスやハッシュなどの技術情報は省略されても完全値へ到達できる。
- No Data / Loading / Errorでも画面骨格と主要導線を維持する。
- Windows表示スケールとユーザー指定文字サイズの双方で確認する。
- 日本語を基準としつつ将来の文字列長増加に耐える。
- 主要操作は最小ウィンドウサイズでも到達できる。
- キーボード操作でも表示崩れによって操作不能にならない。


## 27.16 Review Findings — 2026-09-29

The focused review identified three cross-screen semantic risks and clarified them:

| Finding | Resolution |
|---|---|
| A requested action could be visually mistaken for an executed or verified result | Added Requested / Executed / Verified separation. |
| Back / Cancel / Close semantics could drift between screens | Added one shared semantic definition. |
| Responsive reflow could preserve visibility while weakening the relationship between an action and its state | Added action-state coupling requirements for responsive layouts. |

These are UI semantics and presentation rules only. They do not create new Runtime states or operation contracts.

---

## Extraction Note

Source text was extracted mechanically from the parent UI working draft. Any future semantic change must be reviewed as a design change; an extracted statement must not be treated as a newly approved runtime requirement merely because it now has a dedicated file.

End of Document
