* [Lec24: K8s Operation  Framework](#lec24-k8s-operation--framework)
   * [1. 介绍](#1-介绍)
      * [1. 简单概念](#1-简单概念)
      * [2. 常见 operator 流程](#2-常见-operator-流程)
   * [2. Operation Framework](#2-operation-framework)
      * [1. 主流的框架](#1-主流的框架)
      * [2. 案例描述](#2-案例描述)

Created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc)

### Lec24: K8s Operation  Framework

#### 1. 介绍

##### 1. 简单概念

1. CRD（用户自动以资源）
2. CR: 自定义资源的具体 实例
3. Webhook: webhook 关联到 apiserver，实际上是一种 HTTP 回调，基于 web 应用实现 webhook 会在特定的事件发生时将消息发送到特定的 URL，一般有两种 webhook: mutating webhook(变更传入对象)、validating webhook(传入对象验证，只读)
4. 工作队列： controller 的核心组件，controller 会监控集群资源对象的变化，将资源的时间（动作和 key ）存在工作队列中
5. controller: controller 控制集群资源，关联一个工作队列、每一个 controller 的目的试讲资源的 status 向 spec 进行转化
6. operator: operator=crd + controller + webhook

##### 2. 常见 operator 流程

1. 创建 CRD
2. apiserver 转发请求给 webhook
3. webhook 负责 CRD 缺省值配置和配置项校验
4. controller 获取 CRD，处理关联逻辑
5. controller 实时检测 CRD 相关集群信息反馈到 CRD 状态

#### 2. Operation Framework

给用户提供了 webhook 和 controller 框架，包括消息通知、失败重新入队，开发人员只需要关注应用的运行逻辑即可

##### 1. 主流的框架

[kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)

[operator-sdk](https://github.com/operator-framework/operator-sdk)

共同点是都是用官方的 controller-tool 和 controller-runtime

不同点在于：kubebuilder 在相应的猜测是、部署、代码生成脚本更加完善，如 Makefile 和 Kustomize 等；operator sdk 在上层例如 Ansible Operator 和 Operator Lifecycle Manager 集成

##### 2. 案例描述

[sidercarset](https://github.com/openkruise/kruise/blob/master/pkg/apis/apps/v1alpha1/sidecarset_types.go) 属于 kruise 的一个资源，将通用的逻辑外置后插入 pod ,注意 e8d836 commit 更适合学习

1st step 初始化:

```shell
# domain 是 CRD 的 Group
kubebuilder init --domain=unisound.io
```

2st step 创建 api 和 controller 框架:

```shell
# group+domain 即 apps.unisound.io
# version: v1alpha 初始版本易更改、v1beta1 api 稳定向后兼容，特性可能调整、v1 api 和特性都已经稳定
# kind: CRD 的资源类型
# namespaced 全局唯一还是 Namespace 唯一
kubebuilder create api --group apps --version v1alpha1 --kind SidecarZSet --namespaced=false
```

3st step 填充 CRD:

1. 修改 pkg/apis/apps/v1alpha1/sidecarset_types.go ， code generator 依赖注释生成代码，故可能需要修改注释

   ```go
   +genclient:nonNamespaced //生成非 namespace 对象
   +kubebuilder:subsource:status  //生成 status 子资源
   +kubebuilder:printcolumn:name="MATCHED",type="integer",JSONPath=".status.matchedPods",description="xxx": kubectl get sidecarset 
   ```

   填充 sidecarSetSpec 和 sidecarSetStatus 字段

2. 运行 make 生成代码

4st step: 生成 webhook

1. 生成mutating webhook

   ```shell
   kubebuilder alpha  webhook --group apps --version v1alpha1 --kind SidecarSet --type=mutating --operation=create
   
   kubebuilder alpha  webhook --group core --version v1alpha1 --kind Pod --type=mutating --operation=create #审查各行 Pod 资源对应的 webhook
   ```

2. 生成 validating webhook

   ```shell
   kubebuilder alpha webhook --group apps --version v1alpha1 --kind SidecarSet --type=validating --operation=create,update
   ```

   Group/kind 描述需要处理的资源

   Type 描述需要生成那种类型的框架

   operation s描述关注资源对象的哪些操作

5st: 填充 webhook

在代码中已标记的 TODO

1. 是否需要 Kube client 注入，取决于  webhook 是否需要处理资源对象，是否需要其他资源去获取
2. mutatingSidecarSetFn 或者 alidatingSidecarSetFn (待操作对象指针已经传入，我们直接调整该对象的属性即可完成)

6st: 填充 controller 框架

1. 修改权限注释

   框架会自动生成形如 //+kubebuilder:rbac:groups=apps,resources=deployments/status,verbs=get;update;patch 的注解，按需修改

2. 增加入队逻辑

   CRD本身的入队逻辑会自动填充，但是关联资源例如 pods 需要手动新增入队逻辑

3. 填充业务逻辑

   修改 Reconcile 函数，处理工作队列，根据 spec 完成逻辑，将结果反馈到 status

4. Reconcile 函数的 return reconcile.Result{}, err 会重新入队列
