# 《实战公有云Kubernetes》动手实验营
|||
|------|---------------------|
| 作者 | 陈耿 微软全球技术黑带 |
|联系|微信公众号“云来有道”|

Kubernetes是云计算时代的操作系统，Kubernetes可以运行在几乎所有的主流基础架构之上：笔记本、私有数据中心、私有云、公有云，甚至是Raspberry Pi之上。许多朋友对Kubernetes的认识都是从私有基础架构开始。公有云是Kubernetes的一个重要的场景，具有广阔的前景。本系列实验将通过一些实际的例子给大家介绍有关公有云的Kubernetes实现和特点，帮助大家了解如何使用公有云Kubernetes服务更好地用好Kubernetes，提高应用开发、部署和管理的效率。

# 目录
- [实验一 冲上云霄的Kubernetes！](#lab01)
- [实验二 云上的Kubernetes运维管理](#lab02)
- [实验三 Kuberentes的容器应用开发](#lab03)

# <a name="lab01"></a>实验一 冲上云霄的Kubernetes！
实验难度：初级 | 实验用时：45分钟

创建一个Kubernetes集群有多种方式。用户可以手工创建所需要的主机、网络和存储资源，然后手工地部署一个Kubernetes集群。手工部署的缺点是耗时费力。最便捷的方式是通过Kubernetes公有云服务快速获取一个可用的Kubernetes集群。

## 1 热身运动
### 1.1 注册Azure公有云账号
本文将通过Azure公有云的Kubernetes服务Azure Kubernetes Service（AKS）快速地创建一个Kubernetes集群。还没有Azure账号的同学可以通过以下的连接免费申请，目前Azure为新用户提供了200美金的免费使用额度，足以支持完成本实验的内容。
>提示！点击打开注册页面 https://azure.microsoft.com/zh-cn/free/

### 1.2 云端的Shell
用户可以通过Azure提供的Web控制台，完成对Kubernetes集群的创建。但是为了让描述更为准确，本文使用Azure的命令行工具Azure CLI完成相关的演示操作。

要使用Azure CLI，读者可以在自己的电脑上安装该工具。但是，在云的时代，我们也可以利用云所提供的便利，直接在云上使用所需要的工具。Azure Cloud Shell是Azure提供的一个Web Shell，用户可以在Azure Cloud Shell中使用Azure CLI、Terraform及Ansible等云管工具。

> 提示！ 点击打开Azure Cloud Shell：https://shell.azure.com/

打开Cloud Shell后，输入命令`az -v`可以查看Azure CLI的版本。

    user@Azure$ az -v

### 1.3 本地安装Azure CLI （可选）
Azure Cloud Shell固然很方便，但是有的朋友还是喜欢在本地执行命令。这时可以在本地机器安装Azure CLI，详细安装部署请参考下面的连接。
> 提示！点击打开Azure CLI安装文档 ：https://docs.microsoft.com/zh-cn/cli/azure/install-azure-cli

## 2 创建Kubernetes集群

### 2.1 通过AKS创建Kubernetes集群
执行如下命令在Azure上创建一个Kubernetes集群。

    $ az group create -n k8s-cloud-labs -l eastus
    $ az aks create -g k8s-cloud-labs -n k8s-cluster --disable-rbac --generate-ssh-keys

第一条命令是创建一个资源组，可以认为这是Azure上的存放对象的文件夹。这里我们选择使用East US数据中心。

第二条命令是真正创建Kubernetes集群的命令。`az aks`命令是操作AKS服务的子命令。我们在资源组`k8s-cloud-labs`中创建了一个名为`k8s-cluster`的集群。命令执行后，稍等片刻。5-10分钟后一个生产可用的Kubernetes将会就绪。

> 提示！AKS提供Kubernetes RBAC鉴权模型的支持。为了简化实验环境，本文通过参数`--disable-rbac`禁用了此功能。在安全方面，Azure的Kubernetes用户可以将AKS的Kubernetes集群与Azure Active Directory服务进行集成，实现Kubernetes与企业身份验证系统的对接。

执行如下命令可以查看当前Azure账号下已经创建好的Kubernetes集群列表。

    $ az aks list -o table
   
> 提示！命令`az aks`还有许多有用的参数，通过命令`az aks -h`可以查看更多详细的信息。

### 2.2 访问Kubernetes集群
Kubernetes集群就绪后，下一步就可以通过Kubernetes的命令行`kubectl`对集群进行访问。读者可以自己到Kubernetes的GitHub主页上下载对应版本的kubectl。也可以在本地环境中直接执行如下命令自动下载并安装kubectl。

    $ az aks install-cli

> 提示！Azure Cloud Shell默认已经提供kubectl命令，无需执行上述安装。

kubectl安装完毕后，执行如下命令获取Kubernetes集群的连接信息和访问密钥。

    $ az aks get-credentials -g k8s-cloud-labs -n k8s-cluster
    Merged "k8s-cluster" as current context in /home/nicholas/.kube/config

一切就绪后，便可以通过`kubectl`命令操作Kuberentes集群。

    $ kubectl get nodes
    NAME                       STATUS    ROLES     AGE       VERSION
    aks-nodepool1-16810059-0   Ready     agent     10m       v1.9.11
    aks-nodepool1-16810059-1   Ready     agent     10m       v1.9.11
    aks-nodepool1-16810059-2   Ready     agent     10m       v1.9.11

通过输出看到命令列出了Kubernetes集群的节点列表。`az aks create`命令默认创建了一个含有3个计算节点的集群。通过参数`--node-count`用户可以指定集群计算节点（node）的数量。目前单集群最大支持100个节点。

> 提示！命令`kubectl get nodes`的输出中只包含了Kubernetes集群的Node节点，而没有包含Master节点，这是为什么呢？这是因为AKS提供的是一个Kubernetes的托管服务，Kuberentes的Master节点将由Azure负责运维，用户无需操劳。用户也无需为Master节点所使用的资源支付任何费用。用户可以专注于Kubernetes Node节点的应用部署和管理。这将极大地简化了Kubernetes集群的运维，也节省了费用开销。

### 2.3 部署一个Nginx容器

通过AKS，我们快速地获取了一个生产可用的Kubernetes集群，接下来我们将部署一个简单的Nginx容器到刚创建好的Kubernetes集群。

执行如下命令创建一个Kubernetes的命名空间。

    $ kubectl create namespace lab01

在新创建的命名空间下部署一个Nginx应用。命令和示例输出如下：

    $ kubectl run frontend --image nginx:1.15 -n lab01
    deployment.apps/frontend created

稍等片刻后可以看到Nginx的容器被成功地部署并运行。

    $ kubectl get pod -n lab01
    NAME                        READY     STATUS    RESTARTS   AGE
    frontend-54fccfb9d4-kb5pl   1/1       Running   0          2m

### 2.4 对外发布服务

为了访问这个Nginx容器所提供的HTTP服务，我们创建一个Kubernetes Service对象。执行命令如下：

    $ kubectl expose deployment frontend --type LoadBalancer --port 80 -n lab01

上文指定Service的类型为`LoadBalancer`，这样Azure将会为这个Service分配一个公网IP地址。稍等片刻，查看刚创建的Service时便可以看到`EXTERNAL-IP`一栏中显示了该Service的公网IP地址。

    $ kubectl get svc -n lab01
    NAME       TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
    frontend   LoadBalancer   10.0.90.138   137.135.82.191   80:31650/TCP   3m

通过命令`curl`或浏览器访问Service的公网IP地址便可以看到Nginx服务返回的HTML代码。

    $ curl 137.135.82.191
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>

> 提示！因为Kubernetes在Azure公有云之上，因此可以很便捷地通过LoadBalancer类型的Service将运行在Kubernetes集群上的容器应用服务发布到互联网上。除了Service之外，AKS还提供了Application Routing这一Ingress功能，帮助用户快速将应用发布到互联网上，对外提供服务。

### 2.5 管理控制台
AKS提供的是经过测试的原生的Kubernetes集群，默认也部署了Kubernetes Dashboard。用户在本地主机上可以通过如下命令直接打开Dashboard。
    
    $ az aks browse -g k8s-cloud-labs -n k8s-cluster

### 2.6 清理环境
实验结束，删除Kubernetes命名空间，清理实验环境。

    $ kubectl delete ns lab01

# <a name="lab02"></a>实验二  云上的Kubernetes运维管理
实验难度：中级 | 实验用时：45分钟

## 1 Kubernetes的应用部署和管理
在前面的实验里面，我们通过命令`kubectl run`部署了一个Nginx容器。在实际的工作环境中，应用的部署往往涉及多个组件和复杂的参数配置。用户需要一个高效工具来提升应用部署和管理的效率。Helm是Kubernetes的软件包管理工具。类似Linux世界的RPM和APT，Helm可以帮助用户方便地部署和管理Kubernetes集群上的容器软件。

Helm是Microsoft团队创建的开源项目。目前，Helm也已经是CNCF的项目成员。Helm已经成为了Kubernetes社区最受欢迎的应用部署和管理工具！

### 1.1 安装Helm（可选）
Azure Cloud Shell的环境中已经包含了Helm的执行文件，无需额外安装。使用本地环境的同学可以从Helm的GitHub主页上下载Helm的二进制文件，并放置于系统环境变量`PATH`所指定的可执行文件搜索路径中。

> 提示！点击下载Helm：https://github.com/helm/helm/releases

### 1.2 配置Helm

在使用Helm管理Kuberentes集群的应用以前，需要执行以下命令进行集群的初始化配置。

    $ helm init

### 1.4 部署WordPress应用

执行命令`helm install`安装一个WordPress应用。

    $helm install stable/wordpress --namespace lab02

执行部署后，通过命令`helm list`看查看到当前集群已经部署的容器应用。

    $ helm list
    NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
    mothy-bobcat    1               Sat Dec  1 16:03:38 2018        DEPLOYED        wordpress-4.0.0 4.9.8           lab02

### 1.5 数据持久化

WordPress应用包含了一个前端和一个MariaDB的后端。MariaDB的数据通过Persistent Volume进行了持久化。查看Persistent Volume Claim，可以看到相关的配置。

    $ kubectl get pvc -n lab02
    NAME                          STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    data-mothy-bobcat-mariadb-0   Bound     pvc-aaad63e8-f582-11e8-818c-bad14be25da1   8Gi        RWO            default        1m
    mothy-bobcat-wordpress        Bound     pvc-aa905ca9-f582-11e8-818c-bad14be25da1   10Gi       RWO            default        1m

AKS管理的Kubernetes集群默认已经与Azure Disk进行了集成。当用户在Kubernetes中创建PVC时，Azure Disk PV将会自动创建并分配给相应的PVC。通过命令`az disk`可以查看到自动创建的Azure Disk磁盘。

    $ az disk list -o table |grep -i k8s-cloud-labs|grep pvc
    kubernetes-dynamic-pvc-aa905ca9-f582-11e8-818c-bad14be25da1         MC_K8S-CLOUD-LABS_K8S-CLUSTER_EASTUS                       eastus
    Standard_LRS               10        Succeeded
    kubernetes-dynamic-pvc-aaad63e8-f582-11e8-818c-bad14be25da1         MC_K8S-CLOUD-LABS_K8S-CLUSTER_EASTUS                       eastus
    Standard_LRS               8         Succeeded

### 1.6 访问应用

部署后稍等片刻，可以看到成功启动的WordPress的容器。

    $ kubectl get pod -n lab02
    NAME                                      READY     STATUS    RESTARTS   AGE
    mothy-bobcat-mariadb-0                    1/1       Running   0          4m
    mothy-bobcat-wordpress-868c995d88-r8fdp   0/1       Running   0          4m

Helm部署时同时创建了相应的Service对象。

    $ kubectl get svc -n lab02
    NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
    mothy-bobcat-mariadb     ClusterIP      10.0.136.135   <none>         3306/TCP                     5m
    mothy-bobcat-wordpress   LoadBalancer   10.0.22.40     40.87.91.243   80:32485/TCP,443:30817/TCP   5m

通过WordPress的Service LoadBalancer的公网IP地址就可以访问到WordPress应用。如上面的例子，通过浏览器访问http://40.87.91.243/。

### 1.7 使用Application Routing
Ingress是Kubernetes实现引流外部访问请求到容器的一种手段。AKS提供了Application Routing这一功能作为一种Kubernetes Ingress的实现。用户可以在AKS的Kubernetes集群中启用Application Routing这功能。

    $ az aks enable-addons --addons http_application_routing -g k8s-cloud-labs -n k8s-cluster

Application Routing这一功能启用后，可以在Kubernetes集群中看到相应的容器组件。

    $ kubectl get pod -n kube-system|grep routing
    addon-http-application-routing-default-http-backend-67df49rm2nx   1/1       Running   0          3m
    addon-http-application-routing-external-dns-66c8b5fdb9-7q24p      1/1       Running   0          3m
    addon-http-application-routing-nginx-ingress-controller-6827p6j   1/1       Running   0          3m

使用Appliation Routing需要创建Kubernetes Ingress对象，描述域名与Service的对应关系。这样Ingress Controller在接收到一个访问特定域名的请求时，将会把请求转发给相应的Service及其后端的容器实例。

首先，查看当前WordPress应用的Service名称，这将在定义Ingress对象时使用。下面是命令和示例输出。

    $ kubectl get svc -o name -n lab02|grep wordpress
    service/mothy-bobcat-wordpress

AKS为Kubernetes提供了一个应用可用的域名。通过下面的命令可以查询到当前集群所分配到的域名。下面是命令和示例输出。

    $ az aks list --query "[?name=='k8s-cluster'].addonProfiles.httpApplicationRouting.config.HTTPApplicationRoutingZoneName"|tail -1
    4e8d71ef819b44169868.eastus.aksapp.io

根据前面查询到的域名创建相应的Ingress对象。命令如下：

    $ echo '
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
     name: wordpress
     annotations:
        kubernetes.io/ingress.class: addon-http-application-routing
    spec:
      rules:
      - host: wordpress.4e8d71ef819b44169868.eastus.aksapp.io
        http:
          paths:
          - backend:
              serviceName: mothy-bobcat-wordpress
              servicePort: 80
            path: /
    '|kubectl create -f - -n lab02

Ingress对象创建完毕后，需要稍等片刻。Azure将为应用创建Ingress所指定的域名。通过下面的命令输出可以看到，Azure DNS Zone里多了两条和WordPress应用相关的域名记录。

    $ az network dns record-set list -g mc_k8s-cloud-labs_k8s-cluster_eastus -z 4e8d71ef819b44169868.eastus.aksapp.io
    Fqdn                                              Name       ProvisioningState    ResourceGroup                         Ttl
    ------------------------------------------------  ---------  -------------------  ------------------------------------  ------
    4e8d71ef819b44169868.eastus.aksapp.io.            @          Succeeded            mc_k8s-cloud-labs_k8s-cluster_eastus  172800
    4e8d71ef819b44169868.eastus.aksapp.io.            @          Succeeded            mc_k8s-cloud-labs_k8s-cluster_eastus  3600
    wordpress.4e8d71ef819b44169868.eastus.aksapp.io.  wordpress  Succeeded            mc_k8s-cloud-labs_k8s-cluster_eastus  300
    wordpress.4e8d71ef819b44169868.eastus.aksapp.io.  wordpress  Succeeded            mc_k8s-cloud-labs_k8s-cluster_eastus  300

域名创建成功后，通过下面的域名就可以访问到WordPress应用了。

    http://wordpress.4e8d71ef819b44169868.eastus.aksapp.io

### 1.7 应用日志和监控管理
Kubernetes的日志和监控指标的收集可以通过开源的Fluentd、Elastic Search，Kibana及Prometheus实现。Azure提供了Azure Monitor服务可以对Kubernetes集群的容器日志和监控指标进行收集和展示。通过下面的命令可以为Kubernetes集群开启Azure Monitor的支持。

    $ az aks enable-addons --addons monitoring -g k8s-cloud-labs -n k8s-cluster

服务开启后，可以在Azure的Web控制台中查看容器的日志和监控信息。Azure提供了丰富和直观的查询和展示功能。

> Azure Monitor开启后，需要等待片刻，待完成初始的数据采集后方会返回查询结果和展示图表信息。

### 1.8 Kubernetes集群的伸缩
在公有云上使用Kubernetes一个优势就在于可以利用公有云庞大的计算资源。AKS提供了Kubernetes集群的弹性伸缩，通过简单的命令或者界面操作，用户就可以便捷地为Kubernetes集群添加或者删除节点。

查看当前Kubernetes集群的节点信息。通过下面的示例输出可以看到，当前集群有3个节点。节点的虚拟机规格是`Standard_DS2_v2`，操作系统为Linux。AKS对Windows的支持已经在研发计划中。

    $ az aks list --query "[?name=='k8s-cluster'].agentPoolProfiles[0]" -o table
    Name       Count    VmSize           OsDiskSizeGb    StorageProfile    MaxPods    OsType
    ---------  -------  ---------------  --------------  ----------------  ---------  --------
    nodepool1  3        Standard_DS2_v2  30              ManagedDisks      110        Linux

通过命令`az aks scale`用户可以对集群进行伸缩。下面的例子是将集群扩展至4个节点。

    $ az aks scale -g k8s-cloud-labs -n k8s-cluster -c 4

命令执行后，稍等片刻。在此查看集群节点状态便可以看到集群已经成功扩展至4个节点。
    
    $ az aks list --query "[?name=='k8s-cluster'].agentPoolProfiles[0]" -o table
    Name       Count    VmSize           OsDiskSizeGb    StorageProfile    MaxPods    OsType
    ---------  -------  ---------------  --------------  ----------------  ---------  --------
    nodepool1  4        Standard_DS2_v2  30              ManagedDisks      110        Linux

除了快速为集群扩容外，也可以对集群进行缩容。在业务空闲时间，通过缩容，用户可以节省不必要的开销。在下面的例子里，我们将集群的节点缩减至2个。

    $ az aks scale -g k8s-cloud-labs -n k8s-cluster -c 2

缩容操作完成后检查集群节点，可以看到集群节点数已减少至2个。

    $ az aks list --query "[?name=='k8s-cluster'].agentPoolProfiles[0].count"
    Result
    --------
    2

### 1.9 Kubernetes集群的升级
作为一个开源项目Kubernetes的发展是非常迅速的，集群版本升级也是一个常见的场景。AKS提供了一键式的集群版本升级，使得集群的升级的复杂度大大降低。

通过命令`az aks get-upgrades`用户可以看到当前集群可以升级到哪些更新的版本。

    $ az aks get-upgrades -g k8s-cloud-labs -n k8s-cluster
    Name     ResourceGroup    MasterVersion    NodePoolVersion    Upgrades
    -------  ---------------  ---------------  -----------------  --------------
    default  k8s-cloud-labs   1.9.11           1.9.11             1.10.8, 1.10.9

> 提示！AKS是业界少有同时支持3个以上Kubernetes版本的服务。AKS有一套成文的Kubernetes版本支持策略，详情请参考：https://docs.microsoft.com/en-us/azure/aks/supported-kubernetes-versions

当决定了目标升级版本后，通过命令`az aks upgrade`就可以将集群升级到指定的版本。执行如下命令，将集群升级至版本1.10.9。

    $ az aks upgrade -g k8s-cloud-labs -n k8s-cluster -k 1.10.9
    Kubernetes may be unavailable during cluster upgrades.
    Are you sure you want to perform this operation? (y/n): y

稍等片刻。集群升级完毕后，再次查看集群的状态，可以发现相关的Kubernetes组件已经被更新至指定的版本。

    $ kubectl get nodes
    NAME                       STATUS    ROLES     AGE       VERSION
    aks-nodepool1-16810059-1   Ready     agent     4m        v1.10.9
    aks-nodepool1-16810059-2   Ready     agent     11m       v1.10.9

### 2 清理环境
实验结束，删除Kubernetes命名空间，清理实验环境。

    $ kubectl delete ns lab02

# <a name="lab03"></a>实验三 结合Kuberentes的容器应用开发
实验难度：中级 | 实验用时：45分钟
## 1 准备本地开发环境
请在本地开发环境中安装如下工具。
- Azure CLI > 2.0。安装文档：https://docs.microsoft.com/zh-cn/cli/azure/install-azure-cli?view=azure-cli-latest
- Git > 2.1。安装文档：https://git-scm.com/downloads

使用Windows的同学，可以使用Git Bash执行本文的代码。

### 1.1 下载应用代码

下载本实验所使用的示例代码。该应用是一个基于Spring Boot的RESTful应用。

    $ git clone https://github.com/nichochen/japp-spring-boot-rest.git japp 

### 1.2 应用容器化
要把这个Java应用在Kubernetes上运行起来，首先需要先将应用进行容器化。一般而言，开发人员需要编写Dockerfile。为了方便部署，也需要编写和准备相应的Helm Chart。这个应用容器化的工作，可以手工完成，但是也可以借助工具来提高效率。Draft就是一个帮助应用容器化的工具。Draft是Helm的创作团队的又一力作。

### 1.3 下载Draft
可以在Draft的主页上下载其二进制执行文件。并将其放置于环境变量PATH所指定的二进制搜索路径中。

> 点击下载Draft：https://github.com/Azure/draft/releases

### 1.4 配置Draft
Draft下载完毕后，执行命令`draft init`，Draft将自动配置本地的环境，下载需要的文件和配置。

    $ draft init

### 1.5 创建容器化配置
执行命令`draft create`生成应用容器化所需的Dockerfile和Helm Chart等配置。Draft可以根据代码的类型自动选择生成的模板。

    $ cd japp
    $ draft create

Draft将在应用的目录下生成许多配置相关的文件。

    $ ls -a
    ./   .dockerignore  .draft-tasks.toml  .mvn/    Dockerfile  mvnw*     pom.xml
    ../  .draftignore   .gitignore         charts/  draft.toml  mvnw.cmd  src/

 编辑draft.toml将属性`namespace`修改为`lab03`。这样后续应用将会被部署到命名空间`lab03`里。

    namespace = "lab03"

### 1.6 镜像仓库
应用容器化后将会生成容器镜像。容器镜像的存放也是一个需要重点关注的问题。用户可以将镜像存放在公共的镜像仓库或私有的镜像仓库中。Azure Container Registry（ACR）是Azure提供的一个镜像仓库服务。ACR是一个提供企业级安全和全球镜像同步功能的镜像仓库。

执行以下命令，创建一个ACR仓库。本示例使用的镜像仓库名称为`k8scloudlabs`，请根据实际情况选择可用的镜像仓库名称。

    $ az acr create -g k8s-cloud-labs -n k8scloudlabs --sku Standard
    NAME          RESOURCE GROUP    LOCATION    SKU       LOGIN SERVER             CREATION DATE         ADMIN ENABLED
    ------------  ----------------  ----------  --------  -----------------------  --------------------  ---------------
    k8scloudlabs  k8s-cloud-labs    eastus      Standard  k8scloudlabs.azurecr.io  2018-12-02T11:59:09Z


    $az acr login -g k8s-cloud-labs -n k8scloudlabs

### 1.7 授权访问镜像仓库
ACR是一个带有安全认证机制的镜像仓库，AKS访问ACR下载镜像仓库前需要先进行授权。复制并执行下面命令，获取需要的信息，并执行授权。

    AKS_RESOURCE_GROUP=k8s-cloud-labs
    AKS_CLUSTER_NAME=k8s-cluster
    ACR_RESOURCE_GROUP=k8s-cloud-labs
    ACR_NAME=k8scloudlabs

    CLIENT_ID=$(az aks show -g $AKS_RESOURCE_GROUP -n $AKS_CLUSTER_NAME --query "servicePrincipalProfile.clientId" -o tsv)
    ACR_ID=$(az acr show -n $ACR_NAME -g $ACR_RESOURCE_GROUP --query "id" -o tsv)
    az role assignment create --assignee $CLIENT_ID --role Reader --scope $ACR_ID

> 提示！在Windows桌面中运行`az role`命令出错的情况下，可以将如下命令的输出拷贝至CMD中执行。

    $ echo az role assignment create --assignee $CLIENT_ID --role Reader --scope $ACR_ID

### 1.8 云端容器镜像构建
ACR不单止提供容器镜像的存取服务，ACR还提供基于云端的容器镜像构建服务。这意味着开发人员甚至不需要在本地开发环境安装Docker就可以进行容器镜像的构建。

执行以下命令，让Draft使用ACR进行容器镜像的构建。

    $ draft config set container-builder acrbuild
    $ draft config set registry  k8scloudlabs.azurecr.io
    $ draft config set resource-group-name k8s-cloud-labs

### 1.9 执行应用容器镜像构建
创建所需的命名空间。

    $ kubectl create ns lab03

一切配置就绪，下面将执行构建。执行命令`draft up`。Draft将把本地的代码提交给ACR，ACR将在Azure公有云上进行镜像的构建。
    
    $ cd japp
    $draft up
    Draft Up Started: 'japp': 01CXQGZKF88NDGGKGY4C93P58A
    japp: Building Docker Image: SUCCESS ⚓  (136.1632s)
    japp: Releasing Application: SUCCESS ⚓  (6.9090s)
    Inspect the logs with `draft logs 01CXQGZKF88NDGGKGY4C93P58A`

> 提示！如果构建过程出错，可以尝试执行命令`az login`及`az acr login`重新登陆Azure及ACR服务。

构建结束后，查看ACR镜像仓库的镜像列表，便可以看到相应的镜像已经成功生成。

    $az acr repository list -g k8s-cloud-labs -n k8scloudlabs
    Result
    --------
    japp

容器镜像构建完毕后，Draft还将调用Helm，将应用部署至Kubernetes集群中。通过Helm可以查看到应用部署的情况。

    $helm list
    NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
    calico-ostrich  1               Sun Dec  2 21:54:59 2018        DEPLOYED        japp-v0.1.0                     lab03

### 1.10 访问远程容器服务
当应用容器运行后，通过命令`draft up`可以将远程容器的端口映射到本地进行访问。
    
    $draft connect
    Connect to japp:4567 on localhost:59698
    ...内容省略...

通过命令`curl`访问Draft监听的端口，即可访问远程的容器应用。

    curl http://localhost:59698/greeting?name=Kubernetes

### 1.11 运行本地变更
当本地代码有变化，可以重新运行一次命令`draft up`，这样Draft将根据最新的代码变更进行镜像构建和部署。

## 附录一 实验资源回收

实验完毕后删除实验所创建的公有云资源。执行命令如下：

    $ az aks delete -g k8s-cloud-labs -n k8s-cluster
    $ az group delete -n k8s-clouds-labs

## 附录二 参考信息
- [Azure Kubernetes Service](https://azure.microsoft.com/en-us/services/kubernetes-service/)
- [Azure Container Registry](https://azure.microsoft.com/en-us/services/container-registry/)
- [Azure Monitor](https://docs.microsoft.com/en-us/azure/azure-monitor/overview)
- [Azure Cloud Shell](https://azure.microsoft.com/en-us/features/cloud-shell/)
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/)
- [Helm](https://helm.sh/)
- [Draft](https://github.com/Azure/draft)
