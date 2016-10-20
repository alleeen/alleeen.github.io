---
layout: post
title:  "使用 EdgeMax 路由器自动翻墙"
date:   2016-10-20 16:21:39
comments: true
ads: true
categories: 网络技术
tags: [GFW,防火墙,EdgeMax,路由器]
---

作为肉身在墙内的计算机科学技术人员，不能上google实在是一件很遗憾的事情。

 > 什么？你说用百度？你站出来，我是你老板我肯定会开除你。

 百度搜索简直是垃圾中的战斗机，在使用百度时，你不得不忍受他的各种广告，各种竞价排名，而且英文资料极少。很多领先的开源作品、解决方案、论文什么的基本都是国外的。这些资料你查不到，你说什么与国际先进技术接轨？总之，没用过 Google 之前，你可能没什么感觉，但是用过了之后再用"某度”，你会抱怨，这搜的是些什么破玩意儿。。。但是，因为众所周知的原因，我们无法直接访问 Google，不能访问一些很优秀的国外网站，比如 slideshare（里面有很多优秀的PPT、文档）等等。那么怎么办？答案就是：翻墙！

 <!--more-->

 > 网监同志，我知道你在盯着我，我写这个纯粹是为了方便技术人员查阅资料，作为爱党爱国的四有青年，我翻墙出去后保证不受反动思想的荼毒，努力在墙外为祖国占领舆论高地

 翻墙的姿势有很多，什么 VPN，代理，自由门等等，本人也尝试过不少，目前来说用得最稳定的当属[Shadowsocks](https://shadowsocks.org)。刚开始的时候只是在一个 Shadowsocks 服务提供商处购买了一个账号，在我的 Macbook 上试用，后面发现翻墙速度快且稳定，就萌生了在路由器上安装 Shadowsocks 的想法，正好家中的主路由器是 EdgeRouter Lite 3（Unix 架构，完美！），于是：Let's do it!

## 概述
在搜索引擎中输入`Shadowsocks+路由器`关键字，可以搜索出很多安装教程，采用的方案也不尽相同，建议不想太折腾的话就买一个可以刷 OpenWRT 的路由器，按照这个博客（[https://cokebar.info/archives/978](https://cokebar.info/archives/978)）的教程安装配置就可以了。如果你和我一样入了 EdgeRouter 的坑，那我们继续~。

我采用的方案是使用 Shadowsocks + ChinaDNS + DNSMasq + iptables 来实现路由器智能翻墙，即国内流量走正常网络，国外流量走 Shadowsocks 代理，总体流程如下图：

![流程图](/assets/images/2016-10-20-edgemas-ss-tutorial/proxy-flow.png)

## 所需软件

#### shadowsocks-libev

项目地址：[https://github.com/shadowsocks/shadowsocks-libev](https://github.com/shadowsocks/shadowsocks-libev)，可下载源码在路由器上进行编译，或者直接下载[安装包](https://www.onlyos.com/wp-content/uploads/2015/08/shadowsocks-libev_2.2.4-1_mips.zip)，请注意这个安装包只适用于 EdgeRouter Lite 3，其他 EdgeMax 产品需要自行编译。

#### ChinaDNS

项目地址：[https://github.com/shadowsocks/ChinaDNS](https://github.com/shadowsocks/ChinaDNS)，可下载源码在路由器上进行编译，或者直接下载[安装包](https://www.onlyos.com/wp-content/uploads/2015/08/chinadns-1.3.2.zip)，请注意这个安装包只适用于 EdgeRouter Lite 3，其他 EdgeMax 产品需要自行编译。

#### DNSMasq
EdgeMax 中已经集成了 DNSMasq 无需另外安装。

## 配置
### shadowsocks-libev

shadowsocks-libev 安装好后，会在`/etc/init.d/`中安装一个启动脚本：`shadowsock-libev`，在路由器启动时会默认启动`ss-redir`服务，如果需要重启 Shadowsocks，可以使用命令：`sudo /etc/init.d/shadowsock-libev [start|stop|restart]`。Shadowsocks 的配置在文件`/etc/shadowsocks-libev/config.json`中，在这个文件中配置 Shadowsocks 需要链接的服务器信息：

```
{
"server”:”127.0.0.1″,
"server_port":8388,
"local”: "0.0.0.0”,
"local_port”:1080,
"password”:”barfoo!”,
"timeout”:60,
"method”:null
}
```

配置完成后，`sudo /etc/init.d/shadowsock-libev restart`，这样 Shadowsocks 就在你的路由器上运行了。

### ChinaDNS
使用源码编译后，会生成一个二进制文件：chinadns，可以将这个文件复制到`/usr/bin`中方便后面使用。ChinaDNS 需要一个文件来标识哪些 IP 属于国内，这个文件可以从[这里下载](http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest)：

```
# 下载 ip 列表到： /tmp/chnroute.txt
curl 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | grep ipv4 | grep CN | awk -F| ‘{ printf(“%s/%dn”, $4, 32-log($5)/log(2)) }’ > /tmp/chnroute.txt
```

下载完成后，将文件移动到`/etc/chinadns/chnroute.txt`，接着我们就可以尝试启动 ChinaDNS 了：

```
chinadns -s 223.5.5.5,223.6.6.6,127.0.0.1:5300 -c /etc/chinadns/chnroute.txt -p 35353 -m
```

需要注意的是`-s`参数后面的`127.0.0.1:5300`，这个代表将`ss-tunnel`转发的国外可信DNS站点作为上游DNS服务器。

#### ChinaDNS参数详解
+ -l：虚假IP列表：默认值：/etc/chinadns_iplist.txt

是GFW常见的DNS污染用IP列表，解析出列表中的IP结果时候，ChinaDNS会自动抛弃，保留默认即可；

+ -c：chnroute文件：默认值：/etc/chinadns_chnroute.txt

此文件标识哪些IP属于国内。用于ChinaDNS判断解析结果。ChinaDNS要求解析结果与DNS要匹配，国内网站采用国内DNS解析的结果，国外网站采用国外DNS解析结果，等等规则；确保以上两个文件内容完整无误，否则会造成无法启动；

+ -p：本地端口：默认值：5353

ChinaDNS所监听的端口。根据实际情况更改，注意不能和其他服务的端口重复（特别是DNSMasq和shadowsocks）；

+ -s: 上游服务器：默认值：114.114.114.114,8.8.8.8

可填入一系列的上游DNS服务器，根据实际情况来，可以保留默认，格式为”DNS_IP:PORT,DNS_IP:PORT”注意逗号后面不能有空格。有些ISP会封杀公共DNS，此时请将114DNS改为ISP的DNS；此处必须至少填入一个国内IP的DNS和一个国外IP的DNS，否则会造成ChinaDNS启动失败。额外的用法：ChinaDNS添加可信DNS避免一些异常

+ -y：等待时间： 默认值：0.3

为防止GFW的DNS污染抢答，而设置一个等待时间，请根据自己填写的国外DNS延迟值来填写，留下一定的裕度。GoogleDNS在国内延迟一般在100-200ms，留0.3比较合适。过大的值会造成DNS解析较大的延迟时间，过小的值可能导致无法接收正确的解析结果。

+ -d：双向过滤： 默认：开启

勾选时，当国外DNS服务器返回的查询结果是国内IP，或者当国内DNS服务器返回的查询结果是国外IP，则过滤掉这个结果（较为严格的模式）；去掉勾选的话只是过滤国内DNS的国外IP结果。

+ -m：启用压缩指针： 默认：不开启

利用GFW遇到压缩指针时的一个bug来精确识别来自GFW的抢答污染，从而极大提高识别的准确性和识别的效率，推荐启用，启用后，IPList和等待时间将禁用（因为用不到了）。 （已强制开启）

### ss-tunnel
ss-tunnel 是 Shadowsocks 的一个模块，可以用于 UDP 转发，为了防止 GFW 的 DNS 污染，我们用它来转发国外 DNS，可以通过下面命令来启动 ss-tunnel：

```
ss-tunnel -c /etc/shadowsocks-libev/config.json -u -b 0.0.0.0 -l 5300 -L 8.8.8.8:53
```
为了让 ss-tunnel 在每次重启路由器的时候自动重启，我们可以写个脚本放在`/config/scripts/post-config.d`目录下。

### DNSMasq
首先需要将系统的 DNS 服务器设置为本地

```
configure
# 停止通过pppoe更新dns设置，如果pppoe绑定在其他网口上，eth0需要变更为对应网口
set interfaces ethernet eth0 pppoe 0 name-server none

# 设置 DNSMasq 使用 ChinaDNS
# 需要注意的是，这里无法设置服务器端口，只能先这样设置后，再手工变更配置文件
edit service dns forwarding
set name-server 127.0.0.1

# 告诉路由器使用本地 DNSMasq 来解析域名
set system name-server 127.0.0.1

commit
save
exit
```

修改`/etc/dnsmasq.conf`，去除你之前自定义的规则，在最后加入`conf-dir=/etc/dnsmasq.d`，并将`server=127.0.0.1`修改为`server=127.0.0.1#35353`。新建并进入目录`/etc/dnsmasq.d`，下载 [accelerated-domains.china.conf](https://raw.githubusercontent.com/felixonmars/dnsmasq-china-list/master/accelerated-domains.china.conf) 和[foreign_list.conf](http://pan.baidu.com/s/1eQB7ACi)两个文件后复制两个文件到`/etc/dnsmasq.d`目录。这两个文件都会有更新，建议隔段时间更新一下。分别修改两个文件，将`accelerated-domains.china.conf`（ChinaList）文件中所有的的114.114.114.114修改为自己ISP的DNS或者其他效果更好的国内DNS的IP地址（也可以保留114DNS），格式为：`
server=/0-6.com/IP`；将`foreign_list.conf`（GFWList）文件中所有的 127.0.0.1#5300 修改为自己所用国外DNS，格式为：`server=/.lsxszzg.com/IP#PORT`，如果你使用shadowsocks的UDP转发来提供国外DNS解析，UDP转发的端口号为5300，那么就是默认的 127.0.0.1#5300 ，如果你使用一个非标端口的国外DNS服务，如3.4.5.6，端口5353，那么就改为 3.4.5.6#5353 。注意不要使用国外公共DNS，因为会被污染！最后重启 dnsmasq：

```
sudo /etc/init.d/dnsmasq restart
```

### iptables
上面所做的一切都是为了防止 GFW 的 DNS 污染，从而拿到正确的 IP 地址，那么拿到 IP 地址后，又如何将国外的流量转发到 Shadowsocks 呢？现在该 iptables 登场了，使用下面的代码设置 iptables 转发规则：

```
# Setup the ipset
  ipset -N chnroute hash:net maxelem 65536

  for ip in $(cat '/etc/chinadns/chnroute.txt'); do
    ipset add chnroute $ip
  done

  # 其他请求：
  # shadowsocks
  iptables -t nat -N SHADOWSOCKS

  iptables -t nat -A SHADOWSOCKS -p tcp --dport 33348 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 103.192.224.122 -j RETURN

  # ignore internal ip
  iptables -t nat -A SHADOWSOCKS -d 0.0.0.0/8 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 10.0.0.0/8 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 127.0.0.0/8 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 169.254.0.0/16 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 172.16.0.0/12 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 192.168.0.0/16 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 224.0.0.0/4 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 240.0.0.0/4 -j RETURN

  # ignore asia ip

  # Allow connection to chinese IPs
  iptables -t nat -A SHADOWSOCKS -p tcp -m set --match-set chnroute dst -j RETURN

  iptables -t nat -A SHADOWSOCKS -p tcp -j REDIRECT --to-ports 1080
  iptables -t nat -I PREROUTING -p tcp -j SHADOWSOCKS

```

iptables 配置完后，你应该可以再浏览器中正常的打开 Google 首页了。

### 一键式 DNS 配置脚本
使用下面的脚本可以在重启路由器时（需要将脚本放在`/config/scripts/post-config.d`目录下）更新`foreign_list.conf`和`accelerated-domains.china.conf`两个文件，并配置好 iptables：

```
#!/bin/sh

deleteFile() {
  local filePath=$1
  if [ -f "$filePath" ]; then
   rm "$filePath"
  fi
}
# get chinadns ignore list
updateChnroute() {
  deleteFile /tmp/chnroute.txt

  curl 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | grep ipv4 | grep CN | awk -F\| '{ printf("%s/%d\n", $4, 32-log($5)/log(2)) }' >  /tmp/chnroute.txt
  mv -f /tmp/chnroute.txt /etc/chinadns/
  chmod 644 /etc/chinadns/chnroute.txt

  if pidof chinadns>/dev/null; then
      /etc/init.d/chinadns restart
  fi
}

updateDnsmasqConf() {
  deleteFile /tmp/accelerated-domains.china.conf
  # download accelerated-domains.china.conf
  local DNS=223.5.5.5
  curl 'https://raw.githubusercontent.com/felixonmars/dnsmasq-china-list/master/accelerated-domains.china.conf' > /tmp/accelerated-domains.china.conf
  sed -i "s|^\(server.*\)/[^/]*$|\1/$DNS|"  /tmp/accelerated-domains.china.conf
  mv -f /tmp/accelerated-domains.china.conf /etc/dnsmasq.d/
  chmod 644 /etc/dnsmasq.d/accelerated-domains.china.conf

  # download foreign_list.conf
  python ../gfwlist2dnsmasq_noipset.py

  if pidof dnsmasq>/dev/null; then
      /etc/init.d/dnsmasq restart
  fi
}


configIptables() {
  # Setup the ipset
  ipset -N chnroute hash:net maxelem 65536

  for ip in $(cat '/etc/chinadns/chnroute.txt'); do
    ipset add chnroute $ip
  done

  # 其他请求：
  # shadowsocks
  iptables -t nat -N SHADOWSOCKS

  iptables -t nat -A SHADOWSOCKS -p tcp --dport 33348 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 103.192.224.122 -j RETURN

  # ignore internal ip
  iptables -t nat -A SHADOWSOCKS -d 0.0.0.0/8 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 10.0.0.0/8 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 127.0.0.0/8 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 169.254.0.0/16 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 172.16.0.0/12 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 192.168.0.0/16 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 224.0.0.0/4 -j RETURN
  iptables -t nat -A SHADOWSOCKS -d 240.0.0.0/4 -j RETURN

  # ignore asia ip

  # Allow connection to chinese IPs
  iptables -t nat -A SHADOWSOCKS -p tcp -m set --match-set chnroute dst -j RETURN

  iptables -t nat -A SHADOWSOCKS -p tcp -j REDIRECT --to-ports 1080
  iptables -t nat -I PREROUTING -p tcp -j SHADOWSOCKS
}

updateChnroute
updateDnsmasqConf
configIptables
```

gfwlist2dnsmasq_noipset.py：

```
#!/usr/bin/env python
#coding=utf-8
#
# Generate a list of dnsmasq rules with ipset for gfwlist
#
# Copyright (C) 2014 http://www.shuyz.com
# Ref https://code.google.com/p/autoproxy-gfwlist/wiki/Rules

import urllib2
import re
import os
import datetime
import base64
import shutil
import ssl

mydnsip = '127.0.0.1'
mydnsport = '5300'
# Extra Domain;
EX_DOMAIN=[ \
'.google.com', \
'.google.com.hk', \
'.google.com.tw', \
'.google.com.sg', \
'.google.co.jp', \
'.google.co.kr', \
'.blogspot.com', \
'.blogspot.sg', \
'.blogspot.hk', \
'.blogspot.jp', \
'.blogspot.kr', \
'.gvt1.com', \
'.gvt2.com', \
'.gvt3.com', \
'.1e100.net', \
'.blogspot.tw' \
]

# the url of gfwlist
baseurl = 'https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt'
# match comments/title/whitelist/ip address
comment_pattern = '^\!|\[|^@@|^\d+\.\d+\.\d+\.\d+'
domain_pattern = '([\w\-\_]+\.[\w\.\-\_]+)[\/\*]*'
ip_pattern = re.compile(r'\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b')
tmpfile = '/tmp/gfwlisttmp'
# do not write to router internal flash directly
outfile = '/tmp/dnsmasq_list.conf'
rulesfile = '/etc/dnsmasq.d/foreign_list.conf'

fs =  file(outfile, 'w')
fs.write('# gfw list ipset rules for dnsmasq\n')
fs.write('# updated on ' + datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S") + '\n')
fs.write('#\n')

print 'fetching list...'
if hasattr(ssl, '_create_unverified_context'):
	ssl._create_default_https_context = ssl._create_unverified_context
content = urllib2.urlopen(baseurl, timeout=15).read().decode('base64')

# write the decoded content to file then read line by line
tfs = open(tmpfile, 'w')
tfs.write(content)
tfs.close()
tfs = open(tmpfile, 'r')

print 'page content fetched, analysis...'

# remember all blocked domains, in case of duplicate records
domainlist = []


for line in tfs.readlines():
	if re.findall(comment_pattern, line):
		print 'this is a comment line: ' + line
		#fs.write('#' + line)
	else:
		domain = re.findall(domain_pattern, line)
		if domain:
			try:
				found = domainlist.index(domain[0])
				print domain[0] + ' exists.'
			except ValueError:
				if ip_pattern.match(domain[0]):
					print 'skipping ip: ' + domain[0]
					continue
				print 'saving ' + domain[0]
				domainlist.append(domain[0])
				fs.write('server=/.%s/%s#%s\n'%(domain[0],mydnsip,mydnsport))
		else:
			print 'no valid domain in this line: ' + line

tfs.close()

for each in EX_DOMAIN:
	fs.write('server=/%s/%s#%s\n'%(each,mydnsip,mydnsport))

print 'write extra domain done'

fs.close();
print 'moving generated file to dnsmasg directory'
shutil.move(outfile, rulesfile)

print 'done!'
```
