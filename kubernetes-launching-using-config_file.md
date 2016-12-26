# Kubernetes Learn (通过配置文件创建应用)


## 定义 Deployment

我们可以通过 `kubectl run` 快速部署一个应用, 也可以使用如下的 `YAML` 文件进行部署.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
```

`Deployment` 是 `Kubernetes 1.2` 引入的概念, 它包含了对 `pod` 和 `rc` 的定义.


可以通过 `kubectl create -f` 命令部署应用.

```shellscript
$ kubectl create -f kube-nginx.yaml
deployment "my-nginx" created

$ kubectl get deployment/my-nginx
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
my-nginx   2         2         2            0           28s

$ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
my-nginx-3800858182-6viek   1/1       Running   0          1m
my-nginx-3800858182-uruz1   1/1       Running   0          1m
```


## 定义 Service

`pod` 虽然起来了, 我们需要定义 `Service` 才能被外部访问.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  selector:
    run: my-nginx #选择label run 的值为my-nginx的作为endpoints
```

创建 `Service`

```shellscript
$ kubectl create -f kube-nginx-service.yaml
You have exposed your service on an external port on all nodes in your
cluster.  If you want to expose this service to the external internet, you may
need to set up firewall rules for the service port(s) (tcp:30194) to serve traffic.

See http://releases.k8s.io/release-1.2/docs/user-guide/services-firewalls.md for more details.
service "my-nginx" created

$ kubectl get service/my-nginx
NAME       CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
my-nginx   10.254.88.207   nodes         80/TCP    12s

$ curl 192.168.20.179:30194 &> /dev/null && echo 'Nginx Connected'
Nginx Connected #LB Success
```
