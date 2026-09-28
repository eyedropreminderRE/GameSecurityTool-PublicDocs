# 04-00: Security Trust Enhancement Overview and Principles

**Document ID:** GST-FEAT-TRUST-INDEX-004  
**Version:** 2.1 (Timeline Pagination Default Fixed)
**Status:** Approved Feature Specification Index  
**Feature Category:** Explainable Security / User Decision Support  
**Target Platform:** Windows 10 / Windows 11 (64-bit)  
**Technology Baseline:** .NET 10 LTS / C# 14 / WPF / SQLite + EF Core 10 / Stream Compression  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 0. Purpose & Philosophy

本仕様は、GameSecurityTool（GST）における検知結果・監査証跡・設定状態を、ユーザーが直感的に理解・判断可能な形で提示するための **Trust Enhancement（信頼性・説明可能性補強）** 仕様書群（全7ファイル）の公式統合インデックスである。

本機能は Threat Detection Logic そのものを置換するものではなく、以下を補強する：
- **Detection Engine と Explanation Engine の完全分離:** Domain 層に自然言語文字列や UI 依存を一切混入させない。
- **Non-Binary Security (確信度モデル):** 「安全/危険」の2値ではなく、確信度（Confidence Score: 0.0〜1.0）と危険度（RiskLevel）を組み合わせて提示。
- **Evidence Based Decision:** ハッシュ、デジタル署名、設定スナップショット等の客観的証拠（Evidence）に基づく説明。

GameSecurityTool を単なる検知ツールではなく、**「Explainable Security Assistant（意思決定支援基盤）」** として提供することを目的とする。

---

# 1. Specification Structure (全6ファイル体系)

本機能に関する仕様は、Clean 5-Layer アーキテクチャのレイヤーおよび責務に応じて以下の全7ファイルで管理する。

```text
Documentation/02_Features/Security/Trust_Enhancement_Spec/
├── 04-00_Overview_and_Principles.md              <-- [本書: 統括] 概要・設計原則・DoD
├── 04-01_Domain_Confidence_and_Signals.md        <-- [Domain] 確信度モデル・不変シグナル
├── 04-02_Application_Explanation_Engine.md       <-- [Application] Explanation Engine・多言語解釈
├── 04-03_Configuration_Snapshot_Model.md         <-- [Domain/Infra] 設定スナップショット永続化
├── 04-04_Evidence_Bundle_and_Export.md           <-- [App/Infra] ストリーミングZip出力・プライバシー
├── 04-05_Timeline_View_and_Contracts.md         <-- [Contracts/UI] タイムライン表示・遅延読み込み
└── 04-06_Gemini_Ai_Integration_and_Consent_Spec.md <-- [AI] BYOK Gemini API連携・同意
```

---

# 2. Architecture Placement & Data Flow

```text
[ Domain Layer (GST.Domain) ]
RiskAssessmentResult (純粋な判定データ: RiskLevel, ConfidenceScore, EvidenceCollection)
    │ (UI Text・言語リソース非保持)
    ▼
[ Application Layer (GST.Application) ]
Explanation Engine (証拠解釈・多言語リソース適用・DTO変換)
    │
    ▼
[ Contracts Layer (GST.Contracts) ]
RiskExplanationDto (不変 DTO)
    │
    ▼
[ Presentation Layer (GST.Presentation) ]
RiskExplanationView / ViewModel (UI描画・ユーザー判断受付)
```

---

# 3. コア原則と制約事項

1. **Domain 層の純粋性保護:**
   Domain 内で `Message = "このファイルは危険です"` 等の自然言語文字列を生成することを厳禁とする。Domain は列挙型シグナル（`FileUnsigned`, `HashMismatch` 等）のみを保持する。
2. **確信度による自動処罰の禁止:**
   低い確信度（例: Confidence 30% の High Risk）を理由に、ツールの独断で自動削除・強制隔離・自動許可を行ってはならない。
3. **エビデンス出力のストリーミング必須:**
   `EvidenceBundle.zip` の生成において、`File.ReadAllBytes()` や巨大 `MemoryStream` を禁止し、`FileStream -> ZipArchive -> Entry Stream` によるストリーミング I/O を強制する。
4. **Timeline クエリの仮想化 & ページネーション:**
   監査ログの全件取得（`SELECT * FROM AuditRecords`）を禁止し、日付範囲フィルター付きのページネーション（**標準既定値50件/ページ**）を必須とする。`pageSize` の明示指定は許容するが、UI標準動作の既定値は50件とする。

---

# 4. Definition of Done (完成基準)

- Domain / Application / Contracts / Infrastructure / UI の 5層境界が厳格に維持されていること。
- 監査ログハッシュチェーン（Audit Hash Chain）の完全性が保証されていること。
- エビデンスバンドル出力時のメモリ使用量が 120MB 以下を維持すること（ストリーミング検証）。
- 単体テスト・統合テストが すべての定義済み パスすること。
```

---