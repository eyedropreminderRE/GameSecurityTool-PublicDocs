# GameSecurityTool Architecture & Technology Baseline

本ディレクトリは、GameSecurityTool（GST）の技術基盤、Clean 5-Layer アーキテクチャ、およびそれを補完するUI Design Working-Draft setの公開候補を構成します。UI Working-Draftは設計上の記述であり、それ自体がRuntime実装や機能提供を承認するものではありません。

---

## 技術アーキテクチャ基準 (`01-00` 〜 `01-07`)

* **[01-00_Overview_and_Principles.md](01-00_Overview_and_Principles.md)**  
技術スタック、Solution構造、Source of Truth優先順位、およびコア原則。
* **[01-01_Clean_Hexagonal_Architecture.md](01-01_Clean_Hexagonal_Architecture.md)**  
Presentation / Application / Domain / Contracts / Infrastructure の5層レイヤー依存方向とPort & Adapter境界。
* **[01-02_Technology_Stack_and_Runtimes.md](01-02_Technology_Stack_and_Runtimes.md)**  
C# 14、Nullable参照型、Generic Host、DI等の技術基準。
* **[01-03_Security_Architecture_and_Threat_Model.md](01-03_Security_Architecture_and_Threat_Model.md)**  
Standard User実行モデル、UAC Worker、Named Pipe IPC、TOCTOU防御、暗号化基準。
* **[01-04_Persistence_and_Database_Architecture.md](01-04_Persistence_and_Database_Architecture.md)**  
EF Core 10 / SQLite、WAL、Single Writer Queue、永続化境界。
* **[01-05_OS_Integration_and_Hardware_Boundary.md](01-05_OS_Integration_and_Hardware_Boundary.md)**  
Win32 Native API、Windows Firewall、WMI、WER等のOS境界。
* **[01-06_Presentation_and_UI_Architecture.md](01-06_Presentation_and_UI_Architecture.md)**  
WPF MVVM、Dispatcher境界、Timeline View / QueryのPresentation基準。
* **[01-07_Cross_Cutting_Concerns_and_Lifecycle.md](01-07_Cross_Cutting_Concerns_and_Lifecycle.md)**  
構造化ログ、個人情報マスキング、品質ゲート、ライフサイクル。

## UI Design Working Drafts

* **[01-08_UI_Information_Architecture_and_Interaction_Concept.md](01-08_UI_Information_Architecture_and_Interaction_Concept.md)**  
6画面IA、Navigation、User Journey、Interaction、Cross-Screen UX。Formal Baseline / Feature Specification / Runtime capabilityを上書きしない設計Working Draft。
* **[01-09_UI_Visual_Design_System.md](01-09_UI_Visual_Design_System.md)**  
Visual Foundation、Design Tokens、Common Component States、Visual Safety / Acceptance。
* **[01-10_UI_Home_and_Dashboard_Design.md](01-10_UI_Home_and_Dashboard_Design.md)**  
Home / Dashboard の画面設計。
* **[01-11_UI_Games_Design.md](01-11_UI_Games_Design.md)**  
Games 画面設計。
* **[01-12_UI_Save_Data_Design.md](01-12_UI_Save_Data_Design.md)**  
Save Data 画面設計。Formal Save Backup UI仕様を上書きしない。
* **[01-13_UI_Protection_Design.md](01-13_UI_Protection_Design.md)**  
Protection 画面の表示・横断ステータス設計。
* **[01-14_UI_Recovery_Design.md](01-14_UI_Recovery_Design.md)**  
Recovery 画面・Recovery flow のUI設計。
* **[01-15_UI_Settings_Design.md](01-15_UI_Settings_Design.md)**  
Settings 画面設計。
* **[01-16_UI_Common_Interaction_and_Display_Design.md](01-16_UI_Common_Interaction_and_Display_Design.md)**  
共通Interaction / Display設計。
* **[01-17_UI_Reference_Patterns_and_Common_Component_Library.md](01-17_UI_Reference_Patterns_and_Common_Component_Library.md)**  
Reference Pattern / 共通Component Library。
* **[01-18_UI_User_Journeys_and_Review_Model.md](01-18_UI_User_Journeys_and_Review_Model.md)**  
User Journey / Review model。
* **[01-19_UI_Cross-Screen_Flow_and_Transition_Model.md](01-19_UI_Cross-Screen_Flow_and_Transition_Model.md)**  
Cross-Screen Flow / Transition / responsibility mapping。
* **[01-20_UI_End-to-End_Scenario_Validation.md](01-20_UI_End-to-End_Scenario_Validation.md)**  
End-to-End Scenario validation。
`01-08` と重複するVisual Design本文を持たず、UI見た目の共通設計を一元管理する。

> `01-08` / `01-09` は `01-00`〜`01-07` の技術アーキテクチャ基準を補完する従属文書です。これらの存在だけでRuntime Feature、Public Contract、Domain Status、Security capability、Implementation Readinessが変更されたとはみなしません。

## Other architecture-area specifications

* **[Feature_Integration_Update_v1.0.md](Feature_Integration_Update_v1.0.md)**  
追加3機能（LNK検知、セーブデータ自動バックアップ、Clipboardサニタイザー）の統合更新仕様。

## 関連仕様書

* **[00_Baseline/](../00_Baseline/)** — 製品最高権限ベースライン仕様群
* **[modules/](../modules/)** — 6大コア実行モジュール詳細仕様書
* **[02_Features/](../02_Features/)** — 各機能の詳細設計書
* AI development workflow, internal decision records, and repository-operation procedures are intentionally excluded from the public architecture navigation.
