# Kubernetes使用StorageClass动态生成NFS类型的PV

官方文档：https://github.com/kubernetes-incubator/external-storage/tree/master/nfs

以自己单独安装NFS系统为例

## 1、安装nfs设置共享（生产环境建议使用云服务商提供的云共享存储如：阿里云的NAS，AWS的EBS）
```
# 关闭防火墙
systemctl stop firewalld.service
systemctl disable firewalld.service

# 安装配置 nfs
yum -y install nfs-utils rpcbind

# 共享目录设置权限：
chmod 755 /data/k8s/

# 配置 nfs，nfs 的默认配置文件在 /etc/exports 文件下，在该文件中添加下面的配置信息：
vim /etc/exports
/data/k8s  *(rw,sync,no_root_squash)
```
配置说明：

- /data/k8s：是共享的数据目录
- *：表示任何人都有权限连接，当然也可以是一个网段，一个 IP，也可以是域名
- rw：读写的权限
- sync：表示文件同时写入硬盘和内存
- no_root_squash：当登录 NFS 主机使用共享目录的使用者是 root 时，其权限将被转换成为匿名使用者，通常它的 UID 与 GID，都会变成 nobody 身份

启动服务：
> 注意启动顺序，先启动 rpcbind

```
systemctl start rpcbind.service
systemctl enable rpcbind

systemctl start nfs.service
systemctl enable nfs
```
到这里我们就把 nfs server 给安装成功了，接下来我们在节点上来安装 nfs 的客户端来验证下 nfs

```
# 关闭防火墙
systemctl stop firewalld.service
systemctl disable firewalld.service

# 安装配置 nfs
yum -y install nfs-utils rpcbind
```

安装完成后，和上面的方法一样，先启动 rpc、然后启动 nfs：
```
systemctl start rpcbind.service 
systemctl enable rpcbind.service 
systemctl start nfs.service    
systemctl enable nfs.service
```
挂载数据目录 客户端启动完成后，我们在客户端来挂载下 nfs 测试下：  
首先检查下 nfs 是否有共享目录：
```
showmount -e 10.23.78.228
Export list for 10.23.78.228:
/data/k8s *
```
然后我们在客户端上新建目录：
```
mkdir -p /root/course/kubeadm/data
```

将 nfs 共享目录挂载到上面的目录：
```
mount -t nfs 10.151.30.57:/data/k8s /root/course/kubeadm/data
```
## 2、创建StorageClass

要使用 StorageClass，我们就得安装对应的自动配置程序，比如我们这里存储后端使用的是 nfs，那么我们就需要使用到一个 nfs-client 的自动配置程序，我们也叫它 Provisioner，这个程序使用我们已经配置好的 nfs 服务器，来自动创建持久卷，也就是自动帮我们创建 PV。

- 自动创建的 PV 以${namespace}-${pvcName}-${pvName}这样的命名格式创建在 NFS 服务器上的共享数据目录中
- 而当这个 PV 被回收后会以archieved-${namespace}-${pvcName}-${pvName}这样的命名格式存在 NFS 服务器上。

第一步：配置 Deployment，将里面的对应的参数替换成我们自己的 nfs 配置（nfs-client.yaml）
```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 10.23.78.228
            - name: NFS_PATH
              value: /data/k8s
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.23.78.228
            path: /data/k8s
```
第二步：将环境变量 NFS_SERVER 和 NFS_PATH 替换，当然也包括下面的 nfs 配置，我们可以看到我们这里使用了一个名为 nfs-client-provisioner 的serviceAccount，所以我们也需要创建一个 sa，然后绑定上对应的权限：（nfs-client-sa.yaml）
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```
我们这里新建的一个名为 nfs-client-provisioner 的ServiceAccount，然后绑定了一个名为 nfs-client-provisioner-runner 的ClusterRole，而该ClusterRole声明了一些权限，其中就包括对persistentvolumes的增、删、改、查等权限，所以我们可以利用该ServiceAccount来自动创建 PV。

第三步：nfs-client 的 Deployment 声明完成后，我们就可以来创建一个StorageClass对象了：（nfs-client-class.yaml）
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: course-nfs-storage
provisioner: fuseim.pri/ifs # or choose another name, must match deployment's env PROVISIONER_NAME'
```
我们声明了一个名为 course-nfs-storage 的StorageClass对象，注意下面的provisioner对应的值一定要和上面的Deployment下面的 PROVISIONER_NAME 这个环境变量的值一样。

现在我们来创建这些资源对象吧：

```
kubectl create -f nfs-client.yaml
kubectl create -f nfs-client-sa.yaml
kubectl create -f nfs-client-class.yaml
```

## 使用helm安装

```yaml
helm install course-nfs-storage --set nfs.server=10.23.78.228 --set nfs.path=/data/nfs stable/nfs-client-provisioner 
```

## 测试

创建test-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  annotations:
    volume.beta.kubernetes.io/storage-class: "course-nfs-storage"
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

## FAQ

1. kubernetes v1.20.1不能使用nfs作为storageclass,报错“provision "default/test-pvc" class "nfs-client": unexpected error getting claim reference: selfLink was empty”

```yaml
Current workaround is to edit /etc/kubernetes/manifests/kube-apiserver.yaml

Under here:

spec:
  containers:
  - command:
    - kube-apiserver
Add this line:
- --feature-gates=RemoveSelfLink=false
```