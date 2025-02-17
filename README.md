
Pod的优雅上下线依赖k8s的监控检查机制，以及 Pod lifecycle Hooks，通过这些kubernetes的机制，配合服务发现的流量管理机制，实现业务的优雅上下线。


## 基础概念


### Pod 健康检查


Pod的健康状态由两类探针来检查：LivenessProbe和ReadinessProbe。


1. livenessProbe(存活探针)
	* 表明容器是否正在运行。
	* 如果存活探测失败，则 kubelet 会杀死容器，并且容器将受到其 重启策略的影响。
	* 如果容器不提供存活探针，则默认状态为 Success。
2. readinessProbe(就绪探针)
	* 表明容器是否可以正常接受请求。
	* 如果就绪探测失败，端点控制器将从与 Pod 匹配的所有 Service 的端点中删除该 Pod 的 IP 地址。
	* 初始延迟之前的就绪状态默认为 Failure。
	* 如果容器不提供就绪探针，则默认状态为 Success。
3. StartupProbe（这个 1\.16 版本增加的）
	* 如果三个探针同时存在，先执行 StartupProbe 探针，其他两个探针将会被暂时禁用，直到 pod 满足 StartupProbe 探针配置的条件，其他 2 个探针启动，如果不满足按照规则重启容器。


两种探针的区别
总的来说 ReadinessProbe 和 LivenessProbe 是使用相同探测的方式，只是探测后对 Pod 的处置方式不同：


* ReadinessProbe： 当检测失败后，将 Pod 的 IP:Port 从对应 Service 关联的 EndPoint 地址列表中删除。
* LivenessProbe： 当检测失败后将杀死容器，并根据 Pod 的重启策略来决定作出对应的措施。


例子



```
livenessProbe:
  failureThreshold: 3
  initialDelaySeconds: 600
  periodSeconds: 5
  successThreshold: 1
  tcpSocket:
    port: 7800
  timeoutSeconds: 1
  
readinessProbe:
  failureThreshold: 3
  httpGet:
    path: /ready
    port: 7800
    scheme: HTTP
  periodSeconds: 5
  successThreshold: 1
  timeoutSeconds: 3

```

### Pod lifecycle Hooks


![file](https://img2024.cnblogs.com/other/3300446/202501/3300446-20250110101931011-1816775902.png)


1. PostStart
PostStart hook在容器启动的时候运行，但并不保证该hook一定会比容器指定的ENTRYPOINT命令先运行。就是说PostStart和ENTRYPOINT都会在容器启动后运行，至于谁先运行，谁先结束，并不一定，是随机的。如果容器启动的时候，PostStart没有成功，容器不会处于running状态。
2. PreStop
PreStop会在kubelet给Pod发送TERM信号之前执行。一般API Server会给kubelet发送结束Pod的信号，或者Pod的liveness/startup探针失败，或其它原因导致Pod失败，kubelet会尝试发送TERM信号给Pod里主进程。如果PreStop存在，kubelet则会优先启动PreStop，待PreStop结束之后再发送TERM信号给Pod。但从API Server将Pod标记为Terminating状态开始，整个Pod停止时间不能超过terminationGracePeriodSeconds所设置的时间，如果超过，kubelet需要发送KILL信号给Pod所有的进程。


例子：



```
lifecycle:
      postStart:
        exec:
          command:
          - /bin/sh
          - -c
          - ./online.sh
      preStop:
        exec:
          command:
          - /bin/sh
          - -c
          - ./offline.sh

```

## 实现


![file](https://img2024.cnblogs.com/other/3300446/202501/3300446-20250110101931235-190897315.png)


优雅上线的实现：


* Pod启动后，在服务治理框架（sidecar）中开始注册服务，服务注册中心收到服务治理框架的成功注册消息。
* 同时服务注册中心管理系统会watch应用的Service的Endpoint，当Pod的IP出现时，服务治理框架会任务当前Pod状态ready可以上线服务了，当然Pod的IP出现在Endpoint列表中，本质也是就绪探针成功，这取决于服务治理框架的实现，一般服务治理框架需要提供探针健康检查的接口。
* 服务注册中心会向上下游通知当前节点上线。


Pod下线：


* Pod的IP从Endpoint列表中消失，服务治理框架会通知应用，应用需要下线服务。
* Pod pre stop hook执行后，服务治理框架会通知服务注册中心，服务注册中心会广播此节点下线，停止路由新调用，同时pre stop hook还可以确保服务内已经获得的请求都处理完毕，Pod才可以被回收。


[更多文章](https://github.com)


 本博客参考[西部世界官网](https://www.xbsj9.com)。转载请注明出处！
