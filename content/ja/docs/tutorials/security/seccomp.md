---
title: Restrict a Container's Syscalls with seccomp
content_type: tutorial
weight: 40
min-kubernetes-server-version: v1.22
---

<!-- overview -->

{{< feature-state for_k8s_version="v1.19" state="stable" >}}

seccomp(SECure COMputing mode)はLinuxカーネル2.7.12以降の機能です。
ユーザー空間からカーネルに対して発行できるシステムコールを制限することにより、プロセス権限のサンドボックスを構築することができます。
Kubernetesではノード上で読み込んだseccompプロファイルを、Podやコンテナに対して自動で適用することができます。

あなたのワークロードに必要な権限を特定することは、難しいことかもしれません。
このチュートリアルでは、まずローカルのKubernetesクラスターでseccompプロファイルを読み込むための方法を説明し、seccompプロファイルのPodへの適用方法について学んだ上で、コンテナプロセスに対して必要な権限のみを付与するためのseccompプロファイルを作成する方法を概観していきます。

## {{% heading "objectives" %}}  

* ノードでseccompプロファイルを読み込む方法を学ぶ
* seccompプロファイルをコンテナに適用する方法を学ぶ
* コンテナプロセスが生成するシステムコールの監査出力を確認する
* 存在しないプロファイルが指定された時の挙動を確認する
* seccompプロファイルの侵害を観測する
* きめ細やかなseccompプロファイルの作成方法を学ぶ
* コンテナランタイムのデフォルトseccompプロファイルの適用方法を学ぶ

## {{% heading "prerequisites" %}} 

このチュートリアルのステップを完遂するためには、[kind](/docs/tasks/tools/#kind)と[kubectl](/docs/tasks/tools/#kubectl)をインストールしておく必要があります。

このチュートリアルで利用するコマンドは、[Docker](https://www.docker.com/)をコンテナランタイムとして利用していることを前提としています。
(`kind`が作成するクラスターは内部的に異なるコンテナランタイムを利用する可能性があります)。
[Podman](https://podman.io/)を利用することもできますが、チュートリアルを完了するためには、特定の[手順](https://kind.sigs.k8s.io/docs/user/rootless/)に従う必要があるでしょう。

このチュートリアルでは、現時点(v1.25以降)でのベータ機能を利用する例をいくつか示しますが、その他の例についてはGAなseccomp関連機能です。
利用するKubernetesバージョンを対象としたクラスタの[正しい設定](https://kind.sigs.k8s.io/docs/user/quick-start/#setting-kubernetes-version)がなされていることを確認してください。

チュートリアル内では、サンプルをダウンロードするために`curl`を利用します。
この手順は、ほかの好きなツールを用いて実施してもかまいません。

{{< note >}}
Containerの`securityContext`に`privileged: true`が設定されているコンテナでは、seccompプロファイルを適用することができません。
特権コンテナは常に`Unconfined`な状態で動作します。
{{< /note >}} 

<!-- steps --> 

## サンプルのseccompプロファイルをダウンロードする {#download-profiles}

プロファイルの内容は後で確認しますので、まずはクラスターで読み込むためのseccompプロファイルを`profiles/`ディレクトリ内にダウンロードしましょう。

{{< tabs name="tab_with_code" >}}
{{< tab name="audit.json" >}}
{{% code_sample file="pods/security/seccomp/profiles/audit.json" %}}
{{< /tab >}}
{{< tab name="violation.json" >}}
{{% code_sample file="pods/security/seccomp/profiles/violation.json" %}}
{{< /tab >}}
{{< tab name="fine-grained.json" >}}
{{% code_sample file="pods/security/seccomp/profiles/fine-grained.json" %}}
{{< /tab >}}
{{< /tabs >}}

次のコマンドを実行してください:

```shell
mkdir ./profiles
curl -L -o profiles/audit.json https://k8s.io/examples/pods/security/seccomp/profiles/audit.json
curl -L -o profiles/violation.json https://k8s.io/examples/pods/security/seccomp/profiles/violation.json
curl -L -o profiles/fine-grained.json https://k8s.io/examples/pods/security/seccomp/profiles/fine-grained.json
ls profiles
```

最終的に３つのプロファイルが確認できるはずです:
```
audit.json  fine-grained.json  violation.json
```

## kindでローカルKubernetesクラスターを構築する

手順を簡略化するために、単一ノードクラスターを構築するために[kind](https://kind.sigs.k8s.io/)を利用します。
kindはKubernetesをDocker内で稼働させるため、クラスターの各ノードはコンテナの中にあります。
これにより、ノード上にファイルを展開するのと同じように、各コンテナのファイルシステムに対してファイルをマウントすることが可能です。

{{% code_sample file="pods/security/seccomp/kind.yaml" %}}

kindの設定サンプルをダウンロードして、`kind.yaml`の名前で保存してください:
```shell
curl -L -O https://k8s.io/examples/pods/security/seccomp/kind.yaml
```

ノードのコンテナイメージを設定する際には、特定のKubernetesバージョンを指定することもできます。
この設定方法の詳細については、kindのドキュメンテーションにおける[ノード](https://kind.sigs.k8s.io/docs/user/configuration/#nodes)の項目を参照してください。
このチュートリアルではKubernetes {{< param "version" >}}を使用することを前提とします。

ベータ機能として、`Unconfined`へのフォールバックを防ぐ目的で{{< glossary_tooltip text="container runtime" term_id="container-runtime" >}}が選んだデフォルトのseccompプロファイルを利用することもできます。
この機能を試したい場合、これ以降の手順に進む前に、[全ワークロードに対するデフォルトのseccompプロファイルとして`RuntimeDefault`を使用する](#enable-the-use-of-runtimedefault-as-the-default-seccomp-profile-for-all-workloads)を参照してください。

kindの設定ファイルを設置したら、kindクラスターを作成します:

```shell
kind create cluster --config=kind.yaml
```

Kubernetesクラスターが準備できたら、単一ノードクラスターが稼働しているDockerコンテナを特定してください:

```shell
docker ps
```


`kind-control-plane`という名前のコンテナが稼働していることが確認できるはずです。
出力は次のようになるでしょう:

```
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                       NAMES
6a96207fed4b        kindest/node:v1.18.2   "/usr/local/bin/entr…"   27 seconds ago      Up 24 seconds       127.0.0.1:42223->6443/tcp   kind-control-plane
```

このコンテナのファイルシステムを観察すると、`profiles/`ディレクトリがkubeletのデフォルトのseccompパスとして正しく読み込まれていることを確認できるはずです。
Pod内でコマンドを実行するために`docker exec`を使います:

```shell
# 6a96207fed4b を"docker ps"で確認したコンテナIDに変更してください。
docker exec -it 6a96207fed4b ls /var/lib/kubelet/seccomp/profiles
```

```
audit.json  fine-grained.json  violation.json
```

kind内で稼働しているkubeletからseccompプロファイルが利用できる状態にあることを確認しました。

## コンテナランタイムの標準seccompプロファイルを利用するPodを作成する

ほとんどのコンテナランタイムは、何を許可し/何を拒否するかについての標準的なシステムコールの論理集合を提供しています。

PodやContainerのセキュリティコンテキストでseccompタイプを`RuntimeDefault`に設定することにより、コンテナランタイムが提供するデフォルトのプロファイルを適用することができます。

{{< note >}}
`seccompDefault`の[設定](/docs/reference/config-api/kubelet-config.v1beta1/)を有効化している場合、
他のseccompプロファイルが存在しない場合であってもPodは`RuntimeDefault`seccompプロファイルを使用します。
`seccompDefault`が無効の場合のデフォルトは`Unconfined`です。
{{< /note >}}

Pod内の全てのContainerに対して`RuntimeDefault`seccompプロファイルを要求するマニフェストは次のようなものです:

{{% code_sample file="pods/security/seccomp/ga/default-pod.yaml" %}}

このPodを作成してみます:
```shell
kubectl apply -f https://k8s.io/examples/pods/security/seccomp/ga/default-pod.yaml
```

```shell
kubectl get pod default-pod
```

Podが正常に起動できていることを確認できるはずです:
```
NAME        READY   STATUS    RESTARTS   AGE
default-pod 1/1     Running   0          20s
```

次のセクションに進む前に、Podを削除します:

```shell
kubectl delete pod default-pod --wait --now
```

## システムコール監査のためのseccompプロファイルを利用するPodを作成する

最初に、新しいPodでプロセスの全システムコールを記録するための`audit.json`プロファイルを適用します。

このPodのためのマニフェストは次の通りです:

{{% code_sample file="pods/security/seccomp/ga/audit-pod.yaml" %}}

{{< note >}}
過去のバージョンのKubernetesでは、{{< glossary_tooltip text="annotations" term_id="annotation" >}}を用いてseccompの挙動を制御することが可能でした。
Kubernetes {{< skew currentVersion >}}におけるseccompの設定では`.spec.securityContext`フィールドのみをサポートしており、このチュートリアルではKubernetes {{< skew currentVersion >}}における手順を解説しています。
{{< /note >}}

クラスタ内にPodを作成します:

```shell
kubectl apply -f https://k8s.io/examples/pods/security/seccomp/ga/audit-pod.yaml
```

このプロファイルはシステムコールを禁止しませんので、Podは正常に起動するはずです。

```shell
kubectl get pod audit-pod
```

```
NAME        READY   STATUS    RESTARTS   AGE
audit-pod   1/1     Running   0          30s
```

このコンテナが露出するエンドポイントとやりとりするために、kindのコントロールプレーンコンテナの内部からこのエンドポイントへのアクセスを可能にするためのNodePort{{< glossary_tooltip text="Service" term_id="service" >}}を作成します。

```shell
kubectl expose pod audit-pod --type NodePort --port 5678
```

どのポートがノード上のServiceに割り当てられたのかを確認しましょう。

```shell
kubectl get service audit-pod
```

次のような出力が得られます:
```
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
audit-pod   NodePort   10.111.36.142   <none>        5678:32373/TCP   72s
```

ここまで来れば、kindのコントロールプレーンコンテナの内部からこのエンドポイントに対して、Serviceの露出するポートに対して`curl`で接続することができます。
`docker exec`を使って、コントロールプレーンコンテナに属するコンテナの中から`curl`を実行しましょう:

```shell
# 6a96207fed4b をコントロールプレーンコンテナのIDに変更し、32373を"kubectl get service"で確認したポート番号に変更してください。
docker exec -it 6a96207fed4b curl localhost:32373
```

```
just made some syscalls!
```

プロセスが走っていることを確認できるかと思いますが、実際にどんなシステムコールが発行されたのでしょうか。
このPodはローカルクラスターで稼働しているため、発行されたシステムコールを`/var/log/syslog`で確認することができるはずです。
新しいターミナルを開いて、`http-echo`が発行するシステムコールを`tail`してみましょう:

```shell
# あなたのマシンのログは"/var/log/syslog"以外の場所にあるかもしれません。
tail -f /var/log/syslog | grep 'http-echo'
```

すでに`http-echo`が発行したいくつかのシステムコールのログが見えているはずです。
コントロールプレーンコンテナ内から再度`curl`を実行すると、新たにログに追記された内容が出力されます。

例えば、次のような出力が得られるでしょう:
```
Jul  6 15:37:40 my-machine kernel: [369128.669452] audit: type=1326 audit(1594067860.484:14536): auid=4294967295 uid=0 gid=0 ses=4294967295 pid=29064 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=51 compat=0 ip=0x46fe1f code=0x7ffc0000
Jul  6 15:37:40 my-machine kernel: [369128.669453] audit: type=1326 audit(1594067860.484:14537): auid=4294967295 uid=0 gid=0 ses=4294967295 pid=29064 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=54 compat=0 ip=0x46fdba code=0x7ffc0000
Jul  6 15:37:40 my-machine kernel: [369128.669455] audit: type=1326 audit(1594067860.484:14538): auid=4294967295 uid=0 gid=0 ses=4294967295 pid=29064 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=202 compat=0 ip=0x455e53 code=0x7ffc0000
Jul  6 15:37:40 my-machine kernel: [369128.669456] audit: type=1326 audit(1594067860.484:14539): auid=4294967295 uid=0 gid=0 ses=4294967295 pid=29064 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=288 compat=0 ip=0x46fdba code=0x7ffc0000
Jul  6 15:37:40 my-machine kernel: [369128.669517] audit: type=1326 audit(1594067860.484:14540): auid=4294967295 uid=0 gid=0 ses=4294967295 pid=29064 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=0 compat=0 ip=0x46fd44 code=0x7ffc0000
Jul  6 15:37:40 my-machine kernel: [369128.669519] audit: type=1326 audit(1594067860.484:14541): auid=4294967295 uid=0 gid=0 ses=4294967295 pid=29064 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=270 compat=0 ip=0x4559b1 code=0x7ffc0000
Jul  6 15:38:40 my-machine kernel: [369188.671648] audit: type=1326 audit(1594067920.488:14559): auid=4294967295 uid=0 gid=0 ses=4294967295 pid=29064 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=270 compat=0 ip=0x4559b1 code=0x7ffc0000
Jul  6 15:38:40 my-machine kernel: [369188.671726] audit: type=1326 audit(1594067920.488:14560): auid=4294967295 uid=0 gid=0 ses=4294967295 pid=29064 comm="http-echo" exe="/http-echo" sig=0 arch=c000003e syscall=202 compat=0 ip=0x455e53 code=0x7ffc0000
```


各行の`syscall=`エントリに着目することで、`http-echo`プロセスが必要とするシステムコールを理解していくことができるでしょう。
このプロセスが利用する全てのシステムコールをを網羅するものではなさそうですが、このコンテナのseccompプロファイルの基礎とすることが可能です。

次のセクションに進む前にServiceとPodを削除します:

```shell
kubectl delete service audit-pod --wait
kubectl delete pod audit-pod --wait --now
```

## seccompプロファイルの侵害を引き起こすPodを作成する

デモとして、どのようなシステムコールも許可しないプロファイルをPodに適用してみましょう。

このデモのためのマニフェストは次の通りです:

{{% code_sample file="pods/security/seccomp/ga/violation-pod.yaml" %}}

クラスターにPodを作成してみます:

```shell
kubectl apply -f https://k8s.io/examples/pods/security/seccomp/ga/violation-pod.yaml
```

Podを作成しても、問題が発生します。
Podの状態を確認すると、起動に失敗していることが確認できるはずです。

```shell
kubectl get pod violation-pod
```

```
NAME            READY   STATUS             RESTARTS   AGE
violation-pod   0/1     CrashLoopBackOff   1          6s
```


直前の事例で見てきたように、`http-echo`プロセスはいくつかのシステムコールを必要とします。
ここでは`"defaultAction": "SCMP_ACT_ERRNO"`が設定されているため、あらゆるシステムコールの発行でseccompがエラーを発生させました。。

この構成はとてもセキュアですが、有意義なことは何もできないことを意味します。
実際にやりたいことは、ワークロードが必要とする権限のみを与えることです。

次のセクションに進む前にPodを削除します。

```shell
kubectl delete pod violation-pod --wait --now
```

## 必要なシステムコールのみを許可するseccompプロファイルを用いてPodを作成する

`fine-grained.json`プロファイルの内容を確認すると、`"defaultAction":"SCMP_ACT_LOG"`を設定していた最初の例でsyslogに現れた、いくつかのシステムコールが含まれていることに気づくでしょう。
今回のプロファイルでは`"defaultAction": "SCMP_ACT_ERRNO"`を設定していますが、`"action": "SCMP_ACT_ALLOW"`ブロックで明示的に一連のシステムコールを許可しています。
理論上は、コンテナが正常に稼働することに加えて、`syslog`へのメッセージの送信は確認できないことになります。

この事例で用いるマニフェストは次の通りです:

{{% code_sample file="pods/security/seccomp/ga/fine-pod.yaml" %}}

クラスタ内にPodを作成します:

```shell
kubectl apply -f https://k8s.io/examples/pods/security/seccomp/ga/fine-pod.yaml
```

```shell
kubectl get pod fine-pod
```

Podは正常に起動したことを示しているはずです:
```
NAME        READY   STATUS    RESTARTS   AGE
fine-pod   1/1     Running   0          30s
```

新しいターミナルを開いて、`http-echo`からのシステムコールに関するログエントリを`tail`で監視しましょう:

```shell
# あなたのマシンのログは"/var/log/syslog"以外の場所にあるかもしれません。
tail -f /var/log/syslog | grep 'http-echo'
```

次のPodをNodePort Serviceで公開します:

```shell
kubectl expose pod fine-pod --type NodePort --port 5678
```

どのポートがノード上のServiceに割り当てられたのかを確認します:

```shell
kubectl get service fine-pod
```

出力は次のようになるでしょう:
```
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
fine-pod    NodePort   10.111.36.142   <none>        5678:32373/TCP   72s
```

kindのコントロールプレーンコンテナの内部から、`curl`を用いてエンドポイントにアクセスします:

```shell
# 6a96207fed4b をコントロールプレーンコンテナのIDに変更し、32373を"kubectl get service"で確認したポート番号に変更してください。
docker exec -it 6a96207fed4b curl localhost:32373
```

```
just made some syscalls!
```

`syslog`には何も出力されないことを確認できるはずです。
なぜなら、このプロファイルは必要な全てのシステムコールを許可しており、一覧にないシステムコールが呼び出された時にのみエラーを発生させるように構成しているためです。
これはセキュリティの視点からすると理想的なシチュエーションといえますが、プログラムを解析するためにいくらかの労力を必要とします。
たくさんの労力を割かなくても、これに近いセキュリティが得られるシンプルな手法があったら嬉しいですね。

次のセクションに進む前にServiceとPodを削除します:

```shell
kubectl delete service fine-pod --wait
kubectl delete pod fine-pod --wait --now
```

## 全ワークロードに対するデフォルトのseccompプロファイルとして`RuntimeDefault`を使用する

{{< feature-state state="stable" for_k8s_version="v1.27" >}}

デフォルトのseccompプロファイルを利用するためには、この機能を利用したい全てのノードで`--seccomp-default`[コマンドラインフラグ](/docs/reference/command-line-tools-reference/kubelet)を用いてkubeletを起動する必要があります。

この機能を有効化すると、kubeletはコンテナランタイムが定義する`RuntimeDefault`のseccompプロファイルをデフォルトで使用するようになり、`Unconfined`モード(seccomp無効化)になることはありません。
この標準プロファイルは、ワークロードの機能をそのものを止めてしまうほどの、強力な標準セキュリティルールの束を提供することを目的としています。
標準プロファイルはコンテナランタイムやリリースバージョンによって異なる可能性があります。
例えば、CRI-Oとcontainerdで標準プロファイルを比較してみるとよいでしょう。

{{< note >}}
この機能を有効化しても、Kubernetesの`securityContext.seccompProfile`APIフィールドは変更されず、非推奨のアノテーションがワークロードに追加されることもありません。
これにより、ユーザーはワークロードの設定変更なしでロールバックすることが可能となっています。
[`crictl inspect`](https://github.com/kubernetes-sigs/cri-tools)のようなツールを使えば、コンテナが利用しているseccompプロファイルを確認できます。
{{< /note >}}

いくつかのワークロードで、他のワークロードよりも少ない量のシステムコール制限のみの適用が必要な場合があります。
つまり、`RuntimeDefault`を適用している場合であっても、これらのワークロードの実行は失敗する可能性があります。
このような障害を緩和するために、次のような対策を講じることができます:

- ワークロードを明示的に`Unconfined`として稼働させる。
- `SeccompDefault`機能をノードで無効化する。
また、機能を無効化したノードに対してワークロードが配置されていることを確認しておく。
- ワークロードを対象とするカスタムseccompプロファイル作成する。

実運用環境に近いクラスターに対してこの機能を展開する場合、Kubernetesプロジェクトはクラスター全体に対して変更をロールアウトする前に、一部ノードのみを対象にしてこのフィーチャーゲートを有効化し、ワークロードの実行を検証しておくことをお勧めします。

クラスターに対してとりうるアップグレード・ダウングレード戦略について更に詳細な情報について知りたい場合は、関連するKubernetes Enhancement Proposal (KEP)である[Enable seccomp by default](https://github.com/kubernetes/enhancements/tree/9a124fd29d1f9ddf2ff455c49a630e3181992c25/keps/sig-node/2413-seccomp-by-default#upgrade--downgrade-strategy)を参照してください。

//FIXME: ？仕様／挙動要確認？Kubernetes {{< skew currentVersion >}}では、固有のseccompプロファイルが定義されていないPodのspecがある場合にのみ適用するseccompプロファイルを設定することができます。
ただし、このデフォルト挙動を利用したい全てのノードでこの機能を有効化する必要があります。

稼働中のKubernetes {{< skew currentVersion >}}クラスターでこの機能を有効化したい場合、kubeletに`--seccomp-default`コマンドラインフラグを付与して起動するか、[Kubernetesの設定ファイル](/docs/tasks/administer-cluster/kubelet-config-file/)でこの機能を有効化する必要があります。
[kind](https://kind.sigs.k8s.io)でこのフィーチャーゲートを有効化する場合、`kind`が最低限必要なKubernetesバージョンを満たしていて、かつ[kindの設定ファイル](https://kind.sigs.k8s.io/docs/user/quick-start/#enable-feature-gates-in-your-cluster)で`SeccompDefault`を有効化していることを確認してください:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.28.0@sha256:9f3ff58f19dcf1a0611d11e8ac989fdb30a28f40f236f59f0bea31fb956ccf5c
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            seccomp-default: "true"
  - role: worker
    image: kindest/node:v1.28.0@sha256:9f3ff58f19dcf1a0611d11e8ac989fdb30a28f40f236f59f0bea31fb956ccf5c
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            seccomp-default: "true"
```

クラスターの準備ができたら、Podを走らせます:

```shell
kubectl run --rm -it --restart=Never --image=alpine alpine -- sh
```

このコマンドで標準のseccompプロファイルを紐付けられるはずです。
この結果を確認するためには、kindワーカー上のコンテナを確認するために`docker exec`経由で`crictl inspect`を実行します:

```shell
docker exec -it kind-worker bash -c \
    'crictl inspect $(crictl ps --name=alpine -q) | jq .info.runtimeSpec.linux.seccomp'
```

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_X86", "SCMP_ARCH_X32"],
  "syscalls": [
    {
      "names": ["..."]
    }
  ]
}
```

## {{% heading "whatsnext" %}}

Linuxのseccompについて更に学びたい場合は、次の記事を参考にすると良いでしょう:

* [A seccomp Overview](https://lwn.net/Articles/656307/)
* [Seccomp Security Profiles for Docker](https://docs.docker.com/engine/security/seccomp/)
