class: center, middle

# OpenShift と Scheduler


## Kenjiro Nakayama

---
# Who am I?

- 中山 健次郎 (なかやま けんじろう)

- OpenShift サポートエンジニア

- Fedora Project パッケージメンテナー

---

class: center, middle

# なぜ、Scheduler・・・

---
# なぜ、Scheduler・・・

--

- 勉強すると役にたちますか・・・？

--

- 今、ホットな話題ですか・・・？

--

![Default-aligned upstream-k8sm.png](./image/upstream-k8sm.png)

---
# Outline

- OpenShift Scheduler の仕組み

- Scheduler の苦労

- 参考資料

---

class: center, middle

# OpenShift の Scheduler

---
# もう少し前置き

--

- (今回は) ***Job Scheduler*** の話ではありません

--

![Default-aligned jobscheduler.png](./image/jobscheduler.png)

--

- 今回の Schedulr = **どの Node に Pod を配置するか**

 - CPU/Memory リソースやポート等で、ベストフィットするノードを選択

--

- OpenShift v3 は、Kubernetes の Scheduler をそのまま利用

--

- demo

---
# Scheduling Steps

--
- Step 1 : ノードのフィルター

--

  - 利用可能ノードのフィルター
  - 条件を ***predicates*** で指定
  - 例: Port が競合する Node を振るい落とす、要求したリソースの上限を超えている Node を振るい落とす

--

- Step 2 : 振るい落とされなかったノードに ***Priority*** を付ける

--

  - 0 ~ 10 までのスコアを各Priorityで計算。(10が最もフィットする)
  - 各 Priority の計算で重み付け (***weight***) を設定して、スコアと掛け算させることも可能

--

-  Step 3 : 最もフィットしたノードの選択

--
--

  - もっとも高いスコアのノードに Pod はデプロイされる
  - 同じスコアのノードが複数存在した場合は、ランダムに選択

---

class: center, middle

# Predicates と Priority

---
class: center, middle

# Predicates

---
# Predicates


- PodFitsPorts

  - 既存のPodとportが競合しないノードに絞る

- NoDiskConflict

  - PersistentVolume に GCE と AWSElasticBlockStore を使っている場合に、Disk が conflict しないノードに絞る
  - ****注****: OpenShift v3 では、GCE と EBS は現在TechPreview

- PodFitsResources

  - リクエストした Pod の CPU と Memory がノードの Capacity を超えていないノードに絞る

  - 設定例:
```area
{"name" : "PodFitsResources"}
```

---
# Predicates (続き)

- MatchNodeSelector
  - 指定した Selector がマッチするノードに絞る
  - NodeSelectorについては、[k8sドキュメント](https://github.com/GoogleCloudPlatform/kubernetes/blob/release-1.0/docs/user-guide/node-selection/README.md) や [v3 training コンテンツ](https://github.com/thoraxe/training/blob/master/02-Installation-and-Scheduler.md#the-nodeselector) を参照

- HostName
  - Host 名の指定が既存のノードのホスト名にマッチしているものに絞る

---
# Predicates (続き)

- LabelsPresence

 - ノードの Label で絞る

 - 例 : ***retiring*** の Label がついたノードには、デプロイしない ("presence" の指定に注意)

```area
{"name" : "ZoneRequired", "argument" : {"labels" : ["retiring"], "presence" : false}}
```

- ServiceAffinity
 - Label の value に関係なく、指定した Label が定義されているノードにデプロイする

 - デプロイするPodに、デプロイするノードの Label が指定されていない場合、最初のpodは任意のノードにデプロイされる

 - それ以降のPodで最初のPodと同じサービスのPodについては、最初にデプロイしたノードと同じ Label のついたノードにデプロイする

 - 後ほど具体例

---
class: center, middle

# Priority

---
# Priority

- LeastRequestedPriority

  - 既存のPodに要求されているリソースが最も少ないリソースのノードにデプロイされやすくする

```area
   score = cpu((capacity - sum(requested)) * 10 / capacity)
               + memory((capacity - sum(requested)) * 10 / capacity) / 2
```


- BalancedResourceAllocation

  - 既存のPodが要求しているCPUとMemoryで偏りがないノードにデプロイされやすくする

```area
   fraction = sum(requested) / capacity
   score = 10 - abs( cpuFraction - memoryFraction ) * 10
```

```text
  参考: "Wei Huang et al. An Energy Efficient Virtual Machine
                                   Placement Algorithm with Balanced Resource Utilization"
```

---
# Priority (続き)

- ServiceSpreadingPriority

  - 同一ノード上に同じサービスが稼働しないように分散させる

- EqualPriority

  - すべてのノードのPriorityスコアを 「1」 にする

  - priority の設定が一切ない場合に内部的に利用されるもので、通常使用しない

---
# Priority (続き)

- LabelPreference

 - 指定した Label が定義されているノードに対し、1 × weight だけスコアを増やす

```area
{"name" : "RackPreferred", "weight" : 1, "argument" : {"labelPreference" : {"label" : "rack"}}}
```

- ServiceAntiAffinity

 - 同一Labelグループ内で同一サービスのPodを分散させるようにする

 - 詳細は使用例にて

---
class: center, middle

# 使用例

---
# Affinity (Zone), Anti-Affinity (Region, Rack)

* 最初のPodはZone2/Region22/Rack23にデプロイされたとします・・・

![Default-aligned pod-location1.jpg](./image/pod-location1.jpg)

---
# Affinity (Zone), Anti-Affinity (Region, Rack)

* Podのデプロイ順序とZone(Affinity)、Region,Rack(Anti-Affinity)にご注目

![Default-aligned pod-location2.jpg](./image/pod-location2.jpg)

---
# Default Scheduler Policy (参考)

##### predicates

  1. PodFitsPorts

  2. PodFitsResources

  3. NoDiskConflict

  4. MatchNodeSelector

  5. HostName

##### priorities

  1. LeastRequestedPriority

  2. BalancedResourceAllocation

  3. ServiceSpreadingPriority

---
# Default Scheduler Policy (参考)

##### predicates

  1. PodFitsPorts ... ポートが競合しない

  2. PodFitsResources ... リソースがNodeの上限を超えない

  3. NoDiskConflict ... DiskのConflictがない

  4. MatchNodeSelector ... NodeSelectorが一致する

  5. HostName ... 要求したホスト名とNodeのホスト名が一致する

##### priorities

  1. LeastRequestedPriority ... これまでに要求したリソースが少いNodeを優先

  2. BalancedResourceAllocation ... Memory、CPUの偏りが少ないNodeを優先

  3. ServiceSpreadingPriority ... 同一サービスが稼働していないNodeを優先

---
# Debug in Error

- Scheduling できなかった Pod は ***Pending*** 状態

- 状態を確認するには、 `oc get events` や `oc describe pod <POD名>`

```area
Events:
  FirstSeen                             LastSeen                ...
Mon, 27 Jul 2015 21:44:48 -0400	Mon, 27 Jul 2015 21:44:55 -0400	...

    Message
  Error scheduling: failed to find fit for pod, Node n1.example.com: PodFitsPorts
```

---
# Configuration

- 設定ファイル
  - `/etc/openshift/master/master-config.yaml` で設定ファイルを確認
  - => `schedulerConfigFile: /etc/openshift/master/scheduler.json`

```area
# cat /etc/openshift/master/scheduler.json
{
  "predicates": [
    {"name": "PodFitsResources"},
    {"name": "PodFitsPorts"},
    {"name": "NoDiskConflict"},
    {"name": "Region", "argument": {"serviceAffinity" : {"labels" : ["region"]}}}
  ],"priorities": [
    {"name": "LeastRequestedPriority", "weight": 1},
    {"name": "ServiceSpreadingPriority", "weight": 1},
    {"name": "Zone", "weight" : 2, "argument": {"serviceAntiAffinity" : {"label": "zone"}}}
  ]
}
```

---
class: center, middle

# Resourceについてもう少し
---
# Resource

- 例 : LeastRequestedPriority (既に要求されているリソースが最も少ないリソースのノードにデプロイ)

```area
   score = cpu((capacity - sum(requested)) * 10 / capacity)
               + memory((capacity - sum(requested)) * 10 / capacity) / 2
```

- capacity は、ホストの CPU コア数 と Memory

- requested は、pod のdeploy定義ファイルに設定したリソースの上限

--

 - pod 内のプロセスに対して、cgroupsのグループが割り当てられる

```area
[root@ose3-master ~]# systemd-cgls  |grep docker |grep scope
  ├─docker-15946429100ae87136012f87211ae12689559ab8c954d9794037a405d654954f.scope
  ├─docker-a1327bcbb31a9962a615394bdc424fcd43a9d14cc6233e333781439376e6678b.scope
```

---
class: center, middle

# Scheduler の苦労
---
# Scheduler の苦労

--

- 計算されるリソースは、****CPU**** と ****Memory**** のみ

--

  - ネットワーク帯域やDisk容量は現状考慮されていない

--

  - CPU、Memory 以外の要素を追加した場合の、計算方式の問題

--

- OpenShift(Kubernetes) で管理しているリソースのみを計算

--

  - ホスト側で実行されるプロセスは管理されない

---
# Reference

- ドキュメント:
 - Scheduler - https://docs.openshift.com/enterprise/3.0/admin_guide/scheduler.html#default-scheduler-policy

- ソースコード:

 - https://github.com/openshift/origin/blob/v1.0.1/Godeps/_workspace/src/github.com/GoogleCloudPlatform/kubernetes/plugin/pkg/scheduler/algorithm/priorities/priorities.go

 - https://github.com/openshift/origin/blob/v1.0.1/Godeps/_workspace/src/github.com/GoogleCloudPlatform/kubernetes/plugin/pkg/scheduler/algorithm/predicates/predicates.go

---
class: center middle
# おしまい
