---
layout: post
title: Ubnt UniFi 产品开箱
date: 2017-04-30 16:21:39
comments: true
ads: true
categories: [网络技术]
tags: [Ubnt,Wi-Fi,无线覆盖]
---

家里的无线网络覆盖一直有些问题，虽然说已经在家里部署了两个无线AP，但是还是一些小问题，首先信号覆盖还是有一些死角，比如说，卫生间，一进卫生间，信号强度瞬间掉到只有一格；其次就是两个 AP 之间相互协作好像有点问题，经常出现终端在 AP 1 的旁边，却连接到了 AP 2 上，只能手动断开 Wi-Fi，并重新连接。虽然说，这不是什么大问题，但对于一个有强迫症的 IT 男来说，这就像背痒一样，不挠一下不舒服。于是，我就打算把家里的无线网络改造一下。

<!-- more -->

# 一些背景和需求

先放一下家里的户型图，家虽然不算很大，正常来说，这样的户型靠一个普通的无线路由器就能完成全屋的覆盖。但是这套房子有点特殊，房间的墙不是普通的那种空心砖，全是钢筋混凝土浇筑的墙，不管路由器放在哪个房间，总有地方覆盖不到。

![户型图](/assets/images/2017-04-30-new-home-network-structure/huxingtu.png)

鉴于这种原因，就只能采用有线路由器加多个无线 AP 的网络结构，下面是我家网络原有的拓扑图。客厅和弱电箱之间，由于只拉了一条网线，而且我们家电视用的是电信的IPTV，所以只能通过划分不同的 VLAN 来达到一条网线跑两路数据的要求。其实大的网络结构上并没有什么问题，就是两个无线 AP 之间的协同有点让人闹心，所以这次网络改造的主要重点就是解决多个无线 AP 之间的协同问题。

# 器材选择

在这之前其实做过很多功课，传统的解决方案一般都是瘦 AP 加上 AC 控制器，但是这种方案并不适合于我。首先，市面上并没有太多面向家庭或者小企业的廉价 AC + 瘦 AP 解决方案，向华为、思科这种牌子，一个 AP 就要 2K 左右，AC 控制器更是天价；其次，弱电箱太小了，里面已经塞了一台光猫，一台路由器和一个交换机了，实在是再也塞不下一个 AC 控制器了。

前段时间 Ubnt 面向家用市场推出了一个 Wi-Fi Mesh 解决方案：Amplifi Wi-Fi。路由器和扩展点之间的连接走的是无线，省去了拉网线的烦恼，整个产品整体颜值也高，而且由于是全套解决方案，扩展点之间的协作就更加没有问题了。对于这个产品，我曾经有种草了很久，但是最近看到各路评测的开箱，发现这个产品根本不支持 VLAN 这就没法满足我看 IPTV 的需求了，而且扩展点和主路由之间的无线通讯肯定没有有线来的快，所以只能 PASS 这个产品。

后来我把目光放到了 Ubnt 的 UniFi 产品上，UniFi 的 AP 有个特点，就是可以用软 AC 进行控制，而且不要求 AC 实时在线。使用软 AC 配置好网络后，关闭 AC，整个网络依然可以正常工作，这样就省去了需要硬 AC的烦恼。而且现在家里用的路由器也是 Ubnt 的产品，本着凑一套的思路，我最终选择了 Ubnt UniFi。

# 开箱

通过万能的淘宝购买了三件 Ubnt 产品：UBNT US-8-60W 千兆 PoE 交换机、UBNT UAP-AC LITE 吸顶 PoE 无线 AP 和 UBNT UAP-AC-IW 入墙面板无线 AP，整套价格 2K 出头。好了，下面正式进入开箱环节。

![开箱](/assets/images/2017-04-30-new-home-network-structure/IMG_2384.jpg)

第一个出场的是交换机，因为两个 AP 都需要 PoE 供电，所以一个具备 PoE 输出功能的交换机不可或缺。产品的包装盒十分简洁，没有什么多余的东西，盒子里面的附件除了机器本体外，还有一个 220V 的电源和一个简要的安装说明。

![UBNT US-8-60W 千兆PoE网管型交换机](/assets/images/2017-04-30-new-home-network-structure/IMG_2378.jpg)

将交换机装入弱电箱，接上电源和路由器之间的网线。请忽略杂乱的布线，开发商装的弱电箱不是怎么给力，所以也就懒得去折腾了。

![弱电箱](/assets/images/2017-04-30-new-home-network-structure/IMG_2380.jpg)

接下来是 UBNT UAP-AC LITE，产品包装同样很简洁，飞碟型的造型也很漂亮。家用的话买 LITE 也就足够了，尺寸比较小巧一点。

![吸顶天线](/assets/images/2017-04-30-new-home-network-structure/IMG_2376.jpg)

安装后的效果，别问我为什么吸顶天线不吸顶，因为天花板上没有预留网线，就那么简单。

![吸顶天线安装](/assets/images/2017-04-30-new-home-network-structure/IMG_2385.jpg)

最后一个出场的就是 UBNT UAP-AC-IW，它是 Ubnt 的一款新产品，以前 Ubnt 也有一款入墙式无线 AP，但是那货不支持国内的 86 盒，所以没法用。这个面板有三部分组成，安装底座、AP 本体和外盖板。

![面板外包装](/assets/images/2017-04-30-new-home-network-structure/IMG_2401.jpg)

![面板](/assets/images/2017-04-30-new-home-network-structure/IMG_2402.jpg)

先将面板底座装到 86 盒上

![面板底座](/assets/images/2017-04-30-new-home-network-structure/IMG_2403.jpg)

接好线，盖上面板后的效果

![面板安装完成](/assets/images/2017-04-30-new-home-network-structure/IMG_2405.jpg)

一切安装就绪后，就可以用 UniFi Controller 来配置网络了。

# 配置网络

先到 Ubnt 官方网站将 UniFi Controller 软件下载好，并在电脑上安装。安装后，启动控制器，就可以开始配置网络了。刚开始需要配置时区信息。

![配置时区](/assets/images/2017-04-30-new-home-network-structure/setup-timezoom.png)

然后下一页就能看见设备上线了。

![选择设备](/assets/images/2017-04-30-new-home-network-structure/ap-online.png)

设置好 Wi-Fi 信息

![设置Wi-Fi](/assets/images/2017-04-30-new-home-network-structure/ap-online.png)

接下来设置好软件登陆信息就可以进入软件主界面了

![主页](/assets/images/2017-04-30-new-home-network-structure/login-page.png)

![主页](/assets/images/2017-04-30-new-home-network-structure/home-page.png)

# 总结

整套产品安装部署过程还是比较简单的，通过 UniFi Controller 可以集中管理网络中的 UniFi 设备，包括 AP 和交换机，省去了单独登陆各个设备单独进行配置的麻烦。两个 AP 在覆盖方面的表现也非常棒，房间中再也没有信号死角，而且设备在房间中移动也再也没有之前那种网络闪断的情况，这让我非常满意。

![覆盖情况](/assets/images/2017-04-30-new-home-network-structure/IMG_2415.jpg)
