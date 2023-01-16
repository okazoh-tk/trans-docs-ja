---
layout: blog
title: "Protect Your Mission-Critical Pods From Eviction With PriorityClass"
date: 2023-01-12
slug: protect-mission-critical-pods-priorityclass
description: "Pod priority and preemption help to make sure that mission-critical pods are up in the event of a resource crunch by deciding order of scheduling and eviction."
---


**Author:** Sunny Bhambhani (InfraCloud Technologies)

<!--
Kubernetes has been widely adopted, and many organizations use it as their de-facto orchestration engine for running workloads that need to be created and deleted frequently.

Therefore, proper scheduling of the pods is key to ensuring that application pods are up and running within the Kubernetes cluster without any issues. This article delves into the use cases around resource management by leveraging the [PriorityClass](/docs/concepts/scheduling-eviction/pod-priority-preemption/#priorityclass) object to protect mission-critical or high-priority pods from getting evicted and making sure that the application pods are up, running, and serving traffic.
-->
Kubernetesは広く採用されており、多くの組織が頻繁に作成と削除が必要なワークロードを実行するためのデファクトオーケストレーションエンジンとして使用しています。

そのため、Kubernetesクラスタ内でアプリケーションポッドを問題なく稼働させるためには、ポッドの適切なスケジューリングが鍵となります。この記事では、[PriorityClass](/docs/concepts/scheduling-eviction/pod-priority-preemption/#priorityclass) オブジェクトを活用して、ミッションクリティカルまたは優先度の高いポッドを退避から守り、アプリケーションポッドを確実に稼働させて、トラフィックに対応するためのリソース管理周辺の使用事例を掘り下げます。

## Resource management in Kubernetes 

<!--
The control plane consists of multiple components, out of which the scheduler (usually the built-in [kube-scheduler](/docs/concepts/scheduling-eviction/kube-scheduler/)) is one of the components which is responsible for assigning a node to a pod.

Whenever a pod is created, it enters a "pending" state, after which the scheduler determines which node is best suited for the placement of the new pod.

In the background, the scheduler runs as an infinite loop looking for pods without a `nodeName` set that are [ready for scheduling](/docs/concepts/scheduling-eviction/pod-scheduling-readiness/). For each Pod that needs scheduling, the scheduler tries to decide which node should run that Pod.

If the scheduler cannot find any node, the pod remains in the pending state, which is not ideal.
-->

コントロールプレーンは複数のコンポーネントで構成されており、その中でもスケジューラ（通常は組み込みの[kube-scheduler](/docs/concepts/scheduling-eviction/kube-scheduler/)）は、ノードをポッドに割り当てる役割を持つコンポーネントの1つです。

Podが作成されるたびに「Pending」状態になり、その後スケジューラが新しいPodの配置に最適なノードを決定します。

バックグラウンドでは、スケジューラは無限ループとして実行され、 `nodeName` が設定されていない [スケジューリングの準備ができた](/docs/concepts/scheduling-eviction/pod-scheduling-readiness/) 状態の Pod を探します。スケジューリングが必要な各Podに対して、スケジューラはそのPodを実行すべきノードを決定しようとします。

スケジューラがどのノードも見つけられない場合、そのPodはPending状態のままですが、これは理想的ではありません。

<!--
{{< note >}}
To name a few, `nodeSelector` , `taints and tolerations` , `nodeAffinity` , the rank of nodes based on available resources (for example, CPU and memory), and several other criteria are used to determine the pod's placement.
{{< /note >}}

The below diagram, from point number 1 through 4, explains the request flow: 

{{< figure src=kube-scheduler.svg alt="A diagram showing the scheduling of three Pods that a client has directly created." title="Scheduling in Kubernetes">}}
-->
{{< 注釈 >}}
いくつか例を挙げると、`nodeSelector` 、 `taints and tolerations` 、 `nodeAffinity` 、利用可能なリソース（例えばCPUやメモリなど）に基づくノードのランク、その他いくつかの基準がポッドの配置を決定するのに使用されています。
{{< /note>}}

下図、1番から4番までのリクエストの流れを説明します。

{{< figure src=kube-scheduler.svg alt="クライアントが直接作成した3つのPodのスケジューリングを示す図" title="Scheduling in Kubernetes">}}


## Typical use cases

<!--
Below are some real-life scenarios where control over the scheduling and eviction of pods may be required.

1. Let's say the pod you plan to deploy is critical, and you have some resource constraints. An example would be the DaemonSet of an infrastructure component like Grafana Loki. The Loki pods must run before other pods can on every node. In such cases, you could ensure resource availability by manually identifying and deleting the pods that are not required or by adding a new node to the cluster. Both these approaches are unsuitable since the former would be tedious to execute, and the latter could involve an expenditure of time and money.


2. Another use case could be a single cluster that holds the pods for the below environments with associated priorities:
   - Production (`prod`):  top priority
   - Preproduction (`preprod`): intermediate priority
   - Development (`dev`): least priority

In the event of high resource consumption in the cluster, there is competition for CPU and memory resources on the nodes. While cluster-level autoscaling _may_ add more nodes, it takes time. In the interim, if there are no further nodes to scale the cluster, some Pods could remain in a Pending state, or the service could be degraded as they compete for resources. If the kubelet does evict a Pod from the node, that eviction would be random because the kubelet doesn’t have any special information about which Pods to evict and which to keep.

3. A third example could be a microservice backed by a queuing application or a database running into a resource crunch and the queue or database getting evicted. In such a case, all the other services would be rendered useless until the database can serve traffic again.

There can also be other scenarios where you want to control the order of scheduling or order of eviction of pods.
-->
以下は、Podのスケジューリングと退避の制御が必要とされる可能性のある実際のシナリオです。

1. デプロイ予定のPodがクリティカルで、リソースに制約があるとします。例としては、Grafana Loki のようなインフラストラクチャ コンポーネントの DaemonSet が挙げられます。Lokiポッドは、すべてのノードで他のポッドが実行される前に実行される必要があります。このような場合、必要のないポッドを手動で特定して削除するか、新しいノードをクラスターに追加することで、リソースの可用性を確保することができます。前者は実行が面倒であり、後者は時間と費用の支出を伴う可能性があるため、これらのアプローチはいずれも不向きです。


2. もう1つのユースケースは、以下の環境と関連する優先順位のポッドを保持する1つのクラスターです。
   - プロダクション (`prod`): 最優先事項
   - プリプロダクション (`preprod`): 中程度の優先度
   - 開発 (`dev`): 最も低い優先度

   クラスタのリソース消費量が多い場合、ノード上のCPUとメモリのリソースをめぐって競争が発生します。クラスタレベルのオートスケーリングでノードを追加することは可能ですが、それには時間がかかります。その間、クラスターをスケールするノードがさらにない場合、一部のPodがPending状態のままになったり、リソースを奪い合うためにサービスが低下したりする可能性があります。KubeletがノードからPodを退去させる場合、KubeletはどのPodを退去させてどのPodを維持するかについて特別な情報を持っていないため、その退避はランダムに行われます。

3. 3つ目の例として、キューイングアプリケーションやデータベースに支えられたマイクロサービスがリソース不足に陥り、キューやデータベースが退去させられることが考えられます。このような場合、データベースが再びトラフィックを処理できるようになるまで、他のすべてのサービスは役に立たなくなります。

また、Podのスケジューリングの順番や退避の順番を制御したいシナリオもありえます。


## PriorityClasses in Kubernetes

<!--
PriorityClass is a cluster-wide API object in Kubernetes and part of the `scheduling.k8s.io/v1` API group. It contains a mapping of the PriorityClass name (defined in `.metadata.name`) and an integer value (defined in `.value`). This represents the value that the scheduler uses to determine Pod's relative priority.

Additionally, when you create a cluster using kubeadm or a managed Kubernetes service (for example, Azure Kubernetes Service), Kubernetes uses PriorityClasses to safeguard the pods that are hosted on the control plane nodes. This ensures that critical cluster components such as CoreDNS and kube-proxy can run even if resources are constrained.

This availability of pods is achieved through the use of a special PriorityClass that ensures the pods are up and running and that the overall cluster is not affected.

```console
$ kubectl get priorityclass
NAME                      VALUE        GLOBAL-DEFAULT   AGE
system-cluster-critical   2000000000   false            82m
system-node-critical      2000001000   false            82m
```
-->
PriorityClass は Kubernetes のクラスタ共通 API オブジェクトで、`scheduling.k8s.io/v1` API グループの一部です。これは、PriorityClassの名前（`.metadata.name`で定義）と整数値（`.value`で定義）のマッピングを含んでいます。これは、スケジューラが Pod の相対的な優先度を決定するために使用する値を表します。

さらに、kubeadmまたはマネージドKubernetesサービス（たとえば、Azure Kubernetes Service）を使用してクラスタを作成すると、KubernetesはPriorityClassを使用して、コントロールプレーンノードにホストされるPodを保護するようにします。これにより、CoreDNSやkube-proxyなどの重要なクラスタコンポーネントは、リソースが制約された場合でも実行できるようになります。

このポッドの可用性は、ポッドが稼働し、クラスタ全体に影響が及ばないことを保証する特別なPriorityClassを使用することで実現されています。

```console
$ kubectl get priorityclass
NAME                      VALUE        GLOBAL-DEFAULT   AGE
system-cluster-critical   2000000000   false            82m
system-node-critical      2000001000   false            82m
```

<!--
The diagram below shows exactly how it works with the help of an example, which will be detailed in the upcoming section.

{{< figure src="decision-tree.svg" alt="A flow chart that illustrates how the kube-scheduler prioritizes new Pods and potentially preempts existing Pods" title="Pod scheduling and preemption">}}
-->

{{< figure src="decision-tree.svg" alt="kube-schedulerが新しいPodに優先順位を付け、既存のPodをプリエンプトする可能性があることを示すフローチャート" title="Pod scheduling and preemption">}}


### Pod priority and preemption

<!--
[Pod preemption](/docs/concepts/scheduling-eviction/pod-priority-preemption/#preemption) is a Kubernetes feature that allows the cluster to preempt pods (removing an existing Pod in favor of a new Pod) on the basis of priority. [Pod priority](/docs/concepts/scheduling-eviction/pod-priority-preemption/#pod-priority) indicates the importance of a pod relative to other pods while scheduling. If there aren't enough resources to run all the current pods, the scheduler tries to evict lower-priority pods over high-priority ones.

Also, when a healthy cluster experiences a node failure, typically, lower-priority pods get preempted to create room for higher-priority pods on the available node. This happens even if the cluster can bring up a new node automatically since pod creation is usually much faster than bringing up a new node.
-->
[Pod プリエンプション](/docs/concepts/scheduling-eviction/pod-priority-preemption/#preemption) はKubernetesの機能で、クラスタが優先順位に基づいてPodをプリエンプト（既存のPodを削除して新しいPodを優先）できるようにするものです。[Pod priority](/docs/concepts/scheduling-eviction/pod-priority-preemption/#pod-priority) は、スケジューリング中に他のPodと比較したPodの重要度を示すものです。現在のすべてのPodを実行するのに十分なリソースがない場合、スケジューラは優先度の高いPodよりも低いPodを退避させようとします。

また、健全なクラスタにノード障害が発生した場合、通常は優先度の低いポッドが先行して、利用可能なノードに優先度の高いポッドのためのスペースを確保します。これは、クラスタが新しいノードを自動的に起動できる場合でも発生します。通常、ポッドの作成は新しいノードを起動するよりもはるかに高速だからです。

### PriorityClass requirements

<!--
Before you set up PriorityClasses, there are a few things to consider.

1. Decide which PriorityClasses are needed. For instance, based on environment, type of pods, type of applications, etc.
2. The default PriorityClass resource for your cluster. The pods without a `priorityClassName` will be treated as priority 0.
3. Use a consistent naming convention for all PriorityClasses.
4. Make sure that the pods for your workloads are running with the right PriorityClass.
-->
PriorityClassを設定する前に、いくつか考慮すべき点があります。

1. どのPriorityClassが必要かを決める。例えば、環境、Podの種類、アプリケーションの種類などに基づきます
2. クラスタのデフォルトのPriorityClassリソース。`priorityClassName`がないPodは優先度0として扱われます。
3. すべてのPriorityClassで一貫した命名規則を使用する。
4. ワークロード用のポッドが正しいPriorityClassで実行されていることを確認します。

## PriorityClass hands-on example

<!--
Let’s say there are 3 application pods: one for prod, one for preprod, and one for development. Below are three sample YAML manifest files for each of those.
-->
例えば、prod 用、preprod 用、development 用の 3 つのアプリケーションポッドがあるとします。以下は、それぞれのポッドに対応する 3 つの YAML マニフェストファイルの例です。


```yaml
---
# development
apiVersion: v1
kind: Pod
metadata:
  name: dev-nginx
  labels:
    env: dev
spec:
  containers:
  - name: dev-nginx
    image: nginx
    resources:
      requests:
        memory: "256Mi"
         cpu: "0.2"
      limits:
        memory: ".5Gi"
        cpu: "0.5"
```

```yaml
---
# preproduction
apiVersion: v1
kind: Pod
metadata:
  name: preprod-nginx
  labels:
    env: preprod
spec:
  containers:
  - name: preprod-nginx
    image: nginx
    resources:
      requests:
        memory: "1.5Gi"
        cpu: "1.5"
      limits:
        memory: "2Gi"
        cpu: "2"
```

```yaml
---
# production
apiVersion: v1
kind: Pod
metadata:
  name: prod-nginx
  labels:
    env: prod
spec:
  containers:
  - name: prod-nginx
    image: nginx
    resources:
      requests:
        memory: "2Gi"
        cpu: "2"
      limits:
        memory: "2Gi"
        cpu: "2"
```

<!--
You can create these pods with the `kubectl create -f <FILE.yaml>` command, and then check their status
using the `kubectl get pods` command. You can see if they are up and look ready to serve traffic:
-->
これらのポッドは `kubectl create -f <FILE.yaml>` コマンドで作成し、`kubectl get pods` コマンドでその状態を確認できます。ポッドが起動し、トラフィックを提供する準備ができているかどうかを確認できます。

```console
$ kubectl get pods --show-labels
NAME            READY   STATUS    RESTARTS   AGE   LABELS
dev-nginx       1/1     Running   0          55s   env=dev
preprod-nginx   1/1     Running   0          55s   env=preprod
prod-nginx      0/1     Pending   0          55s   env=prod
```

<!--
Bad news. The pod for the Production environment is still Pending and isn't serving any traffic.

Let's see why this is happening:
-->
悪いニュースです。Production環境のPodはまだPendingで、トラフィックを提供していません。

なぜこのようなことが起こっているのか見てみましょう。

```console
$ kubectl get events
...
...
5s          Warning   FailedScheduling   pod/prod-nginx      0/2 nodes are available: 1 Insufficient cpu, 2 Insufficient memory.
```
<!--
In this example, there is only one worker node, and that node has a resource crunch.

Now, let's look at how PriorityClass can help in this situation since prod should be given higher priority than the other environments.
-->
この例では、ワーカーノードが 1 つしかなく、そのノードがリソース不足に陥っています。

ここで、PriorityClass がこのような状況でどのように役立つかを見てみましょう。

## PriorityClass API

<!--
Before creating PriorityClasses based on these requirements, let's see what a basic manifest for a PriorityClass looks like and outline some prerequisites:
-->
これらの要件に基づいて PriorityClass を作成する前に、PriorityClass の基本的なマニフェストがどのようなものかを確認し、前提条件の概要を説明します。

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: PRIORITYCLASS_NAME
value: 0 # any integer value between -1000000000 to 1000000000 
description: >-
  (Optional) description goes here!
globalDefault: false # or true. Only one PriorityClass can be the global default.
```

<!--
Below are some prerequisites for PriorityClasses:

- The name of a PriorityClass must be a valid DNS subdomain name.
- When you make your own PriorityClass, the name should not start with `system-`, as those names are reserved by Kubernetes itself (for example, they are used for two built-in PriorityClasses).
- Its absolute value should be between -1000000000 to 1000000000 (1 billion).
- Larger numbers are reserved by PriorityClasses such as `system-cluster-critical` (this Pod is critically important to the cluster) and `system-node-critical` (the node critically relies on this Pod).
  `system-node-critical` is a higher priority than `system-cluster-critical`, because a   cluster-critical Pod can only work well if the node where it is running has all its node-level critical requirements met.
- There are two optional fields:
  - `globalDefault`: When true, this PriorityClass is used for pods where a `priorityClassName` is not specified.
    Only one PriorityClass with `globalDefault` set to true can exist in a cluster.  
  If there is no PriorityClass defined with globalDefault set to true, all the pods with no priorityClassName defined will be treated with 0 priority (i.e. the least priority).
  - `description`: A string with a meaningful value so that people know when to use this PriorityClass.

{{< note >}}
Adding a PriorityClass with `globalDefault` set to `true` does not mean it will apply the same to the existing pods that are already running. This will be applicable only to the pods that came into existence after the PriorityClass was created.
{{< /note >}}
-->

以下は、PriorityClass の前提条件です。

- PriorityClassの名前は、有効なDNSサブドメイン名である必要があります。
- PriorityClassを自作する場合、名前は`system-`で始めてはいけません。これらの名前はKubernetes自体によって予約されているからです（例えば、2つの組み込みPriorityClassに使用されています）。
- その絶対値は-1000000000から1000000000（10億）の間であるべきです。
- より大きな数値は `system-cluster-critical` (このPodはクラスタにとって決定的に重要) や `system-node-critical` (ノードが決定的にこのPodに依存している) などのPriorityClassで予約されています。
  `system-node-critical` は `system-cluster-critical` よりも高い優先度です。クラスタクリティカルな Pod は、それが動作するノードがノードレベルのクリティカルな要件をすべて満たしている場合にのみ正常に動作することができるからです。
- オプションのフィールドが2つあります。
  - `globalDefault`: true の場合、この PriorityClass は `priorityClassName` が指定されていない Pod に対して使用されます。
    クラスタ内には、 `globalDefault` が true に設定された PriorityClass を 1 つだけ存在させることができます。 
  globalDefault が true に設定された PriorityClass が定義されていない場合、 priorityClassName が定義されていないすべてのポッドは優先度 0 (つまり、最も低い優先度) で扱われることになります。
  - `description`: このPriorityClassをいつ使えばいいのかがわかるように、意味のある文字列を指定します。

{{< note >}}
`globalDefault` を `true` に設定して PriorityClass を追加しても、すでに起動している既存のポッドに同じものが適用されるわけではありません。PriorityClass が作成された後に誕生したポッドにのみ適用されます。
{{< /note >}}

### PriorityClass in action

<!--
Here's an example. Next, create some environment-specific PriorityClasses:
-->
以下はその例です。次に、環境固有のPriorityClassをいくつか作成します。

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: dev-pc
value: 1000000
globalDefault: false
description: >-
  (Optional) This priority class should only be used for all development pods.
```

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: preprod-pc
value: 2000000
globalDefault: false
description: >-
  (Optional) This priority class should only be used for all preprod pods.
```

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: prod-pc
value: 4000000
globalDefault: false
description: >-
  (Optional) This priority class should only be used for all prod pods.
```

<!--
Use `kubectl create -f <FILE.YAML>` command to create a pc and `kubectl get pc` to check its status.
-->
`kubectl create -f <FILE.YAML>`コマンドを使ってpcを作成し、`kubectl get pc`でそのステータスを確認します。

```console
$ kubectl get pc
NAME                      VALUE        GLOBAL-DEFAULT   AGE
dev-pc                    1000000      false            3m13s
preprod-pc                2000000      false            2m3s
prod-pc                   4000000      false            7s
system-cluster-critical   2000000000   false            82m
system-node-critical      2000001000   false            82m
```

<!--
The new PriorityClasses are in place now. A small change is needed in the pod manifest or pod template (in a ReplicaSet or Deployment). In other words, you need to specify the priority class name at `.spec.priorityClassName` (which is a string value).

First update the previous production pod manifest file to have a PriorityClass assigned, then delete the Production pod and recreate it. You can't edit the priority class for a Pod that already exists.

In my cluster, when I tried this, here's what happened.
First, that change seems successful; the status of pods has been updated:
-->
新しい PriorityClass は現在、所定の位置にあります。PodマニフェストまたはPodテンプレート（ReplicaSetまたはDeployment内）に小さな変更が必要です。つまり、`.spec.priorityClassName`で優先クラス名を指定する必要があります（文字列値です）。

まず、以前のProduction Podのマニフェストファイルを更新してPriorityClassが割り当てられるようにし、Production Podを削除して再作成します。すでに存在するPodの優先度クラスを編集することはできません。

私のクラスタでは、これを試したところ、次のような結果になりました。
まず、その変更は成功したようで、Podのステータスが更新されています。

```console
$ kubectl get pods --show-labels
NAME            READY   STATUS    	RESTARTS   AGE   LABELS
dev-nginx       1/1     Terminating	0          55s   env=dev
preprod-nginx   1/1     Running   	0          55s   env=preprod
prod-nginx      0/1     Pending   	0          55s   env=prod
```

<!--
The dev-nginx pod is getting terminated. Once that is successfully terminated and there are enough resources for the prod pod, the control plane can schedule the prod pod:
-->
dev-nginx ポッドは終了されます。それが正常に終了し、prodポッドに十分なリソースがある場合、コントロールプレーンはprodポッドをスケジュールすることができます。

```console
Warning   FailedScheduling   pod/prod-nginx    0/2 nodes are available: 1 Insufficient cpu, 2 Insufficient memory.
Normal    Preempted          pod/dev-nginx     by default/prod-nginx on node node01
Normal    Killing            pod/dev-nginx     Stopping container dev-nginx
Normal    Scheduled          pod/prod-nginx    Successfully assigned default/prod-nginx to node01
Normal    Pulling            pod/prod-nginx    Pulling image "nginx"
Normal    Pulled             pod/prod-nginx    Successfully pulled image "nginx"
Normal    Created            pod/prod-nginx    Created container prod-nginx
Normal    Started            pod/prod-nginx    Started container prod-nginx
```

## Enforcement

<!--
When you set up PriorityClasses, they exist just how you defined them. However, people (and tools) that make changes to your cluster are free to set any PriorityClass, or to not set any PriorityClass at all.
However, you can use other Kubernetes features to make sure that the priorities you wanted are actually applied.

As an alpha feature, you can define a [ValidatingAdmissionPolicy](/blog/2022/12/20/validating-admission-policies-alpha/) and a ValidatingAdmissionPolicyBinding so that, for example, Pods that go into the `prod` namespace must use the `prod-pc` PriorityClass.
With another ValidatingAdmissionPolicyBinding you ensure that the `preprod` namespace uses the `preprod-pc` PriorityClass, and so on.
In *any* cluster, you can enforce similar controls using external projects such as [Kyverno](https://kyverno.io/) or [Gatekeeper](https://open-policy-agent.github.io/gatekeeper/), through validating admission webhooks.

However you do it, Kubernetes gives you options to make sure that the PriorityClasses are used how you wanted them to be, or perhaps just to [warn](https://open-policy-agent.github.io/gatekeeper/website/docs/violations/#warn-enforcement-action) users when they pick an unsuitable option.
-->
PriorityClassを設定すると、その定義通りに存在するようになります。しかし、クラスタに変更を加える人（およびツール）は、任意のPriorityClassを設定することも、PriorityClassをまったく設定しないことも自由にできます。
しかし、他のKubernetesの機能を使って、望んだ優先順位が実際に適用されることを確認することができます。

アルファ版の機能として、[ValidatingAdmissionPolicy](/blog/2022/12/20/validating-admission-policies-alpha/) と ValidatingAdmissionPolicyBinding を定義すると、例えば `prod` 名前空間に入る Pod には `prod-pc` PriorityClass が必要だとすることが可能です。
別の ValidatingAdmissionPolicyBinding を使って、 `preprod` ネームスペースが `preprod-pc` PriorityClass を使用するようにします。
どのクラスタでも、[Kyverno](https://kyverno.io/) や [Gatekeeper](https://open-policy-agent.github.io/gatekeeper/) などの外部プロジェクトを使って、アドミッションコントローラーのWebhooksで同様の制御を行うことができます。

どのように行うにせよ、KubernetesはPriorityClassesがあなたが望むように使用されることを確認するオプションを提供します。あるいは、ユーザーが不適切なオプションを選択したときに[警告](https://open-policy-agent.github.io/gatekeeper/website/docs/violations/#warn-enforcement-action)を表示するだけでもよいでしょう。

## Summary

<!--
The above example and its events show you what this feature of Kubernetes brings to the table, along with several scenarios where you can use this feature. To reiterate, this helps ensure that mission-critical pods are up and available to serve the traffic and, in the case of a resource crunch, determines cluster behavior.

It gives you some power to decide the order of scheduling and order of [preemption](/docs/concepts/scheduling-eviction/pod-priority-preemption/#preemption) for Pods. Therefore, you need to define the PriorityClasses sensibly.
For example, if you have a cluster autoscaler to add nodes on demand, make sure to run it with the `system-cluster-critical` PriorityClass. You don't want to get in a situation where the autoscaler has been preempted and there are no new nodes coming online.

If you have any queries or feedback, feel free to reach out to me on [LinkedIn](http://www.linkedin.com/in/sunnybhambhani).
-->
上記の例とそのイベントは、Kubernetesのこの機能がもたらすものを、この機能を使用できるいくつかのシナリオとともに示しています。繰り返しになりますが、これはミッションクリティカルなポッドが稼働してトラフィックに対応できることを保証し、リソースが不足した場合にはクラスタの挙動を決定するのに役立ちます。

Podsのスケジューリングの順番や[プリエンプション](/docs/concepts/scheduling-eviction/pod-priority-preemption/#preemption)の順番を決定する力を与えてくれるのです。そのため、PriorityClassを感覚的に定義しておく必要があります。
例えば、オンデマンドでノードを追加するクラスタオートスケーラがある場合、必ず `system-cluster-critical` という PriorityClass で実行します。自動スケーラが先取りされ、新しいノードがオンラインにならないような状況は避けたいものです。

ご質問やご意見がありましたら、[LinkedIn](http://www.linkedin.com/in/sunnybhambhani)までお気軽にご連絡ください。
