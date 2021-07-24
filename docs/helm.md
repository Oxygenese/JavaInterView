# kubernetes实战篇之helm安装

​		Helm是kubernetes的应用包管理工具,是CNCF孵化器下的一个项目,主要用来管理 Charts。类似于 Ubuntu 中的 APT 或 CentOS 中的 YUM.它提供了一种简单的方法来发现,分享和使用为kubernetes准备的软件包.它消除了繁杂的配置和部署,从而极大提高开发者的生效效率.

​		怎样来理解它呢,假设我们的项目非常复杂,同时需要部署api网关,注册中心,配置中心,web服务,数据库中间件,消息队列中间件和缓存中间件...这将会产生大量的配置文件,如果以上操行的顺序不对或者某些参数不对,就可能造成整个系统部署失败.如果你经过一段时间的实践熟悉了整个部署流程,但是把工作交给其它同事时他仍然可能需要大量的时间来了解部署方案,如果你要把自己的方案在互联网上分享,开发者往往想要一键部署,对于繁杂的配置可能会望而却步.实践中部署一个wordpress仅一个web项目和一个mysql服务器的部署就着实把不少开发者折腾的不轻,更不用说像上面复杂的配置了...

​		而helm正是要解决这样的问题,它把一系列复杂的有状态和无状态服务的部署封装起来(实际上就是对yaml文件的组织),然后你可以暴露出一些自定义参数信息供用户选择,这样部署就会变得简单很多.下面我们对helm的一些常用术语进行介绍并展示如何安装helm

## helm相关术语

- **Helm** 是一个命令行下的客户端工具。主要用于 Kubernetes 应用程序 Chart 的创建、打包、发布以及创建和管理本地和远程的 Chart 仓库。

- **Tiller** 是 Helm 的服务端，部署在 Kubernetes 集群中。Tiller 用于接收 Helm 的请求，并根据 Chart 生成 Kubernetes 的部署文件（ Helm 称为 Release ），然后提交给 Kubernetes 创建应用。Tiller 还提供了 Release 的升级、删除、回滚等一系列功能。

- **Chart** Helm 的软件包，采用 TAR 格式。类似于 APT 的 DEB 包或者 YUM 的 RPM 包，其包含了一组定义 Kubernetes 资源相关的 YAML 文件

- **Repoistory** Helm 的软件仓库，Repository 本质上是一个 Web 服务器，该服务器保存了一系列的 Chart 软件包以供用户下载，并且提供了一个该 Repository 的 Chart 包的清单文件以供查询。Helm 可以同时管理多个不同的 Repository。

- **Release** 使用 helm install 命令在 Kubernetes 集群中部署的 Chart 称为 Release

  注：需要注意的是：Helm 中提到的 Release 和我们通常概念中的版本有所不同，这里的 Release 可以理解为 Helm 使用 Chart 包部署的一个应用实例。

## Chart Install 过程

- Helm 从指定的目录或者 TAR 文件中解析出 Chart 结构信息。
- Helm 将指定的 Chart 结构和 Values 信息通过 gRPC 传递给 Tiller。
- Tiller 根据 Chart 和 Values 生成一个 Release。
- Tiller 将 Release 发送给 Kubernetes 用于生成 Release。

## Chart Update 过程

- Helm 从指定的目录或者 TAR 文件中解析出 Chart 结构信息。
- Helm 将需要更新的 Release 的名称、Chart 结构和 Values 信息传递给 Tiller。
- Tiller 生成 Release 并更新指定名称的 Release 的 History。
- Tiller 将 Release 发送给 Kubernetes 用于更新 Release。

## Chart Rollback 过程

- Helm 将要回滚的 Release 的名称传递给 Tiller。
- Tiller 根据 Release 的名称查找 History。
- Tiller 从 History 中获取上一个 Release。
- Tiller 将上一个 Release 发送给 Kubernetes 用于替换当前 Release。

## Chart 处理依赖说明

Tiller 在处理 Chart 时，直接将 Chart 以及其依赖的所有 Charts 合并为一个 Release，同时传递给 Kubernetes。因此 Tiller 并不负责管理依赖之间的启动顺序。Chart 中的应用需要能够自行处理依赖关系。

## 安装过程

1. 安装服务端（Tiller）

```
# 创建服务端
helm init --service-account tiller --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.16.2  --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

# 创建TLS认证服务端，参考地址：https://github.com/gjmzj/kubeasz/blob/master/docs/guide/helm.md
helm init --service-account tiller --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.16.2 --tiller-tls-cert /etc/kubernetes/ssl/tiller001.pem --tiller-tls-key /etc/kubernetes/ssl/tiller001-key.pem --tls-ca-cert /etc/kubernetes/ssl/ca.pem --tiller-namespace kube-system --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts


若遇到错误 failed to list: configmaps is forbidden: User “system:serviceaccount:kube-system:default” cannot list configmaps in the namespace “kube-system”
```

执行以下命令

```bash
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

> 这里在init的时候需要指定镜像源是因为init的时候衬tiller服务端,服务端是以deployment的方式安装的.

> 也可以尝试以下一键安装命令

```bash
$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

> 注意以下内容参照了[这一篇文章](https://www.hi-linux.com/posts/21466.html#部署-helm),实际在安装时没有及时记录下来,所以我的账户已经是正常的了,不知道是新版本的已经默认创建的sa还是我自己手动创建然后忘记了.这里也贴出来,初学的朋友不要害怕,即便是默认已经创建了,再执行以下命令也不会导致错误发生的.

给 Tiller 授权
因为 Helm 的服务端 Tiller 是一个部署在 Kubernetes 中 Kube-System Namespace 下 的 Deployment，它会去连接 Kube-Api 在 Kubernetes 里创建和删除应用。

而从 Kubernetes 1.6 版本开始，API Server 启用了 RBAC 授权。目前的 Tiller 部署时默认没有定义授权的 ServiceAccount，这会导致访问 API Server 时被拒绝。所以我们需要明确为 Tiller 部署添加授权。

```shell
kubectl create ns openebs
```

