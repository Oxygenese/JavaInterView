

前置环境：

```shell
systemctl stop firewalld #关闭防火墙
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0
swapoff -a  #临时关闭swap
vim /etc/fstab
#注释掉/dev/mapper/VolGroup00-LogVol01 swap                    swap    defaults
free -g #验证  swap均为0

vim /etc/hosts #配置主机名与ip
10.0.2.15 	master
10.0.2.4	node1
10.0.2.5	node2


172.30.0.14 master
172.30.0.15 ->139.155.225.40 node1


#把内网 IP 192.168.0.111 转向外网 IP 123.123.123.123
iptables -t nat -A OUTPUT -d 192.168.0.111 -j DNAT --to-destination 123.123.123.123
#把外网 IP 123.123.123.123 转向内网 IP 192.168.0.111
iptables -t nat -A OUTPUT -d 123.123.123.123 -j DNAT --to-destination 192.168.0.111


hostnamectl set-hostname <newhostname> #指定新的主机名

cat>/etc/sysctl.d/k8s.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

#遇见提示是只读文件系统的，运行如下命令
mount -o remount rw /
```



###### 机器环境信息：centos7.8

###### 卸载系统之前的docker：

```shell
sudo yum remove docker \
docker-client \
docker-client-latest \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-engine
```

###### 安装Docker-ce：



###### 安装yum工具

```shell
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

###### 添加docker的yum镜像源

```shell
 sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

###### 安装docker组件

```shell
sudo yum install -y docker-ce docker-ce-di containerd.io
```

###### 创建docker配置文件夹

```shell
sudo mkdir -p /etc/docker
```

###### 添加docker的阿里云镜像加速节点

```shell
sudo tee /etc/docker/daemon.json <<-'EOF'
{
"registry-mirrors":["https://82m9ar63.mirror.aliyuncs.com"]
}
EOF
```

###### 重新加载并重启docker

```shell
sudo systemctl daemon-reload
```

```shell
sudo systemctl restart docker
```

###### 设置开机启动

```shell
sudo systemctl enable docker
```

###### 配置kubernetes镜像源

```shell
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgchek=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

```

###### 查找含kube的软件包

```shell
yum list|grep kube
```

###### 安装指定版本kubernetes三件套

```shell
yum install -y kubelet-1.17.3 kubeadm-1.17.3 kubectl-1.17.3
```

###### 设置开机启动并重启

```shell
systemctl enable kubelet
```

```shell
systemctl start kubelet
```





###### 抓取kubernetes集群的以依赖镜像

**新建脚本文件master-images.sh**

```shell
vim master-images.sh
```

**粘贴以下内容：**

```shell
images=(
	kube-apiserver:v1.17.3
    kube-proxy:v1.17.3
	kube-controller-manager:v1.17.3
	kube-scheduler:v1.17.3
	coredns:1.6.5
	etcd:3.4.3-0
    pause:3.1
)

for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
#   docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName  k8s.gcr.io/$imageName
done
```

**运行脚本：**

```shell
chmod 700 master-images.sh
./master-images.sh
```





初始化master节点：

```shell
kubeadm init \
--apiserver-advertise-address=172.30.0.14 \
--image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \
--kubernetes-version v1.17.3 \
--service-cidr=172.96.0.0/16 \
--pod-network-cidr=172.244.0.0/16
```

由于默认拉取镜像地址为k8s.gcr.io，在国内无法访问，在这里指定阿里云镜像仓库地址。

​	参数解释：无类别域间路由（Classless Inter-Domain Routing、CIDR）是一个用户给用户分配IP地址以及在互联网上有效的路由IP数据包的对IP地址进行归类的方法。

```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
#初始化环境配置，直接执行下面三条命令
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
#接下来执行pod网络配置
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:
#这是master节点的地址及端口，以及访问token有效期两个小时
kubeadm join 172.30.0.14:6443 --token qy4wf6.2dt05hxohhdyhfqm \
    --discovery-token-ca-cert-hash sha256:a5ea9735b9146d7ed9b81c19733de0fddb2b6f8988a2c5cce62ed03eb1a234c7 

```

```shell
kubectl apply -f kube-flannel.yml  

#加入master节点
kubeadm join 10.0.2.15:6443 --token 3caqac.vslqlly1stdtcrkb \
    --discovery-token-ca-cert-hash sha256:bc9e41a370b776218dab7edab59d574e6ecc9028c44a9227749f7743589e96f9 
```

###### kube-flannel.yml  :

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.flannel.unprivileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
    seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
    apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
    apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
spec:
  privileged: false
  volumes:
    - configMap
    - secret
    - emptyDir
    - hostPath
  allowedHostPaths:
    - pathPrefix: "/etc/cni/net.d"
    - pathPrefix: "/etc/kube-flannel"
    - pathPrefix: "/run/flannel"
  readOnlyRootFilesystem: false
  # Users and groups
  runAsUser:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  # Privilege Escalation
  allowPrivilegeEscalation: false
  defaultAllowPrivilegeEscalation: false
  # Capabilities
  allowedCapabilities: ['NET_ADMIN']
  defaultAddCapabilities: []
  requiredDropCapabilities: []
  # Host namespaces
  hostPID: false
  hostIPC: false
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  # SELinux
  seLinux:
    # SELinux is unused in CaaSP
    rule: 'RunAsAny'
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
rules:
  - apiGroups: ['extensions']
    resources: ['podsecuritypolicies']
    verbs: ['use']
    resourceNames: ['psp.flannel.unprivileged']
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes/status
    verbs:
      - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-amd64
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - amd64
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image:  jmgao1983/flannel:v0.11.0-amd64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image:  jmgao1983/flannel:v0.11.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-arm64
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - arm64
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image:  jmgao1983/flannel:v0.11.0-amd64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image:  jmgao1983/flannel:v0.11.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
             add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-arm
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - arm
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image:  jmgao1983/flannel:v0.11.0-amd64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image:  jmgao1983/flannel:v0.11.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
             add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-ppc64le
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - ppc64le
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: jmgao1983/flannel:v0.11.0-amd64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image:  jmgao1983/flannel:v0.11.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
             add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-s390x
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - s390x
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image:  jmgao1983/flannel:v0.11.0-amd64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image:  jmgao1983/flannel:v0.11.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
             add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg

```

查看网络状态：

```shell
kubectl get pods --all-namespaces
```

```shell
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
kube-system   coredns-7f9c544f75-7247c                 1/1     Running   0          47m
kube-system   coredns-7f9c544f75-lc678                 1/1     Running   0          47m
kube-system   etcd-vm-0-14-centos                      1/1     Running   0          47m
kube-system   kube-apiserver-vm-0-14-centos            1/1     Running   0          47m
kube-system   kube-controller-manager-vm-0-14-centos   1/1     Running   0          47m
kube-system   kube-flannel-ds-amd64-svmqk              1/1     Running   0          2m49s
kube-system   kube-proxy-rqvlg                         1/1     Running   0          47m
kube-system   kube-scheduler-vm-0-14-centos            1/1     Running   0          47m
```

若全为Running状态则master节点初始化成功



```shell
kubectl get pods -n kube-system #查看指定空间名称的pods
kubectl get pods --all-namespaces #查看所有名空间的pods 
```

```shell
ip link set cni down #如果网络出现问题，关闭cni0，重启机器继续测试
watch kubectl get pod -n kube-system -o wide  #监控pod进度
#等待3-10分钟，状态全部租转为running以后继续
```





子节点加入Kubernetes Node





在node节点执行

像集群添加新的节点，执行在kubeadm init 输出的kubeadm join命令：

确保node节点成功

在子节点运行以下命令

```shell
kubeadm join 172.30.0.14:6443 --token qy4wf6.2dt05hxohhdyhfqm \
    --discovery-token-ca-cert-hash sha256:a5ea9735b9146d7ed9b81c19733de0fddb2b6f8988a2c5cce62ed03eb1a234c7 
```

若token过期：

```shell
kubeadm token create --print-join-command
```













helm-rbac.yml

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: helm
  name: helm-admin
  namespace: kube-system
 
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: helm-admin
  labels:
    k8s-app: helm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: helm-admin
    namespace: kube-system
```

helm初始化：

```shell
kubectl apply -f helm-rbac.yml
serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
  
kubectl apply -f serviceaccount.yaml

helm init --service-account=tiller --tiller-image=sapcc/tiller:v2.16.2 --history-max 300
```

去除污点：

```shell
kubectl taint nodes master node-role.kubernetes.io/master:NoSchedule-
```

创建openebs名称空间：

```shell
kubectl create ns openebs
```

helm安装openebs：

```shell
helm install --namespace openebs --name openebs stable/openebs --version 1.5.0
```

若遇到以下报错（helm权限问题）：

```shell
Error: release openebs failed: namespaces "openebs" is forbidden: User "system:serviceaccount:kube-system:tiller" cannot get resource "namespaces" in API group "" in the namespace "openebs"
```

则运行以下命令：

```shell
kubectl create serviceaccount --namespace kube-system tiller

kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

设置默认存储空间：

```shell
 kubectl patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

最小化安装：

```shell
kubectl apply -f https://raw.githubusercontent.com/kubesphere/ks-installer/master/kubesphere-minimal.yaml
#如果404就去网上搜这个文件
#master节点一定要保证有足够的cpu资源和8G以上的内存
```

```bash
 kubectl edit cm -n kubesphere-system ks-installer
```