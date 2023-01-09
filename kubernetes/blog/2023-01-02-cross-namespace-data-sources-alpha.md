---
layout: blog
title: "Kubernetes v1.26: Alpha support for cross-namespace storage data sources"
date: 2023-01-02
slug: cross-namespace-data-sources-alpha
---

**Author:** Takafumi Takahashi (Hitachi Vantara)

<!--
Kubernetes v1.26, released last month, introduced an alpha feature that lets you specify a data source for a PersistentVolumeClaim, even where the source data belong to a different namespace. With the new feature enabled, you specify a namespace in the `dataSourceRef` field of a new PersistentVolumeClaim. Once Kubernetes checks that access is OK, the new PersistentVolume can populate its data from the storage source specified in that other namespace. Before Kubernetes v1.26, provided your cluster had the `AnyVolumeDataSource` feature enabled, you could already provision new volumes from a data source in the **same** namespace. However, that only worked for the data source in the same namespace, therefore users couldn't provision a PersistentVolume with a claim in one namespace from a data source in other namespace. To solve this problem, Kubernetes v1.26 added a new alpha `namespace` field to `dataSourceRef` field in PersistentVolumeClaim the API.
-->

先月リリースされたKubernetes v1.26では、ソースデータが異なる名前空間に属している場合でも、PersistentVolumeClaimにデータソースを指定できるアルファ機能が導入されました。  
この新機能を有効にするには、新しいPersistentVolumeClaimの`dataSourceRef`フィールドに名前空間を指定します。Kubernetesがアクセスに問題がないことを確認すると、新しいPersistentVolumeはその他の名前空間で指定されたストレージソースからデータを投入することができます。  
Kubernetes v1.26以前では、クラスタが `AnyVolumeDataSource` 機能を有効にしていれば、すでに **同じ** 名前空間内のデータソースから新しいボリュームをプロビジョニングすることができました。しかし、それは同じ名前空間内のデータソースに対してのみ機能するため、ユーザーは他の名前空間のデータソースから、ある名前空間にクレームを持つPersistentVolumeをプロビジョニングすることができませんでした。  
この問題を解決するために、Kubernetes v1.26では、PersistentVolumeClaimのAPIの`dataSourceRef`フィールドに新しい`namespace`フィールドをアルファ機能として追加しました。

## How it works

<!--
Once the csi-provisioner finds that a data source is specified with a `dataSourceRef` that has a non-empty namespace name, it checks all reference grants within the namespace that's specified by the`.spec.dataSourceRef.namespace` field of the PersistentVolumeClaim, in order to see if access to the data source is allowed. If any ReferenceGrant allows access, the csi-provisioner provisions a volume from the data source.
-->

csi-provisioner は、データソースが `dataSourceRef` で指定され、その名前空間の名前が空でないことがわかると、PersistentVolumeClaim の `.spec.dataSourceRef.namespace` フィールドで指定されている名前空間内のすべての参照許可をチェックして、データソースへのアクセスが許可されるかどうかを確認します。いずれかの ReferenceGrant がアクセスを許可している場合、csi-provisioner はデータ・ソースからボリュームをプロビジョニングします。

## Trying it out

<!--
The following things are required to use cross namespace volume provisioning:

* Enable the `AnyVolumeDataSource` and `CrossNamespaceVolumeDataSource` [feature gates](/docs/reference/command-line-tools-reference/feature-gates/) for the kube-apiserver and kube-controller-manager
* Install a CRD for the specific `VolumeSnapShot` controller
* Install the CSI Provisioner controller and enable the `CrossNamespaceVolumeDataSource` feature gate
* Install the CSI driver
* Install a CRD for ReferenceGrants
-->

cross-namespaceなボリューム・プロビジョニングを使用するには、以下の操作が必要です。

* kube-apiserver と kube-controller-manager の `AnyVolumeDataSource` と `CrossNamespaceVolumeDataSource`([feature gates](/docs/reference/command-line-tools-reference/feature-gates/)) を有効化
* 特定の `VolumeSnapShot` コントローラー用の CRD をインストール
* CSI Provisioner コントローラをインストールし、`CrossNamespaceVolumeDataSource` 機能のゲートを有効化
* CSI ドライバーをインストール
* ReferenceGrants 用の CRD をインストール

## Putting it all together

<!--
To see how this works, you can install the sample and try it out. This sample do to create PVC in dev namespace from VolumeSnapshot in prod namespace. That is a simple example. For real world use, you might want to use a more complex approach.
-->

本サンプルは、prod 名前空間の VolumeSnapshot から、dev 名前空間の PVC を作成するサンプルです。このサンプルは、prod 名前空間の VolumeSnapshot から dev 名前空間の PVC を作成するものです。これは簡単な例です。実際には、もっと複雑な方法を使用する必要があるかもしれません。

### Assumptions for this example {#example-assumptions}

<!--
* Your Kubernetes cluster was deployed with `AnyVolumeDataSource` and `CrossNamespaceVolumeDataSource` feature gates enabled
* There are two namespaces, dev and prod
* CSI driver is being deployed
* There is an existing VolumeSnapshot named `new-snapshot-demo` in the _prod_ namespace
* The ReferenceGrant CRD (from the Gateway API project) is already deployed
-->

* Kubernetes クラスタは `AnyVolumeDataSource` と `CrossNamespaceVolumeDataSource` 機能のゲートを有効にしてデプロイされます
* dev と prod という 2 つの名前空間があります
* CSIドライバがデプロイされています
* _prod_ 名前空間には、`new-snapshot-demo` という名前の VolumeSnapshot が存在します
* Gateway API プロジェクトの ReferenceGrant CRD は既にデプロイされています

### Grant ReferenceGrants read permission to the CSI Provisioner

<!--
Access to ReferenceGrants is only needed when the CSI driver has the `CrossNamespaceVolumeDataSource` controller capability. For this example, the external-provisioner needs **get**, **list**, and **watch** permissions for `referencegrants` (API group `gateway.networking.k8s.io`).
-->

ReferenceGrants へのアクセスは、CSI ドライバが `CrossNamespaceVolumeDataSource` コントローラ機能を持つ場合にのみ必要です。この例では、外部プロビジョナーは `referencegrants` (API グループ `gateway.networking.k8s.io`) に対して **get**, **list**, **watch** のパーミッションが必要です。

```yaml
  - apiGroups: ["gateway.networking.k8s.io"]
    resources: ["referencegrants"]
    verbs: ["get", "list", "watch"]
```

### Enable the CrossNamespaceVolumeDataSource feature gate for the CSI Provisioner

<!--
Add `--feature-gates=CrossNamespaceVolumeDataSource=true` to the csi-provisioner command line. For example, use this manifest snippet to redefine the container:
-->

csi-provisioner のコマンドラインに `--feature-gates=CrossNamespaceVolumeDataSource=true` を追加します。たとえば、このマニフェストの断片を使用して、コンテナを再定義します。

```yaml
      - args:
        - -v=5
        - --csi-address=/csi/csi.sock
        - --feature-gates=Topology=true
        - --feature-gates=CrossNamespaceVolumeDataSource=true
        image: csi-provisioner:latest
        imagePullPolicy: IfNotPresent
        name: csi-provisioner
```

### Create a ReferenceGrant

<!--
Here's a manifest for an example ReferenceGrant.
-->

これはReferenceGrantのサンプルマニフェストです。

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-prod-pvc
  namespace: prod
spec:
  from:
  - group: ""
    kind: PersistentVolumeClaim
    namespace: dev
  to:
  - group: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: new-snapshot-demo
```

### Create a PersistentVolumeClaim by using cross namespace data source

<!--
Kubernetes creates a PersistentVolumeClaim on dev and the CSI driver populates the PersistentVolume used on dev from snapshots on prod.
-->

Kubernetesはdev上にPersistentVolumeClaimを作成し、CSIドライバはprod上のスナップショットからdev上で使用するPersistentVolumeを投入しています。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
  namespace: dev
spec:
  storageClassName: example
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  dataSourceRef:
    apiGroup: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: new-snapshot-demo
    namespace: prod
  volumeMode: Filesystem
```

## How can I learn more?

The enhancement proposal, [Provision volumes from cross-namespace snapshots](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/3294-provision-volumes-from-cross-namespace-snapshots), includes lots of detail about the history and technical implementation of this feature.

Please get involved by joining the [Kubernetes Storage Special Interest Group (SIG)](https://github.com/kubernetes/community/tree/master/sig-storage)
to help us enhance this feature. There are a lot of good ideas already and we'd be thrilled to have more!

## Acknowledgments

It takes a wonderful group to make wonderful software.
Special thanks to the following people for the insightful reviews,
thorough consideration and valuable contribution to the CrossNamespaceVolumeDataSouce feature:

* Michelle Au (msau42)
* Xing Yang (xing-yang)
* Masaki Kimura (mkimuram)
* Tim Hockin (thockin)
* Ben Swartzlander (bswartz)
* Rob Scott (robscott)
* John Griffith (j-griffith)
* Michael Henriksen (mhenriks)
* Mustafa Elbehery (Elbehery)

It’s been a joy to work with y'all on this.
