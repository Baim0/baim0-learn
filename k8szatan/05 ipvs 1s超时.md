> 原文链接：[](https://zhuanlan.zhihu.com/p/630430941)

某个客户在Kubernetes中部署了大规模的业务，因此选用IPVS作为`kube-proxy`的负载均衡转发模式。以追求更高的转发性能和更新规则的速度。

但是，IPVS有一个存在了很久的连接重用问题，当客户发布服务，因为存在大量TCP短连接，客户端出现了`No route to host`报错，业务服务出现故障。

这是一个存在已久的issue：[kubernetes#81775](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/kubernetes/issues/81775)。我们今天就来简单分析一下并介绍解决方案。

### 复现

故障必须经过复现才有排查的手段，我这里介绍一个简单的复现方法。

首先，准备一个测试用的Kubernetes集群，kube-proxy使用IPVS模式，并且为了复现故障，把内核参数`net.ipv4.vs.conn_reuse_mode`设为`0`（在一些低版本的`kube-proxy`或低版本内核中，你会发现它默认为`0`）：

```bash
echo 0 > /proc/sys/net/ipv4/vs/conn_reuse_mode
```

部署一个简单的HTTP服务，包含10个副本，并定义一个`LoadBalancer`类型的Service以模拟负载均衡的场景：

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/ucloud-load-balancer-vserver-protocol: "TCP"
    service.beta.kubernetes.io/ucloud-load-balancer-type: "outer"
  name: httpgraceful
spec:
  type: LoadBalancer
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: httpgraceful
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpgraceful
  labels:
    app: httpgraceful
spec:
  replicas: 10
  selector:
    matchLabels:
      app: httpgraceful
  template:
    metadata:
      labels:
        app: httpgraceful
    spec:
      containers:
      - name: http
        image: uhub.service.ucloud.cn/wxyz/httpgraceful:1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        readinessProbe:
          exec:
            command:
            - cat
            - /tmp/healthz
          initialDelaySeconds: 3
          periodSeconds: 3
```

等待Pod运行起来后，查看Service地址（这里假设为`106.75.10.10`）：

```bash
$ kubectl get svc
NAME           TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
httpgraceful   LoadBalancer   172.17.43.192   106.75.10.10   80:31984/TCP   48s
```

确认该Service可以访问：

```bash
$ curl http://106.75.10.10
hello world
version:1.0.0
```

现在，使用`ab`命令，创建大量对该Service的短连接：

```bash
ab -c 10 -n 2000000 http://106.75.10.10/
```

同时，将HTTP服务的副本数改为5，以模拟服务发布的场景：

```bash
kubectl scale deploy httpgraceful --replicas=5
```

很快，你就可以看到`ab`命令输出错误：

```text
Benchmarking 106.75.10.10 (be patient)
apr_socket_recv: No route to host (113)
Total of 5251 requests completed
```

`No route to host`出现了！复现出了客户的错误。

这时候，登录机器，查看IPVS规则，你会发现该Service对应的RS出现了很多权重为0的项：

```bash
$ ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  106.75.35.59:80 rr
  -> 10.9.19.40:8080              Masq    1      5          2
  -> 10.9.26.56:8080              Masq    0      1          0
  -> 10.9.82.126:8080             Masq    0      1          1
  -> 10.9.99.18:8080              Masq    0      1          0
  -> 10.9.127.64:8080             Masq    0      2          0
  -> 10.9.136.110:8080            Masq    1      6          1
  -> 10.9.157.207:8080            Masq    1      5          0
  -> 10.9.161.247:8080            Masq    1      5          0
  -> 10.9.187.95:8080             Masq    1      5          0
```

没错，这就是导致这次问题的根因，我们会在下面进行分析。

### 原因分析

当业务进行缩容或者滚动迭代过程中，一定有旧的Pod被销毁。`No route to host`的直接原因是，IPVS把访问流量转发到了一个被销毁的Pod上面。

具体地说，当一个Pod被销毁后，`kube-controller-manager`中的`Endpoint Controller`会随之把对应的Pod Ip从`Endpoints`对象中摘除。kube-proxy随即通过informer获取到旧Pod Ip被摘除的事件，相应地更新ipvs转发规则：

- 先将旧的IP对应的RS权重置为0，防止新的连接调度到这个IP上。
- 等存量连接归零后(ActiveConn+InactiveConn=0)，再彻底摘除掉这个旧IP。

从上面`ipvsadm -Ln`的结果也可以看到，几个权重为0的RS都还有连接，所以没有被摘除。

kube-proxy不直接摘除RS是为了实现Pod的优雅退出，以让Pod在处理完所有请求之后再退出。

但我们碰到的问题实际上是，权重被置为0后，依然有新连接被调度到旧IP上。这样存量连接计数永远不会归零，旧RS永远无法被摘除。而由于旧的IP实际已经不存在，就出现了`No route to host`报错。只有把流量撤走一段时间后，连接计数终于为0，旧RS被摘除。

这一切的罪魁祸首是一个内核参数：`net.ipv4.vs.conn_reuse_mode`。

众所周知，在大量TCP短连接的场景下，会有很多连接处于`TIME_WAIT`状态，为了减少资源占用，内核会重用这些连接的端口。

而如果`net.ipv4.vs.conn_reuse_mode`为0，并且客户端的源端口被复用了，那么IPVS会将流量直接转发到之前RS，而绕过了正常的负载均衡调度。这样即使RS的权重为0，也可能会有流量被转发上去。

在大量TCP短连接时，端口复用的频率会非常高，这样即使RS的权重被设为了0，还是会有部分流量打上来，而其背后的Pod已经被删除了（优雅退出有超时时间，超过之后Kubernetes会强制删除Pod），就导致了流量被转发到了一个不存在的IP上，出现`No route to host`错误。

那么其实只要将`net.ipv4.vs.conn_reuse_mode`设为1，强制让复用的连接也经过负载均衡转发不就可以了？

可惜，在`5.9`之前的内核版本中，`net.ipv4.vs.conn_reuse_mode`设为1并使用了`conntrack`时，如果出现了连接复用，IPVS会 DROP 掉第一个 SYN 包，导致 SYN 的重传，有 1s 延迟。而 Kube-proxy 在 IPVS 模式下，使用了 iptables 进行`MASQUERADE`，也正好开启了`net.ipv4.vs.conntrack`。

所以：

- 如果将`net.ipv4.vs.conn_reuse_mode`设为0，当出现大量短TCP连接时，会出现部分流量被转发到销毁Pod的问题。
- 如果将`net.ipv4.vs.conn_reuse_mode`设为1，当出现大量短TCP连接时，会有部分连接多出了1s的延迟的问题。

### 解决

也就是说，在以前的内核版本中，理想的情况是：

- `net.ipv4.vs.conn_reuse_mode`设为1，强制复用连接走负载均衡。
- `net.ipv4.vs.conntrack`设为0，防止IPVS对复用连接进行DROP SYNC操作。

但是，很可惜，这实现不了。**因此，如果你的内核版本低于`5.9`，并且存在大量TCP短连接场景，不建议使用IPVS，建议替换为iptables模式。**

`5.9`的一个[patch](https://link.zhihu.com/?target=http%3A//patchwork.ozlabs.org/project/netfilter-devel/patch/20200701151719.4751-1-ja%40ssi.bg)修复了在启用`conntrack`时的1s延迟问题，因此，如果内核版本大于等于`5.9`，将`net.ipv4.vs.conn_reuse_mode`设为1是安全的，保持这个内核参数为1即可解决问题。

事实上，新的kube-proxy会自动根据你的内核版本来判断是否要修改这个内核参数：

```go
const connReuseFixedKernelVersion = "5.9"

if kernelVersion.LessThan(version.MustParseGeneric(connReuseMinSupportedKernelVersion)) {
    klog.ErrorS(nil, "Can't set sysctl, kernel version doesn't satisfy minimum version requirements", "sysctl", sysctlConnReuse, "minimumKernelVersion", connReuseMinSupportedKernelVersion)
} else if kernelVersion.AtLeast(version.MustParseGeneric(connReuseFixedKernelVersion)) {
    // https://github.com/kubernetes/kubernetes/issues/93297
    klog.V(2).InfoS("Left as-is", "sysctl", sysctlConnReuse)
} else {
    // Set the connection reuse mode
    if err := utilproxy.EnsureSysctl(sysctl, sysctlConnReuse, 0); err != nil {
        return nil, err
    }
}
```

而因为我们提供给客户的内核版本经过了魔改，虽然其已经包含了这个patch，但是版本小于`5.9`，所以需要在kube-proxy启动之后强制将`net.ipv4.vs.conn_reuse_mode`设为1。

如果使用systemd部署kube-proxy，可以用`ExecStartPost`来完成内核参数的修改：

```text
[Service]
...
ExecStartPost=/sbin/sysctl -w net.ipv4.vs.conn_reuse_mode=1
```

如果用Pod部署kube-proxy，可以使用PostStart完成：

```yaml
lifecycle:
  postStart:
    exec:
      command:
      - /sbin/sysctl
      - -w
      - net.ipv4.vs.conn_reuse_mode=1
```



https://mp.weixin.qq.com/s/H5zrelsEVtAmy96ObhuokA