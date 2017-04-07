# ops-kubops

## 在PCF环境中部署和管理kubernetes集群

##### 前序

Bosh是新一代分布式部署平台，在其上Bosh通过CPI将我们需要部署的软件自动分发部署到IaaS平台上。目前bosh已经来到2.0，最近官方重写了bosh客户端，基本能做到本地打包到处复用，云上部署，云上调试。Bosh提供了通用的stemcell将操作系统封装，本质上还是希望运维人员能在其之上快速部署基础组件，淡化OS概念。</br>

目前已经有很多厂商在其上做了自动化封装，最近比较火的kubo也是如此，kubo是google和pivotal工程师一起协作开源的自动化项目,目的在于将kubernetes直接部署到谷歌云的Bosh环境里，后续可能在其上封装kubernetes service broker，不过目前kube有个孵化项目[service-catalog](https://github.com/kubernetes-incubator/service-catalog)，意义在于将应用和服务分离，形成一个open service broker api业界标准。

#### 部署kubo之前，看下官方的架构：
![kubo deployment](https://github.com/pivotal-cf-experimental/kubo-deployment/raw/master/docs/images/kubo-network.png)

从官方解释看，此项目还只是一个合作孵化项目，route-api的功能还没有释放出来，只是在其上注册了tcp router接管了kubernetes api访问终端，由于内部使用TLS证书，所以不同通过kubernetes api终端url直接访问dashboard ui，对于kubectl的操作也相对复杂一些。至于一些周边功能，如日志，监控等还需要自己集成，就不要直接封装到kubo里了，另外打出release，组合部署。

#### 部署步骤：

1.准备uaac客户端，将routing api client注册到uaa认证中心,赋予权限能对route api进行操作
```
uaac target uaa.example.com
uaac token client get admin -s MyAdminPassword
uaac client add routing_api_client --authorities "routing.routes.write,routing.routes.read,routing.router_groups.read" --authorized_grant_type "client_credentials"
```

2.平台开启tcp route支持，在DNS Server中，将kubernetes.yourdomain指向tcp router

3.将制作好的bosh release封装成Ops Manager Zip安装介质，这里可能涉及不止kubo的release</br>
* 目前项目的开源地址
```
https://github.com/pivotal-cf-experimental/kubo-release
```

4.将介质上传到PCF上，准备部署
![ops manager-upload](https://github.com/wdxxs2z/ops-kubops/blob/master/ops/o.JPG)

5.配置kubernetes</br>
* kubernetes api和kubelet 的密码手动填写，这里不使用平台自动生成策略。</br>
* 生成TLS访问证书：填写泛域名 如 *.yourDomain，这里指定了kubernetes.yourDomain</br>
* 填写刚才uaac注册的uaa 的 client id和密码 ：routing_api_client ，your client password</br>
* 如果平台有自己的私有镜像库也可填写自己的镜像库</br>
* 其它可配置参数目前没有释放，如果需要可以自己定制模板，将参数释放出来</br>
* 支持多AZ部署</br>
![ops manager-opts](https://github.com/wdxxs2z/ops-kubops/blob/master/ops/3.JPG)

6.kubernetes的主要组件说明</br>
* etcd:主要存储和管理flannel vxlan网络的配置信息，储存kubernetes的元数据</br>
* kubernetes: v1.4.6的版本，在定制时，可选择不同版本定制</br>
* docker: v1.11版本，如有特殊情况，可以自行定制</br>
* nginx: 对kubernetes-dashboard负载</br>
* kubernetes-api-route-registrar: 将kubernetes api的终端kubernetes.yourDomain:8443注册到tcp router上</br>
* * metron agent：未来可将此组件纳入，接入ELK日志系统</br>
* * grafana: 统一监控界面</br>

7.添加errand支持kubernetes system-namespace service</br>
系统服务在此errand统一部署，如ui,dns,heapster,influxdb等，目前这个版本需要自己封装errand，不支持errand模式(已经提交了issue),所以在官方的基础上，会稍微改变一下部署策略，添加对其的支持。</br>

8.配置组件资源列表</br>
![ops manager-resource](https://github.com/wdxxs2z/ops-kubops/blob/master/ops/4.JPG)

9.使用kubectl操作kubernetes集群</br>
查看pods
```
kubectl --kubeconfig=/var/vcap/jobs/kubeconfig/config/kubeconfig --all-namespaces=true get po 
```
查看所有实例
```
kubectl --kubeconfig=/var/vcap/jobs/kubeconfig/config/kubeconfig describe svc --all-namespaces=true
```
部署一个nginx模板[nginx](https://github.com/pivotal-cf-experimental/kubo-deployment/blob/4f324f2ea7e10b615d842d9a68abc09e1b37f8ee/ci/specs/nginx.yml)
```
kubectl --kubeconfig=/var/vcap/jobs/kubeconfig/config/kubeconfig create -f /tmp/nginx.yml
```
通过nodeip:31000访问dashboard
![kubo-ui](https://github.com/wdxxs2z/ops-kubops/blob/master/ops/7.JPG)

10.部署grafana监控服务,此块服务也可直接放到errand里执行</br>
```
./kubectl --kubeconfig=/var/vcap/jobs/kubeconfig/config/kcreate -f /tmp/grafana.yml
```
* 查看grafana详情</br>
![kubo-gs](https://github.com/wdxxs2z/ops-kubops/blob/master/ops/8.JPG)</br>
* 进入节点查看监控指标.</br>
![kubo-ga](https://github.com/wdxxs2z/ops-kubops/blob/master/ops/9.JPG)</br>
* 集群指标.</br>
![kubo-gb](https://github.com/wdxxs2z/ops-kubops/blob/master/ops/10.JPG)</br>
* pod指标.</br>
![kubo-gc](https://github.com/wdxxs2z/ops-kubops/blob/master/ops/11.JPG)</br>

#### 关于路由注册 

* 目前官方已完成路由发现部分的设计，包括tcp和http都涵盖在内，具体设计如下(个人理解)：
![kubo-gy](https://github.com/wdxxs2z/ops-kubops/blob/master/ops/ks-router.JPG)</br>

* TCP：可以看出tcp是直接通过**route api**联合**uaa**将k8s内被打了标签**tcp-route-sync**的services注册进tcp route,用户可以直接用tcp-route的IP：service_lable_port访问。

* HTTP：http是通过**nats**将k8s内被打了标签**http-route-sync**的services注册进gorouter，用户可以直接用service_lable_hostname加上cf的域名进行访问。

* CTX：持续对k8s所有namespaces内的services进行扫描

* 举个栗子，以nginx为实验对象，以暴露http的方式给cf
```
apiVersion: v1
kind: Service
metadata:
  labels:
    name: nginx
    http-route-sync: nginx  #http 则指定host名 -> gorouter
    #tcp-route-sync: 34567 #tcp 则指定端口 -> tcp router
  name: nginx
spec:
  ports:
    - port: 80
  selector:
    app: nginx
  type: NodePort
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: hub.c.163.com/library/nginx:latest
        ports:
        - containerPort: 80
```
* 执行创建</br>
./kubectl --kubeconfig=..... create -f /tmp/nginx.yml </br>
* 查看dashboard,观察nginx服务的标签</br>
![kubo-gl](https://github.com/wdxxs2z/ops-kubops/blob/master/ops/12.png)</br>
* 查看路由注册组件,观察日志</br>
![kubo-gk](https://github.com/wdxxs2z/ops-kubops/blob/master/ops/13.png)</br>
* 直接访问应用域名,一切正常</br>
![kubo-gh](https://github.com/wdxxs2z/ops-kubops/blob/master/ops/14.png)</br>
* 查看gorouter路由表</br>
![kubo-gf](https://github.com/wdxxs2z/ops-kubops/blob/master/ops/15.JPG)</br>

##### 后续
从目前的计划看，官方会持续集成日志，对接cf的metron agent。
