title: k8s in action
category: k8s
date: 2021-10-11  
tags: [k8s, docker]
toc: false
---

#### 1 简介

1, 虚拟机和容器的区别

每个虚拟机运行在自己的Linux内核上(即下图中Guest OS， 每个VM可以安装不同版本的linux系统)，而容器都是调用同一个宿主机内核(如果一个容器化的应用需要一个特定的内核版本，那么它可能不能再每台机器上工作,也不能将一个x86架构编译的应用容器化之后，又期望他在arm架构的机器上运行)。

![虚拟机vs容器](/img/vmvscontainer.jpg)

2, 创建一个镜像

- 创建一个名为`allen`的镜像
 - `docker build -t allen .` 

- 将该镜像推送到镜像仓库
 - `docker tag allen qinchaowhut/allen:v1`
 - `docker push qinchaowhut/allen:v1`


#### 2 Pods


#### 3 Services


#### 4 Volumns

#### 5 ReplicaSet
复制控制器

#### 6 Deployment

Deployment用于部署应用程序，并且用声明的方式升级应用程序。其中，Deployment由ReplicaSet(1:N)组成，并且由ReplicaSet来创建和管理Pod

![Deployment](/img/deployment.jpg)

##### 6.2 升级应用程序

1, Deployment的升级策略

a, RollingUpdate, 即滚动更新（默认的升级策略），该策略会渐进的删除旧的pod,同时创建新的pod, 是应用程序在整个升级过程中都处于可用状态。（注意：在升级过程中，pod的数量可以在期望的副本数量左右浮动，该上限和下限是可以设置的）

b, Recreate, 即一次性删除所有的Pod, 然后才创建新的Pod(`缺点`：存在服务中断的情况)

需要指出的是升级完成之后，旧的ReplicaSet仍然保留（用于回滚，即升级的逆过程）

在测试环境，滚动更新的yaml文件内容如下：
- [deployment-v1.yaml](https://github.com/Qinch/k8s-in-action-test/blob/master/deployment-v1.yaml)

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testqc 
spec:
  replicas: 3
  template:
    metadata:
      name: allen
      labels:
        app: testing
    spec:
      containers:
      - image: qinchaowhut/allen:v1
        name: nodejs
        imagePullPolicy: IfNotPresent
  selector:
    matchLabels:
      app: testing
```

- run `kubectl create -f deployment-v1yaml --record`

其中record参数用于记录历史版本号，在查看升级history，显示CHANGE-CAUSE.

![Deployment](/img/deploymentv1.jpeg)

- [svc-loadbalancer.yaml](https://github.com/Qinch/k8s-in-action-test/blob/master/svc-loadbalancer.yaml)

```bash
apiVersion: v1
kind: Service
metadata:
  name: allen-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: testing
```

- run `kubectl create -f svc-loadbalancer.yaml`
![svc](/img/svc-loadbalancer.jpeg)


- run `curl http://127.0.0.1:80`

![http response](/img/httpechov1.jpeg)

2, 触发升级

触发条件：只要修改deployment中定义的pod模板，k8s就会自动将实际的系统状态收敛为资源中定义的状态。

可以通过修改Deployment中pod模板的镜像，来触发升级。

![升级前后](/img/update.jpg) 

- 通过`kubectl set image deployment testqc nodejs=qinchaowhut/allen:v2` 来修改Deployment的pod模板内的镜像(其中deployment的name为testqc, nodejs为容器name, qinchaowhut/allen:v2为镜像版本)
![set image](/img/setallenv2.jpeg)

升级完成之后，发送curl请求:
![http response](/img/httpechov2.jpeg)

- 查看升级前后ReplicaSet
![升级前后](/img/updaters.jpeg) 
通过上图, 可以看出旧的rs仍然本保留

- kubectl rollout status deployment testqc 
 - 可以观察整个升级过程

3, 回滚

Deployment始终保持着升级后的版本历史记录，其中历史版本号会被保存在ReplicaSet中。
由于滚动升级成功之后，不会删除老版本的ReplicaSet，这使得可以回滚到任意一个历史版本(注意，如果手工删除ReplicaSet， 便会丢失Deployement的历史版本，而导致无法回滚)。

- `kubectl rollout undo deployment testqc`
 - 回滚到上一个版本

- `kubectl rollout history deployment testqc`
 - 查看升级历史

- `kubectl rollout undo deployment testqc --to-revision=1`
 - 回滚到第一个版本
 - ![revision1](/img/undoversion1.jpeg)

4, 控制滚动升级速率

在Deployment的滚动升级期间，有两个属性会决定一次替换多少个pod,即maxSurge和maxUnavalibale。（可以通过rollingUpdate的子属性来配置），maxSurge和maxUnavailable可以设置成百分数或者绝对值。

- maxSurge:表示除了Deployment中配置的期望副本数之外，最多允许超出的pod的数量。默认值为25%.(如果期望副本数设置为4，那么滚动升级过程中，不会运行超过5个pod).
`需要注意的是`：When converting a percentage to an absolute number, the number is rounded up.

- maxUnavailable:表示滚动升级过程中，相对于期望副本数，允许有多少个pod实例可以处于不可用状态，默认值25%.(如果期望副本数量为4，那么整个发布过程中，只能有一个pod处于不可用状态)
`需要注意的是`：When converting a percentage to an absolute number , the number is rouded down.


- 在我们测试环境中，Deployment设置的期望副本数量为3，所以maxSurge为1, maxUnavailable 是0, 则整个升级过程如下：
![滚动升级过程](/img/rollingupdate.jpg)


- 假设我们配置的期望副本数量仍然为3，但是maxSurge和maxUnavailable都是1，则整个发布过程如下(`需要注意的是`:maxUnavailable是相对于期望副本数而言的， 即maxUnavailable设置为1，但是整个更新过程中，不可用pod数量可以超过1个)：

![滚动升级过程](/img/rollingupdate2.jpg)

5, 暂停/恢复滚动升级

在某次版本发布过程中，可能我们并不想滚动升级所有的pod, 而是在滚动升级过程中，先升级一小部分pod, 然后暂停升级过程。通过查看这一小部分用户请求的处理情况，如果符合预期，就可以用新的pod替换所有旧的pod.(金丝雀发布：是一种可以将应用程序的出错版本和其影响到的用户的风险化为最小的计数。与其直接向每个用户发布新版本，不如用新版本替换一个或者一小部分的pod。)

在通过`kubectl set image`命令触发滚动更新之后，立马执行如下命令，暂停滚动更新：
- `kubectl rollout pause deploymnet testqc`

一旦确认新版本能够正常工作，就可以恢复滚动升级，用新版本pod替换所有旧版本的pod
- `kubectl rollout resume deployment testqc`

6, 阻止出错版本的滚动升级
minReadySeconds:指定新创建的pod至少要成功运行多久之后，才能将其视为可用。在pod尅用之前，滚动升级的过程不会继续。

需要指出的是，在滚动升级过程中，想要在一个确切的位置暂停滚动升级目前无法做到。但是可以用两个不同的Deployment并同时调整他们的pod数量，来进行金丝雀发布。

###### 6.3 Deployment相关命令行
- kubectl rollout status deployment $DEPLOYMENT_NAME
 - 查看部署状态

- kubectl get replicasets
 - 查看deployment创建的ReplicaSet



#### 附录

- 相关的yaml见 https://github.com/Qinch/k8s-in-action-test
- YAML文件可以包含多种资源定义，YAML manifest可以使用包含三个横杠(---)的行来分隔多个对象。


#### 参考资料

《k8s in action》
