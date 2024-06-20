---
title: Jmeter压测
date: 2024-06-20 17:54:35
tags: Jmeter
categories: Others
---

**摘要**
Jmeter是一款java开发的测试工具,本文主要对Jmeter测测进行简单介绍，通过本文可以利用Jmeter快速进行压测

# 0 安装
首先需要安装jdk
在启动Jmeter时，会提示，使用GUI进行配置，使用CLI进行测试。所以在安装Jmeter时，需要在开发机和MAC中分别安装。
![Jmeter-image1.png](/images/Jmeter-image1.png)

mac和开发机分别安装：
直接去官网下载 然后解压 https://jmeter.apache.org/download_jmeter.cgi
然后配置jmeter环境，
``` bash
export JMETER_HOME=/home/jmeter/apache-jmeter-5.6.3
export PATH=${JMETER_HOME}/bin:$PATH
```
![Jmeter-image2.png](/images/Jmeter-image2.png)
jmeter -v出现提示则表示安装成功
![Jmeter-image3.png](/images/Jmeter-image3.png)

由于是在开发机内进行测试，mac仅仅使用图形界面进行参数配置(也可以直接修改jmx文件)，本质上mac上的jmeter只要解压缩即可。
# 1 测试
进入bin目录中，执行./jmeter.sh启动图形界面
![Jmeter-image4.png](/images/Jmeter-image4.png)
首先添加线程组：
![Jmeter-image5.png](/images/Jmeter-image5.png)
然后添加http请求
![Jmeter-image6.png](/images/Jmeter-image6.png)
添加常量吞吐定时器，用于控制压测的qps
![Jmeter-image7.png](/images/Jmeter-image7.png)
注意，常量吞吐定时器中的数值是每分钟的吞吐量，要计算qps时，需要除以60。此外，可以选择基于所有活动线程和单个活动线程，不同的选择实际的qps是不同的。
**实际测试过程中发现，尽快常量吞吐定时器中的qps数值设置高，但是如果设置的线程数不够，qps也无法提高的预设的数值。**
![Jmeter-image8.png](/images/Jmeter-image8.png)
最后，在http请求中添加查看结果树和聚合报告，用于测试后的结果报告。
![Jmeter-image9.png](/images/Jmeter-image9.png)
Web服务器的请求形式如下：
![Jmeter-image10.png](/images/Jmeter-image10.png)
也可以添加断言，来调试测试中返回的结果是否符合预期。
设置完成后，保存,生成jtl后缀的文件，传到开发机中进行最终的压测。
测试命令如下：
``` bash
 jmeter -n -t test.jmx -l test.jtl
 jmeter -g test.jtl -o result
```
其中result是通过jtl生成的可视化结果
在开发机中，可以利用本地文件服务，利用本地浏览器直接浏览。 进入result目录，然后本地(mac)输入http://开发机ip:8888/ 即可。
``` bash
python -m SimpleHTTPServer 8888
```
![Jmeter-image11.png](/images/Jmeter-image11.png)

# 2 踩坑
# 2.1 Non HTTP response code: java.net.NoRouteToHostException" rm="Non HTTP response message: Cannot assign requested address
![Jmeter-image12.png](/images/Jmeter-image12.png)
通常是压测机器的端口耗尽，可以修改本地配置：vim /etc/sysctl.conf 
``` bash
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.ip_conntrack_max = 20000
```

# 2.2 Connect值为0
jtl最后一列Connect表示的连接耗时。
![Jmeter-image13.png](/images/Jmeter-image13.png)
如果开启Same user on each iteration，则表示线程不会重新进行连接，所以没有Connect耗时。
这与不勾选该选项测得的结果会差上Connect的平均耗时。
