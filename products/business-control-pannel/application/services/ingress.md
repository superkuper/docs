## 简介
Ingress 是允许访问到集群内 Service 的规则的集合，您可以通过配置转发规则，实现不同 URL 可以访问到集群内不同的 Service。
为了使 Ingress 资源正常工作，集群必须运行 Ingress-controller。TKE 服务在集群内默认启用了基于腾讯云负载均衡器实现的 `l7-lb-controller`，支持 HTTP、HTTPS，同时也支持 nginx-ingress 类型，您可以根据您的业务需要选择不同的 Ingress 类型。

## 注意事项<span id="annotations"></span>   
- 确保您的容器业务不和 CVM 业务共用一个 CLB。
- 不支持您在 CLB 控制台操作 TKE 管理的 CLB 的监听器、转发路径、证书和后端绑定的服务器，您的更改会被 TKE 自动覆盖。
- 使用已有的 CLB 时：
  - 只能使用通过 CLB 控制台创建的负载均衡器，不支持复用由 TKE 自动创建的 CLB。
  - 不支持多个 Ingress 复用 CLB。
  - 不支持 Ingress 和 Service 共用 CLB。
  - 删除 Ingress 后，复用 CLB 绑定的后端云服务器需要自行解绑，同时会保留一个 `tag tke-clusterId: cls-xxxx`，需自行清理。

## Ingress 控制台操作指引
                                                                                                                                    
### 创建 Ingress

1. 登录TKEStack，切换到业务管理控制台，选择左侧导航栏中的【应用管理】。
2. 选择需要创建Ingress的业务下相应的命名空间，展开工作负载列表，进入Ingress管理页面.
3. 单击【新建】，进入“新建Ingress”页面。目前仅支持yaml方式创建，复制yaml文件到输入框。
4. 单击【创建Ingress】，完成创建。
 
## Kubectl 操作 Ingress 指引


### YAML 示例<span id="YAMLSample"></span>
```Yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: qcloud ## 可选值：qcloud（CLB类型ingress）, nginx（nginx-ingress）
	## kubernetes.io/ingress.existLbId： lb-xxxxxxxx	  ##指定使用已有负载均衡器创建公网/内网访问的Ingress
    ## kubernetes.io/ingress.subnetId: subnet-xxxxxxxx  ##若是创建CLB类型内网ingress需指定该条annotation
  name: my-ingress
  namespace: default
spec:
  rules:
  - host: localhost
    http:
      paths:
      - backend:
          serviceName: non-service
          servicePort: 65535
        path: /
```
- kind：标识 Ingress 资源类型。
- metadata：Ingress 的名称、Label 等基本信息。
- metadata.annotations：Ingress 的额外说明，可通过该参数设置腾讯云 TKE 的额外增强能力。
- spec.rules：Ingress 的转发规则，配置该规则可实现简单路由服务、基于域名的简单扇出路由、简单路由默认域名、配置安全的路由服务等。

#### annotations: 使用已有负载均衡器创建公网/内网访问的 Ingress

如果您已有的应用型 CLB 为空闲状态，需要提供给 TKE 创建的 Ingress 使用，或期望在集群内使用相同的 CLB ，您可以通过以下 annotations 进行设置：
>?请了解 [注意事项](#annotations) 后开始使用。
>
```Yaml
metadata:
  annotations:
    kubernetes.io/ingress.existLbId： lb-6swtxxxx
```

#### annotations: 创建 CLB 类型内网 Ingress

如果您需要使用内网负载均衡，可以通过以下 annotations 进行设置：
```Yaml
metadata:
  annotations:
    kubernetes.io/ingress.subnetId: subnet-xxxxxxxx
```

#### 说明事项
如果您使用的是 **IP 带宽包** 账号，在创建公网访问方式的服务时需要指定以下两个 annotations 项：
- `kubernetes.io/ingress.internetChargeType` 公网带宽计费方式，可选值有：
 - TRAFFIC_POSTPAID_BY_HOUR（按使用流量计费）
 - BANDWIDTH_POSTPAID_BY_HOUR（按带宽计费）
- `kubernetes.io/ingress.internetMaxBandwidthOut` 带宽上限，范围：[1,2000] Mbps。
例如：
```Yaml
metadata:
  annotations:
    kubernetes.io/ingress.internetChargeType: TRAFFIC_POSTPAID_BY_HOUR
    kubernetes.io/ingress.internetMaxBandwidthOut: "10"
```
关于 **IP 带宽包** 的更多详细信息，欢迎查看文档 [共享带宽包产品类别](https://cloud.tencent.com/document/product/684/15246)。

### 创建 Ingress

1. 参考 [YAML 示例](#YAMLSample)，准备 Ingress YAML 文件。
2. 安装 Kubectl，并连接集群。操作详情请参考 [通过 Kubectl 连接集群](https://cloud.tencent.com/document/product/457/8438)。
3. 执行以下命令，创建 Ingress YAML 文件。
```shell
kubectl create -f Ingress YAML 文件名称
```
例如，创建一个文件名为 my-ingress.yaml 的 Ingress YAML 文件，则执行以下命令：
```shell
kubectl create -f my-ingress.yaml
```
4. 执行以下命令，验证创建是否成功。
```shell
kubectl get ingress
```
返回类似以下信息，即表示创建成功。
```
NAME          HOSTS       ADDRESS   PORTS     AGE
clb-ingress   localhost             80        21s
```

### 更新 Ingress

#### 方法一

执行以下命令，更新 Ingress。
```
kubectl edit  ingress/[name]
```

#### 方法二

1. 手动删除旧的 Ingress。
2. 执行以下命令，重新创建 Ingress。
```
kubectl create/apply
```