* [可观测性：应用健康](#可观测性应用健康)
   * [1. 需求来源](#1-需求来源)
      * [1. 应用迁移到 k8s ，如何保障应用的健康稳定？](#1-应用迁移到-k8s-如何保障应用的健康稳定)
   * [2. Liveness与 Readiness](#2-liveness与-readiness)
      * [1. Liveness probe 存活探针](#1-liveness-probe-存活探针)
      * [2. readiness probe 就绪探针](#2-readiness-probe-就绪探针)
   * [3. 应用健康状态](#3-应用健康状态)
      * [1. 使用方式](#1-使用方式)
   * [4. 应用故障排查-状态机制](#4-应用故障排查-状态机制)

Created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc)

### 可观测性：应用健康

#### 1. 需求来源

##### 1. 应用迁移到 k8s ，如何保障应用的健康稳定？

1. 提高应用的可观测性
   1. 应用的健康状态
   2. 应用资源的使用
   3. 应用实时日志
2. 提高应用的可恢复能力
   1. 降低问题的影响范围，进行问题的调试、诊断
   2. 应用出现问题通过自愈机制恢复

#### 2. Liveness与 Readiness

##### 1. Liveness probe 存活探针  

判断是否存活，如果不存活是否利用机制拉起来

##### 2. readiness probe 就绪探针 

判断 pod 是否就绪可以对外提供服务，接入层会将外界的流量才会打上去

#### 3. 应用健康状态

##### 1. 使用方式

1. 探测方式

   1. httpGet 通过发送 HTTP 请求，根据请求返回的状态码进行判断（200-399之间表示健康）

      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: liveness
      spec:
        containers:
          image: k8s.gcr.io/liveness
          args:
          - /server
          restartPolicy: Never
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
              httpHeaders:
              - name: Custom-Header # 用 head 头判断
                value: Awesome
              initialDelaySeconds: 5 # pod 启动之后多久执行，单位: s
              periodSuccess: 5 # 检查间隔时间，默认10s
              timeoutSeconds: 1 # 探针超时时间 默认 1s
              successThreshold: 1 # 探针失败后再次判断成功的阈值 默认1
              failureThreshold: 3 # 探测失败重试的次数
      ```

      

   2. Exec 命令返回是0 的时候为健康 

      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: busybox
      spec:
        containers:
          image: busybox
          conmand: ["/bin/sh", "-c", "ls /etc/config"]
          restartPolicy: Never
          livenessProbe:
            exec:
              command:
              - cat
              - /tmp/healthy
              initialDelaySeconds: 5 # pod 启动之后多久执行，单位: s
              periodSuccess: 5 # 检查间隔时间，默认10s
            
      ```

      

   3. tcpSocket 通过容器的 ip 和 port 执行TCP检查，如果能建立TCP 连接的话表明容器健康

      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: goproxy
      spec:
        containers:
          image: k8s.gcr.io/goproxy:0.1
          ports:
          - containerPort: 8080
          restartPolicy: Never
          livenessProbe:
            tcpSocket:
              port: 8080
              initialDelaySeconds: 5 # pod 启动之后多久执行，单位: s
              periodSuccess: 5 # 检查间隔时间，默认10s
          livenessProbe:
            tcpSocket:
              port: 8000
              initialDelaySeconds: 15 # pod 启动之后多久执行，单位: s
              periodSuccess: 20 # 检查间隔时间，默认10s
      ```

      

2. 探测结果

   1. Success 通过检查
   2. Failure 未通过检测，进行操作（Readiness 不把流量打到  liveness会判断是否需要重启  ）
   3. Unkonwn 未能执行检查，不采取任何行动

3. 重启策略

   1. Always 
   2. OnFailure
   3. Never

4. 注意事项

   1. 调大判断的超时阈值，防止容器压力过高发生偶发超荷
   2. 调整判断次数阈值，设置为3 在短周期内不是最佳实践
   3. exec 如果执行的是脚本判断的话，可能会实践比较长
   4. 使用 tcpSocket 使用 TLS ，需要业务层判断是否有影响

#### 4. 应用故障排查-状态机制

Pod: pending 会到 running 或者 unknown 然后转换到 successed 或者 failed

其中 pod 的 status 显示目前 pod 的状态，containers.status 具体到容器的状态， conditions 书具体细节状态：例如是否初始化、pod是否 ready、容器是否ready、pod 是否调度成功，也可以通过 event 监控

pendig： 未调度，大概率和资源有关

Wating: 已调度，可能未能下载镜像

Crashing: 完成调度，下载镜像，但是启动未成功,可能是配置或者权限

Running: 启动起来但是未正常工作可能是字段不对（一般 kubectl apply 可以检查，但是 api 不一定）

service未正常工作： 排查网络插件自身，检查 label（和 pod 不一致）,也可以检查 endpoints



应用远程调试：

1. Pod

```bash
kubectl exec -it podname -c containerName /bin/bash
```

2. service

   当集群中的应用依赖的应用需要本地调试，可以使用Telepresence 将本地应用代理到集群中的 service 上面

   ```bash
   Telepresence --swap-deployment $Deployment_name
   ```

   本地开发的应用调用集群服务，将远程应用的流量代理到本地端口

   ```bash
   kubectl port-forward svc/app -n app-namespace
   ```

   [kubectl-debug](https://github.com/aylei/kubectl-debug) 方便调试

   

   

