---
title: "Certbot 自动更新 ssl"
date: 2023-11-01T15:53:23+08:00
draft: false
author: "Sven"
summary: "Certbot 自动更新 ssl"
showtoc: true
tags: ["devops","SSL"]
---

https://certbot.eff.org/instructions

```bash
# 安装snapd
sudo apt update
sudo apt install snapd
sudo snap install core; sudo snap refresh core
# 安装certbot
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# 运行
sudo certbot certonly --nginx
# 配置
sudo certbot --nginx
# 手动更新证书（会自动更新，这里只是测试）
sudo certbot renew --dry-run
```

