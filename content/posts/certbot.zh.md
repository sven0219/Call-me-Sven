---
title: "Certbot 续期 SSL"
date: 2023-11-01T15:53:23+08:00
draft: false
author: "Sven"
summary: "使用 Certbot 续期 SSL"
showtoc: true
tags: ["devops","SSL"]
---

https://certbot.eff.org/instructions

```bash
# 安装 snapd
sudo apt update
sudo apt install snapd
sudo snap install core; sudo snap refresh core
# 安装 certbot
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# 运行
sudo certbot certonly --nginx
# 配置
sudo certbot --nginx
# 续期 --dry-run
sudo certbot renew --dry-run
```
