# Kubernetes Helm

## Helm基础

​	Helm把Kubernetes的资源打包到一个Chart中，并将制作、测试完成的各个Chart保存到仓库进行存储和分发。Helm还实现了可配置的发布，它支持应用配置的版本管理，简化了Kubernetes部署应用的版本控制、打包、发布、删除、更新等操作，它有以下几个概念。

- Chart：即一个Helm程序包，它包含了运行一个Kubernetes应用所需要的镜像、依赖关系和资源定义等。
- Repository：集中存储和分发的Chart的仓库。
- Config：Chart实例化安装运行时使用的配置信息。
- Release：Chart实例化配置后运行于Kubernetes集群中的一个应用实例；在同一个集群上，一个Chart可以使用不同的Config重复安装多次，每次安装都会创建一个新的release。

因此，Chart更像是存储在Kubernetes集群之外的程序，它的每次安装是指集群中使用专用配置运行的一个实例，执行过程有点儿类似于在操作系统上基于程序启动一个进程。

​	通常，用户在Helm客户端本地遵循其格式编写Chart文件，而后即可部署在Kubernetes集群上，运行为一个特定的release。仅在有分发需求时，才应该把同一应用的Chart文件打包成归档压缩格式提交到特定的Chart仓库。仓库可以运行为公共托管平台，也可以是用户自建的服务器，仅供特定的组织或个人使用。

​	目前，Helm的主流可用版本主要有v2和v3两个。版本v2中，Helm主要由与用户交互的客户端、与Kubernetes API交互的服务端Tiller和Chart仓库（repository）组成。Helm客户端是一个命令行工具，采用Go语言开发，它主要负责本地Chart开发、管理Chart仓库，以及基于gRPC协议与Tiller交互，从而完成应用部署、查询等管理任务。而Tiller服务器则托管运行在Kubernetes上，负责接受Helm客户端请求、将Chart转换为最终配置生成一个release，随后部署、跟踪以及管理各release等功能。

​	版本v2进化到版本v3的过程中，Helm客户端基本保持了原貌，但肩负重要任务的服务端组件Tiller被移除，取而代之是专用的CRD资源。换句话说，版本v3的Helm使用CRD将release直接保存到Kubernetes之上，且无须再跟踪各release状态，而将Chart渲染成release的功能也移往Helm客户端，从而不必再用到Tiller组件。

​	事实上，由于Tiller拥有管理Kubernetes集群的密钥，在集群内公开了无须身份验证的gRPC端点且又无法在客户端用户上实现RBAC方式的授权管理功能，从而为Kubernetes集群引入了许多不确定的安全风险。因而，移除Tiller也是势在必行。好在，Helm的优势都基本保持未变。以下优势依然存在。

- 管理复杂应用：Chart能够描述哪怕是最复杂的程序结构，提供了可重复使用的应用安装定义。
- 易于升级：使用就地升级和自定义钩子来解决更新的难题。
- 简单分享：Chart易于通过公共或私有服务完成版本化、共享及主机构建，且目前有众多成熟的Chart可供使用。
- 回滚：使用helm rollback命令轻松实现快速回滚。

​	但凡事皆有两面性，Helm也存在着一些问题，而且有些问题甚至是原生的，甚至是其立足之本。例如，Helm有着众多的抽象层，因而学习曲线比较陡峭；即便程序包管理器甚至默认的设定就能运行，但它也几乎必然会要求用户通过自定义来适配到自有环境，这不仅提升了在CI/CD管道中的处理复杂度，而且可读性较差的模板几乎不可避免地会随着时间的流逝而越来越丧失可定制性等。

## Helm快速入门

​	Helm客户端工具支持预编译的二进制程序和源码编译两种安装方式，但它释出的每个版本都为各主流操作系统提供了适用的预编译版本，用户安装前按需下载适合的发型版本即可，大大简化了程序包的部署难度。

​	Helm项目托管在GitHub之上，项目地址为https://github.com/kubernetes/helm，下载到合用版本的压缩包后并将其展开，而后将二进制程序文件复制或移动到系统PATH环境变量指向的目录中即可完成安装。

```shell
# tar -xf helm-v3.2.4-linux-amd64.tar.gz
# mv linux-amd64/helm /usr/local/bin
```

​	Helm的各种管理功能均通过其子命令完成，例如要获取其简要使用帮助，直接使用如下所示的help子命令即可。但需要注意的是，Helm与Kubernetes API Server通信依赖于本地安装并配置的kubectl，因此运行Helm的节点也应该是可以正常使用kubectl命令的主机，或者至少是有着可用kubeconfig配置文件的主机。

​	如前面所述，Helm可基于本地自行开发的Chart或经由公开仓库获取到的Chart完成应用管理。Helm项目维护的stable和incubator仓库中保存了一系列精心制作与维护的Chart，因而使用Helm完成应用管理的第一步通常是将它们添加为helm命令可以使用本地仓库，从而能够使用内部的Chart。

```shell
# helm repo add stable https://kubernetes-charts.storage.googleapis.com/
"stable" has been added to your repositories
# helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com
"incubator" has been added to your repositories
```

​	Helm Hub上还维护着众多第三方仓库，这些仓库都可以由helm repo 命令添加到本地直接使用，而且也只能添加为本地仓库后才能够作为内部Chart使用。例如，维护有众多应用Chart且灵活度较高的Bitnami组织也拥有自己的Chart仓库，我们同样可以将其添加为本地仓库。

```shell
# helm  repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
```

​	helm repo 相关命令用于管理使用Charts仓库，其update子命令能够更新使用的默认仓库的元数据信息，其命令及执行结果如下。

```shell
# helm repo update
wang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "incubator" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "bitnami" chart repository
```

​	另外，除了add和update之外，helm repo命令还支持list和remove等子命令，前者用于列出本地配置的Chart仓库，而后者则用于移除指定的仓库。若要了解有哪些Chart可用，则要使用支持基于关键字搜索功能的helm search命令，他结合hub子命令可搜索helm hub或Monocular实例。而与repo子命令一起使用则能够搜索本地配置的各个仓库。前面添加的stable和incubator仓库中维护着主流Chart集合，下面的搜索命令可直接列出该仓库中含有etcd关键字的所有Chart及简要描述。

```shell
# helm search repo etcd
```

​	在整个Helm Hub中进行搜索，意味着直接在线搜索Hub中存在的所有仓库，因而通常能得到更多的结果。

```shell
# helm search hub etcd
```

​	省略搜索命令中的关键字能列出Helm Hub或本地配置的仓库中的所有Chart，而若要查看某个Chart的详细信息，则要使用helm show或helm inspect命令，它能够通过all子命令显示Chart的所有信息，或者通过chart、readme和values子命令分别显示Chart的定义、README信息及默认的值文件内容。

```shell
# helm show all stable/etcd-operator
```

​	如前面所示，将Chart中定义的应用部署到Kubernetes集群上生成的部署实例称为Release，一个集群上可能会存在某一Chart的多个实例，而Release名称便成了区分这些实例的关键标识，因此，部署命令helm install便要求为各Release指定一个专有的名称。

```shell
# helm install [NAME] [CHART] [flags]
```

​	我们知道，Helm是一个模板引擎，它使用值文件渲染模板生成具体的资源配置，但每个Chart通常都要为其模板提供一个具有通用性的默认值文件，因而多数情况下，用户无须提供这类文件也能够直接部署生成一个实例。例如，下面的命令尝试在test名称空间中基于指定的Chart(stable/etcd-operator)及其默认的值文件部署出一个名为testop的实例。

```shell
# helm install testop stable/etcd-operator --dry-run -n test
```

​	由于使用了--dry-run选项，该命令仅会打印生成的最终资源配置，并会给出部署后的使用提示。若无错误信息返回且各项配置项符合需要，移除该选项即可进入安装流程。Helm自身并不会创建目标名称空间，因此请务必确保引用的名称空间事先存在。

```shell
# kubectl create namespace test
# helm install testop stable/etcd-operator -n test
manifest_sorter.go:192: info: skipping unknown hook: "etcd-install"
NAME: testop                              # Relase名称
LAST DEPLYED: Thu Sep 17 11:45:36 2020    # 部署时间
NAMESPACE: test                           # 名称空间
STATUS: deployed                          # 部署状态
REVISION: 1                               # 历史版本
TEST SUITE: None                          # 测试套件状态，拥有测试套件的Release可由helm test命令进行测试
NOTES:
.........
```

​	命令返回的结果称为Release的状态信息，这些信息随后还可以通过helm status命令再次获取。另外，helm install命令支持基于多种安装源进行应用部署，这包括Chart仓库、本地的Chart压缩包、本地Chart目录，甚至是指定一个Chart的URL等。

​	对于多数有状态应用，其Helm Chart主要用于部署生成管理该类应用集群的Operator，而后由用户根据Operator提供的CRD实现必要的应用管理功能，例如etcd-operator项目提供的集群部署和管理(etcdclusters)、数据备份(etcdbackups)和数据恢复(etcdrestores)等自定义资源类型及相关的Operator控制器等。我们可以使用Helm客户端命令列出已经安装的Release，来了解该Operator部署状况。

```shell
# helm list -n test
NAME         NAMESPACE    REVISION        UPDATED         STATUS        .....
testop          test         1          2020-09-17       deployed       .....
```

​	该Operator生成的Pod资源也都是自定义的控制器程序，它们用于确保相关的自定义资源符合用户期望的状态。

```shell
# kubectl api-resources --api-group etcd.database.coreos.com
NAME               SHOPTNAMES    APIGROUP                    NAMESPACED    KIND
etcdbackups                      etcd.database.coreos.com       true       EtcdBackup
etcdclusters         etcd        etcd.database.coreos.com       true       EtcdCluster
etcdrestores                     etcd.database.coreos.com       true       EtcdRestore
# kubectl get pods -n test -l release=testop
NAME                                                               READY         ......
testop-etcd-operator-etcd-backup-operator-66f5cf5bc8-w2mfl          1/1        
testop-etcd-operator-etcd-operator-7f857bdd84-rssmf                 1/1
testop-etcd-operator-etcd-restore-operator-7597fc47f7-2tslk         1/1
```

​	于是，用到etcd存储集群时，用户根据自定义资源etcdCluster声明出一个etcd集群规范，并应用于Kubernetes之上，该Operator即会自行确保其能够正确运行。

​	对于运行中的Release，我们可以使用helm get命令获取相关的钩子、最终资源配置清单、注意事项和用户自定义的模板参数值信息，相关信息的获取各自依赖于相关的子命令，或者直接使用子命令all一次性的列出所有这些信息。下面命令结果显示出，用户在部署testop实例时未提供任何自定义值。

```shell
# helm get values testop -n test
USER-SUPPLIED VALUES:
null
```

​	升级或回滚应用，要分别使用helm upgrade和helm rollback命令，还可以使用helm history命令获取指定Release的变更历史。

