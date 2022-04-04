# 合理设置 Request 与 Limit

如何为容器配置 Request 与 Limit? 这是一个即常见又棘手的问题，这个根据服务类型，需求与场景的不同而不同，没有固定的答案，这里结合生产经验总结了一些最佳实践，可以作为参考。

## 所有容器都应该设置 request

request 的值并不是指给容器实际分配的资源大小，它仅仅是给调度器看的，调度器会 "观察" 每个节点可以用于分配的资源有多少，也知道每个节点已经被分配了多少资源。被分配资源的大小就是节点上所有 Pod 中定义的容器 request 之和，它可以计算出节点剩余多少资源可以被分配(可分配资源减去已分配的 request 之和)。如果发现节点剩余可分配资源大小比当前要被调度的 Pod 的 reuqest 还小，那么就不会考虑调度到这个节点，反之，才可能调度。所以，如果不配置 request，那么调度器就不能知道节点大概被分配了多少资源出去，调度器得不到准确信息，也就无法做出合理的调度决策，很容易造成调度不合理，有些节点可能很闲，而有些节点可能很忙，甚至 NotReady。

所以，建议是给所有容器都设置 request，让调度器感知节点有多少资源被分配了，以便做出合理的调度决策，让集群节点的资源能够被合理的分配使用，避免陷入资源分配不均导致一些意外发生。

## CPU request 与 limit 的一般性建议

* 如果不确定应用最佳的 CPU 限制，可以不设置 CPU limit，参考: [Understanding resource limits in kubernetes: cpu time](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b)。
* 如果要设置 CPU request，大多可以设置到不大于 1 核，除非是 CPU 密集型应用。

## 老是忘记设置怎么办?

有时候我们会忘记给部分容器设置 request 与 limit，其实我们可以使用 LimitRange 来设置 namespace 的默认 request 与 limit 值，同时它也可以用来限制最小和最大的 request 与 limit。
示例:

``` yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: test
spec:
  limits:
  - default:
      memory: 512Mi
	  cpu: 500m
    defaultRequest:
      memory: 256Mi
	  cpu: 100m
    type: Container
```

## 重要的线上应用改如何设置

节点资源不足时，会触发自动驱逐，将一些低优先级的 Pod 删除掉以释放资源让节点自愈。没有设置 request，limit 的 Pod 优先级最低，容易被驱逐；request 不等于 limit 的其次； request 等于 limit 的 Pod 优先级较高，不容易被驱逐。所以如果是重要的线上应用，不希望在节点故障时被驱逐导致线上业务受影响，就建议将 request 和 limit 设成一致。

## 怎样设置才能提高资源利用率?

如果给给你的应用设置较高的 request 值，而实际占用资源长期远小于它的 request 值，导致节点整体的资源利用率较低。当然这对时延非常敏感的业务除外，因为敏感的业务本身不期望节点利用率过高，影响网络包收发速度。所以对一些非核心，并且资源不长期占用的应用，可以适当减少 request 以提高资源利用率。

如果你的服务支持水平扩容，单副本的 request 值一般可以设置到不大于 1 核，CPU 密集型应用除外。比如 coredns，设置到 0.1 核就可以，即 100m。

## 尽量避免使用过大的 request 与 limit

如果你的服务使用单副本或者少量副本，给很大的 request 与 limit，让它分配到足够多的资源来支撑业务，那么某个副本故障对业务带来的影响可能就比较大，并且由于 request 较大，当集群内资源分配比较碎片化，如果这个 Pod 所在节点挂了，其它节点又没有一个有足够的剩余可分配资源能够满足这个 Pod 的 request 时，这个 Pod 就无法实现漂移，也就不能自愈，加重对业务的影响。

相反，建议尽量减小 request 与 limit，通过增加副本的方式来对你的服务支撑能力进行水平扩容，让你的系统更加灵活可靠。

## 避免测试 namespace 消耗过多资源影响生产业务

若生产集群有用于测试的 namespace，如果不加以限制，可能导致集群负载过高，从而影响生产业务。可以使用 ResourceQuota 来限制测试 namespace 的 request 与 limit 的总大小。
示例:

``` yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-test
  namespace: test
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```


## 参考资料

* [Understanding Kubernetes limits and requests by example](https://sysdig.com/blog/kubernetes-limits-requests/)
* [Understanding resource limits in kubernetes: cpu time](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b)
* [Understanding resource limits in kubernetes: memory](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-memory-6b41e9a955f9)
* [Kubernetes best practices: Resource requests and limits](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-resource-requests-and-limits)
* [Kubernetes 资源分配之 Request 和 Limit 解析](https://cloud.tencent.com/developer/article/1004976)