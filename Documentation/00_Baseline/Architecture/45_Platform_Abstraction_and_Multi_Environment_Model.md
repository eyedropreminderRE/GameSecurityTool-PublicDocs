# GameSecurityTool

# Platform Abstraction and Multi Environment Model

## Environment Compatibility / Platform Layer / Runtime Adaptation Specification

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-045 |
| Version | 2.0 (Cross-Reference Links Synchronized) |
| Status | Formal Baseline Specification |
| Category | Platform Architecture |
| Authority Level | Environment Compatibility Governance |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書はGameSecurityTool（以下GST）における、

- Platform差異吸収
- Runtime互換性
- Environment Adaptation
- 将来移行性

を定義する。

---

目的：

```
Maintain Stable Behavior
Reduce Platform Dependency
Enable Future Migration
```

---

# 1. Platform Abstraction Philosophy

## 1.1 Core Principle

GSTは、

特定環境へ過度に依存しない。

---

基本原則：

```
Separate Platform Logic
Preserve Core Logic
Adapt Safely
```

---

# 2. Layered Architecture Model

Platform依存処理は分離する。

構造：

```
Application Layer
↓
Service Layer
↓
Platform Abstraction Layer
↓
Operating System
```

---

# 3. Core Logic Isolation

Core機能：

可能な限りPlatform非依存とする。

対象：

```
Business Logic
Validation Logic
Data Processing
Security Decision
```

---

# 4. Platform Specific Component

分離対象：

```
File System Access
Registry Access
Process Control
Network Interface
System API
```

---

# 5. Operating System Compatibility

対応対象：

```
Windows 10
Windows 11
Future Windows Version
```

---

確認：

```
API Availability
Permission Behavior
Security Policy
```

---

# 6. Runtime Compatibility

対象：

```
.NET Runtime
Framework Version
Native Dependency
```

---

原則：

```
Supported Runtime Must Be Explicit
```

---

# 7. Hardware Abstraction

考慮対象：

```
CPU Architecture
Memory Size
Storage Type (NVMe SSD / SATA SSD / Spinning HDD / External Drive)
GPU Environment
Power State (AC / Battery / Sleep / Resume)
```

---

禁止：

```
Assume Specific Hardware
Assume Instant Storage Response (Disregard Spin-up Delay)
Ignore Power State Transitions
```

---

## 7.1 【確定】ストレージ特性・スピンダウン遅延耐性の抽象化 (`IStorageResilienceProvider`)

外付けハードディスクや省電力モードにより回転停止（スピンダウン）した磁気ディスク（HDD）へのアクセス遅延を安全に吸収する：

* **スピンアップ遅延耐性 (Spin-up Tolerance):** モーター回転待ちに伴う Win32 エラー `ERROR_NOT_READY (21)` や `ERROR_BUSY (170)` を一時的遅延として捕捉し、最大 15 秒間の指数バックオフで安全に待機する。
* **事前スピンアップ・プローブ (Pre-Wakeup Probe):** 大規模セーブデータやバックアップの読み込み前に、軽量なボリューム問い合わせ（`DriveInfo.AvailableFreeSpace`）を発行して事前にスピンドルモーターを安全に覚醒させる。
* **ホットアンプラグ安全保護 (Hot-Unplug Protection):** 外付けドライブの突然の抜去（`DBT_DEVICEREMOVECOMPLETE`）を検知した場合、進行中ジョブを即座にクラッシュさせず中断（Suspend）状態へ移行させ、ドライブ再接続時に安全に再開を試行する。

---

## 7.2 【確定】OS 電源状態変化（Suspend / Resume）の抽象化 (`IPowerStateService`)

PC のスリープ（スタンバイ）・復帰に伴うライフサイクル不整合を防止する：

* **電源遷移イベントの抽象化:** OS のサスペンド（`PowerModes.Suspend`）およびレジューム（`PowerModes.Resume`）をプラットフォーム非依存イベントとして Application 層へ通知。
* **トランザクション中のスリープ一時抑止:** セーブデータ復元や暗号化隔離などの不可分なトランザクション実行中、Win32 `SetThreadExecutionState` (`ES_SYSTEM_REQUIRED | ES_CONTINUOUS`) を発行して OS の勝手なスリープ突入を抑止し、処理完了時に確実に解放する。
* **復帰時自律再同期トリガー:** スリープ復帰時に WMI プロセス監視の再接続およびゴーストプロセス狩りを自動キックし、OS 状態との整合性を保つ。

---

# 8. Launcher Integration Model

対象：

```
Steam
Epic Games Launcher
GOG
Custom Launcher
```

---

設計：

```
Common Interface
↓
Launcher Adapter
↓
Platform Implementation
```

---

# 9. Game Detection Abstraction

Game検出：

分離する。

```
Detection Engine
↓
Game Provider Module
↓
Platform Data
```

---

# 10. Path Management

パス処理：

固定値禁止。

---

禁止：

```
Hardcoded Installation Path
```

---

必須：

```
Dynamic Discovery
Validation
Fallback Handling
```

---

# 11. Permission Environment Handling

環境差：

考慮する。

```
Administrator
Standard User
Restricted Environment
```

---

# 12. Security Software Compatibility

対象：

```
Antivirus
Firewall
Endpoint Protection
```

---

対応：

```
Detect Interference
Explain Cause
Provide Guidance
```

---

# 13. Configuration Abstraction

環境設定：

分離管理する。

```
User Configuration
System Configuration
Runtime Configuration
```

---

# 14. Multi Machine Support

PC移行：

考慮する。

対象：

```
User Profile
Configuration
Backup Data
```

---

# 15. Environment Detection

起動時：

確認可能な情報：

```
OS Version
Runtime Version
Permission State
Hardware Information
```

---

# 16. Compatibility Matrix

管理：

```
Supported
Limited Support
Deprecated
Unsupported
```

---

# 17. Unsupported Environment Handling

非対応時：

```
Detect
Explain
Prevent Unsafe Operation
```

---

禁止：

```
Unknown Environment Forced Execution
```

---

# 18. Future Platform Migration

将来変更：

考慮：

```
New Runtime
New OS
New Distribution Model
```

---

# 19. Dependency Isolation

外部依存：

管理：

```
Version
Purpose
Compatibility
```

---

# 20. Update Compatibility

更新時：

確認：

```
Existing Configuration
Existing Data
Existing Environment
```

---

# 21. Testing Requirement

環境別：

確認：

☐ Supported OS Test  
☐ Runtime Test  
☐ Permission Test  
☐ Migration Test  
☐ Recovery Test  

---

# 22. AI Development Rule

AIがPlatform処理を生成する場合：

確認：

```
Platform Dependency
Compatibility Impact
Security Impact
```

---

禁止：

```
Environment Specific Hack
```

---

# 23. Architecture Relationship (DEFECT-01 是正完了)

Architecture & Development Governance:
```
07_Architecture_and_Development_Governance.md
```

Source Code Structure & Modules:
```
28_Source_Code_Structure_and_Module_Architecture_Model.md
```

External Integration & Compatibility:
```
30_External_Integration_and_Platform_Compatibility_Model.md
```

Save Format Compatibility:
```
44_Game_Save_Format_Reverse_Compatibility_Model.md
```

Zero Trust Security:
```
42_Zero_Trust_Security_Model.md
```

---

# 24. Final Platform Statement

GSTにおけるPlatform対応とは、

すべての環境を無条件に対応することではない。

---

環境差を正しく認識し、

安全な抽象化によって、

安定した利用体験を提供することである。

---

Final Principle:

```
Abstract Differences
Protect Core
Adapt Safely
Evolve Continuously
```

---

End of Document

---

