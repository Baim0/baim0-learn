---
title: 03 如何删除终止状态的名称空间 
date: 2023-04-22 21:25:07
tags:
- kubernetes杂谈
categories:
- kubernetes杂谈
---



尝试删除Kubernetes命名空间后，长时间停留在终止状态。本文介绍如何解决命名空间处于终止状态的问题。

## 问题现象

尝试删除Kubernetes命名空间后，长时间停留在终止状态。

```csharp
kubectl delete ns <namespacename>
Error from server (Conflict): Operation cannot be fulfilled on namespaces "<namespacename>": The system is ensuring all content is removed from this namespace. Upon completion, this namespace will automatically be purged by the system. 

kubectl describe ns <namespacename>
Name: <namespacename> 
Labels: <none> 
Annotations: kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Namespace","metadata":{"annotations":{},"name":"<namespacename>","namespace":""}} 

Status: Terminating 
```

## 可能原因

通常是因为从集群中删除的这些命名空间下存在资源。

## 解决方案

删除命名空间的finalizers。

该选项将会快速清除处于终止状态的命名空间，但可能会导致属于该命名空间的资源留在集群中，因为无法自动删除它们。在finalizers数组为空并且状态为终止之后，Kubernetes将删除命名空间。

1. 打开Shell终端，为您的Kubernetes集群创建一个反向代理。

   ```bash
   kubectl proxy
   ```

   输出示例如下：

   ```bash
   Starting to serve on 127.0.0.1:8001
   ```

2. 打开一个新的Shell终端，通过定义环境变量来连接到Kubernetes集群，使用curl测试连接性和授权。

   ```bash
   export TOKEN=$(kubectl describe secret $(kubectl get secrets | grep default | cut -f1 -d ' ') | grep -E '^token' | cut -f2 -d':' | tr -d '\t')
   curl http://localhost:8001/api/v1/namespaces --header "Authorization: Bearer $TOKEN" --insecure
   ```

3. 获取命名空间定义的内容，以命名空间istio-system为例。

   ```sql
   kubectl get namespace istio-system -o json > istio-system.json
   ```

4. 将finalizers数组置为空，并重新保存文件。

   ```json
       "spec": {
           "finalizers": [
           ]
       },
   ```

5. 执行以下命令去除finalizers，以命名空间istio-system为例。

   ```bash
   curl -X PUT --data-binary @istio-system.json http://localhost:8001/api/v1/namespaces/istio-system/finalize -H "Content-Type: application/json" --header "Authorization: Bearer $TOKEN" --insecure
   ```