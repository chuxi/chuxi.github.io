+++
title = "shadowproxy for openwrt"
date = "2024-01-03"
description = """基于shadowsocks-rust redir&dns开发的openwrt的透明代理软件"""

[taxonomies]
tags = [ "network", "shadowsocks", "rust" ]
+++

## 介绍

在`shadowsocks-rust openwrt`部署过程中，需要大量的手工配置，十分繁琐，所以为了将整个流程自动化以及配置文件可视化，在openwrt平台上开发了项目`shadowproxy`，便于快速搭建部署openwrt的透明代理服务。

目前该服务在个人设备上，家庭内全局代理模式下，运行十分流畅，所以分享出来，同时禁止任何非法使用。

## 安装

在[shadowproxy](https://github.com/9566618/shadowproxy)下载最新的发布版本，通过openwrt软件包页面安装，或者使用命令 `opkg install xxx.ipk` 。

该项目包含已经编译好的执行文件，与 `shadowsocks-rust` 开源项目相比，默认采用 `local-dns,local-redir,security-replay-attack-detect`，以及部分优化，若有安全顾虑可以使用官方源代码自行编译。

## 配置

除默认配置项目外，用户仅需配置 `servers` 一项，增加自己的服务器信息，并且勾选启用。其他配置项目参考官方文档配置即可。

在ACL配置中，默认域名解析均不会通过代理服务器，可以通过ACL配置内增加域名，将该域名加入代理名单。 

在启动过程中， `shadowproxy`会自动配置路由信息，`dnsmasq`转发，以及`nftables` 透明路由，整个流程非常简单，易于使用。

## 优势

相比于当前一些比较流行的透明代理配置，比如 Clash，V2ray等，基于 `shadowsocks-rust` 本身协议，能最轻量化部署透明代理服务。

同时该配置十分简单，易于使用，相比于原官方的 `shadowsocks-rust` 配置，大大缩小需要设置的项目，能及时更新代理域名列表，实现ACL分流。

出于个人强迫症心理，不喜欢使用旁路由模式，认为应当有自由的权利，通过方便的分流配置，实现完全无感知的透明代理上网。

## 扩展

该透明代理部署能够搭配 `OpenVPN` + `DDNS` 服务，支持所有平台设备代理上网，即在家里部署该透明代理，手机笔记本等在外面时，通过 `OpenVPN` 访问家里的 `OpenWrt`网关设备，即可实现代理科学上网。该模式下能将一个网关设备分享给众多的其他设备使用，对于公司组织等提供极大的便利。

## 捐赠

目前个人使用服务器节点部署在 `vultr`上，如果感兴趣可以通过下面的推荐码注册账号并且部署服务器，后续该项目也会完善服务器端一键部署功能。

通过[Vultr邀请码](https://www.vultr.com/?ref=6849375)注册，我可以获得$35的服务器额度，而新用户可以获得价值$100的优惠券，有问题也可以在github上提交issue。

## Reference

1. [OpenVPN Server on OpenWrt](https://openwrt.org/docs/guide-user/services/vpn/openvpn/server)
2. [OpenVPN Client](https://openvpn.net/client/)
3. [DDNS Cloudflare](https://p3terx.com/archives/openwrt-cloudflare-ddns.html)
