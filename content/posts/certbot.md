---
title: "Certbot renew SSL"
date: 2023-11-01T15:53:23+08:00
draft: false
author: "Sven"
summary: "Certbot renew ssl"
showtoc: true
tags: ["devops","SSL"]
---

https://certbot.eff.org/instructions

```bash
# install snapd
sudo apt update
sudo apt install snapd
sudo snap install core; sudo snap refresh core
# install certbot
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# run 
sudo certbot certonly --nginx
# config
sudo certbot --nginx
# renew --dry-run
sudo certbot renew --dry-run
```

