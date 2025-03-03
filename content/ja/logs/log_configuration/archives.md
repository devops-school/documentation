---
aliases:
- /ja/logs/s3/
- /ja/logs/gcs/
- /ja/logs/archives/s3/
- /ja/logs/archives/gcs/
- /ja/logs/archives/gcp/
- /ja/logs/archives/
description: 収集されたログをすべて長期的なストレージへ転送します。
further_reading:
- link: /logs/explorer/
  tag: ドキュメント
  text: ログエクスプローラー
- link: /logs/logging_without_limits/
  tag: ドキュメント
  text: Logging without Limits*
kind: documentation
title: ログアーカイブ
---

## 概要

Datadog アカウントを構成して、独自のクラウドストレージシステムへ収集されたすべてのログ（[インデックス化][1]の有無にかかわらず）を転送します。ストレージに最適化されたアーカイブにログを長期間保管し、コンプライアンス要件を満たすことができると同時に、アドホック調査のための監査適合性を[リハイドレート][2]で維持できます。

{{< img src="logs/archives/log_archives_s3_multiple.png" alt="アーカイブページ"  style="width:75%;">}}

このガイドでは、クラウドホスト型ストレージバケットに収集したログを転送するためのアーカイブの設定方法を説明します。

1. まだの場合は、お使いのクラウドプロバイダーと Datadogの[インテグレーション](#set-up-an-integration)を設定してください
2. [ストレージバケット](#create-a-storage-bucket)を作成します
3. そのアーカイブへの `read` および `write` [許可](#set-permissions)を設定します
4. アーカイブへ、およびアーカイブから[ログをルーティング](#route-your-logs-to-a-bucket)します
5. 暗号化、ストレージクラス、タグなどの[詳細設定](#advanced-settings)を構成します
6. Datadog で検出される可能性のある構成ミスがないか、設定を[確認](#validation)します

**注:** [logs_write_archive 権限][3]のある Datadog ユーザーだけがログアーカイブ構成を作成、変更、または削除できます。

## ログアーカイブの構成

### インテグレーションを設定

{{< tabs >}}
{{% tab "AWS S3" %}}

{{< site-region region="gov" >}}
<div class="alert alert-warning">AWS Role Delegation は、Datadog for Government site でサポートされていません。アクセスキーを使用する必要があります。</div>
{{< /site-region >}}

まだ構成されていない場合は、S3 バケットを保持する AWS アカウントの [AWS インテグレーション][1]をセットアップします。

* 一般的なケースでは、これには、Datadog が AWS S3 との統合に使用できるロールの作成が含まれます。
* 特に AWS GovCloud または China アカウントの場合は、ロール委任の代わりにアクセスキーを使用します。

[1]: /ja/integrations/amazon_web_services/?tab=automaticcloudformation#setup
{{% /tab %}}
{{% tab "Azure Storage" %}}

新しいストレージアカウントのあるサブスクリプション内で [Azure インテグレーション][1]をセットアップしていない場合、セットアップします。これには、[Datadog が統合に使用できるアプリ登録の作成][2]も含まれます。

[1]: https://app.datadoghq.com/account/settings#integrations/azure
[2]: /ja/integrations/azure/?tab=azurecliv20#integrating-through-the-azure-portal
{{% /tab %}}

{{% tab "Google Cloud Storage" %}}

GCS ストレージバケットを持つプロジェクト用の [GCP インテグレーション][1]をセットアップしていない場合、セットアップします。これには [Datadog が統合に使用できる GCP サービスアカウントの作成][2] も含まれます。

[1]: https://app.datadoghq.com/account/settings#integrations/google-cloud-platform
[2]: /ja/integrations/google_cloud_platform/?tab=datadogussite#setup
{{% /tab %}}
{{< /tabs >}}

### ストレージバケットを作成

{{< tabs >}}
{{% tab "AWS S3" %}}

[AWS コンソール][1]にアクセスし、アーカイブを転送する [S3 バケットを作成][2]します。

**注:**

- バケットは一般ユーザーが読み取り可能になるよう設定してください。
- まれに最後のデータを書き換える必要があるため、[オブジェクトロック][3]を設定しないでください (通常はタイムアウト)。

[1]: https://s3.console.aws.amazon.com/s3
[2]: https://docs.aws.amazon.com/AmazonS3/latest/user-guide/create-bucket.html
[3]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock-overview.html
{{% /tab %}}

{{% tab "Azure Storage" %}}

* [Azure ポータル][1]にアクセスし、アーカイブを転送する[ストレージアカウントを作成][2]します。ストレージアカウントの名前と種類を指定し、**hot** アクセス層を選択します。
* そのストレージアカウントに **container** サービスを作成します。Datadog アーカイブページに追加する必要があるため、コンテナ名をメモしてください。

**注:** まれに最後のデータを書き換える必要があるため、[不変性ポリシー][3]を設定しないでください (通常はタイムアウト)。

[1]: https://portal.azure.com/#blade/HubsExtension/BrowseResource/resourceType/Microsoft.Storage%2FStorageAccounts
[2]: https://docs.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-portal
[3]: https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-immutability-policies-manage
{{% /tab %}}

{{% tab "Google Cloud Storage" %}}

[GCP アカウント][1]にアクセスし、アーカイブを転送する [GCS バケットを作成][2]します。「**Choose how to control access to objects**」で、「**Set object-level and bucket-level permissions**」を選択します。

**注:** まれに最後のデータを書き換える必要があるため、[保持ポリシー][3]を追加しないでください (通常はタイムアウト)。

[1]: https://console.cloud.google.com/storage
[2]: https://cloud.google.com/storage/docs/quickstart-console
[3]: https://cloud.google.com/storage/docs/bucket-lock
{{% /tab %}}
{{< /tabs >}}

### アクセス許可を設定

{{< tabs >}}
{{% tab "AWS S3" %}}

1. 次の 2 つのアクセス許可ステートメントを持つ[ポリシーを作成][1]します。

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "DatadogUploadAndRehydrateLogArchives",
         "Effect": "Allow",
         "Action": ["s3:PutObject", "s3:GetObject"],
         "Resource": [
           "arn:aws:s3:::<MY_BUCKET_NAME_1_/_MY_OPTIONAL_BUCKET_PATH_1>/*",
           "arn:aws:s3:::<MY_BUCKET_NAME_2_/_MY_OPTIONAL_BUCKET_PATH_2>/*"
         ]
       },
       {
         "Sid": "DatadogRehydrateLogArchivesListBucket",
         "Effect": "Allow",
         "Action": "s3:ListBucket",
         "Resource": [
           "arn:aws:s3:::<MY_BUCKET_NAME_1>",
           "arn:aws:s3:::<MY_BUCKET_NAME_2>"
         ]
       }
     ]
   }
   ```
     * `GetObject` および `ListBucket` アクセス許可は、[アーカイブからリハイドレート][2]を可能にします。
     * アーカイブのアップロードには、`PutObject` アクセス許可で十分です。

2. バケット名を編集します。
3. オプションで、ログアーカイブを含むパスを指定します。
4. Datadog のインテグレーションロールに新しいポリシーをアタッチします。
   a. AWS IAM コンソールで **Roles** に移動します。
   b. Datadog インテグレーションで使用されるロールを見つけます。デフォルトでは、 **DatadogIntegrationRole** という名前になっていますが、組織で名前を変更した場合は、名前が異なる場合があります。ロール名をクリックすると、ロールのサマリーページが表示されます。
    c. **Add permissions**、**Attach policies** の順にクリックします。
   d. 上記で作成したポリシーの名称を入力します。
   e. **Attach policies** をクリックします。

**注**: `s3:PutObject` と `s3:GetObject` アクションのリソース値は `/*` で終わっていることを確認してください。これらの権限はバケット内のオブジェクトに適用されるからです。

[1]: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_create-console.html
[2]: /ja/logs/archives/rehydrating/

{{% /tab %}}
{{% tab "Azure Storage" %}}

* Datadog アプリに、ストレージアカウントへ書き込み、ここからリハイドレートするための許可を与えます。
* [ストレージアカウントのページ][1]でストレージアカウントを選択し、**Access Control (IAM)** で **Add -> Add Role Assignment** を選択します。
* Role に **Storage Blob Data Contributor** を入力し、Azure と統合するために作成した Datadog アプリを選択して、保存します。

{{< img src="logs/archives/logs_azure_archive_permissions.png" alt="Storage Blob Data Contributor ロールを Datadog アプリに追加します。" style="width:75%;">}}

[1]: https://portal.azure.com/#blade/HubsExtension/BrowseResource/resourceType/Microsoft.Storage%2FStorageAccounts
{{% /tab %}}
{{% tab "Google Cloud Storage" %}}

Datadog GCP サービスアカウントに、バケットへアーカイブを書き込むための許可を与えます。

* 新しいサービスアカウントを作成する場合は、[GCP Credentials ページ][1]でこれを実行できます。
* 既存のサービスアカウントを更新する場合は、[GCP IAM Admin ページ][2]で実行できます。

**Storage** の下に **Storage Object Admin** というロールを追加します。

  {{< img src="logs/archives/gcp_role_storage_object_admin.png" alt="Datadog GCP サービスアカウントに Storage Object Admin ロールを追加。" style="width:75%;">}}

[1]: https://console.cloud.google.com/apis/credentials
[2]: https://console.cloud.google.com/iam-admin/iam
{{% /tab %}}
{{< /tabs >}}

### ログをバケットにルーティング

Datadog アプリの[アーカイブページ][4]に移動し、下にある **Add a new archive** オプションを選択します。

**注:** 
* [logs_write_archive 権限][3]のある Datadog ユーザーだけがこの手順と次の手順を完了させることができます。
* Azure Blob Storage へのログのアーカイブには、App Registration が必要です。[Azure インテグレーションページ][5]の手順を参照し、ドキュメントページの右側にある「サイト」を「US」に設定してください。アーカイブ目的で作成された App Registration は、"Storage Blob Data Contributor" ロールのみが必要です。ストレージバケットが Datadog Resource を通じて監視されているサブスクリプションにある場合、App Registration が冗長である旨の警告が表示されます。この警告は無視することができます。

{{< tabs >}}
{{% tab "AWS S3" %}}

S3 バケットに適した AWS アカウントとロールの組み合わせを選択します。

バケット名を入力します。**任意**: ログアーカイブのすべてのコンテンツにプレフィックスディレクトリを入力します。

{{< img src="logs/archives/logs_archive_aws_setup.png" alt="Datadog で S3 バケットの情報を設定"  style="width:75%;">}}

{{% /tab %}}
{{% tab "Azure Storage" %}}

**Azure Storage** アーカイブタイプを選択し、ストレージアカウントで Storage Blob Data Contributor ロールのある Datadog アプリ用の Azure テナントとクライアントを選択します。

ストレージアカウント名とアーカイブのコンテナ名を入力します。**任意**: ログアーカイブのすべてのコンテンツにプレフィックスディレクトリを入力します。

{{< img src="logs/archives/logs_archive_azure_setup.png" alt="Datadog で Azure ストレージアカウントの情報を設定"  style="width:75%;">}}


{{% /tab %}}
{{% tab "Google Cloud Storage" %}}

**GCS** のアーカイブタイプを選択し、ストレージバケットに書き込む権限を持つ GCS サービスアカウントを選択します。バケット名を入力します。

バケット名を入力します。**任意**: ログアーカイブのすべてのコンテンツにプレフィックスディレクトリを入力します。

{{< img src="logs/archives/logs_archive_gcp_setup.png" alt="Datadog で Azure ストレージアカウントの情報を設定"  style="width:75%;">}}

{{% /tab %}}
{{< /tabs >}}

### 高度な設定

#### Datadog のアクセス許可

デフォルト:

* すべての Datadog 管理者ユーザーは、アーカイブの作成、編集、並べ替えができます ([複数アーカイブの構成](#multiple-archives)を参照)
* すべての Datadog 監理者および標準ユーザーは、アーカイブからリハイドレーションできます
* Datadog の読み取り専用ユーザーを含むすべてのユーザーは、リハイドレーションされたログにアクセスできます

オプションで、コンフィギュレーションステップを使用し、アーカイブにロールを割り当て、以下を実行できるユーザーを設定できます。

* アーカイブのコンフィギュレーションを編集（[logs_write_archive][6] 権限を参照）。
* アーカイブからのリハイドレーション（[logs_read_archives][7] および [logs_write_historical_view][8] を参照）。
* レガシーを使用する場合に、リハイドレートされたログへアクセス（[read_index_data 権限][9]を参照）。

{{< img src="logs/archives/archive_restriction.png" alt="アーカイブおよびリハイドレート済みログへのアクセスを制限"  style="width:75%;">}}

#### Datadog タグ

以下のためにこのオプションの構成ステップを使用します。

* アーカイブ内のすべてのログタグを含める (デフォルトでは、すべての新規アーカイブに有効化されています)。**注**: 結果のアーカイブサイズが増大します。
* リハイドレート済みのログのタグを制限クエリポリシーに追加（[logs_read_data][10] 権限を参照）。

{{< img src="logs/archives/tags_in_out.png" alt="アーカイブタグの構成"  style="width:75%;">}}

#### 最大スキャンサイズを定義する

このオプションの構成ステップを使用して、ログアーカイブでリハイドレートのためにスキャンできるログデータの最大量 (GB 単位) を定義します。

最大スキャンサイズが定義されているアーカイブの場合、すべてのユーザーは、リハイドレートを開始する前にスキャンサイズを推定する必要があります。推定されたスキャンサイズがそのアーカイブで許可されているものより大きい場合、ユーザーはリハイドレートを要求する時間範囲を狭めなければなりません。時間範囲を減らすと、スキャンサイズが小さくなり、ユーザーがリハイドレートを開始できるようになります。

{{< img src="logs/archives/max_scan_size.png" alt="アーカイブの最大スキャンサイズを設定する" style="width:75%;">}}

#### ストレージクラス

{{< tabs >}}
{{% tab "AWS S3" %}}

[S3 バケットにライフサイクルコンフィギュレーションを設定][1]して、ログアーカイブを最適なストレージクラスに自動的に移行できます。

[リハイドレート][2]は、Glacier および Glacier Deep Archive を除くすべてのストレージクラスをサポートしています (Glacier Instant Retrieval は例外です)。Glacier または Glacier Deep Archive ストレージクラスのアーカイブからリハイドレートする場合は、まずそれらを別のストレージクラスに移動する必要があります。

[1]: https://docs.aws.amazon.com/AmazonS3/latest/dev/how-to-set-lifecycle-configuration-intro.html
[2]: /ja/logs/archives/rehydrating/
{{% /tab %}}

{{< /tabs >}}

#### サーバー側の暗号化 (SSE)

{{< tabs >}}
{{% tab "AWS S3" %}}

##### SSE-S3

サーバー側の暗号化を S3 ログアーカイブに追加するもっとも簡単な方法は、S3 のネイティブサーバーサイド暗号化 [SSE-S3][1] を利用することです。

有効化するには S3 バケットの **Properties** タブに移動し、**Default Encryption** を選択します。`AES-256` オプションを選択して、**Save** を選択します。

{{< img src="logs/archives/log_archives_s3_encryption.png" alt="AES-256 オプションを選択し、保存します。"  style="width:75%;">}}

##### SSE-KMS

また、Datadog は CMK を利用した [AWS KMS][2] からのサーバーサイド暗号化もサポートしています。有効化するには次の手順に従ってください。

1. CMK の作成
2. CMK に付随する CMK ポリシーに以下のコンテンツを添加して、AWS アカウント番号と Datadog IAM ロール名を適切なものに置き換えます。

```
{
    "Id": "key-consolepolicy-3",
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Enable IAM User Permissions",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<MY_AWS_ACCOUNT_NUMBER>:root"
            },
            "Action": "kms:*",
            "Resource": "*"
        },
        {
            "Sid": "Allow use of the key",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<MY_AWS_ACCOUNT_NUMBER>:role/<MY_DATADOG_IAM_ROLE_NAME>"
            },
            "Action": [
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:ReEncrypt*",
                "kms:GenerateDataKey*",
                "kms:DescribeKey"
            ],
            "Resource": "*"
        },
        {
            "Sid": "Allow attachment of persistent resources",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<MY_AWS_ACCOUNT_NUMBER>:role/<MY_DATADOG_IAM_ROLE_NAME>"
            },
            "Action": [
                "kms:CreateGrant",
                "kms:ListGrants",
                "kms:RevokeGrant"
            ],
            "Resource": "*",
            "Condition": {
                "Bool": {
                    "kms:GrantIsForAWSResource": "true"
                }
            }
        }
    ]
}
```

3. S3 バケットの **Properties** タブに移動し、**Default Encryption** を選択します。"AWS-KMS" オプション、CMK ARN の順に選択して保存します。


[1]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/default-bucket-encryption.html
[2]: https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingKMSEncryption.html
{{% /tab %}}

{{< /tabs >}}

### 検証

Datadog アカウントでアーカイブ設定が正常に構成された時点から、処理パイプラインは Datadog が収集したすべてのログを加工し始めます。その後アーカイブに転送されます。

ただし、アーカイブ構成を作成または更新してから次にアーカイブのアップロードが試行されるまで、数分かかることがあります。ログは15分ごとにアーカイブにアップロードされるので、**15 分待ってストレージバケットをチェックし**、Datadog アカウントからアーカイブが正常にアップロードされたことを確認してください。その後、アーカイブが依然として保留中の場合は、包含フィルターをチェックしクエリが有効であることと、[live tail][11] でログイベントが一致することを確認します。

Datadog でコンフィギュレーションの問題が検出された場合、該当するアーカイブがコンフィギュレーションページでハイライトされます。エラーアイコンをチェックして、問題を修正するためにとるべきアクションを確認します。

{{< img src="logs/archives/archive_validation.png" alt="アーカイブが適切にセットアップされたことを確認。"  style="width:75%;">}}

## 複数のアーカイブ

複数のアーカイブが定義された場合、ログはフィルターに基づく最初のアーカイブに保存されるため、アーカイブの順番は慎重に決定する必要があります。

たとえば、最初に `env:prod` タグで絞り込まれるアーカイブを作成し、次にフィルターなし (`*` と同等) でアーカイブを作成した場合、すべてのプロダクションログは一方のストレージバケット/パスに転送され、その他のログはもう一方のアーカイブに転送されます。

{{< img src="logs/archives/log_archives_s3_multiple.png" alt="ログは、フィルターに一致する最初のアーカイブに保存されます。"  style="width:75%;">}}

## アーカイブの形式

Datadog がストレージバケットに転送するログアーカイブは、圧縮 JSON 形式（`.json.gz`）になっています。アーカイブは、指定したプレフィックスの下の (指定しなかった場合は `/`)、アーカイブファイルが生成された日時を示すディレクトリ構造に保存されます。

```
/my/bucket/prefix/dt=20180515/hour=14/archive_143201.1234.7dq1a9mnSya3bFotoErfxl.json.gz
/my/bucket/prefix/dt=<YYYYMMDD>/hour=<HH>/archive_<HHmmss.SSSS>.<DATADOG_ID>.json.gz
```

このディレクトリ構造により、過去のログアーカイブを日付に基づいてクエリする処理が簡略化されます。

圧縮 JSON ファイル内の各イベントは、以下の形式で内容が表されます。

```json
{
    "_id": "123456789abcdefg",
    "date": "2018-05-15T14:31:16.003Z",
    "host": "i-12345abced6789efg",
    "source": "source_name",
    "service": "service_name",
    "status": "status_level",
    "message": "2018-05-15T14:31:16.003Z INFO rid='acb-123' status=403 method=PUT",
    "attributes": { "rid": "abc-123", "http": { "status_code": 403, "method": "PUT" } },
    "tags": [ "env:prod", "team:acme" ]
}
```

## その他の参考資料

{{< whatsnext desc="次に、Datadog からアーカイブされたログコンテンツにアクセスする方法を説明します。" >}}
    {{< nextlink href="/logs/archives/rehydrating" >}}<u>アーカイブからリハイドレート</u>: ログイベントをアーカイブから取得し、Datadog の Log Explorer に戻します。{{< /nextlink >}}
{{< /whatsnext >}}

{{< partial name="whats-next/whats-next.html" >}}

<br>
*Logging without Limits は Datadog, Inc. の商標です。

[1]: /ja/logs/indexes/#exclusion-filters
[2]: /ja/logs/archives/rehydrating/
[3]: /ja/account_management/rbac/permissions/?tab=ui#logs_write_archives
[4]: https://app.datadoghq.com/logs/pipelines/archives
[5]: /ja/integrations/azure/
[6]: /ja/account_management/rbac/permissions#logs_write_archives
[7]: /ja/account_management/rbac/permissions#logs_read_archives
[8]: /ja/account_management/rbac/permissions#logs_write_historical_view
[9]: /ja/account_management/rbac/permissions#logs_read_index_data
[10]: /ja/account_management/rbac/permissions#logs_read_data
[11]: /ja/logs/explorer/live_tail/