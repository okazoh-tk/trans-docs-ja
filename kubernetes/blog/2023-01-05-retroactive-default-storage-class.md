---
layout: blog
title: "Kubernetes 1.26: Retroactive Default StorageClass"
date: 2023-01-05
slug: retroactive-default-storage-class
---

**Author:** Roman Bednář (Red Hat)

<!--
The v1.25 release of Kubernetes introduced an alpha feature to change how a default StorageClass was assigned to a PersistentVolumeClaim (PVC).
With the feature enabled, you no longer need to create a default StorageClass first and PVC second to assign the class. Additionally, any PVCs without a StorageClass assigned can be updated later.
This feature was graduated to beta in Kubernetes 1.26.

You can read [retroactive default StorageClass assignment](/docs/concepts/storage/persistent-volumes/#retroactive-default-storageclass-assignment) in the Kubernetes documentation for more details about how to use that, or you can read on to learn about why the Kubernetes project is making this change.
-->

Kubernetesのv1.25リリースでは、デフォルトのStorageClassをPersistentVolumeClaim（PVC）に割り当てる方法を変更するアルファ機能が導入されました。
この機能を有効にすると、クラスを割り当てるために、最初にデフォルトのStorageClassを作成し、次にPVCを作成する必要がなくなります。さらに、StorageClassが割り当てられていないPVCは、後で更新することができます。
この機能は、Kubernetes v1.26でベータ版になりました。

使い方については、Kubernetesのドキュメントにある[retroactive default StorageClass assignment](/docs/concepts/storage/persistent-volumes/#retroactive-default-storageclass-assignment) を読むとよいでしょう。
Kubernetesプロジェクトがこの変更を行う理由については、本文書をお読みください。

## Why did StorageClass assignment need improvements

<!--
Users might already be familiar with a similar feature that assigns default StorageClasses to **new** PVCs at the time of creation. This is currently handled by the [admission controller](/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass).

But what if there wasn't a default StorageClass defined at the time of PVC creation? 
Users would end up with a PVC that would never be assigned a class. 
As a result, no storage would be provisioned, and the PVC would be somewhat "stuck" at this point. 
Generally, two main scenarios could result in "stuck" PVCs and cause problems later down the road. 
Let's take a closer look at each of them.
-->

ユーザは、作成時にデフォルトのStorageClassを**新しい**PVCに割り当てる同様の機能を既に知っているかもしれません。これは現在、[admission controller](/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass) によって処理されます。

しかし、もしPVC作成時にデフォルトのStorageClassが定義されていなかったらどうでしょう？
おそらくクラスが割り当てられないPVCで終わるでしょう。
その結果、ストレージはプロビジョニングされず、PVCはこの時点でいくらか「スタック」することになります。
一般的に、2つの主なシナリオが「スタック」したPVCをもたらし、後々問題を引き起こす可能性があります。
それぞれについて詳しく見ていきましょう。

### Changing default StorageClass

<!--
With the alpha feature enabled, there were two options admins had when they wanted to change the default StorageClass:

1. Creating a new StorageClass as default before removing the old one associated with the PVC. 
This would result in having two defaults for a short period.
At this point, if a user were to create a PersistentVolumeClaim with storageClassName set to <code>null</code> (implying default StorageClass), the newest default StorageClass would be chosen and assigned to this PVC.

2. Removing the old default first and creating a new default StorageClass.
This would result in having no default for a short time.
Subsequently, if a user were to create a PersistentVolumeClaim with storageClassName set to <code>null</code> (implying default StorageClass), the PVC would be in <code>Pending</code> state forever.
The user would have to fix this by deleting the PVC and recreating it once the default StorageClass was available.
-->

alpha機能を有効にすると、管理者がデフォルトのStorageClassを変更したい場合、2つのオプションがありました。

1. 新しいStorageClassをデフォルトとして作成してから、PVCに関連する古いStorageClassを削除する。
この場合、短期間、2つのデフォルトが存在することになります。
この時点で、ユーザが storageClassName を <code>null</code> に設定した PersistentVolumeClaim を作成する(デフォルト StorageClass を指定することを意味する)と、最新のデフォルト StorageClass が選択されてこの PVC に割り当てられることになります。

2. 古いデフォルトを最初に削除し、新しいデフォルトStorageClassを作成する。
この場合、しばらくの間、デフォルトがない状態になります。
その後、もしユーザーがstorageClassNameを<code>null</code>に設定したPersistentVolumeClaimを作成する(デフォルト StorageClass を指定することを意味する)と、PVCは永遠に<code>Pending</code>状態になるでしょう。
ユーザーは、PVCを削除し、デフォルトのStorageClassが利用可能になったら、それを再作成することによって、この問題を修正する必要があります。


### Resource ordering during cluster installation

<!--
If a cluster installation tool needed to create resources that required storage, for example, an image registry, it was difficult to get the ordering right. 
This is because any Pods that required storage would rely on the presence of a default StorageClass and would fail to be created if it wasn't defined.
-->

クラスタインストールツールがイメージレジストリのようなストレージを必要とするリソースを作成する必要がある場合、その順序を正しくすることは困難でした。
これは、ストレージを必要とするPodは、デフォルトのStorageClassの存在に依存しており、それが定義されていない場合は作成に失敗するためです。

## What changed

<!--
We've changed the PersistentVolume (PV) controller to assign a default StorageClass to any unbound PersistentVolumeClaim that has the storageClassName set to <code>null</code>.
We've also modified the PersistentVolumeClaim admission within the API server to allow the change of values from an unset value to an actual StorageClass name.
-->

storageClassName に <code>null</code> が設定されている未束縛の PersistentVolumeClaim に対して、デフォルトの StorageClass を割り当てるように PersistentVolume (PV) コントローラーを変更しました。
また、API サーバー内の PersistentVolumeClaim アドミッションも変更し、未設定の値から実際の StorageClass 名に値を変更できるようにしました。

### Null `storageClassName` versus `storageClassName: ""` - does it matter? { #null-vs-empty-string }

<!--
Before this feature was introduced, those values were equal in terms of behavior. Any PersistentVolumeClaim with the storageClassName set to <code>null</code> or <code>""</code> would bind to an existing PersistentVolume resource with storageClassName also set to <code>null</code> or <code>""</code>.

With this new feature enabled we wanted to maintain this behavior but also be able to update the StorageClass name.
With these constraints in mind, the feature changes the semantics of <code>null</code>. If a default StorageClass is present, <code>null</code> would translate to "Give me a default" and <code>""</code> would mean "Give me PersistentVolume that also has <code>""</code> StorageClass name." In the absence of a StorageClass, the behavior would remain unchanged.

Summarizing the above, we've changed the semantics of <code>null</code> so that its behavior depends on the presence or absence of a definition of default StorageClass.

The tables below show all these cases to better describe when PVC binds and when its StorageClass gets updated.
-->

本機能が導入される前は、これらの値は動作の点で等しくなっていました。storageClassName が <code>null</code> または <code>""</code> に設定された PersistentVolumeClaim は、同じく storageClassName が <code>null</code> または <code>""</code> に設定された既存の PersistentVolume リソースにバインドされました。

本機能を有効にすると、この動作を維持しつつ、StorageClass 名を更新できるようにしたいと考えました。
これらの制約を念頭に置いて、この機能は <code>null</code> のセマンティクスを変更します。もし、デフォルトの StorageClass があれば、<code>null</code> は「デフォルトをくれ」と訳し、<code>""</code> は「<code>""</code> StorageClass name も持つ PersistentVolume をくれ」を意味します。StorageClassがない場合は、動作は変わりません。

上記をまとめると、我々は<code>null</code>のセマンティクスを変更し、その動作がデフォルトStorageClassの定義の有無に依存するようにしたのです。

以下の表は、PVC がいつバインドされ、いつ StorageClass が更新されるかをよりよく説明するために、これらのすべてのケースを示しています。

<!--
<table>
  <caption>PVC binding behavior with Retroactive default StorageClass</caption>
  <thead>
     <tr>
        <th colspan="2"></th>
        <th>PVC <tt>storageClassName</tt> = <code>""</code></th>
        <th>PVC <tt>storageClassName</tt> = <code>null</code></th>
     </tr>
  </thead>
  <tbody>
     <tr>
        <td rowspan="2">Without default class</td>
        <td>PV <tt>storageClassName</tt> = <code>""</code></td>
        <td>binds</td>
        <td>binds</td>
     </tr>
     <tr>
        <td>PV without <tt>storageClassName</tt></td>
        <td>binds</td>
        <td>binds</td>
     </tr>
     <tr>
        <td rowspan="2">With default class</td>
        <td>PV <tt>storageClassName</tt> = <code>""</code></td>
        <td>binds</td>
        <td>class updates</td>
     </tr>
     <tr>
        <td>PV without <tt>storageClassName</tt></td>
        <td>binds</td>
        <td>class updates</td>
     </tr>
  </tbody>
</table>
-->

<table>
  <caption>PVC binding behavior with Retroactive default StorageClass</caption>
  <thead>
     <tr>
        <th colspan="2"></th>
        <th>PVC <tt>storageClassName</tt> = <code>""</code></th>
        <th>PVC <tt>storageClassName</tt> = <code>null</code></th>
     </tr>
  </thead>
  <tbody>
     <tr>
        <td rowspan="2">デフォルトクラスなし</td>
        <td>PV <tt>storageClassName</tt> = <code>""</code></td>
        <td>バインド</td>
        <td>バインド</td>
     </tr>
     <tr>
        <td>PV without <tt>storageClassName</tt></td>
        <td>バインド</td>
        <td>バインド</td>
     </tr>
     <tr>
        <td rowspan="2">デフォルトクラスあり</td>
        <td>PV <tt>storageClassName</tt> = <code>""</code></td>
        <td>バインド</td>
        <td>クラス更新</td>
     </tr>
     <tr>
        <td>PV without <tt>storageClassName</tt></td>
        <td>バインド</td>
        <td>クラス更新</td>
     </tr>
  </tbody>
</table>

## How to use it

<!--
If you want to test the feature whilst it's alpha, you need to enable the relevant feature gate in the kube-controller-manager and the kube-apiserver. Use the `--feature-gates` command line argument:
-->

アルファ版で機能をテストしたい場合は、kube-controller-managerとkube-apiserverで該当するフィーチャーゲートを有効化する必要があります。次のようにコマンドライン引数として `--feature-gates` を使用してください。

```
--feature-gates="...,RetroactiveDefaultStorageClass=true"
```

### Test drive

<!--
If you would like to see the feature in action and verify it works fine in your cluster here's what you can try:
-->

もし、この機能が実際に動作しているところを見たり、クラスタで問題なく動作することを確認したい場合は、以下のことを試してみてください。

<!--
1. Define a basic PersistentVolumeClaim:
-->
1. PersistentVolumeClaimを定義します
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-1
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
<!--
2. Create the PersistentVolumeClaim when there is no default StorageClass. The PVC won't provision or bind (unless there is an existing, suitable PV already present) and will remain in <code>Pending</code> state.
-->
2. デフォルトのStorageClassが存在しない場合に、PersistentVolumeClaimを作成します。このPVCはプロビジョニングもバインディングも行わず（既存の適切なPVが既に存在しない限り）、<code>Pending</code>状態のままとなります。
```
$ kc get pvc
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-1     Pending   
```

<!--
3. Configure one StorageClass as default.
-->
3. 1つのStorageClassをデフォルトに設定します。
```
$ kc patch sc -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
storageclass.storage.k8s.io/my-storageclass patched
```

<!--
4. Verify that PersistentVolumeClaims is now provisioned correctly and was updated retroactively with new default StorageClass.
-->
4. PersistentVolumeClaims が正しくプロビジョニングされ、新しいデフォルトの StorageClass で更新されたことを確認します。
```
$ kc get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
pvc-1     Bound    pvc-06a964ca-f997-4780-8627-b5c3bf5a87d8   1Gi        RWO            my-storageclass   87m
```

### New metrics

<!--
To help you see that the feature is working as expected we also introduced a new <code>retroactive_storageclass_total</code> metric to show how many times that the PV controller attempted to update PersistentVolumeClaim, and <code>retroactive_storageclass_errors_total</code> to show how many of those attempts failed.
-->

この機能が期待通りに動作していることを確認するために、PV コントローラーが何回 PersistentVolumeClaim を更新しようとしたかを示す新しい <code>retroactive_storageclass_total</code> メトリクスと、それらの試みのうち何回失敗したかを示す <code>retroactive_storageclass_errors_total</code> も導入されました。

## Getting involved

We always welcome new contributors so if you would like to get involved you can join our [Kubernetes Storage Special-Interest-Group](https://github.com/kubernetes/community/tree/master/sig-storage) (SIG).

If you would like to share feedback, you can do so on our [public Slack channel](https://app.slack.com/client/T09NY5SBT/C09QZFCE5).

Special thanks to all the contributors that provided great reviews, shared valuable insight and helped implement this feature (alphabetical order):

- Deep Debroy ([ddebroy](https://github.com/ddebroy))
- Divya Mohan ([divya-mohan0209](https://github.com/divya-mohan0209))
- Jan Šafránek ([jsafrane](https://github.com/jsafrane/))
- Joe Betz ([jpbetz](https://github.com/jpbetz))
- Jordan Liggitt ([liggitt](https://github.com/liggitt))
- Michelle Au ([msau42](https://github.com/msau42))
- Seokho Son ([seokho-son](https://github.com/seokho-son))
- Shannon Kularathna ([shannonxtreme](https://github.com/shannonxtreme))
- Tim Bannister ([sftim](https://github.com/sftim))
- Tim Hockin ([thockin](https://github.com/thockin))
- Wojciech Tyczynski ([wojtek-t](https://github.com/wojtek-t))
- Xing Yang ([xing-yang](https://github.com/xing-yang))
