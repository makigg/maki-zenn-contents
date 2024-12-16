---
title: "誤自宅KaaSその2 ClusterClass設定編"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "おうちkubernetes", "clusterapi", "proxmox", "kubeadm"]
published: false
---

## はじめに

全3回のシリーズで、自宅にKaaS(Kubernetes as a Service)を構築する方法を解説しています。

- [Cluster APIインストール編](https://zenn.dev/articles/kaas-kubeadm-proxmox-part1)
- ClusterClass設定編 ←ココ
- [ストレージ設定編](https://zenn.dev/articles/kaas-kubeadm-proxmox-part3)

目指すべき全体像はこんな感じ。Management ClusterにCluster APIからKubernetesのクラスタが自動でデプロイされていきます。

![overview](/images/kaas-kubeadm-proxmox/overview.drawio.png)

前回、Cluster APIをインストールして、とりあえず、Kubernetesクラスタをプロビジョニングするところまでは行いました。しかし、その手順ではKaaS利用者がインフラ構成を意識しないといけないうえ、マニフェストが巨大でした。
:::details cluster01-gen.yaml (266行)
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

今回はClusterClassという仕組みを使って、24行のマニフェストでクラスタをデプロイできるようにします。

```yaml:cluster01.yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: cluster01
  namespace: kaas
spec:
  topology:
    class: kubeadm
    version: v1.30.5
    controlPlane:
      replicas: 1
    workers:
      machineDeployments:
      - class: default-worker
        name: cpu-node
        replicas: 1
    variables:
    - name: controlPlaneHost
      value: "192.168.10.210"
    - name: ipV4Addresses
      value:
      - "192.168.10.211-192.168.10.214"
```

ClusterClassはPVCに対するStorageClassのようなもので、ClusterClassにインフラに関する情報を書いておけば、Clusterを作成するときはそのClusterClassを指定するだけで、簡単にClusterを作成することができるようになります。

## ClusterClass設定編でやること

ClusterClassを定義して、ClusterClassからKubernetesクラスタをプロビジョニングします。

説明すること

- ProxmoxClusterClassとそれに必要なリソースの定義方法

説明しないこと

- ClusterResourceSetの設定方法 (次回)
- ClusterClassの運用方法
  - ClusterClassは一度作成すると変更できないようだが、バージョンアップとかどう対応していくのか？

## 今回設定するCluster APIのリソース

今回利用するCluster APIのリソースと利用した設定を記載します。他にも色々なことが設定できるのでしょうが、使った機能以外は調べていません。。。

ProxmoxMachineTemplate, KubeadmControlPlaneTemplate, KubeadmConfigTemplate, ProxmoxClusterTemplateはあくまでテンプレートなので、実際にClusterを作成するとそれぞれのテンプレートに応じたProxmoxMachine, KubeadmControlPlane, KubeadmConfig, ProxmoxClusterが作成されます。

### ProxmoxMachineTemplate

ProxmoxMachineTemplateは払いだすVMの設定です。

- CPU
- メモリ
- ディスク
- 複製元ゴールデンイメージ

### KubeadmControlPlaneTemplate

KubeadmControlPlaneTemplateはコントロールプレーンの設定です。OS上の設定+kubeadmの設定といったイメージです。ここに書かれた設定がcloud-initの設定ファイルに書かれてVM払い出し時に実行されます。

- kubeadm init/join時の設定
- ファイルの作成
- コマンド実行
- ユーザーの作成

### KubeadmConfigTemplate

KubeadmConfigTemplateはワーカーノードの設定です。OS上の設定+kubeadmの設定といったイメージです。ここに書かれた設定がcloud-initの設定ファイルに書かれてVM払い出し時に実行されます。

- kubeadm join時の設定
- ユーザーの作成
  - SSH公開鍵の配置

### ProxmoxClusterTemplate

ProxmoxClusterTemplateは、私の語彙だと一言で言い表すことができないです。以下のようなクラスタ全体の設定をします。

- コントロールプレーンのエンドポイント
  - ProxmoxClusterTemplateが直接KubernetesのノードにIPアドレスの設定をするというわけではありません。実際にkube-vipを使ってエンドポイントを設定する設定はKubeadmControlPlaneTemplateに記載する必要があります。
- 各ノードにdnsServerの設定
- 各ノードに割り当てるIPアドレスの範囲

### ClusterClass

ClusterClassはKubernetesクラスタのテンプレートです。インフラの構成やブートストラップの方法(kubeadm, RKE2など)をKaaS利用者に意識させずにKubernetesクラスタを払いだす仕組みを提供します。

ProxmoxMachineTemplate, KubeadmControlPlaneTemplate, KubeadmConfigTemplate, ProxmoxClusterTemplateを指定して、クラスタの構成を設定していくのですが、IPアドレスなどクラスタごとに変更したい設定をパッチする機能も提供されています。

## ClusterTopology機能を有効にする

ClusterClassを使うためにClusterTopologyを有効にします。[Cluster APIインストール編](https://zenn.dev/articles/kaas-kubeadm-proxmox-part1#management-clusterの構築)に書かれた手順通りCluster APIをインストールした方は、この手順はスキップして問題ありません。

Cluster APIのDeploymentの`args`に指定してある`--feature-gates`という名前の引数の設定項目を修正することで、ClusterTopology機能を有効にします。

```diff
-        - --feature-gates=MachinePool=true,ClusterResourceSet=false,ClusterTopology=false,RuntimeSDK=false,MachineSetPreflightChecks=false
+        - --feature-gates=MachinePool=true,ClusterResourceSet=false,ClusterTopology=true,RuntimeSDK=false,MachineSetPreflightChecks=false
```

設定対象

| Namespace                         | Deployment                                    |
| --------------------------------- | --------------------------------------------- |
| capi-system                       | capi-controller-manager                       |
| capmox-system                     | capmox-controller-manager                     |
| capi-kubeadm-control-plane-system | capi-kubeadm-control-plane-controller-manager |

## 各リソースの定義

`clusterctl generate cluster`コマンドで出力したマニフェストを参考にClusterClassの定義に必要な各リソースを定義していくのがおすすめです。

### ProxmoxMachineTemplate

ProxmoxMachineTemplateは払いだすVMの設定です。コントロールプレーンとワーカーでVMのスペックを分けたかったので、コントロールプレーン用とワーカー用の2つ定義します。

VMの情報を設定してくことになるので、以下の情報を改めて確認するとよいでしょう。

- CPU
- メモリ
- ディスク
- 複製元ゴールデンイメージ

#### コントロールプレーン

自動生成されたProxmoxMachineTemplateからCPU、メモリ、ディスクサイズだけ変更しました。

```yaml:machinetemplate-controlplane.yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: ProxmoxMachineTemplate
metadata:
  name: kubeadm-control-plane
  namespace: kaas
spec:
  template:
    spec:
      disks:
        bootVolume:
          disk: scsi0
          sizeGb: 50
      format: qcow2
      full: true
      network:
        default:
          bridge: vmbr0
          model: virtio
      numSockets: 1
      numCores: 2
      memoryMiB: 4096
      sourceNode: pve1
      templateID: 106
```

ちなみに、リンククローンを使いたくて`spec.template.spec.full`を`false`に設定してみましたが、`spec.template.spec.format`が指定されているとエラーになります。また、`spec.template.spec.format`を指定しなければ、どこかでクローン元のformatが入ってきてしまいます。そのため、フルコピーするしかなさそうです。

#### ワーカーノード

自動生成されたProxmoxMachineTemplateからディスクサイズだけ変更しました。

```yaml:machinetemplate-worker.yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: ProxmoxMachineTemplate
metadata:
  name: kubeadm-default-worker
  namespace: kaas
spec:
  template:
    spec:
      disks:
        bootVolume:
          disk: scsi0
          sizeGb: 50
      format: qcow2
      full: true
      memoryMiB: 16384
      network:
        default:
          bridge: vmbr0
          model: virtio
      numSockets: 4
      numCores: 2
      sourceNode: pve1
      templateID: 106
```

### KubeadmControlPlaneTemplate

KubeadmControlPlaneTemplateはコントロールプレーンの設定です。自動生成されたKubeadmControlPlaneを参考に作成していきます。

設定することは以下の通りです。全部自動生成された物から変更していません。

- kubeadm init/join時のオプション
- kube-vipのsuper-admin.conf対応スクリプト実行
- ユーザーの作成
  - SSH公開鍵の配置

また、もともとにマニフェストには以下の情報がありましたが、削除しています。

- kube-vipのStaticPodマニフェスト: コントロールプレーンの仮想IPアドレスがKubernetesクラスタごとに変更される値なので、ClusterClassからパッチで入れる。
- spec.machineTemplate: ClusterClassを使う場合、ClusterClassで指定するのでKubeadmControlPlaneTemplateには書かない
- spec.replicas: ClusterClassを使う場合、ClusterClassで指定するのでKubeadmControlPlaneTemplateには書かない
- spec.version: ClusterClassを使う場合、ClusterClassで指定するのでKubeadmControlPlaneTemplateには書かない

```yaml:kubeadmcontrolplanetemplate.yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlaneTemplate
metadata:
  name: kubeadm
  namespace: kaas
spec:
  template:
    spec:
      kubeadmConfigSpec:
        initConfiguration:
          nodeRegistration:
            kubeletExtraArgs:
              provider-id: "proxmox://{{ ds.meta_data.instance_id }}"
        joinConfiguration:
          nodeRegistration:
            kubeletExtraArgs:
              provider-id: "proxmox://{{ ds.meta_data.instance_id }}"
        files:
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
        preKubeadmCommands:
        - /etc/kube-vip-prepare.sh
        users:
        - name: root
          # 自分のPCからコントールプレーンのノードにSSHでログインできるよう公開鍵を設定できる
          sshAuthorizedKeys:
          - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDhQKh2EmK+5FWCw89zHw1aBewgp8S2L/mNlZr10Dx6Dv7x6HCt/ooLJiwIXz+Lfj9pB2ikLbZ14zOP635dCjnSRFky8yHJwH5RpvuLFFOLmoELRFFNJPq5cuVsDXjMod/1rDSajqXY0Ml0cmQIQKvR1Nr/+8rVWrvVjt0pwsWEPpBNPl0L/xgrHgaQIQj6JAQco14XnY0n6bXEOewswKDWmUxdPYoOgMH+dokjjKuvJAuBDhYiGYrD9XwClduyDORDt3OwmuZYDGEIQLBa4oXLF1J8zF0tIDDwYlN+vh2zx2HxLJ6kIk+2FgW+mnvVi3s3v174GqjeIqoMF8EJtcB+IH4ACqruPdjiAgKR581wStcdfjISjcP0AYe9t6L86uG1aIViWPWL7RPZNY4KR4RsYwk9kXGOA6LVr0VmfJvNslU7+oCb67XlxLRHUNBOMy8EfadIXDoIz3FUch1HW2BNbhkStrujrEqk1B5/zlsFRQ7shmI3ccHr9QByslxcFgs=
          - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCjp0UvoAQy+wUpfEHr1YVU04htWc0qnVhtSCeXVD1Q8kiz/7itIKKfKGvz+PFU9kNcQlrQaUxQ/lh9SUC0y/epVLcxm3TBI99HTeeWTrO1RLvDIR0CS3iC2USPMEX0jbhF2fUoE+Q+vaZArNsP/zR1fEV2Dc5HW/dgjcKwNN9Xr7p6BujkwOQXM8DdqcTTMIdRoZhisD7LGDfWkro1fcoo0eeMGgdcGx/DJLw0s840t1teVxE0Payt4LGMtg/jOCOCXRTzmvLDSfdxBjxYGhT2IK/GFPoC/L2frFaie0N2dRRGI0j4SBgNQbJudQanJvsuT35UkMUrxgkANyrbFQmVmcJhuwo3mbXRLaBSojpBv3AQ3eF5X1WKvTNcZ3H04E0acReUtQcy00CP+8rI1uZ+bhC6NxgR2g1GPKoJZA2EJp4LF9oqyczQzKz62dXGXV5I9c2Sf2Q+q4bVIQjKc79U5EdQnQkTEIyEfI+ib+XvclrT1c1M95uTw1cogOUBDhk=
```

### KubeadmConfigTemplate

KubeadmConfigTemplateはワーカーノードの設定です。自動生成されたKubeadmConfigTemplateから変更していません。

- kubeadm join時の設定
- ユーザーの作成
  - SSH公開鍵の配置

```yaml:kubeadmconfigtemplate.yaml
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfigTemplate
metadata:
  name: kubeadm-default-worker
  namespace: kaas
spec:
  template:
    spec:
      joinConfiguration:
        nodeRegistration:
          kubeletExtraArgs:
            provider-id: "proxmox://{{ ds.meta_data.instance_id }}"
      users:
      - name: root
        sshAuthorizedKeys:
        - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDhQKh2EmK+5FWCw89zHw1aBewgp8S2L/mNlZr10Dx6Dv7x6HCt/ooLJiwIXz+Lfj9pB2ikLbZ14zOP635dCjnSRFky8yHJwH5RpvuLFFOLmoELRFFNJPq5cuVsDXjMod/1rDSajqXY0Ml0cmQIQKvR1Nr/+8rVWrvVjt0pwsWEPpBNPl0L/xgrHgaQIQj6JAQco14XnY0n6bXEOewswKDWmUxdPYoOgMH+dokjjKuvJAuBDhYiGYrD9XwClduyDORDt3OwmuZYDGEIQLBa4oXLF1J8zF0tIDDwYlN+vh2zx2HxLJ6kIk+2FgW+mnvVi3s3v174GqjeIqoMF8EJtcB+IH4ACqruPdjiAgKR581wStcdfjISjcP0AYe9t6L86uG1aIViWPWL7RPZNY4KR4RsYwk9kXGOA6LVr0VmfJvNslU7+oCb67XlxLRHUNBOMy8EfadIXDoIz3FUch1HW2BNbhkStrujrEqk1B5/zlsFRQ7shmI3ccHr9QByslxcFgs=
        - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCjp0UvoAQy+wUpfEHr1YVU04htWc0qnVhtSCeXVD1Q8kiz/7itIKKfKGvz+PFU9kNcQlrQaUxQ/lh9SUC0y/epVLcxm3TBI99HTeeWTrO1RLvDIR0CS3iC2USPMEX0jbhF2fUoE+Q+vaZArNsP/zR1fEV2Dc5HW/dgjcKwNN9Xr7p6BujkwOQXM8DdqcTTMIdRoZhisD7LGDfWkro1fcoo0eeMGgdcGx/DJLw0s840t1teVxE0Payt4LGMtg/jOCOCXRTzmvLDSfdxBjxYGhT2IK/GFPoC/L2frFaie0N2dRRGI0j4SBgNQbJudQanJvsuT35UkMUrxgkANyrbFQmVmcJhuwo3mbXRLaBSojpBv3AQ3eF5X1WKvTNcZ3H04E0acReUtQcy00CP+8rI1uZ+bhC6NxgR2g1GPKoJZA2EJp4LF9oqyczQzKz62dXGXV5I9c2Sf2Q+q4bVIQjKc79U5EdQnQkTEIyEfI+ib+XvclrT1c1M95uTw1cogOUBDhk=
```

### ProxmoxClusterTemplate

ProxmoxClusterTemplateはクラスタ全体の設定です。自動生成されたProxmoxClusterから変更はありません。

ただし、以下2つはClusterClassからパッチを当てて、クラスタごとに変更されるようにします。であれば指定しなければよいだけなのですが、なぜか必須フィールドだったので、適当な値を指定しています。

- `spec.template.spec.controlPlaneEndpoint.host`
- `spec.template.spec.ipv4Config.addresses`

```yaml:proxmoxclustertemplate.yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: ProxmoxClusterTemplate
metadata:
  name: kubeadm
  namespace: kaas
spec:
  template:
    spec:
      allowedNodes:
      - pve1
      controlPlaneEndpoint:
        # should replaced by jsonPatches
        host: 192.168.10.210
        port: 6443
      dnsServers:
      - 8.8.8.8
      - 8.8.4.4
      ipv4Config:
        addresses:
        # should replaced by jsonPatches
        - "192.168.10.211-192.168.10.214"
        gateway: 192.168.10.1
        prefix: 24
```
### ClusterClass

ClusterClassはKubernetesクラスタのテンプレートです。今まで作成したリソースを指定し、Kubernetesクラスタを構成します。また、利用者に指定してもらう変数と、ProxmoxClusterTemplateとKubeadmControlPlaneTemplateに対するパッチを定義します。

:::details clusterclass.yaml
```yaml:clusterclass.yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: ClusterClass
metadata:
  name: kubeadm
  namespace: kaas
spec:
  ######## クラスタ構成 ########
  # コントロールプレーンの設定
  controlPlane:
    ref:
      apiVersion: controlplane.cluster.x-k8s.io/v1beta1
      kind: KubeadmControlPlaneTemplate
      name: kubeadm
      namespace: kaas
    machineInfrastructure:
      ref:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
        kind: ProxmoxMachineTemplate
        name: kubeadm-control-plane
        namespace: kaas
  # ワーカーノードの設定
  workers:
    # 今回は1つだけですが、複数のワーカークラスを定義することができます。
    # 例えばworker-gpu, worker-tinyなど用途に応じたテンプレートを用意したりします。
    machineDeployments:
    - class: default-worker # Cluster側でどのワーカークラスを使うか指定します。
      template:
        bootstrap:
          ref:
            apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
            kind: KubeadmConfigTemplate
            name: kubeadm-default-worker
            namespace: kaas
        infrastructure:
          ref:
            apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
            kind: ProxmoxMachineTemplate
            name: kubeadm-default-worker
            namespace: kaas
  # クラスタ全体の設定
  infrastructure:
    ref:
      apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
      kind: ProxmoxClusterTemplate
      name: kubeadm
      namespace: kaas

  ######## 変数定義 & パッチ ########
  # Clusterリソースで指定させる変数を2つ定義
  #   controlPlaneHost: コントロールプレーンのエンドポイントIPアドレス(kube-vipにより設定される仮想IPアドレス)
  #   ipV4Addresses: 各ノードに割り振られるIPアドレス
  variables:
  - name: controlPlaneHost
    required: true
    schema:
      openAPIV3Schema:
        type: string
        description: controlPlaneHost is Control Plane IP Address.
        example: "192.168.10.210"
  - name: ipV4Addresses
    required: true
    schema:
      openAPIV3Schema:
        type: array
        description: ipV4Addresses is IPv4 Address list.
        items:
          type: string
        example:
          - "192.168.10.211-192.168.10.214"
  # ProxmoxClusterTemplateとKubeadmControlPlaneTemplateへのパッチ
  patches:
  - name: ipv4Addresses
    definitions:
    - selector:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
        kind: ProxmoxClusterTemplate
        matchResources:
          infrastructureCluster: true
      jsonPatches:
      - op: replace
        path: /spec/template/spec/ipv4Config/addresses
        valueFrom:
          variable: ipV4Addresses
  - name: kubeVipPod
    definitions:
    - selector:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
        kind: ProxmoxClusterTemplate
        matchResources:
          infrastructureCluster: true
      jsonPatches:
      - op: add
        path: /spec/template/spec/controlPlaneEndpoint/host
        valueFrom:
          variable: controlPlaneHost
    - selector:
        apiVersion: controlplane.cluster.x-k8s.io/v1beta1
        kind: KubeadmControlPlaneTemplate
        matchResources:
          controlPlane: true
      jsonPatches:
      - op: add
        path: /spec/template/spec/kubeadmConfigSpec/files/-
        valueFrom:
          # templateの中身がyamlの場合、Unstructured構造体として読み取られる。
          # 例えば、contentだけをパッチしようとすると、contentはstringを期待しているのに、Unstructured構造体がパッチされようとするので、エラーになる。
          # そのため、このようにfilesフィールド全体に対してパッチをあてている。
          template: |
            owner: root:root
            path: /etc/kubernetes/manifests/kube-vip.yaml
            content: |
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
                    value: {{ .controlPlaneHost }}
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
```
:::

## ClusterClassのデプロイ

ClusterClassの依存リソースをデプロイします。

```shell
kubectl apply \
  -f machinetemplate-controlplane.yaml \
  -f machinetemplate-worker.yaml \
  -f kubeadmcontrolplanetemplate.yaml \
  -f kubeadmconfigtemplate.yaml \
  -f proxmoxclustertemplate.yaml
```

ClusterClassをデプロイします。

```shell
kubectl apply -f clusterclass.yaml
```

## Kubernetesクラスタのデプロイ

Clusterのマニフェストを作ります。

```yaml:cluster01.yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: cluster01
  namespace: kaas
spec:
  topology:
    # 先ほど作成したClusterClassを指定する
      class: kubeadm
    version: v1.30.5
    controlPlane:
      replicas: 1
    workers:
      machineDeployments:
      - class: default-worker # ClusterClass内で定義したワーカークラスを指定する
        name: cpu-node
        replicas: 1
    # ClusterClass内で定義した変数を指定する
    variables:
    - name: controlPlaneHost
      value: "192.168.10.210"
    - name: ipV4Addresses
      value:
      - "192.168.10.211-192.168.10.214"
```

Kubernetesクラスタを払いだします。

```shell
kubectl apply -f cluster01.yaml
```

以下のようにノードが払いだされているのを確認できます。

```shell
$ kubectl get machine
NAME                                   CLUSTER     NODENAME                               PROVIDERID                                       PHASE     AGE    VERSION
cluster01-cnsr2-hjqc8                  cluster01   cluster01-cnsr2-hjqc8                  proxmox://357a389e-e706-460a-a917-b334afc7ecd3   Running   2m8s   v1.30.5
cluster01-cpu-node-nkgn4-qzcp7-smcvd   cluster01   cluster01-cpu-node-nkgn4-qzcp7-smcvd   proxmox://48694587-ab3f-4002-86b7-cb89e73f7448   Running   2m9s   v1.30.5
```

## Kubernetesクラスタの動作確認

kubeconfigの取得

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


払いだされたクラスタのコンテキストに切り替えます。

```shell
kubectl config use-context cluster01-admin@cluster01
```

CNIがインストールされていないので、まだノードがNotReadyですね。

```shell
$ kubectl get node
NAME                                   STATUS     ROLES           AGE     VERSION
cluster01-cnsr2-hjqc8                  NotReady   control-plane   2m12s   v1.30.5
cluster01-cpu-node-nkgn4-qzcp7-smcvd   NotReady   <none>          60s     v1.30.5
```

CNIをインストールします。

```shell
cilium install
```

ノードがReadyになりました。

```shell
$ kubectl get node
NAME                                   STATUS   ROLES           AGE     VERSION
cluster01-cnsr2-hjqc8                  Ready    control-plane   3m47s   v1.30.5
cluster01-cpu-node-nkgn4-qzcp7-smcvd   Ready    <none>          2m35s   v1.30.5
```

## まとめ

ClusterClassを定義することでClusterに書くべきことがぐっと減って、Clusterの作成が簡単になり、よりKaaSっぽくなったのではないでしょうか？

しかし、まだStorageClassの設定がされていないので、クラスタ上でステートフルなアプリケーションを動かすことができません。

```shell
$ kubectl get storageclass
No resources found
```

次回はProxmoxのストレージからボリュームを作成するための設定をClusterResourceSetというリソースを使って設定していきます。

https://zenn.dev/articles/kaas-kubeadm-proxmox-part3
