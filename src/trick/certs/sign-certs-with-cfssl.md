# 使用 cfssl 生成证书

搭建各种云原生环境的过程中，经常需要生成证书，比如最常见的 etcd，本文记录使用 cfssl 快速生成证书的方法。

## 安装 cfssl

**方法1**: 去 [release](https://github.com/cloudflare/cfssl/releases) 页面下载，然后解压安装。

**方法2**: 使用 go install 安装:

```bash
go install github.com/cloudflare/cfssl/cmd/cfssl@latest
go install github.com/cloudflare/cfssl/cmd/cfssljson@latest
```

## 创建 CA 证书

由于各个组件都需要配置证书，并且依赖 CA 证书来签发证书，所以我们首先要生成好 CA 证书以及后续的签发配置文件:

``` bash
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "SiChuan",
      "L": "ChengDu",
      "O": "Kubernetes",
      "OU": "CA"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "876000h"
      }
    }
  }
}
EOF
```

生成的文件中有下面三个后面会用到:

* `ca-key.pem`: CA 证书密钥
* `ca.pem`: CA 证书
* `ca-config.json`: 证书签发配置，用 CA 证书来签发其它证书时需要用

csr 文件字段解释:

* `CN`(Common Name): apiserver 从证书中提取该字段作为请求的用户名 (User Name)
* `names[].O`(Organization): apiserver 从证书中提取该字段作为请求用户所属的组 (Group)

> 由于这里是 CA 证书，是签发其它证书的根证书，这个证书密钥不会分发出去作为 client 证书，所有组件使用的 client 证书都是由 CA 证书签发而来，所以 CA 证书的 CN 和 O 的名称并不重要，后续其它签发出来的证书的 CN 和 O 的名称才是有用的。

      > CA是什么？

      CA是Certificate Authority的缩写，也叫“证书授权中心”。它是负责管理和签发证书的第三方机构，作用是检查证书持有者身份的合法性，并签发证书，以防证书被伪造或篡改。
      所以，CA实际上是一个机构，负责“证件”印制核发。就像负责颁发身份证的公安局、负责发放行驶证、驾驶证的车管所。

      > CA证书是什么？

      顾名思义，CA 证书就是CA颁发的证书。 CA证书也就我们常说的数字证书，包含证书拥有者的身份信息，CA机构的签名，公钥和私钥。身份信息用于证明证书持有者的身份；CA签名用于保证身份的真实性；公钥和私钥用于通信过程中加解密，从而保证通讯信息的安全性。

      主要为以下两点：CA是权威可信的第三方机构，是“发证机关”。CA证书是CA发的“证件”，用于证明自身身份，就像身份证和驾驶证。

      > CA证书的选择

      电子合同平台的核心是CA，平台通过CA的数字认证技术和数字证书产品来实现有效的电子签名，让电子合同符合《电子签名法》，具备防篡改、抗抵赖的特性，从而拥有和纸质合同同等的法律效力。一般来说，CA必须是所有行业和所有公众都信任的、认可的。因此它必须具有足够的权威性。




## 为 ETCD 签发证书

这里证书可以只创建一次，所有 etcd 实例都共用这里创建的证书:

``` bash
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "*.karmada-system.svc",
    "*.karmada-system.svc",
    "*.karmada-system.svc.cluster",
    "*.karmada-system.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "SiChuan",
      "L": "Chengdu",
      "O": "etcd",
      "OU": "etcd"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  etcd-csr.json | cfssljson -bare etcd
```

> hosts 需要包含 etcd 被访问时用到的地址，可以用 IP ，域名或泛域名。

会生成下面两个重要的文件:

* `etcd-key.pem`: etcd 密钥。
* `etcd.pem`: etcd 证书。
