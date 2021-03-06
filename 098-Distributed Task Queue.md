# 示例: 分布式任务队列 Celery, RabbitMQ和Flower
`译者：White` `校对：无`

## 介绍
Celery是基于分布式消息传递的异步任务队列。它可以用来创建一些执行单元（例如，一个任务），这些任务可以同步，异步的在一个或者多个工作节点执行。

Celery基于Python实现。

因为Celery基于消息传递，需要一些叫作消息代理的中间件（他们用来在发送者和接受者之间处理传递的消息）。RabbitMQ是一种和Celery联合使用的消息中间件。

下面的示例将向你展示，如何使用Kubernetes来建立一个基于Celery作为任务队列，RabbitMQ作为消息代理的分布式任务队列系统。同时，还要展示如何建立一个基于Flower的任务监控前端。

## 目标
### 在例子的最后，我们可以看到：

* 3个pods:
    * 一个Celery任务队列
    * 一个RabbitMQ消息代理
    * Flower前端
* 一个提供访问消息代理的服务
* 可以传递给工作节点的级别Celery任务

## 先决条件

你应该已经拥有一个Kubernetes集群。要完成大部分的例子，确保Kubernetes创建一个以上的节点（例如，通过设置`NUM_MINIONS`环境变量为2或者更多）。

### 第一步：启动RabbitMQ服务

Celery任务队列需要连接到RabbitMQ代理。RabbitMQ队列最终会出现在一个独立的pod上，但是，由于pod是短暂存在的，需要一个服务来透明的路由请求到RabbitMQ。

使用这个文件`examples/celery-rabbitmq/rabbitmq-service.yaml`

```json
apiVersion: v1
kind: Service
metadata:
  labels:
    name: rabbitmq
  name: rabbitmq-service
spec:
  ports:
  - port: 5672
  selector:
    app: taskQueue
    component: rabbitmq
```
这样运行一个服务：
```
$ kubectl create -f examples/celery-rabbitmq/rabbitmq-service.yaml
```
这个服务允许其他pods连接到rabbitmq。对于它们可以使用5672端口，服务也会将流量路由到容器（也通过5672端口）。

### 第二步：启动RabbitMQ

RabbitMQ代理可以通过这个文件启动`examples/celery-rabbitmq/rabbitmq-controller.yaml`:

```json
apiVersion: v1
kind: ReplicationController
metadata:
  labels:
    name: rabbitmq
  name: rabbitmq-controller
spec:
  replicas: 1
  selector:
    component: rabbitmq
  template:
    metadata:
      labels:
        app: taskQueue
        component: rabbitmq
    spec:
      containers:
      - image: rabbitmq
        name: rabbitmq
        ports:
        - containerPort: 5672
        resources:
          limits:
            cpu: 100m
```
运行`$ kubectl create -f examples/celery-rabbitmq/rabbitmq-controller.yaml `这个命令来创建副本控制器，确保当一个RabbitMQ实例运行时，一个pod已经存在。

请注意创建这个pod需要一些时间来拉取一个docker镜像。在这个例子中，这些操作也适用于其他pods。

### 第三步：启动Celery

通过`$ kubectl create -f examples/celery-rabbitmq/celery-controller.yaml`来创建一个celery worker,文件如下：

```json
apiVersion: v1
kind: ReplicationController
metadata:
  labels:
    name: celery
  name: celery-controller
spec:
  replicas: 1
  selector:
    component: celery
  template:
    metadata:
      labels:
        app: taskQueue
        component: celery
    spec:
      containers:
      - image: endocode/celery-app-add
        name: celery
        ports:
        - containerPort: 5672
        resources:
          limits:
            cpu: 100m
```

一些地方需要指出...:

像RabbitMQ控制器，需要确保总是有一个pod运行着Celery worker实例。

celery-app-add这个镜像是对标准的Celery Docker镜像的扩展。Dockerfile如下：

```
FROM library/celery

ADD celery_conf.py /data/celery_conf.py
ADD run_tasks.py /data/run_tasks.py
ADD run.sh /usr/local/bin/run.sh

ENV C_FORCE_ROOT 1

CMD ["/bin/bash", "/usr/local/bin/run.sh"]
```
celery_conf.py文件包含了一个简单的celery加法运算任务。最后一行启动Celery worker。

注意：`ENV C_FORCE_ROOT 1`用来确保Celery以root用户身份运行，不提倡在生产环境中采用这种方式。

celery_conf.py文件的内容如下：

```python
import os

from celery import Celery

# Get Kubernetes-provided address of the broker service
broker_service_host = os.environ.get('RABBITMQ_SERVICE_SERVICE_HOST')

app = Celery('tasks', broker='amqp://guest@%s//' % broker_service_host, backend='amqp')

@app.task
def add(x, y):
    return x + y
```

假设你已经熟悉Celery的运行机制，除了这个`os.environ.get('RABBITMQ_SERVICE_SERVICE_HOST')`部分。第一步创建的RabbitMQ服务的IP地址已经在环境变量中设置。Kubernetes会自动对所有定义了名为RabbitMQ服务的应用程序标签的容器提供环境变量（这个例子中叫“任务队列”）。上面那段Python代码，会在pod运行时自动填充代理地址。

第二个python脚本(run_tasks.py) ，会以五秒为间隔，周期性的执行add随机数字的任务。

现在的问题是，你怎么看发生了什么？

### 第四步：弄一个前端

Flower是一个基于web的工具，用来监控和管理Celery集群。通过连接到一个包含Celery的节点，你可以实时看到所有worker以及他们的任务的工作情况。

首先，通过` $ kubectl create -f examples/celery-rabbitmq/flower-service.yaml.`命令来启动一个Flower服务。这个服务定义如下：

```
apiVersion: v1
kind: Service
metadata:
  labels:
    name: flower
  name: flower-service
spec:
  ports:
  - port: 5555
  selector:
    app: taskQueue
    component: flower
  type: LoadBalancer
```

它会被标记为一个外部负载均衡器。然而，在许多平台上，必须添加一个明确的防火墙规则，打开5555端口。GCE上可以这么操作：
```
 $ gcloud compute firewall-rules create --allow=tcp:5555 --target-tags=kubernetes-minion kubernetes-minion-5555
```
请记住在运行完这个例子后删除这条规则（on GCE: `$ gcloud compute firewall-rules delete kubernetes-minion-5555`）。

运行下面命令来启动pods,` $ kubectl create -f examples/celery-rabbitmq/flower-controller.yaml`。这个控制器是这么定义的：
```
apiVersion: v1
kind: ReplicationController
metadata:
  labels:
    name: flower
  name: flower-controller
spec:
  replicas: 1
  selector:
    component: flower
  template:
    metadata:
      labels:
        app: taskQueue
        component: flower
    spec:
      containers:
      - image: endocode/flower
        name: flower
        resources:
          limits:
            cpu: 100m
```

这将创建一个新的pod,这个pod安装了Flower，服务节点会暴露5555端口（Flower的默认端口）。这个镜像使用下面命令启动Flower:

```
flower --broker=amqp://guest:guest@${RABBITMQ_SERVICE_SERVICE_HOST:localhost}:5672//
```

同样，它使用Kubernetes提供的环境变量来获取RabbitMQ服务的IP地址。

一旦所有的pods启动并且运行，运行`kubectl get pods`命令会显示下面内容：

```
NAME                                           READY     REASON       RESTARTS   AGE
celery-controller-wqkz1                        1/1       Running      0          8m
flower-controller-7bglc                        1/1       Running      0          7m
rabbitmq-controller-5eb2l                      1/1       Running      0          13m
```

`kubectl get service flower-service`命令会帮助你获取Flower服务的外部IP地址。

```
NAME             LABELS        SELECTOR                         IP(S)            PORT(S)
flower-service   name=flower   app=taskQueue,component=flower   10.0.44.166      5555/TCP
                                                                162.222.181.180
```

在你的网页浏览器中输入正确的flower服务地址和5555端口，（在这个例子中，[http://162.222.181.180:5555](http://162.222.181.180:5555/)）。如果你点击叫作“任务”的标签，你应该会看到一个不断增长的名为"celery_conf.add"的任务表单，它显示了`run_tasks.py`脚本的调度情况。
