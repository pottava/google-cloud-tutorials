# Slurm と Lustre による HPC 環境の構築（共有 VPC 版）

<walkthrough-watcher-constant key="region" value="asia-northeast1"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="zone" value="asia-northeast1-c"></walkthrough-watcher-constant>

## 始めましょう

[共有 VPC](https://cloud.google.com/vpc/docs/shared-vpc?hl=ja) 上に Lustre による分散ストレージと Slurm ベースの計算クラスタを構築するための手順です。

**所要時間**: 約 60 分

**前提条件**:

- [組織が作成](https://cloud.google.com/resource-manager/docs/creating-managing-organization?hl=ja) してある
- 共有 VPC / ストレージ / 計算クラスタのための 3 つのプロジェクトが作成してある
- 組織管理者・組織ポリシー管理者・各プロジェクトのオーナー権限をもつアカウントでログインしている

**[開始]** ボタンをクリックして次のステップに進みます。

## プロジェクトの設定

この手順の中で実際にリソースを構築する対象のプロジェクトを選択してください。

<walkthrough-project-billing-setup permissions="compute.googleapis.com,iam.roles.create,iam.roles.delete,iam.roles.get,iam.roles.list,iam.roles.undelete,iam.roles.update,resourcemanager.organizations.get,resourcemanager.organizations.getIamPolicy,resourcemanager.projects.get,resourcemanager.projects.getIamPolicy,resourcemanager.projects.list"></walkthrough-project-billing-setup>

## 権限付与

Compute Shared VPC 管理者、Project IAM 管理者権限を付与します。

```bash
org_id="$( gcloud organizations list --format 'value(name)' )"
account="$( gcloud auth list --format 'value(account)' )"
gcloud organizations add-iam-policy-binding "${org_id}" \
    --member="user:${account}" \
    --role="roles/compute.xpnAdmin"
gcloud organizations add-iam-policy-binding "${org_id}" \
    --member="user:${account}" \
    --role="roles/resourcemanager.projectIamAdmin"
```

## 共有 VPC の有効化

共有 VPC のプロジェクト ID を設定します。

```bash
export host_project_id='?'
```

共有 VPC を有効化します。

```bash
gcloud --project "${host_project_id}" services enable compute.googleapis.com
gcloud compute shared-vpc enable "${host_project_id}"
```

## サービスプロジェクトの接続

ストレージ、計算クラスタのプロジェクト ID をそれぞれ設定します。

```bash
export storage_project_id='?'
export compute_project_id='?'
```

サービスプロジェクトとして、それぞれのプロジェクトを共有 VPC プロジェクトに接続します。

```bash
gcloud --project "${storage_project_id}" services enable compute.googleapis.com
gcloud compute shared-vpc associated-projects add "${storage_project_id}" \
    --host-project "${host_project_id}"
gcloud --project "${compute_project_id}" services enable compute.googleapis.com
gcloud compute shared-vpc associated-projects add "${compute_project_id}" \
    --host-project "${host_project_id}"
```

接続状態を確認します。

```bash
gcloud compute shared-vpc list-associated-resources "${host_project_id}"
```

## 必要なユーザにサブネットへのアクセスを許可

各サービス プロジェクト側でインスタンス作成などを実施するユーザーに対し、ホスト プロジェクトのサブネットを扱う権限とインスタンスに対し [IAP トンネリング](https://cloud.google.com/iap/docs/using-tcp-forwarding?hl=ja) する権限を付与します（後半、より安全な SSH 接続をするために利用します）。

```bash
account="$( gcloud auth list --format 'value(account)' )"
gcloud projects add-iam-policy-binding "${host_project_id}" \
    --member "user:${account}" \
    --role "roles/compute.networkUser"
gcloud projects add-iam-policy-binding "${host_project_id}" \
    --member "user:${account}" \
    --role "roles/compute.networkUser"
gcloud projects add-iam-policy-binding "${compute_project_id}" \
    --member="user:${account}" \
    --role=roles/iap.tunnelResourceAccessor
gcloud projects add-iam-policy-binding "${storage_project_id}" \
    --member="user:${account}" \
    --role=roles/iap.tunnelResourceAccessor
```

## サービスアカウントにサブネットへのアクセスを許可

[Google API サービス エージェント](https://cloud.google.com/iam/docs/service-accounts?hl=ja#google-managed)、計算クラスタの VM にアタッチする予定のサービスアカウントに対し、ホスト プロジェクトのサブネットを扱う権限を付与します。

```bash
gcloud projects add-iam-policy-binding "${host_project_id}" \
    --member "serviceAccount:$( gcloud projects list \
        --filter="${compute_project_id}" --format="value(PROJECT_NUMBER)" \
        )@cloudservices.gserviceaccount.com" \
    --role "roles/compute.networkUser"
gcloud projects add-iam-policy-binding "${host_project_id}" \
    --member "serviceAccount:$( gcloud projects list \
        --filter="${compute_project_id}" --format="value(PROJECT_NUMBER)" \
        )-compute@developer.gserviceaccount.com" \
    --role "roles/compute.networkUser"
gcloud projects add-iam-policy-binding "${host_project_id}" \
    --member "serviceAccount:$( gcloud projects list \
        --filter="${storage_project_id}" --format="value(PROJECT_NUMBER)" \
        )@cloudservices.gserviceaccount.com" \
    --role "roles/compute.networkUser"
```

## Cloud NAT の起動

**計算クラスタのノードにパブリックな静的 IP アドレスを付与しない場合**、手動で Cloud NAT を作成しておく必要があります。まず Cloud Router を作ります。

```bash
gcloud config set project "${host_project_id}"
gcloud config set compute/region {{region}}
gcloud compute routers create nat-router --network default
```

いま作成したルータを指定し、Cloud NAT を作成します。

```bash
gcloud compute routers nats create nat-config \
    --router=nat-router \
    --auto-allocate-nat-external-ips \
    --nat-all-subnet-ip-ranges \
    --enable-logging
```

## ネットワーク設定の変更

### SSH / RDP のデフォルト許可設定を無効化

デフォルトでは接続元 IP アドレスを縛っていません。以下のデフォルト設定は無効化しつつ、[Identity-Aware Proxy (IAP) を併用した SSH](https://cloud.google.com/iap/docs/using-tcp-forwarding?hl=ja) によるより安全な接続を利用します。IAP の利用する IP レンジからの接続も許可します。

```bash
gcloud config set project "${host_project_id}"
gcloud compute firewall-rules update default-allow-ssh --disabled
gcloud compute firewall-rules update default-allow-rdp --disabled
gcloud compute firewall-rules create allow-ssh-from-google-iap \
    --network default --allow tcp:22 --source-ranges 35.235.240.0/20
```

## Lustre クラスタの設定

利用可能なサブネットやその IP レンジを確認しましょう。

```text
ip_range=$( gcloud --project "${host_project_id}" compute networks subnets list-usable \
    --filter 'subnetwork~.*\/{{region}}\/subnetworks\/default' \
    --format 'value(ip_cidr_range)' ) && echo "${ip_range}"
```

スクリプトをダウンロードし、

```bash
git clone https://github.com/GoogleCloudPlatform/deploymentmanager-samples.git
cd deploymentmanager-samples/community/lustre/
```

設定値を編集します。

```text
cat << EOF >lustre.yaml
imports:
- path: lustre.jinja

resources:
- name: lustre
  type: lustre.jinja
  properties:
    ## Cluster Configuration
    cluster_name            : lustre
    zone                    : {{zone}}
    cidr                    : 10.146.0.0/20
    shared_vpc_host_proj    : ${host_project_id}
    vpc_net                 : default
    vpc_subnet              : default
    external_ips            : True

    ## Filesystem Configuration
    fs_name                 : lustre
    lustre_version          : latest-release
    e2fs_version            : latest

    ## MDS/MGS Configuration
    mds_node_count          : 1
    mds_ip_range_start      : 10.146.0.2
    mds_machine_type        : n1-standard-32
    mds_boot_disk_type      : pd-standard
    mds_boot_disk_size_gb   : 20
    mdt_disk_type           : pd-ssd
    mdt_disk_size_gb        : 1000

    ## OSS Configuration
    oss_node_count          : 4
    oss_ip_range_start      : 10.146.0.5
    oss_machine_type        : n1-standard-16
    oss_boot_disk_type      : pd-standard
    oss_boot_disk_size_gb   : 20
    ost_disk_type           : pd-ssd
    ost_disk_size_gb        : 1000
EOF
```

## Lustre クラスタの構築

設定内容を確認しましょう。一般的なテキストエディタで開いてもいいのですが、参考までに Cloud Shell Editor を起動してみましょう。

<walkthrough-editor-open-file filePath="~/cloudshell_open/google-cloud-tutorials/deploymentmanager-samples/community/lustre/lustre.yaml">Editor を起動</walkthrough-editor-open-file>

### クラスタを構築

[Deployment Manager](https://cloud.google.com/deployment-manager?hl=ja) という機能を使い、テンプレートの内容を正としたクラスタを構築します。

```bash
gcloud config set project "${storage_project_id}"
gcloud services enable deploymentmanager.googleapis.com
gcloud deployment-manager deployments create lustre --config lustre.yaml
```

### クラスタの初期化

初期化処理が完了するまで進行状況をトラッキングします。*‘Started Google Compute Engine Startup Scripts.’ と出力されるまで* お待ち下さい。n1-standard-32 で 15 分程度かかります。

```bash
gcloud compute ssh lustre-mds1 --zone {{zone}} --tunnel-through-iap \
    --command "sudo journalctl -fu google-startup-scripts.service"
```

## 動作確認と、計算のためのディレクトリ作成

管理サーバ（メタデータサーバ兼任）にログインし

```bash
gcloud compute ssh lustre-mds1 --zone {{zone}} --tunnel-through-iap
```

メタデータターゲットがローカルにマウントされていることを確認してみましょう。

```bash
mount | grep lustre
```

管理サーバ、計算クラスタにマウントするためのディレクトリを用意します。OSS の台数分ストライピングされるよう  -c -1  の指定もしておきます。

```bash
mkdir work
sudo mount -t lustre lustre-mds1:/lustre work && cd work
sudo mkdir -p apps users
sudo lfs setstripe -c -1 apps
sudo lfs setstripe -c -1 users
sudo lfs getstripe users
```

後片付けをして SSH 接続を閉じます。

```bash
cd .. && sudo umount work && sudo rm -rf work/
exit
```

## 計算クラスタの構築の準備 (1)

作業ディレクトリのルートにもどり、計算クラスタ用のプロジェクト ID に切り替えます。

```bash
cd ~/cloudshell_open/google-cloud-tutorials/
gcloud config set project "${compute_project_id}"
```

VM がより近接した配置となるよう、[プレイスメントポリシー](https://cloud.google.com/compute/docs/instances/define-instance-placement?hl=ja) を事前に作成します。

```bash
gcloud compute resource-policies create group-placement \
    --collocation=collocated --vm-count=22 hpc-cluster \
    --region {{region}}
```

## 計算クラスタの構築の準備 (2)

スクリプトをダウンロードし、作業ディレクトリを移動します。

```bash
git clone https://github.com/SchedMD/slurm-gcp.git && cd slurm-gcp
```

slurm-cluster.yaml を編集しましょう。

```text
cat << EOF >slurm-cluster.yaml
imports:
- path: slurm.jinja

resources:
- name: slurm-cluster
  type: slurm.jinja
  properties:
    cluster_name            : hpc
    zone                    : {{zone}}
    vpc_net                 : default
    vpc_subnet              : default

    shared_vpc_host_project : ${host_project_id}
    # suspend_time            : 300

    # ヘッドノード
    controller_machine_type : n1-standard-16
    controller_disk_size_gb : 20
    external_controller_ip  : True

    # ログインノード（本チュートリアルでは使用しません）
    login_machine_type        : n1-standard-2
    external_login_ips        : True
    login_node_count          : 0

    # 計算用 VM イメージ作成用
    compute_image_machine_type : n1-standard-2
    external_compute_ips       : True

    # バージョン
    slurm_version             : 19.05.8
    ompi_version              : v3.1.x

    # ファイルシステムのマウント（共通）
    network_storage:
      - fs_type: lustre
        server_ip: lustre-mds1.{{zone}}.c.${storage_project_id}.internal
        remote_mount: /lustre/users
        local_mount: /home

    # 計算クラスタ
    partitions:
      - name              : debug
        machine_type      : c2-standard-4
        static_node_count : 1
        max_node_count    : 22
        zone              : {{zone}}
        vpc_subnet        : default

        # プレイスメントポリシーを利用しない場合はコメントアウト
        resource_policies : ["hpc-cluster"]

        # ファイルシステムのマウント
        network_storage:
          - fs_type: lustre
            server_ip: lustre-mds1.{{zone}}.c.${storage_project_id}.internal
            remote_mount: /lustre/apps
            local_mount: /apps
EOF
```

## 計算クラスタの構築

設定内容を確認しましょう。マシンタイプや VM の起動数は適宜変更してください。

<walkthrough-editor-open-file filePath="~/cloudshell_open/google-cloud-tutorials/slurm-gcp/slurm-cluster.yaml">Editor を起動</walkthrough-editor-open-file>

### slurm.jinja の編集

現在、CentOS7 で [Lustre クライアントモジュールが正常に認識できない問題](https://jira.whamcloud.com/browse/LU-6062) が起こっています。いったんの回避策として、RHEL でクラスタが構成されるよう設定を変更します。

```bash
sed -ie "s|centos-cloud/global/images/family/centos-7|rhel-cloud/global/images/family/rhel-7|g" slurm.jinja
```

### 計算クラスタの VM 作成

以下コマンドで、計算クラスタを構築しましょう。

```bash
gcloud config set project "${compute_project_id}"
gcloud deployment-manager deployments create hpc-cluster \
    --config slurm-cluster.yaml
```

### 計算クラスタの初期化

クラスタの構成が完了するまで進行状況をトラッキングします。‘Started Google Compute Engine Startup Scripts.’ と出力されるまで お待ち下さい。およそ 15 分程度かかります。

```bash
gcloud compute ssh hpc-controller --zone {{zone}} --tunnel-through-iap \
    --command "sudo journalctl -fu google-startup-scripts.service"
```

## 挙動の確認

ログインノードに入り、クラスタの状況を確認、そして試しにジョブを投入してみます。

```bash
gcloud compute ssh hpc-controller --zone {{zone}} --tunnel-through-iap
```

ジョブを投入し、状況を確認してみます。ジョブの投入をトリガーにクラスタがスケールするため、 squeue の応答 ST（ステータス）が CD（COMPLETED: 完了）になるのを待ちます。

```bash
sbatch -N2 --wrap="srun hostname"
squeue
ls
sacct
exit
```

## 環境の削除

計算クラスタの削除

```bash
cd ~/cloudshell_open/google-cloud-tutorials/slurm-gcp/
gcloud config set project "${compute_project_id}"
gcloud deployment-manager deployments delete hpc-cluster
```

Lustre クラスタの削除

```bash
cd ~/cloudshell_open/google-cloud-tutorials/deploymentmanager-samples/community/lustre/
gcloud config set project "${storage_project_id}"
gcloud deployment-manager deployments delete lustre
```

Cloud NAT の削除

```bash
cd ~/cloudshell_open/google-cloud-tutorials/
gcloud config set project "${host_project_id}"
gcloud compute routers nats delete nat-config
gcloud compute routers delete nat-router --region {{region}}
```

## これで終わりです

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

すべて完了しました。

**作業後は忘れずにクリーンアップする**: テスト プロジェクトを作成した場合は、不要な料金の発生を避けるためにプロジェクトを削除してください。

```bash
gcloud projects delete {{project-id}}
```
