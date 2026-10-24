---
title: "Debugging AirPlay Across Huawei FTTR Nodes: ARP and the IPv6 Firewall"
date: 2026-10-24T14:40:00+08:00
draft: true
author: "Sven"
summary: "A Mac on a Huawei K155 satellite gateway could not reach a Xiaomi TV on the V175 main gateway. ARP captures, ADB BSSID checks, and an IPv6 firewall toggle narrowed down the issue."
showtoc: true
tags: ["FTTR", "Huawei", "AirPlay", "ARP", "IPv6", "Troubleshooting"]
categories: ["Networking", "Troubleshooting"]
---

# Debugging AirPlay Across Huawei FTTR Nodes: ARP and the IPv6 Firewall

## 1. Symptoms

This investigation started when I tried to debug my Xiaomi TV (`192.168.1.8`) over ADB, but my Mac (`192.168.1.156`) could not connect to it. That reminded me that screen mirroring had also worked only intermittently, so I decided to track down the connectivity issue. My Huawei FTTR setup has a V175 main gateway (`192.168.1.1`) and a K155 satellite gateway (`192.168.1.13`). During troubleshooting, I found that the Mac sometimes could not reach the TV while connected to the satellite, whereas it could reach the TV and discover its AirPlay service on the main gateway.

Both nodes share the same SSID, and clients use `192.168.1.1` as their default gateway. Neither the SSID nor the gateway IP identifies the access point. The relevant **BSSIDs** are `f0:c4:78:d6:2c:d8` for the V175's 5 GHz radio and `c8:84:cf:16:92:54` for the K155's.

## 2. Investigation

1. **Rule out a complete satellite outage.** While the Mac's BSSID identified the K155, it could reach both gateway addresses, but all four pings to the TV failed. TCP port `47000` timed out, and `arp -n 192.168.1.8` showed `incomplete`. The gateway UI showed the K155 and its FTTR link online; the failure was also observed after restarting it.

2. **Capture the ARP exchange.** A second Mac mini (`192.168.1.160`) connected to the K155 also could not reach the TV. Running `sudo tcpdump -i en1 -n -e -l 'arp and host 192.168.1.8'` on its Wi-Fi interface while pinging the TV showed repeated `Request who-has 192.168.1.8 tell 192.168.1.160` packets, with no ARP response. The request left the client, but a one-sided capture cannot tell whether it reached the TV or whether the reply was lost on its way back. This was more than an AirPlay discovery or application-port issue.

3. **Verify that the devices really used different nodes.** Once the TV became reachable, `adb connect 192.168.1.8:5555` followed by `adb -s 192.168.1.8:5555 shell cmd wifi status` reported the TV's current BSSID as `f0:c4:78:d6:2c:d8` (V175). At the same time, the Mac's Wi-Fi details showed `c8:84:cf:16:92:54` (K155). This confirmed a cross-node connection at that moment; either client can roam later.

4. **Compare the IPv6 firewall setting.** A [Huawei FTTR troubleshooting article](https://blog.csdn.net/weixin_32932149/article/details/167112271) suggested checking the V175's **Advanced Configuration → Security Settings → IPv6 Firewall → “Enable IPv6 firewall control of forwarded packets”** option (UI label translated). After turning it off, connectivity returned according to the user; with the Mac on K155, the TV answered 4/4 pings, ARP resolved, TCP `47000` connected, and AirPlay discovery worked. The user then deliberately turned the option back on: with the Mac still on K155, all four pings failed, ARP became `incomplete`, and TCP `47000` reported `Host is down`, while both gateways stayed reachable. The TV later became reachable again with the Mac on K155, but **the setting was not recorded alongside that later test**. It cannot be counted as a documented “off again → working again” trial.

## 3. Conclusion

During the failure, clients on the satellite could not resolve the TV's IPv4 address via ARP. We also verified that the Mac on K155 could communicate with the TV on V175 at a different point in time. The observed **reachable with the IPv6 firewall option off, unreachable when turned back on** sequence makes this setting the strongest lead so far.

The internal mechanism is still unknown: a normal IPv6 firewall should not directly filter IPv4 ARP. We did not inspect gateway forwarding rules or record a complete repeat test of the firewall state and actual screen mirroring after turning it off again. So this does not prove that the option directly blocked ARP or that disabling it is a permanent fix. A conclusive check would keep the TV awake and the Mac on K155, record both BSSIDs and each firewall state, and compare ping, ARP, and actual mirroring each time.
