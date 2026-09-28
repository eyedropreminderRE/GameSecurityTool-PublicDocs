# GameSecurityTool

# Game Environment Protection Model

## Game Lifecycle / Safe Onboarding / Archive Restoration Definition

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-003 |
| Version | 4.0 (Safe Game Onboarding, Archive Management & Launch Modes Master Edition) |
| Status | Formal Baseline Specification |
| Category | Game Environment Protection |
| Authority Level | Core Functional Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書はGameSecurityTool（以下GST）における、

- ゲーム環境認識とゲームプロファイル管理
- 起動モード体系（通常起動 vs 厳格モード起動）の制御
- ポータブルゲーム（ZIP/7z/RAR）の安全展開 ＆ 自動オンボーディング
- ゲーム本体アーカイブの復元・保管管理
- ライフサイクル管理とポスト監査
- 保存領域保護と Firewall 管理
- クリーンアップ方針

を定義する。

---

基本原則：
```
Understand the Environment
Protect the Environment
Never Own the Environment
```

---

# 1. Game Environment Protection Philosophy

## 1.1 Protection Definition
GST におけるゲーム環境保護とは、以下を意味する。
```
Detection * Organization * Monitoring * Evidence Preservation * Recovery Assistance
```
保護とは、ユーザーの同意なき強制的な改変、勝手な自動削除、または環境の支配を意味しない。

---

# 2. 2段階 起動モード体系 (Game Launch Modes)

GST は、ゲームの素性やユーザーの目的に応じて 2 種類の起動モードを提供する。

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        GST 起動モード体系                              │
├────────────────────────────────────────────────────────────────────────┤
│ 1. ▶️ 通常起動 (Standard Launch)                                       │
│    - Universal Rule Scope (Global ＋ GameProfile 設定) に完全準拠      │
│    - Steam 等の公式・信頼済みゲーム向け                                │
├────────────────────────────────────────────────────────────────────────┤
│ 2. 🛡️ 厳格モード起動 (Strict Safe Launch)                              │
│    - In/Out 通信強制ブロック ＆ WER クラッシュ送信完全抑止             │
│    - 素性の分からないインディーゲーム、フリーゲーム、エミュレータ向け   │
└────────────────────────────────────────────────────────────────────────┘
```

## 2.1 通常起動 (Standard Launch)
* **動作仕様:**  
  該当ゲームの GameProfile に登録されたポリシー（Effective Policy: 通信制御、Web保護モード等）に準拠して起動する。
* **ライフサイクル:**  
  起動検知 ➔ セーブデータ事前スナップショット採取 ➔ 設定された Firewall/WER 適用 ➔ ゲーム実行 ➔ 終了時ポスト監査。

## 2.2 厳格モード起動 (Strict Safe Launch)
* **目的:**  
  未知の実行ファイルによる裏側での不正な外部通信や個人データ収集を未然に遮断する。
* **動作仕様 (セッション限定オーバーライド):**  
  1. **In/Out 完全遮断:** 対象プロセスの `.exe` に対し、インバウンドおよびアウトバウンドのすべての通信を強制遮断する一時ルールを適用。
  2. **WER テレメトリ強制抑止:** Windows Error Reporting によるクラッシュダンプ送信を完全ミュート。
  3. **不審書き込み監視:** セーブデータフォルダ以外への不審バイナリドロップを通常より高感度で監視。

---

# 3. ポータブルゲーム安全展開 ＆ 自動オンボーディング (Safe Game Onboarding)

ZIP / 7z / RAR 形式で配布されるインストーラーのないフリーゲームや同人ゲームを、安全かつ快適に管理下へ導入する仕組み。

```text
[ 配布アーカイブ (ZIP / 7z / RAR) のドラッグ＆ドロップ ]
                           │
                           ▼
1. 【Zip Slip / Zip Bomb 防御展開】 (SharpCompress ＋ PathValidationBarrier)
   - 相対パス (../) による管理フォルダ外への脱出を構造的に遮断
   - ストリーミング容量監視により異常な肥大化アーカイブを展開中断
                           │
                           ▼
2. 【事前レントゲン診断】 (FastFileSystemScanner)
   - 展開された全ファイルを即座にスキャン
   - 未署名バイナリ、外部通信 API (ws2_32.dll) の有無を特定
                           │
                           ▼
3. 【自動オンボーディング】
   - メイン実行ファイル (.exe) を自動検出し、GameProfile へ登録
   - 初期バニラ・ベースラインを自動記録
                           │
                           ▼
4. 【厳格モードでの即時プレイ ＆ 後腐れゼロ消去の担保】
   - そのまま [🛡️ 厳格モードで起動] で安全にプレイ可能
   - 不要時は [🗑️ 完全消去] で本体・セーブ・設定ゴミを一括削除
```

---

# 4. ゲーム本体アーカイブの復元・保管管理 (Game Environment Archive Management)

セーブデータバックアップ（進行度の保護）とは明確に区別し、**「ゲームフォルダ全体の構造・バージョンを外部保管し、必要な時に直接復元する管理機能」** を提供する。

## 4.1 目的とユースケース
* **意図しないゲーム本体のアップデート対策:**  
  公式アップデートによって MOD が動作不能になった際、いつでも「以前の正常に動作していたバージョン」へ丸ごと復元可能にする。
* **ローカライズ済み環境の保管:**  
  パッチ適用後のクリーンな基準状態をアーカイブとして保管する。
* **オフライン・再ダウンロード不要の復旧:**  
  大容量ゲームが破損した際、回線帯域を使わずにローカルストレージから高速に復元する。

## 4.2 役割分担と復元仕様
* **アーカイブの作成:**  
  GST 自体はゲーム本体（数十GB〜）の圧縮機能を持たず、PC 負荷やメモリ消費を抑えるため、ユーザーが公式 7-Zip 等の外部ツールを用いて作成する。
* **保管場所の登録と復元 (GST 内蔵機能):**  
  GST にアーカイブ保管フォルダを登録することで、配置されたアーカイブ（`.7z`, `.zip`, `.rar`）を自動検出し、**ボタン 1 つでゲームフォルダへ直接ストリーミング展開・復元** を実行する。

---

# 5. Game Profile Model

## 5.1 構造
```text
Game Profile
├ Identity (GameId, DisplayName, ExecutablePath)
├ Launch Policy (Standard / Strict Safe Launch)
├ Installation State (Installed / Missing / Archived)
├ Save Location Reference (セーブデータ保存先)
├ Archive Storage Directory (本体アーカイブ保管フォルダ)
├ Firewall Policy (Global準拠 / 送信のみ遮断 / 送受信両方遮断 / 全許可)
├ Web Link Policy (ブラウザ選択 / 既定直接 / 常に却下)
└ Game Snapshot References (外部アーカイブ管理メタデータ ＆ メモ)
```

---

# 6. Game Lifecycle Model

GST ではゲーム状態を以下で管理する。
```text
Registered ➔ Detected ➔ Installed ➔ Missing ➔ Archived ➔ Removed
```
* **Missing Handling:** ゲームがアンインストールされてもプロファイルやバックアップ履歴を勝手に削除せず、「過去のゲーム」として保護を継続する。

---

# 7. Save Location Protection Model

* **所有権:** セーブデータはユーザーの完全な資産である。
* **非破壊性:** セーブデータを勝手に自動編集・自動削除しない。
* **RescueSnapshot:** 復元実行時は現行データを一時退避し、万が一の障害時にも安全な自動ロールバックを保証する。

---

# 8. Firewall Ownership ＆ Foreign Rule Protection

* **GST 管理ルール:** `CreatedBy = GST` および `GameProfileId` を付与して管理。
* **Foreign Rule (外部ルール):** 他アプリや手動で作成されたルールは削除対象外とし、誤爆削除を防止する。

---

# 9. Cleanup Model (整理支援)

Cleanup は強制削除機能ではない。
* **対象:** 不要になった GST 内部メタデータ、孤立した GST 作成 Firewall ルール。
* **非対象 (所有権保護):** ゲーム本体ファイル、セーブデータ、ユーザー作成 MOD。

---

# 10. Shortcut (.LNK) Protection Model

ゲーム起動ショートカット（`.lnk`）の改ざん（TargetPath, Arguments 等）を検知し、LotL 攻撃（スクリプト経由の不正起動）を未然に防止する。

---

# 11. Relationship With Other Models
* Save Backup: `05_Save_Backup_and_Data_Ownership_Model.md`
* UI / Launch: `11_UI_UX_and_User_Interaction_Model.md`
* Migration: `09_PC_Migration_and_Recovery_Model.md`
* Audit: `04_Audit_and_Evidence_Model.md`

---

# 12. Final Game Protection Statement

GST はゲーム環境を理解し、安全に管理する。
しかし、ゲーム環境の所有者にはならない。

最終原則：
```
Track Without Owning
Protect Without Altering
Assist Without Controlling
```

---
End of Document
```

---
