# [Google Cloud](https://cloud.google.com/?hl=ja) チュートリアル

## アプリケーション モダナイゼーション

### Buildpacks によるサーバーレス開発

1. 以下をクリックし、Cloud Shell 環境を起動してください。

[![Open in Cloud Shell](https://gstatic.com/cloudssh/images/open-btn.png)](https://console.cloud.google.com/home/dashboard?cloudshell=true)

2. 以下のコマンドをブラウザ上のターミナルで実行してください。チュートリアルが開始します。

```sh
cloudshell_open --repo_url "https://github.com/pottava/runrara-runrunrun.git" \
    --page "shell" --tutorial "buildpacks/tutorial.md"
```

3. Cloud Shell の再起動や予期せずチュートリアルが消えてしまった場合は以下で再開できます。

```sh
teachme ~/cloudshell_open/runrara-runrunrun/buildpacks/tutorial.md
```

### セキュアな GKE クラスタ の作成

[![Open in Cloud Shell](https://gstatic.com/cloudssh/images/open-btn.png)](https://console.cloud.google.com/home/dashboard?cloudshell=true)

```sh
cloudshell_open --repo_url "https://github.com/pottava/secured-gke-tutorial.git" \
    --page "shell" --tutorial "tutorial.md"
teachme ~/cloudshell_open/secured-gke-tutorial/tutorial.md
```

## HPC

### Slurm による HPC 環境の構築

1. 以下をクリックし、Cloud Shell 環境を起動してください。

[![Open in Cloud Shell](https://gstatic.com/cloudssh/images/open-btn.png)](https://console.cloud.google.com/home/dashboard?cloudshell=true)

2. 以下のコマンドをブラウザ上のターミナルで実行してください。チュートリアルが開始します。

```sh
cloudshell_open --repo_url "https://github.com/pottava/google-cloud-tutorials.git" \
    --page "shell" --tutorial "slurm/tutorial.md"
```

3. Cloud Shell の再起動や予期せずチュートリアルが消えてしまった場合は以下で再開できます。

```sh
teachme ~/cloudshell_open/google-cloud-tutorials/slurm/tutorial.md
```

### Slurm と Lustre による HPC 環境の構築

[![Open in Cloud Shell](https://gstatic.com/cloudssh/images/open-btn.png)](https://console.cloud.google.com/home/dashboard?cloudshell=true)

```sh
cloudshell_open --repo_url "https://github.com/pottava/google-cloud-tutorials.git" \
    --page "shell" --tutorial "slurm-lustre/tutorial.md"
teachme ~/cloudshell_open/google-cloud-tutorials/slurm-lustre/tutorial.md
```

### Slurm と Lustre による HPC 環境の構築（共有 VPC 版）

[![Open in Cloud Shell](https://gstatic.com/cloudssh/images/open-btn.png)](https://console.cloud.google.com/home/dashboard?cloudshell=true)

```sh
cloudshell_open --repo_url "https://github.com/pottava/google-cloud-tutorials.git" \
    --page "shell" --tutorial "slurm-lustre-with-shared-vpc/tutorial.md"
teachme ~/cloudshell_open/google-cloud-tutorials/slurm-lustre-with-shared-vpc/tutorial.md
```
