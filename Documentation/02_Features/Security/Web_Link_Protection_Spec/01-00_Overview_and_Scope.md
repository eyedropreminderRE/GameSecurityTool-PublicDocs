# 01-00: Web Link Protection Overview and Scope

**Document ID:** GST-SPEC-WEBLINK-001-PART0  
**Version:** 2.2 (PID Reuse Time Window & Clean 5-Layer Hardened)
**Status:** Approved Feature Specification Index  
**Feature Category:** Web Privacy / Browser Launch Control  
**Target Platform:** Windows 10 / Windows 11 (64-bit)  
**Technology Baseline:** .NET 10 LTS / C# 14 / WPF / WMI Event / Win32 SafeProcessHandle  

> **Lifecycle note:** This specification describes approved feature scope. Approval does not by itself indicate that the feature is implemented, Windows-verified, or released.

---

# 1. Purpose & Objectives

GameSecurityTool（GST）の **Web Link Protection** は、ゲーム内および関連ツールから発生する Web リンク起動について、以下の保護を提供する仕様書群（全7ファイル）の公式統合インデックスである。

- **意図しないブラウザ起動の抑制:** ゲーム内広告や誤操作によるブラウザ起動を未然に遮断。
- **SNS / コミュニティリンクの制御:** ゲームプレイ中の SNS・外部コミュニティへのアクセス制御。
- **トラッキング・認証トークン漏洩の遮断:** URL に含まれるセッション ID、認証トークン、UTM 追跡パラメータを構造的に完全破棄。
- **Browser Picker による選択制御:** 起動するブラウザ（通常/プライベートモード等）を用途に応じてユーザーが選択。
- **想定外のブラウザ起動に対する緊急検知:** WMI Emergency Detection による迂回プロセスの安全な検知・確認。

---

# 2. Responsibility Boundary (保証範囲と非目標)

```text
┌────────────────────────────────────────────────────────┐
│  GameSecurityTool が保証する範囲 (Managed Launch Path) │
│  ・GST の URL Broker を経由して起動される Web リンク   │
│  ・ブラウザプロセス起動「前」のポリシー判定・遮断       │
│  ・Strict Query Stripping によるパラメータ完全除去    │
│  ・ユーザー選択に基づく安全なブラウザプロセスの起動    │
└────────────────────────────────────────────────────────┘
                           │
                           ▼ (管理外の直接起動)
┌────────────────────────────────────────────────────────┐
│  多層防御 (Defense in Depth) による補助検知            │
│  ・WMI Emergency Detection (迂回起動の検知・確認)      │
│  ・Zero Trust Firewall (実際のネットワーク通信遮断)    │
└────────────────────────────────────────────────────────┘
```

### 非目標（保証しない事項）:
- PC 全体の全ブラウザ起動・全プロトコルハンドラーの完全監視
- 全 DNS 通信の完全遮断（ネットワーク通信自体の遮断は Firewall の責務）
- OS の既定ブラウザ設定（`UserChoice` レジストリ）の強制書き換え
- ゲームプロセスへの DLL 注入やメモリ改変による API フック
- Kernel Driver や常駐 Windows サービスによるパケット監視

---

# 3. Specification Structure (全7ファイル体系)

```text
Documentation/02_Features/Security/Web_Link_Protection_Spec/
├── 01-00_Overview_and_Scope.md                   <-- [本書: 統括] 概要・保証境界・非目標
├── 01-01_Domain_Model_and_Policies.md            <-- [Domain] ホストルール・判定優先順位・正規化
├── 01-02_Application_UseCases_and_Pipeline.md   <-- [Application] 起動ユースケース・7段階パイプライン
├── 01-03_Contracts_and_DTOs.md                  <-- [Contracts] Port (Interface)・不変 DTO 定義
├── 01-04_Infrastructure_and_Adapters.md        <-- [Infrastructure] URL サニタイザー・WMI・プロセス制御
├── 01-05_Presentation_and_Dialogs.md           <-- [Presentation] Browser Picker・確認ダイアログ・VM
└── 01-06_Security_Audit_and_Edge_Cases.md       <-- [Security/Test] 監査ログ規約・テスト・Fail-Safe
```

> **【ナンバリングに関する注意事項】**  
> 本ディレクトリ配下のファイルは `01-00` 〜 `01-06` のプレフィックスを使用しています。`01_Architecture/` 配下のファイル（`01-00` 〜 `01-07`）とナンバリングが重複しているため、参照時は必ず親ディレクトリパスを含めて識別してください。

---

# 4. 共通セキュリティ規約

1. **Strict Query Stripping (クエリ原則全削除):**
   個人識別情報（PII）やセッショントークンの漏洩を構造的に遮断するため、`?` 以降のパラメータはデフォルトで完全破棄する（明示許可された Parameter Allowlist のみ例外維持）。
2. **ホスト境界マッチング ＆ 公式サポートドメイン厳格照合:**
   `host.Contains("example.com")` のような部分一致を厳禁とし、完全一致（ExactHost）またはドット境界サブドメイン一致（`*.example.com`）のみを許可する。公式サポートドメインのホワイトリスト照合時、キリル文字等を用いたホモグラフ攻撃（Punycode 偽装）やタイポスクワッティングを厳格に検知・ブロックする。
3. **PID Reuse（再利用）時間対称検証:**
   WMI 緊急検知時、プロセス作成時刻（`CreationTime`）がイベント受信時刻前後の対称ウィンドウ内（`eventTimeUtc - 3s <= CreationTime <= eventTimeUtc + 3s`）にあることを同一ハンドル上で確認する。
4. **外部ブラウザ安全起動 ＆ 引数インジェクション排除:**
   シェル実行（`UseShellExecute = true`）を禁止し、`ProcessStartInfo.ArgumentList` を使用して URL を引数として明示的に渡す。ハイフンで始まる URL やクォートインジェクションを検知・無害化し、オプション終端セパレータ（`--`）を付与して外部ブラウザ起動時の引数インジェクションを物理的に遮断する。

---

# 5. Definition of Done (完成基準)

- Managed Launch Path においてブラウザ起動前の Block / Confirm / Allow が機能すること。
- Strict Query Stripping により、デフォルトで `?` 以降のクエリが完全除去されること。
- 監査ログに生の URL や Query String が一切記録されないこと。
- WMI 検知時に PID Reuse 対策の時刻検証・パス検証が正しく動作すること。
- 公式ドメイン照合および Punycode 偽装検知が正しく機能すること。
- 単体テスト・セキュリティテストがすべて合格すること。

---
