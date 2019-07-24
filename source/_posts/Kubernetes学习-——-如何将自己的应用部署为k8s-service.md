title: Kubernetes学习 —— 如何将自己的应用部署为k8s service
author: 天渊
tags:
  - k8s
  - devops
  - 云原生
categories: []
date: 2019-07-12 21:14:00
---
`Service`是kubernetes对用户应用服务的一层抽象封装，一个`Service`对应多个具有相同功能的应用实例（`Pod`），为外界访问服务提供统一的入口，将请求负载均衡分发到多个`Pod`上
<!--more-->
用户在k8s上将自己的应用发布为`Deployment`后，只能通过`kubernetes Proxy`间接访问`Pod`的形式来调用服务，由于`Pod`生命周期的不确定性，这种方法可行性不高，因此需要将应用程序以`Service`的形式进行暴露，将应用程序实例和服务抽象进行充分解耦，集群中其他服务对该服务的调用就不会受到集群down机和动态缩/扩容的影响，用户在调试时也可以通过Node Port的方式直接在外界访问这个服务

#### 发布一个Nginx服务

将应用程序发布为`Service`有以下几个基本步骤：

1. 创建docker image
2. 基于应用程序的Docker Image发布k8s deployment，并设置需要暴露的端口和副本数
3. 查看`replica set`和`pod`的状态，并指定`Labels`和`Selector`
4. 将`deployment`暴露为`service`

##### 创建docker image

（略）

##### 发布deployment

`deployment`是k8s提供的用于发布无状态服务的资源形式，对应由`Deployment Controller`对用户发布的无状态应用程序进行统一管理，基于`deployment`可以随时启动，删除和动态缩/扩容`pod`，并暴露为外界可调用的`service`

一旦应用发布为`deployment`，`Deployment Controller`便创建相应的`ReplicaSet`和`Pod`，并交给k8s scheduler调度到有空闲资源的服务器节点上启动运行

用`kubectl run`命令直接创建一个Nginx的`deployment`（类似于docker run命令创建容器）：

```shell
kubectl run nginx --image=nginx:latest --port=80 --replicas=1
```

1. run后面指定该`deployment`的名称，这个名称是该应用在集群中的唯一标识
2. `--image`是必带参数，指定Docker镜像，k8s会自动从远程registry拉取所需要的docker镜像
3. `--port`是可选参数，指定该`deployment`的`pod`需要暴露的端口号，比如nginx服务就需要暴露它的80端口
4. `--replicas`是可选参数，指定`pod`副本数量

启动完成后，`Deployment Controller`会自动创建`Replica Set`（管理pod的副本集）和多个`pod`（由`--replicas`参数指定）

用kubectl get deployments命令查看创建的nginx deployment (如果不指定名称nginx，则显示所有的deployment)：

```shell
[irteam@dev-ncc-client-ncl ~]$ kubectl get deployments nginx
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           17h
```

使用describe命令查看这个deployment的配置细节：

```shell
[irteam@dev-ncc-client-ncl ~]$ kubectl describe deployments nginx
Name:                   nginx
Namespace:              default
CreationTimestamp:      Mon, 08 Jul 2019 17:46:20 +0900
Labels:                 run=nginx
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               run=nginx
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  run=nginx
  Containers:
   nginx:
    Image:        registry.navercorp.com/ncp-image/ncp-nginx:latest
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-79b9dfdd46 (1/1 replicas created)
Events:          <none>
```

在这里面可以查看当前deployment的状态，比如名称，namespace，创建时间，当前副本整体状态，还有就是`Labels`和`Selector`，用于后续`replica set`和`pod`还有`service`和`pod`之间的配对关系

##### 查看`replica set`和`pod`

使用`kubectl run`命令创建`deployment`的话，会自动创建默认的`replica set`和`pod` ，这也是k8s官方推荐的方式 （如果不采用这种方式，则需要自己指定template然后使用`kubectl create`分别创建deployment, replica set和pod）

使用以下命令查看刚创建的`replica set`

```shell
[irteam@dev-ncc-client-ncl ~]$ kubectl get replicasets
NAME               DESIRED   CURRENT   READY   AGE
curl-6bf6db5c4f    1         1         1       25h
nginx-79b9dfdd46   1         1         1       18h
```

默认创建的`replica set`是nginx-79b9dfdd46，使用`kubectl describe`命令查看详情：

```shell
[irteam@dev-ncc-client-ncl ~]$ kubectl describe replicaset nginx-79b9dfdd46
Name:           nginx-79b9dfdd46
Namespace:      default
Selector:       pod-template-hash=79b9dfdd46,run=nginx
Labels:         pod-template-hash=79b9dfdd46
                run=nginx
Annotations:    deployment.kubernetes.io/desired-replicas: 1
                deployment.kubernetes.io/max-replicas: 2
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/nginx
Replicas:       1 current / 1 desired
Pods Status:    1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  pod-template-hash=79b9dfdd46
           run=nginx
  Containers:
   nginx:
    Image:        registry.navercorp.com/ncp-image/ncp-nginx:latest
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:           <none>
```

参数与`deployment`差别不大，可重点关注以下参数：

1. `deployment.kubernetes.io/desired-replicas`和`deployment.kubernetes.io/max-replicas`参数，配置了期望副本数和最大副本数
2. Controlled By参数，说明是由nginx的这个`deployment`来进行管理
3. Pods Status：当前管理的pod状态
4. Replicas：当前副本集状态，`1 current / 1 desired`说明当前已经成功启动一个pod，并且期望的pod副本数也为1个

使用以下命令查看`pod`：

```shell
[irteam@dev-ncc-client-ncl ~]$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
curl-6bf6db5c4f-4mkxk    1/1     Running   0          18h
nginx-79b9dfdd46-qc94z   1/1     Running   0          18h
```

可以看到，刚创建的默认pod只有一个，status为Running，说明当前运行健康（若为Pending或者Unknown等状态说明pod调度失败）

查看`pod`详情：

```shell
[irteam@dev-ncc-client-ncl ~]$ kubectl describe pods nginx-79b9dfdd46-qc94z
Name:           nginx-79b9dfdd46-qc94z
Namespace:      default
Priority:       0
Node:           dev-ncc-slave-1-ncl/10.106.147.158
Start Time:     Mon, 08 Jul 2019 18:58:40 +0900
Labels:         pod-template-hash=79b9dfdd46
                run=nginx
Annotations:    <none>
Status:         Running
IP:             10.244.2.2
Controlled By:  ReplicaSet/nginx-79b9dfdd46
Containers:
  nginx:
    Container ID:   docker://4c06715be9d3fc575285621f595c5c2d9f67ef5fbd6d792618f0fb3449f85892
    Image:          registry.navercorp.com/ncp-image/ncp-nginx:latest
    Image ID:       docker-pullable://registry.navercorp.com/ncp-image/ncp-nginx@sha256:650cfc6f4e39b5bd5ec6bc57063886ba6e8808d691ac99200ac39fac2252c6ea
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 08 Jul 2019 18:59:01 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-zbdxq (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-zbdxq:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-zbdxq
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>
```

重点关注以下参数：

1. Node：当前pod被调度至服务器节点的host name
2. Labels：标签，只有当labels与之前`replica set`的`Selector`保持一致，才会被相应的`Replica Controller`纳入管理进行动态缩/扩容；后续的`service`也是通过标签来discover当前有效的pod；标签可以在运行时动态修改
3. IP：pod在k8s集群内部的ip
4. Controlled By：标明当前pod是由哪个`replica set`进行管理
5. Containers：pod封装的容器信息，一个pod可以有多个容器
6. Tolerations：指定该pod多长时间未达到Ready状态或者k8s多长时间未检测到pod心跳后，允许k8s重新调度pod
7. Events：pod经历的事件，deployment的滚动升级和缩/扩容等都会产生事件

在确认`replica set`和`pod`状态确认无误后，即可将该应用暴露为服务

##### 暴露服务

使用`kubectl expose`命令将nginx deployment暴露为服务：

```shell
kubectl expose deployment/nginx --type="NodePort" --port 80
```

1. --type：当前暴露形式，指定`NodePort`的话，k8s会给当前服务随机分派一个30000-32767之间的端口号，外界可以直接通过`服务器node ip + node port `的方式访问这个服务
2. --port：指定应用所需要暴露的端口，nginx服务则需要暴露他的80端口

查看暴露的服务：

```shell
$ kubectl get services
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes            ClusterIP   10.96.0.1       <none>        443/TCP          12m
nginx                 NodePort    10.109.107.109   <none>        80:31482/TCP   5m1s
```

查看详情：

```shell
[irteam@dev-ncc-client-ncl ~]$ kubectl describe services nginx
Name:                     nginx
Namespace:                default
Labels:                   run=nginx
Annotations:              <none>
Selector:                 run=nginx
Type:                     NodePort
IP:                       10.109.107.109
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31482/TCP
Endpoints:                10.244.2.2:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

默认情况下，`service`名称和`deployment`名称保持一致，其他参数：

- Selector：`service`通过`Selector`选择来匹配对应的`pod`，k8s会通过`Endpoints Controller`来定期更新健康的符合`Selector`匹配规则的`pod`路由表，在这里nginx service将寻找所有labels为run=nginx的`pod`作为路由对象
- IP：`service`在集群中的唯一ip地址

`service`的`NodePort`是31482，因此我们可以直接在本机使用localhost访问这个nginx服务了：

```shell
[irteam@dev-ncc-client-ncl ~]$ curl localhost:31482
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
...
```

至此已经成功发布nginx service

`Service`和`Deployment`,`Replica Set`还有`Pod`之间的关系：

![upload successful](\blog\images\image.png)

一个`deployment`可以创建多个`replica set`和`pod`以及`service`，`replica set`和`service`通过`Selector`指定的值来匹配带有相关`Labels`的`pod`

#### k8s内部的服务发现

k8s内部通过何种方式发现我们发布的`nginx service` ?

目前有三种方式：

1. `NodePort方式`：即上面通过`node ip + node port`将访问路径固定，这种方式不够灵活，通常只能用于外界调试

2. `环境变量方式`：k8s默认会在每个 pod 启动时候会把所有服务的 IP 和 port 信息配置到当前pod的环境变量中，这样 pod 中的应用可以通过读取环境变量来获取依赖服务的地址信息。这种方式服务和环境变量的匹配关系有一定的规范，使用起来也相对简单，但是有个很大的问题：依赖的服务必须在 pod 启动之前就存在，不然是不会出现在环境变量中的。

3. `kube-dns`方式：k8s官方推荐通过`kubeDNS + dnsmasq`的方式配置kube-dns插件，kube-dns可以缓存所有已经存在的`service`信息供服务调用方发现并调用服务，其他服务可以直接使用以下方式调用nginx服务：

   ```shell
   http://<service_name>.<namespace>.svc.<domain>:80/
   ```

   `service_name`：即服务名nginx

   `namespace`：k8s命名空间，创建deployment时不特别指定的话，`namespace`均为"default"

   `domain`：域名后缀，默认为`cluster.local`

   在 `pod` 中访问也可以使用缩写 `service_name.namespace`，如果 pod 和 service 在同一个 `namespace`，可以直接使用 `service_name`，因此如果同一个`namespace`有其他服务要访问nginx，则直接使用`nginx`作为域名即可：

   ```shell
   http://nginx:80/
   ```

##### 测试服务发现

手动测试`service`的服务发现需要进入到`pod`内部，执行以下命令进入`pod`内部环境，进入到`pod`后可以通过curl命令访问`http://nginx:80/` (镜像没有安装curl，可用yum进行安装)：

```shell
[irteam@dev-ncc-client-ncl ~]$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
curl-6bf6db5c4f-4mkxk    1/1     Running   0          22h
nginx-5ff9d6cc77-5nxpn   1/1     Running   0          59m
[irteam@dev-ncc-client-ncl ~]$ kubectl exec nginx-5ff9d6cc77-5nxpn -it -- /bin/bash
root@nginx-5ff9d6cc77-5nxpn:/# curl nginx:80/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
......
```

使用`Ctrl P + Ctrl Q`命令退出`pod`环境

#### 删除service和deployment

`kubectl`工具提供一键式删除`service`和`deployment`，当`deployment`被删除后，对应的`pod`同时被回收

使用以下命令删除nginx service:

```shell
[irteam@dev-ncc-client-ncl ~]$ kubectl delete services nginx
service "nginx" deleted
```

使用以下命令删除nginx deployment:

```shell
[irteam@dev-ncc-client-ncl ~]$ kubectl delete deployments nginx
deployment.extensions "nginx" deleted
```

随后再查看`pod`可以发现之前创建的`pod`均被删除