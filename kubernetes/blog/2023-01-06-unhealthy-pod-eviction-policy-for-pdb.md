---
layout: blog
title: "Kubernetes 1.26: Eviction policy for unhealthy pods guarded by PodDisruptionBudgets"
date: 2023-01-06
slug: "unhealthy-pod-eviction-policy-for-pdbs"
---

**Authors:** Filip Křepinský (Red Hat), Morten Torkildsen (Google), Ravi Gudimetla (Apple)

<!--
Ensuring the disruptions to your applications do not affect its availability isn't a simple task. Last month's release of Kubernetes v1.26 lets you specify an  _unhealthy pod eviction policy_ for [PodDisruptionBudgets](/docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets) (PDBs) to help you maintain that availability during node management operations.
In this article, we will dive deeper into what modifications were introduced for PDBs to give application owners greater flexibility in managing disruptions.
-->

アプリケーションの中断がその可用性に影響を与えないようにすることは、簡単な作業ではありません。先月のKubernetes v1.26のリリースでは、ノード管理の操作中にその可用性を維持するために、[PodDisruptionBudgets](/docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets) (PDBs) に対して _unhealthy pod eviction policy_ を指定することができるようになりました。
この記事では、アプリケーション所有者がより柔軟にdisruptionを管理できるようにするために、PDBにどのような修正が導入されたかを深く掘り下げて説明します。

<!--
## What problems does this solve?
-->
## 解決したい問題

<!--
API-initiated eviction of pods respects PodDisruptionBudgets (PDBs). This means that a requested [voluntary disruption](https://kubernetes.io/docs/concepts/scheduling-eviction/#pod-disruption) via an eviction to a Pod, should not disrupt a guarded application and `.status.currentHealthy` of a PDB should not fall below `.status.desiredHealthy`. Running pods that are [Unhealthy](/docs/tasks/run-application/configure-pdb/#healthiness-of-a-pod) do not count towards the PDB status, but eviction of these is only possible in case the application is not disrupted. This helps disrupted or not yet started application to achieve availability as soon as possible without additional downtime that would be caused by evictions.

Unfortunately, this poses a problem for cluster administrators that would like to drain nodes without any manual interventions. Misbehaving applications with pods in `CrashLoopBackOff` state (due to a bug or misconfiguration) or pods that are simply failing to become ready make this task much harder. Any eviction request will fail due to violation of a PDB,  when all pods of an application are unhealthy. Draining of a node cannot make any progress in that case.

On the other hand there are users that depend on the existing behavior, in order to:
- prevent data-loss that would be caused by deleting pods that are guarding an underlying resource or storage
- achieve the best availability possible for their application

Kubernetes 1.26 introduced a new experimental field to the PodDisruptionBudget API: `.spec.unhealthyPodEvictionPolicy`.
When enabled, this field lets you support both of those requirements.
-->

PodのAPIを起点にした退避は、PodDisruptionBudget (PDB)の設定を尊重します。これは、Podの退避から要求された [voluntary disruption](https://kubernetes.io/docs/concepts/scheduling-eviction/#pod-disruption) が、保護されているアプリケーションを混乱させてはならず、PDB の `.status.currentHealthy` が `.status.desiredHealthy` を下回ってはならない、ということを意味します。[Unhealthy](/docs/tasks/run-application/configure-pdb/#healthiness-of-a-pod) である実行中のPodはPDBのステータスにカウントされませんが、アプリケーションを中断させない場合にのみ、これらのPodを退避させることが可能です。これにより、アプリケーションが停止している場合やまだ起動していない場合に、退避によって生じる追加のダウンタイムを発生させることなく、できるだけ早く可用性を確保することができます。

残念ながら、これは手動で介入することなくノードを排除したいクラスタ管理者にとっては問題となります。バグや設定ミスで `CrashLoopBackOff` 状態のPodを持つ誤動作するアプリケーションや、単に準備が整わないPodは、このタスクを非常に難しくしています。アプリケーションのすべてのPodが Unhealthy である場合、PDB違反のためにすべての退避要求が失敗します。その場合、ノードの排除には何の進展もありません。

一方、次のような目的で既存の動作に依存するユーザーもいます。
- 基盤となるリソースやストレージを保護しているPodがあり、それが削除されることによって発生しうるデータ損失を防止している
- アプリケーションで可能な限り最高の可用性を実現している

Kubernetes 1.26 では、PodDisruptionBudget API に新しい実験的フィールド `.spec.unhealthyPodEvictionPolicy` が導入されました。
このフィールドを有効にすると、これらの要件の両方をサポートすることができます。

<!--
## How does it work?
-->
## どのように動作するか？

<!--
API-initiated eviction is the process that triggers graceful pod termination.
The process can be initiated either by calling the API directly, by using a `kubectl drain` command, or other actors in the cluster.
During this process every pod removal is consulted with appropriate PDBs, to ensure that a sufficient number of pods is always running in the cluster.

The following policies allow PDB authors to have a greater control how the process deals with unhealthy pods.

There are two policies `IfHealthyBudget` and `AlwaysAllow` to choose from.

The former, `IfHealthyBudget`, follows the existing behavior to achieve the best availability that you get by default. Unhealthy pods can be disrupted only if their application has a minimum available `.status.desiredHealthy` number of pods.

By setting the `spec.unhealthyPodEvictionPolicy` field of your PDB to `AlwaysAllow`, you are choosing the best effort availability for your application.
With this policy it is always possible to evict unhealthy pods.
This will make it easier to maintain and upgrade your clusters.

We think that `AlwaysAllow` will often be a better choice, but for some critical workloads you may still prefer to protect even unhealthy Pods from node drains or other forms of API-initiated eviction.
-->

APIを起点にした退避は、graceful pod terminationのトリガーとなるプロセスです。
このプロセスは、API を直接呼び出すか、`kubectl drain` コマンドを使用するか、クラスタ内の他のアクターによって開始することができます。
このプロセスの間、クラスタ内で常に十分な数のPodが動作するように、すべてのPodの削除は適切なPDBに従って実行されます。

以下のポリシーにより、PDBの作者はプロセスが不健全なPodをどのように処理するかをより細かく制御することができます。

ポリシーは、 `IfHealthyBudget` と `AlwaysAllow` の2つから選択することができます。

前者の `IfHealthyBudget` は、デフォルトで得られる最高の可用性を実現するために、既存の動作に従います。Unhealthy なPodは、そのアプリケーションが最低限利用可能な `.status.desiredHealthy` 個のPodを持っている場合にのみ、中断させることができます。

PDB の `spec.unhealthyPodEvictionPolicy` フィールドを `AlwaysAllow` に設定することで、アプリケーションに最適な可用性を選択することになります。
このポリシーでは、UnhealthyなPodを常に退去させることが可能です。
これにより、クラスタの保守とアップグレードが容易になります。

しかし、クリティカルなワークロードでは、ノードの排除や他のAPIを起点にした退避からUnhealthyなPodを保護することが望ましい場合があります。

<!--
## How do I use it?
-->
## どのように使うか？

<!--
This is an alpha feature, which means you have to enable the `PDBUnhealthyPodEvictionPolicy` [feature gate](/docs/reference/command-line-tools-reference/feature-gates/), with the command line argument `--feature-gates=PDBUnhealthyPodEvictionPolicy=true` to the kube-apiserver.

Here's an example. Assume that you've enabled the feature gate in your cluster, and that you already defined a Deployment that runs a plain webserver. You labelled the Pods for that Deployment with `app: nginx`.
You want to limit avoidable disruption, and you know that best effort availability is sufficient for this app.
You decide to allow evictions even if those webserver pods are unhealthy.
You create a PDB to guard this application, with the `AlwaysAllow` policy for evicting unhealthy pods:
-->

これはアルファ機能です。この `PDBUnhealthyPodEvictionPolicy`([feature gate](/docs/reference/command-line-tools-reference/feature-gates/)) を有効にするには、kube-apiserver にコマンドライン引数 `--feature-gates=PDBUnhealthyPodEvictionPolicy=true` を与える必要があります。

以下はPodDisruptionBudgetの例です。
クラスタでフィーチャーゲートを有効にし、プレーンなWebサーバーを実行するDeploymentが既に定義されています。その Deployment の Pod には、`app: nginx` というラベルが付いています。回避可能なdisruptionを制限したいと考え、アプリケーションにはベストエフォートの可用性で十分であることを分かっているとします。そこで、WebサーバのPodがUnhealthyであっても、退避を許可することにします。
アプリケーションを保護するために、UnhealthyなPodを退避させる `AlwaysAllow` ポリシーを持つ PDB を以下のように作成します。

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  selector:
    matchLabels:
      app: nginx
  maxUnavailable: 1
  unhealthyPodEvictionPolicy: AlwaysAllow
```


## How can I learn more?


- Read the KEP: [Unhealthy Pod Eviction Policy for PDBs](https://github.com/kubernetes/enhancements/tree/master/keps/sig-apps/3017-pod-healthy-policy-for-pdb)
- Read the documentation: [Unhealthy Pod Eviction Policy](/docs/tasks/run-application/configure-pdb/#unhealthy-pod-eviction-policy) for PodDisruptionBudgets
- Review the Kubernetes documentation for [PodDisruptionBudgets](docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets), [draining of Nodes](docs/tasks/administer-cluster/safely-drain-node/) and [evictions](docs/concepts/scheduling-eviction/api-eviction/)


## How do I get involved?

If you have any feedback, please reach out to us in the [#sig-apps](https://kubernetes.slack.com/archives/C18NZM5K9) channel on Slack (visit https://slack.k8s.io/ for an invitation if you need one), or on the SIG Apps mailing list: kubernetes-sig-apps@googlegroups.com

