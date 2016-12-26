# Kubernetes Learn (管理资源)

## 组织多个资源

许多应用程序需要创建多个资源, 例如需要创建一个 `Deployment` 来管理 `rc` 和 `pod`, 创建 `Service` 来对 `Endpoints` 负载均衡以及接受外部请求, 之前我们都是通过两个 `yaml` 文件来做, 其实只需要在不同资源之间加上 `---` 即可组合在一起.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-svc
  labels:
    app: nginx
spec:
  type: NodePort
  ports:
  - port: 80
  selector:
    app: nginx
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```


```shellscript
$ kubectl create -f kube-nginx-app.yaml
You have exposed your service on an external port on all nodes in your
cluster.  If you want to expose this service to the external internet, you may
need to set up firewall rules for the service port(s) (tcp:30263) to serve traffic.

See http://releases.k8s.io/release-1.2/docs/user-guide/services-firewalls.md for more details.
service "my-nginx-svc" created
deployment "my-nginx" created

$ curl 192.168.20.179:30263  &> /dev/null && echo 'Connectd Nginx Success'
Connectd Nginx Success
```

也可以以向下面这样指定多个文件进行部署.

```shellscript
$ kubectl create -f kube-nginx.yaml -f kube-nginx-service.yaml
```

还可以指定一个目录, `kubectl` 会读取后缀为 `.json`、`.yaml`、`yml` 的文件.

```shellscript
$ kubectl create -f docs/user-guide/nginx/
```

## Kubectl 批量操作

可以使用 `kubectl` 命令对多个资源进行批量操作, 像下面这样

```shellscript
$ kubectl get service,deployment -l app=nginx #列出 service, deployment 下所有标签 app=nginx的资源
NAME           CLUSTER-IP      EXTERNAL-IP   PORT(S)      AGE
my-nginx-svc   10.254.131.14   nodes         80/TCP       6m
NAME           DESIRED         CURRENT       UP-TO-DATE   AVAILABLE   AGE
my-nginx       3               3             3            3           6m

$ kubectl delete service,deployment -l app=nginx #删除 service, deployment 下所有标签 app=nginx的资源.
service "my-nginx-svc" deleted
deployment "my-nginx" deleted
```

## Labels 操作

查看 `pods` 对应 `Label` 的值

```shellscript
$ kubectl get pods -Lapp
NAME                        READY     STATUS    RESTARTS   AGE       APP
my-nginx-2035384211-147bn   1/1       Running   0          3m        nginx
my-nginx-2035384211-jthg5   1/1       Running   0          3m        nginx
my-nginx-2035384211-loisa   1/1       Running   0          3m        nginx
```

更新 `Labels`

```shellscript
$ kubectl label pod my-nginx-2035384211-147bn version=latest
pod "my-nginx-2035384211-147bn" labeled

$ kubectl get pods -Lversion
NAME                        READY     STATUS    RESTARTS   AGE       VERSION
my-nginx-2035384211-147bn   1/1       Running   0          4m        latest
my-nginx-2035384211-jthg5   1/1       Running   0          4m        <none>
my-nginx-2035384211-loisa   1/1       Running   0          4m        <none>
```

正好验证一下 `Service` 的 `Label Selector`与后端 `Pod` 的关联性.

```shellscript
$ kubectl get endpoints
NAME           ENDPOINTS                                   AGE
kubernetes     172.24.1.164:6443                           3d
my-nginx-svc   172.17.0.2:80,172.17.0.3:80,172.17.0.4:80   6m #目前有三个endpoints

$ kubectl label  pod my-nginx-2035384211-147bn app=nginx2 --overwrite #将其中一个pod label app对应的值改为nginx2
pod "my-nginx-2035384211-147bn" labeled

$ kubectl get endpoints
NAME           ENDPOINTS                     AGE
kubernetes     172.24.1.164:6443             3d
my-nginx-svc   172.17.0.2:80,172.17.0.3:80   6m #endpoints变为两个

$ kubectl label  pod my-nginx-2035384211-147bn app=nginx --overwrite #再改回去
pod "my-nginx-2035384211-147bn" labeled

$ kubectl get endpoints
NAME           ENDPOINTS                                   AGE
kubernetes     172.24.1.164:6443                           3d
my-nginx-svc   172.17.0.2:80,172.17.0.3:80,172.17.0.4:80   6m #endpoints又恢复成三个了
```

## 应用扩展

可以通过 `kubectl` 命令简单的实现应用 `replicas` 的扩展.

```shellscript
$ kubectl scale deployment/my-nginx --replicas=5
deployment "my-nginx" scaled

$ kubectl get pods
NAME                        READY     STATUS              RESTARTS   AGE
my-nginx-2035384211-8rtap   0/1       ContainerCreating   0          3s
my-nginx-2035384211-dugol   0/1       ContainerCreating   0          3s
my-nginx-2035384211-jthg5   1/1       Running             0          13m
my-nginx-2035384211-loisa   1/1       Running             0          13m
my-nginx-2035384211-s4pl2   0/1       ContainerCreating   0          3s
```

也可以设置自动扩展, 将有系统决定应用的 `replicas` 数量

```shellscript
$ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
my-nginx-2035384211-warn8   1/1       Running   0          1m

$ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
my-nginx-2035384211-warn8   1/1       Running   0          1m

$ kubectl autoscale deployment/my-nginx --min=2 --max=5
deployment "my-nginx" autoscaled

$ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
my-nginx-2035384211-warn8   1/1       Running   0          1m
my-nginx-2035384211-z35fl   1/1       Running   0          6s
```


## 资源升级

我们可以使用 `kubectl apply` 命令通过指定资源模板文件的方式, 对资源进行升级

```shellscript
$ kubectl apply -f kube-nginx-app.yaml
service "my-nginx-svc" configured
deployment "my-nginx" configured
```

也可以使用 `kubectl edit` 命令直接对资源进行修改

```shellscript
$ kubectl edit deployment/my-nginx
deployment "my-nginx" edited #我们修改 replicas 值为 5

$ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
my-nginx-2035384211-ooug9   1/1       Running   0          43s
my-nginx-2035384211-warn8   1/1       Running   0          6m
my-nginx-2035384211-xomiu   1/1       Running   0          2m
my-nginx-2035384211-z35fl   1/1       Running   0          5m
my-nginx-2035384211-z80mv   1/1       Running   0          43s
```
