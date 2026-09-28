# GameSecurityTool

# PC Migration and Recovery Model

## PC Replacement / Environment Transfer / Disaster Recovery Definition

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-009 |
| Version | 3.0 (Atomic Resealing Pipeline & Caller-Owns Key Ownership Edition) |
| Status | Formal Baseline Specification (Highest Migration Authority) |
| Category | Migration and Recovery Management |
| Authority Level | Core Recovery Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- PC 買替時の環境移行（PC Migration）
- Migration Package（`.gstmgr`）構造
- Backup アーカイブ（Standard ZIP）との連携
- 隔離（Quarantine）コンテナのアトミック再封緘（Re-sealing）
- 認証情報・秘密情報の安全な中間暗号化と Caller-Owns メモリ消去契約
- 移行失敗時および非互換バージョン時の自力復元フォールバック

を定義する。

---

GST における PC 移行の目的は、単なるファイルのコピーではなく、

```text
Preserve User Environment  *  Preserve Isolated Assets  *  Rebuild Secure Context
(ユーザー環境の保全、隔離資産の確実な引き継ぎ、安全なセキュリティコンテキストの再構築)
```

である。

---

# 1. Migration Philosophy

## 1.1 Core Principle
PC 移行では以下を優先する。

```text
User Data Preservation
*
Security Continuity
*
Reversible Quarantine Preservation
*
Minimal User Effort
*
Zero Data Corruption on Interruption
```

---

# 1.2 Migration Boundary (移行境界)

### Migration 対象:
```text
GST Configuration & Profile Presets
Game Profile & Target Paths
Protection Policies (Rule Scope)
Backup References & Manifest Records
Quarantine Metadata & Re-encrypted Container Keys (中間暗号化済み隔離鍵)
Audit History & Hash Chain Records
User Approved Metadata
```

---

### Migration 対象外:
```text
Game Executables & Installed Binaries
Machine-Specific Machine GUID / Hardware ID
Raw Windows DPAPI Blobs (Without Re-encryption)
Windows User Identity (SID)
```

理由：  
ゲーム本体や OS 認証環境は新 PC 側の環境に依存するため。ただし、暗号化隔離されたファイルは正当なユーザー資産（誤検知救済用）として、新環境へ復号鍵を再封緘移送しなければならない。

---

# 2. Migration Package Model

## 2.1 Definition
Migration Package（`.gstmgr`）とは、GST 環境を別 PC へ安全に移送するための暗号化パッケージである。

構成：
```text
Migration Package (.gstmgr)
├ PackageFormatVersion (Major.Minor ヘッダー)
├ GST Configuration & Profile Presets
├ Game Profile Data & Allow Lists
├ Firewall Rule Configurations (Metadata Only)
├ Backup References & Manifests
├ Quarantine Entries & Re-encrypted AES Keys (中間暗号化済み隔離鍵)
├ Audit History (Hash Chain Log)
├ Re-encrypted Secrets (Argon2id + AES-256-GCM Encrypted)
└ Package Integrity Manifest (SHA256)
```

---

# 3. Migration Package and Backup Relationship

## 3.1 分離モデル (Separation Model)
```text
Migration Package (.gstmgr)
    │
    │ references (ポインタ参照)
    ▼
Backup Archive (Standard ZIP)
```

* **Migration Package:** GST 環境の再現（Environment Transfer）
* **Backup Archive:** ユーザーデータ資産の保全（User Data Preservation / No Lock-in）

## 3.2 独立性の保証
Migration Package 内に大容量セーブデータ実体を内包させず、肥大化を防ぐ。

---

# 4. PC Replacement Normal Flow

## 4.1 旧 PC エクスポート (Old PC Export)
```text
Open GST ➔ Select Create Migration Package
    ↓
Enter User Migration Passphrase
    ↓
Derive Migration Master Key (Argon2id: Iterations=3, Memory=64MB, Salt=16B)
    ↓
Decrypt Local DPAPI Secrets & Quarantine AES Keys
    ↓
Re-encrypt with Migration Master Key (AES-256-GCM)
    ↓
Package & Validate Integrity (with Format Version)
    ↓
Export .gstmgr Package ➔ finally で Migration Master Key を ZeroMemory 消去
```

---

## 4.2 新 PC インポート (New PC Import)
```text
Install GST ➔ Select Import Migration Package
    ↓
Validate Package Integrity & Format Version Compatibility
  ※ 非互換バージョンの場合は安全中断し、8.1 の手動復元へ誘導
    ↓
Enter User Migration Passphrase ➔ Derive Migration Master Key (Argon2id)
    ↓
Execute DPAPI Re-sealing Transaction:
  - 秘密情報を中間復号し、新 PC のローカル DPAPI で再封緘
  - 隔離コンテナヘッダーを新 DPAPI 鍵で【アトミックに再封緘】
    ↓
Import Database & Restore Game References
    ↓
finally で Migration Master Key を ZeroMemory 消去 ➔ 完了
```

---

# 5. Credential & Quarantine Re-sealing Transaction (C-4 ＆ H-2 是正)

## 5.1 DPAPI 境界問題
Windows DPAPI（`CurrentUser`）は端末の LSA / ユーザー資格情報に物理バインドされているため、旧 PC で暗号化された DPAPI Blob は別 PC では復号できない（`CryptographicException` 発生）。

---

## 5.2 安全な二段階再暗号化 ＆ アトミック再封緘パイプライン

```text
[ 旧 PC エクスポート時 ]
1. DB および隔離コンテナから DPAPI 保護対象を一時復号 (隔離鍵, Auditアンカー, 設定)。
2. パスフレーズから Argon2id で 256-bit Migration Master Key を導出。
3. 全秘密情報を AES-256-GCM で暗号化し、.gstmgr 内へ格納。
4. 完了後、呼び出し元 (MigrationCoordinator) が Migration Master Key を ZeroMemory 消去。

[ 新 PC インポート時 (アトミック再封緘トランザクション - C-4 規約) ]
1. パスフレーズ検証後、Migration Master Key で秘密情報を中間復号。
2. 新 PC のローカル DPAPI (ProtectedData.Protect) で各秘密情報を再暗号化。
3. 【アトミック再封緘】隔離コンテナ (.qrt) を直接上書きせず、一時ファイル (.reseal.tmp) へ
   新ヘッダー＋暗号化ペイロードを出力完了後、File.Move(overwrite: true) でアトミック置換。
4. DB レコード (QuarantineEntryRecord) を新環境向けに確定コミット。
5. 【H-2 規約】全サブシステム再封緘完了後、MigrationCoordinator が Migration Master Key を
   finally ブロックで確実に ZeroMemory 消去。
```

---

# 6. Migration Authentication & Ownership

* **パスフレーズの所有権:** 移行パスフレーズはユーザーが完全管理する。GST はパスワード復旧やバックドアを提供しない。
* **保存禁止事項:** 平文パスワード、生暗号鍵、端末固定 DPAPI Blob（中間暗号化なし）の永続化を厳禁とする。

---

# 7. Migration Success Scenario

正常時：
```text
Migration Package Import
    ↓
GST Environment Restored & Re-sealed
    ↓
Quarantine Containers Restored & Re-sealed (いつでも安全復元可能)
    ↓
Backup References Connected (既存 Standard ZIP と自動再リンク)
    ↓
Protection Active
```

---

# 8. Migration Failure Recovery (非互換パッケージのフォールバック)

## 8.1 移行パッケージ紛失・破損・非互換時の自力復旧経路
Migration Package が破損・紛失した場合、または **Format Version が非互換（新旧メジャーバージョン相違: `MigrationPackageVersionMismatch`）** の場合でも、ユーザーデータの復旧経路を完全に保証する。

```text
Migration Failed, Lost, or Incompatible Version Detected
    ↓
GST Aborts Import Safely (既存環境の破壊ゼロ)
    ↓
Locate Standard Backup ZIP (外側コンテナは7-Zip等で展開可能)
    ↓
User Enters Backup Password (Argon2id + AES-256-GCM-CHUNKED; encrypted payload requires a GST-compatible independent recovery implementation)
    ↓
Register Backup Manually into GST ➔ Rebuild Game Profile ➔ 復旧完了
```

---

# 9. Manual Recovery Model

手動復元時、以下の GST 固有内部ファイルはゲームディレクトリへ復元してはならない。

* **除外対象:** GST Cache, GST Internal Metadata (`.gstcontainer`), Old Machine Specific Data, Old DPAPI Blobs
* **復元対象:** Game Save Data, User Configuration (.ini, .json, .cfg), User Created Mods & Assets

---

# 10. Firewall Rule Rebuild

旧 PC の Firewall ルールは直接コピーせず、新 PC 側の実行バイナリ存在確認を経て、Out-of-Process UAC Worker 経由で再構築する。

---

# 11. Migration Validation Checklist

インポート完了前に確認：

☐ Package Integrity & Format Version Compatibility Check  
☐ Passphrase Decryption Success (Argon2id)  
☐ **すべての隔離コンテナが一時ファイル経由でアトミックに再封緘されていること**  
☐ **`migrationMasterKey`（`byte[]`）が全処理完了後に `ZeroMemory` されていること**  
☐ Database Schema Version Compatibility  
☐ Game Profile References Restored  
☐ Backup Storage Accessibility Verified  

---

# 12. Final Migration Statement

GST の Migration は、単純なファイルコピーではない。

---

目的：

```text
Transfer Environment Safely
Preserve User Asset Ownership
Preserve Quarantine Reversibility Atomically
Rebuild Security Context
Never Leak Secrets in Memory
```

---

Final Principle:

```text
Re-seal Every Secret Atomically
Zeroize Migration Master Key in Memory
Preserve User Asset Independence Always
```

---

End of Document
```
