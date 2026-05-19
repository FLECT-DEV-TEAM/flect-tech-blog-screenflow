# リリース手順書

技術ブログ「Salesforce画面フローの設計：見落としがちなユーザー操作を想定する」のデモ実装をSalesforce組織にリリースするための手順書です。

## 1. リリース内容

### 1.1. 含まれるメタデータ

| 種別 | API名 | 概要 |
|---|---|---|
| CustomField | `Account.Status__c` | ステータス（Picklist: Prospect/Negotiating/Dormant/Lost） |
| CustomField | `Account.LastContactDate__c` | 最終接触日（Date） |
| CustomField | `Account.PreviousContactDate__c` | 前回接触日（Date） |
| QuickAction | `Account.Bad_Create_Opportunity` | 「商談作成（悪）」ボタン |
| QuickAction | `Account.Good_Create_Opportunity` | 「商談作成」ボタン |
| Flow | `Bad_Account_Opportunity_Creation` | 悪い実装の画面フロー |
| Flow | `Good_Account_Opportunity_Creation` | 正しい実装の画面フロー |
| Layout | `Account-Blog Demo Layout` | QuickActionを配置したAccountページレイアウト |
| PermissionSet | `Account_Opportunity_Editor` | 想定ユーザー用権限セット（Account編集可・Opportunity作成可・両フロー実行可） |
| PermissionSet | `Account_Editor_No_Opportunity` | 想定外ユーザーシナリオ用権限セット（Opportunity作成不可） |

### 1.2. 前提条件

- Salesforce エディション：Enterprise / Unlimited / Developer Edition
- APIバージョン：63.0 以上（Roll Back Records 要素のサポートのため v53.0 以降が必須）
- Salesforce CLI（`sf`コマンド）がローカルにインストール済み
- 対象組織への認証済み接続が存在すること

### 1.3. 依存関係

メタデータ間の依存関係上、デプロイは package.xml に列挙された順序ではなく Salesforce が自動的に解決します。`sf project deploy` を利用すれば手動でのデプロイ順制御は不要です。

ただし手動でMetadata API経由でデプロイする場合、以下の順序を推奨します：

1. CustomField（フロー・レイアウト・権限セットが参照）
2. QuickAction（レイアウトが参照、フローを参照）
3. Flow（QuickActionから参照される）
4. Layout（QuickActionを参照）
5. PermissionSet（フローを参照）

QuickActionとFlowは相互参照に近い構造ですが、Salesforceは内部的に2-passデプロイを行うため `sf project deploy` で問題なくデプロイ可能です。

## 2. デプロイ手順

### 2.1. 事前準備

#### 2.1.1. 認証

対象組織に認証していない場合は以下のコマンドで認証します。

```pwsh
sf org login web --alias my-target-org
```

#### 2.1.2. デプロイ前の差分確認（推奨）

```pwsh
sf project deploy preview --manifest release/package.xml --target-org my-target-org
```

### 2.2. デプロイ実行

プロジェクトルートで以下を実行します。

```pwsh
sf project deploy start --manifest release/package.xml --target-org my-target-org --test-level NoTestRun
```

Apexクラスを含まないリリースのため `--test-level NoTestRun` を指定しています（本番組織の場合は組織のテスト要件に従ってください）。

本番組織へのデプロイの場合：

```pwsh
sf project deploy start --manifest release/package.xml --target-org my-prod-org --test-level RunLocalTests
```

### 2.3. デプロイ完了確認

```pwsh
sf project deploy report --target-org my-target-org
```

## 3. デプロイ後の手動設定

メタデータデプロイだけでは反映されない設定が3点あります。

### 3.1. ページレイアウトの割り当て

`Account-Blog Demo Layout` をユーザーに割り当てます。

1. Setup を開く
2. Object Manager → Account → Page Layouts
3. **Page Layout Assignment** をクリック
4. **Edit Assignment** をクリック
5. レイアウトを割り当てたいProfile（例：System Administrator）を選択
6. 「Page Layout To Use」プルダウンで **Blog Demo Layout** を選択
7. **Save** をクリック

> このレイアウトはデモ用であり、組織の標準Accountレイアウトを置き換えません。デモ終了後は元のレイアウトに戻すか、レイアウト自体を削除してください。

### 3.2. 権限セットの割り当て

本リリースには2つの権限セットが含まれます。シナリオに応じて適切な方を割り当ててください。

#### 3.2.1. 想定ユーザー用：`Account_Opportunity_Editor`

フローが正常完了する「想定通りのユーザー」をシミュレートするための権限セット。Account編集可・Opportunity作成可・両フロー実行可。

1. Setup → Permission Sets → **Account_Opportunity_Editor** を開く
2. **Manage Assignments** → **Add Assignments**
3. 想定ユーザー（営業担当ロール想定）を選択し **Assign**

> System Administrator プロファイルのユーザーは既に全権限を持つため割り当て不要です。Standard User などの最小プロファイルでテストする場合に使います。

#### 3.2.2. 想定外ユーザー用：`Account_Editor_No_Opportunity`

「想定外ユーザーにはそもそも正しいフローを実行させない」を実証するための権限セット。Account編集可・Opportunity作成不可。正しいフロー（Good_Account_Opportunity_Creation）の実行権限は付与せず、悪いフロー（Bad_Account_Opportunity_Creation）のみ実行可。

1. Setup → Permission Sets → **Account_Editor_No_Opportunity** を開く
2. **Manage Assignments** → **Add Assignments**
3. テスト用ユーザーを選択し **Assign**

> このユーザーで「商談作成」（Good）を起動するとフローアクセス権限がないためフロー自体が起動しません。一方「商談作成（悪）」（Bad）は実行可能だが、Create Records 時点で Opportunity 作成権限エラーで失敗し、取引先側だけ更新が残る挙動（=悪いフローの問題）を再現できます。両権限セットを同時に割り当てると Opportunity 作成権限が「許可」として合算されてしまうため、想定外シナリオを試す際は `Account_Opportunity_Editor` を外してください。

### 3.3. フローの起動有効化確認

通常 `<status>Active</status>` を指定したフローはデプロイ時に有効化されます。
組織のフロー設定で「画面フローのインタビュー作成権限」がProfileに付与されていることを確認してください。

## 4. 動作確認

### 4.1. 基本動作確認

1. デモ用Account（例：Status__c=新規、LastContactDate__c=昨日の日付）を1件作成
2. レコードページを開き、Highlights Panelに「商談作成」「商談作成（悪）」が表示されることを確認
3. 「商談作成」（正しいフロー）を起動し、入力→確認→完了 まで進めることを確認
4. 取引先のStatus__c=「商談中」、LastContactDate__c=今日、PreviousContactDate__c=昨日 になっていることを確認
5. 商談が1件作成されていることを確認

### 4.2. 悪いフローの問題再現

ブログ本文と仕様書（`blog-demo-spec.md` セクション2.5 / 6.2）の通り、ブラウザバック・途中離脱・想定外実行の3シナリオで挙動を確認します。

### 4.3. 正しいフローの確認

想定外ユーザー（権限セット `Account_Editor_No_Opportunity` のみ割り当て済み）で取引先レコードページを開き、以下を確認します。

1. 「商談作成」（Good）を起動しようとすると、フローへのアクセス権がないため起動できないこと
2. 比較として「商談作成（悪）」（Bad）を起動した場合は権限エラーで失敗し、取引先側だけ更新が残ること（=悪いフローの問題）

正しいフローは「権限で実行ユーザーを絞ることで、そもそも想定外ユーザーには触らせない」という方針を採っており、Fault Path / Roll Back Records による事後リカバリは行いません。

## 5. ロールバック手順

デプロイをロールバックする必要が生じた場合、Salesforceには標準のロールバック機構がないため、以下の手順で手動削除します。

### 5.1. destructiveChanges.xml の作成

`release/destructiveChanges.xml` として以下を作成します：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>Bad_Account_Opportunity_Creation</members>
        <members>Good_Account_Opportunity_Creation</members>
        <name>Flow</name>
    </types>
    <types>
        <members>Account.Bad_Create_Opportunity</members>
        <members>Account.Good_Create_Opportunity</members>
        <name>QuickAction</name>
    </types>
    <types>
        <members>Account-Blog Demo Layout</members>
        <name>Layout</name>
    </types>
    <types>
        <members>Account_Editor_No_Opportunity</members>
        <members>Account_Opportunity_Editor</members>
        <name>PermissionSet</name>
    </types>
    <types>
        <members>Account.LastContactDate__c</members>
        <members>Account.PreviousContactDate__c</members>
        <members>Account.Status__c</members>
        <name>CustomField</name>
    </types>
    <version>63.0</version>
</Package>
```

加えて、destructiveなパッケージにはダミーの `package.xml`（types要素を持たない）も必要です。

### 5.2. ロールバック実行

事前にフローを非アクティブ化してから削除します（アクティブなフローは削除不可）。

1. Setup → Flows で両フローを **Deactivate**
2. 以下を実行：

```pwsh
sf project deploy start --pre-destructive-changes release/destructiveChanges.xml --manifest release/package-empty.xml --target-org my-target-org
```

### 5.3. データの取り扱い

カスタム項目を削除すると、その項目に保存されていたデータも失われます。本リリースで追加した3項目（Status__c、LastContactDate__c、PreviousContactDate__c）に重要なデータが入っている場合は、削除前にエクスポートしてください。

## 6. リリース時の注意点

- **Lightning Runtime限定**：Classic Runtime / Experience Cloud での挙動は本実装の保証範囲外。
- **実行コンテキスト**：両フローとも `DefaultMode`（ユーザーコンテキスト）で動作。System Mode への変更が必要な場合はフローXMLの `<runInMode>` を編集。
- **本番組織への影響**：`Account-Blog Demo Layout` はデモ専用です。本番では使用しないか、適切な命名規約に従って変更してください。
- **QuickAction のソース配置**：SFDX source format では `quickActions/<Object>.<Name>.quickAction-meta.xml` の配置が必要。`objects/<Object>/quickActions/` 配下に置くと SFDX が認識せずデプロイから漏れる。
- **Flow 型 QuickAction**：`<quickActionLayout>` 要素を含めるとデプロイエラー（"QuickActions of type Flow cannot have a layout"）。
- **Page Layout 内の QuickAction 参照**：同一オブジェクトの QuickAction でも `<actionName>Account.Good_Create_Opportunity</actionName>` のようにフルプレフィックスで指定する必要がある。プレフィックスなしだとデプロイ時に "no QuickAction named X found" となる。
