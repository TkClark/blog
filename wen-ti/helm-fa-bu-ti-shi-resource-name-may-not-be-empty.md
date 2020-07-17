---
description: '2020-07-17 14:36:16'
---

# Helm 发布提示: resource name may not be empty

由于架构需要，重新建立了非通用模板的charts，在使用 `helm upgrade` 的时候提示如下错误:

```text
helm upgrade --wait --force '--timeout=60' --debug --install <releaseName> <charts>

Error: UPGRADE FAILED: Upgrade --force successfully deleted the previous release, but encountered 1 error(s) and cannot continue: resource name may not be empty
```

`resource name may not be empty` \(资源名称不可以为空\)   
这个错误提示让我们排查的时候绕了很多弯路

看到这个错误的第一眼以为是渲染错误导致渲染后的yaml中`metadata.name`为空，所以将命令改成了 `helm upgrade --dry-run` 来看是否是模板渲染问题，返回结果如下：

```text
# Source: default-go-template/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: <serviceName>
  labels:
    app: <serviceName>
  annotations:
    prometheus.io/port: http-10000
    prometheus.io/scheme: http
    prometheus.io/scrape: "true"
spec:
  type: "ClusterIP"
  ports:
  - name: grpc-9000
    port: 9000
    targetPort: 9000
  - name: http-10000
    port: 10000
    targetPort: 10000
  selector:
    app: <serviceName>
Release "v1-<serviceName>" has been upgraded. Happy Helming!
```

可以看到渲染出来的模板并没有什么问题，经过尝试直接使用 `kubectl apply -f`  导入渲染出来的yaml也是正常的。

然后，我们就开始怀疑是否是因为我们看到的`--dry-run` 出来的结果跟`helm`实际发给`kube-apiserver`的数据不一样？

为了排除这个问题，我们先排除了`helm`自身渲染的问题，将`templates/serivce.yaml`中的`metadata.name`直接写死。`templates/serivce.yaml`如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: serviceName
  labels:
    app: serviceName
  annotations:
    {{- if (eq .Values.prometheus.scrape true)}}
    {{- range $key, $val := .Values.prometheus}}
    prometheus.io/{{ $key }}: {{ $val | quote}}
    {{- end}}
    {{- end}}
spec:
  {{- if eq .Values.grpcProxy false }}
  type: {{ default "ClusterIP" .Values.service.type | quote }}
  {{- else }}
  clusterIP: "None"
  {{- end }}
  ports:
  {{- range $index, $port := .Values.service.ports }}
  - name: {{ $port.name }}
    port: {{ $port.port }}
    targetPort: {{ $port.targetPort }}
  {{- end }}
  selector:
    app: "{{ template "serviceName" . }}"
{{- end }}
```

再次执行`helm upgrade` 命令进行尝试，发现还是返回同样的错误，也就是说我们的`templates`模板并没有问题。

于是检查`helm`是否存在对应的`release` ，通过命令 `helm list releaseName` 发现并没有生成对应的`release` 。回忆想起来之前集群应该是有发不过该服务的。

于是，想起来`helm v2` 的历史数据都是存在 `kube-system` 命名空间下的`configmap` 内。

{% hint style="info" %}
helm v3的历史记录存在 kube-system 下的secret中.
{% endhint %}

```yaml
[root@k8s-jenkins templates]# kubectl get cm -n kube-system|grep v1-antman-banner
v1-antman-banner.v1 
```

发现果然有这个发布记录，删除后重新进行`helm`发布正常。

> 思考

由于`helm upgrade` 发布的时候会对比上一次的发布记录，只`patch` 与上一次发布不同的配置参数。

因此，大部分`helm` 发布异常问题可以通过`helm delete releaseName --purge` 完成。`--purge` 是完全删除`release` 相关数据。

本次异常就是因为`release` 没有删除干净导致的。

