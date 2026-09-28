# GameSecurityTool

# Core Philosophy and Principles

## Product Philosophy Definition

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-001 |
| Version | 2.1 (Anti-Wiper Emergency-Containment Boundary Clarification Edition) |
| Status | Formal Baseline Specification (Second Highest Authority) |
| Category | Core Design Philosophy |
| Authority Level | Second Highest Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- 基本理念（Protection without Destruction）
- 判断原則（Detect First / User Control First / Privacy First）
- ユーザー主権と説明責任（Explainable Security）
- データ所有権と脱ベンダーロックイン思想（No Vendor Lock-In）
- アトミック可逆性（Atomic Reversible Operations）およびフェイルセーフ原則

を定義する。

---

本書は GST 開発における全機能設計・実装の最上位判断基準となる。

Feature や Implementation が本書の哲学と矛盾する場合、本書を絶対優先とする。

---

# 1. Protection without Destruction

## 1.1 Core Principle
GST の最重要理念：

```text
Protection without Destruction
(ゲーム環境を守るが、ゲーム環境そのものを壊さない)
```

である。

GST の目的は、
```text
ゲーム環境を深く理解・監視・保護する
しかし
ゲーム環境そのものを決して壊さない・支配しない
```
ことである。

---

## 1.2 Protection Definition
GST における保護とは、以下を意味する。

```text
Observe (安全に観察する)
*
Understand (客観的に理解する)
*
Preserve (ユーザー資産を確実に保全する)
*
Assist Recovery (安全な復元・ロールバックを支援する)
```

保護とは、以下を決して意味しない（通常時の禁止事項）。

```text
Delete (独断での自動削除)
*
Modify (勝手なファイル改変)
*
Block (不透明な強制遮断)
*
Control (ユーザー環境の支配)
```

---

# 2. Detect First Principle

## 2.1 Detection Philosophy
GST では、すべてのセキュリティ判断は「検知」から開始する。

基本フロー：
```text
Detect ➔ Analyze ➔ Explain ➔ User Decision ➔ Optional Action
(検知 ➔ 分析 ➔ 証拠説明 ➔ ユーザー主権の意思決定 ➔ 同意に基づく実行)
```

---

## 2.2 No Immediate Action Rule (短絡的自動処罰の禁止)
検知結果（マルウェアの疑い等）だけを理由として、GST が自動的な環境改変を行ってはならない。

禁止事項:
- 検知即自動削除（Auto Delete）
- 検知即自動隔離（Auto Quarantine）
- 検知即自動修復（Auto Repair）

理由：  
未署名の MOD、ファンメイドツール、シェーダーキャッシュ、エミュレータ設定などの正当なカスタマイズを誤検知で破壊することを防ぐため。

> **Anti-Wiper の限定例外:** ユーザーが明示的に有効化した Anti-Wiper Defense の定義済み緊急シグナルに対して、GST は監視対象ゲームプロセスを一時停止し、設定またはユーザー選択に基づいて終了できる。この例外はプロセス制御に限定され、ユーザーデータの自動削除・隔離・修復を許可するものではない。

---

# 3. Risk Communication Principle

## 3.1 Non-Binary Judgment (二値判定の排除)
GST は検知結果を「Safe / Threat」の単純な 2 値（Binary）で表現しない。

提示すべき情報：
```text
Risk Level (危険度)
*
Confidence (確信度: 0.0〜1.0)
*
Evidence (客観的証拠: ハッシュ, 署名, PE 構造, 出所)
*
Explanation (専門用語を排した平易な理由と推奨アクション)
```

---

## 3.2 Confidence Representation (確信度の意味)
Confidence は GST の分析確信度を表す。  
ただし、

```text
Confidence  !=  Absolute Truth (確信度は絶対の真理ではない)
```

であり、どれほど高確信度であっても最終判断はユーザーが行う。

---

# 4. User Control First

## 4.1 Principle
GST では、ユーザーが最終判断者（Ultimate Decision Maker）である。  
GST は **「Assistant（支援者）」** であり、**「Authority（支配権力）」** ではない。

---

## 4.2 Explicit User Approval (明示的ユーザー承認)
環境へ影響を与える操作には、必ず事前の明示的ユーザー操作を必須とする。

| 操作種別 | ユーザー承認 | 備考 |
| :--- | :---: | :--- |
| **Restore (復元)** | **必須** | RescueSnapshot 退避と事前確認を義務付け |
| **Cleanup (整理)** | **必須** | 削除候補の一覧プレビュー表示が必須 |
| **Firewall Change (通信変更)** | **必須** | UAC 昇格理由と影響範囲を明示 |
| **Quarantine (隔離)** | **必須** | 64KB AEAD 隔離庫への退避承認 |
| **Migration Import (移行)** | **必須** | パスフレーズ照合と再封緘承認 |
| **Backup Delete (削除)** | **必須** | 差分参照依存性の事前検証 |

---

## 4.3 User Approval Definition
承認とは、ユーザーによる明示的な UI 操作（ダイアログ確認、ボタン押下）を意味する。  
過去の承認を理由として、事後的に永続的・サイレントな自動変更を行うことを禁止する。

---

# 5. Privacy First & Local First

## 5.1 Privacy Principle
GST は Privacy First / Local First を採用し、必要最小限の情報のみをローカルで処理する。

## 5.2 Forbidden Collection (収集・送信の絶対禁止)
1. **外部クラウドへの自動アップロード禁止:** ゲームセーブデータ、個人ファイルを外部へ自動送信しない。
2. **テレメトリ・行動監視の禁止:** アプリ起動状況、プレイ履歴、ユーザー行動分析データを一切収集しない。
3. **無関係な個人データの収集禁止:** クリップボード常時監視、ブラウザ履歴収集を厳禁とする。

---

# 6. Evidence Based Decision

## 6.1 Evidence Principle
GST のすべての判定および説明は、客観的証拠（Evidence）に基づく。

構成要素：
```text
Detection * Evidence * Context * Risk Evaluation
```

## 6.2 Evidence Examples
- ファイルハッシュ (SHA256), Authenticode デジタル署名
- PE エクスポート／インポート構造解析シグナル
- NTFS `Zone.Identifier` 出所メタデータ
- 判定時の `ConfigurationSnapshot`
- 改ざん検知監査ログ（Hash Chain 証跡）

---

# 7. Local Data Ownership Principle (No Vendor Lock-In)

## 7.1 Ownership Rule
GST が管理・生成するすべてのデータは、最終的にユーザーの完全な所有物でなければならない。

対象：
- セーブデータバックアップ
- 監査ログエクスポート
- PC 移行パッケージ (`.gstmgr`)
- 設定エクスポート

---

## 7.2 No Vendor Lock-In (脱ベンダーロックインの絶対保証)
GST 専用の独自バイナリ形式にユーザー資産を囲い込むことを厳格に禁止する。

```text
[禁止] GST Backup ──> GST 独自閉域形式 ──> GST なしでは復元不能 (Vendor Lock-in)
[採用] Standard ZIP 互換形式 + Argon2id パスワード暗号化 ──> 7-Zip 等で自力復元可能
```

GST が世の中から消滅しても、ユーザーは自身のパスワードと標準アーカイバのみでセーブデータを 100% 復元できる。

---

# 8. Fail Safe & Anti-Fail-Open Principle

## 8.1 Safety Priority (異常時の優先順位)
異常・障害発生時の意思決定優先順位：
1. **OS 保護**
2. **ゲーム環境保護**
3. **ユーザーデータ資産保護**
4. **検知継続**

---

## 8.2 Safe Failure & Zero Silent Fail-Open
異常発生時、GST は安全側（ブロック・保護継続・キュー停止）へ倒れる。

### サイレント Fail-Open の根絶規約:
プロセス監視エンジン（WMI）の停止や、特権管理者アンカー（Layer 2）の未同期など、**保護機能が低下・無効化されている状態を隠蔽し、ユーザーに「正常に保護中」と誤認させることを重大な理念違反として禁止する。** 劣化状態はダッシュボードへ直ちに可視化されなければならない。

---

# 9. Reversible Operation Principle (アトミック可逆性原則)

## 9.1 Principle
環境へのすべての変更操作は、元の状態へ安全かつ 100% 巻き戻し可能（Reversible）でなければならない。

必須要件：
- OS 設定変更直前のディスクジャーナル（`ChangeJournal` / `WerBeforeState.json`）書き出し
- セーブデータ復元直前の `RescueSnapshot` アトミック退避
- **一時ファイル経由のアトミック置換（`.rollback.tmp` / `.reseal.tmp` ➔ `File.Move`）による、復旧処理中のデータ破壊リスクゼロ保証**

不可逆な直接上書き・即時抹消を厳禁とする。

> **可逆性の適用範囲:** 本原則の「100% 巻き戻し可能」は、GST が行うファイル・設定等の環境変更に適用する。Anti-Wiper の明示的オプトインによるゲームプロセス終了は本原則の対象外であり、プロセス終了そのものを可逆操作とは扱わない。

---

# 10. Final Philosophy Statement

GST は、ユーザー環境を監視・保護するが、ユーザー環境を決して支配しない。

---

GST の最終判断原則：

```text
Detect Carefully
Explain Clearly
Preserve Safely
Never Lock-in User Assets
Mutate Files Atomically
Change Only With Explicit Consent
```

---

End of Document
```

