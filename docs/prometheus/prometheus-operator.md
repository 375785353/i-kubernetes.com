## 介绍

首先我们先来了解下 Prometheus-Operator 的架构图：
![prometheus](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200410141511.png)

上图是 Prometheus-Operator 官方提供的架构图，各组件以不同的方式运行在 Kubernetes 集群中，其中 Operator 是最核心的部分，作为一个控制器，他会去创建Prometheus、ServiceMonitor、AlertManager 以及 PrometheusRule 4个 CRD 资源对象，然后会一直监控并维持这4个资源对象的状态。

- Operator：根据自定义资源来部署和管理 Prometheus Server，同时监控这些自定义资源事件的变化来做相应的处理，是整个系统的控制中心。

- Prometheus：声明 Prometheus 资源对象期望的状态，Operator 确保这个资源对象运行时一直与定义保持一致。

- Prometheus Server：Operator 根据自定义资源 Prometheus 类型中定义的内容而部署的 Prometheus Server 集群，这些自定义资源可以看作是用来管理 Prometheus Server 集群的 StatefulSets 资源。

- ServiceMonitor：声明指定监控的服务，描述了一组被 Prometheus 监控的目标列表，就是 exporter 的抽象，用来提供 metrics 数据接口的工具。该资源通过 Labels 来选取对应的 Service Endpoint，让 Prometheus Server 通过选取的 Service 来获取 Metrics 信息。

- Service：简单的说就是 Prometheus 监控的对象。

- Alertmanager：定义 AlertManager 资源对象期望的状态，Operator 确保这个资源对象运行时一直与定义保持一致。

这样我们要在集群中监控什么数据，就变成了直接去操作 Kubernetes 集群的资源对象了，是不是方便很多了。

## 安装

我们可以使用 Helm 来快速安装 Prometheus Operator，也可以通过 https://github.com/coreos/kube-prometheus 项目来手动安装，我们这里采用手动安装的方式可以去了解更多的实现细节。

首先 clone 项目代码：

```
$ git clone https://github.com/coreos/kube-prometheus.git
$ cd manifests
```

进入到 manifests 目录下面，首先我们需要安装 setup 目录下面的 CRD 和 Operator 资源对象：

```
$ kubectl apply -f setup/
$ kubectl get pods -n monitoring
NAME                                   READY   STATUS    RESTARTS   AGE
prometheus-operator-6694b5cb64-z64ns   2/2     Running   0          114s
$ kubectl get crd |grep coreos
alertmanagers.monitoring.coreos.com              2020-04-10T07:00:12Z
podmonitors.monitoring.coreos.com                2020-04-10T07:00:13Z
prometheuses.monitoring.coreos.com               2020-04-10T07:00:14Z
prometheusrules.monitoring.coreos.com            2020-04-10T07:00:15Z
servicemonitors.monitoring.coreos.com            2020-04-10T07:00:16Z
thanosrulers.monitoring.coreos.com               2020-04-10T07:00:17Z
```

这会创建一个名为 monitoring 的命名空间，以及相关的 CRD 资源对象声明和 Prometheus Operator 控制器。前面章节中我们讲解过 CRD 和 Operator 的使用，当我们声明完 CRD 过后，就可以来自定义资源清单了，但是要让我们声明的自定义资源对象生效就需要安装对应的 Operator 控制器，这里我们都已经安装了，所以接下来就可以来用 CRD 创建真正的自定义资源对象了。其实在 manifests 目录下面的就是我们要去创建的 Prometheus、Alertmanager 以及各种监控对象的资源清单。

没有特殊的定制需求我们可以直接一键安装：

```
$ kubectl apply -f .
```
这会自动安装 node-exporter、kube-state-metrics、grafana、prometheus-adapter 以及 prometheus 和 alertmanager 组件，而且 prometheus 和 alertmanager 还是多副本的。

```
$ kubectl get pods -n monitoring
NAME                                   READY   STATUS              RESTARTS   AGE
alertmanager-main-0                    2/2     Running             0          10m
alertmanager-main-1                    2/2     Running             0          10m
alertmanager-main-2                    2/2     Running             0          10m
grafana-86b55cb79f-jpnmr               1/1     Running             0          9m53s
kube-state-metrics-dbb85dfd5-hl2sn     3/3     Running             0          9m49s
node-exporter-482tf                    2/2     Running             0          95s
node-exporter-9g2cv                    2/2     Running             0          9m47s
node-exporter-dxr2d                    2/2     Running             0          9m47s
node-exporter-h4f6c                    2/2     Running             0          9m47s
node-exporter-hxwqb                    2/2     Running             0          9m47s
node-exporter-lzdw2                    2/2     Running             0          9m47s
node-exporter-n2qj6                    2/2     Running             0          9m47s
prometheus-adapter-5cd5798d96-4r6lx    1/1     Running             0          9m40s
prometheus-k8s-0                       3/3     Running             0          9m27s
prometheus-k8s-1                       3/3     Running             1          9m25s
prometheus-operator-6694b5cb64-z64ns   2/2     Running             0          18m
$ kubectl get svc -n monitoring
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-main       ClusterIP   10.111.28.173    <none>        9093/TCP                     12m
alertmanager-operated   ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   12m
grafana                 ClusterIP   10.99.62.32      <none>        3000/TCP                     11m
kube-state-metrics      ClusterIP   None             <none>        8443/TCP,9443/TCP            11m
node-exporter           ClusterIP   None             <none>        9100/TCP                     11m
prometheus-adapter      ClusterIP   10.100.102.211   <none>        443/TCP                      11m
prometheus-k8s          ClusterIP   10.111.105.155   <none>        9090/TCP                     11m
prometheus-operated     ClusterIP   None             <none>        9090/TCP                     11m
prometheus-operator     ClusterIP   None             <none>        8443/TCP                     20m
```

可以看到上面针对 grafana、alertmanager 和 prometheus 都创建了一个类型为 ClusterIP 的 Service，当然如果我们想要在外网访问这两个服务的话可以通过创建对应的 Ingress 对象或者使用 NodePort 类型的 Service，我们这里为了简单，直接使用 NodePort 类型的服务即可，编辑 grafana、alertmanager-main 和 prometheus-k8s 这3个 Service，将服务类型更改为 NodePort:

```
# 将 type: ClusterIP 更改为 type: NodePort
$ kubectl edit svc grafana -n monitoring  
$ kubectl edit svc alertmanager-main -n monitoring
$ kubectl edit svc prometheus-k8s -n monitoring
$ kubectl get svc -n monitoring
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-main       NodePort    10.111.28.173    <none>        9093:30733/TCP               18m
grafana                 NodePort    10.99.62.32      <none>        3000:32150/TCP               17m
prometheus-k8s          NodePort    10.111.105.155   <none>        9090:30206/TCP               17m
```
更改完成后，我们就可以通过上面的 NodePort 去访问对应的服务了，比如查看 prometheus 的服务发现页面：

![img](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200410153402.png)

可以看到已经监控上了很多指标数据了，上面我们可以看到 Prometheus 是两个副本，我们这里通过 Service 去访问，按正常来说请求是会去轮询访问后端的两个 Prometheus 实例的，但实际上我们这里访问的时候始终是路由到后端的一个实例上去，因为这里的 Service 在创建的时候添加了 sessionAffinity: ClientIP 这样的属性，会根据 ClientIP 来做 session 亲和性，所以我们不用担心请求会到不同的副本上去：

```
apiVersion: v1
kind: Service
metadata:
  labels:
    prometheus: k8s
  name: prometheus-k8s
  namespace: monitoring
spec:
  ports:
  - name: web
    port: 9090
    targetPort: web
  selector:
    app: prometheus
    prometheus: k8s
  sessionAffinity: ClientIP
```

## 配置

我们可以看到上面的监控指标大部分的配置都是正常的，只有两三个没有管理到对应的监控目标，比如 kube-controller-manager 和 kube-scheduler 这两个系统组件。

![img](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/20200410154135.png)

这其实就和 ServiceMonitor 的定义有关系了，我们先来查看下 `kube-scheduler `组件对应的 `ServiceMonitor` 资源的定义，`manifests/prometheus-serviceMonitorKubeScheduler.yaml`：

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: kube-scheduler
  name: kube-scheduler
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s  # 每30s获取一次信息
    port: http-metrics  # 对应 service 的端口名
  jobLabel: k8s-app
  namespaceSelector:  # 表示去匹配某一命名空间中的service，如果想从所有的namespace中匹配用any: true
    matchNames:
    - kube-system
  selector:  # 匹配的 Service 的 labels，如果使用 mathLabels，则下面的所有标签都匹配时才会匹配该 service，如果使用 matchExpressions，则至少匹配一个标签的 service 都会被选择
    matchLabels:
      app.kubernetes.io/name: kube-scheduler
```

上面是一个典型的 ServiceMonitor 资源对象的声明方式，上面我们通过 selector.matchLabels 在 kube-system 这个命名空间下面匹配具有 `app.kubernetes.io/name: kube-scheduler` 这样的 Service，但是我们系统中根本就没有对应的 Service：

```
$ kubectl get svc -n kube-system -l app.kubernetes.io/name=kube-scheduler
No resources found in kube-system namespace.
```
所以我们需要去创建一个对应的 Service 对象，才能核 `ServiceMonitor` 进行关联：(prometheus-kubeSchedulerService.yaml)
```
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-scheduler
  labels:
    app.kubernetes.io/name: kube-scheduler
spec:
  selector:
    component: kube-scheduler
  ports:
  - name: http-metrics
    port: 10251
    targetPort: 10251
```
