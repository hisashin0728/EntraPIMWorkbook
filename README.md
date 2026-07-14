# PIM Audit History Workbook

Microsoft Entra ID **Privileged Identity Management (PIM)** の監査ログ（`AuditLogs`）を、Azure Monitor / Microsoft Sentinel のブック（Workbook）で可視化するためのギャラリーテンプレートです。

`RoleManagement`（Entra ロール）、`GroupManagement`（グループ）、`ResourceManagement`（Azure リソース）の3カテゴリを、カテゴリごとに最適化した表と円グラフで確認できます。

## 主な機能

- **カテゴリ単位の切り替え表示** — パラメータ `Category`（単一選択プルダウン）で選んだカテゴリの分析グラフと監査履歴の表だけを表示。
- **カテゴリ別の監査履歴テーブル** — カテゴリごとに意味のあるフィールドのみを日本語ラベルで表示。
  - いつ / 誰が / 対象（Entraロール・Entraグループ・Azureリソース）/ 承認対象ユーザー / 申請内容 / 結果 / コメント
- **分析用の円グラフ（各カテゴリ4種）**
  - RoleManagement: Most Activated Roles / Top Requesting Users / Activity Type Breakdown / Top Approvers
  - GroupManagement: Most Activated Groups / Top Requesting Users / Activity Type Breakdown / Top Approvers
  - ResourceManagement: Most Activated Resources / Top Requesting Users / Activity Type Breakdown / Top Assigned Roles
- **絞り込みパラメータ** — `ActivityDisplayName` / `ResourceName` / `Group` / `Result`（複数選択・既定はすべて）。
- **`TargetResources` の type ベース抽出** — インデックス依存を避け、`Role` / `Group` / `Other` / `User` / `Subscription` などの `type` から値を抽出することで、イベント種別による項目ずれを防止。
- **承認イベントの明確化** — `request approved` イベントで「誰が（承認者）」と「承認対象ユーザー」を区別して表示。

## 前提条件

- Microsoft Entra ID **PIM**（Microsoft Entra ID P2 / Microsoft Entra ID Governance）が有効であること。
- PIM の監査ログが **Log Analytics ワークスペース** に送信されていること（Entra の *診断設定* で `AuditLogs` をワークスペースにエクスポート）。
- ブックを表示・保存する権限（例: Log Analytics ワークスペース / Sentinel に対する *Workbook Contributor* 以上）。

## デプロイ手順

### 方法A: ポータルに手動インポート（ギャラリーテンプレート）

1. Azure Portal で対象の **Log Analytics ワークスペース** または **Microsoft Sentinel** を開きます。
2. **ブック（Workbooks）** → **新規（New）** → 編集モードで **`</>` 詳細エディター（Advanced Editor）** を開きます。
3. テンプレートの種類（Template Type）を **Gallery Template** にします。
4. [`PIM-Workbook.galleryTemplate.v1.5.json`](PIM-Workbook.galleryTemplate.v1.5.json) の内容を貼り付けて **適用（Apply）** → **完了（Done Editing）**。
5. 上部パラメータで **Workspace**（対象ワークスペース）と **TimeRange**、**Category** を選択します。
6. 名前を付けて保存します。

### 方法B: ARM テンプレートでデプロイ（推奨・自動化向け）

ブックを Azure リソース（`microsoft.insights/workbooks`）として展開します。

- テンプレート: [`PIM-Workbook.deploy.json`](PIM-Workbook.deploy.json)
- パラメータ例: [`PIM-Workbook.deploy.parameters.json`](PIM-Workbook.deploy.parameters.json)

**Azure CLI:**

```bash
az deployment group create \
  --resource-group <ResourceGroup> \
  --template-file PIM-Workbook.deploy.json \
  --parameters @PIM-Workbook.deploy.parameters.json
```

**Azure PowerShell:**

```powershell
New-AzResourceGroupDeployment `
  -ResourceGroupName <ResourceGroup> `
  -TemplateFile PIM-Workbook.deploy.json `
  -TemplateParameterFile PIM-Workbook.deploy.parameters.json
```

#### ARM テンプレートのパラメータ

| パラメータ | 既定値 | 説明 |
|---|---|---|
| `workbookDisplayName` | `ISPF-PIM_Sample` | ブックの表示名 |
| `workbookType` | `workbook` | ブックのカテゴリ |
| `workbookSourceId` | `azure monitor` | Azure Monitor ギャラリーに置く場合は `azure monitor`。**Microsoft Sentinel / 特定ワークスペースに紐付ける場合は Log Analytics ワークスペースのリソースID**（例: `/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.OperationalInsights/workspaces/<workspace>`） |
| `workbookId` | `[newGuid()]` | ブックリソースの一意な GUID（未指定なら自動生成） |

> **Sentinel に表示したい場合:** `workbookSourceId` に対象 Log Analytics ワークスペースのリソースIDを指定してデプロイしてください。

## パラメータ

| パラメータ | 種別 | 説明 |
|---|---|---|
| `Workspace` | リソース選択（複数） | 対象の Log Analytics ワークスペース |
| `TimeRange` | 時間範囲 | 集計対象期間（既定: 過去14日） |
| `Category` | 単一選択 | 表示するカテゴリ（RoleManagement / GroupManagement / ResourceManagement） |
| `ActivityDisplayName` | 複数選択 | 操作種別での絞り込み |
| `ResourceName` | 複数選択 | Azure リソース（ResourceManagement 用） |
| `Group` | 複数選択 | Entra グループ（GroupManagement 用） |
| `Result` | 複数選択 | 結果（success / failure） |

## フィールドの意味（監査履歴テーブル）

| 表示ラベル | 内容 |
|---|---|
| いつ | `ActivityDateTime` |
| 誰が | 実行者。`request approved` では**承認者**（`InitiatedBy` / `Identity`） |
| 対象Entraロール / 対象Entraグループ / 対象Azureリソース | カテゴリごとの対象オブジェクト |
| 承認対象ユーザー | `request approved` イベントで承認されたユーザー（表示名 + UPN） |
| 申請内容 | `ActivityDisplayName` |
| 結果 | `Result`（成功/失敗アイコン付き） |
| コメント | 申請理由（`AdditionalDetails` の Justification） |

## リポジトリ構成

| ファイル | 説明 |
|---|---|
| `PIM-Workbook.galleryTemplate.v1.5.json` | ブックのギャラリーテンプレート（ポータル貼り付け用） |
| `PIM-Workbook.deploy.json` | ARM デプロイテンプレート（ブックをリソースとして展開） |
| `PIM-Workbook.deploy.parameters.json` | ARM デプロイ用パラメータ例 |
| `README.md` | 本ドキュメント |

> **注意:** 顧客テナント ID・ユーザー名・サブスクリプション ID などの環境固有情報や機密情報はリポジトリにコミットしないでください（[.gitignore](.gitignore) で除外設定済み）。

## 版歴

| 版数 | 変更内容 |
|---|---|
| 1.0 | 初版作成（PIM 監査履歴の可視化） |
| 1.1 | GroupManagement 分析（円グラフ4種）と絞り込みパラメータを追加 |
| 1.2 | 監査履歴を Category 別に3つの表へ再設計、列を日本語ラベル化 |
| 1.3 | Category を単一選択プルダウン化し、選択カテゴリの表のみ表示 |
| 1.4 | `request approved` イベントに「承認対象ユーザー」列を追加 |
| 1.5 | RoleManagement / ResourceManagement Analysis を追加、分析グラフを Category で切替表示 |

## ライセンス

社内利用を想定。公開時はライセンスを別途設定してください。
