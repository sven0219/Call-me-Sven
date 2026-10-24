---
title: "Debugging AirPlay Across Huawei FTTR Nodes: ARP, BSSIDs, and an IPv6 Firewall Toggle"
date: 2026-10-24T14:40:00+08:00
draft: false
author: "Sven"
summary: "A Mac on a K155 satellite gateway could not reach a Xiaomi TV on a V175 main gateway. A field report using ARP captures, ADB to confirm BSSIDs, and a still-incomplete IPv6 firewall toggle comparison."
showtoc: true
tags: ["FTTR", "Huawei", "AirPlay", "ARP", "IPv6", "Troubleshooting"]
categories: ["Networking", "Troubleshooting"]
---

# Debugging AirPlay Across Huawei FTTR Nodes

My Mac and Xiaomi TV used the same Wi-Fi network name, but screen mirroring stopped working. The network consists of a Huawei V175 main gateway and a K155 satellite gateway. Eventually we observed that a Mac connected to the K155 sometimes could not even reach the TV's IP address, while a Mac connected to the V175 could. This is a record of what we actually tested, including an **incomplete** comparison involving a gateway IPv6 firewall setting.

## Topology: an SSID is not an access point

| Device | Address | How we identified the node |
|---|---|---|
| V175 main gateway | `192.168.1.1` | 5 GHz BSSID `f0:c4:78:d6:2c:d8` |
| K155 satellite gateway | `192.168.1.13` | 5 GHz BSSID `c8:84:cf:16:92:54` |
| Xiaomi TV | `192.168.1.8` | Queried its current BSSID through ADB |
| Mac | `192.168.1.156` | BSSID in macOS Wi-Fi details |
| Mac mini | `192.168.1.160` | Confirmed on K155; used for an independent capture |

Both nodes share the SSID, and clients on both use **`192.168.1.1` as their default gateway**. The gateway address therefore tells us nothing about which access point a client is using. The gateway's `SSID5` and `ONU1` labels alone were not enough to identify the TV's node either. Wi-Fi roaming means the BSSID should be noted for each test.

## When the TV could not be reached

With the Mac associated to the K155 (`c8:84:cf:16:92:54`), it could ping both gateway addresses but not the TV. A connection to the TV's TCP port `47000` timed out, and `arp -n 192.168.1.8` reported `incomplete`. Earlier, on the V175, the Mac had been able to ping the TV, discover its AirPlay service, and connect to port `47000`. The gateway UI showed the K155 online with its FTTR link and configuration synchronized; restarting the K155 did not immediately eliminate the failure.

A packet capture on the Mac mini's Wi-Fi interface (`en1`) while pinging the TV showed repeated broadcasts:

```text
Request who-has 192.168.1.8 tell 192.168.1.160
Request who-has 192.168.1.8 tell 192.168.1.160
```

No ARP reply appeared in that capture. The client transmitted a request and did not receive an answer, but a **one-sided capture cannot tell** whether the request failed to reach the TV, the TV did not answer, or the reply failed to return. Since the client could not resolve the TV's IPv4 neighbor address, this was not merely an AirPlay discovery or application-port failure.

To repeat the capture on the Mac mini, run these in separate terminals:

```bash
sudo tcpdump -i en1 -n -e -l 'arp and host 192.168.1.8'
ping -c 4 192.168.1.8
```

SSH port 22 timed out on both gateways. The web administration credential worked on the main gateway, but not on the K155, so we could not inspect forwarding rules on the gateway itself.

## Proving that the TV was on the other node

Once the TV became reachable, we connected to its already-enabled network ADB service and queried its current Wi-Fi status:

```bash
adb connect 192.168.1.8:5555
adb -s 192.168.1.8:5555 shell cmd wifi status
```

The TV reported IP `192.168.1.8` and BSSID **`f0:c4:78:d6:2c:d8` (V175)**. At the same time, the Mac's Wi-Fi details showed **`c8:84:cf:16:92:54` (K155)**. This confirmed that the successful connectivity test crossed FTTR nodes. Either device could roam later; this observation applies to that point in time.

## The IPv6 firewall comparison

A [Huawei FTTR troubleshooting article](https://blog.csdn.net/weixin_32932149/article/details/167112271) suggested that some firmware's IPv6 firewall settings can affect LAN client connectivity. On the V175, we examined **Advanced Configuration → Security Settings → IPv6 Firewall → “Enable IPv6 firewall control of forwarded packets”** (translated from the UI).

The observed sequence was:

1. **With the option turned off**, the user reported that connectivity returned. In a subsequent test while the Mac was on K155, the TV answered all four pings, ARP resolved its MAC, TCP `47000` connected, and its AirPlay service was discovered. The Mac mini could ping it too.
2. The user **intentionally turned the option back on** to test the hypothesis. With the Mac again confirmed on K155, the TV failed all four pings; TCP `47000` returned `Host is down`, and ARP was `incomplete`. Both gateways remained reachable.
3. In a later test with the Mac on K155, four pings, ARP, and TCP `47000` worked again. ADB also confirmed that the TV was on the V175. **The firewall setting was not recorded alongside this later recovery**, so it cannot be counted as a confirmed “off again → working again” result. Actual screen mirroring was not recorded as a final verified outcome either.

The toggle and connectivity appear correlated, but **the mechanism and a lasting fix are not established**. An ordinary IPv6 firewall would not directly filter IPv4 ARP. Firmware behavior or another change in network state could be involved; timing alone does not prove what filtered the packets.

## Checking the other listed IP addresses

We also sent two pings per IPv4 address from the Mac to devices shown as online in the gateway UI. These responded: `192.168.1.8`, `.13`, `.21`, `.22`, `.26`, `.28`, `.32`, `.84`, `.98`, `.129`, `.142`, `.160`, `.169`, and `.170`. These did not respond and had incomplete ARP entries: `.14`, `.20`, `.24`, `.27`, `.128`, and `.130`. Address `.91` did not answer ping but had an ARP entry. Address `.156` was the testing Mac itself, not a remote connectivity result.

This sweep did not record the Mac's BSSID or the firewall state at the same moment. A gateway's “online” indicator does not guarantee an immediate ARP or ICMP reply: sleeping devices and devices that limit ping are possible explanations. The sweep alone does not establish that all non-responders are blocked across FTTR nodes.

## What would close the investigation

Keep the TV awake at `192.168.1.8`, verify its BSSID with `cmd wifi status`, and keep the Mac on the K155 while checking its BSSID before and after each test. Change **only the IPv6 firewall setting**, record the time and state, and repeat:

```bash
ping -c 4 192.168.1.8
arp -n 192.168.1.8
nc -G 3 -zv 192.168.1.8 47000
```

Finally, try actual screen mirroring. Repeated “on → ARP fails; off → ARP works” results under the same association and TV state would provide stronger evidence for a gateway firmware or forwarding issue to raise with the ISP. Disabling the setting may change how forwarded IPv6 traffic is protected, so its scope should be understood before leaving it disabled long-term.

References: [Huawei home FTTR overview](https://e.huawei.com/cn/solutions/enterprise-optical-network/ftth/fttr-home); [Huawei OptiXstar K155 support page](https://support.huawei.com/enterprise/zh/optical-business/optixstar-k155-pid-261856599).
