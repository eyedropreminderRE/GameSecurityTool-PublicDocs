# 01-17: UI Reference Patterns and Common Component Library

**Document ID:** GST-ARCH-UI-PATTERNS-001  
**Version:** 0.3 (Annotated Common Pattern / Visual UX Review)  
**Status:** Design Working Draft — Not an independent feature authority  
**Authority Relationship:** Subordinate to GST-BASELINE-011, GST-BASELINE-047, applicable approved module / feature specifications, and Change Control decisions  
**Parent Concept:** [01-08_UI_Information_Architecture_and_Interaction_Concept.md](01-08_UI_Information_Architecture_and_Interaction_Concept.md)  
**Historical Source Mapping:** Former Section 28 + Section 29 of `01-08_UI_Information_Architecture_and_Interaction_Concept.md`  
**Current Ownership:** This document is the maintained location for the former Section 28/29 material after the working-draft split.
**Purpose:** Organize existing UI information-architecture and interaction design for this scope without expanding Runtime Features, Public Contracts, Domain Status, Security capabilities, or Implementation Readiness.

## Document Boundary

This document preserves the UI design material that was formerly organized as Sections 28 and 29 of `01-08_UI_Information_Architecture_and_Interaction_Concept.md`. After the working-draft split, `01-08` retains global IA / interaction principles while this document owns the common-pattern material.

The preserved section numbering is historical traceability only. It does not imply that Sections 28/29 still exist in `01-08`.

This document does not authorize Runtime implementation by itself. Existing formal baselines, approved module / feature specifications, Public Contracts, Security boundaries, and Change Control decisions remain authoritative where scope overlaps.

## Related Authoritative Documents

- [47_User_Experience_and_Interface_Design_Governance_Model.md](../00_Baseline/Governance/47_User_Experience_and_Interface_Design_Governance_Model.md) — Formal UX governance
- [11_UI_UX_and_User_Interaction_Model.md](../00_Baseline/Governance/11_UI_UX_and_User_Interaction_Model.md) — Formal UI/UX baseline

---

# 28. Reference UI Pattern Research

この章は、GSTのUIへ取り入れる価値がある既存ソフトウェアのUIパターンを調査した結果である。特定製品の画面を模倣するものではなく、既存のGST UX / Security Baselineと矛盾しないパターンだけを採用候補とする。

## 28.1 Windows Security — Status Hub + Action-Oriented Detail

Windows SecurityはHomeをセキュリティ領域への中心ハブとして使い、状態表示と各保護領域への迅速な導線を提供する。セキュリティプロバイダーでは、カテゴリをカードとしてまとめ、必要な詳細を展開する構成も使われている。citeturn698855search0turn698855search6

GSTでは次の情報階層を採用候補とする:
全体文脈 → 目的別保護領域 → 現在状態 → 次の行動 → 必要なら詳細

ただし、Windows Securityの見た目や「安全度」表現をそのまま移植しない。GSTではProtection StatusとConfiguration Statusの意味を既存仕様どおり独立させる。

## 28.2 Windows NavigationView — Adaptive Navigation

MicrosoftのNavigationViewは、横幅に応じて展開型、コンパクト型、最小型へナビゲーションを適応させる設計を提供している。citeturn428657search0turn428657search1

GSTでは:
Wide → アイコン + ラベル
Compact → アイコン中心
Narrow → コンパクトなナビゲーション入口

という原則を採用候補とする。これは本文フォントを縮小せず、ナビゲーション側で画面密度を調整するために有効である。

## 28.3 Windows BreadcrumbBar — Persistent Context

BreadcrumbBarは現在位置への経路を表示し、深い階層から以前の場所へ戻りやすくするパターンである。幅が足りない場合に左側の履歴を省略し、展開して確認する適応も定義されている。citeturn249095search0turn249095search6

GSTでは深い画面に限定して利用する候補とする。
例: Games → Game A → Protection → Network → Details
例: Save Data → Game A → Backup → Generation 3 → Details
浅い画面では通常のBack操作を優先する。

## 28.4 Windows Navigation Guidance — Avoid Pogo-Sticking

Microsoftは、関連情報を見るたびに親画面へ戻って再び子画面へ入る「pogo-sticking」を避けることをナビゲーション上の原則としている。citeturn249095search1

GSTでは選択中のゲームなどのContextを保持し、関連項目へ直接移動できる導線を設ける。

Games → Game A
Game A → Overview / Protection / Save Data / Activity

といったContextual Navigationを候補とする。

## 28.5 VS Code — Command Palette / Search-First Access

VS CodeのCommand Paletteは多くのコマンドへ一つの検索窓から到達でき、Quick Openでは名前を入力して項目へ直接移動できる。citeturn748012search4

GSTではGlobal Searchを単なるヘルプ検索ではなく、ページ、設定、ゲーム、マニュアル、Recovery Guidanceを横断して探す入口にする。

ただし、検索から危険操作を即時実行する設計にはしない。検索で見つけた後も既存のReview / Confirm / Execute契約を通す。

## 28.6 VS Code — Settings Search and Modified Filter

VS CodeのSettings Editorは検索による設定絞り込みを提供し、既定値から変更された項目だけを表示するModifiedフィルターも持つ。citeturn249095search5

GSTでは次の設定ビューを採用候補とする:
All / Recommended / Modified

特にModifiedは「自分がStandardから何を変えたか」を理解するために有効で、既存のStandard / Custom状態と相性がよい。

## 28.7 PowerToys — Quick Access + System Tray

PowerToysはバックグラウンド動作とSystem Trayを組み合わせ、Quick Accessから設定を素早く開いたり、キーボード中心のクイックランチャーを提供している。citeturn748012search0turn748012search1turn748012search9

GSTでは、Trayから:
GSTを開く / 現在状態を確認 / 要対応項目へ移動 / 設定を開く / GSTを終了

程度の小さなQuick Accessを採用候補とする。

Quick Accessから危険操作を直接実行して、通常の確認フローを迂回させない。

## 28.8 OneDrive — Glanceable Status Markers

OneDriveは、ファイルに状態アイコンを付け、同期状態やローカル可否などを一目で把握できるようにしている。citeturn748012search5turn748012search6

GSTでは対象オブジェクトの近くに状態を置く考え方を採用候補とする。
Game A / Backup / Recoveryなどに、状態を対応付けて表示する。

ただしGSTは色だけに意味を持たせない。既存のProtection Status語彙を優先し、テキスト・アイコン・説明を組み合わせる。

## 28.9 Windows Notifications — Progress Without Reopening

Microsoftの通知ガイダンスでは、長時間処理の進捗を通知側で示し、ユーザーがアプリを開き直さなくても状態を把握できるパターンが示されている。citeturn748012search7

GSTではBackup / Restore / Scan / Migrationなどの長時間処理で採用候補とする。

通知に進捗を表示しつつ、必要時にはGST本体へ戻れる導線を提供する。

Critical security/data-protection notificationの扱いはGST既存ルールを優先する。

## 28.10 Windows Dialogs and Flyouts — Interrupt Only When Necessary

Microsoftは、重要なブロッキング判断や確認にはDialog、追加情報やコンテキスト詳細にはFlyout等を使い分ける考え方を示している。頻繁な操作には、毎回の確認を強制する代わりにUndoを設ける選択肢も示されている。citeturn249095search2turn249095search7

GSTでは:
Inline / Flyout → 詳細説明、補足情報、軽い確認
Dialog → 重要な判断、承認、危険操作

とする。

特にゲーム中に毎回モーダルを表示しないよう、非クリティカルな説明はInline / Flyoutへ寄せる。ただしGSTの安全境界でConfirmが要求されている操作は、その要件を維持する。

## 28.11 Reference Pattern Adoption Matrix

| Pattern | GST Use | Adoption Status | Authority / Capability Gate |
|---|---|---|---|
| Status hub + actionable cards | Home / Protection | Working Design | Existing state semantics only |
| Adaptive navigation | Main Window | Working Design | Presentation-only reflow |
| Breadcrumbs | Deep detail / recovery | Working Design | Navigation / context only |
| Contextual related navigation | Games / Save Data | Working Design | Existing destinations only |
| Command Palette / search-first | Global Search | Capability-Dependent Working Design | Search capability must exist in an approved scope; never creates new Execute paths |
| Modified settings filter | Settings | Capability-Dependent Working Design | Existing Settings model must expose a meaningful Modified state |
| Tray quick access | Resident operation | Capability-Dependent Working Design | Existing resident/tray behavior governs lifecycle and exit semantics |
| Object-attached status markers | Games / backups / recovery | Working Design | Canonical status only; no heuristic state synthesis |
| Progress notifications | Long-running operations | Capability-Dependent Working Design | Existing operation/progress contract only |
| Inline vs dialog distinction | All interaction design | Working Design | Confirm requirements from applicable authority remain mandatory |

## 28.12 Patterns Not Adopted Blindly

人気ソフトのUIだからという理由だけで採用しない。

採用しない方向:
- 単一の総合Security Score
- 通知を大量に出すセキュリティUI
- 説明なしの隠しExpert操作
- Command Paletteから確認を迂回して危険操作を即実行
- 色だけによる状態表示
- 英語固定幅前提のカード
- 深い階層への反復移動を要求するメニュー
- 軽微な情報まで全画面モーダルで表示

採用評価は、操作負荷の低減、理解しやすさ、ユーザーの判断権、GSTの安全境界の4条件を同時に満たすことを基準とする。

## 28.13 Research Basis

調査対象は主にMicrosoft Windows Security、Windows App / Fluent Design guidance、VS Code、Microsoft PowerToys、OneDriveの現行公開UI/UX資料である。

これらから得た知見はUIパターンの参考資料であり、GSTの製品仕様・Security Boundary・Change Control・UX Baselineの権威ではない。

# 29. GST Common UI Pattern Library

この章では、複数画面で繰り返し使用するUI部品と操作パターンを共通化する。これはVisual Design Systemそのものではなく、情報の意味・配置・操作責務を統一するためのPattern Libraryである。

## 29.1 Common Pattern Design Rule

共通UI部品は表示とユーザー操作を担当し、Security判定・Domain判断・Infrastructure操作を内部に持たない。

したがって:

View = visual presentation
ViewModel = UI state and interaction orchestration
Application = user operation/use-case
Domain = business/security rules
Infrastructure = OS/filesystem/database integration

共通部品の導入を理由にこの責務境界を崩さない。

### 29.1.1 Adoption and Capability Gate

Pattern Libraryの記載だけで、Runtime capability・Public Contract・Domain State・Security capabilityが存在すると扱ってはならない。

Patternは次の3段階で扱う。

- **Working Design** — 現在のUI Working Draftで採用する表示・操作原則。ただし既存の権威的な機能・状態・Contractの範囲を超えない。
- **Capability-Dependent Working Design** — UI上の利用形は定義するが、実際に表示・有効化できるかは適用対象のFeature / Module / Contract / Build availabilityに従う。未成立なら未実装操作を推測で表示しない。
- **Reference Only** — 外部製品から得た参考パターンであり、GSTへの採用を意味しない。

共通コンポーネントは、存在しないデータ項目・存在しない状態・存在しない操作をUI都合で新設してはならない。

特にSearch、Tray、Progress、Modified filter等は、見た目のPatternが定義されていても、それ自体を新Featureの承認とはみなさない。

## 29.2 App Shell

Application ShellはGST全体で固定される外枠。

構成候補:
- Window title / identity
- Primary Navigation
- Current Context
- Global Search
- Help
- Page Content
- Context / status area

各ページは独自のMain Navigationを再定義しない。

## 29.3 Page Header

各主要ページは、ページタイトルと目的を最初に示す。

基本:
Page Title
Purpose / short explanation
Optional primary action

必要な場合のみBreadcrumbとCurrent Contextを追加する。

## 29.4 Breadcrumb / Back

Backは直前のページへ戻るための共通操作。
Breadcrumbは深い階層の現在位置を示すための補助操作。

両者を常に同時表示する必要はない。

深い階層:
Games → Game A → Protection → Network → Details

Backによって検索・フィルター・選択対象などの安全な文脈を維持する。

## 29.5 Context Header

ゲームやバックアップなど対象が明確な画面では、画面上部に対象を固定表示できる。

例:
🎮 Game A
Protection / Network

Context Headerの目的は「現在何を操作しているのか」を常に見失わないことであり、対象固有のSecurity判定を実行する場所ではない。

## 29.6 Status Marker

対象の現在状態を対象の近くに表示する小さな共通パターン。

表示例:
🎮 Game A   🟢 保護中
💾 Backup   ✓ 完了
↩ Recovery  ⚠ 要確認

色だけで意味を伝えない。

Protection Statusについては既存の公式語彙を使用し、「緑だから安全」という独自解釈を追加しない。

## 29.7 Status Card

Status CardはHome / Protectionなどで状態を短時間で把握するためのカード。

基本構造:
Title → Current State → Short Explanation → Details / Action

Status Cardは「今の状態」を表示するが、総合Security Scoreや独自Risk Scoreを生成しない。

## 29.8 Needs Attention Card

要対応状態が存在するときだけ表示するコンポーネント。

基本構造:
Problem → Impact → What to do → Review / Details

通常状態では大きな空白や不要な警告を残さない。

Critical状態については通常のNeeds Attention Cardより上位の通知ルールを使用する。

## 29.9 Action Bar

ページ内の主要操作をまとめる共通領域。

原則:
Primary actionは、そのタスク文脈で中心となる操作を原則1つにする。
複数の独立した同等目的がある場合、クリック数削減のために無理に1つを「唯一の正解」のように扱わない。
Secondary actionsは必要最小限。
Destructive actionsは通常操作から視覚的・空間的に区別する。

Action Barは操作を集約するが、危険操作の確認要件を短絡させない。

## 29.10 Details Expander

技術的な説明や証拠を必要なユーザーだけが展開するための共通パターン。

通常表示:
目的 / 状態 / 影響

Details:
技術情報 / 詳細理由 / 許可された証拠 / 技術値（権威仕様・Privacy境界で許可される範囲のみ）

秘密情報や不要な個人情報を「詳細だから」という理由で表示しない。

## 29.11 Refresh Control

読み取り情報を最新状態へ再取得する共通操作。

表示候補:
最終更新: 12:43
[ ↻ 更新 ]

Refreshは副作用を持たず、設定変更・Restore・Deleteなどを開始しない。

更新中は多重実行を安全に抑制し、完了後は更新時刻を再評価する。

## 29.12 Loading / Empty / Unavailable / Error State

一覧・詳細ページでは、少なくとも次の意味を区別できる設計とする。

Loading:
現在ページの構造を可能な限り維持 + Loading表示

Empty:
その機能は現在利用可能だが、対象データが存在しない状態。「なぜ空なのか」と「次に何をすべきか」を示す。

Unavailable:
対象機能または対象操作を現在利用できない状態。Not Applicable、依存関係未成立、権限 / 環境制約、または現行Buildの可用性など、分かっている理由を示す。未実装機能を将来利用できることまで推測させる表現は使用しない。

Error:
本来利用可能な処理で予期しない失敗が発生した状態。何が起きたか / 制限 / 次の行動 / RetryまたはDetailsを示す。

EmptyをError風に表示せず、Unavailableを「データが空」と誤認させない。

## 29.13 Inline Help / Flyout

ユーザーを別ページへ移動させずに補足説明を表示する。

適用例:
- 設定項目の意味
- Firewallの通信方向説明
- Statusの定義
- Backup保持期間の説明

重要な承認や危険操作にはFlyoutだけで済ませず、正式な確認UIを使用する。

## 29.14 Confirmation Dialog

適用対象のFeature / Module / ContractがMaterial ChangeまたはHigh-Risk Operationとして定義する操作の共通確認パターン。

Minimum content:
Target
Action
Impact
Expected Result
Confirm
Cancel

可能な場合は既存のReview → Confirm → Execute → Verifyへ接続する。

## 29.15 Progress Surface

長時間処理の共通進捗表示。

表示候補:
Operation name
Current step
Progress / state
Cancellation availability
Result / verification

進捗表示そのものが完了を意味するとは限らないため、最終結果とVerificationを区別する。
進捗表示は、Underlying operationが実際に存在することを前提とし、存在しない処理をUIだけで作らない。

## 29.16 Notification Surface

通知はCritical / Important / Informationalの階層を共通化する。

通知には可能な範囲で:
Context
Event
Current state
Next action

を含める。

ゲーム中は通常通知の表示を抑制できるが、Critical security/data-protection notificationは既存仕様に従い完全抑制しない。
通知を閉じてもCritical状態そのものが消えたこと、解決済みであること、Verifiedであることを意味させない。

## 29.17 Search Surface

Global Searchは、承認済みの検索能力が存在する場合に、ページ・設定・ゲーム・Manual・Recovery Guidanceへの入口として使用する。

検索結果は「移動先」と「説明」を明確に区別する。

検索結果からMaterial / High-Risk operationを直接Executeする設計は採用しない。検索結果は既存のReview / Confirm / Execute / Verify境界へ接続する。

## 29.18 Settings Control Pattern

設定項目は可能な限り次の形で表示する。

Setting Name
Purpose
Current value
Standard value where relevant
Scope
Impact
Details

Standardから変更された項目はModifiedとしてフィルター可能な設計を採用候補とする。

## 29.19 Contextual Action Links

関連画面へのリンクを、ユーザーが現在の文脈を失わずに移動できるようにする。

例:
Game A → [保護を見る] [セーブを見る] [アクティビティを見る]

これはPogo-Sticking回避のためのNavigation Patternであり、各リンク先で現在対象を再選択させる設計を避ける。

## 29.20 Resident / Tray Quick Access

System TrayからGSTの状態を把握し、Main Windowへ戻れる小規模Quick Accessを設ける設計を採用候補とする。実際の常駐・終了・再表示semanticsは、既存のWindow Lifecycle / resident behaviorおよび適用仕様を優先する。

Quick Accessは:
- GSTを開く
- 状態確認
- 要対応項目への移動
- 設定を開く
- GSTを終了

程度に限定する。

Restore / Delete / Firewall Change等の高リスク操作をQuick Accessの直接実行項目にはしない。

## 29.21 Common Placement Rules

共通操作について可能な範囲で配置を揃える。

| Element | Preferred Role |
|---|---|
| Back | 上部の安定した位置 |
| Breadcrumb | Page Header周辺 |
| Refresh | 一覧/状態情報の近傍 |
| Primary Action | Page HeaderまたはAction Bar |
| Details | 状態説明の直後 |
| Confirm / Cancel | ダイアログ下部で一貫配置 |
| Help | Headerの共通領域 |

厳密なピクセル座標はVisual Design段階で決定する。

## 29.21.1 Annotated Wireframe — Common Pattern / Component Library Working Baseline

このAnnotated Wireframeは、複数画面で再利用するCommon Patternの配置・意味・状態表現を統一するための作業基準である。PatternそのものはCapabilityやDomain判断を持たず、適用対象の既存Feature / Module / Contractに従う。

```text
┌────────────────────────────────────────────────────────────────────────────┐
│ [← Back]  GST / Current Context                         [Help] [Refresh]   │
├────────────────────────────────────────────────────────────────────────────┤
│ Page Title                                                                  │
│ Purpose / short explanation                                                 │
│ 🎮 Game A · Save Data · Generation 2026-09-28 23:10                       │
├────────────────────────────────────────────────────────────────────────────┤
│ Status                                                                       │
│ 🟡 要確認   現在の状態 / Scope / 短い説明                                 │
│ [詳細]                                                                      │
├────────────────────────────────────────────────────────────────────────────┤
│ Action Bar                                                                   │
│ [Primary / Review]    [Secondary]    [Destructive — separated]             │
│                                                                              │
├────────────────────────────────────────────────────────────────────────────┤
│ Page Content                                                                 │
│                                                                              │
│ Contextual content / list / setting / status cards                         │
│                                                                              │
│ ┌────────────────────────────────────────────────────────────────────────┐ │
│ │ Needs Attention                                                        │ │
│ │ Problem → Impact → What to do                                         │ │
│ │ [Review / Details]                                                     │ │
│ └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│ Operation / Progress (when an existing operation is active)                │
│ Current Step → Progress / State → Cancellation → Result → Verification     │
├────────────────────────────────────────────────────────────────────────────┤
│ Navigation / Safe Exit: Back ≠ Cancel ≠ Close                              │
└────────────────────────────────────────────────────────────────────────────┘
```

### 29.21.1.1 Pattern Region Annotations

| ID | Pattern | Visual / UX rule | Boundary |
|---|---|---|---|
| CP1 | App Shell / Page Header | Identity → Title → Purpose → Contextの順 | Shellは各Pageの責務を決めない |
| CP2 | Context Header | Target / Scopeを状態の近くに保持 | Context表示はSecurity判定を実行しない |
| CP3 | Status Marker / Card | Text + Icon + placementで状態を表現 | 色やPatternから新しいStatusを推測しない |
| CP4 | Action Bar | Primary / Secondary / Destructiveを視覚的に分離 | Patternは確認要件を短縮しない |
| CP5 | Needs Attention | Problem → Impact → What to do | DismissでUnderlying stateをResolvedにしない |
| CP6 | Details Expander | 技術情報を段階開示 | Details表示は副作用を持たない |
| CP7 | Progress | Step / State / Cancel / Result / Verifyを関連付ける | Progress 100%からSuccessを自動生成しない |
| CP8 | Notification | Context / Event / State / Next actionを近接表示 | 通知閉鎖 ≠ 解決 / Verified |
| CP9 | Search | Discovery / Navigationとして表示 | Search結果からHigh-impact Executeを直接開始しない |
| CP10 | Settings Control | Current / Standard / Scope / Impactを表示 | UI変更を実効状態へ自動変換しない |
| CP11 | Contextual Link | 移動先と現在Contextを明確化 | Navigation linkがExecuteを意味しない |
| CP12 | Tray Quick Access | 状態確認・Navigation中心に限定 | 高リスク操作の直接Executeを提供しない |

### 29.21.1.2 Common Pattern State Mapping

Common Patternの状態表現は、既存の共通State Modelを再利用する。

```text
Interactive control
  Default → Hover → Focus → Pressed → Disabled
                           └→ Loading / Busy

Data / Page
  Loading / Empty / Partial / Error / Unavailable

Operation
  Requested → Executed / Attempted → Observe / Reconcile → Verified / Needs Attention / Failed
```

- Patternは、適用対象のFeature / Contractが公開していない状態を独自に生成しない。
- `Selected` はNavigation / selection contextを示し、Protection Statusを意味しない。
- `Disabled` はユーザー設定による無効化、`Unavailable` は現在の環境・Build・Contextで利用不能という既存意味を保持する。
- `Resolved` はUnderlying authoritative stateの解消を必要とし、通知のDismissやCardの非表示だけでは成立しない。

### 29.21.1.3 Capability-Dependent Pattern Presentation

| Pattern | Working Design | Capability gate |
|---|---|---|
| Search | 探索・移動UI | Search capabilityが成立している場合のみ利用可能 |
| Tray Quick Access | 状態確認・再表示・Navigation | Resident / Tray仕様が成立している場合のみ |
| Progress | 長時間Operationの進捗 | 対応する既存Operationが実在する場合のみ |
| Modified Filter | 変更済み設定の絞り込み | 既存Settings検索 / filter能力が成立している場合のみ |
| Contextual Action Link | 関連画面へのNavigation | destinationが正式に存在する場合のみ |
| Confirmation Dialog | 高影響Operationの確認 | 適用対象Contractの確認要件に従う |

見た目のPatternを先に実装して、存在しないCapabilityを後から接続する構成は採用しない。未成立の場合はUnavailable / Not Available等の既存Presentation semanticsへ変換する。

### 29.21.1.4 Responsive / Accessibility Rules

- WideではPage Header → Status → Action Bar → Content → Attention / Progressの意味順を維持し、Compactでは1列へ再配置できる。
- 文字拡大時にPrimary actionだけを残してTarget / Impact / Confirmationを削除しない。
- PatternのstateはColorだけでなくText / Icon / Accessible Nameで区別する。
- Action Barを折り返す場合も、Destructive actionがDefault actionのように先頭へ移動しない。
- ProgressとVerification resultが別Operationの情報に見えないよう、同一Operationのまとまりを維持する。
- Search結果、Contextual Link、Details ExpanderはKeyboard navigationでも通常のReview / Execute境界を迂回しない。
- Tray / Notificationなど補助Surfaceへ情報を移動しても、Critical stateや必要なRecovery導線を失わせない。

### 29.21.1.5 Pattern-to-Authority Boundary

Common Patternは、次の順で既存の権威を参照する。

```text
Formal Baseline / Approved Feature / Module / Contract
                    ↓
          Existing authoritative state
                    ↓
            Common Pattern mapping
                    ↓
             Visual presentation
```

逆方向、すなわちPatternの存在から新しいRuntime Feature、Public Contract、Domain Status、Security Capability、Storage behaviorを推論してはならない。

### 29.21.1.6 Wireframe Boundary

このAnnotated WireframeはCommon Pattern / Component LibraryのVisual UX Working Baselineであり、個別Featureの承認一覧でも実装完了一覧でもない。Patternを採用したことだけを理由に、未承認・未実装・未検証のCapabilityをUIへ追加しない。
## 29.22 Common Pattern Acceptance Criteria

- 同じ意味の操作が画面ごとに別の名称・位置にならない。
- Statusの表示パターンと状態語彙が統一される。
- Backで対象と文脈を可能な範囲で維持する。
- Refreshが副作用を持たない。
- Loading / Empty / Errorの意味が明確に区別される。
- Material / High-Risk operationは共通Confirm patternへ接続される。
- Critical notificationが通常通知パターンに埋没しない。
- 詳細情報の開示でPrivacy境界を越えない。
- Common UI componentにSecurity / Domain判断を実装しない。
- Patternの存在だけを根拠に未承認のFeature / Contract / Status / Security capabilityを表示・生成しない。
- Empty / Unavailable / Errorの意味が明確に区別される。
- 共通化によってクリック数削減を優先しすぎ、安全確認を省略しない。

---

## Extraction Note

The material was originally extracted from the former Section 28/29 scope of the parent UI working draft. The working-draft split changed document ownership, not product authority. Any future semantic change must be reviewed as a design change; the presence of a pattern in this file must not be treated as a newly approved runtime requirement.

End of Document
