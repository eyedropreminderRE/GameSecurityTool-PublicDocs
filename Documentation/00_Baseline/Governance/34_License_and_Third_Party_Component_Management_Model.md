# GameSecurityTool

# License and Third Party Component Management Model

## Open Source / Dependency / License Compliance Specification

---

# Document Control

| Item | Value |
|---|---|
| Document ID | GST-BASELINE-034 |
| Version | 3.2 (Vulnerability Gate, Lock-File & Fixed Toolchain Edition) |
| Status | Formal Baseline Specification (Highest Dependency Authority) |
| Category | Software Compliance |
| Authority Level | Dependency Governance Specification |
| Parent Document | 00_Formal_Baseline_Overview.md |

---

# 0. Purpose

本書は GameSecurityTool（以下GST）における、

- 第三者ライブラリ（Third-Party Dependencies / NuGet Packages）の採用基準
- OSS ライセンス適合性管理（MIT / Apache 2.0 等）
- 脆弱性監視およびサプライチェーンセキュリティ
- 公式承認パッケージインベントリ
- 全26章にわたる依存関係ライフサイクル統治（審査、更新、不要依存削除、内部vs外部判断、UI/DB/ビルド依存、ライセンス文書化、障害対応）

を定義する。

---

目的：

```text
Use External Components Safely
*
Maintain 100% Legal & License Compliance
*
Protect Product Reliability & Supply Chain
*
Prevent Unverified Dependencies
```

---

# 1. License Philosophy

## 1.1 Core Principle
GST では、「便利だから安易に追加する」のではなく、**セキュリティレビュー・ライセンス適合性・長期保守性が証明されたコンポーネントのみを厳選して採用する。**

基本方針：
```text
Every Dependency Has Responsibility
(すべての外部依存関係はセキュリティと品質の責任を負う)
```

---

# 2. Third Party Component Classification (依存関係の分類)

## 2.1 Runtime Dependency (実行時必須依存)
アプリケーション動作に不可欠な ORM、暗号、アーカイブ、UI、DI 基盤。
- **例:** EF Core 10, SQLite, `SharpCompress`, `Konscious.Security.Cryptography.Argon2`, `CommunityToolkit.Mvvm`

## 2.2 Development Dependency (開発・テスト専用依存)
テスト自動化やコード解析専用フレームワーク。
- **例:** `NetArchTest.Rules`, `xUnit`, `Moq` (※ リリースバイナリへ混入させない)

## 2.3 Optional Dependency (追加機能用依存)
BYOK Gemini API 通信クライアント（.NET 10 標準 `HttpClient` を基本とし、不要な巨大 SDK 依存を排除）。

---

# 3. Dependency Approval Process (導入審査プロセス)

新規ライブラリ導入時は、以下の審査フローを必須とする：

```text
Identify Component (導入候補の特定)
       ↓
Check License (GPL 系・感染性ライセンスの排除確認: MIT / Apache 2.0 限定)
       ↓
Security & Vulnerability Review (既知の CVE / 過去のセキュリティ履歴の確認)
       ↓
Compatibility Review (.NET 10 LTS / Windows 10/11 適合性確認)
       ↓
Approve Usage (最上位仕様書への登録 ＆ 正式承認)
```

---

# 4. License Classification (ライセンス分類)

確認対象：
- **MIT License** (最優先採用)
- **Apache License 2.0**
- **BSD 2-Clause / 3-Clause**
- **GPL Family (GPL / AGPL / LGPL)** (採用禁止)
- **Commercial / Proprietary License** (採用制限)

---

# 5. License Compatibility Rule (ライセンス互換性規則 - 完全復元)

採用時に以下を厳格に確認する：
- 配布方式（Distribution Method）との適合性
- ソースコード開示義務の有無（感染性ライセンスの完全排除）
- 著作権表示および許諾条文の保持要件
- 商用・非商用を問わない利用権

---

# 6. Restricted License Handling (制限ライセンスの取り扱い - 完全復元)

注意・禁止対象：
- **Strong Copyleft License (GPL / AGPL):** GST プロジェクト全体へのソース開示義務波及を防ぐため、混入を厳禁とする。
- **Unknown / Unclear License:** 著作権者やライセンス条文が不明瞭なパッケージの採用禁止。

原則：
```text
Do Not Include Without Explicit Legal Review
(法的レビューと承認のないライセンスは一切含めてはならない)
```

---

# 7. Official Approved Dependency Inventory (公式承認インベントリ)

製品全体で利用が公式承認されている外部 NuGet パッケージのマスターインベントリ：

| パッケージ名 | バージョン基準 | ライセンス | 採用目的 / レイヤー |
| :--- | :---: | :---: | :--- |
| **`Microsoft.Extensions.Hosting`** | 10.x | MIT | Generic Host 基盤 / ライフサイクル管理 |
| **`Microsoft.Extensions.Logging`** | 10.x | MIT | 構造化ログ出力基盤 |
| **`Microsoft.Extensions.DependencyInjection`** | 10.x | MIT | DI コンテナ / サービス登録 |
| **`CommunityToolkit.Mvvm`** | 8.x | MIT | WPF MVVM Source Generators 基盤 |
| **`Microsoft.EntityFrameworkCore`** | 10.x | MIT | ORM 永続化基盤 |
| **`Microsoft.EntityFrameworkCore.Sqlite`** | 10.x | MIT | SQLite プロバイダ (PRAGMA WAL) |
| **`Microsoft.Data.Sqlite`** | 10.x | MIT | SQLite ネイティブ接続プール管理 |
| **`Konscious.Security.Cryptography.Argon2`** | 1.x | MIT | **ポータブル KDF / PC 移行暗号化基盤** |
| **`SharpCompress`** | 0.38+ | MIT | **.7z / .rar / .zip マネージド安全展開基盤** |
| **`NetArchTest.Rules`** *(Test only)* | 1.x | MIT | **CI アーキテクチャ 5層依存自動検証** |
| **`xunit` / `Moq`** *(Test only)* | 2.x / 4.x | Apache/BSD | 単体・結合・セキュリティ自動テスト |

※ 上記以外のサードパーティ製 NuGet パッケージの無断追加を禁止する。

---

# 8. Version Management (バージョン管理 - 完全復元)

依存関係の更新時に確認する項目：
- 新バージョンにおける変更内容（Changelog）
- 破壊的変更（Breaking Changes）の有無
- セキュリティ修正（Security Patches）の内容
- ライセンス条文の変更の有無

---

# 9. Vulnerability Management (脆弱性管理)

1. **既知脆弱性の監視:** CI パイプラインは `dotnet package list --vulnerable --include-transitive --format json --output-version 1` を実行し、直接依存だけでなく推移的依存を含む既知の脆弱性を検査する。
2. **CI Failure Gate:** 既知の脆弱性が1件でも検出された場合、CI は失敗として扱う。NU1901〜NU1904 も抑制せず、WarningsAsErrors 対象として扱う。
3. **Lock File:** 現行 Production / Test プロジェクトは `packages.lock.json` を追跡し、CI の restore は `dotnet restore --locked-mode` で実行する。lock file の変更を伴わない依存解決結果の更新は CI で許可しない。
4. **固定ツールチェーン:** `global.json` は SDK `10.0.401` を `rollForward=disable` / `allowPrerelease=false` で固定し、`Directory.Build.props` は C# `14.0` を指定する。現行本番 PackageReference は stable `10.0.0` を明示固定する。Preview 依存が必要な場合は独立した Change Control が必要である。
5. **例外:** セキュリティ上の例外を設ける場合は、対象パッケージ、影響、緩和策、有効期限、解除条件を独立した Change Control として承認・記録する。
6. **対応プロトコル:** 脆弱性が発見された場合、直ちに Update（更新）、Replace（代替置換）、または Mitigate（緩和策適用）を実行する。

---

# 10. Security Priority (セキュリティ優先原則 - 完全復元)

依存関係の管理においては、新機能の便利さよりも安全性を絶対優先する。

禁止事項：
```text
Unmaintained Critical Dependency
(長期間メンテナンスが停止している重要ライブラリの放置を禁止する)
```

---

# 11. Package Source Verification (取得元検証 - 完全復元)

パッケージ取得元の統制：
- 公式リポジトリ（`nuget.org`）からの取得に限定する。
- 発行元（Publisher）の正当性、パッケージハッシュ、およびデジタル署名を検証する。
- 不明な外部 URL やプライベートフィードからの無検証パッケージ取得を禁止する。

---

# 12. Hash and Integrity Verification (ハッシュ ＆ 完全性検証 - 完全復元)

重要外部バイナリおよびパッケージのチェックサム（SHA-256）とデジタル署名を検証し、サプライチェーン攻撃による改ざんを防止する。

---

# 13. Dependency Update Policy (更新方針 - 完全復元)

依存関係の更新手順：
```text
Security Review ➔ Compatibility Test ➔ Regression Verification ➔ Documentation Update ➔ Release
```
無検証での自動メジャーアップデートを禁止する。

---

# 14. Unused Dependency Management (不要依存の整理 - 完全復元)

定期的な依存関係の棚卸しを実施する：
- 現在の機能で本当に必要か
- メンテナンスが継続されているか
- 不要になったパッケージはプロジェクトから即座に削除する（攻撃対象領域 Attack Surface の最小化）

---

# 15. Internal Code vs External Code (自前実装 vs 外部依存の判断基準 - 完全復元)

外部ライブラリ導入前の判断基準：
1. **GST で安全に自前実装可能か:** 数十行のユーティリティ関数等のために不要な外部ライブラリを導入してはならない。
2. **外部コンポーネントが不可欠か:** 暗号アルゴリズム（Argon2id）やマルチアーカイブ展開（SharpCompress）のように、自作がリスクを生む高度な技術領域に限定して外部ライブラリを採用する。
3. **長期保守コスト:** 外部ライブラリの将来的な保守負担を評価する。

---

# 16. Security Library Rule (暗号ライブラリ統制)

1. **カスタム暗号実装の完全禁止:** 暗号アルゴリズム（AES, SHA-256, Argon2id）の自作を厳禁とし、OS 標準（.NET 10 `AesGcm` / DPAPI）および承認済みライブラリ（`Konscious.Security`）のみを使用する。
2. **統一パラメータの遵守:**
   - **Argon2id:** Iterations = 3, Memory = 64MB (65536 KB), Parallelism = 4, Salt = 16 Bytes
   - **AES-256-GCM:** KeySize = 256-bit, Nonce = 12 Bytes, Tag = 16 Bytes, ChunkSize = 64 KB

---

# 17. UI Framework Management (UI フレームワーク管理 - 完全復元)

- **採用基盤:** Windows 10/11 公式 WPF + Fluent Design（Mica / Acrylic 素材）
- **MVVM 基盤:** `CommunityToolkit.Mvvm`（Microsoft 公式サポート / 長期保守性保証）
- **ライセンス適合性:** MIT ライセンスに準拠。

---

# 18. Database Component Management (DB コンポーネント管理 - 完全復元)

- **採用基盤:** `Microsoft.EntityFrameworkCore.Sqlite` ＆ `Microsoft.Data.Sqlite`
- **データ安全性:** SQLite PRAGMA WAL モード、`IDbWriteQueue` 直列化、およびマイグレーション互換性の維持。

---

# 19. Build Tool Management (ビルドツール管理 - 完全復元)

- **コンパイラ / SDK:** .NET 10 SDK / C# 14 LTS
- **ビルド再現性:** ビルド環境差異による不具合を防ぐため、SDK バージョンおよび依存パッケージバージョンを厳格に固定管理する。

---

# 20. AI Generated Dependency Rule (AI 提案ライブラリの禁止)

AI コーディングエージェントが、第7章の承認インベントリに記載されていない未知のライブラリや非推奨パッケージを提案・追加することを厳禁とする（Blind Adoption の完全排除）。

---

# 21. License Documentation (ライセンス文書化 - 完全復元)

リリース成果物への添付要件：
- 配布パッケージ内に `ThirdPartyNotices.txt`（利用している全 OSS の著作権表示およびライセンス全文）を必ず同梱する。

---

# 22. Release Requirement (リリース前要件 - 完全復元)

正式公開前の必須確認：
- [ ] 依存関係インベントリが 100% 正確に文書化されていること
- [ ] すべての外部パッケージのライセンスレビューが完了していること
- [ ] 既知の脆弱性（CVE）が存在しないこと

---

# 23. Dependency Failure Handling (依存関係障害対応 - 完全復元)

外部パッケージに重大な脆弱性や不具合が発覚した場合の対応手順：
1. 影響を受けるコンポーネントおよび機能の即時特定
2. 修正パッチ版への更新、または安全な代替パッケージへの置換
3. 回帰テスト完了後の緊急パッチリリース

---

# 24. Compliance Checklist

確認項目：

☐ 使用しているすべての外部パッケージが第7章の承認インベントリに記載されていること  
☐ **すべてのランタイム依存が MIT / Apache 2.0 等の非コピーレフトライセンスであること**  
☐ **`Konscious.Security` および `SharpCompress` のバージョンとライセンスが適合していること**  
☐ カスタム暗号アルゴリズムの実装が存在しないこと  
☐ リリースパッケージに Third-Party Notices（ライセンス条文一覧）が添付されていること  
☐ 未使用・不要になった外部依存がプロジェクトから削除されていること  

---

# 25. Relationship With Other Models (他モデルとの関係性 - 完全復元)

## 25.1 Technology Stack Standards
全社技術スタックおよび暗号化標準：
```text
01_Architecture/01-02_Technology_Stack_and_Runtimes.md §57
```

## 25.2 Source Code Structure
ソースコード構造およびモジュール責務：
```text
Architecture/28_Source_Code_Structure_and_Module_Architecture_Model.md
```

## 25.3 Security Implementation
セキュアコーディングおよび暗号実装指針：
```text
Security/14_Security_Implementation_Guideline.md
```

## 25.4 External Integration
外部サービス連携およびプラットフォーム互換性：
```text
Architecture/30_External_Integration_and_Platform_Compatibility_Model.md
```

## 25.5 Release Gate
品質保証およびリリース判定ゲート：
```text
Quality/27_Quality_Assurance_and_Release_Gate_Model.md
```

---

# 26. Final License Statement

GST における第三者コンポーネント管理とは、単なるライブラリ一覧管理ではない。

---

**安全性・継続性・法的利用可能性を維持し、ユーザーが安心して 5 年先・10 年先でも利用し続けられる信頼性の高いソフトウェアを提供するための基盤である。**

---

Final Principle:

```text
Know What We Use Completely
Verify Before Adoption Always
Update Responsibly
Respect Every License
```

---

End of Document
```