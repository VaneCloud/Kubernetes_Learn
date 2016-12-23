# Kubernetes Learn(术语、概念介绍)

`Kubernetes` 中有许多概念, `Node`、`Pod`、`Replication Controller`、`Service` 等概念都可以看做一种资源对象, 可以通过 `Kubernetes API` 或者 `kubectl` 进行操作, 并保存到 `etcd` 中.


## Node

`Node` 节点是相对于 `Master` 而言的的工作主机, `Node` 可以是物理机也可以是虚拟机, 每个 `Node` 上都运行 `kubelet` 服务来启动和管理 `Pod`, 并且能够被 `Master` 管理, `Node` 上运行着 `kubelet`、`docker daemon`、`kube-proxy`。

`Node` 包含如下信息

* `Node` 地址: 主机的 `IP` 地址或者 `Node ID`
* `Node` 运行状态: 包括 `Pending`、`Running`、`Terminated` 三种状态
* `Node` `Condition`: 描述 `Running` 状态 `Node` 的运行条件, 目前只用一种指令, `Ready` 表示 `Node` 可以接受从 `Master` 发送过来的指令.
* `Node` 系统容量: 描述 `Node` 可用资源, 可用 `CPU`、`Memory`、`Max Pods` 等.
* `System Info`: 其他信息, `Kernel Version`、`Kubernetes Api Version`、`OS Image` 等.

还可以通过 `kubectl describe node 127.0.0.1` 查看特定 `Node` 的资源.

```
$ kubectl describe node 127.0.0.1
Name:                   127.0.0.1
Labels:                 kubernetes.io/hostname=127.0.0.1
CreationTimestamp:      Thu, 22 Dec 2016 06:01:16 +0000
Phase:
Conditions:
  Type          Status  LastHeartbeatTime                       LastTransitionTime                      Reason                          Message
  ----          ------  -----------------                       ------------------                      ------                          -------
  OutOfDisk     False   Thu, 22 Dec 2016 08:21:47 +0000         Thu, 22 Dec 2016 06:01:16 +0000         KubeletHasSufficientDisk        kubelet has sufficient disk space available
  Ready         True    Thu, 22 Dec 2016 08:21:47 +0000         Thu, 22 Dec 2016 06:10:11 +0000         KubeletReady                    kubelet is posting ready status
Addresses:      127.0.0.1,127.0.0.1
Capacity:
 memory:        3882448Ki
 pods:          110
 cpu:           2
System Info:
 Machine ID:                    9487861ec767b42060fd9cff1c2b8f16
 System UUID:                   4E433470-B3C7-49B8-92CE-63FFF112921F
 Boot ID:                       b97886c8-3fbb-41fc-91a9-e845e8f73ea3
 Kernel Version:                3.10.0-327.13.1.el7.x86_64
 OS Image:                      CentOS Linux 7 (Core)
 Container Runtime Version:     docker://1.10.3
 Kubelet Version:               v1.2.0
 Kube-Proxy Version:            v1.2.0
ExternalID:                     127.0.0.1
Non-terminated Pods:            (2 in total)
  Namespace                     Name                                    CPU Requests    CPU Limits      Memory Requests Memory Limits
  ---------                     ----                                    ------------    ----------      --------------- -------------
  default                       my-nginx-3800858182-5pyt5               0 (0%)          0 (0%)          0 (0%)          0 (0%)
  default                       my-nginx-3800858182-wcedc               0 (0%)          0 (0%)          0 (0%)          0 (0%)
Allocated resources:
  (Total limits may be over 100%, i.e., overcommitted. More info: http://releases.k8s.io/HEAD/docs/user-guide/compute-resources.md)
  CPU Requests  CPU Limits      Memory Requests Memory Limits
  ------------  ----------      --------------- -------------
  0 (0%)        0 (0%)          0 (0%)          0 (0%)
Events:
  FirstSeen     LastSeen        Count   From                    SubobjectPath   Type            Reason                  Message
  ---------     --------        -----   ----                    -------------   --------        ------                  -------
  50m           50m             2       {kubelet 127.0.0.1}                     Warning         MissingClusterDNS       kubelet does not have ClusterDNS IP configured and cannot create Pod using "ClusterFirst" policy. pod: "my-nginx-3800858182-itzyj_default(89a91089-c818-11e6-bc92-fa163efae568)". Falling back to DNSDefault policy.
  47m           33m             4       {kubelet 127.0.0.1}                     Warning         MissingClusterDNS       kubelet does not have ClusterDNS IP configured and cannot create Pod using "ClusterFirst" policy. pod: "my-nginx-3800858182-5pyt5_default(ffbab391-c818-11e6-bc92-fa163efae568)". Falling back to DNSDefault policy.
  47m           33m             4       {kubelet 127.0.0.1}                     Warning         MissingClusterDNS       kubelet does not have ClusterDNS IP configured and cannot create Pod using "ClusterFirst" policy. pod: "my-nginx-3800858182-wcedc_default(ffbabb4e-c818-11e6-bc92-fa163efae568)". Falling back to DNSDefault policy.
```

## Pod

`Pod` 是 `Kubernetes` 最小操作单元, 一个 `Pod` 包含了一个或多个紧密相连的容器, 一个 `Pod` 中多个容器的应用通常是紧耦合的。

### 为什么 `Container` 不是 `Kubernetes` 最小单元

因为 `Docker` 中 通常一个容器通过 `Link` 的方式访问另一个容器, 多个容器 `Link` 是非常繁琐的, 所以 `Kubernetes` 通过 `Pod` 的概念将多个容器组合在一个 `Pod` 中, 这样多个容器通过 `Localhost` 就能够互相通信了。

`Pod` 中的容器共享如下资源.

* `PID Namespace` : `Pod` 中的多个容器可以看到彼此的进程 `ID`
* `Net Namespace` : `Pod` 中多个容器共享一个 `IP` 地址.
* `IPC Namespace` : `Pod` 中的多个容器可以直接通过 `System V`、`POSIX` 消息队列进行通信.
* `UTS Namespace` : `Pod` 中的多个容器共享一个主机名.
* `Volume`: `Pod` 中的多个容器都可以访问 `Pod` 级别定义的存储卷.

### `Pod` 的生命周期

`Pod` 的生命周期通过 `Replication Controller` 来管理的。`Pod` 的生命周期包括, 通过模板定义 --> 分配到 `Node`运行 --> `Pod` 所有容器运行结束后, `Pod` 也结束.

`Pod` 有以下几种状态.

* `Pending`: `Pod` 定义正确, 但是还未调度到 `Node` 上或者 `Node` 正在下载镜像.
* `Running`: `Pod` 已经成功运行起来
* `Succeeded`: `Pod` 中所有容器都已经运行结束, 且不会被重启
* `Failed`: `Pod` 中所有容器都结束了, 但至少一个容器是失败结束的.


## Label

`Label` 是 `Kubernetes` 的核心的概念, `Label` 以 `Key/Value` 的形式附加到各类对象中, `Pod`、`Service`、`Replication Controller`、`Node` 等。
`Label` 定义了这些对象的可识别属性, 可以用于对象的管理和选择, `Label` 可以在创建时指定, 也可以在对象创建后通过 `API` 进行管理.

为对象定义好 `Label` 之后, 其他对象可以通过 `Label Selector` 来定义其作用的对象了.

**Example: Label Selector**

```
"labels": {
  "key1": "value1",
  "key2": "value2"
}
```

`Label Selector` 可以基于等式和集合来定义.

```
# Equality-based

name=my-nginx

# Set-based

name in (my-nginx, redis-slave)

# 也可以组合

name in (my-nginx, redis-slave), release!=production
```

`Replication Controller` 通过 `Label Selector` 来定义管理的 `Pods`, 同样, `Service` 也通过 `Label Selector` 来选择后端 `Pods` 加入其 `Load Balance` 列表.


## Replication Controller

`Replication Controller` 是 `Kubernetes` 的核心概念, 用于定义 `Pod` 副本数量。`Master` 节点的 `Controller Manager` 进程通过 `Replication Controller` 的定义完成 `Pod` 的创建, 监控, 启停.

根据 `Replication Controller` 的定义, `Kubernetes` 能够确保任意时候都能运行着用户指定的 `Pod` 副本数量, 如果运行的 `Pod` 数高于预期, 会停止一些, 反之会再启动一些.

还可以通过 `kubectl scale` 命令对 `Pod` 数量进行扩展.

```
$ kubectl scale rc redis-slave --replicas=3
```

删除 `Replication Controller` 并不会导致其所管控的 `Pod` 被销毁, 如果期望删除所有的 `Pod`, 我们可以调整 `replicas` 为 `0` 然后更新 `Replication Controller` 既可, 不过 `Kubectl` 工具提供了 `stop` 和 `delete` 命令一次性删除 `Replcaion Controller` 下的所有 `Pod`.


## Service

`Service` 是 `Kubernetes` 的核心概念, 在 `Kubernetes` 中, 每一个 `Pod` 都有一个独立的 `IP` 地址, 如果底层容器技术是 `Docker` 那么这个地址是默认通过 `Docker` 的 `Bridge` 分配的, 但在 `Kubernetes` 中 `Pod` 是最小单元, 生命周期最短, 可能会被 `Replication Controller` 随时销毁, 就算重新启动, `IP` 地址也可能发生变化, 那么用户该如何访问呢? 这时候就需要 `Service`。

`Service` 是一组提供相同服务的 `Pod` 对外的负载均衡器, 到底是哪些 `Pod` 取决于 `Label Selector` 的定义. 用户或者其他 `Pod` 无需关心 `Service` 后端到底有多少个 `Pod` 以及 `Pod` 的状态, 只需要访问 `Service` 即可.

在 `Pod` 正常启动后, 系统会根据 `Service` 的定义创建出与 `Pod` 对应的 `Endpoint` 对象, 建立起 `Service` 与后端 `Pod` 的对应关系. 随着 `Pod` 的创建、销毁, `Endpoint` 对象也会被更新. `Endpoint` 主要由 `Pod` 的 `IP` 和 容器需要监听的端口组成, 可以通过 `kubectl get endpoints` 查看所有的 `Endpoint`

```
$ kubectl get endpoints
NAME         ENDPOINTS                                   AGE
kubernetes   172.24.1.164:6443                           20h
my-nginx     172.17.0.2:80,172.17.0.3:80,172.17.0.4:80   19h
```

### Cluster IP

`Service` 在创建时会分配一个 `Cluster IP`,  `Service` 被销毁之前, `Cluster IP` 都不会变化, `Cluster IP Range` 在 `apiserver` 的配置文件中有定义.

```
$ cat /etc/kubernetes/apiserver |grep cluster-ip
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
```

但是这个地址在外部是无法访问的, 目前想要外部访问 `Service` 有两种方式, 通过 `NodePort` 和 `LoadBalancer`

* `NodePort`: 指定这个选项之后, 系统会在每一个 `Node` 上开启一个相同的 `Port`, 能够访问 `Kubernetes Node` 的用户都能够访问 `Service`
* `LoadBalancer`: 如果云提供商支持 `LoadBalancer`, 则可以通过这个选项指定云提供商分配的 `LoadBalancer` 的外部地址, 从而实现外部用户的访问.


## Next

其实还有很多概念没有介绍, 例如: `namespace`、`volume`、`deployment`、`Daemon Set` 等, 在后面的文章中会陆续引出.
