# 01-15: UI Settings Design

**Document ID:** GST-ARCH-UI-SETTINGS-001  
**Version:** 0.4 (Settings Annotated Wireframe / Visual UX Review)  
**Status:** Design Working Draft — Not an independent feature authority  
**Authority Relationship:** Subordinate to GST-BASELINE-011, GST-BASELINE-047, applicable approved module / feature specifications, and Change Control decisions  
**Parent Concept:** [01-08_UI_Information_Architecture_and_Interaction_Concept.md](01-08_UI_Information_Architecture_and_Interaction_Concept.md)  
**Historical Source Mapping:** Former Section 34 of the parent 01-08 UI working draft  
**Current Ownership:** This document is the maintained location for the Settings material after the working-draft split.
**Purpose:** Organize existing UI information-architecture and interaction design for this scope without expanding Runtime Features, Public Contracts, Domain Status, Security capabilities, or Implementation Readiness.

## Document Boundary

This document preserves UI design material that was formerly organized as Section 34 in the parent 01-08 UI working draft. After the working-draft split, 01-08 retains global IA / interaction principles while this document owns the Settings material.

The preserved section numbering is historical traceability only. It does not imply that those sections still exist in 01-08.

This document does not authorize Runtime implementation by itself. Existing formal baselines, approved module / feature specifications, Public Contracts, Security boundaries, and Change Control decisions remain authoritative where scope overlaps.

## Related Authoritative Documents

- [03-06_CFG_Configuration_and_Gamer_UX.md](../modules/03-06_CFG_Configuration_and_Gamer_UX.md) — Configuration authority
- [09_PC_Migration_and_Recovery_Model.md](../00_Baseline/GameProtection/09_PC_Migration_and_Recovery_Model.md) — Migration / environment-transfer authority
- [03-01_SYS_System_Safety_and_Lifecycle.md](../modules/03-01_SYS_System_Safety_and_Lifecycle.md) — Uninstall / system-reversion authority
- [32_Localization_and_Internationalization_Model.md](../00_Baseline/Governance/32_Localization_and_Internationalization_Model.md) — Localization authority
- [47_User_Experience_and_Interface_Design_Governance_Model.md](../00_Baseline/Governance/47_User_Experience_and_Interface_Design_Governance_Model.md) — Formal UX governance

---

# 34. Settings UX Design

Settingsは「GST内部の機能一覧」ではなく、**ユーザーが自分の環境に対して何を選択し、どの範囲に適用し、何が変わり、どう戻せるかを管理する画面**とする。

最初にStandard Profileを基準として見せ、個別設定との差分を理解できる構造を採用する。

基本原則:

    現在の設定 → Scope → Purpose → Impact → Review → Apply → Verify → Recover

設定画面だけを見て、内部のSQLite、Repository、Worker、Win32 API等を理解する必要がない構成とする。

## 34.1 Settings Mental Model

設定の意味を次の4つへ分解する。

    Profile  ← 標準設定を基準にした全体の方針
    Scope    ← GST全体 / Game単位 / 機能単位など、既存仕様で定義された適用範囲
    Value    ← 現在選択されている値
    Impact   ← その値を変更すると何が変わるか

「カスタム設定」は異常状態ではない。

ただし、設定変更によって実際に一部保護が利用できなくなる場合は、Configuration StatusとProtection Statusを別軸で表示し、設定変更そのものを隠さない。

## 34.2 Settings Screen Primary Structure

Settingsは次の5領域を基本とする。

1. Profile / Standard Configuration
2. Everyday Preferences
3. Protection / Gamer Features
4. Privacy / AI / External Tools
5. Advanced / Maintenance / Migration

概念配置:

    ┌────────────────────────────────────────────────────────────────────┐
    │ ← 設定                              [🔎 設定を検索]                │
    ├────────────────────────────────────────────────────────────────────┤
    │ 🛡️ 現在の設定                                                     │
    │ 標準設定 / カスタム設定                                            │
    │ [🛡️ 標準の推奨設定に戻す]                                        │
    ├───────────────────────┬────────────────────────────────────────────┤
    │ 設定カテゴリ           │ 設定内容                                   │
    │                       │                                             │
    │ プロファイル           │ Standard Profile                          │
    │ 通知                   │ 通知 / ゲーム中DND                         │
    │ ゲーム保護             │ IME / Gamer Defense 等                     │
    │ セーブデータ           │ Backup Policy                             │
    │ プライバシー / AI      │ 送信前確認 / AI設定                        │
    │ 外部ツール / OS        │ 自動起動等                                 │
    │ 高度な設定             │ Firewall / KDF / Audit / Migration         │
    │ 保守                   │ Time Machine / Maintenance / Reset          │
    └───────────────────────┴────────────────────────────────────────────┘

これは構造検討用Wireframeであり、最終的な寸法、色、カード形状、アイコンはVisual Design段階で決定する。

## 34.3 Standard Profile as Reference

Settings上部には既存UXベースラインで定義された「🛡️ 標準の推奨設定に戻す」を明確な導線として配置する。

ここでいう「Standard」は、全機能を無条件に有効化するという意味ではなく、GSTの承認済みStandard Profileを基準として扱うことを意味する。

Standard / Customの表示は、設定の基準を理解するために利用する。

例:

    設定状態: カスタム設定
    Standard Profileとの差分: 2件

    [変更した設定を見る]

差分がない場合は、無理に「変更なし」を警告として表示しない。

## 34.4 Setting Card Pattern

各設定項目は可能な範囲で次の順に表示する。

    Name
    Current Value
    Scope
    Purpose
    Impact
    Standard Reference
    Details / Advanced

例:

    ゲーム中の通知
    現在: 自動抑制
    適用範囲: フルスクリーンゲーム中
    目的: プレイ中の中断を減らす
    影響: Critical通知は完全には抑制されません
    Standard: 自動抑制

設定値だけを表示し、「なぜ存在するのか」をユーザーに推測させる構成を避ける。

## 34.5 Scope Clarity

設定変更時には、ユーザーが「どこに効くのか」を常に理解できるようにする。

明示対象が既存仕様で定義されている場合:

    GST全体
    このゲーム
    この機能

などを表示する。

Global設定に従っているゲーム設定と、GameProfile個別設定を同じ見た目で表示しない。

未定義のスコープをクリック数削減などの理由でUIに新設しない。

## 34.6 Search and Modified Settings

設定数が多くなった場合は、Settings内検索を主要な補助導線とする。

検索結果では:

- 設定名
- 現在値
- 変更済みかどうか
- 設定のScope
- 関連するカテゴリ

を確認できる構成を採用候補とする。

検索は表示・移動の操作であり、検索結果を選択しただけで設定変更を実行しない。

「変更済みだけ表示」は候補機能として扱うが、これを理由に設定モデルや保存構造を先行変更しない。

## 34.7 Change Review

影響の大きい設定変更は、トグルを押した瞬間に静かに適用するのではなく、変更内容を理解できる構造とする。

基本:

    Current Value
        ↓
    New Value
        ↓
    Scope
        ↓
    Why / Impact
        ↓
    Confirm
        ↓
    Apply
        ↓
    Verify

特にFirewall、Protection Mode、Factory Reset、Migrationなどの高影響操作では、既存のReview → Confirm → Execute契約を使用する。

単純な表示設定や、既存仕様上で即時変更が許可されている低影響設定まで一律に確認ダイアログを増やさない。

### 34.7.1 Draft vs Applied vs Verified Configuration

The Settings surface must distinguish at least conceptually between:

- **Current / Verified:** the last configuration state confirmed to be effective.
- **Pending Change:** a user-selected value that has not yet been applied.
- **Applying:** an operation currently attempting to apply the approved change.
- **Applied / Verification Pending:** the apply operation completed but final verification is not yet complete.
- **Verified:** the resulting configuration has been checked according to the applicable contract.

Standard / Custom status shown as a configuration summary should describe the effective configuration state, not an unconfirmed UI draft.

A user changing a control must not cause the Home / Protection status surfaces to silently display the new value as effective before the relevant Apply / Verify boundary is satisfied.

This is a Presentation-state rule and does not introduce a new Domain status or configuration contract.

## 34.8 Configuration Time Machine

Configuration Time Machineは、設定の過去状態を「全部戻す」だけの画面にせず、差分を理解して選択できる構造とする。

推奨:

    Snapshot History
      ↓
    Snapshotを選択
      ↓
    Diff Preview
      ↓
    復元対象を選択
      ↓
    Review
      ↓
    Confirm
      ↓
    Partial Restore
      ↓
    Verify

既存仕様で定義されるゲーム単位の差分表示や、アンインストール済みゲームの扱いを、ユーザーに理解できる言葉へ変換する。

「スナップショットを復元」は、選択しただけで実行しない。

## 34.9 Standard Profile Reset

Standard Profileへの復帰は、設定画面から見つけやすくする。

ただし、実際に多数の設定を変更する操作であるため、単なるUIテーマ変更のような低影響操作とは扱いを分ける。

確認時には少なくとも:

- 現在の主要な変更点
- Standardへ戻した場合の主要な変更
- 対象範囲
- 復帰後の状態
- 必要な場合のRecovery導線

を確認できる構造とする。

Resetを実行したことを「すべての問題が解消した」と自動的に表現しない。

## 34.10 Notifications and Smart DND

通知設定では、通常通知とCritical通知を同じトグルで扱わない。

例:

    通常通知
    [✓] ゲーム中は抑制

    Critical Security / Data Protection
    常に確認が必要な状態は完全抑制しない

Critical状態の表示境界は既存UXベースラインに従う。

ゲーム中DNDを有効にしても、保護停止、重大な脅威、復旧失敗、重大な確認要求まで完全に消えると誤解させない。

## 34.11 Smart Game IME Lock

Smart Game IME Lockは「Windows入力設定を恒久変更する設定」ではなく、既存仕様に基づくゲームセッション向け機能として表示する。

設定画面では可能な範囲で:

- 現在のPreset
- Hold duration
- Japanese / English transition behavior
- Visual badge behavior
- 対象セッション範囲
- フェイルセーフ境界

を確認できるようにする。

既定の設定を変更する場合でも、ゲーム内部への侵入や入力注入を行う機能であるかのような説明をしない。

## 34.12 Gamer Defense / Anti-Wiper Settings

Anti-Wiper等の高影響機能は、単純な「ON / OFF」だけでは意味が不足する。

設定前後に:

    Protection Root
    Intended Purpose
    Current State
    Trigger / Action Policy
    Known Limitations
    Recovery / User Action

を確認できる構造を採用する。

既定OFFの機能について、OFFを異常状態と自動解釈しない。

個人資産rootを対象にできる機能についても、ユーザーが明示的に選択した対象だけが保護範囲になる既存契約を維持する。

## 34.13 Privacy / AI Settings

AI関連設定は、利便性だけを強調せず「何がGST外へ出る可能性があるか」を中心にする。

基本導線:

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

AIの回答は参考情報として扱い、AIの助言による設定変更や復旧操作を自動実行しない。

送信前のPreviewで、ユーザーが確認すべき情報を最初に表示し、生データや不要な内部情報を「安全そうだから」とそのまま表示する構成を避ける。

## 34.14 External Tools and OS Auto-start

外部ツール連携やOS自動起動は、ユーザーが「GSTが勝手にWindows設定を変える」と誤解しないよう、対象を明示する。

例:

    Windowsログオン時のGST自動起動
    現在: OFF
    Scope: このユーザー
    影響: ログオン時にGSTを開始します

既存仕様どおりOS自動起動は既定OFFとする。

外部圧縮ツール、外部セキュリティスキャナなどについては、GST自身の機能と外部アプリケーションの責務を区別する。

## 34.15 File Lock / Troubleshooting

設定変更や復旧操作が外部ソフトウェアによるファイルロックで失敗した場合、内部エラーコードだけを表示しない。

既存FileLockInspectorで利用できる情報を、可能な範囲で:

    対象ファイル
    ロック元として確認できたプロセス
    現在の影響
    推奨される次の確認

として表示する。

他社セキュリティソフト等が関係している場合でも、確認できた情報と推測を区別する。

## 34.16 Maintenance

セルフメンテナンスは、通常ユーザーが内部ファイルを手作業で削除することを要求しない。

Maintenance画面では:

    最終メンテナンス
    確認できた対象
    実施結果
    次に必要な操作

を中心に示す。

内部の一時ファイル名やSQLite/WALの専門情報は、必要な場合のみAdvanced Detailsへ開示する。

Maintenanceの実行とFactory Resetを同一のPrimary Action群へ並べない。

## 34.17 Migration

PC Migrationは、単なる設定コピーではなく「新しいPCへGST環境を移す操作」として扱う。

基本導線:

    Create / Import
      ↓
    Package / Destination Review
      ↓
    User Authentication / Passphrase
      ↓
    Execute
      ↓
    Verify
      ↓
    Result / Recovery Path

Migration PackageとBackup Archiveを同一物として表示しない。

非互換・破損などでMigrationできない場合は、利用可能なBackupからの復旧経路を明示する。

Migrationの秘密情報、パスフレーズ、内部鍵などを通常画面へ不用意に表示しない。

## 34.18 Factory Reset

Factory ResetはSettingsの日常設定操作ではなく、明確に最後の復旧手段として配置する。

通常Settings画面では目立つPrimary Actionにせず、Advanced / Maintenanceから到達する。

実行前に既存仕様どおりType-to-Confirmを使用し、少なくとも:

- 初期化対象
- 残るもの
- 影響
- 復旧の難しさ
- 実行後の初期状態

を確認できるようにする。

単純な「設定をリセットしますか？」だけでは完了させない。

## 34.19 Uninstall / System Reversion

完全アンインストールやOS設定復元は、通常Settingsと混同しない。

ユーザーには:

    GST本体の削除
    GSTが変更したOS設定の復元
    セーブデータ / Backup資産の扱い

を分離して説明する。

アンインストール後もユーザー資産として残るBackupについて、「GST削除時に一緒に消える」と誤解させない。

## 34.20 Settings Context and Back

Settings内でカテゴリや詳細へ移動した場合も、現在の検索条件・カテゴリ・スクロール位置などを安全な範囲で保持する。

例:

    設定
      → 通知
        → ゲーム中DND
          → 詳細

Backは前の画面・確認面へ戻る操作とする。

設定変更を完了した後のBackを、変更そのもののUndoと誤認させない。

## 34.21 Settings Wireframe — Working Baseline

    ┌────────────────────────────────────────────────────────────────────┐
    │ ← 設定                              [🔎 設定を検索]                │
    ├───────────────────────┬────────────────────────────────────────────┤
    │ カテゴリ               │ 🛡️ 現在の設定                            │
    │                       │ カスタム設定                               │
    │ ● プロファイル         │ Standardとの差分: 2件                     │
    │ ○ 通知                 │ [変更した設定を見る]                     │
    │ ○ ゲーム保護           │                                            │
    │ ○ セーブデータ         │ ゲーム中の通知                            │
    │ ○ プライバシー / AI    │ 現在: 自動抑制                            │
    │ ○ 外部ツール / OS      │ Scope: フルスクリーンゲーム中            │
    │ ○ 高度な設定           │ [詳細]                                    │
    │ ○ 保守 / 移行          │                                            │
    │                       │ Smart Game IME Lock                       │
    │                       │ Preset: FpsAction                         │
    │                       │ Scope: 登録ゲームセッション               │
    │                       │ [詳細]                                    │
    │                       │                                            │
    │                       │ [🛡️ 標準の推奨設定に戻す]                 │
    └───────────────────────┴────────────────────────────────────────────┘

## 34.21.1 Annotated Wireframe — Settings / Visual UX Working Baseline

このAnnotated Wireframeは、Settingsの既存意味論を視覚構造へ落とし込む作業基準である。設定値の表示、変更Draft、Apply、Verificationを別状態として扱い、未適用の変更が他画面の実効状態へ漏れないことを優先する。

```text
┌────────────────────────────────────────────────────────────────────────────┐
│ GST / Settings Context                                      設定            │
│ [←]                                                      [🔎 設定を検索]  │
├───────────────────────┬────────────────────────────────────────────────────┤
│ カテゴリ               │ Configuration Summary                             │
│                       │ 実効設定: カスタム設定                             │
│ ● プロファイル         │ Standardとの差分: 2件                             │
│ ○ 通知                 │ 確認状態: Verified / Pending Review               │
│ ○ ゲーム保護           │                                                    │
│ ○ セーブデータ         │ ┌──────────────────────────────────────────────┐ │
│ ○ プライバシー / AI    │ │ ゲーム中の通知                                │ │
│ ○ 外部ツール / OS      │ │ 現在の値: 自動抑制                            │ │
│ ○ 高度な設定           │ │ Scope: フルスクリーンゲーム中                │ │
│ ○ 保守 / 移行          │ │ Standard: 自動抑制                            │ │
│                       │ │ 目的: プレイ中の中断を減らす                  │ │
│                       │ │ 影響: Critical通知は完全抑制されない          │ │
│                       │ │ [詳細]                                         │ │
│                       │ └──────────────────────────────────────────────┘ │
│                       │                                                    │
│                       │ ┌──────────────────────────────────────────────┐ │
│                       │ │ Draft Changed                                 │ │
│                       │ │ 現在: ON → 提案: OFF                          │ │
│                       │ │ Scope: 登録ゲームセッション                   │ │
│                       │ │ [変更をレビュー]                              │ │
│                       │ └──────────────────────────────────────────────┘ │
│                       │                                                    │
│                       │ [🛡️ 標準の推奨設定に戻す]                        │
├───────────────────────┴────────────────────────────────────────────────────┤
│ Footer: 未適用の変更 1件 · Apply前 / Verify前は他画面の実効状態へ反映しない │
└────────────────────────────────────────────────────────────────────────────┘
```

### 34.21.1.1 Region Annotations

| ID | Region | Visual / UX rule | Safety meaning |
|---|---|---|---|
| T1 | Settings Context | Category / search / Backの現在Contextを保持 | どの設定を操作しているかを失わせない |
| T2 | Configuration Summary | Effective configurationとStandard基準を上位表示 | Draftを実効設定と誤認させない |
| T3 | Setting Card | Name → Current Value → Scope → Purpose → Impact → Standard | 値だけを見て変更影響を推測させない |
| T4 | Draft Change | CurrentとProposedを明確に対比 | UI操作だけで適用済みに見せない |
| T5 | Review Entry | 高影響変更はReview導線へ分離 | Toggle = Executeと誤認させない |
| T6 | Standard Reset | Standardへの復帰を明示的な別Action groupにする | 大量変更を日常設定と混同しない |
| T7 | Status Footer | Pending / Applying / Applied / Verifiedを明示 | 変更完了と検証完了を分離する |
| T8 | Search | Discovery / Navigationとして扱う | 検索結果選択で設定変更を開始しない |

### 34.21.1.2 Configuration State Mapping

Settingsの視覚状態は、少なくとも次の意味を独立して表現する。

```text
Current / Verified
      │
      └─ user edits control
             ↓
        Draft Changed
             ↓
        Pending Review
             ↓
        Confirm / Apply
             ↓
          Applying
             ↓
     Applied / Verification Pending
             ↓
          Verified
```

- `Standard` は基準となるProfile、`Customized` は実効設定がStandardと異なることを示す。
- `Draft Changed` は未適用の提案変更であり、Home / Protectionの実効状態を変えない。
- `Pending Review` はReview待ちであり、Apply済みではない。
- `Applying` はApply Operation中であり、二重実行を防ぐ。
- `Applied` は適用処理の結果であり、別Verificationが必要な場合は`Verified`とは表示しない。
- `Verified` は適用対象の契約に従って結果確認が完了した状態を意味する。
- `Unavailable` は現在のBuild / Environment / Contextで利用不能であり、単なるOFFやユーザー選択によるDisabledとは区別する。

### 34.21.1.3 State Variants

**Standard / Verified**

- 実効設定がStandard Profileと一致し、適用結果が必要な範囲で確認済みであることを表示する。
- Standardであること自体をSecurity Statusへ自動変換しない。

**Customized / Verified**

- Standardとの差分を示すが、Customizationだけを異常表示しない。
- Protection Statusは別の権威的なStatus surfaceに従う。

**Draft Changed / Pending Review**

- CurrentとProposedを並べて表示する。
- Pending中はHome / ProtectionへProposed valueを実効設定として反映しない。
- Reviewを離れた場合、旧Approvalをそのまま再利用しない。

**Applying / Applied / Verification Pending**

- Apply中は二重実行を防止し、現在Stepを示す。
- AppliedとVerifiedを別表示する。
- 検証失敗時は成功表示へ丸めず、Needs Attention等の既存Presentation stateへ変換する。

**Unavailable / Error**

- 現在変更できない理由または確認できた範囲を表示する。
- UI側で代替の設定Capabilityや新しいFallbackを推測しない。

### 34.21.1.4 High-Impact Settings Action Mapping

| UI Action | Meaning | Must not imply |
|---|---|---|
| 詳細を見る | Read-only inspection | 設定変更 |
| 検索 | Discovery / navigation | 設定適用 |
| 値を変更 | Draft proposal | 実効状態の変更完了 |
| 変更をレビュー | Review entry | Apply / Execute |
| 標準へ戻す | Standard reset entry | 全問題の解消 |
| Apply | Existing approved operation | Verified result |
| Cancel | Unexecuted pending changeの中止 | 適用済み変更のUndo |
| Back / Close | Navigation / safe exit | Applyの暗黙実行 |

高影響設定は、既存Feature / Contractに定義された`Review → Confirm → Execute / Apply → Verify`へ接続する。低影響設定については、既存仕様が即時変更を許す場合に限り不要な確認を追加しない。

### 34.21.1.5 Cross-Screen State Boundary

設定変更後の情報伝播は、次の境界を基本とする。

```text
Settings Draft
    ↓
Review / Confirm
    ↓
Apply / Execute
    ↓
Observe / Reconcile
    ↓
Verify effective configuration
    ↓
Home / Protection may display the verified effective state
```

- DraftまたはPending ReviewだけでHome / Protectionの実効Configuration Statusを変更しない。
- Apply結果が未検証の場合、他画面でも必要に応じて`Applied / Verification Pending`相当の既存表示 semanticsを維持する。
- SettingsのBack / Closeで未実行Draftを離れる場合、暗黙Applyや暗黙Discardを発生させない。
- 対象Game、Scope、Current value、Fingerprint / Integrity条件が変化した場合、旧Approvalを再利用しない。

### 34.21.1.6 Responsive / Accessibility Rules

- WideではCategory navigationとSetting detailを2領域に配置し、Compactでは1列へ再配置できる。
- 文字拡大時にもCategory / Current Value / Scope / Primary Review action / Backを優先して残す。
- Standard / Customized、Current / Proposed、Applied / VerifiedはColorだけでなくText + Icon + Groupingで区別する。
- 長い日本語設定名やImpact説明は自然に折り返し、トグルやReview actionを画面外へ押し出さない。
- Keyboard focusはCategory → Summary → Setting Card → Review → Reset / Maintenanceの順で理解しやすい構造を基本とする。
- Search結果から直接High-impact Executeへ移らず、選択後も通常のReview / Confirm boundaryを維持する。

### 34.21.1.7 Wireframe Boundary

このAnnotated WireframeはSettingsの既存情報・状態・操作境界を視覚化するものであり、新しいConfiguration capability、Public Contract、Domain Status、Storage behavior、Security capabilityを追加しない。

Wireframe上でActionableに見えるApply / Reset / Migration / Factory Reset等は、適用対象の正式なFeature / Module / Contractと実装・検証条件が成立している場合にのみActionableとする。UIの存在だけで機能の承認・実装完了とはみなさない。
## 34.22 Review Findings — 2026-09-29

The focused review against the Configuration Management baseline, CFG module specification, and UX governance identified one important presentation ambiguity and clarified it:

| Finding | Resolution |
|---|---|
| A changed control could be interpreted as already effective before Apply / Verify completed | Added an explicit Draft / Applied / Verified configuration boundary and tied Standard / Custom summary to effective state. |

No new configuration status contract or runtime behavior is created by this clarification.

## 34.22 Settings Acceptance Criteria

- Settings画面を開いた時点で「何を設定する画面か」が理解できる。
- Standard / Customの意味が明確で、Customだけを保護異常として扱わない。
- 各設定でCurrent Value / Scope / Purpose / Impactを確認できる。
- Standard Profileとの差分を追跡できる。
- Searchは設定変更を自動実行しない。
- 高影響設定変更は既存のReview → Confirm → Executeへ接続される。
- Configuration Time MachineはDiff Previewを経由して復元対象を理解できる。
- Standard Profile Resetは対象と影響を確認してから実行できる。
- 通常通知とCritical通知の抑制境界が明確に分かれる。
- Smart Game IME LockがWindowsの恒久入力設定変更やゲーム内部介入と誤認されない。
- Anti-Wiper等の高影響機能で対象範囲・作用・制約・ユーザー操作が理解できる。
- AI送信前にSanitize / Preview / Re-approveの境界を確認できる。
- OS自動起動などのWindows変更でScopeと影響が表示される。
- 外部ツールとGST自身の責務が区別される。
- File Lock等のトラブル時に確認済み情報と推測が区別される。
- MaintenanceとFactory Resetが同じ日常操作として扱われない。
- Migration PackageとBackup Archiveが混同されない。
- Factory Resetが誤クリックで即時実行されず、Type-to-Confirmへ接続される。
- Uninstall時にBackup資産とGST本体の扱いが区別される。
- Back / Search / Category navigationでSettings Contextを可能な範囲で保持する。
- 画面サイズや文字サイズ変更で検索、主要設定、Back、Reset導線が消えない。
- Settings UX追加だけを理由に、新しいRuntime Feature、Public Contract、Domain status、設定スコープを導入しない。

---

## Extraction Note

Source text was extracted mechanically from the parent UI working draft. Any future semantic change must be reviewed as a design change; an extracted statement must not be treated as a newly approved runtime requirement merely because it now has a dedicated file.

End of Document
