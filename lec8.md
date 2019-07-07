### 8. 应用配置管理

#### 1. 需求

1. 不可变基础设施（pod）的可变配置 ConfigMap
2. 敏感信息存储（Secret token）Secret
3. 集群中 pod 自我身份验证 ServiceAccount
4. 容器运行的资源配置 Spec.Containers[].Resources.Limits/requests
5. 容器的运行安全管控 Spec.Containers[].SecurityContext
6. 容器启动前的校验，例如确认 DNS 正确，网络连通 Spec.InitContainers

#### 2. ConfigMap

容器运行时需要的一些配置信息，环境变量或者命令行参数。目的是为了解耦镜像和可变参数，便于移植

例如 flannel 的 配置文件

##### 1. 如何创建 configmap

两种方式创建 configmap 中的 data

1. 文件名：文件

1. ```
   $ kubectl create configmap kube-flannel-cfg --from-file=xx.json -n kube-system
   data: 
     cni-config.json: |
       {
       xxx
       }
   ```

2. 指定键值对

   ```
   $ kubectl create configmap special-config --from-literal=special.how=very 
   data:
     special.how: very
   ```

##### 2. 如何使用 ConfigMap

1. 配置环境变量

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: busybox
   spec:
     containers:
       image: busybox
       conmand: ["/bin/sh", "-c", "env"]
       env:
         - name: SPECIAL_LEVEL_KEY
           valueFrom:
             configMapKeyRef:
               name: special-config
               key: special.how
       restartPolicy: Never
   ```

2. 用于管控命令行参数

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: busybox
   spec:
     containers:
       image: busybox
       conmand: ["/bin/sh", "-c", "echo ${SPECIAL_LEVEL}"]
       env:
         - name: SPECIAL_LEVEL_KEY
           valueFrom:
             configMapKeyRef:
               name: special-config
               key: SPECIAL_LEVEL
       restartPolicy: Never
   ```

3. 用 configmap 挂载文件

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
       volumeMount:
         - name: config-volume
           mountPath: /etc/config
     volumes:
       - name: config-volume
         configMap:
           name: special-config
         
   ```

   ConfigMap 使用需要注意

   1. 不能超过 1M(etcd 要求)
   2. 只能使用相同 namespace 下面的 configMap
   3. 如果引用的 CM 不存在将会创建失败
   4. envFrom 时出现 key 是数字时引用错误，可以创建 pod
   5. 只有通过 k8s api 才能使用configmap, 而类似 manifest 创建 的static pod 不能使用 configmap



#### 3. Secret

存储密码 token 等敏感信息，使用的是 Base64 编码

```shell
# 文件 base64 编码/解码
openssl base64 -in xx -out xx

# 字符 base64 编码/解码
openssl base64 -e <<< xieyuandong
openssl base64 -d <<< eGlleXVhbmRvbmcK

```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-xieyuandong
  namespace: kube-system
type: Opaque # 常用四种类型 Opaque kubernetes.io/service-account-token kubernetes.io/dockerconfigjson 拉取私有镜像使用的 bootstrap.kubernetes.io/token 用于节点接入集群校验的数据
data:
  username: xieyuandong
  passwd: eGlleXVhbmRvbmcK
  
```

##### 1. 创建 Secret

手动 或者自动（给每一个 namespace 创建默认的 ServiceAccount）

1. 手动

   ```shell
   kubectl create secret genenic [NAME] [DATA] [TYPE]
   ```

2. 指定文件

   ```
   kubectl create secret genenic myregistrykey --from-file=xxx.json -type=kubernetes.io/dockertokenjson
   ```

3. 指定键值对

   ```shell
   kubectl create secret genenic prod-db-secret --from-literal=username=xieyuandong --from-literal=password=xieyuandong
   ```

##### 2. 使用 secret

1. secret 挂载在 pod 的指定目录中

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
       volumeMount:
         - name: secret-volume
           mountPath: /etc/secret
           readOnly: true
     volumes:
       - name: secret-volume
         secret:
           secretName: mysecret
   ```

   

2. serviceaccount 的 secret 自动挂载到目录上，生成 ca.crt  和 token 中

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
       serviceAccount: default
       serviceAccountName: default
       volumeMount:
         - name: default-token-xas23
           mountPath: /var/run/secrets/kubernetes.io/serviceaccount
           readOnly: true
     volumes:
       - name: default-token-xas23
         secret:
           defaultMode: 420
           secretName: default-token-xas23
   ```

##### 3. 如何使用集群中的 secret 访问到私有仓库镜像

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - name: busybox
    images: busybox:latest
  imagePullSecrets:
  - name: regcred
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: default
secret:
- name: default-token-xasds
imagePullSecrets:
- name: myregistrykey
```

自动为使用该 serviceaccount 的 pod 注入 imagepullsecret 信息



##### 4. Secret 使用注意点

1. 大小限制  1MB
2. 虽然使用 base64 编码，但是和明文差不多，需要更高加密的话使用 kubernetes+ vault 加密
3. 最佳实践，限制 List/watch 使用 get 即可



#### 4.  ServiceAccount

serviceaccount 主要解决的是 pod 在集群中的身份认证信息，认证需要的信息在 secret 中

##### 1. Pod 如何访问自身集群信息

[client-go rest InClusterConfig](https://github.com/kubernetes/client-go/blob/master/rest/config.go#L399)

```go
func InClusterConfig() (*Config, error) {
	const (
		tokenFile  = "/var/run/secrets/kubernetes.io/serviceaccount/token"
		rootCAFile = "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
	)
	host, port := os.Getenv("KUBERNETES_SERVICE_HOST"), os.Getenv("KUBERNETES_SERVICE_PORT")
	if len(host) == 0 || len(port) == 0 {
		return nil, ErrNotInCluster
	}

	token, err := ioutil.ReadFile(tokenFile)
	if err != nil {
		return nil, err
	}

	tlsClientConfig := TLSClientConfig{}

  // ca.crt 用于校验服务端
	if _, err := certutil.NewPool(rootCAFile); err != nil {
		klog.Errorf("Expected to load root CA config from %s, but got err: %v", rootCAFile, err)
	} else {
		tlsClientConfig.CAFile = rootCAFile
	}

	return &Config{
		// TODO: switch to using cluster DNS.
		Host:            "https://" + net.JoinHostPort(host, port),
		TLSClientConfig: tlsClientConfig,
    // 使用  token 去认证
		BearerToken:     string(token),
		BearerTokenFile: tokenFile,
	}, nil
}
```

1. Pod 创建的时候 AdmissionController 会将该 namespace 下的 serviceaccount 的secret 挂载到容器的 `var/run/secrets/kubernetes.io/serviceaccount/` 下面
2. 当访问集群的时候如上图代码，会先从 pod 的文件中读取

#### 5. 容器资源类似管理

##### 1. 支持资源类型

- CPU millicore 1 core = 1000 millicore
- Memory byte
- Ephemeral storage 临时存储
- 自定义： 必须为整数

##### 2. 配置方法

- Request/limit spec.containers[].resources.limit/request

##### 3. pod QOS(服务质量) 配置

1. Guaranteed: 请求和限制一样

2. Burestable 至少有一个容器有内存或者 CPU 的 request

3. BestEffort 

   当节点不满足资源会按照 1 2 3 的顺序驱逐 pod

#### 6. Secret Context

Secretcontext 主要是限制 pod 的行为保护系统的安全

##### 1. 3个级别

1. 容器级别
   1. Pod 级别
2. PSP(Pod Security Policy) 对所有 pod 生效

##### 2. 限制访问设置

1. Discretionary access control : 通过 uid 和 gid 控制文件访问权限
2. SELinux: 通过 SELinux 策略配置控制用户、进程等对于问价的控制
3. Privileged 设置容器为特权容器
4. Linux Capabilities: 给特定进程 privileged 的能力
5. AppArmor 控制可执行文件的访问权限（读写文件目录以及网络 端口）
6. Seccomp 控制进程可以操作系统调用
7. AllowPrivilegeEscalation: 控制一个进程是否比父进程更多的权限



#### 7. InitContainer

和 container 的区别：

1. 会先 于 container 启动直到所有的 initcontainer 启动后启动
2. initcontainer 是按照顺序依次启动，普通的是并发启动
3. initcontainer 启动会直接退出，为普通 container 配置文件或者环境校验

Example: [flannel initcontainer prepare  file](https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml#L235)



