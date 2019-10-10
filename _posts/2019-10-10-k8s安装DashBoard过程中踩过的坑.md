---
layout:     post
title:      kubeadm部署k8s后安装DashBoard过程中踩过的坑
date:       2019-10-9
author:     JC
header-img: img/kubernetes.jpg
catalog: false
tags:
    - kubernetes
---

### 镜像被墙

```
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1 k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
```
**把镜像copy到其他节点**

docker save k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1 -o dashboard.tar

scp dashboard.tar [IP Address]:/root/

**在其他node节点导入镜像(这里必须！所有的节点都要有镜像！！不然会报pull err)**

docker load -i dashboard.tar

docker image ls|grep k8s.gcr.io/kubernetes-dashboard-amd64

### 连不上apiserver

错误日志:

```
2018/07/28 08:51:12 Starting overwatch
2018/07/28 08:51:12 Using in-cluster config to connect to apiserver
2018/07/28 08:51:12 Using service account token for csrf signing
2018/07/28 08:51:12 No request provided. Skipping authorization
2018/07/28 08:51:42 Error while initializing connection to Kubernetes apiserver. This most likely means that the cluster is misconfigured (e.g., it has invalid apiserver certificates or service accounts configuration) or the --apiserver-host param points to a server that does not exist. Reason: Get https://10.96.0.1:443/version: dial tcp 10.96.0.1:443: i/o timeout
Refer to our FAQ and wiki pages for more information: https://github.com/kubernetes/dashboard/wiki/FAQ
```

问题原因:

官方的yaml配置文件中有这么一段:
```
Comment the following tolerations if Dashboard must not be deployed on master
tolerations:
- key: node-role.kubernetes.io/master
effect: NoSchedule
```
使得pod不会分配到到master节点, 并且kubeadm部署的apiserver中启用的验证方式为Node和RBAC, 且关闭了insecure-port, 我猜测可能是这个原因导致连接不上apiServer, 即使是手动修改也不行--apiserver-host参数也不行

解决方法:

注释掉这三行：
```
tolerations:
- key: node-role.kubernetes.io/master
effect: NoSchedule
```
添加nodeName(设置后dashboard将会在这个节点上运行)

![](/img/k8s/dashboard.jpg)

最后执行

kubectl delete -f kubernetes-dashboard.yaml

删除pod, 再执行kubectl apply -f kubernetes-dashboard.yaml即可


### 更新https证书有效期，解决浏览器报错 NET::ERR_CERT_INVSALID


```
mkdir key && cd key

openssl genrsa -out dashboard.key 2048 

openssl req -new -out dashboard.csr -key dashboard.key -subj '/CN=[IP Address]'

openssl x509 -req -in dashboard.csr -signkey dashboard.key -out dashboard.crt 

kubectl delete secret kubernetes-dashboard-certs -n kube-system

kubectl create secret generic kubernetes-dashboard-certs --from-file=dashboard.key --from-file=dashboard.crt -n kube-system  #新的证书

kubectl delete pod kubernetes-dashboard-xxxxxxxxx -n kube-system    #重启服务
```

### 获取admin的token登录

在kube-system名称空间创建一个名为dashboard-admin的ServiceAccount

将dashboard-admin这个ServiceAccount和cluster-admin绑定

```
cat > dashboard-admin.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dashboard-admin
subjects:
  - kind: ServiceAccount
    name: dashboard-admin
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```
```
[root@master ~]# kubectl apply -f dashboard-admin.yaml
serviceaccount/dashboard-admin created
clusterrolebinding.rbac.authorization.k8s.io/dashboard-admin created
```

```
[root@master ~]# kubectl describe secret dashboard-admin-token-twrjp -n kube-system
Name:         dashboard-admin-token-twrjp
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: 4c2caffd-37fe-49ae-a443-d0b3e345da07

Type:  kubernetes.io/service-account-token

Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tdHdyanAiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNGMyY2FmZmQtMzdmZS00OWFlLWE0NDMtZDBiM2UzNDVkYTA3Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.OEaz9gm3ZB3jVxc4sp4peD4XwO-zPg5on4yV0u4UKpKa6mQcNF0qJ5f1mMO6AztZUPLSgsd46tu1p1ZOEh3FFCdlw7fRT2DSZsPFHP-4ahlJcEVD1egBHnQlvdoEo1Rhxkji157QjegCIu09TPe8m-2cd5Mlw_5rlOnMcJyJuGvyUIIqUi00AHXilEZ1kiI939HhKfqzJtnXwgNUEhmKcNHboGPt7yoKEaMHio-uHQoyQVUXSPXUWhvFtCq1La25oDJBV5SMO5cq3PqqDnCaPMNDLslMh8lv5mYzMvdrz-47hdhuMvc1-pR7LbD2J8hI0XxeAVWt9c4oATaQtj8vLA
ca.crt:     1025 bytes
namespace:  11 bytes
```



