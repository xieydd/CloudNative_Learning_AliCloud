### Sp: 从0构建一个云原始应用

#### 1. Helm Chart

##### 1. Why?

Easy to Create and Delete

```bash
# Helm v3
helm install  redis stable/redis
helm delete redis
```

#### 2. How to build a hello world app?

##### 1. 构建程序 （golang、Java、python...）

```go
package main

import (
	"os"
  "net/http"
  "fmt"
)

func main() {
  port := os.Getenv("PORT")
  if port == "" {
    port = "80"
  }
  username := os.Getenv("USERNAME")
  if username == "" {
    username = "world"
  }
  helloFunc := func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello %s\n", username)
  }
  http.HandleFunc("/", helloFunc)
  
  http.ListenAndServe(":"+port, nil)
}
```

##### 2. 构建打包镜像（环境加编译）

```dockerfile
FROM golang:1.13-rc-alpine3.10 as builder
WORKDIR /app
COPY main.go .
RUN go build -o hello-world main.go

FROM alpin:3.10
WORKDIR /app
ARG PORT=80
COPY --from=builder /app/hello-world /app/hello-world
ENTRYPOINT ./hello-world
EXPOSE 80
```

##### 3. 创建空白   helm 应用 

```bash
helm  create my-hello-world
```

​	大致修改相应的变量

##### 4. 检查创建的应用

```bash
helm lint --strict my-hello-world
```

##### 5. 打包

```bash
helm package my-hello-world
```

##### 6. 使用打包的应用创建

```bash
helm create my-hello-world-chart-test my-hello-world-0.1.0.tgz
```

##### 7. 验证服务

```bash
kubectl get pods
kubectl port-forwrd my-hello-world-chart-test-xxx 8080:80
curl localhost:8080
```

##### 8. 如果需要进行参数覆盖

```bash
1. helm install my-hello-world-chart-test my-hello-world-0.1.0.tgz --set Username="xieydd"
2. 创建 my-value.yaml 写入
	 helm install my-hello-world-chart-test my-hello-world-0.1.0.tgz -f my-value.yaml
```

