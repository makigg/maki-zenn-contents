---
title: "Kubestronautに認定されたので勉強方法など書いてく"
emoji: "📘"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "kubestronaut", "cks", "cka", "cncf"]
published: true
---

## はじめに

Kubestronautに認定されました。必要なKubernetesの認定資格5つを一気に取得したので、各資格の勉強方法や模擬試験と本番の得点比較などを書いていきます。これからKubernetes周りの資格勉強する方の参考になれば幸いです。

https://www.credly.com/badges/ab1bb86e-a7af-4cc9-a801-7ce68c091e85/public_url

なお、ここに書いてある情報は私が試験を受けた2024/7月-9月の情報です。最新の情報は公式ページを参照した方が確実です。

## Kubestronautとは

Kubernetes関連の認定資格5つに合格すると得られる称号です。詳しい条件などは公式ページを参照してください。

https://training.linuxfoundation.org/ja/resources/kubestronaut-program/

各認定試験の概要です。

- **CertifiedKubernetes Administrator (CKA)**:
  ハンズオン形式。Kubernetesの基本とクラスタ管理について問われます。
- **Certified Kubernetes Application Developer (CKAD)**:
  ハンズオン形式。Kubernetes上でのアプリケーション開発について問われます。
- **Certified Kubernetes Security Specialist (CKS)**:
  ハンズオン形式。Kubernetes周りのセキュリティについて問われます。CKAの上位資格にあたり、CKAに合格しないと受けることができません。
- **Kubernetes and Cloud Native Security Associate (KCSA)**:
  選択式。Kubernetesのセキュリティについて問われます。2024/9時点では英語しかありません。
- **Kubernetes and Cloud Native Associate (KCNA)**:
  選択式。Kubernetesの基本について問われます。

ハンズオン形式の試験では、実際にKubernetesのクラスタを触りながら、Kubernetesリソースの作成やトラブルシュートを行います。また、試験中、公式ドキュメントを参照することができます。

私の感触では、難易度は
KCNA < KCSA < CKA =< CKAD << CKS
という感じで、CKSが激ムズでした。

## 受けた順番と勉強時間

あまり覚えてないので大体ですが、ご参考までに勉強に使った時間を載せておきます。

| 認定 | 受けた日付 | 勉強期間 | 勉強時間 |
| ---- | ---------- | -------- | -------- |
| CKA  | 2024/7/5   | 2 ヶ月   | 120 時間 |
| CKAD | 2024/7/17  | 2 週間   | 30 時間  |
| CKS  | 2024/8/22  | 1 ヶ月   | 60 時間  |
| KCSA | 2024/9/5   | 2 週間   | 10 時間  |
| KCNA | 2024/9/9   | 3 日     | 0 時間   |

CKSを受ける前提がCKAになっているので、CKA --> CKSの順番は守る必要があるのですが、それ以外はどの順番で受けてもOKです。私は社内手続きに時間がかかったので、最初に難易度の高いハンズオン形式の試験から受けることにしました。

集中して勉強していた期間は1ヶ月=60時間くらいと想定して算出しています。CKAの勉強時間が他と比べて異常に長いのは、社内手続き中、ずっと勉強していたからです。KCSAはCKS取得したらすぐに受けるつもりでしたが、資格システムの不具合で期間が空いたので少し勉強しました。

## 模擬試験と本番の得点比較

試験は数日前までであればキャンセルできるので、模試を受けてみて手応えがなければキャンセルして勉強し直すこともできます。ご参考に擬試験の得点と試験本番での得点を載せておきます。

CKA, CKAD, CKSは認定試験を購入したらついてくる[Killer Shell](https://killer.sh/)の模試を受けました。KCSAはUdemyの[KCSA: Kubernetes & Cloud Native Security Associate EXAM-PREP](https://www.udemy.com/course/kcsa-kubernetes-cloud-native-security-associate-exam-prep)の模試を受けました。KCNAは模擬試験を受けてないので、データはありません。

| 認定 | 模擬試験             | 模試得点 | 必要得点 | 本番得点 |
| ---- | -------------------- | -------- | -------- | -------- |
| CKA  | Killer Shell         | 63       | 66       | 100      |
| CKAD | Killer Shell         | 62       | 66       | 93       |
| CKS  | Killer Shell         | 38       | 67       | 73       |
| KCSA | Udemy で購入した模試 | 83       | 75       | 90       |
| KCNA | NaN                  | NaN      | 75       | 93       |

Killer Shellは満点が100点ではないので、得点率に計算し直しています。

上記からもわかるようにKiller Shellの模擬試験は本番と比べてかなり難しく作られているので、合格に必要な得点に達していなかったとしても気にする必要はありません。そもそも、時間内に解き終わりませんでした。私は60点あれば、当日も安心できるのかなという感触があります。
逆にCKSのように30点台だと、次回からは試験を延期しようかなと考えています。

将来的に変わることもあるかもしれませんが、必要得点は私が受けたときのものを記載しています。

## 各認定試験の概要と勉強したこと

### Certified Kubernetes Administrator (CKA)

Certified Kubernetes Administrator (CKA)ではKubernetesの基本的な操作や、クラスタの管理方法が問われます。

具体的には以下のような内容が出題されます。詳しくは[カリキュラム](https://github.com/cncf/curriculum)を確認してみてください。

- Deploymentの作成/アップグレード
- kubeadmを使ったクラスタの構築
- Serviceの作成
- Storage Class, PVCの作成、Podへのアタッチ
- クラスタコンポーネントのトラブルシュート(例えば、Nodeが落ちているから修正せよみたいな内容)

試験時間は2時間です。

#### CKA対策

[KodeKloud](https://kodekloud.com/)を使って学習しました。KodeKloudはDevOpsスキルを動画学習+実機演習で習得できるサービスで、その中のCKA対策を受講しました。学習のコツは演習問題をしっかりやり込むことです。特に最後の模擬試験は繰り返し解いて、kubectlコマンドをスムーズにたたけるようにしました。

内容的にはKodeKloudだけでも十分な気もしますが、いかんせんKodeKloudは英語なのがきついです。Kubernetesの知識に不安がある方や、日本語での説明がほしい方は[Kubernetes完全ガイド](https://amzn.asia/d/8OHkYx0)を事前に読んだうえでKodeKloudの演習をやるなどすると良いかもしれません。

1週間前に資格試験についてきたKiller Shellの模擬試験を受けました。Killer Shellの解説は丁寧なので、模擬試験を受けたら解説を読みながら復習しました。

### Certified Kubernetes Application Developer (CKAD)

CKAと比べると、CKADではよりアプリケーション開発に特化した知識を問われます。例えば、サイドカーパターンとかを実装する必要があったりします。その代わり、クラスタの構築やkubeletの設定などは問われません。Kubernetes本体の機能だけでなく、HelmやKustomizeもカリキュラムに含まれているのでご注意ください。

具体的には以下のような内容が出題されます。詳しくは[カリキュラム](https://github.com/cncf/curriculum)を確認してみてください。

- Deployment, DaemonSet, CronJobなどの作成
- Deploymentのローリングアップグレード
- Dockerfileからコンテナをビルド
- requests/limitsの設定
- Ingressの設定
- Ephemeral Volumeの設定

試験時間は2時間です。

#### CKAD対策

[CKAD Exercises](https://github.com/dgkanatsios/CKAD-exercises)を使って学習しました。CKADExercisesは有志の方々が作ってくれたCKADの問題集です。こちらをスラスラ解けるようになるまで解きました。CKAD Exercisesの解説を見てもわからない場合、一度[Kubernetes完全ガイド](https://amzn.asia/d/8OHkYx0)を読むことをおすすめします。

また、1週間前に資格試験についてきたKiller Shellの模擬試験を受けました。

### Certified Kubernetes Security Specialist (CKS)

CKAの上位資格に当たり、CKAに認定されていないと受けることができません。
クラスタ、アプリケーションのセキュリティ対策について問われます。
Kubernetesの内容だけに留まらず、各種セキュリティツール(CIS benchmark, Trivy, kubesecなど)知識も問われます。

具体的には以下のような内容が出題されます。詳しくは[カリキュラム](https://github.com/cncf/curriculum)を確認してみてください。

- kubesecやCIS Benchで指摘された内容の是正
- AppArmorやseccompの設定、Podへの適用
- RuntimeClassの設定(gvisorやkata containersなど)
- SecurityContextsの設定
- 監査ログの設定
- システムコールの監視(Falcoなど)

試験時間は2時間です。

#### CKS対策

そもそもセキュリティの知識に不安があったため、まずは[Docker/Kubernetes開発・運用のためのセキュリティ実践ガイド](https://amzn.asia/d/8bN4gRO)を一通り読んで、基礎知識を学習しました。出版が2020年なので一分内容が古い箇所があります。また、CKSはハンズオン形式なので、実際に手を動かす訓練が必要でした。そのため、足りない部分は[KodeKloud](https://kodekloud.com/)補完します。書籍で一通りの基礎知識は知っていたので、KodeKloudでは知らないところだけ動画学習して、あとは演習を解きまくりました。普段からセキュリティ周りの業務やってないと意識しない設定や、使わないツールが多いため、覚えることが多かったです。

こちらも1週間前にKiller Shellを解いて解説を読みました。

CKSはKiller Shellを読んでも理解することが難しい箇所がちらほらあって、勉強が足りないかなと感じました。Kubernetesの認定試験は落ちても1度だけ受け直すこともできるので、当たって砕けろと思って受けたらギリギリ合格しました。

### Kubernetes and Cloud Native Security Associate (KCSA)

Kubernetesのセキュリティの基礎について問われます。選択式試験です。

具体的には以下のような内容が出題されます。詳しくは[カリキュラム](https://github.com/cncf/curriculum)を確認してみてください。

- クラウドネイティブセキュリティの4C (Cloud, Container, Cluster, Code)
- kube-apiの設定
- Secretの仕様
- Pod Security Admissions
- サプライチェーンセキュリティ
- CI/CDツール

#### KCSA対策

本当は最初はCKSを受けて仕上がった状態でKCSAを受けたかったのですが、資格システムの不具合で受けるまで２週間空いたので、Udemyで[KCSA: Kubernetes & Cloud Native Security Associate EXAM-PREP](https://www.udemy.com/course/kcsa-kubernetes-cloud-native-security-associate-exam-prep)という模試を購入して解きました。模試2回付きです。ほとんど解説はつきません。
CKSに合格した私であれば余裕であろうと油断していたのですが、セキュリティ一般のことも問われるということがここでわかりました。そこで、わからなかった問題を中心に改めて調べて学習し直しました。模試を受けておいてよかったです。CKSに合格したことがある人でも、一度模試を受けて見て、足りない知識を確認することをおすすめします。

また、2024/9時点では日本語で受けることができず、英語で受けることになります。日本語の情報のみで勉強していた人とかは、セキュリティ周りの英単語を叩き込んでおいたほうが良いでしょう。私はCKSの学習で英語の教材を使っていたので、ある程度の英単語は覚えることができていました。

### Kubernetes and Cloud Native Associate (KCNA)

Kubernetesを中心とするクラウドネイティブエコシステム全体の知識を問われます。Kubernetesのことが多いですが、それ以外にもGitOpsやCI/CDのことも問われます。

#### KCNAの対策

他の上位資格をパスしてるんだから余裕でしょと高をくくってやっていません。

> CKSに合格した私であれば余裕であろうと油断していたのですが、セキュリティ一般のことも問われるということがここでわかりました。そこで、わからなかった問題を中心に改めて調べて学習し直しました。模試を受けておいてよかったです。

どうやらKCSAで学んだことを忘れてしまったらしいです。

結果、合格することはできましたが、やはり、Kubernetesのペルソナのような一般知識を問うような問題など、わからない問題が数問ありました。CKA、CKADでは問われないけどKCNAでは問われるみたいな問題がそこそこあるので、満点を目指す人とかはちゃんと勉強することをおすすめします。

## その他Tips

### よく使うコマンドは`~/.bashrc`に書いておく

ハンズオン形式のテストではいかに早くマニフェストを作成できるかが重要になってきます。`kubectl`などよく使うコマンドはエイリアスとして、`~/.bashrc`に定義しておくことをおすすめします。

- `kubectl`のエイリアス。[認定試験のハンドブック](https://docs.linuxfoundation.org/tc-docs/certification/tips-cka-and-ckad#exam-technical-instructions)には全環境に設定してあると書かれているのですが、私がCKAを受けたときは設定されていなかったので、自分で[kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)を見て設定できるようにしておいたほうが良いでしょう。
  ```sh
  alias k=kubectl
  complete -o default -F __start_kubectl k
  ```
  以下のようにPod取得できるようになります。
  ```sh
  k get po
  ```
- Namespace切り替え
  ```sh
  alias kn='kubectl config set-context --current --namespace'
  ```
  以下のように操作対象のNamespaceを`kube-system`に変更できるようになります。
  ```sh
  kn kube-system
  ```
- YAML生成
  ```sh
  export dry='--dry-run=client -o yaml'
  ```
  以下のようなコマンドでYAMLファイルのテンプレートを出力できるようになります。
  ```sh
  k create deploy my-deployment --replicas=3 --image=nginx $dry
  ```

全部まとめると、私は以下を`~/.bashrc`に設定した状態から問題を解き始めていました。

```sh
alias k=kubectl                            # 設定済みのはず
complete -o default -F __start_kubectl k   # 設定済みのはず
alias kn='kubectl config set-context --current --namespace'
export dry='--dry-run=client -o yaml'
```

CKADは各コントロールプレーンにログインしてから問題を解かないと行けないので、メモ帳アプリに以下をメモして、流し込むようにしていました。

```sh
cat <<EOF >> ~/.bashrc
alias kn='kubectl config set-context --current --namespace'
export dry='--dry-run=client -o yaml'
EOF
```

### 使えたほうが良いツール

- **vimやemacs**: マニフェストの編集をするので必要です。
- **tmux**: 複数ターミナルを立ち上げてもいいのですが、ターミナル分割ツールがあると便利です。

CKAD環境の各コントロールプレーンにはtmuxが入ってなかったので、問題を解く前にインストールしていました。

### 書いてある日本語がわからない場合

翻訳が微妙でなんて書いてあるかわからない場合、英語に切り替えて読むと良いです。
