# Kubernetes Learn (基础介绍和安装)


`Kubernetes` 是 `Google` 开源的一套分布式容器管理系统, 诞生至今才两年多, 近日发布了 1.5 版本, 虽然 `Kubernetes` 是一个非常年轻的开源项目, 但是在业内影响巨大, `HPC`、`IBM`、 `RedHat`、 `CoreOS`、 `Mirantis` 业界巨头纷纷加入.


`Kubernetes` 是一个 一切以 `Service` 为中心的系统, 一切围绕服务运转, `Kubernetes` 可以构建在物理机上, 也可以构建在 私有云, 公有云之上. `Kubernetes` 的最大亮点是自动化, 一个 `Kubernetes` 的 `Service` 可以自我扩展, 自动诊断, 自动扩展.


## Kubernetes 和 Docker

通常我们认为 `Kubernetes` 只是 `Docker` 的上层架构, 其实并非完全正确. `Kubernetes` 以 `Docker` 为基础打造一个云计算时代的分布式系统架构.

目前 `Kubernetes` 支持两种开源容器技术, 一个是 `Docker`, 另一个是 `CoreOS` 推出的 `Rocket`, 这是当年 `CoreOS` 和 `Docker` 分家所导致的, 有兴趣可以 `Google` 相关事件.


## 为什么要使用 Kubernetes

很多时候我们可能需要设计一套分布式系统, 但是可能需要很多运维专家才能够完成, 使用 `Kubernetes` 之后, 只需要将系统中的各组件拆分成为多个 `Service`, 就能够实现自动扩展, 自动修复等高级功能.

其次使用 `Kubernetes` 就是在全面拥抱微服务, 微服务就是将一个巨大的单体应用拆分为很多个互相连接的微服务, 一个微服务后端可能有多个实例副本在支撑, 副本数会根据当前系统负载而调整, 每个组件项目连接都不知道也不需要知道一个服务后端有多少个副本, 这里是通过内嵌的负载均衡器实现的. 从而使每个微服务都可以独立开发、升级、扩展, 从而使整个系统更加稳定、快速迭代。

甚至于我们可以在负载过高时将应用无缝迁移到公有云以提升系统的吞吐量, 足以证明 `Kubernetes` 提供了超强的横向扩展能力。


## Kubernetes all in one 安装

在 `CentOS 7` 中, `Kubernetes` 的安装非常的简单, 直接通过 `yum` 即可安装, `kubernetes` 只依赖 `docker` 和 `etcd` 这两个外部组件

```shellscript
$ setenforce 0 && systemctl disable firewalld

$ yum install kubernetes etcd

$ vim /etc/kubernetes/apiserver
  删除 --admission_control 参数中的 ServiceAccount

$ systemctl start docker etcd kube-apiserver kube-controller-manager kube-proxy kubelet

$ kubectl get nodes   #查看当前所有节点
NAME        STATUS    AGE
127.0.0.1   Ready     38s
```


## Nginx 部署

```
$ kubectl run my-nginx --image=nginx --replicas=2 --port=80 #部署一个 nginx
deployment "my-nginx" created

$ kubectl expose deployment my-nginx --target-port=80 --type=NodePort #创建与之对应的service
service "my-nginx" exposed
```

```
$ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
my-nginx-3800858182-5pyt5   1/1       Running   0          32s
my-nginx-3800858182-wcedc   1/1       Running   0          32s

$ kubectl describe service/my-nginx #查看分配到了哪个端口
Name:                   my-nginx
Namespace:              default
Labels:                 run=my-nginx
Selector:               run=my-nginx
Type:                   NodePort
IP:                     10.254.42.51
Port:                   <unset> 80/TCP
NodePort:               <unset> 30849/TCP
Endpoints:              172.17.0.2:80,172.17.0.3:80
Session Affinity:       None
No events.
```

### 访问测试

```
$ curl 192.168.20.179:30849 #访问
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
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### 容错测试

```
$ docker ps #可以看到我们运行了两个 nginx 的 container
CONTAINER ID        IMAGE                                                        COMMAND                  CREATED             STATUS              PORTS               NAMES
3bd031919294        nginx                                                        "nginx -g 'daemon off"   6 minutes ago       Up 6 minutes                            k8s_my-nginx.631bfda2_my-nginx-3800858182-5pyt5_default_ffbab391-c818-11e6-bc92-fa163efae568_c2c892c5
45f2a5c38d0d        nginx                                                        "nginx -g 'daemon off"   6 minutes ago       Up 6 minutes                            k8s_my-nginx.631bfda2_my-nginx-3800858182-wcedc_default_ffbabb4e-c818-11e6-bc92-fa163efae568_2cb7664c
476b520c14c0        registry.access.redhat.com/rhel7/pod-infrastructure:latest   "/pod"                   13 minutes ago      Up 13 minutes                           k8s_POD.c36b0a77_my-nginx-3800858182-wcedc_default_ffbabb4e-c818-11e6-bc92-fa163efae568_3aad0673
9f68e6cef738        registry.access.redhat.com/rhel7/pod-infrastructure:latest   "/pod"                   13 minutes ago      Up 13 minutes                           k8s_POD.c36b0a77_my-nginx-3800858182-5pyt5_default_ffbab391-c818-11e6-bc92-fa163efae568_307f6584

$ docker rm -fv 3bd031919294 45f2a5c38d0d #删除容器

$ kubectl get pods  #可以看到pod自动恢复了
NAME                        READY     STATUS              RESTARTS   AGE
my-nginx-3800858182-5pyt5   0/1       ContainerCreating   0          14m
my-nginx-3800858182-wcedc   0/1       ContainerCreating   0          14m

$ kubectl get pods  #过一会又runnig了
NAME                        READY     STATUS    RESTARTS   AGE
my-nginx-3800858182-5pyt5   1/1       Running   0          14m
my-nginx-3800858182-wcedc   1/1       Running   0          14m

$ curl 192.168.20.179:30849 #还是能够访问
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
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Next

下一篇介绍 `Kubernetes` 概念、术语的介绍.
