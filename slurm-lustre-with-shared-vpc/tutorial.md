# Slurm と Lustre による HPC 環境の構築（共有 VPC 版）

## 始めましょう

[共有 VPC](https://cloud.google.com/vpc/docs/shared-vpc?hl=ja) 上に Lustre による分散ストレージと Slurm ベースの計算クラスタを構築するための手順です。

**所要時間**: 約 60 分

**前提条件**: [組織が作成](https://cloud.google.com/resource-manager/docs/creating-managing-organization?hl=ja) してあり、共有 VPC / ストレージ / 計算クラスタのための 3 つのプロジェクトと、組織管理者兼各プロジェクトのオーナー権限をもつアカウントでログインしていることを確認してください。

**[開始]** ボタンをクリックして次のステップに進みます。

## プロジェクトの設定

この手順の中で実際にリソースを構築する対象のプロジェクトを選択してください。

<walkthrough-project-billing-setup permissions="iam.roles.create,iam.roles.delete,iam.roles.get,iam.roles.list,iam.roles.undelete,iam.roles.update,resourcemanager.organizations.get,resourcemanager.organizations.getIamPolicy,resourcemanager.projects.get,resourcemanager.projects.getIamPolicy,resourcemanager.projects.list"></walkthrough-project-billing-setup>

## CLI の設定

本チュートリアルをコマンドラインからも適切に実行できるよう、デフォルト設定を実施します。

```bash
gcloud config set compute/region {{region}}
gcloud config set compute/zone {{zone}}
```

チュートリアルで利用するいくつかの API を有効化します。

```bash
gcloud services enable container.googleapis.com containerregistry.googleapis.com cloudbuild.googleapis.com
```
