---
layout: post  
title: "Rancher RKE与F5 BIGIP集成部署实践"  
subtitle:  通过F5实现Kubernetes容器化业务的蓝绿发布
date: 2020-01-31  
author: "李鸿宇"  
header-img: "img/post-bg-rwd.jpg"  
header-mask: "0.1"  
catalog: True  
tags:  

- F5 CIS
- Rancher RKE
- Kubernetes    
---





# 1、Rancher RKE与F5 BIGIP部署架构图

![image-20200616154739209](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200616154739209.png)



# 2、配置F5 BIGIP VE软件负载均衡

## 2.1 配置F5 BIGIP VE管理地址

| 默认角色        | 默认用户名 | 默认密码 |
| --------------- | ---------- | -------- |
| F5 BIGIP VE系统 | root       | default  |
| F5 BIGIP VE UI  | admin      | admin    |

```
# 进入F5 tmos操作环境
tmsh
# 关闭dhcp功能
modify sys global-settings mgmt-dhcp disabled
# [可选项]默认的管理登陆端口为8443，此步骤可省略
modify sys httpd ssl-port 8443
# 删除默认的管理IP
delete sys management-ip 192.168.1.245/24
# 创建管理IP为阿里云为F5分配的地址
create sys management-ip 172.17.206.198/20
# 创建F5所在网段的网关地址
create sys management-route default gateway 172.17.207.253
```

![image-20200613222641750](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200613222641750.png)



## 2.2 查看F5 BIGIP的license

从F5 BIGIP 13x版本以后，对Kubernetes Flanenl Cluster Mode支持的SDN license已经包含在了Local Traffic Manager, VE license里面了。但如果使用需要边界网关协议（BGP）的网络模式（例如Calico），则还必须使用包含Routing Bundle的BIG-IP system license。

```
show sys license
```

![image-20200616110836743](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200616110836743.png)

![image-20200616111824138](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200616111824138.png)



## 2.3 配置F5 BIGIP对接Flannel网络的Vxlan相关配置

```
# 修改允许各种协议和服务连接这个self-IP。
modify net self interface01 allow-service all
```

![image-20200612215828672](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200612215828672.png)




```
# 配置vxlan profile
create net tunnels vxlan fl-vxlan port 8472 flooding-type none
```

![image-20200611205750041](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200611205750041.png)



```
# 配置vxlan VTEP
create net tunnels tunnel flannel_vxlan key 1 profile fl-vxlan local-address 172.17.206.198
```

![image-20200611194609753](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200611194609753.png)



```
# 配置F5和Pod通信的接口地址。
create net self interface02 address 10.42.100.3/16 allow-service all vlan flannel_vxlan
```

![image-20200614232227235](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200614232227235.png)




```
# 保存F5配置信息
save sys config
# 查看vxlan VTEP的Mac地址，用于配置F5 BIGIP伪节点。
show net tunnels tunnel flannel_vxlan all-properties
```

![image-20200615020841539](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615020841539.png)



## 2.4 配置F5 BIGIP partition分区

由于F5 CIS无法针对默认的common的partition分区进行操作，所以需要提前创建partition分区。

```
# 创建名字为Rancher的partition分区
create auth partition Rancher
# [可选]删除名字为Rancher的partition分区
delete auth partition Rancher
```

![image-20200611213847234](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200611213847234.png)

```
# 查看partiton分区列表
list auth partition
```

![image-20200615112442945](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615112442945.png)

![image-20200611215626567](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200611215626567.png)



#  3、在Kubernetes集群部署F5 CIS控制器

## 3.1 配置F5 BIGIP访问凭证

将F5 BIGIP登陆信息的用户名、密码和登陆地址，通过base64加密并设置成kubernetes的secrets


```
echo -n 'admin' | base64 
echo -n 'Rancher@123' | base64 
```

```yaml
tee > f5-bigip-secret.yml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: bigip-credentials
  namespace: kube-system
type: Opaque
data:
  username: YWRtaW4=
  password: UmFuY2hlckAxMjM=
EOF
```

```
kubectl apply -f f5-bigip-secret.yml
```



## 3.2 配置F5 BIGIP集群伪节点

将F5 BIGIP设备配置成kubernetes集群Flannel网络里面的伪节点

```yaml
tee > f5-kctlr-bigip-node.yaml << EOF
apiVersion: v1
kind: Node
metadata:
  name: bigip
  annotations:
    # Provide the MAC address of the BIG-IP VXLAN tunnel
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"00:16:3e:10:1b:d2"}'
    flannel.alpha.coreos.com/backend-type: "vxlan"
    flannel.alpha.coreos.com/kube-subnet-manager: "true"
    # Provide the IP address you assigned as the BIG-IP VTEP
    flannel.alpha.coreos.com/public-ip: 172.17.206.198
  labels:
    # 为f5伪节点设置如下标签,使Rancher忽略对此节点的错误显示。
    cattle.rancher.io/node-status: ignore
spec:
  # Define the flannel subnet you want to assign to the BIG-IP device.
  # Be sure this subnet does not collide with any other Nodes' subnets.
  podCIDR: 10.42.100.0/24
EOF
```

```
kubectl apply -f f5-kctlr-bigip-node.yaml
```

```
kubectl get nodes
```

![image-20200613172236843](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200613172236843.png)

在Rancher UI看到的主机列表如下。

![image-20200616111236838](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200616111236838.png)



## 3.3 配置F5 CSI控制器RBAC权限

```
kubectl create serviceaccount bigip-ctlr -n kube-system
```

```yaml
tee > f5-k8s-rbac.yaml << EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: bigip-ctlr-clusterrole
rules:
- apiGroups: ["", "apps", "extensions"]
  resources: ["nodes", "services", "endpoints", "namespaces", "ingresses", "secrets", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["", "apps", "extensions"]
  resources: ["configmaps", "events", "ingresses/status"]
  verbs: ["get", "list", "watch", "update", "create", "patch"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: bigip-ctlr-clusterrole-binding
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: bigip-ctlr-clusterrole
subjects:
- kind: ServiceAccount
  name: bigip-ctlr
  namespace: kube-system
EOF
```

```
kubectl apply -f f5-k8s-rbac.yaml
```



## 3.4 部署F5 CIS控制器

```yaml
vim k8s-bigip-ctlr-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-bigip-ctlr
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-bigip-ctlr
  template:
    metadata:
      name: k8s-bigip-ctlr
      labels:
        app: k8s-bigip-ctlr
    spec:
      serviceAccountName: bigip-ctlr
      containers:
        - name: k8s-bigip-ctlr
          # 由于F5 CIS控制器2.0采用AS3的命令声明模式，所以此次依然采用1.x版本控制器镜像。
          image: "f5networks/k8s-bigip-ctlr:1.14.0"
          env:
            - name: BIGIP_USERNAME
              valueFrom:
                secretKeyRef:
                  name: bigip-credentials
                  key: username
            - name: BIGIP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: bigip-credentials
                  key: password
          command: ["/app/bin/k8s-bigip-ctlr"]
          args:
            - "--bigip-username=$(BIGIP_USERNAME)"
            - "--bigip-password=$(BIGIP_PASSWORD)"
            - "--bigip-url=https://172.17.206.198:8443"
            - "--bigip-partition=Rancher"
            - "--namespace=default"
            - "--pool-member-type=cluster" 
            - "--log-level=INFO"
            - "--flannel-name=/Common/flannel_vxlan"
            - "--insecure=true"
```

```
kubectl apply -f k8s-bigip-ctlr-deployment.yaml
```



# 4、配置工作负载的L4负载均衡

## 4.1 创建工作负载Deployment和服务发现Service

通过Rancher UI创建nginx-l4工作负载，Pod副本数为5个。

![image-20200614233035566](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200614233035566.png)

通过Rancher UI查看nginx工作负载的服务发现service。

![image-20200614233317068](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200614233317068.png)



## 4.2 创建L4负载均衡的配置映射ConfigMap

通过configmap为工作负载创建L4负载均衡

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: f5-nginx-l4
  namespace: default
  labels:
    f5type: virtual-server
data:
  schema: "f5schemadb://bigip-virtual-server_v0.1.7.json"
  data: |
    {
      "virtualServer": {
        "backend": {
          "servicePort": 80,
          "serviceName": "nginx-l4",
          "healthMonitors": [{
          "interval": 30,
          "protocol": "http",
          "send": "HEAD / HTTP/1.0\r\n\r\n",
          "timeout": 91
          }]
        },
        "frontend": {
          "virtualAddress": {
            "port": 8000,
            "bindAddr": "172.17.206.198"
          },
          "partition": "Rancher",
          "balance": "round-robin",
          "mode": "tcp"
        }
      }
    }

```

通过Rancher UI创建F5 L4的负载均衡的方法如下；

![image-20200614234320465](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200614234320465.png)

![image-20200614234451365](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200614234451365.png)



## 4.3 验证L4负载均衡的配置结果

在F5 BIGIP管理界面上查看L4负载均衡配置的下发情况。

![image-20200615002639647](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615002639647.png)

![image-20200615011603865](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615011603865.png)

通过浏览器访问http://172.17.206.198:8000查看应用的访问情况

![image-20200615003152532](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615003152532.png)



# 5、配置工作负载的L7负载均衡

## 5.1 创建工作负载Deployment和服务发现Service

通过Rancher UI创建nginx-l7工作负载，Pod副本数为5个。

![image-20200615004608556](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615004608556.png)

通过Rancher UI查看nginx工作负载的服务发现service。

![image-20200615004716160](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615004716160.png)



## 5.2 创建L7负载均衡Ingress

通过ingress为工作负载创建L7负载均衡

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: f5-nginx-l7
  namespace: default
  annotations:
    virtual-server.f5.com/partition: "Rancher"
    virtual-server.f5.com/ip: 172.17.206.198
    virtual-server.f5.com/http-port: "80"
    virtual-server.f5.com/ssl-redirect: "false"
    virtual-server.f5.com/balance: "round-robin"
    virtual-server.f5.com/health: |
      [
        {
          "path": "f5.rancher.com/",
          "send": "HTTP GET /",
          "interval": 5,
          "timeout":  10
        }
      ]
spec:
  rules:
  - host: f5.rancher.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-l7
          servicePort: 80
```

通过Rancher UI创建F5 L7的负载均衡的方法如下；

![image-20200615083751786](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615083751786.png)

![image-20200615011108608](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615011108608.png)

![image-20200615083841427](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615083841427.png)



## 5.3 验证L7负载均衡的配置结果

在F5 BIGIP管理界面上查看L7负载均衡配置的下发情况。

![image-20200615011400870](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615011400870.png)

相比L4负载均衡，L7负载均衡增加了应用层策略的支持，可以灵活实现蓝绿、灰度等应用发布支持。

![image-20200615015333201](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615015333201.png)

![image-20200615011641660](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615011641660.png)

通过浏览器访问http://f5.rancher.com查看应用的访问情况

![image-20200615012815660](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615012815660.png)

# 6、配置应用蓝绿发布

## 6.1 创建测试蓝绿发布的工作负载

创建一个内容为绿色的nginx工作负载，名字叫nginx-green，pod副本数为2，同时创建一个内容为蓝色的nginx工作负载，名字叫nginx-blue，pod副本数为2。

![image-20200615210027902](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615210027902.png)

查看nginx-green和nginx-blue的服务发现service

![image-20200615210618098](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200615210618098.png)



## 6.2 创建测试应用的蓝绿发布策略

通过ingress为工作负载创建应用蓝绿发布

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: f5-green-blue
  namespace: default
  annotations:
    virtual-server.f5.com/partition: "Rancher"
    virtual-server.f5.com/ip: 172.17.206.198
    virtual-server.f5.com/http-port: "80"
    virtual-server.f5.com/ssl-redirect: "false"
    virtual-server.f5.com/balance: "round-robin"
    virtual-server.f5.com/health: |
      [
        {
          "path": "green.rancher.com/",
          "send": "HTTP GET /",
          "interval": 5,
          "timeout":  10
        },
        {
          "path": "blue.rancher.com/",
          "send": "HTTP GET /",
          "interval": 5,
          "timeout":  10
        }
      ]
    kubernetes.io/ingress.class: "f5"
spec:
  rules:
  - host: green.rancher.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-green
          servicePort: 80
  - host: blue.rancher.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-blue
          servicePort: 80
```

通过Rancher UI创建ingress转发策略。

![image-20200616112524638](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200616112524638.png)

![image-20200616112224355](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200616112224355.png)

![image-20200616112316965](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200616112316965.png)

![image-20200616112439674](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200616112439674.png)

## 6.3 验证应用蓝绿发布的配置结果

在F5 BIGIP管理界面上查看应用蓝绿发布的运行和策略下发情况。

![image-20200616122554561](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200616122554561.png)

![image-20200616122648028](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200616122648028.png)

通过浏览器访问F5 BIGIP的地址http://172.17.206.198，同时使用chome modheader插件修改请求的header信息为**host:blue.rancher.com**，查看应用的访问情况。

![image-20200616151200957](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200616151200957.png)

![image-20200616151554149](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200616151554149.png)

通过浏览器访问F5 BIGIP的地址http://172.17.206.198，同时使用chome modheader插件修改请求的header信息为**host:green.rancher.com**，查看应用的访问情况。

![image-20200616151655908](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200616151655908.png)

![image-20200616151723184](https://cloudwalking.oss-cn-beijing.aliyuncs.com/md-images/image-20200616151723184.png)