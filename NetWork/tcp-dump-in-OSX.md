> 本文是 [theagileadmin.com](https://theagileadmin.com/2017/05/26/tcpdump-and-wireshark-on-osx/)的中文翻译版本，内容有删减

其他wire-shark的文章：
- [wireshark 抓包新手使用教程 ](https://www.cnblogs.com/linyfeng/p/9496126.html)

在此之前，我遇到了亚马逊网络服务ELB的连接时间激增的问题，因此是时候用tcpdump以获取数据包跟踪和wireshark（很久以前就很虚无缥缈）来分析TCP连接问题了。



tcpdump默认已经在OSX 上安装了，因此第一步是确定要转储的网络接口。以下命令将列出您的所有网络接口。

```shell
networksetup -listallhardwareports


(base)  ~/ networksetup -listallhardwareports

Hardware Port: Ethernet Adapter (en4)
Device: en4
Ethernet Address: ba:cc:8a:ee:3f:32

Hardware Port: Wi-Fi
Device: en0
Ethernet Address: bc:d0:74:0a:34:98

Hardware Port: Thunderbolt 1
Device: en1
Ethernet Address: 36:0e:31:4d:99:80

Hardware Port: Thunderbolt 2
Device: en2
Ethernet Address: 36:0e:31:4d:99:84

Hardware Port: Thunderbolt 3
Device: en3
Ethernet Address: 36:0e:31:4d:99:88

Hardware Port: Thunderbolt Bridge
Device: bridge0
Ethernet Address: 36:0e:31:4d:99:80

VLAN Configurations
===================
```

然后，运行该接口上的数据包跟踪。我使用en0主无线接口，因此我运行：


``` shell
tcpdump -i en0 -s 0 -B 524288 -w ~/Desktop/demo.pcap

```



然后在命令行中另开一个tab，运行下面的命令，我使用的是ab（Apachebench），它是OSX自带的。其他常用的URL-hitters是curl，wget和siege。

```shell 
ab -n 1000 -c 100 https://www.ebay.com/

(base)  ~/ ab -n 1000 -c 100 https://www.ebay.com/
This is ApacheBench, Version 2.3 <$Revision: 1901567 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking www.ebay.com (be patient)
Completed 100 requests
Completed 200 requests
Completed 300 requests
Completed 400 requests
Completed 500 requests
Completed 600 requests
Completed 700 requests
Completed 800 requests
Completed 900 requests
Completed 1000 requests
Finished 1000 requests


Server Software:        ebay-proxy-server
Server Hostname:        www.ebay.com
Server Port:            443
SSL/TLS Protocol:       TLSv1.2,ECDHE-RSA-AES128-GCM-SHA256,2048,128
Server Temp Key:        ECDH P-256 256 bits
TLS Server Name:        www.ebay.com

Document Path:          /
Document Length:        274015 bytes

Concurrency Level:      100
Time taken for tests:   11.180 seconds
Complete requests:      1000
Failed requests:        999
   (Connect: 0, Receive: 0, Length: 999, Exceptions: 0)
Non-2xx responses:      894
Total transferred:      25431453 bytes
HTML transferred:       24854366 bytes
Requests per second:    89.44 [#/sec] (mean)
Time per request:       1118.043 [ms] (mean)
Time per request:       11.180 [ms] (mean, across all concurrent requests)
Transfer rate:          2221.33 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:      121  407 135.6    364     855
Processing:    38  557 842.6    361    4273
Waiting:       38  401 399.0    358    2287
Total:        342  964 911.7    691    4917

Percentage of the requests served within a certain time (ms)
  50%    691
  66%    747
  75%    838
  80%    852
  90%   1649
  95%   3786
  98%   4244
  99%   4521
 100%   4917 (longest request)
```

然后回到最初的命令行，使用control-C停止tcpdump捕获数据。现在我有了一个网络dump，其中包含我访问该URL的数据包（以及我计算机在此期间执行的其他任何其他网络数据，因此其中可能包含大量来自聊天客户端等的噪音）。


现在用wireshark来对上面的demo.pcap进行分析。下面是wireshark安装步骤我。



如果你想要UI界面，你需要安装它：

```shell
brew install wireshark --with-qt
```

如果你只是安装wireshark而不是–with-qt，你不会得到wireshark，你会得到一个叫tshark的命令行，然后你需要重新安装……对于这个，以及大多数事情，你需要Xcode或至少Xcode命令行工具（我总是只安装工具）。你可以使用下面的命令安装它们：

```shell
xcode-select --install

```

但是，如果你有一个旧版本（<8.2.1），wireshark构建将失败。要更新命令行工具，你……据说你不再需要了。App Store不再提供命令行工具更新，苹果公司对它们是否仍然存在变得越来越模糊和混乱。所以我只是从App Store安装了完整的XCode，无论如何，它只是网络和磁盘空间，以及对宇宙的热死亡做出贡献，但我不生气，然后它构建。

然后，你需要安装wireshark-chmodbpf cask：


或者，如果你想在本地wireshark中捕获而不是单独使用tcpdump，你还需要运行下面的命令：


```shell
brew cask install wireshark-chmodbpf

```
要分析你的tcpdump文件，只需运行

```shell

wireshark
```

并且加载捕获文件。然后将其排序到你想要的最快方法是找到一个感兴趣的事务的一部分——比如在我的例子中通过过滤“TCP”或者只是看看——然后右键单击一个包，并用“Follow… TCP stream”，你就可以得到一个完整的事务。

![](./pics/tcp01.jpg)
![](./pics/tcp02.jpg)
![](./pics/tcp03.jpg)

现在你可以通过查看并查看是否可以找到出错的原因来测试你的TCP/IP网络知识！