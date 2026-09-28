# GameSecurityTool Documentation

GameSecurityTool（GST）の公開用ドキュメント入口です。

このインデックスは、設計思想・ベースライン・アーキテクチャ・機能仕様・コアモジュール仕様へ、公開読者が到達するための導線を整理しています。

## 公開範囲

この公開用コピーでは、内部開発運用資料、公開準備の作業ログ、作業コンテキスト、レガシーなビルド成果物を通常の仕様ナビゲーションから除外します。

仕様書の存在は、当該機能の実装済み・検証済み・リリース済みを意味しません。PublicDocs は WIP のため、GST本体の現行文書・実装・検証状態を完全にミラーするものではありません。

## 主要ドキュメント

### 1. Baseline

[00_Baseline/](00_Baseline/) は GST のFormal Baseline文書群です。文書ごとの権威レベルと現行性は各Document Controlおよびauthority-chain資料で確認します。

- [Baseline Directory Overview](00_Baseline/README.md)
- [Core Philosophy and Principles](00_Baseline/01_Core_Philosophy_and_Principles.md)

Source側Baselineの物理Markdown数は58です。ただし、すべてが既定の公開ナビゲーション対象ではありません。Governance / Operations などは公開境界を個別にレビューし、内部寄り資料はリンクから除外します。

### 2. Architecture

[01_Architecture/](01_Architecture/) は Clean 5-Layer、技術スタック、永続化、OS統合、UI等のアーキテクチャ仕様を収録します。

Architecture は **22仕様 + README = 23 Markdown** です。UI Design Working Drafts を含む現行Architecture領域全体の物理ファイル数です。

- [Architecture Overview](01_Architecture/01-00_Overview_and_Principles.md)
- [Clean Hexagonal Architecture](01_Architecture/01-01_Clean_Hexagonal_Architecture.md)
- [Technology Stack and Runtimes](01_Architecture/01-02_Technology_Stack_and_Runtimes.md)
- [Security Architecture and Threat Model](01_Architecture/01-03_Security_Architecture_and_Threat_Model.md)
- [Persistence and Database Architecture](01_Architecture/01-04_Persistence_and_Database_Architecture.md)
- [OS Integration and Hardware Boundary](01_Architecture/01-05_OS_Integration_and_Hardware_Boundary.md)
- [Presentation and UI Architecture](01_Architecture/01-06_Presentation_and_UI_Architecture.md)
- [Cross-Cutting Concerns and Lifecycle](01_Architecture/01-07_Cross_Cutting_Concerns_and_Lifecycle.md)
- [Feature Integration Update](01_Architecture/Feature_Integration_Update_v1.0.md)

### 3. Features

[02_Features/](02_Features/) はユーザー向け機能の詳細仕様です。

Features は **31仕様 + README = 32 Markdown** です。詳細な機能別導線は [02_Features/README.md](02_Features/README.md) を参照してください。

#### Feature families

- Save Data Secure Auto-Backup: 11 files (`02-00`〜`02-10`)
- Web Link Protection: 7 files (`01-00`〜`01-06`)
- Trust Enhancement: 7 files (`04-00`〜`04-06`)
- Privacy: 2 files
- Additional Security features: LNK detection, MOD provenance, In-Game Overlay HUD, and advanced protection

### 4. Core Modules

[modules/](modules/) contains the six core execution-module specifications.

There are **7 Markdown files** including the module overview.

- [Modules Overview](modules/03-00_Modules_Overview.md)
- [SYS — System Safety and Lifecycle](modules/03-01_SYS_System_Safety_and_Lifecycle.md)
- [SCAN — File Scanner and Security Engine](modules/03-02_SCAN_File_Scanner_and_Security_Engine.md)
- [FW — Zero Trust Firewall](modules/03-03_FW_Zero_Trust_Firewall.md)
- [QUAR — Quarantine and Restore](modules/03-04_QUAR_Quarantine_and_Restore.md)
- [ALLOW — Allow List and Trust](modules/03-05_ALLOW_Allow_List_and_Trust.md)
- [CFG — Configuration and Gamer UX](modules/03-06_CFG_Configuration_and_Gamer_UX.md)

## Public-only entry points

- [Error Handling and Event Codes](ERROR_HANDLING_AND_EVENT_CODES.md)
- [Security Boundary and Test Vectors](SECURITY_BOUNDARY_AND_TEST_VECTORS.md)
- [Project Structure and Namespace Specification](PROJECT_STRUCTURE_AND_NAMESPACE_SPEC.md)

These documents should be read as design and implementation-planning references. They do not by themselves establish that the complete product is implemented or released. The former roadmap file is intentionally not a default public entry point while the publication scope is being reconstructed.

## Status and authority notes

GST本体の製品ガバナンス情報はSource of Truth側で管理されています。PublicDocs は WIP であり、現時点ではGST本体の全公開候補文書との完全同期を完了していません。公開文書は、公開時点で確認したGST本体 `main` の内容を基準とするスナップショットとして扱います。

`Approved`, `確定`, and `正式登録` indicate design/document status unless separate implementation, verification, or release evidence establishes a later lifecycle state.

## Historical material

Archive is not part of the default current-specification navigation. Curated historical documents may be published later with explicit **Historical / Non-authoritative** labelling.

`Archive/Source_Legacy_Backup/` and build artifacts such as DLL, PDB, EXE, cache, `bin/`, and `obj/` are excluded from the public documentation surface.

## License

GST-authored documentation in this public documentation surface is licensed under **Creative Commons Attribution 4.0 International (CC BY 4.0)**. The final PublicDocs repository root will provide the official legal code. Third-party material is not automatically relicensed and remains subject to its own rights and license terms.

## Internal material excluded from this index

AI coding rules, AI development profiles, internal work context, and publication-work/audit files are intentionally omitted from public navigation.

End of Document
