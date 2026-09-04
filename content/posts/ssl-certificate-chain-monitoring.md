---
title: "Debugging Missing SSL Expiration Metrics: The Incomplete Certificate Chain"
date: 2026-09-04 15:30:00+08:00
draft: false
author: "Sven"
summary: "A blackbox exporter Probe suddenly stopped reporting SSL expiry metrics. The real cause was not the probe but the site serving an incomplete TLS certificate chain. Includes an SSL/TLS primer plus a practical debugging walkthrough."
showtoc: true
tags: ["SSL","TLS","Certificate Chain","Prometheus","Blackbox Exporter","Kubernetes"]
Categories: ["DevOps","Monitoring"]
---

# Debugging Missing SSL Expiration Metrics: The Incomplete Certificate Chain

We had just renewed the SSL certificate for one of our sites when we noticed
the expiry monitoring had gone quiet. The blackbox exporter `Probe` kept
reporting `probe_success 0`, and the SSL expiration metrics
(`probe_ssl_earliest_cert_expiry`,
`probe_ssl_last_chain_expiry_timestamp_seconds`) were simply gone.

This post starts with a quick SSL/TLS primer, then walks through how we found
the root cause and why an "incomplete certificate chain" is more common than
you think.

## 1. SSL/TLS Primer

### 1.1 What SSL/TLS actually is

SSL (Secure Sockets Layer) and its successor TLS (Transport Layer Security)
are protocols that provide **encryption**, **authentication**, and
**integrity** for network connections. HTTPS is simply HTTP running over TLS.

When you connect to an HTTPS site, two things happen during the **TLS
handshake**:

1. **Server authentication**: the server proves its identity by presenting a
   certificate, and the client verifies it is signed by a trusted CA.
2. **Key exchange**: client and server negotiate a shared session key to
   encrypt the traffic that follows.

If step 1 fails, the handshake is aborted and the connection is refused —
which is exactly what happened to our probe.

### 1.2 Public/private key cryptography

TLS relies on asymmetric (public-key) cryptography:

- A **private key** is secret, kept only by the server. It is used to sign
  and to decrypt.
- A **public key** is distributed in the certificate. It is used to verify
  signatures and to encrypt.

The two keys form a mathematical pair: data encrypted with one can only be
decrypted with the other. This is why your private key must never leave the
server, while the certificate (containing the public key) is public.

### 1.3 What an X.509 certificate contains

An SSL certificate is an X.509 document. The important fields:

| Field | Meaning |
|-------|---------|
| Subject (CN / SAN) | Who this certificate belongs to — the domain name |
| Issuer | The CA that signed this certificate |
| Validity period | notBefore / notAfter — the expiry date we monitor |
| Public key | The server's public key (RSA / ECDSA) |
| Extensions | SAN, Key Usage, CRL/OCSP revocation info, Basic Constraints |
| Signature | CA's digital signature over all the above (tamper-proof) |

### 1.4 CSR — the request, not the certificate

A CSR (Certificate Signing Request) is what you submit to a CA to request a
certificate. It contains your identity info, your **public key**, and a
signature made with your **private key** (proving you hold it). The CSR
itself is public and contains no private key. The CA verifies you control
the domain, signs a certificate for it, and returns the certificate.

### 1.5 Types of certificates

- **DV (Domain Validation)**: proves domain control only. Fast and cheap.
- **OV (Organization Validation)**: also verifies the organization. Common for
  businesses.
- **EV (Extended Validation)**: stricter vetting, historically shown in the
  address bar.
- **Self-signed**: signed by yourself, not a CA. Good for internal testing,
  but clients will reject it unless explicitly trusted.

### 1.6 The chain of trust

Clients trust root CAs that are built into their OS/browser trust stores.
Since a root CA does not sign every site certificate directly, it delegates
to **intermediate CAs**:

```
Leaf certificate  ← signed by  Intermediate CA  ← signed by  Root CA (in trust store)
```

For verification to succeed, the client needs the leaf **and** every
intermediate up to a root it trusts. Missing intermediates are one of the
most common certificate misconfigurations — and the subject of this post.

## 2. The Symptom

We monitor HTTPS endpoints with a Prometheus `Probe`:

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

The `https_ssl` module performs a TLS handshake and extracts certificate
expiry data. When it works you see metrics like:

```
probe_success 1
probe_ssl_earliest_cert_expiry 1.804463999e+09   # ~ 2027-03-07
```

But we only saw:

```
probe_success 0
```

No SSL metrics at all. The first instinct is "the probe is broken", but the
probe was fine — it was the site that refused to complete a verified TLS
handshake.

## 3. Debugging the Probe Directly

Skip the whole Prometheus pipeline and hit the blackbox exporter directly:

```bash
# inside the blackbox-exporter pod
curl "http://localhost:9115/probe?module=https_ssl&target=https://www.example.com&debug=true"
```

The debug output immediately reveals the error:

```
level=error msg="Error for HTTP request"
  err="Get \"https://1.2.3.4\": tls: failed to verify certificate:
       x509: certificate signed by unknown authority"
level=error msg="Probe failed"
```

`certificate signed by unknown authority` means the client cannot build a
trust chain from the served certificate to a trusted root. The handshake is
aborted, so there is no TLS session, hence no SSL expiry metrics.

## 4. Inspecting What the Server Actually Sends

Use `openssl s_client` to see the certificate chain the server presents
during the handshake:

```bash
echo | openssl s_client \
  -connect www.example.com:443 \
  -servername www.example.com 2>/dev/null \
  | grep -E "^\s+\d+ s:|i:|Verify return code"
```

Typical broken output:

```
Certificate chain
 0 s:CN=www.example.com
   i:CN=Example Intermediate CA
Verify return code: 21 (unable to verify the first certificate)
```

Two clues:

- The server sends only **one** certificate (the leaf). The intermediate CA
  is missing.
- `Verify return code: 21` confirms the chain cannot be verified.

## 5. The Certificate Chain Basics

An SSL certificate chain has three layers:

| Part | Purpose | Served to clients? |
|------|---------|--------------------|
| Leaf certificate | The site's own cert, contains the public key | Yes |
| Intermediate CA | Signed the leaf; the missing piece in this story | Yes |
| Root CA | Signed the intermediate; built into clients' trust stores | No |

Root CAs are **not** sent by servers — clients already have them. But the
intermediate is required: without it, clients cannot link the leaf up to a
root they trust.

In our case, the Kubernetes `tls.crt` secret contained only the leaf:

```bash
kubectl -n some-ns get secret example-tls -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | grep -c "BEGIN CERTIFICATE"
# 1   <- only the leaf, intermediate missing
```

## 6. Finding and Downloading the Right Intermediate

The rule: **the issuer of the leaf certificate is the intermediate you need
to download.**

```bash
kubectl -n some-ns get secret example-tls -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -issuer
# issuer=..., CN=Example Intermediate CA
```

Then download that intermediate from the CA's official download page
(every public CA publishes its intermediates). Ours was `Example Intermediate CA`.

Build the full chain — leaf first, intermediate second:

```bash
cat leaf.crt intermediate.pem > fullchain.crt
grep -c "BEGIN CERTIFICATE" fullchain.crt   # 2
```

## 7. Verifying Before You Deploy

Download the corresponding root and verify offline:

```bash
openssl verify \
  -CAfile root.pem \
  -untrusted intermediate.pem \
  leaf.crt
# leaf.crt: OK
```

`OK` means the chain is now complete and will validate. The intermediate's
subject must equal the leaf's issuer — that's the sanity check.

## 8. Deploying and Verifying at the Origin

**1. Update the origin.** Put the full chain into the TLS secret, keeping the
private key untouched:

```bash
kubectl get secret example-tls -o jsonpath='{.data.tls\.key}' | base64 -d > key.pem
kubectl create secret tls example-tls \
  --cert=fullchain.crt --key=key.pem \
  -n some-ns --dry-run=client -o yaml | kubectl apply -f -
```

The secret only stores certificates; the ingress controller watches it and
reloads its TLS config automatically.

**2. Verify the served chain.** After the ingress reloads, check the chain
again:

```
 0 s:CN=www.example.com
   i:CN=Example Intermediate CA
 1 s:CN=Example Intermediate CA
   i:CN=Example Root G2
Verify return code: 0 (ok)
```

Two certs, verification `OK`. The fix is complete.

**3. Re-check the probe.** The same blackbox endpoint should now report:

```
probe_success 1
probe_ssl_earliest_cert_expiry 1.804463999e+09
```

## 9. Key Takeaways

- **`probe_success 0` on an SSL probe usually means the TLS handshake failed,
  not that the probe is broken.** Debug the probe directly with `debug=true`.
- **An incomplete chain (leaf without intermediate) is a common misconfig.**
  The server must serve leaf + intermediate; the root is not needed.
- **Read the issuer**: the intermediate you need is exactly the issuer of the
  leaf certificate.
- **Verify offline before deploying** with `openssl verify` — it takes seconds
  and catches the problem before it reaches production.
- **Certificates are public; only private keys are secret.** Never put a
  private key in a CSR, a blog post, or a repo.

The whole story came down to one missing line in a certificate bundle, but it
looked like a broken monitoring setup. Knowing how to read a certificate
chain makes these issues surface in minutes instead of days.