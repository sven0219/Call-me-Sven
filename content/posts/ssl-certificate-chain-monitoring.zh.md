---
title: "SSL 到期监控数据丢失排查：不完整的证书链"
date: 2026-09-04 15:30:00+08:00
draft: false
author: "Sven"
summary: "blackbox exporter 的 SSL 探针突然不再上报证书到期指标，真正原因不是探针本身，而是站点下发了一条不完整的 TLS 证书链。文章包含 SSL/TLS 基础科普与完整排查过程。"
showtoc: true
tags: ["SSL","TLS","证书链","Prometheus","Blackbox Exporter","Kubernetes"]
Categories: ["DevOps","监控"]
---

# SSL 到期监控数据丢失排查：不完整的证书链

某天我们刚为站点续期了 SSL 证书，随后发现证书到期监控悄悄没有数据了：
blackbox exporter 的 `Probe` 一直报 `probe_success 0`，而证书到期相关的指标
（`probe_ssl_earliest_cert_expiry`、`probe_ssl_last_chain_expiry_timestamp_seconds`）
完全消失。

本文先从 SSL/TLS 基础讲起，再记录完整的排查过程，以及为什么"证书链不完整"
比想象中更常见。

## 1. SSL/TLS 基础科普

### 1.1 SSL/TLS 到底是什么

SSL（Secure Sockets Layer）和它的继任者 TLS（Transport Layer Security）
是提供**加密**、**身份认证**和**完整性**的传输协议。HTTPS 就是跑在
TLS 之上的 HTTP。

连接 HTTPS 站点时，**TLS 握手**中会发生两件事：

1. **服务器身份认证**：服务器出示证书证明自己的身份，客户端校验证书
   是否由受信任的 CA 签发。
2. **密钥交换**：客户端与服务器协商出共享会话密钥，用来加密后续流量。

如果第 1 步失败，握手直接中断、连接被拒绝——这正是我们探针遇到的情况。

### 1.2 公钥/私钥加密

TLS 依赖非对称（公钥）加密：

- **私钥**保密，只保存在服务器上，用于签名和解密。
- **公钥**通过证书公开分发，用于验证签名和加密。

两个密钥是数学配对的：用一个加密的数据只能用另一个解。所以私钥绝不能
离开服务器，而证书（包含公钥）是公开的。

### 1.3 X.509 证书包含哪些内容

SSL 证书本质是一份 X.509 文档，关键字段：

| 字段 | 含义 |
|------|------|
| Subject（CN / SAN） | 证书归属——域名 |
| Issuer | 签发这张证书的 CA |
| 有效期 | notBefore / notAfter——我们监控的到期时间 |
| 公钥 | 服务器的公钥（RSA / ECDSA） |
| 扩展 | SAN、Key Usage、CRL/OCSP 吊销信息、Basic Constraints |
| 签名 | CA 对以上所有内容的数字签名（防篡改） |

### 1.4 CSR 是申请，不是证书

CSR（Certificate Signing Request，证书签名请求）是向 CA 申请证书时提交
的内容。它包含你的身份信息、**公钥**，以及用**私钥**做的签名（证明你持有
对应私钥）。CSR 本身是公开的，不含私钥。CA 验证你确实拥有该域名后，
签发一张证书给你。

### 1.5 证书类型

- **DV（域名验证）**：只验证域名所有权，签发快、便宜。
- **OV（组织验证）**：额外验证企业组织，企业常用。
- **EV（扩展验证）**：审核最严格，历史上会在地址栏展示企业名。
- **自签名**：自己签发的证书，未经过 CA。适合内部测试，但客户端默认
  不信任，除非显式信任。

### 1.6 信任链

客户端信任内置在操作系统/浏览器信任库里的根 CA。根 CA 不会为每个网站
直接签发证书，而是授权给**中间 CA**：

```
叶证书  ← 签发者  中间 CA  ← 签发者  根 CA（在信任库中）
```

校验要成功，客户端需要拿到叶证书以及通往它信任的根证书之间**每一级**
中间证书。缺中间证书是最常见的证书配置错误之一——也正是本文的主角。

## 2. 现象

我们用 Prometheus `Probe` 监控 HTTPS 站点的证书：

```yaml
spec:
  interval: 60s
  module: https_ssl
  prober:
    scheme: http
    url: blackbox-exporter.monitoring.svc.cluster.local:9115
  targets:
    staticConfig:
      static:
      - https://www.example.com
```

`https_ssl` 模块会建立 TLS 握手并提取证书到期信息。正常时会看到：

```
probe_success 1
probe_ssl_earliest_cert_expiry 1.804463999e+09   # 约 2027-03-07
```

但实际只有：

```
probe_success 0
```

没有任何 SSL 指标。第一反应往往是"探针坏了"，其实探针一切正常——
是站点拒绝完成一次"能通过校验"的 TLS 握手。

## 3. 直接调试探针

跳过整个 Prometheus 链路，直接调 blackbox exporter：

```bash
# 在 blackbox-exporter 容器内执行
curl "http://localhost:9115/probe?module=https_ssl&target=https://www.example.com&debug=true"
```

debug 输出立刻暴露了错误：

```
level=error msg="Error for HTTP request"
  err="Get \"https://1.2.3.4\": tls: failed to verify certificate:
       x509: certificate signed by unknown authority"
level=error msg="Probe failed"
```

`certificate signed by unknown authority` 表示客户端无法把站点下发的证书
链到任一受信任的根证书，握手直接中断。没有 TLS 会话，自然就没有证书到期指标。

## 4. 看服务器到底下发了什么

用 `openssl s_client` 查看服务器在握手中实际下发的证书链：

```bash
echo | openssl s_client \
  -connect www.example.com:443 \
  -servername www.example.com 2>/dev/null \
  | grep -E "^\s+\d+ s:|i:|Verify return code"
```

故障时典型输出：

```
Certificate chain
 0 s:CN=www.example.com
   i:CN=Example Intermediate CA
Verify return code: 21 (unable to verify the first certificate)
```

两个线索：

- 服务器只下发了 **一张** 证书（叶证书），缺少中间证书。
- `Verify return code: 21` 确认链路无法通过校验。

## 5. 证书链基础知识

SSL 证书链分三层：

| 部分 | 作用 | 需要下发给客户端吗 |
|------|------|--------------------|
| 叶证书 | 站点自己的证书，包含公钥 | 是 |
| 中间证书 | 签发叶证书的 CA，本案缺失的一环 | 是 |
| 根证书 | 签发中间证书，内置在客户端信任库 | 否 |

根证书**不**由服务器下发——客户端信任库里本来就有。但中间证书必须有：
没有它，客户端无法把叶证书向上链接到它信任的根。

本案中，Kubernetes 的 `tls.crt` secret 里只放了叶证书：

```bash
kubectl -n some-ns get secret example-tls -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | grep -c "BEGIN CERTIFICATE"
# 1   <- 只有叶证书，缺中间证书
```

## 6. 找到并下载正确的中间证书

规则很简单：**叶证书的 Issuer（颁发者）就是你要下载的中间证书。**

```bash
kubectl -n some-ns get secret example-tls -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -issuer
# issuer=..., CN=Example Intermediate CA
```

然后去该 CA 的官方下载页下载这张中间证书（每个公开 CA 都会发布自己的
中间证书）。拼出完整链路——叶证书在前、中间证书在后：

```bash
cat leaf.crt intermediate.pem > fullchain.crt
grep -c "BEGIN CERTIFICATE" fullchain.crt   # 2
```

## 7. 部署前先离线验证

下载对应的根证书做离线校验：

```bash
openssl verify \
  -CAfile root.pem \
  -untrusted intermediate.pem \
  leaf.crt
# leaf.crt: OK
```

输出 `OK` 说明链路已完整、可以校验通过。中间证书的 subject 必须等于
叶证书的 issuer，这是最基本的自检。

## 8. 在源站部署并验证

**1. 更新源站。** 把完整链路写进 TLS secret，私钥保持不变：

```bash
kubectl get secret example-tls -o jsonpath='{.data.tls\.key}' | base64 -d > key.pem
kubectl create secret tls example-tls \
  --cert=fullchain.crt --key=key.pem \
  -n some-ns --dry-run=client -o yaml | kubectl apply -f -
```

secret 只存证书，ingress controller 会监听它并自动重载 TLS 配置。

**2. 验证下发链路。** 等 ingress 重载后再次检查证书链：

```
 0 s:CN=www.example.com
   i:CN=Example Intermediate CA
 1 s:CN=Example Intermediate CA
   i:CN=Example Root G2
Verify return code: 0 (ok)
```

两张证书、校验 `OK`，修复完成。

**3. 复测探针。** 同一个 blackbox 端点现在应该上报：

```
probe_success 1
probe_ssl_earliest_cert_expiry 1.804463999e+09
```

## 9. 经验总结

- **SSL 探针 `probe_success 0` 通常意味着 TLS 握手失败，而不是探针坏了。**
  用 `debug=true` 直接调探针定位。
- **"只有叶证书、缺中间证书"是很常见的配置错误。** 服务器要下发
  叶 + 中间，根证书不需要。
- **看 Issuer 就行**：要下载的中间证书，就是叶证书的 Issuer。
- **部署前先离线验证**：`openssl verify` 几秒钟就能发现问题，避免把
  错误配置带到生产。
- **证书是公开的，只有私钥需要保密。** 私钥绝不能写进 CSR、博客或代码仓库。

整个问题归根结底只是证书文件里少了一行，看起来却像监控坏了。会读证书链，
这类问题几分钟就能定位，而不是排查好几天。