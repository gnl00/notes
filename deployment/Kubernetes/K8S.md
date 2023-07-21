# K8s



## 前言

使用 Docker 时可能会出现这样几个问题：

* 如何协调和调度在 Docker 容器内以及不同容器之间运行的服务？
* 如何保证在升级应用程序时不会中断服务？
* 如何监视应用程序的运行状况？
* 如何批量重新启动容器里的程序？
* …

我们需要对容器内的应用服务进行编排、管理和调度，由此催生出 K8s。K8s 主要围绕 Pod 进行工作，Pod 是 K8s 中的最小调度单位，可以包含一个或多个容器。



## 架构

K8s 一般都是以集群的形式出现的，一个 K8s 集群主要包括两部分：Master 节点和 Node 节点。前者又称为主节点；后者又被称为计算节点。一般来说都是单  Master，多个 Node。

> K8s 架构可以看看这篇文章：http://docs.kubernetes.org.cn/251.html



### 组成

Master 节点包括以下内容：

* API Server，整个 K8s 服务对外的接口，供客户端和其他组件调用。
* Controller Manager，负责维护集群的状态，比如副本数量、故障检测、自动扩展、滚动更新等。
* Scheduler，负责对集群内部的资源进行调度。
* etcd，键值数据库，负责保存 K8s 集群的所有数据。



Node 节点包括以下内容：

* Kubelet，负责维护 Node 状态并和 Master 节点通信。

* Kube Proxy，负责实现集群网络服务。

* Pod*，K8s 中部署的最小单位，可以包含一个或多个 Docker 容器。

  > 除了有容器之间紧密耦合的情况下，通常都一个 Pod 中只有一个容器，方便管理不同服务、并易于各自独立服务的扩展。

<br/>



### K8s 集群

K8s 的 Master 节点和多个 Node 工作节点组成 K8s 集群。

![K8s-cluster](https://d33wubrfki0l68.cloudfront.net/99d9808dcbf2880a996ed50d308a186b5900cec9/40b94/docs/tutorials/kubernetes-basics/public/images/module_01_cluster.svg)

Master 负责集群的管理，协调集群中的所有行为/活动，例如应用的运行、修改、更新等。Node 节点作为工作节点，可以是 VM 虚拟机、物理机。



## 使用

### 安装

**前置**

使用 K8s 需要安装下面这些东西：

* 先安装 Docker，作为虚拟化主机（host）；
* K8s 的命令行客户端 kubectl；
* K8s 运行环境，比如 minikube ；



**开始**

1、下载 kubectl

```bash
curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/v1.6.4/bin/linux/amd64/kubectl

chmod +x kubectl
```

2、安装 minikube（以 Linux 为例）

> 参考：https://minikube.sigs.K8s.io/docs/start/

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

3、启动 minicube

```bash
minikube start
```



### 上手

> 这里有一个快速指南还不错：https://zhuanlan.zhihu.com/p/39937913

1、使用 Kubectl 创建 Deployment

```bash
# 创建 sample Deployment
kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
```

当你创建了一个 Deployment，K8s 会创建一个 Pod 实例。Deployment 可以看成是 pod 的部署、管理工具，可以进行 pod 更新，控制副本数量，回滚，重启等操作。

2.1、查看所有的 Deployment

```bash
kubectl get deployments
```

2.2、查看所有 pods

```bash
kubectl get pods
```

> 使用 `kubectl get pods -A` 会把 K8s 系统级 Pods 也显示出来。

3、将 K8s 内网的 Deployment 暴露给外网访问

> Pods that are running inside Kubernetes are running on a private, isolated network. By default they are visible from other pods and services within the same kubernetes cluster, but not outside that network. 

运行在 K8s 内部的 Pods 是使用的是 K8s 私有的、与外部环境相互隔离的网络。默认情况下，Pods 的网络服务只对同一个 K8s 集群环境中的其他 Pods 可见，对外部环境不可见。

因此我们需要使用 `kubectl` 命令来创建代理，将网络请求代理到 K8s 集群内部的、私有的网络里面。

打开新的终端窗口，运行下面的命令，即可完成对 K8s 内部网络的代理。

```bash
kubectl proxy
```

> In order for the new Deployment to be accessible without using the proxy, a Service is required.
>
> 后续讲到 Service 的时候会介绍别的办法来暴露服务。

4、访问测试，现在可以在 K8s 外部测试能够访问到 K8s 内部的服务了

```bash
curl http://localhost:8001/version
```

返回类似下面的结果：

```json
{
  "major": "1",
  "minor": "27",
  "gitVersion": "v1.27.3",
  "gitCommit": "25b4e43193bcda6c7328a6d147b1fb73a33f1598",
  "gitTreeState": "clean",
  "buildDate": "2023-06-14T09:47:40Z",
  "goVersion": "go1.20.5",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

至此，一个简单的上手 demo 就完成了。



### Kubectl

Kubectl 命令管理工具常见的命令如下：

- kubectl get <deployments | pods | services | rs>，列出资源；
- kubectl describe <pods | nodes | services>，显示资源的详细信息；
- kubectl logs，打印 pod 中的容器日志；
- kubectl exec，运行 pod 中容器内部的命令。

<br/>

## Pod

![pods overview](https://d33wubrfki0l68.cloudfront.net/fe03f68d8ede9815184852ca2a4fd30325e5d15a/98064/docs/tutorials/kubernetes-basics/public/images/module_03_pods.svg)

> Pods are the atomic unit on the Kubernetes platform.

Pod 是 K8s 上的最小的可操作单元。当我们在 K8s 上创建 Deployment 的时候，Deployment 会创建具有一个或多个容器的 Pod。

> A Pod is a Kubernetes abstraction that represents a group of one or more application containers (such as Docker), and some shared resources for those containers.

Pod 是一个抽象（逻辑）概念，包含了一个或者多个容器（Docker 容器或者其他容器），同时还包含了容器之间共享资源。

共享资源包括：

* 共享的存储，以 Volume 的形式表示；
* 网络，同一个 Pod 中的容器 IP 地址相同，共享同一片端口区域；
* 每个容器的运行信息，比如容器镜像版本、容器使用的端口等信息。

> A Pod models an application-specific "logical host" and can contain different application containers which are relatively tightly coupled. 

Pod 是一个逻辑上的主机，可以包含多个耦合的容器。比如，一个 Pod 中可以有 Web 后端服务和前端服务。

每个 Pod 都会被绑定到 Node 节点上，直到被终止或删除。



## Node

![node overview](https://d33wubrfki0l68.cloudfront.net/5cb72d407cbe2755e581b6de757e0d81760d5b86/a9df9/docs/tutorials/kubernetes-basics/public/images/module_03_nodes.svg)

Pod 总是运行在 Node 上的。Node 是 K8s 上的*工作节点*，Node 可以是虚拟机或者物理机。Node 由 K8s 的控制面板（*Control Panel*）进行管理。

> *Control Panel* 实际上就是 Master 节点。

一个 Node 可以包含多个 Pod，K8s 通过 Master 自动管理和调度 Node 中的 Pod。

每个 Node 上至少运行以下内容：

* Kubelet，管理 Master 和 Node 节点之间的通信；管理机器上运行的 Pod 和 Container 容器。
* Container runtime，例如 Docker。

<br/>

## Service

![service](https://d33wubrfki0l68.cloudfront.net/cc38b0f3c0fd94e66495e3a4198f2096cdecd3d5/ace10/docs/tutorials/kubernetes-basics/public/images/module_04_services.svg)

Pod 是有生命周期的。当一个 Node 工作节点销毁时，节点上运行的 Pod 也会销毁，ReplicationSet 会动态创建新的 Pod 来保持应用的运行，让集群回到正常的状态。

> A Service in Kubernetes is an abstraction which defines a logical set of Pods and a policy by which to access them. 

Service 是一个抽象的概念，它定义了 Pod 的逻辑分组和访问策略，能达到 Pod 解耦的目的。可以使用 YAML 或 JSON 来创建 Service。

尽管每个 Pod 都有唯一的 IP，但是没有 Service 的控制， Pod 的 IP 地址都不会从 K8s 内部暴露出去。可以指定不同的 type 字段，通过不同的方式将 Service  暴露出去：

* ClusterIP，默认值，IP 只暴露在集群内部。
* NodePort，将 Node 中的对应端口暴露，外部可以通过 `<NodeIP>:<NodePort>` 来访问集群内的服务。
* LoadBalancer，通过云服务提供商的负载均衡器（如果支持）像外部暴露服务。
* ExternalName，通过返回 CNAME 和它的值，将服务映射到 ExternalName 字段。没有任何类型代理被创建。

<br/>

Service  通过 *label selector* 匹配一组 Pod 集合，以对 K8s 中的一组对象进行逻辑分组。Label 是一个 key/value 键值对，主要用来描述以下几个内容的对象：

* 区分生产、开发、测试环境；
* 对 Pod 进行分类；
* 对 Pod 版本进行标记。

![services and labels](https://d33wubrfki0l68.cloudfront.net/7a13fe12acc9ea0728460c482c67e0eb31ff5303/2c8a7/docs/tutorials/kubernetes-basics/public/images/module_04_labels.svg)

Label 可以在 Pod 创建时指定，也可以在任何时间进行修改。



### 创建 Service

1、检查是否存在 Service 

```bash
kubectl get services
```

2、将之前 [Deploment](#上手) 的服务通过 Service 暴露

```bash
# 将 deployment/kubernetes-bootcamp 服务从 K8s 内部暴露
# --type="NodePort" 暴露的方式时 NodePort
# --port 8080 指定服务端口为 8080，表示外部请求通过公开端口进入 K8s 内部后被转发到 Pod 的 8080 端口
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
```

再次执行

```bash
kubectl get services
```

结果如下：

```
NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes-bootcamp   NodePort    10.100.174.206   <none>        8080:30201/TCP   5m59s
```

可以看到现在有了一个 Service，名叫 kubernetes-bootcamp，并将 30201 端口暴露了出去。

此外还可以通过下面的命令，查看 Pod 描述，查看哪个端口被暴露

```bash
kubectl describe services/kubernetes-bootcamp
```

得到以下结果：

```bash
Name:                     kubernetes-bootcamp
Namespace:                default
Labels:                   app=kubernetes-bootcamp
Annotations:              <none>
Selector:                 app=kubernetes-bootcamp
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.100.174.206
IPs:                      10.100.174.206
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30201/TCP
Endpoints:                10.244.0.5:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

可以看到，30201 端口被暴露出去了。

3、获取 Minukube 的 IP

```
minikube ip
```

4、接下来只要通过 `<Minikube IP>:<暴露出来的端口>` 即可访问目标服务

```bash
# 也可以先把 NodePort 和 minikube-ip 先保存起来，再访问
export NODE_PORT="$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')"
# 查看 export 的端口是否正确
echo "NODE_PORT=$NODE_PORT"
# 访问目标服务
curl http://"$(minikube ip):$NODE_PORT"
```

5、按理来说到这里就完了，但是在这里*踩 Minikube 的坑了*：目前的环境是 WSL 下的 Ubuntu 安装 Minikube，底层虚拟化容器是 Docker，通过 `minikube ip` 获取到 IP 后发现无法 ping 通该 IP。

查看 Minikube 文档，发现在 Minikube 中创建一个 service 并暴露网络的流程如下：

> The easiest way to access this service is to let minikube launch a web browser for you:
>
> ```shell
> minikube service hello-minikube
> ```
>
> Alternatively, use kubectl to forward the port:
>
> ```shell
> kubectl port-forward service/hello-minikube 7080:8080
> ```
>
> Tada! Your application is now available at http://localhost:7080/.


执行命令：

```
minikube service kubernetes-bootcamp
```

显示内容大概长这样：

```
|-----------|---------------------|-------------|---------------------------|
| NAMESPACE |        NAME         | TARGET PORT |            URL            |
|-----------|---------------------|-------------|---------------------------|
| default   | kubernetes-bootcamp |        8080 | http://192.168.49.2:30201 |
|-----------|---------------------|-------------|---------------------------|
🏃  Starting tunnel for service kubernetes-bootcamp.
|-----------|---------------------|-------------|------------------------|
| NAMESPACE |        NAME         | TARGET PORT |          URL           |
|-----------|---------------------|-------------|------------------------|
| default   | kubernetes-bootcamp |             | http://127.0.0.1:38451 |
|-----------|---------------------|-------------|------------------------|
🎉  Opening service default/kubernetes-bootcamp in default browser...
👉  http://127.0.0.1:38451
❗  Because you are using a Docker driver on linux, the terminal needs to be open to run it.
```

Minikube 会开启一个 Tunel 将外部的请求转发到对应的端口上。然后就可以通过 http://127.0.0.1:38451 来访问内部服务了。



### 使用 Label

1、查看 Label

在我们使用 `kubectl create deploment` 创建服务的时候，Deploment 会帮我们的 Pod 自动创建一个 Label，可以通过下面的命令查看：

```bash
kubectl describe deployment
```

 显示内容大概如下：

```
Name:                   kubernetes-bootcamp
Namespace:              default
CreationTimestamp:      Thu, 20 Jul 2023 11:58:00 +0800
Labels:                 app=kubernetes-bootcamp
```

2、使用 Label，接下来就可以使用 Label 来过滤信息：

```bash
kubectl get pods -l app=kubernetes-bootcamp
# or
kubectl get services -l app=kubernetes-bootcamp
```

3、创建 Label

```bash
# 在 <your-pod-name>  这个 pod 上创建一个 label：version=v1
kubectl label pods <your-pod-name> version=v1
```

4、查看创建的 Label

```
kubectl describe pods <your-pod-name>
```

返回结果如下：

```
Name:             kubernetes-bootcamp-855d5cc575-w7xxs
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Thu, 20 Jul 2023 11:58:00 +0800
Labels:           app=kubernetes-bootcamp
                  pod-template-hash=855d5cc575
                  version=v1
```

可以看到新创建的 Label 生效了。接下来就可以使用这个 Label 了：

```bash
kubectl get pods -l version=v1
```



### 删除 Service

> To delete Services you can use the `delete service` subcommand. Labels can be used also here:
>
> ```bash
> kubectl delete service -l app=kubernetes-bootcamp
> ```
>
> Confirm that the Service is gone:
>
> ```
> kubectl get services
> ```
>
> This confirms that our Service was removed. To confirm that route is not exposed anymore you can `curl` the previously exposed IP and port:
>
> ```bash
> curl http://"$(minikube ip):$NODE_PORT"
> ```
>
> This proves that the application is not reachable anymore from outside of the cluster. You can confirm that the app is still running with a `curl` from inside the pod:
>
> ```bash
> kubectl exec -ti $POD_NAME -- curl http://localhost:8080
> ```
>
> We see here that the application is up. This is because the Deployment is managing the application. To shut down the application, you would need to delete the Deployment as well.



## 应用伸缩/多实例部署

### 实例扩张

**应用伸缩之前**

![before scale](https://d33wubrfki0l68.cloudfront.net/043eb67914e9474e30a303553d5a4c6c7301f378/0d8f6/docs/tutorials/kubernetes-basics/public/images/module_05_scaling1.svg)

**应用伸缩之后**

![after scale](https://d33wubrfki0l68.cloudfront.net/30f75140a581110443397192d70a4cdb37df7bfc/b5f56/docs/tutorials/kubernetes-basics/public/images/module_05_scaling2.svg)



> *Scaling* is accomplished by changing the number of replicas in a Deployment.

在创建 Deploment 的时候只需要修改 `replicas` 参数的值就可以实现应用伸缩。伸缩/扩展 Deployment 能确保新的 Pod 在具有可用资源的 Node 上被创建，并且能减少 Pod 的数量到理想状态。

> K8s 可以实现应用的自动伸缩扩展，同时也能实现将 Pod 的数量减少到 0。

运行应用的多个实例需要一个方法将网络请求分发给它们，很巧，Service 中的 LoadBalancer 暴露模式正好能完成。Service 将使用端点持续监控正在运行的 Pod，以确保流量仅发送到可用的 Pod。

**实现应用伸缩/扩展**

1、查看已经创建的 ReplicaSet 

```bash
kubectl get rs
```

输出类似下面的内容：

```
NAME                             DESIRED   CURRENT   READY   AGE
kubernetes-bootcamp-855d5cc575   1         1         1       5h28m
```

我们需要关注的字段有 2 个：

* *DESIRED*，显示当前应用期望的副本数，可以在创建 Deployment 的时候指定。
* *CURRENT*，表示当前有多少副本数正在运行。

 2、扩展副本数，使用 `kubectl scale` 命令

```
# 将期望副本数增加到 4 个
kubectl scale deployments/kubernetes-bootcamp --replicas=4
```

再次通过下面的命令查看，发现应用实例已经变成了 4 个。

```bash
kubectl get deployments
```

通过下面的命令查看当前 Pod 数量：

```
kubectl get pods -o wide
```

输出内容如下：

```
NAME                                   READY   STATUS    RESTARTS      AGE   IP           NODE       NOMINATED NODE   READINESS GATES
kubernetes-bootcamp-855d5cc575-gr922   1/1     Running   1 (15h ago)   15h   10.244.0.9   minikube   <none>           <none>
kubernetes-bootcamp-855d5cc575-ngsp2   1/1     Running   1 (15h ago)   15h   10.244.0.5   minikube   <none>           <none>
kubernetes-bootcamp-855d5cc575-nzsbl   1/1     Running   1 (15h ago)   15h   10.244.0.2   minikube   <none>           <none>
kubernetes-bootcamp-855d5cc575-w7xxs   1/1     Running   3 (15h ago)   21h   10.244.0.4   minikube   <none>           <none>
```

可以看到，每个 Pod 的 IP 地址都是不同的。

3、创建针对多个实例的 service，命令还是和之前的一样

```bash
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
```

4、使用 `minikube service <service-name>` 将 minikube 网络暴露出来

```bash
🎉  Opening service default/kubernetes-bootcamp in default browser...
👉  http://127.0.0.1:44013
❗  Because you are using a Docker driver on linux, the terminal needs to be open to run it.
```

6、打开另一个终端窗口，`curl http://127.0.0.1:44013`，请求几次就可以看到，K8s 会将请求以 LoadBalance 的形式分发到各个可用的 Pod 上。

```bash
> curl http://127.0.0.1:44013
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-855d5cc575-ngsp2 | v=1
> curl http://127.0.0.1:44013
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-855d5cc575-gr922 | v=1
> curl http://127.0.0.1:44013
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-855d5cc575-w7xxs | v=1
```



### 实例缩减

除了扩张应用实例，K8s 同样支持缩减应用实例，命令和扩张一致，只是参数 `replicas` 的值不同

```bash
kubectl scale deployments/kubernetes-bootcamp --replicas=2
```



## 应用滚动更新

更新之前

![rolling-update-1](https://d33wubrfki0l68.cloudfront.net/30f75140a581110443397192d70a4cdb37df7bfc/fa906/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates1.svg)

更新第一个实例

![rolling-update-2](https://d33wubrfki0l68.cloudfront.net/678bcc3281bfcc588e87c73ffdc73c7a8380aca9/703a2/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates2.svg)

更新第二个实例

![rolling-update-3](https://d33wubrfki0l68.cloudfront.net/9b57c000ea41aca21842da9e1d596cf22f1b9561/91786/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates3.svg)

更新第三、四个实例

![rolling-update-4](https://d33wubrfki0l68.cloudfront.net/6d8bc1ebb4dc67051242bc828d3ae849dbeedb93/fbfa8/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates4.svg)

K8s 中的滚动更新通过 Deployments 实现应用实例在不中断、不停机情况下更新，新的 Pod 会逐步调度到有可用的资源 Node 节点上。

K8s 的滚动更新支持以下功能：

* 应用升级；
* 版本回退；
* 不停机实现持续集成和分发。

<br/>

### 应用升级

1、首先获取一个新版本的镜像

```bash
# kubectl set image 更新一个或多个 Pod 镜像
# deployments/kubernetes-bootcamp 指定要更新的 Deployment
# kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2 指定新的镜像
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
```

2、查看 Pod 变更状态

```bash
> kubectl get pods
NAME                                   READY   STATUS              RESTARTS      AGE
kubernetes-bootcamp-69b6f9fbb9-jzdkl   0/1     ContainerCreating   0             1s
kubernetes-bootcamp-69b6f9fbb9-n8w8x   0/1     ContainerCreating   0             8s
kubernetes-bootcamp-69b6f9fbb9-xx9s6   1/1     Running             0             8s
kubernetes-bootcamp-855d5cc575-gr922   1/1     Running             1 (15h ago)   15h
kubernetes-bootcamp-855d5cc575-ngsp2   1/1     Terminating         1 (15h ago)   15h
kubernetes-bootcamp-855d5cc575-nzsbl   1/1     Terminating         1 (15h ago)   15h
kubernetes-bootcamp-855d5cc575-w7xxs   1/1     Running             3 (15h ago)   21h
```

3、等到 Pod 状态都重新变成 Running 时继续将网络暴露，发送请求验证更新

```bash
> curl http://127.0.0.1:37417
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-69b6f9fbb9-wwkvx | v=2
> curl http://127.0.0.1:37417
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-69b6f9fbb9-n8w8x | v=2
> curl http://127.0.0.1:37417
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-69b6f9fbb9-xx9s6 | v=2
```

从结果可以看到，应用负载均衡状态正常，版本更新状态正常，从 v1 升级到了 v2。

4、还可以使用 `kubectl rollout status deployments/<deployment-name>` 命令检查更新状态

```bash
> kubectl rollout status deployments/kubernetes-bootcamp
deployment "kubernetes-bootcamp" successfully rolled out
```

输出类似的结果表示应用滚动更新成功。

5、此外，还可以通过检查 Pod 镜像查看滚动更新结果

```bash
$ kubectl describe pods
Name:             kubernetes-bootcamp-69b6f9fbb9-jzdkl
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Fri, 21 Jul 2023 09:30:38 +0800
Labels:           app=kubernetes-bootcamp
                  pod-template-hash=69b6f9fbb9
Annotations:      <none>
Status:           Running
IP:               10.244.0.12
IPs:
  IP:           10.244.0.12
Controlled By:  ReplicaSet/kubernetes-bootcamp-69b6f9fbb9
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://e63f22e3bb9082ff462859f4c37ce1898e5e38bc1c0960f442ad8af033195ecf
    Image:          jocatalin/kubernetes-bootcamp:v2
    Image ID:       docker-pullable://jocatalin/kubernetes-bootcamp@sha256:fb1a3ced00cecfc1f83f18ab5cd14199e30adc1b49aa4244f5d65ad3f5feb2a5
```

从输出结果可以看到，Image 已经变成了新指定的镜像。



### 版本回退

该命令会撤销更新操作，默认回退到上一个已知版本。更新是有版本控制的，可以恢复到任何以前已知的部署状态。

1、可以使用 `kubectl rollout undo` 命令来进行版本回退，默认回退到上一个版本

```bash
kubectl rollout undo deployments/kubernetes-bootcamp
```

2、查询历史发布的 Deployment

```bash
kubectl rollout history deployments/<deployment-name>
```

输出内容大概如下：

```bash
deployment.apps/node-hello
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```

3、回退到指定版本

```bash
kubectl rollout undo deployments/<roll-back-name> --to-revision=<revision-number>
```



## 自定义服务部署

1、编写一个 Node.js 应用，命名为 server.js

```js
const http = require('http');

const handleRequest = function(request, response) {
  console.log('Received request for URL: ' + request.url);
  response.writeHead(200);
  response.end('Hello World! V1');
};
const www = http.createServer(handleRequest);
www.listen(8080);
```

2、编写 Dockerfile

```dockerfile
FROM node:latest
EXPOSE 8080
COPY server.js .
CMD node server.js
```

3、构建镜像，有两种方式：

3.1、使用 `minikube build`，进入 Dockerfile 所在的目录

```bash
minikube image build -t <image-name>:<build-version> .
```

3.2、使用与 Minikube VM 相同的 Docker 主机构建镜像

```bash
eval $(minikube docker-env)
docker build -t <image-name>:<build-version> .

# 退出 VM 主机
eval $(minikube docker-env -u)
```

4、查看镜像构建结果

```bash
minikube image ls
```

输出结果大概如下：

```bash
$ minikube image ls
docker.io/library/node-hello:v1
docker.io/library/hello-node:v11
```

说明镜像构建成功

5、创建 Deployment

```bash
kubectl create deployment --image=<image-name>:<version>
```

6、修改代码，发布 V2

7、重新构建镜像，使用 `kubectl set image deployments` 滚动更新应用

8、修改代码，发布 V3

9、重新构建，滚动更新应用

10、从 V3 回滚 V1

```bash
kubectl rollout undo deployments/ndoe-hello --to-revision=1
```



## K8s 中的对象

K8s 对象是 K8s 系统中的持久实体，K8s 使用这些实体来表示集群的状态。可以使用对象来描述：

* 应用如何运行，在哪些节点上运行；
* 应用可用资源；
* 应用运行策略、重启策略、升级和容错策略。

K8s 对象的表现行为是 *record of intent* 的，一旦创建了对象，K8s 系统就会确保对象存在。通过创建对象，可以告诉 K8s 系统你希望集群的工作负载是什么样的。



## Namespace

当存在大量不同类型的应用时，可以使用 namespace 来区分；还能隔离资源的使用。在 K8s 中，相同 namespace 下的应用具有相同的资源访问控制策略。



1、查看当前的 namespace

```bash
kubectl get namespace
```

输出内容大概如下：

```bash
NAME                   STATUS   AGE
default                Active   29h
kube-node-lease        Active   29h
kube-public            Active   29h
kube-system            Active   29h
kubernetes-dashboard   Active   29h
```

可以看到 K8s 默认存在多个 namesapce，我们创建的应用若未指定 namespace，那就会被分配到 default 这个命名空间中。

2、创建 namespace

2.1、命令行创建

```
kubectl create namespace new-namespace
```

2.2、通过文件创建

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: new-namespace
```

3、删除 namespace

```
kubectl delete namespaces new-namespace
```

> 注意：
>
> * 删除一个 namespace 会自动删除该 namespace 下的所有资源。
> * default 和 kube-system 命名空间不可删除。



### 配置 Pod 限额

1、创建 namespace

```bash
kubectl create namespace quota-pod
```





### 配置 CPU 限额

### 配置内存限额



## 微服务配置外部化



## Ingress



## Rancher

K8s 的配置、使用、集群管理方面基本上都是基于 yml 文件，并且字段对于开发人员来说比较难以理解。因此可以使用 Rancher 来管理 K8s 集群，进行项目部署等工作。

> Rancher 和 K8s 有什么区别？参考：https://www.zhihu.com/question/309076492。
>
> rancher 和 K8s都是用来作为容器的调度与编排系统。但是 rancher 不仅能够管理应用容器，更重要的一点是能够管理 K8s 集群。Rancher 2.x 底层基于 K8s 调度引擎，通过 Rancher 的封装，用户可以在不熟悉 K8s 概念的情况下轻松的通过 rancher 来部署容器到 K8s 集群当中。
>
> 为实现上述的功能，Rancher 自身提供了一套完整的用于管理 K8s 的组件，包括Rancher API Server, Cluster Controller, Cluster Agent, Node Agent 等等。组件相互协作使得 Rancher 能够掌控每个 K8s 集群，从而将多集群的管理和使用整合在统一的 Rancher 平台中。Rancher 增强了一些 K8s 的功能，并提供了面向用户友好的使用方式。



## kubesphere

> https://www.kubesphere.io/zh/
>
> https://zhuanlan.zhihu.com/p/467174069



## K3S/K8s/K9S

> https://juejin.cn/post/6955368911705473060



## 参考

### 概念

* https://kubernetes.io/docs
* http://docs.kubernetes.org.cn/
* https://zhuanlan.zhihu.com/p/345798544
* https://zhuanlan.zhihu.com/p/53260098
* https://zhuanlan.zhihu.com/p/39937913
* https://blog.csdn.net/lly337/article/details/110917930

### K8s 落地方式

* https://zhuanlan.zhihu.com/p/82666719

### 上手

* https://zhuanlan.zhihu.com/p/39937913