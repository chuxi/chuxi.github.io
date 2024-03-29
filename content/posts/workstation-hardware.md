+++
title = "个人开发工作站配置"
date = "2023-08-28"
description = """AI开发用途的工作站配置"""

[taxonomies]
tags = [ "ai", "hardware" ]

+++

## 工作站

有两年没有关注AI的发展，近期受到`ChatGPT`的震撼，个人心血来潮就想配置一个能简单运行大模型的工作站放在家里，后续可以方便部署一套个人使用的小智能系统。除此之外还有一个原因是，个人不喜欢`apple macbook pro 16`的设计，感觉无比丑陋，所以也不想升级目前的`macbook pro 15 2019`，就打算继续用着，但考虑平时硬件性能有些方面会比较勉强，例如之前编译`LLVM` `Clang`时，轻轻松松达到了100GB存储空间，而且编译速度也不是很理想，所以需要再提高一下满足性能需求。此外，自己偶尔使用的云服务器上linux系统，需要硬件配置高时每次临时开一个，也是比较麻烦，而低硬件配置海外服务器跑几年也基本上成本不小。所以就开始考虑家里部署一套工作站，随时能远程访问，开发一些项目时，也可以利用`intellij idea remote dev`功能，还能因此让自己的macbook pro再用几年。

## 硬件配置

在综合研究京东组装服务商的产品之后，自己罗列了一套完整的配置，选择了一家店的配置再额外做了个升级。相比之下，找服务商购买，自己如果采购一系列的硬件再组装，可能成本会更高，因为这些店家都会有`OEM`优势，很多的硬件采购我们自己个人是很难买到厂家供应版本的，例如`Intel CPU 13900K`，不如就省点力气，此外，如果按照`AMD撕裂者`这种真工作站配置，成本就会急剧上升。另外如果搜索AI开发服务器，`supermicro`超微是当前领域里做得比较好的。所以自己也顺便参考了一下，看看能不能采购到超微的主板然后自己配置一套，但实际上走下来非常困难，国内采购成本很高，且不一定有售后服务。所以综合考虑下来还是直接走`intel z790 芯片组`路线。

| 组件 | 型号                     |
|:---|:-----------------------|
| CPU | Intel Core 13900k      |
| 主板 | 华硕 ProArt Z790-CREATOR |
| 内存 | 海盗船 2*32GB DDR5        |
| 显卡 | 2*24GB 4090 水冷七彩虹      |
| 硬盘 | SAMSUNG 1TB            |
| 电源 | 长城巨龙 2000w             |
| 机箱 | PHANTEKS PK620         |

这个配置中，CPU与主板基本上绑定，店家默认也不会提供`PROART Z790`主板，需要自己设置，这个主板支持双路PCIE 5.0x8，`CPU 13900k`一共也就支持20(16+4 or 2x8+4)通道PCIe，所以双显卡方案里面几乎很少有消费级主板能支持双路PCIex8，一般都是一条PCIex8，一条PCIex4，跟店家确认之后就满市场找性价比高又支持双路PCIex8的主板，除了这款也就`ROG HERO`能够支持相同的特性。此外，`PROART Z790`还有一个优势是有个万兆网卡接口，虽然家庭工作站用不起来，但也不妨碍心理上更舒服一点。

比较坑的还是在店里买的时候没有具体确认硬盘与机箱的型号，事实上硬盘应该配置2TB，默认的OEM供应都还是`SAMSUNG 980 1TB`，而机箱实际上安装两张显卡后，基本上也就没啥硬盘位置，如果想做存储则可能需要额外配置。

双显卡比较容易理解，目前的大模型看看都很大，但如果只是单张显卡24GB，很多模型都是跑的比较勉强，而且考虑4090这种级别显卡，未来可能同等配置也不容易买到，毕竟A100是真的买不起的。两张4090的配置性价比只会更高。

还有一点是整体采用了水冷，毕竟家里用，噪音会有点考虑，但实际上风扇这个东西吧，只要开起来都是有不小的声音的，自己配置也就买个心理安慰。

下单之后让店里安装`ubuntu server 22.04`系统，到手之后动手安装了显卡，开机启动，目前在家里工作运行良好。此外，家里的服务器对外没有开放端口，而是在openwrt的网关上安装了一个openvpn服务，每次只需要连接openvpn客户端之后即可访问家里的服务器，十分方便。

