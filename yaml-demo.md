## Pod

### 运行两个容器的 demo

```yaml
apiVersion: v1 #指定当前描述文件遵循v1版本的Kubernetes API
kind: Pod #我们在描述一个pod
metadata:
  name: twocontainers #指定pod的名称
  namespace: default #指定当前描述的pod所在的命名空间
  labels: #指定pod标签
    app: twocontainers
  annotations: #指定pod注释
    version: v0.5.0
    releasedBy: david
    purpose: demo
spec:
  containers:
  - name: sise #容器的名称
    image: quay.io/openshiftlabs/simpleservice:0.5.0 #创建容器所使用的镜像
    ports:
    - containerPort: 9876 #应用监听的端口
  - name: shell #容器的名称
    image: centos:7 #创建容器所使用的镜像
    command: #容器启动命令
    - "bin/bash"
    args:
    - "-c"
    - "sleep 10000"
```



### 使用 Probe Handler 探测 Pod 的 demo：

使用三种 Probe：

- startupProbe
- livenessProbe
- readinessProbe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
  namespace: default
spec:
  containers:
  - name: sise
    image: quay.io/openshiftlabs/simpleservice:0.5.0
    ports:
    - containerPort: 9876
    readinessProbe:
      tcpSocket:
        port: 9876
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /health
        port: 9876
      periodSeconds: 5
    startupProbe:
      httpGet:
        path: /health
        port: 9876
      failureThreshold: 3
      periodSeconds: 2
```

在 Kubelet 创建好对应的容器以后，会先运行 startupProbe 中的配置，这里我们用 HTTP handler 每隔 2 秒钟通过 http://localhost:9876/health 来判断服务是不是启动好了。这里我们会尝试 3 次检测，如果 6 秒以后还未成功，那么这个容器就会被干掉。而是否重启，这就要看 Pod 定义的重启策略。

一旦容器通过了 startupProbe 后，Kubelet 会每隔 5 秒钟进行一次探活检测 （livenessProbe），每隔 10 秒进行一次就绪检测（readinessProbe）。



### 使用 lifecycle 完成 Pod 优雅启停

两个 lifecycle handle：

- postStart：应用启动后执行一些操作
- preStop：应用退出之前执行一些操作，阻塞应用退出

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
  namespace: default
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx:1.19
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello world"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```



### init 容器

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-container
  namespace: default
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.31
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initcontainers:
  - name: init-container
    image: busybox:1.31
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox:1.31
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```



## Deployment

nginx-demo：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: nginx-deployment
    labels:
        app: nginx
spec:
    replicas: 5
    selector:
        matchLabels:
            app: nginx
    template:
        metadata:
            labels:
                app: nginx
                version: v1
        spec:
            containers:
            - name: nginx
              image: nginx:1.7.9
              ports:
              - containerPort: 80
```



