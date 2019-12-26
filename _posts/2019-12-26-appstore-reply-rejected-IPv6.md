---
layout: post
title: 苹果app申请IPV6不能访问解决方案
date: 2019-12-26 00:00:00 +0300
description:  苹果应用审核IPV6无法访问通过 # Add post description (optional)
img: how-to-start.jpg # Add image post (optional)
tags: [apple, Appstore, reject, nginx] # add tag
---

## 前言
我们要想把自己开发出来放到App Store或者Google Play上供玩家下载，中间还有一个非常的重要的步骤：app审核。
为了区分app审核和线上正式运营的游戏服，我们需要重新部署一套专供审核使用的游戏服，但是由于有诸多因素，比如阿里云DNS服务器不支持ipv6，
登陆服务器，SDK服务器都部署在中国大陆，导致App store进行审核时访问不到服务器，app就被打回来，为了解决这一技术问题，
下面将说明如何构建整套流程和需要注意的事项。

## 访问机制和App被拒主要原因
苹果AppStore审核员在美国的IPv6-Only环境下对APP进行访问（审核），如果APP Server支持IPv6，则可直接访问；如果APP Server不支持IPv6，
则通过DNS64 +NAT64进行访问；很明显，大部分开发者的APP服务器都是不支持IPv6直接访问的，所以基本是用NAT64+DNS64进行访问的。那么我们就先了解NAT64+DNS64的访问机制吧，直接看图：
![NAT64+DNS64的访问机制]({{site.baseurl}}/assets/img/apple-reject-ipv6/dns64_nat64.png)

从这里看出审核的关键在于能不能获取一个有效的Server IPv6地址。当苹果公司的APP审核员在进行审核时，由于国内大部分开发者的APPserver没有IPv6地址，只能通过苹果公司自己的NAT64+DNS64服务器进行测试，而最关键的是苹果的服务器不能有效的给APPserver返回一个IPv6地址，这就导致了审核失败，APP被拒。
就国内目前来说审核被拒的主要原因有第三个：
    1、国内大部分APP服务器没有IPv6地址，导致DNS无法解析；
    2、苹果公司的审核环境不能自动将中国APP内URL转换成IPv6可访问的格式，导致访问失败；
    3、由于国际线路带宽严重拥堵等原因造成访问不稳定，失败率高；

## 解决思路
就目前国内的现状，能够提供这种服务的当属教育网了，中国教育网坐拥全国几百所高校，拥有真实的IPv6骨干网络，国际出口，IPv6资源丰富，服务质量好。
既然审核被拒是因为IPV6，那么我们就让服务器支持就可以了，但是很多运营商的服务器不提供IPv6地址，这样的话就要使用IPv6隧道技术,通过建立隧道使自己的服务器通过IPv6隧道来支持IPv6,方案示意图如下：
![支持IPv6]({{site.baseurl}}/assets/img/apple-reject-ipv6/ipv6_deploy.png)
    使用IPv6隧道服务APP服务器必须满足三个条件：
    ① 服务器拥有公网IPv4地址
    ② 服务器支持IPv6协议
    ③ 服务器放行6in4协议

## 准备材料
    一台海外vps，支持nginx ipv6版本安装包，tunnelbroker ipv6隧道申请

### 审核服游戏服务器支持ipv6协议
使用ipv6一键支持脚本让审核服支持ipv6，脚本放在运维控制机器上，目录为/data/svr
脚本名为check_ip6_support.sh，使用scp命令脚本传输到审核服上，停掉所有运行服务器，停掉MySQL，使用reboot指令让服务器快速重启(必须重启)。

## Web服务器支持ipv6协议
查看web服务器上nginx是否设置支持ipv6，如果没有则需要充运行控制机器上拷贝nginx安装软件进行安装，nginx版本为1.10.0，不要搞错了，在编译过程参数要加—with-ipv6参数，编译完成后覆盖安装，重启nginx。

## 购买一台海外的vps
这里选择的老牌vps产商搬网工，官方网址为https://www.bwh1.net，注册和登陆请按照英文官网来就好，注册请按照真实信息填写，否则可能不会通过。注册的时候必须要支持google翻墙，否则验证码刷新不出来哦，选配置机型请选择OVZ类型，此类型支持ipv6，不过并不可用，下面会介绍相应的解决方案。
 
## 编译安装Nginx
在vps创建构建目录，从运维控制机器上把Nginx安装包传输过来，并编译并安装nginx，需要注意编译参数.
```
mkdir /data
cd /data/
mkdir www
scp root@ip:/data/svr/playbooks/public/install/nginx-1.10.0.tar.gz .
tar -zxvf nginx-1.10.0.tar.gz
cd  nginx-1.10.0
./configure --user=www --group=www --prefix=/usr/local/nginx   --with-http_stub_status_module   --with-http_ssl_module   --with-http_v2_module   --with-http_gzip_static_module  --with-ipv6   --with-http_sub_module
Make && make install
```
## 配置nginx代理转发配置
Nginx负责将App store的请求转发给国内服务器，所以国内web服务器配置地址和https证书也要配置。
 
## 申请ipv6代理隧道
刚才我们相当于把nginx环境搭建起来，但是还没有配置ipv6地址，vps官网此服务器的ipv6地址目前测试不可用，需要借助第三方走ipv6代理隧道。
申请ipv6地址网站：https://tunnelbroker.net/
申请方法和配置方法请参考 https://blog.csdn.net/EI__Nino/article/details/71331717文章中
IPv6隧道配置。
申请ipv6隧道成功如下图所示：
 
申请完成后，我们会有一个ipv6地址2001:470:1f06:ac8::2，确认申请了IPv6隧道服务并按照上述模板进行配置完成后，请检查防火墙（iptables）是否放行了6in4协议，并确认(/etc/sysctl.conf)中IPv6转发已打开。
## 绑定ipv6域名
打开DNS解析网站,添加AAAA记录解决，把上面的ipv6地址添加上去。
	 
## 安装ipv6转发软件
添加ipv6解析了，环境搭建好了，你以为就能访问了吗？少年你真是天真了。我们还差一个类似需要ipv6协议转发器。
1.下载https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/tb-tun/tb-tun_r18.tar.gz
2.解压tar zxf tb-tun_r18.tar.gz
3.编译tb-tun
```
gcc tb_userspace.c -l pthread -o tb_userspace
```
这一步编译将会生成tb_userspace二进制文件。
4.移动tb_userspace到/usr/bin/，授权
```
mv tb_userspace /usr/bin/
chmod a+x /usr/bin/tb_userspace
```
5.Vim编辑/etc/init.d/ipv6hetb文件，如果没有就把下列的内容复制进去，将下图的红色框框的信息换成申请的ipv6协议页面的信息。
```
#!/bin/bash
#这是一段高度简化的配置流程的代码，网站已经很难找得到了，直接复制不要手打。
touch /var/lock/ipv6hetb
#Variables
SERVER_IP4_ADDR="" #Server IP From Hurricane Electric
CLIENT_IP4_ADDR="" #Your server IPv4 Address
CLIENT_IP6_ADDR="2001:470:1f06:ac8::2/64" #Client IPv6 Address from Hurricane Electric
ROUTED_IP6_ADDR="2001:470:1f06:ac8::1/64" #Your Routed IPv6 From Hurricane Electric
case "$1" in
start)
    echo "Starting ipv6hetb "
    setsid tb_userspace tb $SERVER_IP4_ADDR $CLIENT_IP4_ADDR sit > /dev/null 2>&1 &
    sleep 3s
    ifconfig tb up
    ifconfig tb inet6 add $CLIENT_IP6_ADDR
    ifconfig tb inet6 add $ROUTED_IP6_ADDR
    ifconfig tb mtu 1480
    route -A inet6 add ::/0 dev tb
    route -A inet6 del ::/0 dev venet0
    ;;
stop)
    echo "Stopping ipv6hetb"
    ifconfig tb down
    route -A inet6 del ::/0 dev tb
    killall tb_userspace
    ;;
*)
    echo "Usage: /etc/init.d/ipv6hetb {start|stop}"
    exit 1
    ;;
esac
exit 0
```

## 授权
```
chmod a+x /etc/init.d/ipv6hetb
```

## 启动服务器
```
service ipv6hetb start
#使用ifconfig命令进行网卡信息输出
ifconfig
```
如果配置正确，会有下图红色框信息产生。
 <br>

## 使用ipv6网站测试
测试网站：http://ipv6-test.com/
打开网站后，找到Website，在输入网址框输入测试网址,查看是否各项结构是否正常，这里省略掉。
 <br>

## 服务器测试
服务器nginx的access.log也会打印测试信息，请自行查看.
<br>

## 注意事项
1. 购买海外VPS不允许安装翻墙软件，以免被强；
2. tb-tun的安装包太难找了，请注意备份；
3. 审核需要用到的阿里云服务器必须要用ipv6支持脚本跑一次，这一步骤已经集成在服务器初始化中去了；
4. nginx版本一定要支持ipv6，否则无法接受ipv6协议的请求；
5. 由于以上的做法也属于翻墙，要考虑国家政策，选择vps厂商要尽量稳定；
<br>

## 参考链接
认识ipv6和app审核访问机制: http://www.solve6.com/  
节点配置ipv6地址，搭建思路: https://bbs.aliyun.com/read/299254.html  海外
Nginx代理转发配置: http://blog.csdn.net/faye0412/article/details/75200607  
Aliyun服务器配置支持ipv6: https://blog.csdn.net/EI__Nino/article/details/71331717 
配置tunnelbroker.net的IPv6: https://qiaodahai.com/openvz-virtual-machine-configuration-ipv6-with-tunnelbroker-net.html 
