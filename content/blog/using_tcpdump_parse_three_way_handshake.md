---
title: "Using tcpdump parse three way handshake"
date: 2020-09-09T14:14:58+08:00
draft: true
---

# 引言
tcp是传输控制控制协议，建立点对点的全双工通信，稳定的通信建立和释放是怎么一个过程？


# 三次握手过程

三次握手是建立稳定连接的最少次数，再多也没有必要。

```
环境： Macos
抓包工具： tcpdump
```

打开terminal，输入如下命令
```
sudo tcpdump -vv -i en0 -S host www.baidu.com
```

--v 显示更多详细信息
-i 制定适配器「网卡」
host 指定目标主机，其他主机的包就会被过滤掉

命令运行之后
```
~ sudo tcpdump -vv -i en0 -S host www.baidu.com
tcpdump: listening on en0, link-type EN10MB (Ethernet), capture size 262144 bytes
```

在命令行发起http请求给百度，比如发起一个get请求百度的首页

```
curl www.baidu.com
```

执行之后命令行会抓到如下包
```
22:24:49.613343 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 64)
    192.168.1.3.61716 > 104.193.88.77.http: Flags [S], cksum 0x4d64 (correct), seq 2412988450, win 65535, options [mss 1460,nop,wscale 6,nop,nop,TS val 1440303018 ecr 0,sackOK,eol], length 0
22:24:50.621419 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 64)
    192.168.1.3.61716 > 104.193.88.77.http: Flags [S], cksum 0x497c (correct), seq 2412988450, win 65535, options [mss 1460,nop,wscale 6,nop,nop,TS val 1440304018 ecr 0,sackOK,eol], length 0
22:24:50.780632 IP (tos 0x0, ttl 51, id 0, offset 0, flags [DF], proto TCP (6), length 60)
    104.193.88.77.http > 192.168.1.3.61716: Flags [S.], cksum 0xe556 (correct), seq 3439104772, ack 2412988451, win 28960, options [mss 1400,sackOK,TS val 3284722769 ecr 1440303018,nop,wscale 7], length 0
22:24:50.780713 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)
    192.168.1.3.61716 > 104.193.88.77.http: Flags [.], cksum 0x7875 (correct), seq 2412988451, ack 3439104773, win 2060, options [nop,nop,TS val 1440304176 ecr 3284722769], length 0
22:24:50.780798 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 129)
```

本机192.168.1.3.61716 > 104.193.88.77往百度发送了syn消息 seq 2412988450
百度104.193.88.77.http > 192.168.1.3.61716往本机发送了一条seq+ack消息 seq 3439104772, ack 2412988451， 其中ack是上一个seq+1「2412988450+1」的值
本机192.168.1.3.61716 > 104.193.88.77 往百度发送了一条ack消息 seq 2412988451, ack 3439104773 其中ack上一条消息的seq+1 「3439104772+1」

你也许会看到flags [DF]，这是说A packet with the IP don‘t fragment flag is marked with a trailing (DF)

至此，我们看到了tcp三次握手的过程。

# 四次挥手过程

客户端ctl+c，主动断开连接
```
 192.168.1.3.61282 > 104.193.88.123.http: Flags [F.], cksum 0xb8de (correct), seq 4118482580, ack 3905661117, win 2048, options [nop,nop,TS val 1442650912 ecr 1755042800], length 0
23:04:25.432766 IP (tos 0x0, ttl 51, id 20136, offset 0, flags [DF], proto TCP (6), length 52)
    104.193.88.123.http > 192.168.1.3.61282: Flags [.], cksum 0xc00f (correct), seq 3905661117, ack 4118482581, win 57, options [nop,nop,TS val 1755042950 ecr 1442650912], length 0
23:04:25.432775 IP (tos 0x0, ttl 51, id 20137, offset 0, flags [DF], proto TCP (6), length 52)
    104.193.88.123.http > 192.168.1.3.61282: Flags [F.], cksum 0xc00e (correct), seq 3905661117, ack 4118482581, win 57, options [nop,nop,TS val 1755042950 ecr 1442650912], length 0
23:04:25.432905 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 52)
    192.168.1.3.61282 > 104.193.88.123.http: Flags [.], cksum 0xb7b5 (correct), seq 4118482581, ack 3905661118, win 2048, options [nop,nop,TS val 1442651058 ecr 1755042950], length 0
```
TODO 杨修 四次挥手的过程分析

# 参数解释

|  TCP Flag  | tcpdump Flag  | Meaning |
|  ----  | ----  |---- |
| SYN  | [S] | Syn packer, a session establishment request. |
| ACK  | [A] | Ack packet, acknowledge sender's data|
| FIN  | [F] | Finish flag, indication of termination.|
| RESET| [R] | Reset, indication of immediate abort of conn.|
| PUSH | [P] | Push, immediate push of data from sender. |
| URGENT| [U]| Urgent, takes precedence over other data. |
| NONE | [.] | Placeholder, usually used for ACK. |



作者：舌尖上的大胖
链接：https://www.jianshu.com/p/a57a5b0e58f0#
來源：简书
简书著作权归作者所有，任何形式的转载都请联系作者获得授权并注明出处。

# 总结