* [Lec18: kubernetes 调度和资源管理](#lec18-kubernetes-调度和资源管理)
   * [1. Kubernetes  调度过程](#1-kubernetes--调度过程)
      * [1. 调度过程](#1-调度过程)
   * [2. Kubernetes 基础调度能力](#2-kubernetes-基础调度能力)
      * [1. 资源调度-满足 pod 资源需求](#1-资源调度-满足-pod-资源需求)
      * [2. 关系调度-满足 pod/node 特殊关系/条件要求](#2-关系调度-满足-podnode-特殊关系条件要求)
   * [3. Kubernetes 高级调度功能](#3-kubernetes-高级调度功能)
      * [1. 优先级调度和抢占 v1.14.1 stable 默认 On](#1-优先级调度和抢占-v1141-stable-默认-on)
   * [4. 调度器架构简介和具体算法](#4-调度器架构简介和具体算法)

Created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc)

### Lec18: kubernetes 调度和资源管理

#### 1. Kubernetes  调度过程

##### 1. 调度过程

1. apiserver 创建 pod (需要一些系列过程 回合 controller-manager 交互)，这时候创建的  pod nodeName 是没有的，status 是 pending 状态
2. 基础调度器 scheduler ，获取 pod 后根据调度算法以及打分机制将 Pod 和 Node 做匹配，pod.nodeName = scheduler 选出的 node 
3. 选出 node 的 Kubelet 控制 cni 和 容器 engine 创建配置容器

#### 2. Kubernetes 基础调度能力

##### 1. 资源调度-满足 pod 资源需求

1. Resource: CPU、Memory、ephemeral-Storage(短暂的非持久性存储)、GPU、FPGA

   Containers.request/limit.cpu/memory(1Gi=1024Mi)

2. QOS

   1. 为什么会有 request/limit? 提供一种弹性的控制
   2. 三类 QOS: Guaranteed 高保障 Burstable 中弹性 BestEffort 低尽力而为
   3. 隐性的QoS Class，用户无法自己定义，是 k8s 根据资源指定自己判断
   4. 如何配置 QoS: 
      1. 配置 Guaranteed ： limit 的 CPU 和 memory == request 的，其他可不等
      2. Burstable: limit 的 CPU 和 memory ！= request 的
      3. BestEffort：不填 limit 和 request
   5. 不同 QoS 区别？
      1. 调度变现：调度器会使用 request 进行调度
      2. 底层变现不同：
         1. cpu 按照request 划分权重 —cpu-manager-policy = static 的时候 guaranteed 整数会绑定核 非整数的 三类 Qos 会将剩余的 CPU 组成一个 cpushare pool
         2. Memory 会根据QoS划分 OOMScore: Guaranteed -998 Burstable 2~999 BestEffort 1000 得分越高在宿主机 OOM 的时候会被优先 kill 掉
         3. 节点 Eviction: 优先 BestEffort Kubelet-CpuManager

3. Resource Quota

   ```yaml
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: demo-resourcequota
     namespace: demo
   spec:
     hard: 
       cpu: "1000"
       memory: 200Gi
       pods: 10
     scoreSelector: # 可以不填
       matchExpression:
       - operator: Exist
         scopeName: NotBestEffort
   ```

   意思是限制 namespace 为 demo 下非BestEffort QoS 的Quota

   scopeName: 

   1. Terminating/NotTerminating 
   2. BestEffort/NotBestEffort
   3. PriorityClass

   当超过 Quota 创建资源的时候会禁止： forbidden: execeeded quota

##### 2. 关系调度-满足 pod/node 特殊关系/条件要求

1. Pod 和 pod 关系

   1. pod 亲和调度 PodAffinity 
      1. Pod 必须和 pod 调度到一起 : 严格亲和 `requiredDuringSchedulingIgnoredDuringExecution`
      2. Pod 优先和 Pod 调度到一起： `preferredDuringSchedulingIgnoredDuringExecution`
   2. Pod 反亲和性调度 PodAntiAffinity
      1. 禁止和 pod 调度到一起： `requiredDuringSchedulingIgnoredDuringExecution`
      2. 优先不某 pod 调度到一起：`preferredDuringSchedulingIgnoredDuringExecution`
   3. Operator 
      1. In/Notin/Exists/DoesNotExist

   Demo1: pod 必须调度到带 k1=v1 pod 所在的节点

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     namespace: demo
     name: demo
   spec:
     containers:
     - image: nginx:latest
       name: demo-nginx
     affinity:
       podAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:
         - labelSelector:
           matchExpressions:
           - key: k1
             operator: In
             values:
             - v1
           topologykey: "kubernetes.io/hostname"
   ```

   Demo2: pod 倾向于调度到 k2=v2 pod 所在的节点

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     namespace: demo
     name: demo
   spec:
     containers:
     - image: nginx:latest
       name: demo-nginx
     affinity:
       podAffinity:
         preferredDuringSchedulingIgnoredDuringExecution:
         - weight: 100
         podAffinityTerm:
           labelSelector:
             matchExpressions:
             - key: k2
               operator: In
               values:
               - v2
             topologykey: "kubernetes.io/hostname"
   ```

2. Pod 和 Node 的关系

   1. NodeSelector: 
      1. 必须调度到某些标签的 node
      2. Map[string]string
   2. NodeAffinity
      1. 必须调度到某个节点 requiredDuringSchedulingIgnoredDuringExecution
      2. 优先调度到某些节点 preferredDuringSchedulingIgnoredDuringExecution
      3. Operator: In\NotIn\Exists\DoseNotExist\Gt\Lt

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     namespace: demo
     name: demo
   spec:
     containers:
     - image: nginx:latest
       name: demo-nginx
     labelSelector:
       k1: v1
   ```

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     namespace: demo
     name: demo
   spec:
     containers:
     - image: nginx:latest
       name: demo-nginx
     affinity:
       nodeAffinity:
         preferredDuringSchedulingIgnoredDuringExecution:
         nodeAffinityTerm:
         - matchExpressions:
           - key: k1
             operator: In
             values:
           	- v1
           	- v2
   ```

3. 限制 pod 调度到某节点

   1. NodeTaint
      1. 一个 node 可以有多个 nodetaint
      2. Effect(不能为空)
         1. NoSchedule 禁止新的 pods 调度上来
         2. PrefreNoSchedule 尽量不调度到该节点
         3. NoExecute 会 evict 没有对应 toleration 的 pod 并且不会调度新的 Pod 到上面
   2. Pod Tolerations
      1. 一个 pod 可以有多个 tolerations
      2. Effect 可以为空，匹配所有：
         1. 取值和 Tanits 的 Effect 一致
      3. Operator 
         1. Exists、Equal

   创建 nodetaint

   ```yaml
   apiVersion: v1
   kind: Node
   metadata:
   	name: demo
   spec:
     taints:
       - key: "k1"
         value: "v1"
         effect: "NoSchedule"
   ```

   pod 上加 noschedule 的 taint

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: demo
     namespace: demo
   spec:
     containers:
     - name: nginx
       images: nginx:latest
     toleration:
     - key: "k1"
       operator: "Equal" # Exists 可以省略
       value: "v1"
       effect: "NoSchedule"
   ```

   

#### 3. Kubernetes 高级调度功能

##### 1. 优先级调度和抢占 v1.14.1 stable 默认 On

当集群资源不够时：

先到先得（FIFO）、优先级策略（符合公司业务）

1. Priority

   如何使用 ?

   1. 创建  PriorityClass 

      ```yaml
      apiVersion: scheduling.k8s.io/v1
      kind: PriorityClass
      metadata:
        name: high
      value: 10000
      globalDefault: false
      ---
      apiVersion: scheduling.k8s.io/v1
      kind: PriorityClass
      metadata:
        name: low
      value: 100
      globalDefault: false
      ```

   2. 为各个 Pod 配置不同的 priorityClassName 

      ```yaml
      spec:
        priorityClassName: high
      ```

   优先级调度过程：

   发生在 pod pending 出队列时开始：

   1. pod1 和 pod2 先后进入调度队列，但是未开始调度
   2. 出队列，PriorityQueue 会优先 pop 更大优先级的 pod
   3. 调度成功后 pod1 绑定，开始调度  pod2

2. Preemption

   优先级抢占过程：

   1. pod2 进行调度，分配到 node1 上运行
   2. pod1 再进行调度，发现 node1 上的资源不满足，调度失败进入抢占模式
   3. 在经过抢占算法后，选中 pod2 作为 pod1 的让渡者
   4. 驱逐 node1 上的 pod2 ，调度 pod1 到 node1上

   挑选被抢占节点的策略在；

   1. 优先选择打破 PDB 最小的节点
   2. 其次，选择待抢占 pods中最大优先级最小节点
   3. 再次选择待抢占pods 优先级加和最小节点
   4. 接下来选择待抢占pods数目最小节点
   5. 最后选择最晚启动节点

#### 4. 调度器架构简介和具体算法
