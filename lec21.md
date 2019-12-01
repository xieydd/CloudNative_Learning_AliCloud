* [Lec21: Kubernetes  存储架构和插件](#lec21-kubernetes--存储架构和插件)
   * [1. Kubernetes 存储体系架构](#1-kubernetes-存储体系架构)
      * [1. Kubernetes 挂载 Volume 过程](#1-kubernetes-挂载-volume-过程)
      * [2.Kubernetes 存储架构](#2kubernetes-存储架构)
      * [3. PV Controller 介绍](#3-pv-controller-介绍)
      * [4. PV Controller 实现](#4-pv-controller-实现)
      * [5. AD Controller](#5-ad-controller)
      * [6. Volume Manager](#6-volume-manager)
      * [7.  Volume Plugin](#7--volume-plugin)
      * [8.  Volume Plugins 插件管理](#8--volume-plugins-插件管理)
      * [9. K8s 存储卷调度](#9-k8s-存储卷调度)
   * [2. FlexVolume 介绍和使用](#2-flexvolume-介绍和使用)
      * [1. Flexvolume 接口](#1-flexvolume-接口)
      * [2. FlexVolume 挂载分析](#2-flexvolume-挂载分析)
      * [3. Flexvolume 代码示例](#3-flexvolume-代码示例)
      * [4. FlexVolume 使用](#4-flexvolume-使用)
   * [3. CSI 的介绍和使用](#3-csi-的介绍和使用)
      * [1. CSI  介绍](#1-csi--介绍)
      * [2. CSI  系统结构](#2-csi--系统结构)
      * [3. CSI 对象](#3-csi-对象)
      * [4. CSI 组件](#4-csi-组件)
      * [5. CSI  部署](#5-csi--部署)
      * [6. CSI 使用示例](#6-csi-使用示例)
      * [7. CSI  其他功能](#7-csi--其他功能)
      * [8. CSI 近期 features](#8-csi-近期-features)

Created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc)

### Lec21: Kubernetes  存储架构和插件

#### 1. Kubernetes 存储体系架构

##### 1. Kubernetes 挂载 Volume 过程

![image-20191124152718064](./images/image-20191124152718064.png)

1. 创建  pvc 和 包含 pvc 的pod
2. PV Controller 发现这个 pvc 处于未绑定状态，调用 volume  plugin 创建存储卷和PV 对象，并将 PV 和 PVC 绑定
3. Scheduler 调度 pod 到 worker 上
4. AD controller 发现 pod 和  PVC 处于待挂载状态，调用 Volume Plugin 将设备挂载到目的节点上
5. worker 节点等到设备挂载完成，kubelet 通过 Volume plugin 将设备挂载到指定位置
6. 完成后启动 containers ，包含映射文件

##### 2.Kubernetes 存储架构

![image-20191124163818614](./images/image-20191124163818614.png)

1. PV  controller: 实现 PV  和 PVC 绑定，声明周期管理，根据需求进行数据卷的 Provision/Delete 操作
2. AD Controller 实现存储设备的 Attach/Delete 操作，将设备挂载到目标节点
3. Volume Manager: 管理卷的 Mount 和  UnMount 和卷的格式化
4. Volume Plugin: 拓展卷的类型管理能力，针对第三方存储操作和 Kubernetes 存储相结合
5. Scheduler: 实现 pod 的调度，并且结合存储卷的需求

##### 3. PV Controller 介绍

![image-20191124164625919](./images/image-20191124164625919.png)

1. PV(Persistent Volume): 持久化存储卷，详细定义预挂载空间的各项参数，无 Namespace 限定
2. PVC(Persistent Volume Claim) 持久化存储声明，用户使用的存储接口，无感知，限定在 Namespace 
3. StorageClass：存储类，创建 PV 存储的模板，根据SC定义的模板创建存储卷，创建 PV 
4. PVC 和 PV 是一一对应关系

##### 4. PV Controller 实现

![image-20191124171845137](./images/image-20191124171845137.png)

1. Claim Worker:
   1. 实现 PVC 的状态迁移
   2. 通过系统标签 pv.kubernetes.io/bind-completed 判断 pvc 是 bound 状态
   3. 当 PVC 是 unbound 的时候，如果找到合适的 PV ，则进行 bound, 如果找不到触发 Provision(不是 In-Tree Provisioner 则等待)
2. Volume Worker
   1. 实现 PV 的状态迁移
   2. 通过  PV  的ClaimRef 判断 PV 是Bound 还是Released ，如果 PV 状态为 Release ，Reclaim 为 delete 的时候，执行删除操作

##### 5. AD Controller

![image-20191124173419516](./images/image-20191124173419516.png)

1. DesiredStateOfWorld 即 spec； ActualStateOfWorld 是 Status
2. desiredStateOFWorldPopulation 指的是同步集群全局变化； reconcile 会根据全局的变化执行将当前 status 转换到 spec 的状态

##### 6. Volume Manager

![image-20191201114135406](./images/image-20191201114135406.png)

注意 AD controller 有 attach 和 detach 的操作，volume manager 也有，到底谁来做？

可以通过 kubelet 的标签 `--enable-controller-attach-detach` 来指定， True AD,False VM 做

##### 7.  Volume Plugin

![image-20191201114931489](./images/image-20191201114931489.png)

##### 8.  Volume Plugins 插件管理

![image-20191201115350775](./images/image-20191201115350775.png)

##### 9. K8s 存储卷调度

![image-20191201115638366](./images/image-20191201115638366.png)

#### 2. FlexVolume 介绍和使用

![image-20191201124423516](./images/image-20191201124423516.png)

##### 1. Flexvolume 接口

![image-20191201125226614](./images/image-20191201125226614.png)

##### 2. FlexVolume 挂载分析

![image-20191201125616300](./images/image-20191201125616300.png)

##### 3. Flexvolume 代码示例

![image-20191201130653711](./images/image-20191201130653711.png)

##### 4. FlexVolume 使用

![image-20191201130918971](./images/image-20191201130918971.png)

#### 3. CSI 的介绍和使用

##### 1. CSI  介绍

![image-20191201131226181](./images/image-20191201131226181.png)

##### 2. CSI  系统结构

![image-20191201131645041](./images/image-20191201131645041.png)

##### 3. CSI 对象

![image-20191201142824857](./images/image-20191201142824857.png)

##### 4. CSI 组件

1. Node-Driver-Registry

   ![image-20191201143415897](./images/image-20191201143415897.png)

2. External Attacher

   ![image-20191201143814011](./images/image-20191201143814011.png)

##### 5. CSI  部署

![image-20191201144211946](./images/image-20191201144211946.png)

##### 6. CSI 使用示例

![image-20191201144251810](./images/image-20191201144251810.png)

##### 7. CSI  其他功能

![image-20191201144438948](./images/image-20191201144438948.png)

##### 8. CSI 近期 features

![image-20191201144634674](./images/image-20191201144634674.png)
