---
title: 'CA证书签名'
date: 2024-05-21T21:30:00+08:00
description: "ca generate with cfssl"
tags: ["linux","ssl"]
categories: ["system"]
draft: false
article.showTableOfContents: true
---
# CA配置 
`cfssl` 是一个开源的PKI/TLS工具包，它可以用于生成证书颁发机构（CA）。 `ca-config.json` 是 `cfssl` 的配置文件，用于定义CA的行为和策略。以下是 `ca-config.json` 的一些常见字段及其含义：   
1. `signing`：这个字段定义了CA的签名策略和默认设置。它包含以下子字段：   
    - `default`：默认的签名配置。   
    - `profiles`：一组命名的签名配置，可以在生成证书时选择使用。   
2. `signing.default`：这个字段定义了默认的签名配置，包含以下子字段：   
    - `usages`：定义了证书可以用于哪些用途，例如"signing"，"key encipherment"，"server auth"，"client auth"等。   
    - `expiry`：定义了证书的默认有效期。   
    - `ca_constraint`：定义了证书是否可以被用作CA。如果 `is_ca` 字段为true，那么这个证书可以被用来签发其他证书。   
3. `signing.profiles`：这个字段定义了一组命名的签名配置。每个配置都可以有自己的 `usages`， `expiry` 和 `ca_constraint` 设置。这样，你可以在生成证书时根据需要选择不同的配置。   
4. `auth_keys`：这个字段定义了一组用于API身份验证的密钥。   
5. `csr`：这个字段定义了默认的证书签名请求（CSR）参数，例如 `CN`（Common Name）， `key`（密钥参数）等。   
6. `db`：这个字段定义了用于存储证书和密钥的数据库配置。   
   

## CA配置及证书认证过程

在`ca-config.json`中,我们可以定义多个profiles,  例如
```json
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "peer": {
        "usages": [
            "signing", //用作给其他证书签名
            "key encipherment", //用作加密
            "server auth", //双向认证，这里server和client autch均打开
            "client auth"
        ],
        "expiry": "876000h"
      }
    }
  }
}
```

在证书颁发机构（CA）中， `server auth`和`client auth`是扩展密钥用途（Extended Key Usage，EKU）的两种类型，它们定义了证书的使用场景。   
`client auth`（客户端认证）是指证书被用于验证客户端的身份。这在需要双向SSL/TLS认证的场景中很常见。在这种情况下，不仅服务器需要向客户端提供证书，客户端也需要向服务器提供证书。例如，某些企业可能会为其员工发放客户端证书，员工在访问企业内部网站时需要提供这个证书，以证明他们是该企业的员工。这样，服务器就可以确认正在与正确的客户端通信。   
   
   
`server auth`（服务器认证）是指证书被用于验证服务器的身份。例如，当你访问一个HTTPS网站时，服务器会向你的浏览器提供一个证书，这个证书就是用于`server auth`。浏览器会检查这个证书，确认它是由一个可信的CA签发的，并且证书中的域名与你正在访问的网站的域名匹配。这样，你就可以确信你正在与正确的服务器通信，而不是一个冒充该服务器的攻击者。   

{{<mermaid>}}
sequenceDiagram
    participant C as 客户端
    participant S as 服务器
    C->>S: 发起连接请求
    S->>C: 发送服务器证书
    C->>C: 验证服务器证书
    C->>S: 发送客户端证书
    S->>S: 验证客户端证书
    S->>C: 确认连接

{{</mermaid>}}


## 生成根证书及签名
使用 `cfssl` 和 `ca-config.json` 文件生成CA证书和私钥的步骤如下：   
1. 首先，我们需要一个CA证书的证书签名请求（CSR）文件，例如 `ca-csr.json`。这个文件定义了CA证书的信息，例如Common Name，Organization等。以下是一个例子：   
   
```json
{
  "CN": "My CA", //这是Common Name的缩写，通常包含证书的主题。在这个例子中，它被设置为"My CA"。
  "key": { //这个字段包含了生成私钥的算法和大小。在这个例子中，我们使用的是RSA算法，密钥大小为2048位。
    "algo": "rsa",
    "size": 2048
  },
  "names": [ //这个字段包含了一系列的Distinguished Name（DN）字段。每个DN字段都是一个包含多个属性的对象，这些属性可以用来识别证书的主题
    {
      "C": "US", //这是Country的缩写，表示证书主题的国家。在这个例子中，它被设置为"US"。
      "ST": "California", //这是State或Province的缩写，表示证书主题的州或省份。在这个例子中，它被设置为"California"。
      "L": "San Francisco", //这是Locality的缩写，表示证书主题的城市。在这个例子中，它被设置为"San Francisco"。
      "O": "My Company Name", //这是Organization的缩写，表示证书主题的组织。在这个例子中，它被设置为"My Company Name"。
      "OU": "My Organizational Unit" //这是Organizational Unit的缩写，表示证书主题的部门。在这个例子中，它被设置为"My Organizational Unit"。
    }
  ]
}

```

   
2. 然后，我们可以使用 `cfssl` 命令行工具和 `ca-config.json` 文件生成CA证书和私钥：   

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

这个命令会生成两个文件： `ca.pem` 和 `ca-key.pem`。 `ca.pem` 是CA证书， `ca-key.pem` 是CA的私钥。   

3. 你可以使用这个CA证书和私钥来签发其他证书。例如，如果你有一个服务器证书的CSR文件 `server-csr.json`，你可以使用以下命令来签发服务器证书： 

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server-csr.json | cfssljson -bare server
```
这个命令会生成两个文件： `server.pem` 和 `server-key.pem`。 `server.pem` 是服务器证书， `server-key.pem` 是服务器的私钥。   
   
以上是如何使用 `cfssl` 和 `ca-config.json` 文件生成CA证书和私钥，以及如何使用这个CA证书和私钥来签发其他证书的步骤。   

`cfssl` 默认会在当前执行路径下查找 `ca-config.json` 文件。我们也可以使用 `-config` 参数指定其他路径的配置文件。例如：   
```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=/path/to/your/ca-config.json -profile=server server-csr.json | cfssljson -bare server
```

{{<mermaid>}}
sequenceDiagram
user --> cfssl: 创建ca-csr.json
cfssl --> cfssl: 生成CA证书和私钥
user --> cfssl: 创建server-csr.json
cfssl --> cfssl: 使用CA证书和私钥签发服务器证书
cfssl --> user: 返回服务器证书和私钥
{{</mermaid>}}
   
   
   
   
