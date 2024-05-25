---
title: 04 coredns相关配置
date: 2023-04-30 13:52:07
tags:
- kubernetes杂谈
categories:
- kubernetes杂谈
---

## 场景说明

使用`dnsPolicy: ClusterFirst`策略。示例配置如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: alpine
  namespace: default
spec:
  containers:
  - image: alpine
    command:
      - sleep
      - "10000"
    imagePullPolicy: Always
    name: alpine
  dnsPolicy: ClusterFirst
```

关于**dnsPolicy**配置和场景说明，请参见[DNS原理和配置说明](https://help.aliyun.com/document_detail/188179.htm#task-1985778)。

## CoreDNS的默认配置

在命名空间kube-system下，ACK集群有一个CoreDNS配置项（有关如何查看配置项的具体步骤，请参见[管理配置项](https://help.aliyun.com/document_detail/86769.htm#section-hly-ioi-ozy)）。CoreDNS会基于该配置项启用和配置插件。不同CoreDNS版本的配置项有略微差异，修改配置前请仔细阅读[CoreDNS官方文档](https://coredns.io/plugins/)。以下是一个1.6.2版本CoreDNS默认采用的配置文件：



```bash
  Corefile: |
    .:53 {
        errors
        log
        health {
           lameduck 15s
        }
        ready
        kubernetes {{.ClusterDomain}} in-addr.arpa ip6.arpa {
          pods verified
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
              prefer_udp
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

**说明** 配置文件中`ClusterDomain`代指集群创建过程中填写的集群本地域名，默认值为`cluster.local`。

| 参数                       | 描述                                                         |
| :------------------------- | :----------------------------------------------------------- |
| **errors**                 | 错误信息到标准输出。                                         |
| **health**                 | CoreDNS自身健康状态报告，默认监听端口8080，一般用来做健康检查。您可以通过`http://localhost:8080/health`获取健康状态。 |
| **ready**                  | CoreDNS插件状态报告，默认监听端口8181，一般用来做可读性检查。可以通过`http://localhost:8181/ready`获取可读状态。当所有插件都运行后，ready状态为200。 |
| **kubernetes**             | CoreDNS Kubernetes插件，提供集群内服务解析能力。             |
| **prometheus**             | CoreDNS自身metrics数据接口。可以通过`http://localhost:9153/metrics`获取prometheus格式的监控数据。 |
| **forward**（或**proxy**） | 将域名查询请求转到预定义的DNS服务器。默认配置中，当域名不在Kubernetes域时，将请求转发到预定义的解析器（/etc/resolv.conf）中。默认使用宿主机的/etc/resolv.conf配置。 |
| **cache**                  | DNS缓存。                                                    |
| **loop**                   | 环路检测，如果检测到环路，则停止CoreDNS。                    |
| **reload**                 | 允许自动重新加载已更改的Corefile。编辑ConfigMap配置后，请等待两分钟以使更改生效。 |
| **loadbalance**            | 循环DNS负载均衡器，可以在答案中随机A、AAAA、MX记录的顺序。   |

## CoreDNS的扩展配置

针对以下不同场景，您可以扩展CoreDNS的配置：

- 场景一：开启日志服务

  如果需将CoreDNS每次域名解析的日志打印出来，您可以开启Log插件，在Corefile里加上log。示例配置如下：

  ```bash
    Corefile: |
      .:53 {
          errors
          log
          health {
             lameduck 15s
          }
          ready
          kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
          }
          prometheus :9153
          forward . /etc/resolv.conf {
                prefer_udp
          }
          cache 30
          loop
          reload
          loadbalance
      }
  ```

- 场景二：特定域名使用自定义DNS服务器

  如果example.com类型后缀的域名需要经过自建DNS服务器（IP为10.10.0.10）进行解析的话，您可为域名配置一个单独的服务块。示例配置如下：

  ```bash
  example.com:53 {
    errors
    cache 30
    forward . 10.10.0.10
    prefer_udp
  }
  ```

  完整配置如下：

  ```bash
    Corefile: |
      .:53 {
          errors
          health {
             lameduck 15s
          }
          ready
          kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
          }
          prometheus :9153
          forward . /etc/resolv.conf {
            prefer_udp
          }
          cache 30
          loop
          reload
          loadbalance
      }
      example.com:53 {
          errors
          cache 30
          forward . 10.10.0.10
          prefer_udp
      }
  ```

- 场景三：外部域名完全使用自建DNS服务器

  如果您需要使用的自建DNS服务的域名没有统一的域名后缀，您可以选择所有集群外部域名都使用自建DNS服务器（此时需要您将自建的DNS服务不能解析的域名转发到阿里云DNS，禁止直接更改集群ECS上的/etc/resolv.conf文件）。例如，您自建的DNS服务器IP为10.10.0.10和10.10.0.20，可以更改**forward**参数进行配置。示例配置如下：

  ```bash
    Corefile: |
      .:53 {
          errors
          health {
             lameduck 15s
          }
          ready
          kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
          }
          prometheus :9153
          forward . 10.10.0.10 10.10.0.20{
            prefer_udp
          }
          cache 30
          loop
          reload
          loadbalance
      }
  ```

- 场景四：自定义Hosts

  如果您需要为特定域名指定hosts，如为www.example.com指定IP为127.0.0.1，可以使用Hosts插件来配置。示例配置如下：

  ```go
    Corefile: |
      .:53 {
          errors
          health {
             lameduck 15s
          }
          ready
          
          hosts {
            127.0.0.1 www.example.com
            fallthrough
          }
        
          kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
          }
          prometheus :9153
          forward . /etc/resolv.conf {
            prefer_udp
          }
          cache 30
          loop
          reload
          loadbalance
      }
  ```

  **重要** 请配置**fallthrough**，否则会造成非定制Hosts域名解析失败。

- 场景五：集群外部访问集群内服务

  如果您希望运行在集群ECS上的进程能够访问到集群内的服务，虽然可以通过将ECS的/etc/resolv.conf文件内`nameserver`配置为集群kube-dns的ClusterIP地址来达到目的，但不推荐您直接更改ECS的/etc/resolv.conf文件的方式来达到任何目的。

  内网场景下，您可以将集群内的服务通过内网SLB进行暴露，然后在[云解析PrivateZone控制台](https://dns.console.aliyun.com/#/dns/domainList)通过添加A记录到该SLB的内网IP进行解析。具体操作，请参见[添加解析记录](https://help.aliyun.com/document_detail/64635.html)。

- 场景六：统一域名访问服务或是在集群内对域名的做CNAME解析

  您可以实现在公网、内网和集群内部通过统一域名foo.example.com访问您的服务，原理如下：

  - 集群内的服务`foo.default.svc.cluster.local`通过公网SLB进行了暴露，且有域名`foo.example.com`解析到该公网SLB的IP。

  - 集群内服务`foo.default.svc.cluster.local`通过内网SLB进行了暴露，且通过[云解析PrivateZone](https://dns.console.aliyun.com/#/dns/domainList)在VPC内网中将`foo.example.com`解析到该内网SLB的IP。具体步骤，请参见上述[场景四：自定义Hosts](https://help.aliyun.com/document_detail/380963.html?spm=a2c4g.750001.0.i1#section-6jf-fgj-j2f)。

  - 在集群内部，您可以通过Rewrite插件将

    ```go
    foo.example.com
    ```

     

    CNAME到

    ```go
    foo.default.svc.cluster.local
    ```

    。示例配置如下：

    ```go
      Corefile: |
        .:53 {
            errors
            health {
               lameduck 15s
            }
            ready
            
            rewrite stop {
              name regex foo.example.com foo.default.svc.cluster.local
              answer name foo.default.svc.cluster.local foo.example.com 
            }
    
            kubernetes cluster.local in-addr.arpa ip6.arpa {
              pods insecure
              fallthrough in-addr.arpa ip6.arpa
              ttl 30
            }
            prometheus :9153
            forward . /etc/resolv.conf {
              prefer_udp
            }
            cache 30
            loop
            reload
            loadbalance
        }
    ```

- 场景七：禁止CoreDNS对IPv6类型的AAAA记录查询返回

  当业务容器不需要AAAA记录类型时，可以在CoreDNS中将AAAA记录类型拦截，返回空（NODATA），以减少不必要的网络通信。示例配置如下：

  ```yaml
    Corefile: |
      .:53 {
          errors
          health {
             lameduck 15s
          }
          #新增以下一行Template插件，其它数据请保持不变。
          template IN AAAA .
      
      }
  ```