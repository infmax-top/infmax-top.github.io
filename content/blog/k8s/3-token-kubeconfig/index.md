---
title: 'k8s通过token创建受限权限的kubeconfig'
date: 2024-06-11T22:53:02+08:00
description: ""
tags: ["云原生", "k8s", "网络", "分布式调度"]
categories: ["分布式系统", "k8s"]
draft: false
---

在生产上，我们往往需要循序最小化权限控制原则，避免权限过度开放导致集群被误操作损坏或者影响到同集群其他用户资源等。 比如，我们希望部门A的运维人员仅可以操作`namespace A`下的资源。 而kubectl默认配备的config文件通常都是绑定在`cluster-admin`角色下，权限非常高。因此，我们需要重新生成权限受控的kubeconfig，来分发给不同角色的人员。 

> TODO 关于kubeconfig的工作原理解读参见另一篇文章， 这里不再赘述

如果是裸k8s集群,  我们拥有k8s CA的根证书和私钥，那么我们可以k8s CA通过签发一个子证书，该证书的OU对应关联的用户具有一定的权限控制，在kubeconfig中通过set-credentials 来配置子证书和私钥,  会用在客户端的TLS认证和服务端的身份鉴定。 这种方式在我们搭建k8s集群系列中，为kube-scheduler和kubelet等都使用过

但是，往往我们都是在云环境下直接使用托管的k8s集群，不具备签发证书的能力。所以，我们可以采用Token的方式来控制权限。 K8S为ServiceAccount采用了`JWT Token`方式的权限认证方式,  指定serviceAccount的pod会在启动后自动挂在该token文件，详情参见pod中的serviceAccount是如何生效的 。 

本文重点详述如何通过token来生成一个首先权限的kubeconfig

## 1、Token认证模式下的kubeconfig
在k8s中，token文件(Secret类型) 通常包含一个 `JWT（JSON Web Token）`，用于`ServiceAccount`的认证。`ca.crt` 是一个证书，它代表了集群的根证书或证书颁发机构（CA）的证书，用于验证 JWT 令牌的签名。
`ca.crt` 的全称是 `Certificate Authority Certificate` ，它包含了一个公共密钥，用于验证由该 CA 签发的证书（如ServiceAccount的 JWT 令牌）的签名。 `ca.crt` 通常与 `service-account-token Secret` 结合使用，以确保服务账户与 API 服务器之间的通信是安全的。当我们使用 kubectl 与 Kubernetes API 服务器通信时， token 选项用于提供 JWT 令牌，而 kubeconfig 选项通常用于提供包含集群配置的文件，其中包括 ca.crt 证书。

kubeconfig 通常包含以下部分：
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <base64-encoded-ca.crt>
    server: https://<kubernetes-apiserver-url>
  name: <cluster-name>
contexts:
- context:
    cluster: <cluster-name>
    user: <user-name>
  name: <context-name>
current-context: <context-name>
kind: Config
preferences: {}
users:
- name: <user-name>
  user:
    token: <service-account-token>
```

在这个配置中，
- `certificate-authority-data` 是 ca.crt 的 Base64 编码版本， ca.crt 的作用是验证与 API 服务器通信时的 TLS 连接的证书，确保连接是安全的，防止中间人攻击。在 Kubernetes 集群中，ca.crt 通常用于验证 API 服务器的证书，而服务账户的 JWT 令牌用于认证服务账户的身份。
- `server` 是 API 服务器的 URL
- `token` 是ServiceAccount的 JWT 令牌。

## 2、JWT Token认证

`JWT（JSON Web Token）` 是一种开放标准（RFC 7519），用于在各方之间安全地传输信息作为一个 JSON 对象。这个信息可以被验证和信任，因为它是数字签名的。JWT 通常用于认证和授权，特别是在 RESTful API 中。
JWT 结构分为三个部分，由点（.）分隔：
- `Header`：包含令牌的类型（通常为 JWT）和签名算法（如 HS256 或 RS256）。
- `Payload`：包含声明（claims）的 JSON 对象，声明可以是关于发行者、用户信息、过期时间等的数据。声明分为三种类型：
	- Registered Claims（注册声明）：预定义的一组声明，如 iss（发行者/签发机构）、exp（过期时间）和 sub（被签名的主体）。
	- Public Claims（公开声明）：任何一方都可以使用的声明，但应避免冲突。
	- Private Claims（私有声明）：由应用定义的声明，用于传输应用特定的信息。
- `Signature`：由 Header 和 Payload 一起通过签名算法（如 HMAC SHA-256 或 RSA）签名生成。签名用于验证令牌的完整性和来源，防止篡改。

## 3、创建Role并设置权限
假如我们需要一个角色，对pod/deployments资源仅有只读权限。这里我们创建一个ClusterRole进行说明。
```bash
cat > ops-role.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ops
rules:
- apiGroups:
    - ""
  resources:
    - pods
  verbs:
    - get
    - list
    - watch
EOF

kubectl create -f ops-role.yaml
```
上述配置我们创建了一个名为ops的clusterrole 的权限，仅具备 pod 资源的只读操作权限， 其他资源对象没有权限。

## 4、为ServiceAccount生成Token
创建ServiceAccount，并生成token

```bash
cat > infmax-ops-sa.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: infmax-ops
  namespace: default
EOF

kubectl create -f infmax-ops-sa.yaml
```

{{< alert >}}
自1.24之后，k8s不再为ServiceAccount自动生成token secret。 在1.24之前，当我们创建完ServiceAccount之后，k8s会自动生成对应的token。
{{< /alert >}}

这里我们手动创建token secret(所有版本均适用),  如果是为ServiceAccount创建对应的token secret, 需要遵循以下几个规则
- 在Annotation中指定serviceAccount的名字
- type类型为`kubernetes.io/service-account-token`  表示这是为serviceAccount生成的token。

```bash
cat > sa-secret.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: infmax-ops-token
  annotations:
    kubernetes.io/service-account.name: infmax-ops
type: kubernetes.io/service-account-token
EOF

kubectl create -f sa-secret.yaml  #secret会自动填充出token的字段
```

创建好之后我们查看下token内容
```bash
kubectl describe secret infmax-ops-token
```
可以看到token中包含三部分： `ca.crt` 为k8s的ca证书, `namespace` ，`token` 为JWT格式的token。上述describe的时候已经显示出token的base64解码后的值。 

```yaml
Name:         infmax-ops-token
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: infmax-ops
              kubernetes.io/service-account.uid: 3ad15dcc-f4cf-4378-a0cc-70730658cf5c

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1322 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjRIc2FyckEtdm84UUF2a0ZDUWpLbm1oeGIwcjlmLWdxbmZnd29TWDlnRzQifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImluZm1heC1vcHMtdG9rZW4iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiaW5mbWF4LW9wcyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjNhZDE1ZGNjLWY0Y2YtNDM3OC1hMGNjLTcwNzMwNjU4Y2Y1YyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmluZm1heC1vcHMifQ.LJi8e1e2QuJ9S91-wZdHQFJ-jo29eahPek0mB6oDVs0PPruz9yJLsrUN-sS4A4HfgpPiy72qmBLEZ7f1DrunbiSlbVY3914JnIa9v3onPanIuWrJBIWdR7acEOKZRXMGWP86LbweSdJHQG8If3XqowAALHOUWXuQFi10m9M7FD8ot-gTdskGtUrB25r4TZvjY2ZwD_48Tc-ib21U4sof9lToCLP7gYWMhkof-GswtYbtIWCeQw23kpDvycajMD10lAUmlRNxvJ-dwx8wiL8dWFhlwRVdzor5DarMhWKUDZ2wP5-07QlyRix4UtPpLCuV2EWr4K2ayrfUI8BWfKVt-Q```
```

或者可以通过get secret获取原始的数据，自己来进行解码， 结果和describe中的token值是一致的。
```bash
kubecet get secret  infmax-ops-token -o jsonpath='{.data.token}' | base64 -d 
```

JWT Token通过 `.` 分为三部分`header` `payload` `signature`三部分, 这三部分均是通过base64URL编码(不是base64)，我们分别解码后，可以得到token对应的JWT信息 ，JWT解码工具可以使用jwt.io在线工具查看会比较方便。如下

`header`

```json
{
  "alg": "RS256",
  "kid": "4HsarrA-vo8QAvkFCQjKnmhxb0r9f-gqnfgwoSX9gG4"
}
```
`payload`

```json
{
  "iss": "kubernetes/serviceaccount",
  "kubernetes.io/serviceaccount/namespace": "default",
  "kubernetes.io/serviceaccount/secret.name": "infmax-ops-token",
  "kubernetes.io/serviceaccount/service-account.name": "infmax-ops",
  "kubernetes.io/serviceaccount/service-account.uid": "3ad15dcc-f4cf-4378-a0cc-70730658cf5c",
  "sub": "system:serviceaccount:default:infmax-ops"
}
```

`signature`  是一个256bit的信息，sha256通常使用16进制表示。 这个值是通过header指定的RS256 算法，并通过私钥进行加密(这个私钥是签发ca.crt的CA对应的私钥)


## 5、使用Token生成kubeconfig
kubectl一般都具有admin的权限，我们通过kubectl来获取apiserver等信息，来构建kubeconfig.
```bash
cat >  gen_kubeconfig.sh << EOF
#!/bin/bash
KUBE_APISERVER=`kubectl config view -o jsonpath='{.clusters[0].cluster.server}'`  #apiserver地址
K8S_CERT=./ca.pem  #CA证书
kubectl get secret infmax-ops-token -o jsonpath='{.data.ca\.crt}' | base64 -d > $K8S_CERT #CA证书
KUBECONFIG_FILE=infmax-ops-kubeconfig
CLUSTER_NAME=kubernetes
USER_NAME=infmax-ops
CONTEXT_NAME=${USER_NAME}@${CLUSTER_NAME}
TOKEN=`kubectl get secret infmax-ops-token -o jsonpath='{.data.token}' | base64 -d`

# 设置集群参数
kubectl config set-cluster ${CLUSTER_NAME} \
    --certificate-authority=${K8S_CERT} \
    --embed-certs=true \
    --server=${KUBE_APISERVER} \
    --kubeconfig=${KUBECONFIG_FILE}

# 设置用户认证参数
kubectl config set-credentials ${USER_NAME} \
        --kubeconfig=${KUBECONFIG_FILE} \
        --token=$TOKEN

# 设置context—-将用户和集群关联起来
kubectl config set-context ${CONTEXT_NAME} \
    --cluster=${CLUSTER_NAME} \
    --user=${USER} \
    --kubeconfig=${KUBECONFIG_FILE}

# 设置默认context
kubectl config use-context ${CONTEXT_NAME} \
    --kubeconfig=${KUBECONFIG_FILE}

EOF

sh gen_kubeconfig.sh
```
在当前路径下生成infmax-ops-kubeconfig, 后面我们通过这个来访问apiserver

## 6、为ServiceAccount绑定角色或者更新权限
ServiceAccount创建并生成Token之后，这个Token是保持不变的。但是我们可以随时修改ServiceAccount对应的权限。
这里我们为`infmax-ops` 这个ServiceAccount绑定`ops` role

```bash
cat > infmax-ops-crb.yaml << EOF
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: infmax-ops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ops
subjects:
  - kind: ServiceAccount
    name: infmax-ops
    namespace: default
EOF

kubectl create -f infmax-ops-crb.yaml

```

至此，我们已经kubeconfig绑定了pod只读权限。

## 7、实验验证
kubectl指定我们刚刚生成的infmax-ops-kubeconfig文件进行操作。
```bash
#获取deployment被拒绝
kubectl --kubeconfig infmax-ops-kubeconfig get deployment
Error from server (Forbidden): deployments.apps is forbidden: User "system:serviceaccount:default:infmax-ops" cannot list resource "deployments" in API group "apps" in the namespace "default"

#获取pod
kubectl —kubeconfig infmax-ops-kubeconfig get pod -A -l app=nginx
NAMESPACE   NAME                     READY   STATUS    RESTARTS   AGE
default     nginx-578b6698c8-jf4xd   1/1     Running   0          4d1h
infmax      nginx-578b6698c8-bxhbv   1/1     Running   0          10m

#删除pod
kubectl --kubeconfig infmax-ops-kubeconfig delete pod nginx-578b6698c8-jf4xd
Error from server (Forbidden): pods "nginx-578b6698c8-jf4xd" is forbidden: User "system:serviceaccount:default:infmax-ops" cannot delete resource "pods" in API group "" in the namespace "default"
```

## 参考文献
- https://stackoverflow.com/questions/58957358/how-to-encode-and-decode-data-in-base64-and-base64url-by-using-unix-commands
- https://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html
- https://www.cnblogs.com/zhangmingcheng/p/17304954.html