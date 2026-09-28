# GameSecurityTool 追加3機能 統合更新仕様書

**文書ID:** GST-FEAT-INTEGRATION-001  
**版:** 1.0
  
**状態:** 設計承認済みの統合方針（実装完了を意味しない）

## 1. 統合対象として承認された機能

以下3機能を統合対象として定義する。各機能の実装状態は、対応するFeature / Module仕様および検証結果に従う。

1. `.LNK` 改ざん検知
2. セーブデータ安全自動バックアップ
3. On-Demand Clipboard URL Sanitizer

## 2. 採用理由

### LNK改ざん検知

既存のSecurity Engine、Snapshot、ThreatReason、AuditLogを再利用でき、ゲームショートカットを利用した永続化・起動経路改変への可視性を高められる。

### セーブデータ自動バックアップ

現在の「検知・隔離・復元」を補完し、正常状態への復旧可能性を高める。

### Clipboard Sanitizer

Game-Originだけでなく、ユーザー自身がコピーしたUser-Origin URLにも、ユーザー主導で安全化する手段を提供できる。

## 3. 共通アーキテクチャ原則

- Clean Architecture
- Port & Adapter
- DomainはOS API非依存
- ApplicationはInfrastructure具象を直接参照しない
- DTOはContractsで境界化
- OperationIdで監査追跡
- TransactionIdは原子性が必要な処理だけで使用
- UIはMVVM
- Standard User起動を維持
- UAC常時要求を禁止
- Kernel Driver禁止
- 常駐Windows Service禁止
- 外部通信禁止（既定）
- Telemetry禁止

## 4. 既存機能との統合

| 新機能 | 再利用する既存基盤 |
|---|---|
| LNK改ざん検知 | SecurityEngine / ThreatReason / PathValidation / AuditLog / Snapshot |
| Save Backup | OperationId / TransactionId / RestoreTransaction / SecureStorage / PathValidation |
| Clipboard Sanitizer | StrictQueryUrlSanitizer / WebLinkPolicy / PrivacyLogSanitizer / Managed Browser Launcher |

## 5. 優先度

```text
P1
├─ LNK Hijack Detection
└─ On-Demand Clipboard URL Sanitizer

P2
└─ Save Data Secure Auto-Backup
```

## 6. 将来の境界候補

### LNK

- `ILinkInspector`

### Save Backup

- `ISaveBackupService`
- `ISaveBackupRepository`

### Clipboard

- `IClipboardUrlService`
- `IUrlSanitizer`
- `IManagedBrowserLauncher`

※ここに示す境界・DTOは統合設計上の候補であり、本書単独では新規Port / DTO / Enum / Contractの追加を承認しない。実際の追加や変更は、対応する正式仕様と承認済み変更管理に従う。

## 7. 将来のDTO候補

### LNK

- `ShortcutInspectionDto`
- `ShortcutDiffDto`

### Save Backup

- `SaveBackupRecordDto`
- `SaveRestoreTransactionDto`

### Clipboard

- `SanitizedUrlResultDto`
- `ClipboardUrlAuditDto`

## 8. ThreatReasonCode追加候補

- `LnkTargetChanged`
- `LnkArgumentsChanged`
- `LnkSuspiciousCommand`
- `LnkUnexpectedTarget`
- `LnkExternalTargetResolved`
- `LnkUnsignedTarget`
- `SaveBackupIntegrityFailed`
- `SaveRestorePartialFailed`
- `ClipboardUrlQueryRemoved`
- `ClipboardUrlInvalidScheme`
- `ClipboardUrlNormalizationFailed`

## 9. AuditEventType追加候補

- `ShortcutChangedDetected`
- `SaveBackupCreated`
- `SaveBackupFailed`
- `SaveRestoreStarted`
- `SaveRestoreCompleted`
- `SaveRestoreFailed`
- `ClipboardSanitized`
- `ClipboardSanitizationFailed`

## 10. 実装時の重要な禁止事項

- LNKを検知しただけで自動削除しない
- セーブバックアップ復元を自動実行しない
- クリップボードを常時監視しない
- Original URLをログ保存しない
- セーブ内容を外部送信しない
- ファイル操作時にパス文字列だけを信用しない
- `File.ReadAllBytesAsync`による巨大ファイル一括読み込みをしない
- Windows OS固有APIをDomainへ入れない

## 11. テスト戦略

### Unit Test

- Risk評価
- URL正規化
- Query stripping
- LNK差分判定
- Backup世代管理

### Integration Test

- Windows shortcut inspection
- Clipboard access
- Save backup / restore
- DPAPI
- SQLite

### Security Test

- LNK command injection
- Reparse Point substitution
- PID再利用関連フロー
- Backup path traversal
- URL scheme abuse
- Clipboard data leakage

## 12. ロードマップ反映

### MVP Alpha

既存コアに変更なし。

### 将来の統合段階

LNK検知、Clipboard Sanitizer、Save Data Secure Auto-Backupは、対応する正式仕様・実装計画・検証条件が成立した段階で順次統合する。ここで示す段階名は計画上の目安であり、実装済み・リリース済みを意味しない。

## 13. 完了条件

3機能について、以下の条件を満たすまで実装完了とは扱わない。

- Design Review完了
- Port / DTO定義完了
- Unit Test追加
- Integration Test追加
- Security Test追加
- 対応するビルド検証が成功していること
- Nullable警告がないこと
- Code Analysis上の未解決問題がないこと
- 既存機能への回帰テストが成功していること
