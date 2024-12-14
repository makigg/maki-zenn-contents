---
title: "誤自宅KaaSその1 Cluster APIインストール編"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "おうちkubernetes", "clusterapi", "proxmox", "kubeadm"]
published: false
---

## はじめに

Kubernetesはどこのご家庭にも1クラスタはあると思いますが、1人1クラスタほしいなと考える人も多いのではないでしょうか？
そんな声にお応えして、全3回のシリーズで、自宅にKaaS(Kubernetes as a Service)を構築する方法を解説します。

- Cluster APIインストール編 ←ココ
- ClusterClass設定編
- ストレージ設定編

目指すべき全体像はこんな感じ。Management ClusterにCluster APIからKubernetesのクラスタが自動で払いだされていきます。

![overview](/images/kaas-kubeadm-proxmox/overview.drawio.png)

## Cluster APIインストール編でやること

Cluster APIをインストールして、適当なクラスタを払いだします。

- Proxmoxの設定
- K3s構築
- K3s上にCluster APIをインストール
- Kubernetesクラスタのデプロイ

説明すること

- Cluster APIの動作フロー(概要)
- Cluster APIを使い始めるのに必要な操作
  - Packerの使い方
  - K3sのインストール方法
  - Cluster APIのインストール方法
  - Cluster APIを使ったKubernetesクラスタのプロビジョニング

説明しないこと

- ClusterClassを使ったインフラ構成の隠ぺい
- ClusterResourceSetを使ったクラスタの設定
- Cluster APIの各リソースの説明
- Proxmoxのインストール

## ハードウェアの構成

CPU: Ryzen 7 5800x
Memory: 64GB
SSD1: 500GB (OS用, /dev/nvme1n1)
SSD2: 2TB (/dev/nvme0n1)
GPU: RTX 3060 8GB (今回はGPU使わない)
IPアドレス: 192.168.10.138

## 使う技術の説明

### Proxmox VE

Proxmox VEはVM管理プラットフォームです。似たようなツールだと、VM WareのvSphereやMicrosoftのHyper-Vが有名ですね。

今回、Proxmox VEの使い道はこの3つです。

- Management Cluster(K3s)、Packer実行用VM
- Cluster APIに対して、VMを提供
- コンテナに対するストレージ提供 ([Proxmox CSI Plugin](https://github.com/sergelogvinov/proxmox-csi-plugin))

### Packer

PackerはVMテンプレートを作成するツールです。ゴールデンイメージの作成に使います。

https://www.packer.io/

### K3s

K3sは軽量版Kubernetesです。

自宅環境なので可用性不要で、とにかくリソース消費が小さいものが良いかなということで、Cluster APIのインストール先(Management Cluster)として採用しました。

https://k3s.io/

### Cluster API

Cluster APIとは、プロビジョニング、アップグレードなどのK8sクラスタの管理を宣言的に行うことができるツールです。今回はCluster APIを使ってKaaSを実現します。

https://cluster-api.sigs.k8s.io/introduction

Cluster APIでKubernetesクラスタを払いだすためには、事前に以下のものを準備しておく必要があります。

- Proxmox VE: Kubernetesクラスタの払い出し先インフラです。他にもDocker、KubeVirt、vSphere、各種ハイパースケーラーなどを選ぶことができます。各払い出し先インフラに合わせたCluster APIのInfrastructure Providerを使うことになるので、Proxmoxの場合、[Cluster API Provider Proxmox](https://github.com/k8s-proxmox/cluster-api-provider-proxmox)を使います。
- Management Cluster: KaaSを管理するKubernetesクラスタです。今回はK3sを使います。Management Clusterはどこに構築してもいいのですが、別のPCも用意したりはしてなかったので、Management Cluster自身もProxmox VE上のVMに構築しています。
- ゴールデンイメージ: 各KubernetesノードのベースとなるVMテンプレートです。kubelet, kubeadmが事前にインストールされています。今回は、Packerを使って作成しました。

上記を準備しておけば、Cluster APIは以下のフローを自動的に行い、Kubernetesクラスタをデプロイしてくれます。

1. ゴールデンイメージ(VMテンプレート)からControl Plane用のVMを作成する。その際、cloud-initの設定をISOでVMにマウントしておく。
2. cloud-initにより、kubeadm initの実行やStatic Podの作成が行われる。
3. ゴールデンイメージからNode用のVMを作成する。その際、cloud-initの設定をISOでVMにマウントしておく。
4. cloud-initにより、kubeadm joinが実行される。

![](/images/kaas-kubeadm-proxmox/cluster-api-deploy-flow.drawio.png)


## 手順

ほぼ、[Quick Start](https://cluster-api.sigs.k8s.io/user/quick-start)の手順通りです。

### Proxmox VEの初期設定

普通に初期設定するだけで良いと思います。
私は残ったストレージをすべてLVM-Thinプールにしました。

Proxmoxのストレージから直接コンテナ用のボリュームを作成したい場合([Proxmox CSI Plugin](https://github.com/sergelogvinov/proxmox-csi-plugin)を使う場合)、Proxmoxのクラスタを作成しておきます。

今回は`region1`という名前でクラスタを作成。

![](https://www.tecmint.com/wp-content/uploads/2024/02/Create-Cluster-in-Proxmox.png)

https://www.tecmint.com/proxmox-clustering-and-high-availability/

### Packer & Management Cluster用VMの作成

PackerとManagement Clusterを動作させる環境が必要なので、Proxmox上にVMを作成します。私はUbuntu 24.04.1 LTSを使いました。
私はProxmox上に構築しましたが、PackerとK3sが動かせればそれでよいので、別の手段でもよいでしょう。

### Packerでゴールデンイメージを作る

[Proxmox用のイメージ作成手順](https://image-builder.sigs.k8s.io/capi/providers/proxmox)を参考にしてゴールデンイメージを作成していきます。

Packerは以下のようにゴールデンイメージを作成するフローを自動化してくれます。

![](/images/kaas-kubeadm-proxmox/packer.drawio.png)

WSL2上でPackerを使ってみたのですが、VMのcloud-initが完了せずタイムアウトになりました。あきらめて、Proxmox上のVM上で実行しました。

#### Packerの準備

[公式サイトを参考](https://developer.hashicorp.com/packer/tutorials/docker-get-started/get-started-install-cli)にPackerをインストールします。

[kubernetes-sigs/image-builder](https://github.com/kubernetes-sigs/image-builder)を自分の環境用に書き換えていきます。

image-builderをcloneします。

```shell
git clone https://github.com/kubernetes-sigs/image-builder.git
```

依存関係をインストール

```shell
cd images/capi
make deps-proxmox
```

私は以下の内容を書き換えました。

- LVM-Thinプールを使う場合、ディスクフォーマットを`qcow2`から`raw`に変更。
- SCSI Controllerに`virtio-scsi-single`を設定。詳しくはストレージ設定編で説明するのですが、コンテナ用の永続ボリュームにProxmoxストレージを使いたい場合、SCSIコントローラーにVirtIO SCSI singleもしくは、VirtIO SCSI を設定する必要があります。
- IPv6を使わない場合、`boot_command_prefix`に`ipv6.disable=1`を追加。

```diff
diff --git a/images/capi/packer/proxmox/packer.json.tmpl b/images/capi/packer/proxmox/packer.json.tmpl
index bb0bdc226..8b6df41d0 100644
--- a/images/capi/packer/proxmox/packer.json.tmpl
+++ b/images/capi/packer/proxmox/packer.json.tmpl
@@ -13,7 +13,7 @@
       "disks": [
         {
           "disk_size": "{{user `disk_size`}}",
-          "format": "qcow2",
+          "format": "raw",
           "storage_pool": "{{user `storage_pool`}}",
           "storage_pool_type": "{{user `storage_pool_type`}}",
           "type": "scsi"
@@ -42,7 +42,8 @@
       "template_name": "{{ user `artifact_name` }}",
       "type": "proxmox-iso",
       "unmount_iso": "{{user `unmount_iso`}}",
-      "vm_id": "{{user `vmid`}}"
+      "vm_id": "{{user `vmid`}}",
+      "scsi_controller": "virtio-scsi-single"
     }
   ],
   "post-processors": [
diff --git a/images/capi/packer/proxmox/ubuntu-2404.json b/images/capi/packer/proxmox/ubuntu-2404.json
index 2e5987105..82ba7001e 100644
--- a/images/capi/packer/proxmox/ubuntu-2404.json
+++ b/images/capi/packer/proxmox/ubuntu-2404.json
@@ -1,5 +1,5 @@
 {
-  "boot_command_prefix": "c<wait>linux /casper/vmlinuz --- autoinstall ds='nocloud-net;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/24.04/'<enter><wait10s>initrd /casper/initrd<enter><wait10s>boot<enter><wait10s>",
+  "boot_command_prefix": "c<wait>linux /casper/vmlinuz ipv6.disable=1 --- autoinstall ds='nocloud-net;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/24.04/'<enter><wait10s>initrd /casper/initrd<enter><wait10s>boot<enter><wait10s>",
   "build_name": "ubuntu-2404",
   "distribution_version": "2404",
   "distro_name": "ubuntu",
```

#### Proxmoxのトークン作成

PackerからProxmoxを操作するためのユーザーとトークンを生成します。ProxmoxのGUI/CLIのどちらを使っても使うことができます。今回はCLIで操作します。

```shell
pveum user add packer@pve
pveum aclmod / -user packer@pve -role PVEVMAdmin
pveum user token add packer@pve packer -privsep 0
```

こんな感じの出力が得られる。トークンIDとトークン(value)をメモします。

```shell
┌──────────────┬──────────────────────────────────────┐
│ key          │ value                                │
╞══════════════╪══════════════════════════════════════╡
│ full-tokenid │ packer@pve!packer                    │
├──────────────┼──────────────────────────────────────┤
│ info         │ {"privsep":"0"}                      │
├──────────────┼──────────────────────────────────────┤
│ value        │ xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx │
└──────────────┴──────────────────────────────────────┘
```

#### Packerを実行してゴールデンイメージを作成

Packerを実行して、VMのゴールデンイメージを作成します。

Packerの接続設定をします。

```shell
# 先ほど生成したトークン
export PROXMOX_USERNAME='packer@pve!packer'
export PROXMOX_TOKEN="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Proxmox VE APIのURL
export PROXMOX_URL="https://192.168.10.138:8006/api2/json"
# Proxmoxのノード名、Proxmoxの画面から確認できる
export PROXMOX_NODE="pve1"
# ISOの格納されたストレージプール
# Datacenter > Storageから確認する
export PROXMOX_ISO_POOL="local"
# ネットワークデバイス。vmbr0指定しておけばよさそう。
export PROXMOX_BRIDGE="vmbr0"
# ゴールデンイメージ用VMのボリュームで使うストレージプール
# Datacenter > Storageから確認する
export PROXMOX_STORAGE_POOL="local-lvm"
```

Packerを実行します。

```shell
make build-proxmox-ubuntu-2404
```

Kubernetesクラスタをデプロイするときに使うので、ゴールデンイメージのVM IDを確認します。今回は`106`ですね。

![](/images/kaas-kubeadm-proxmox/packer-golden-image.drawio.png)

### Management Clusterの構築

Proxmox上に立てたVMにKaaSを実現するためのManagement Clusterを構築します。

#### K3sのインストール

K3sのインストールはこれだけです。

```shell
curl -sfL https://get.k3s.io | sh -
```

https://docs.k3s.io/quick-start

#### Cluster APIをManagement Clusterにインストール

Cluster APIがProxmoxを操作するためのユーザーとトークンを生成します。

```shell
pveum user add capmox@pve
pveum aclmod / -user capmox@pve -role PVEVMAdmin
pveum user token add capmox@pve capi -privsep 0
```

トークンIDとトークン(value)をメモする。

```shell
┌──────────────┬──────────────────────────────────────┐
│ key          │ value                                │
╞══════════════╪══════════════════════════════════════╡
│ full-tokenid │ capmox@pve!capi                      │
├──────────────┼──────────────────────────────────────┤
│ info         │ {"privsep":"0"}                      │
├──────────────┼──────────────────────────────────────┤
│ value        │ xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx │
└──────────────┴──────────────────────────────────────┘
```

Cluster APIの設定を環境変数に入れる。

```shell
# Proxmox VE APIのURL
export PROXMOX_URL="https://192.168.10.138:8006"
# 先ほど生成したトークン
export PROXMOX_TOKEN='capmox@pve!capi'
export PROXMOX_SECRET="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# ClusterClass設定編で使う
export CLUSTER_TOPOLOGY=true
# ストレージ設定編で使う
export EXP_CLUSTER_RESOURCE_SET=true
```

Cluster APIをManagement Clusterにインストールする

```shell
clusterctl init --infrastructure proxmox --ipam in-cluster
```

### Kubernetesクラスタをプロビジョニング

KaaS払い出し用Namespaceを作成する

```shell
kubectl create kaas
```

環境変数で設定をします。

```shell
# Proxmoxのノード名、Proxmoxの画面から確認できる
export PROXMOX_SOURCENODE="pve1"

# Packerで作ったゴールデンイメージのVM ID
# Proxmoxの画面見るとubuntu-2404-kube-v1.30.5みたいな名前で作成されているので、そのVM IDを指定する
export TEMPLATE_VMID=106

# 払いだしたKubernetesの各ノードにSSHするときに使う公開鍵（カンマ区切りで複数書ける）
# 私はWindowsとWSL2の公開鍵を入れている
export VM_SSH_KEYS="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDhQKh2EmK+5FWCw89zHw1aBewgp8S2L/mNlZr10Dx6Dv7x6HCt/ooLJiwIXz+Lfj9pB2ikLbZ14zOP635dCjnSRFky8yHJwH5RpvuLFFOLmoELRFFNJPq5cuVsDXjMod/1rDSajqXY0Ml0cmQIQKvR1Nr/+8rVWrvVjt0pwsWEPpBNPl0L/xgrHgaQIQj6JAQco14XnY0n6bXEOewswKDWmUxdPYoOgMH+dokjjKuvJAuBDhYiGYrD9XwClduyDORDt3OwmuZYDGEIQLBa4oXLF1J8zF0tIDDwYlN+vh2zx2HxLJ6kIk+2FgW+mnvVi3s3v174GqjeIqoMF8EJtcB+IH4ACqruPdjiAgKR581wStcdfjISjcP0AYe9t6L86uG1aIViWPWL7RPZNY4KR4RsYwk9kXGOA6LVr0VmfJvNslU7+oCb67XlxLRHUNBOMy8EfadIXDoIz3FUch1HW2BNbhkStrujrEqk1B5/zlsFRQ7shmI3ccHr9QByslxcFgs=,ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCjp0UvoAQy+wUpfEHr1YVU04htWc0qnVhtSCeXVD1Q8kiz/7itIKKfKGvz+PFU9kNcQlrQaUxQ/lh9SUC0y/epVLcxm3TBI99HTeeWTrO1RLvDIR0CS3iC2USPMEX0jbhF2fUoE+Q+vaZArNsP/zR1fEV2Dc5HW/dgjcKwNN9Xr7p6BujkwOQXM8DdqcTTMIdRoZhisD7LGDfWkro1fcoo0eeMGgdcGx/DJLw0s840t1teVxE0Payt4LGMtg/jOCOCXRTzmvLDSfdxBjxYGhT2IK/GFPoC/L2frFaie0N2dRRGI0j4SBgNQbJudQanJvsuT35UkMUrxgkANyrbFQmVmcJhuwo3mbXRLaBSojpBv3AQ3eF5X1WKvTNcZ3H04E0acReUtQcy00CP+8rI1uZ+bhC6NxgR2g1GPKoJZA2EJp4LF9oqyczQzKz62dXGXV5I9c2Sf2Q+q4bVIQjKc79U5EdQnQkTEIyEfI+ib+XvclrT1c1M95uTw1cogOUBDhk="

# Control Planeの仮想IPアドレス
# Control Planeの各ノードには別途IPアドレスが割り当てられます
export CONTROL_PLANE_ENDPOINT_IP=192.168.10.210

# Kubernetesの各ノード(Control Plane, Node)用IPアドレス
export NODE_IP_RANGES="[192.168.10.211-192.168.10.214]"

export GATEWAY="192.168.10.1"
export IP_PREFIX=24

# ネットワークデバイス。vmbr0指定しておけばよさそう。
export BRIDGE="vmbr0"
# 各ノードに設定されるDNSサーバー
export DNS_SERVERS="[8.8.8.8,8.8.4.4]"
# VMが払いだされるProxmoxノード
export ALLOWED_NODES="[pve1]"
export BOOT_VOLUME_DEVICE="scsi0"
```

Kubernetesクラスタを払いだします。

```shell
clusterctl generate cluster cluster01 --kubernetes-version v1.30.5 > cluster01-gen.yaml
kubectl apply -n kaas -f cluster01-gen.yaml
```

:::details cluster01-gen.yaml
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: cluster01
  namespace: kaas
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
      - 192.168.0.0/16
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: cluster01-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
    kind: ProxmoxCluster
    name: cluster01
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: ProxmoxCluster
metadata:
  name: cluster01
  namespace: kaas
spec:
  allowedNodes:
  - pve1
  controlPlaneEndpoint:
    host: 192.168.10.210
    port: 6443
  dnsServers:
  - 8.8.8.8
  - 8.8.4.4
  ipv4Config:
    addresses:
    - 192.168.10.211-192.168.10.214
    gateway: 192.168.10.1
    prefix: 24
---
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: cluster01-control-plane
  namespace: kaas
spec:
  kubeadmConfigSpec:
    files:
    - content: |
        apiVersion: v1
        kind: Pod
        metadata:
          creationTimestamp: null
          name: kube-vip
          namespace: kube-system
        spec:
          containers:
          - args:
            - manager
            env:
            - name: cp_enable
              value: "true"
            - name: vip_interface
              value: ""
            - name: address
              value: 192.168.10.210
            - name: port
              value: "6443"
            - name: vip_arp
              value: "true"
            - name: vip_leaderelection
              value: "true"
            - name: vip_leaseduration
              value: "15"
            - name: vip_renewdeadline
              value: "10"
            - name: vip_retryperiod
              value: "2"
            image: ghcr.io/kube-vip/kube-vip:v0.7.1
            imagePullPolicy: IfNotPresent
            name: kube-vip
            resources: {}
            securityContext:
              capabilities:
                add:
                - NET_ADMIN
                - NET_RAW
            volumeMounts:
            - mountPath: /etc/kubernetes/admin.conf
              name: kubeconfig
          hostAliases:
          - hostnames:
            - localhost
            - kubernetes
            ip: 127.0.0.1
          hostNetwork: true
          volumes:
          - hostPath:
              path: /etc/kubernetes/admin.conf
              type: FileOrCreate
            name: kubeconfig
        status: {}
      owner: root:root
      path: /etc/kubernetes/manifests/kube-vip.yaml
    - content: |
        #!/bin/bash

        # Copyright 2020 The Kubernetes Authors.
        #
        # Licensed under the Apache License, Version 2.0 (the "License");
        # you may not use this file except in compliance with the License.
        # You may obtain a copy of the License at
        #
        #     http://www.apache.org/licenses/LICENSE-2.0
        #
        # Unless required by applicable law or agreed to in writing, software
        # distributed under the License is distributed on an "AS IS" BASIS,
        # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
        # See the License for the specific language governing permissions and
        # limitations under the License.

        set -e

        # Configure the workaround required for kubeadm init with kube-vip:
        # xref: https://github.com/kube-vip/kube-vip/issues/684

        # Nothing to do for kubernetes < v1.29
        KUBEADM_MINOR="$(kubeadm version -o short | cut -d '.' -f 2)"
        if [[ "$KUBEADM_MINOR" -lt "29" ]]; then
          exit 0
        fi

        IS_KUBEADM_INIT="false"

        # cloud-init kubeadm init
        if [[ -f /run/kubeadm/kubeadm.yaml ]]; then
          IS_KUBEADM_INIT="true"
        fi

        # ignition kubeadm init
        if [[ -f /etc/kubeadm.sh ]] && grep -q -e "kubeadm init" /etc/kubeadm.sh; then
          IS_KUBEADM_INIT="true"
        fi

        if [[ "$IS_KUBEADM_INIT" == "true" ]]; then
          sed -i 's#path: /etc/kubernetes/admin.conf#path: /etc/kubernetes/super-admin.conf#' \
            /etc/kubernetes/manifests/kube-vip.yaml
        fi
      owner: root:root
      path: /etc/kube-vip-prepare.sh
      permissions: "0700"
    initConfiguration:
      nodeRegistration:
        kubeletExtraArgs:
          provider-id: proxmox://'{{ ds.meta_data.instance_id }}'
    joinConfiguration:
      nodeRegistration:
        kubeletExtraArgs:
          provider-id: proxmox://'{{ ds.meta_data.instance_id }}'
    preKubeadmCommands:
    - /etc/kube-vip-prepare.sh
    users:
    - name: root
      sshAuthorizedKeys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDhQKh2EmK+5FWCw89zHw1aBewgp8S2L/mNlZr10Dx6Dv7x6HCt/ooLJiwIXz+Lfj9pB2ikLbZ14zOP635dCjnSRFky8yHJwH5RpvuLFFOLmoELRFFNJPq5cuVsDXjMod/1rDSajqXY0Ml0cmQIQKvR1Nr/+8rVWrvVjt0pwsWEPpBNPl0L/xgrHgaQIQj6JAQco14XnY0n6bXEOewswKDWmUxdPYoOgMH+dokjjKuvJAuBDhYiGYrD9XwClduyDORDt3OwmuZYDGEIQLBa4oXLF1J8zF0tIDDwYlN+vh2zx2HxLJ6kIk+2FgW+mnvVi3s3v174GqjeIqoMF8EJtcB+IH4ACqruPdjiAgKR581wStcdfjISjcP0AYe9t6L86uG1aIViWPWL7RPZNY4KR4RsYwk9kXGOA6LVr0VmfJvNslU7+oCb67XlxLRHUNBOMy8EfadIXDoIz3FUch1HW2BNbhkStrujrEqk1B5/zlsFRQ7shmI3ccHr9QByslxcFgs=
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCjp0UvoAQy+wUpfEHr1YVU04htWc0qnVhtSCeXVD1Q8kiz/7itIKKfKGvz+PFU9kNcQlrQaUxQ/lh9SUC0y/epVLcxm3TBI99HTeeWTrO1RLvDIR0CS3iC2USPMEX0jbhF2fUoE+Q+vaZArNsP/zR1fEV2Dc5HW/dgjcKwNN9Xr7p6BujkwOQXM8DdqcTTMIdRoZhisD7LGDfWkro1fcoo0eeMGgdcGx/DJLw0s840t1teVxE0Payt4LGMtg/jOCOCXRTzmvLDSfdxBjxYGhT2IK/GFPoC/L2frFaie0N2dRRGI0j4SBgNQbJudQanJvsuT35UkMUrxgkANyrbFQmVmcJhuwo3mbXRLaBSojpBv3AQ3eF5X1WKvTNcZ3H04E0acReUtQcy00CP+8rI1uZ+bhC6NxgR2g1GPKoJZA2EJp4LF9oqyczQzKz62dXGXV5I9c2Sf2Q+q4bVIQjKc79U5EdQnQkTEIyEfI+ib+XvclrT1c1M95uTw1cogOUBDhk=
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
      kind: ProxmoxMachineTemplate
      name: cluster01-control-plane
  replicas: 1
  version: v1.30.5
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: ProxmoxMachineTemplate
metadata:
  name: cluster01-control-plane
  namespace: kaas
spec:
  template:
    spec:
      disks:
        bootVolume:
          disk: scsi0
          sizeGb: 100
      format: qcow2
      full: true
      memoryMiB: 16384
      network:
        default:
          bridge: vmbr0
          model: virtio
      numCores: 4
      numSockets: 2
      sourceNode: pve1
      templateID: 106
---
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: cluster01-workers
  namespace: kaas
spec:
  clusterName: cluster01
  replicas: 0
  selector:
    matchLabels: null
  template:
    metadata:
      labels:
        node-role.kubernetes.io/node: ""
    spec:
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: cluster01-worker
      clusterName: cluster01
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
        kind: ProxmoxMachineTemplate
        name: cluster01-worker
      version: v1.30.5
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: ProxmoxMachineTemplate
metadata:
  name: cluster01-worker
  namespace: kaas
spec:
  template:
    spec:
      disks:
        bootVolume:
          disk: scsi0
          sizeGb: 100
      format: qcow2
      full: true
      memoryMiB: 16384
      network:
        default:
          bridge: vmbr0
          model: virtio
      numCores: 4
      numSockets: 2
      sourceNode: pve1
      templateID: 106
---
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfigTemplate
metadata:
  name: cluster01-worker
  namespace: kaas
spec:
  template:
    spec:
      joinConfiguration:
        nodeRegistration:
          kubeletExtraArgs:
            provider-id: proxmox://'{{ ds.meta_data.instance_id }}'
      users:
      - name: root
        sshAuthorizedKeys:
        - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDhQKh2EmK+5FWCw89zHw1aBewgp8S2L/mNlZr10Dx6Dv7x6HCt/ooLJiwIXz+Lfj9pB2ikLbZ14zOP635dCjnSRFky8yHJwH5RpvuLFFOLmoELRFFNJPq5cuVsDXjMod/1rDSajqXY0Ml0cmQIQKvR1Nr/+8rVWrvVjt0pwsWEPpBNPl0L/xgrHgaQIQj6JAQco14XnY0n6bXEOewswKDWmUxdPYoOgMH+dokjjKuvJAuBDhYiGYrD9XwClduyDORDt3OwmuZYDGEIQLBa4oXLF1J8zF0tIDDwYlN+vh2zx2HxLJ6kIk+2FgW+mnvVi3s3v174GqjeIqoMF8EJtcB+IH4ACqruPdjiAgKR581wStcdfjISjcP0AYe9t6L86uG1aIViWPWL7RPZNY4KR4RsYwk9kXGOA6LVr0VmfJvNslU7+oCb67XlxLRHUNBOMy8EfadIXDoIz3FUch1HW2BNbhkStrujrEqk1B5/zlsFRQ7shmI3ccHr9QByslxcFgs=
        - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCjp0UvoAQy+wUpfEHr1YVU04htWc0qnVhtSCeXVD1Q8kiz/7itIKKfKGvz+PFU9kNcQlrQaUxQ/lh9SUC0y/epVLcxm3TBI99HTeeWTrO1RLvDIR0CS3iC2USPMEX0jbhF2fUoE+Q+vaZArNsP/zR1fEV2Dc5HW/dgjcKwNN9Xr7p6BujkwOQXM8DdqcTTMIdRoZhisD7LGDfWkro1fcoo0eeMGgdcGx/DJLw0s840t1teVxE0Payt4LGMtg/jOCOCXRTzmvLDSfdxBjxYGhT2IK/GFPoC/L2frFaie0N2dRRGI0j4SBgNQbJudQanJvsuT35UkMUrxgkANyrbFQmVmcJhuwo3mbXRLaBSojpBv3AQ3eF5X1WKvTNcZ3H04E0acReUtQcy00CP+8rI1uZ+bhC6NxgR2g1GPKoJZA2EJp4LF9oqyczQzKz62dXGXV5I9c2Sf2Q+q4bVIQjKc79U5EdQnQkTEIyEfI+ib+XvclrT1c1M95uTw1cogOUBDhk=
```
:::

Kubernetesクラスタが払いだされているのが確認できます。

```shell
❯ kubectl get cluster -n kaas
NAME        CLUSTERCLASS      PHASE         AGE   VERSION
cluster01   kubeadm-v1.30.5   Provisioned   9h    v1.30.5
```

払いだされたノード(VM)を確認できます。

```shell
❯ kubectl get machine -n kaas
NAME                                   CLUSTER     NODENAME                               PROVIDERID                                       PHASE     AGE   VERSION
cluster01-cpu-node-zl6vk-2klbn-mk22j   cluster01   cluster01-cpu-node-zl6vk-2klbn-mk22j   proxmox://866739c6-30cc-4a1b-8fba-a1875474ade1   Running   9h    v1.30.5
cluster01-wt4kp-2ttpn                  cluster01   cluster01-wt4kp-2ttpn                  proxmox://b6a092f1-0266-45e6-b216-f73634d60627   Running   9h    v1.30.5
```

### クラスタにアクセス

kubeconfigを取得します。

```shell
clusterctl get kubeconfig cluster01 -n kaas > ~/.kube/cluster01.config

if [ -z "$KUBECONFIG" ]; then
  export KUBECONFIG=${HOME}/.kube/config
fi
export KUBECONFIG=${KUBECONFIG}:${HOME}/.kube/cluster01.config
```

:::message
Tips: `KUBECONFIG`は`:`で複数ファイルをつなげることができます。
:::

払いだされたクラスタのコンテキストに切り替えて、クラスタの初期設定をしていきましょう。

```shell
kubectl config use-context cluster01-admin@cluster01
```

CNIをインストールします。(今回はCiliumを使います)

```shell
cilium install
```

これでクラスタの初期設定は完了したので、好きにPodをデプロイできる状態になります。

### クリーンアップ

払いだしたクラスタを削除します。

Management Clusterで操作します。

```shell
kubectl config use-context k3s
```

clusterリソースを削除すると、他もすべて削除されます。

```shell
kubectl delete cluster cluster01
```

## まとめ

自宅のPCに構築したProxmox上にCluster APIをインストールして、Kubernetesクラスタを払いだしてみました。

さて、これでKaaS完成といいたいところですが、KaaSというにはまだ課題が残ります。

- 1つクラスタを作るためにインフラ構成を意識したパラメーターを環境変数に設定しないといけない
- 1クラスタを構成するマニフェストが266行と巨大なので、払い出しのたびに作成するのも、IaCで管理するのも大変
  :::details cluster01-gen.yaml
  ```yaml
  apiVersion: cluster.x-k8s.io/v1beta1
  kind: Cluster
  metadata:
    name: cluster01
    namespace: kaas
  spec:
    clusterNetwork:
      pods:
        cidrBlocks:
        - 192.168.0.0/16
    controlPlaneRef:
      apiVersion: controlplane.cluster.x-k8s.io/v1beta1
      kind: KubeadmControlPlane
      name: cluster01-control-plane
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
      kind: ProxmoxCluster
      name: cluster01
  ---
  apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
  kind: ProxmoxCluster
  metadata:
    name: cluster01
    namespace: kaas
  spec:
    allowedNodes:
    - pve1
    controlPlaneEndpoint:
      host: 192.168.10.210
      port: 6443
    dnsServers:
    - 8.8.8.8
    - 8.8.4.4
    ipv4Config:
      addresses:
      - 192.168.10.211-192.168.10.214
      gateway: 192.168.10.1
      prefix: 24
  ---
  apiVersion: controlplane.cluster.x-k8s.io/v1beta1
  kind: KubeadmControlPlane
  metadata:
    name: cluster01-control-plane
    namespace: kaas
  spec:
    kubeadmConfigSpec:
      files:
      - content: |
          apiVersion: v1
          kind: Pod
          metadata:
            creationTimestamp: null
            name: kube-vip
            namespace: kube-system
          spec:
            containers:
            - args:
              - manager
              env:
              - name: cp_enable
                value: "true"
              - name: vip_interface
                value: ""
              - name: address
                value: 192.168.10.210
              - name: port
                value: "6443"
              - name: vip_arp
                value: "true"
              - name: vip_leaderelection
                value: "true"
              - name: vip_leaseduration
                value: "15"
              - name: vip_renewdeadline
                value: "10"
              - name: vip_retryperiod
                value: "2"
              image: ghcr.io/kube-vip/kube-vip:v0.7.1
              imagePullPolicy: IfNotPresent
              name: kube-vip
              resources: {}
              securityContext:
                capabilities:
                  add:
                  - NET_ADMIN
                  - NET_RAW
              volumeMounts:
              - mountPath: /etc/kubernetes/admin.conf
                name: kubeconfig
            hostAliases:
            - hostnames:
              - localhost
              - kubernetes
              ip: 127.0.0.1
            hostNetwork: true
            volumes:
            - hostPath:
                path: /etc/kubernetes/admin.conf
                type: FileOrCreate
              name: kubeconfig
          status: {}
        owner: root:root
        path: /etc/kubernetes/manifests/kube-vip.yaml
      - content: |
          #!/bin/bash

          # Copyright 2020 The Kubernetes Authors.
          #
          # Licensed under the Apache License, Version 2.0 (the "License");
          # you may not use this file except in compliance with the License.
          # You may obtain a copy of the License at
          #
          #     http://www.apache.org/licenses/LICENSE-2.0
          #
          # Unless required by applicable law or agreed to in writing, software
          # distributed under the License is distributed on an "AS IS" BASIS,
          # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
          # See the License for the specific language governing permissions and
          # limitations under the License.

          set -e

          # Configure the workaround required for kubeadm init with kube-vip:
          # xref: https://github.com/kube-vip/kube-vip/issues/684

          # Nothing to do for kubernetes < v1.29
          KUBEADM_MINOR="$(kubeadm version -o short | cut -d '.' -f 2)"
          if [[ "$KUBEADM_MINOR" -lt "29" ]]; then
            exit 0
          fi

          IS_KUBEADM_INIT="false"

          # cloud-init kubeadm init
          if [[ -f /run/kubeadm/kubeadm.yaml ]]; then
            IS_KUBEADM_INIT="true"
          fi

          # ignition kubeadm init
          if [[ -f /etc/kubeadm.sh ]] && grep -q -e "kubeadm init" /etc/kubeadm.sh; then
            IS_KUBEADM_INIT="true"
          fi

          if [[ "$IS_KUBEADM_INIT" == "true" ]]; then
            sed -i 's#path: /etc/kubernetes/admin.conf#path: /etc/kubernetes/super-admin.conf#' \
              /etc/kubernetes/manifests/kube-vip.yaml
          fi
        owner: root:root
        path: /etc/kube-vip-prepare.sh
        permissions: "0700"
      initConfiguration:
        nodeRegistration:
          kubeletExtraArgs:
            provider-id: proxmox://'{{ ds.meta_data.instance_id }}'
      joinConfiguration:
        nodeRegistration:
          kubeletExtraArgs:
            provider-id: proxmox://'{{ ds.meta_data.instance_id }}'
      preKubeadmCommands:
      - /etc/kube-vip-prepare.sh
      users:
      - name: root
        sshAuthorizedKeys:
        - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDhQKh2EmK+5FWCw89zHw1aBewgp8S2L/mNlZr10Dx6Dv7x6HCt/ooLJiwIXz+Lfj9pB2ikLbZ14zOP635dCjnSRFky8yHJwH5RpvuLFFOLmoELRFFNJPq5cuVsDXjMod/1rDSajqXY0Ml0cmQIQKvR1Nr/+8rVWrvVjt0pwsWEPpBNPl0L/xgrHgaQIQj6JAQco14XnY0n6bXEOewswKDWmUxdPYoOgMH+dokjjKuvJAuBDhYiGYrD9XwClduyDORDt3OwmuZYDGEIQLBa4oXLF1J8zF0tIDDwYlN+vh2zx2HxLJ6kIk+2FgW+mnvVi3s3v174GqjeIqoMF8EJtcB+IH4ACqruPdjiAgKR581wStcdfjISjcP0AYe9t6L86uG1aIViWPWL7RPZNY4KR4RsYwk9kXGOA6LVr0VmfJvNslU7+oCb67XlxLRHUNBOMy8EfadIXDoIz3FUch1HW2BNbhkStrujrEqk1B5/zlsFRQ7shmI3ccHr9QByslxcFgs=
        - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCjp0UvoAQy+wUpfEHr1YVU04htWc0qnVhtSCeXVD1Q8kiz/7itIKKfKGvz+PFU9kNcQlrQaUxQ/lh9SUC0y/epVLcxm3TBI99HTeeWTrO1RLvDIR0CS3iC2USPMEX0jbhF2fUoE+Q+vaZArNsP/zR1fEV2Dc5HW/dgjcKwNN9Xr7p6BujkwOQXM8DdqcTTMIdRoZhisD7LGDfWkro1fcoo0eeMGgdcGx/DJLw0s840t1teVxE0Payt4LGMtg/jOCOCXRTzmvLDSfdxBjxYGhT2IK/GFPoC/L2frFaie0N2dRRGI0j4SBgNQbJudQanJvsuT35UkMUrxgkANyrbFQmVmcJhuwo3mbXRLaBSojpBv3AQ3eF5X1WKvTNcZ3H04E0acReUtQcy00CP+8rI1uZ+bhC6NxgR2g1GPKoJZA2EJp4LF9oqyczQzKz62dXGXV5I9c2Sf2Q+q4bVIQjKc79U5EdQnQkTEIyEfI+ib+XvclrT1c1M95uTw1cogOUBDhk=
    machineTemplate:
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
        kind: ProxmoxMachineTemplate
        name: cluster01-control-plane
    replicas: 1
    version: v1.30.5
  ---
  apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
  kind: ProxmoxMachineTemplate
  metadata:
    name: cluster01-control-plane
    namespace: kaas
  spec:
    template:
      spec:
        disks:
          bootVolume:
            disk: scsi0
            sizeGb: 100
        format: qcow2
        full: true
        memoryMiB: 16384
        network:
          default:
            bridge: vmbr0
            model: virtio
        numCores: 4
        numSockets: 2
        sourceNode: pve1
        templateID: 106
  ---
  apiVersion: cluster.x-k8s.io/v1beta1
  kind: MachineDeployment
  metadata:
    name: cluster01-workers
    namespace: kaas
  spec:
    clusterName: cluster01
    replicas: 0
    selector:
      matchLabels: null
    template:
      metadata:
        labels:
          node-role.kubernetes.io/node: ""
      spec:
        bootstrap:
          configRef:
            apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
            kind: KubeadmConfigTemplate
            name: cluster01-worker
        clusterName: cluster01
        infrastructureRef:
          apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
          kind: ProxmoxMachineTemplate
          name: cluster01-worker
        version: v1.30.5
  ---
  apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
  kind: ProxmoxMachineTemplate
  metadata:
    name: cluster01-worker
    namespace: kaas
  spec:
    template:
      spec:
        disks:
          bootVolume:
            disk: scsi0
            sizeGb: 100
        format: qcow2
        full: true
        memoryMiB: 16384
        network:
          default:
            bridge: vmbr0
            model: virtio
        numCores: 4
        numSockets: 2
        sourceNode: pve1
        templateID: 106
  ---
  apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
  kind: KubeadmConfigTemplate
  metadata:
    name: cluster01-worker
    namespace: kaas
  spec:
    template:
      spec:
        joinConfiguration:
          nodeRegistration:
            kubeletExtraArgs:
              provider-id: proxmox://'{{ ds.meta_data.instance_id }}'
        users:
        - name: root
          sshAuthorizedKeys:
          - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDhQKh2EmK+5FWCw89zHw1aBewgp8S2L/mNlZr10Dx6Dv7x6HCt/ooLJiwIXz+Lfj9pB2ikLbZ14zOP635dCjnSRFky8yHJwH5RpvuLFFOLmoELRFFNJPq5cuVsDXjMod/1rDSajqXY0Ml0cmQIQKvR1Nr/+8rVWrvVjt0pwsWEPpBNPl0L/xgrHgaQIQj6JAQco14XnY0n6bXEOewswKDWmUxdPYoOgMH+dokjjKuvJAuBDhYiGYrD9XwClduyDORDt3OwmuZYDGEIQLBa4oXLF1J8zF0tIDDwYlN+vh2zx2HxLJ6kIk+2FgW+mnvVi3s3v174GqjeIqoMF8EJtcB+IH4ACqruPdjiAgKR581wStcdfjISjcP0AYe9t6L86uG1aIViWPWL7RPZNY4KR4RsYwk9kXGOA6LVr0VmfJvNslU7+oCb67XlxLRHUNBOMy8EfadIXDoIz3FUch1HW2BNbhkStrujrEqk1B5/zlsFRQ7shmI3ccHr9QByslxcFgs=
          - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCjp0UvoAQy+wUpfEHr1YVU04htWc0qnVhtSCeXVD1Q8kiz/7itIKKfKGvz+PFU9kNcQlrQaUxQ/lh9SUC0y/epVLcxm3TBI99HTeeWTrO1RLvDIR0CS3iC2USPMEX0jbhF2fUoE+Q+vaZArNsP/zR1fEV2Dc5HW/dgjcKwNN9Xr7p6BujkwOQXM8DdqcTTMIdRoZhisD7LGDfWkro1fcoo0eeMGgdcGx/DJLw0s840t1teVxE0Payt4LGMtg/jOCOCXRTzmvLDSfdxBjxYGhT2IK/GFPoC/L2frFaie0N2dRRGI0j4SBgNQbJudQanJvsuT35UkMUrxgkANyrbFQmVmcJhuwo3mbXRLaBSojpBv3AQ3eF5X1WKvTNcZ3H04E0acReUtQcy00CP+8rI1uZ+bhC6NxgR2g1GPKoJZA2EJp4LF9oqyczQzKz62dXGXV5I9c2Sf2Q+q4bVIQjKc79U5EdQnQkTEIyEfI+ib+XvclrT1c1M95uTw1cogOUBDhk=
  ```
  :::

ということで、次回はClusterClassを使って、インフラ構成を意識しない短いマニフェストだけでKubernetesクラスタをデプロイできるようにする仕組みを紹介します。
