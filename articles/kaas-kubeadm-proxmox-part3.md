---
title: "誤自宅KaaSその3 ストレージ設定編"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "おうちkubernetes", "clusterapi", "proxmox", "kubeadm"]
published: true
---

## はじめに

この記事は全3回で、自宅にKaaS(Kubernetes as a Service)を構築する方法を解説するシリーズの3つめです。

- [Cluster APIインストール編](https://zenn.dev/makidev/articles/kaas-kubeadm-proxmox-part1)
- [ClusterClass設定編](https://zenn.dev/makidev/articles/kaas-kubeadm-proxmox-part2)
- **ストレージ設定編** ←ココ

目指すべき全体像はこんな感じ。Management ClusterにCluster APIからKubernetesのクラスタが自動でデプロイされていきます。

![overview](/images/kaas-kubeadm-proxmox/overview.drawio.png)

前回、ClusterClassを使って、Kubernetesクラスタを払いだすKaaSを作りました。

今回は、ClusterResourceSetという機能を使ってラベルを付けるだけで払いだしたKubernetesクラスタからProxmoxのストレージを使えるように設定していきます。

```yaml:cluster01.yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: cluster01
  namespace: kaas
  labels:
    csi-proxmox: "true" # ラベル追加
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

## ストレージ設定編でやること

Clusterにラベルを追加するだけでProxmox用のストレージの設定が行われるようにします。

やること

- ゴールデンイメージの変更
- ClusterResourceSetを使ったストレージ設定自動化
- 適当なPostgreSQLのクラスタをデプロイ

やらないこと

- CNIのインストール自動化

## 使う技術の説明

### Proxmox CSI Plugin

[Container Storage Interface](https://github.com/container-storage-interface/spec)(CSI)は、コンテナからストレージを利用するための仕様です。CSIはKubernetesと統合されており、PersistentVolumeClaimに対して、ボリュームのプロビジョニング、マウントなどの操作を行います。CSIはあくまで仕様であり、各ストレージ用のCSI Driverが実装されています。今回はProxmoxのストレージ用CSI DriverであるProxmox CSI Pluginを使います。

https://github.com/sergelogvinov/proxmox-csi-plugin

### ClusterResourceSet

ClusterResourceSetは、ConfigMapやSecret内に定義したマニフェストを払いだしたKubernetesクラスタにデプロイすることができます。

https://cluster-api.sigs.k8s.io/tasks/experimental-features/cluster-resource-set

## 手順

### ゴールデンイメージの変更

[ドキュメント](https://github.com/sergelogvinov/proxmox-csi-plugin/blob/main/docs/install.md)にも記載がある通り、Proxmox CSI Pluginを使うためには、VMのSCSIコントローラーはVirtIO SCSI single、もしくはVirtIO SCSIに設定されている必要があります。VMクローン時にSCSIコントローラーを指定する方法がないかProxmoxMachineのパラメーターを一通り眺めてみたのですが、それらしいものはありませんでした。



ゴールデンイメージのSCSI ControllerがVirtIO SCSI single、もしくはVirtIO SCSIに設定されているか確認します。[Cluster APIインストール編の手順](https://zenn.dev/makidev/articles/kaas-kubeadm-proxmox-part1#packerでゴールデンイメージを作る)通りにゴールデンイメージを作った方は既に設定されているはずです。

![](/images/kaas-kubeadm-proxmox/proxmox-scsi-controller.drawio.png)

設定されていない場合、Proxmoxの画面からゴールデンイメージを修正するか、Packerを使ってゴールデンイメージを作り直してください。以下の手順はPackerを使ってゴールデンイメージを作成しなおす方法です。

[image-builder](https://github.com/kubernetes-sigs/image-builder.git)の設定ファイルを変更して、ゴールデンイメージのVMのSCSIコントローラーをVirtIO SCSI singleに設定します。

```diff
--- a/images/capi/packer/proxmox/packer.json.tmpl
+++ b/images/capi/packer/proxmox/packer.json.tmpl
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
```

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

新しくVM IDが発行されるので、それをProxmoxMachineTemplateに反映させてください。

### ClusterResourceSetを有効化する

ClusterResourceSetを有効にします。[Cluster APIインストール編](https://zenn.dev/makidev/articles/kaas-kubeadm-proxmox-part1#management-clusterの構築)に書かれた手順通りCluster APIをインストールした場合、既に設定されています。

Namespace capi-systemのDeployment capi-controller-managerの`args`に指定してある`--feature-gates`という名前の引数の設定項目を修正することで、ClusterResourceSet機能を有効にします。

```diff
-        - --feature-gates=MachinePool=true,ClusterResourceSet=false,ClusterTopology=true,RuntimeSDK=false,MachineSetPreflightChecks=false
+        - --feature-gates=MachinePool=true,ClusterResourceSet=true,ClusterTopology=true,RuntimeSDK=false,MachineSetPreflightChecks=false
```

### Proxmox CSI Plugin用トークンを発行する

Proxmox CSI Pluginが使うトークンを作成します。私はProxmoxのホスト上で実行しましたが、画面から操作しても問題ありません。

```shell
pveum role add CSI -privs "VM.Audit VM.Config.Disk Datastore.Allocate Datastore.AllocateSpace Datastore.Audit"
pveum user add kubernetes-csi@pve
pveum aclmod / -user kubernetes-csi@pve -role CSI
pveum user token add kubernetes-csi@pve csi -privsep 0
```

```shell
┌──────────────┬──────────────────────────────────────┐
│ key          │ value                                │
╞══════════════╪══════════════════════════════════════╡
│ full-tokenid │ kubernetes-csi@pve!csi               │
├──────────────┼──────────────────────────────────────┤
│ info         │ {"privsep":"0"}                      │
├──────────────┼──────────────────────────────────────┤
│ value        │ xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx │
└──────────────┴──────────────────────────────────────┘
```

`full-tokenid`と`value`はProxmox CSI Pluginに書くので控えておきます。

### Proxmox CSI Pluginのマニフェストを作成する

ClusterResourceSetで読み込むマニフェストなので、マニフェストを直接定義するのではなく、ConfigMapやSecretの中にマニフェストを定義していきます。

Proxmox CSI Pluginの設定です。

```yaml:resources/config.yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-proxmox-config
  namespace: kaas
type: addons.cluster.x-k8s.io/resource-set
stringData:
  config.yaml: |
    apiVersion: v1
    kind: Secret
    metadata:
      name: proxmox-csi-plugin
      namespace: csi-proxmox
    stringData:
      config.yaml: |
        clusters:
            # Proxmox VE APIのURL
          - url: https://192.168.10.138:8006/api2/json
            # 証明書をチェックしない
            insecure: true
            # 先ほど生成したトークンID
            token_id: "kubernetes-csi@pve!kaas"
            # 先ほど生成したトークン
            token_secret: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
            # ProxmoxのClusterを指定する
            region: region1
```

Proxmox CSI Plugin本体です。GitHubにある[マニフェスト](https://github.com/sergelogvinov/proxmox-csi-plugin/blob/main/docs/deploy/proxmox-csi-plugin-release.yml)をそのまま入れてます。

:::details resources/proxmox-csi-plugin.yaml
```yaml:resources/proxmox-csi-plugin.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: proxmox-csi-plugin
  namespace: kaas
data:
  # https://github.com/sergelogvinov/proxmox-csi-plugin/blob/main/docs/deploy/proxmox-csi-plugin-release.yml
  proxmox-csi-plugin-release.yml: |
    ---
    # Source: proxmox-csi-plugin/templates/namespace.yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: csi-proxmox
      labels:
        pod-security.kubernetes.io/enforce: privileged
        pod-security.kubernetes.io/audit: baseline
        pod-security.kubernetes.io/warn: baseline
    ---
    # Source: proxmox-csi-plugin/templates/serviceaccount.yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: proxmox-csi-plugin-controller
      namespace: csi-proxmox
      labels:
        helm.sh/chart: proxmox-csi-plugin-0.2.13
        app.kubernetes.io/name: proxmox-csi-plugin
        app.kubernetes.io/instance: proxmox-csi-plugin
        app.kubernetes.io/version: "v0.8.2"
        app.kubernetes.io/managed-by: Helm
    ---
    # Source: proxmox-csi-plugin/templates/serviceaccount.yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: proxmox-csi-plugin-node
      namespace: csi-proxmox
      labels:
        helm.sh/chart: proxmox-csi-plugin-0.2.13
        app.kubernetes.io/name: proxmox-csi-plugin
        app.kubernetes.io/instance: proxmox-csi-plugin
        app.kubernetes.io/version: "v0.8.2"
        app.kubernetes.io/managed-by: Helm
    ---
    # Source: proxmox-csi-plugin/templates/controller-clusterrole.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: proxmox-csi-plugin-controller
      namespace: csi-proxmox
      labels:
        helm.sh/chart: proxmox-csi-plugin-0.2.13
        app.kubernetes.io/name: proxmox-csi-plugin
        app.kubernetes.io/instance: proxmox-csi-plugin
        app.kubernetes.io/version: "v0.8.2"
        app.kubernetes.io/managed-by: Helm
    rules:
      - apiGroups: [""]
        resources: ["persistentvolumes"]
        verbs: ["get", "list", "watch", "create", "patch", "delete"]
      - apiGroups: [""]
        resources: ["persistentvolumeclaims"]
        verbs: ["get", "list", "watch", "update"]
      - apiGroups: [""]
        resources: ["persistentvolumeclaims/status"]
        verbs: ["patch"]
      - apiGroups: [""]
        resources: ["events"]
        verbs: ["get","list", "watch", "create", "update", "patch"]

      - apiGroups: ["storage.k8s.io"]
        resources: ["storageclasses"]
        verbs: ["get", "list", "watch"]
      - apiGroups: ["storage.k8s.io"]
        resources: ["csinodes"]
        verbs: ["get", "list", "watch"]
      - apiGroups: [""]
        resources: ["nodes"]
        verbs: ["get", "list", "watch"]

      - apiGroups: ["storage.k8s.io"]
        resources: ["volumeattachments"]
        verbs: ["get", "list", "watch", "patch"]
      - apiGroups: ["storage.k8s.io"]
        resources: ["volumeattachments/status"]
        verbs: ["patch"]
    ---
    # Source: proxmox-csi-plugin/templates/node-clusterrole.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: proxmox-csi-plugin-node
      namespace: csi-proxmox
      labels:
        helm.sh/chart: proxmox-csi-plugin-0.2.13
        app.kubernetes.io/name: proxmox-csi-plugin
        app.kubernetes.io/instance: proxmox-csi-plugin
        app.kubernetes.io/version: "v0.8.2"
        app.kubernetes.io/managed-by: Helm
    rules:
      - apiGroups:
          - ""
        resources:
          - nodes
        verbs:
          - get
    ---
    # Source: proxmox-csi-plugin/templates/controller-rolebinding.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: proxmox-csi-plugin-controller
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: proxmox-csi-plugin-controller
    subjects:
      - kind: ServiceAccount
        name: proxmox-csi-plugin-controller
        namespace: csi-proxmox
    ---
    # Source: proxmox-csi-plugin/templates/node-rolebinding.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: proxmox-csi-plugin-node
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: proxmox-csi-plugin-node
    subjects:
      - kind: ServiceAccount
        name: proxmox-csi-plugin-node
        namespace: csi-proxmox
    ---
    # Source: proxmox-csi-plugin/templates/controller-role.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: proxmox-csi-plugin-controller
      namespace: csi-proxmox
      labels:
        helm.sh/chart: proxmox-csi-plugin-0.2.13
        app.kubernetes.io/name: proxmox-csi-plugin
        app.kubernetes.io/instance: proxmox-csi-plugin
        app.kubernetes.io/version: "v0.8.2"
        app.kubernetes.io/managed-by: Helm
    rules:
      - apiGroups: ["coordination.k8s.io"]
        resources: ["leases"]
        verbs: ["get", "watch", "list", "delete", "update", "create"]

      - apiGroups: ["storage.k8s.io"]
        resources: ["csistoragecapacities"]
        verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["get"]
      - apiGroups: ["apps"]
        resources: ["replicasets"]
        verbs: ["get"]
    ---
    # Source: proxmox-csi-plugin/templates/controller-rolebinding.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: proxmox-csi-plugin-controller
      namespace: csi-proxmox
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: proxmox-csi-plugin-controller
    subjects:
      - kind: ServiceAccount
        name: proxmox-csi-plugin-controller
        namespace: csi-proxmox
    ---
    # Source: proxmox-csi-plugin/templates/node-deployment.yaml
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: proxmox-csi-plugin-node
      namespace: csi-proxmox
      labels:
        helm.sh/chart: proxmox-csi-plugin-0.2.13
        app.kubernetes.io/name: proxmox-csi-plugin
        app.kubernetes.io/instance: proxmox-csi-plugin
        app.kubernetes.io/version: "v0.8.2"
        app.kubernetes.io/managed-by: Helm
    spec:
      updateStrategy:
        type: RollingUpdate
      selector:
        matchLabels:
          app.kubernetes.io/name: proxmox-csi-plugin
          app.kubernetes.io/instance: proxmox-csi-plugin
          app.kubernetes.io/component: node
      template:
        metadata:
          labels:
            app.kubernetes.io/name: proxmox-csi-plugin
            app.kubernetes.io/instance: proxmox-csi-plugin
            app.kubernetes.io/component: node
        spec:
          priorityClassName: system-node-critical
          enableServiceLinks: false
          serviceAccountName: proxmox-csi-plugin-node
          securityContext:
            runAsUser: 0
            runAsGroup: 0
          containers:
            - name: proxmox-csi-plugin-node
              securityContext:
                privileged: true
                capabilities:
                  drop:
                  - ALL
                  add:
                  - SYS_ADMIN
                  - CHOWN
                  - DAC_OVERRIDE
                seccompProfile:
                  type: RuntimeDefault
              image: "ghcr.io/sergelogvinov/proxmox-csi-node:v0.8.2"
              imagePullPolicy: IfNotPresent
              args:
                - "-v=5"
                - "--csi-address=unix:///csi/csi.sock"
                - "--node-id=$(NODE_NAME)"
              env:
                - name: NODE_NAME
                  valueFrom:
                    fieldRef:
                      fieldPath: spec.nodeName
              resources:
                {}
              volumeMounts:
                - name: socket
                  mountPath: /csi
                - name: kubelet
                  mountPath: /var/lib/kubelet
                  mountPropagation: Bidirectional
                - name: dev
                  mountPath: /dev
                - name: sys
                  mountPath: /sys
            - name: csi-node-driver-registrar
              securityContext:
                allowPrivilegeEscalation: false
                capabilities:
                  drop:
                  - ALL
                readOnlyRootFilesystem: true
                seccompProfile:
                  type: RuntimeDefault
              image: "registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.9.4"
              imagePullPolicy: IfNotPresent
              args:
                - "-v=5"
                - "--csi-address=unix:///csi/csi.sock"
                - "--kubelet-registration-path=/var/lib/kubelet/plugins/csi.proxmox.sinextra.dev/csi.sock"
              volumeMounts:
                - name: socket
                  mountPath: /csi
                - name: registration
                  mountPath: /registration
              resources:
                requests:
                  cpu: 10m
                  memory: 16Mi
            - name: liveness-probe
              securityContext:
                allowPrivilegeEscalation: false
                capabilities:
                  drop:
                  - ALL
                readOnlyRootFilesystem: true
                seccompProfile:
                  type: RuntimeDefault
              image: "registry.k8s.io/sig-storage/livenessprobe:v2.11.0"
              imagePullPolicy: IfNotPresent
              args:
                - "-v=5"
                - "--csi-address=unix:///csi/csi.sock"
              volumeMounts:
                - name: socket
                  mountPath: /csi
              resources:
                requests:
                  cpu: 10m
                  memory: 16Mi
          volumes:
            - name: socket
              hostPath:
                path: /var/lib/kubelet/plugins/csi.proxmox.sinextra.dev/
                type: DirectoryOrCreate
            - name: registration
              hostPath:
                path: /var/lib/kubelet/plugins_registry/
                type: Directory
            - name: kubelet
              hostPath:
                path: /var/lib/kubelet
                type: Directory
            - name: dev
              hostPath:
                path: /dev
                type: Directory
            - name: sys
              hostPath:
                path: /sys
                type: Directory
          tolerations:
            - effect: NoSchedule
              key: node.kubernetes.io/unschedulable
              operator: Exists
            - effect: NoSchedule
              key: node.kubernetes.io/disk-pressure
              operator: Exists
    ---
    # Source: proxmox-csi-plugin/templates/controller-deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: proxmox-csi-plugin-controller
      namespace: csi-proxmox
      labels:
        helm.sh/chart: proxmox-csi-plugin-0.2.13
        app.kubernetes.io/name: proxmox-csi-plugin
        app.kubernetes.io/instance: proxmox-csi-plugin
        app.kubernetes.io/version: "v0.8.2"
        app.kubernetes.io/managed-by: Helm
    spec:
      replicas: 1
      strategy:
        type: RollingUpdate
        rollingUpdate:
          maxUnavailable: 1
      selector:
        matchLabels:
          app.kubernetes.io/name: proxmox-csi-plugin
          app.kubernetes.io/instance: proxmox-csi-plugin
          app.kubernetes.io/component: controller
      template:
        metadata:
          annotations:
            checksum/config: c69436cb1e16c36ff708b1003d3ca4c6ee6484d2524e2ba7d9b68f473acaa1ca
          labels:
            app.kubernetes.io/name: proxmox-csi-plugin
            app.kubernetes.io/instance: proxmox-csi-plugin
            app.kubernetes.io/component: controller
        spec:
          priorityClassName: system-cluster-critical
          enableServiceLinks: false
          serviceAccountName: proxmox-csi-plugin-controller
          securityContext:
            fsGroup: 65532
            fsGroupChangePolicy: OnRootMismatch
            runAsGroup: 65532
            runAsNonRoot: true
            runAsUser: 65532
          hostAliases:
            []
          initContainers:
            []
          containers:
            - name: proxmox-csi-plugin-controller
              securityContext:
                allowPrivilegeEscalation: false
                capabilities:
                  drop:
                  - ALL
                readOnlyRootFilesystem: true
                seccompProfile:
                  type: RuntimeDefault
              image: "ghcr.io/sergelogvinov/proxmox-csi-controller:v0.8.2"
              imagePullPolicy: IfNotPresent
              args:
                - "-v=5"
                - "--csi-address=unix:///csi/csi.sock"
                - "--cloud-config=/etc/proxmox/config.yaml"
              ports:
              resources:
                requests:
                  cpu: 10m
                  memory: 16Mi
              volumeMounts:
                - name: socket-dir
                  mountPath: /csi
                - name: cloud-config
                  mountPath: /etc/proxmox/
            - name: csi-attacher
              securityContext:
                allowPrivilegeEscalation: false
                capabilities:
                  drop:
                  - ALL
                readOnlyRootFilesystem: true
                seccompProfile:
                  type: RuntimeDefault
              image: "registry.k8s.io/sig-storage/csi-attacher:v4.4.4"
              imagePullPolicy: IfNotPresent
              args:
                - "-v=5"
                - "--csi-address=unix:///csi/csi.sock"
                - "--timeout=3m"
                - "--leader-election"
                - "--default-fstype=ext4"
              volumeMounts:
                - name: socket-dir
                  mountPath: /csi
              resources: 
                requests:
                  cpu: 10m
                  memory: 16Mi
            - name: csi-provisioner
              securityContext:
                allowPrivilegeEscalation: false
                capabilities:
                  drop:
                  - ALL
                readOnlyRootFilesystem: true
                seccompProfile:
                  type: RuntimeDefault
              image: "registry.k8s.io/sig-storage/csi-provisioner:v3.6.4"
              imagePullPolicy: IfNotPresent
              args:
                - "-v=5"
                - "--csi-address=unix:///csi/csi.sock"
                - "--timeout=3m"
                - "--leader-election"
                - "--default-fstype=ext4"
                - "--feature-gates=Topology=True"
                - "--enable-capacity"
                - "--capacity-ownerref-level=2"
              env:
                - name: NAMESPACE
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.namespace
                - name: POD_NAME
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.name
              volumeMounts:
                - name: socket-dir
                  mountPath: /csi
              resources: 
                requests:
                  cpu: 10m
                  memory: 16Mi
            - name: csi-resizer
              securityContext:
                allowPrivilegeEscalation: false
                capabilities:
                  drop:
                  - ALL
                readOnlyRootFilesystem: true
                seccompProfile:
                  type: RuntimeDefault
              image: "registry.k8s.io/sig-storage/csi-resizer:v1.9.4"
              imagePullPolicy: IfNotPresent
              args:
                - "-v=5"
                - "--csi-address=unix:///csi/csi.sock"
                - "--timeout=3m"
                - "--handle-volume-inuse-error=false"
                - "--leader-election"
              volumeMounts:
                - name: socket-dir
                  mountPath: /csi
              resources: 
                requests:
                  cpu: 10m
                  memory: 16Mi
            - name: liveness-probe
              securityContext:
                allowPrivilegeEscalation: false
                capabilities:
                  drop:
                  - ALL
                readOnlyRootFilesystem: true
                seccompProfile:
                  type: RuntimeDefault
              image: "registry.k8s.io/sig-storage/livenessprobe:v2.11.0"
              imagePullPolicy: IfNotPresent
              args:
                - "-v=5"
                - "--csi-address=unix:///csi/csi.sock"
              volumeMounts:
                - name: socket-dir
                  mountPath: /csi
              resources: 
                requests:
                  cpu: 10m
                  memory: 16Mi
          volumes:
            - name: socket-dir
              emptyDir: {}
            - name: cloud-config
              secret:
                secretName: proxmox-csi-plugin
          topologySpreadConstraints:
            - maxSkew: 1
              topologyKey: kubernetes.io/hostname
              whenUnsatisfiable: DoNotSchedule
              labelSelector:
                matchLabels:
                  app.kubernetes.io/name: proxmox-csi-plugin
                  app.kubernetes.io/instance: proxmox-csi-plugin
                  app.kubernetes.io/component: controller
    ---
    # Source: proxmox-csi-plugin/templates/csidriver.yaml
    apiVersion: storage.k8s.io/v1
    kind: CSIDriver
    metadata:
      name: csi.proxmox.sinextra.dev
    spec:
      attachRequired: true
      podInfoOnMount: true
      storageCapacity: true
      volumeLifecycleModes:
      - Persistent
```
:::

最後にStorageClassです。指定可能なパラメーターは[こちら](https://github.com/sergelogvinov/proxmox-csi-plugin/blob/main/docs/options.md)に記載があります。

```yaml:resources/storageclass.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: storage-class-proxmox
  namespace: kaas
data:
  storage-class.yaml: |
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: proxmox-data
      annotations:
        storageclass.kubernetes.io/is-default-class: "true"
    provisioner: csi.proxmox.sinextra.dev
    allowVolumeExpansion: true
    volumeBindingMode: WaitForFirstConsumer
    reclaimPolicy: Delete
    parameters:
      csi.storage.k8s.io/fstype: ext4
      # 指定するストレージはDatacenter > Storageから選びます。
      storage: local-lvm
```

作ったマニフェストをManagement ClusterにApplyします。

```shell
kubectl apply -f resources/config.yaml -f resources/proxmox-csi-plugin.yaml -f resources/storageclass.yaml
```

### ClusterResourceSetの設定

```yaml:clusterresourceset.yaml
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterResourceSet
metadata:
  name: csi-proxmox
  namespace: kaas
spec:
  # ClusterResourceSetのresourcesが変更された場合、その変更をクラスターに反映させるならReconcileを選びます。最初に一度だけ適用したい場合はApplyOnceを選びます。
  strategy: Reconcile
  # 適用対象のクラスタにつけるラベル
  # ここに指定されたラベルをClusterリソースにつけるとそのクラスタにマニフェストが適用されます。
  clusterSelector:
    matchLabels:
      csi-proxmox: "true"
  # 適用されるマニフェストが格納されたConfigMapとSecretを指定します。
  resources:
    - name: proxmox-csi-plugin
      kind: ConfigMap
    - name: csi-proxmox-config
      kind: Secret
    - name: storage-class-proxmox
      kind: ConfigMap
```

ClusterResourceSetをManagement ClusterにApplyします。

```shell
kubectl apply -f clusterresourceset.yaml
```

### クラスタにストレージ設定を適用する

既に払いだされているcluster01にClusterResourceSetで定義したラベルを付けます。新しくClusterリソースを作成しても問題ありません。

```shell
kubectl label cluster cluster01 csi-proxmox="true"
```

すると、以下のようなClusterResourceSetBindingというリソースが作成されます。ここでハッシュ値が管理されて、差分があったことを判定できる仕組みっぽいですね。

```shell
$ kubectl get clusterresourcesetbinding cluster01 -o yaml
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterResourceSetBinding
metadata:
  creationTimestamp: "2024-12-16T14:45:26Z"
  generation: 2
  name: cluster01
  namespace: kaas
  ownerReferences:
  - apiVersion: addons.cluster.x-k8s.io/v1beta1
    kind: ClusterResourceSet
    name: csi-proxmox
    uid: bde7aa13-9266-4792-a605-e37013c52757
  resourceVersion: "4942140"
  uid: 1b57c316-88b2-4a34-a359-113c0f8a0def
spec:
  bindings:
  - clusterResourceSetName: csi-proxmox
    resources:
    - applied: true
      hash: sha256:0fc06e0572c5108445cea2ae7f564b820e99612f2d9fa1469c874679886dc2e6
      kind: ConfigMap
      lastAppliedTime: "2024-12-16T14:45:27Z"
      name: proxmox-csi-plugin
    - applied: true
      hash: sha256:1d4d16e2df095be725981f8673a25be2c535b2b830a922ea4e8115392f9bf858
      kind: Secret
      lastAppliedTime: "2024-12-16T14:45:27Z"
      name: csi-proxmox-config
    - applied: true
      hash: sha256:e8277c1651dd091857a782b0958136e44206c8bbe17f242e450c2b9f1b144387
      kind: ConfigMap
      lastAppliedTime: "2024-12-16T14:45:27Z"
      name: storage-class-proxmox
  clusterName: cluster01
```

### ノードにラベルを付ける

Proxmox CSI Pluginの仕様で各ノードにtopologyラベルがついていないといけません。しかし、そこまで自動化する良い方法が思いつかなかったので、手動で設定します。(ご存じの方いらっしゃったら教えていただけるとうれしいです)

cluster01のコンテキストに切り替えます。

```shell
kubectl config use-context cluster01-admin@cluster01
```

各ノードにTopologyラベルを付けます。

```shell
kubectl label node --all topology.kubernetes.io/region=region1 topology.kubernetes.io/zone=pve1
```

## 動作確認

設定が適用されているのか確認していきます。

Proxmox CSI Pluginが正常に動いているか確認。

```shell
$ kubectl get po -n csi-proxmox
NAME                                             READY   STATUS    RESTARTS       AGE
proxmox-csi-plugin-controller-5c8f4bd897-g6h89   5/5     Running   0              32m
proxmox-csi-plugin-node-5hv7w                    3/3     Running   10 (10m ago)   32m
```

StorageClassが作成されているか確認。

```shell
$ kubectl get sc
NAME                     PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
proxmox-data (default)   csi.proxmox.sinextra.dev   Delete          WaitForFirstConsumer   true                   32m
```

問題なさそうですね。

## ステートフルアプリを動かしてみる

PostgreSQLを動かしてみます。

```shell
helm install postgres oci://registry-1.docker.io/bitnamicharts/postgresql
```

PVCも問題なく使えてそうです。

```shell
$ kubectl get sts
NAME                  READY   AGE
postgres-postgresql   1/1     90s
$ kubectl get pvc
NAME                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
data-postgres-postgresql-0   Bound    pvc-f00c0430-586f-4c41-ba08-c7bc8b5ae6da   8Gi        RWO            proxmox-data   <unset>                 85s
```

Proxmoxの画面からもボリュームが作成されていることを確認できます。

![](/images/kaas-kubeadm-proxmox/proxmox-pvc.drawio.png)

## まとめ

ClusterResourceSetを使ってKubernetesクラスでストレージを使えるようにしました。ClusterResourceSetは任意のマニフェストをCluster APIで作成したKubernetesクラスタに適用することができます。同じ要領でCNIの設定も行えます。

全3回のシリーズで皆様のご自宅にKaaSを構築する方法を解説しました。正直、どこに需要があるのかもわかりませんが、何かしらの参考になると嬉しいです。
