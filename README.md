# GameSecurityTool (GST)

> **🚧 WIP — Public Documentation Under Active Development**
>
> GameSecurityTool (GST) は、**Windows PC でゲームを楽しむユーザーの安全と、ゲーム関連データ・ユーザー資産の保護を目的として設計しているセキュリティツール**です。
>
> **現在は開発中です。** このリポジトリは GST の公開用ドキュメントを整理・公開するためのものであり、ここに仕様書が存在することは、機能が実装済み・Windows実機で検証済み・製品としてリリース済みであることを意味しません。

## GST とは？

GameSecurityTool (GST) は、ゲーム環境の安全性とユーザー資産の保護を中心に設計している Windows 向けセキュリティツールです。

ゲームのセーブデータや関連ファイルを守るだけでなく、**不審なファイル、リンク、MOD など、ゲームを取り巻く周辺環境まで含めて安全性を高めること**を目標としています。

## 🧭 GST Feature Catalog

GST本体では、現在 **14の独立した製品Feature** を定義しています。公開ドキュメントの「代表例」だけを見て機能全体を判断できないよう、このREADMEでは14 Featureを一覧化しています。

| Feature | 公開仕様の状態 |
|---|---|
| **F-01 Zero Trust Firewall** | 公開仕様未収録 |
| **F-02 Quarantine & Safe Restore** | 公開仕様未収録 |
| **F-03 Allow List & Trust Exceptions** | 公開仕様未収録 |
| **F-04 Web Link Protection** | 公開仕様あり |
| **F-05 On-Demand Clipboard URL Sanitizer** | 公開仕様あり |
| **F-06 In-Game Overlay HUD** | 公開仕様あり |
| **F-07 PC Migration & Recovery** | 公開仕様未収録 |
| **F-08 Smart Game IME Lock** | 公開仕様未収録 |
| **F-09 Anti-Wiper Defense & Auto-Circuit Breaker** | 公開仕様未収録 |
| **F-10 Privacy-safe Crash Diagnostics / Community Report** | 公開仕様あり |
| **F-11 BYOK Gemini AI Assistance** | 公開仕様あり |
| **F-12 LNK Hijack Detection** | 公開仕様あり |
| **F-13 MOD Security & Provenance** | 公開仕様あり |
| **F-14 Game Archive Restore / External Game Backup** | 公開仕様未収録 |

**「公開仕様未収録」は、機能が存在しないという意味ではありません。** GST本体側では仕様対象として管理されている一方、この公開スナップショットには独立した公開Feature Specを収録していないことを示します。

逆に、**公開仕様あり**も、実装済み・Windows実機検証済み・リリース済みを意味しません。公開資料のライフサイクル情報と、実装・検証・リリースの事実は分けて扱ってください。

### 現在公開している Feature 系ドキュメント

- セーブデータの安全な自動バックアップ・復元
- Web Link Protection
- LNK Hijack Detection
- MOD Security / Provenance
- In-Game Overlay HUD
- On-Demand Clipboard URL Sanitizer
- Privacy-safe Crash Diagnostics / Report
- Trust Enhancement と Gemini AI 連携
- Advanced User Protection に関する統合仕様

この一覧は、**GST本体で定義されている全Featureのカタログ**と、**この公開スナップショットで実際に読めるFeature仕様**を分離するためのものです。公開範囲や公開仕様は今後の整理・レビューに応じて更新される場合があります。

## このリポジトリは何ですか？

ここは **GST の公開ドキュメント専用リポジトリ**です。

ソースコードや内部の開発作業記録を配布する場所ではありません。公開しても問題ないと判断した設計・仕様・アーキテクチャ関連文書を、外部の読者が読みやすい形で掲載しています。

そのため、ここにある文書は次のような情報を知りたい人向けです。

- GST がどのような考え方で設計されているか
- セキュリティ境界や安全設計がどう定義されているか
- アーキテクチャがどのように構成されているか
- 各機能が何を目的としているか
- どのような条件・制約・検証方針を設けているか

**このリポジトリ自体が GST 本体ではありません。**

また、公開ドキュメントは GST 本体の仕様をそのまま逐次ミラーするものではなく、公開時点で選定・確認された内容による公開スナップショットです。

## 🚧 現在のステータス

**GST は WIP (Work in Progress) です。**

この公開リポジトリは、2026-09-29 時点で整理・確認された**公開対象ドキュメントのスナップショット**です。GST本体の内部リポジトリを逐次ミラーするものではありません。Feature CatalogはGST本体で定義されているFeature全体を可視化するための一覧であり、ここに仕様本文が収録されているかどうかとは別に管理します。内部AI作業資料・Change Control・作業用Handoff・AgentContext等は公開ナビゲーションの対象外です。

この段階では、設計文書の充実度が高くても、それだけで製品全体が完成したことにはなりません。

特に、次の状態は区別して扱います。

| 状態 | 意味 |
|---|---|
| Design Approved | 設計・仕様として承認された状態 |
| Implementation In Progress | 実装を進めている状態 |
| Implemented | 実装された状態 |
| Verified | 所定の検証を完了した状態 |
| Released | ユーザー向けにリリースされた状態 |

公開文書に **Approved / 確定 / 正式登録** などの表現があっても、それだけで Implemented / Verified / Released を意味するものではありません。

## 📚 どこから読めばいい？

まずは [Documentation/README.md](Documentation/README.md) を入口として利用してください。

### Architecture

[01 Architecture](Documentation/01_Architecture/README.md) では、GST の構造、技術スタック、セキュリティ境界、永続化、Windows 統合、UI などを確認できます。

### Features

[02 Features](Documentation/02_Features/README.md) では、ユーザー向け機能の詳細仕様を確認できます。

### Core Modules

[Core Modules](Documentation/modules/03-00_Modules_Overview.md) では、GST の主要な実行モジュールと、それぞれの責務・安全設計を確認できます。

### Security / Error Handling

- [Security Boundary and Test Vectors](Documentation/SECURITY_BOUNDARY_AND_TEST_VECTORS.md)
- [Error Handling and Event Codes](Documentation/ERROR_HANDLING_AND_EVENT_CODES.md)
- [Project Structure and Namespace Specification](Documentation/PROJECT_STRUCTURE_AND_NAMESPACE_SPEC.md)

## 🔐 GST の設計方針

公開されている設計文書では、主に次の考え方を重視しています。

- Local First / Privacy First / Security First / User Control First
- 最小権限を基本とする設計
- 保護操作とユーザー資産の責務分離
- 検知・説明・復旧の分離
- 不必要な自動削除や重大な環境変更を避けること
- 必要な特権操作を分離して扱うこと
- 外部通信を原則として限定し、AI 連携などは明示的なオプトインを前提とすること
- 不確実な状態では安全側に倒す Fail-Closed の考え方

これらは GST の設計思想・要件を説明するものであり、個々の機能について実証済みの保証を意味するものではありません。

## 🧭 公開範囲について

このリポジトリには、内部開発用の AI 指示、作業コンテキスト、Change Control、作業用Handoff、AgentContext などは公開ナビゲーションとして含めません。

特に、旧 `Documentation/AgentContext/` の作業用文書は公開対象ではありません。内部リポジトリ側で整理・アーカイブされた履歴資料を、この公開リポジトリの現行仕様として扱わないでください。

また、旧ビルド成果物、バイナリ、PDB、キャッシュ、`bin/`、`obj/` なども公開対象外です。

公開資料を読む際は、**現在の設計資料・歴史資料・実装済み機能・検証済み機能を混同しないこと**が重要です。

## 📌 このリポジトリで期待できること

このリポジトリでは、GST を「完成した製品」としてではなく、**どのようなセキュリティツールを作ろうとしているのかを公開設計資料から確認できる場所**として見てください。

仕様の改善、文書の整理、実装、検証が進むにつれて、公開内容も更新される可能性があります。

## 📄 License

Except where otherwise noted, the GST-authored documentation in this repository is licensed under the **Creative Commons Attribution 4.0 International (CC BY 4.0)**.

See [LICENSE](LICENSE) for the applicable license text.

Third-party works, trademarks, logos, and other material subject to rights not held by the project are not automatically relicensed by this notice and remain subject to their own terms.
