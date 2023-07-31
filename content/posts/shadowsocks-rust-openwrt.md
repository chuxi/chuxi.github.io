+++
title = "shadowsocks rust openwrt配置"
date = "2023-01-05"
description = """在openwrt环境下配置shadowsocks-rust，实现透明代理上网"""

[taxonomies]
tags = [ "network", "shadowsocks", "rust" ]

+++

## Openwrt

* update 2023-07-31, 优化shadowsocks-dns使用，配置nftables tproxy

在十年前便研究过嵌入式设备上运行最小化linux系统OpenWRT，当时还是在学校跟随老师上《嵌入式原理》课程， 拿着`TP-Link WR`系列的mini路由器刷OpenWRT系统，当时也就是对计算机硬件略微了解，只是一个普普通通使用者。

如今重新开始研究这个，主要是为了彻底解决网络问题，之前原本一直借助`Shadowsocks`自己购买VPS顺顺利利上网， 做着一个老老实实本本份份忍受丢包延迟狗苟蝇营用着Google的普通人。但无奈最后这点生存空间也被剥夺了，上Google的速度越来越慢，丢包也越来越严重，一气之下，便开始了研究。

原本开源版本的[Shadowsocks](https://github.com/shadowsocks/shadowsocks)已经因为监管的原因， 作者喝茶之后，转行去做了计算机图形学的光学追踪。后来新的维护人员重新`Fork`项目[Shadowsock-libev](https://github.com/shadowsocks/shadowsocks-libev)，核心库采用`C`语言维护，再后来因为维护这个项目实在是太累，因为C语言编程项目在模块化编程以及生态库这方面全靠代码拷贝，所以新的作者又开发了[Shadowsocks-rust](https://github.com/shadowsocks/shadowsocks-rust)，目前该项目已经成为主要的`Shadowsocks`版本。另外，此处特殊注明一下，该文章写的东西全都是无效的，只是个人研究使用。

这不妨碍这篇文章的目的，只是在技术上探讨一下如何实现个人的网络安全。当然花钱买第三方网络服务才是最简单的解决方案，比如`Clash`， `V2Ray`， `ExpressVPN`等等。其实最刺激群众购买这些软件服务的，还是大家的炒币需求，只要稍稍想正经一点的数字货币交易，就不可能在自己国家允许交易，兔子不吃窝边草，韭菜还是国外的香。但这种第三方存在一个安全问题，涉及到中间人劫持以及很多的技术细节，如果服务商想看数据消息，基本上没有能逃过的。所以在这篇文章里继续强调，我们研究的目的是为了网络安全，绝对不是自由，派出所请不要找我喝茶。

回到正题，这篇文章主要记录一下在`NanoPi`的路由器`R2C-Plus`设备上部署搭建`Shadowsocks-Rust`的过程，也希望帮助一下其他人，避开相同的错误，毕竟很多问题在嵌入式开发里，需要搭进去不少的青春岁月。

## NanoPi

`NanoPi`是一款ARM开发板的系列产品，其中包括非常有名的`R2S`，`R4S`等等产品，就是开发板包了一个外壳当`软路由器`卖。这个产品的公司有一个非常公益的名字《友善之家》，每次看到都会浮现出杭州路边的《城管驿站》。经常有人会困惑于如何选择合适的`软路由器`，其实关键的核心在于成本与功耗，其他一些花里胡哨的功能，比如`docker`，真的没什么必要。所以目标200块钱上下，满足千兆网络上下数据转发顺顺利利，也就足够家庭使用了。而功耗这方面，`x86_64`处理器，只能说非常不适合，跑多了还得搞个风扇散热。综合之下，290块钱买了`R2C-Plus`这款路由器，跟`R2S`配置多了一个8GB固定存储，省下再购买TF存储卡的钱。这里说一下也是觉得该公司产品非常不错，定位非常之准确，算是比较下来众多路由器产品里性价比最合适的。

另外一个需求是能够运行`Shadowsocks-Rust`，`Rust`本身的跨平台编译是支持嵌入式设备开发的，所以只需要能够运行openwrt， 以及硬件设备的CPU指令集不是特别离谱，就应该能编译，所以看`R2C-Plus`采用`ARM64`架构，也就不会有太大的问题。

在通电之后，根据[R2C-Plus wiki文档](https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R2C_Plus/zh)， 稍稍熟悉之后，便能够开始完成一些基础的配置操作。本来买个路由器，有一些预装的功能服务，但如果个人没什么必要的话也是可以 直接删除，避免占用路由器硬件资源。

整个过程可以分为：
1. 编译shadowsocks-rust `-target aarch64-unknown-linux-musl -features local-redir,local-dns`
2. 直接解压安装，启动 `local-dns`, `local-redir`，不需要打包成 `ipk`
3. 配置透明代理 nftables tproxy，此处建议OpenWrt系统升级到22，使用nftables配置防火墙规则

`R2S-Plus`官方版本的OpenWrt比较稳定，但使用较长一段时间后遇到过断网的情况，后来直接升级最新的稳定版本解决，这个过程中先做好原系统备份，再操作升级修复。毕竟OpenWrt属于开源系统，厂家能一直更新迭代，相比于其他自然生长的路由设备已经好了很多。

## 编译openwrt可执行`shadowsocks-rust`

目前个人使用的[shadowsocks-rust fork](https://github.com/chuxi/shadowsocks-rust/tree/1.15.2-fix)，可以基于此版本编译

在配置`Rust`开发环境之后，可以参考`.github/workflows/build-release.yml`文件，编译目标平台的可执行文件。
此处针对`R2C-Plus`以及整个系列的产品，目标平台均为`aarch64-unknown-linux-musl`。

`./build/build-release -t aarch64-unknown-linux-musl -f local-redir,local-dns,security-replay-attack-detect,security-iv-printable-prefix`

注意该脚本使用[cross](https://github.com/cross-rs/cross)实现跨平台交叉编译打包，需要预先安装[docker](https://docs.docker.com/engine/)。

此处暂不考虑集成到OpenWRT的包管理系统内，所以可以直接使用打包好的`tar.xz`文件，解压后执行对应的`sslocal`即可。注意在路由器上，需要配置`redir`与`dns`功能搭配使用。其中`security-replay-attack-detect`非常重要，VPS服务器会经常收到探测的连接数据包，如果不在配置中打开`重放攻击`防护功能，则很容易被黑名单而无法连接。至于`security-iv-printable-prefix`，被社区从`master`版本内移除，可能是当时开发人员利益相关的缘故，个人认为算是一个错误的修改。但这个特性非常有用，所以自己就在个人版本内保留了下来，并且做了改进优化，具体可以看[commit](https://github.com/chuxi/shadowsocks-rust/commit/4c59a6ac6b54c7ce4c8c50dd213cb2824f19c82b).

如此编译的版本已经可以正常使用`socks5`与`http[s]`代理功能，但如果在路由设备上则需要更进一步。在个人笔记本或者手机上，解析一个网页或者软件的网络请求时，需要先解析域名`DNS`获取`IP`，再进一步将请求发送到服务器。国内主要无法上海外网站的原因，来自于请求解析域名`DNS`时，某个传说中的势力，返回了一个无法连接的假`IP`,即使配置了正确的域名解析服务器，也会在中间链路层串改其中返回的`IP`信息。如果是`Socks5`代理模式，则请求会直接转发到代理程序，包括目标的域名，所以不需要单独处理域名解析`DNS`问题。

## `local-redir`与`local-dns`

`local-redir`是将收到的数据包，转发到国外VPS上的`ssserver`，所以路由器配置之后能够直接转发数据包。但此处路由器收到数据包时，是根据`IP`完成路由转发的。

`local-dns`根据ACL模式，接收到解析域名的请求时，提取出其中的域名部分，根据ACL列表的配置，决定本地`DNS`服务器获取解析结果，还是转发到`ssserver`服务代理解析。

因为`Shadowsocks-Rust`支持多模块同时运行，所以两个功能可以一个进程完成启动。此处有非常多的细节，一个不小心便可能懵逼一整晚。

创建`shadow.sh`脚本

```bash
root@FriendlyWrt:~# cat shadow.sh
#!/bin/bash

DIR=$( cd $( dirname $0 ) && pwd )
cd "$DIR"

# 停止之前的sslocal进程
cat ss.pid | xargs kill -15 || true

./sslocal -c config.json -d --daemonize-pid ss.pid
```

1. `-c config.json` 配置文件
2. `-d --daemonize-pid ss.pid` 保存启动后的`sslocal`守护进程id

此外，`config.json`配置文件内，同时配置`redir`与`dns`本地服务

```json
{
  "locals": [
    {
      "local_address": "::",
      "local_port": 60080,
      "protocol": "redir",
      "tcp_redir": "tproxy",
      "udp_redir": "tproxy",
      "mode": "tcp_and_udp"
    },
    {
      "local_address": "::",
      "local_port": 5333,
      "protocol": "dns",
      "local_dns_address": "114.114.114.114",
      "local_dns_port": 53,
      "remote_dns_address": "1.1.1.1",
      "remote_dns_port": 53,
      "mode": "udp_only"
    }
  ],


  "servers": [
    {
      "server": "server ip",
      "server_port": 8080,
      "password": "password",
      "method": "aes-256-gcm"
    }
  ],

  "timeout": 5,
  "keep_alive": 5,
  "nofile": 51200,
  "ipv6_first": false,
  "ipv6_only": false,
  "acl": "shadowsocks.acl",
  "outbound_fwmark": 255,

  "log": {
    "config_path": "log4rs.yml"
  },

  "runtime": {
    "mode": "multi_thread",
    "worker_count": 3
  }
}
```

注意替换 `servers`内服务器信息即可

1. `locals`里`redir`的`tcp_redir`需要配置`tproxy`，当前透明代理基本采用tproxy模式
2. `locals`下`dns`的`local_port`配置端口5333，可以由`dnsmasq`配置指定域名转发到`local-dns`端口，也可以直接配置使用53端口
3. `nofile`配置注意不要超过系统支持最大值，`cat /proc/sys/fs/file-max`查看
4. `security.replay_attack.policy`是必须强调的一个配置，建议在`ssserver`上启动，此处local本身没太大必要
5. `log.config_path`的配置文件[log4rs.yml](https://github.com/shadowsocks/shadowsocks-rust/blob/master/configs/log4rs.yaml)
6. `runtime.worker_count`，根据处理器最大线程数量-1配置
7. `local_address`如果绑定失败，可以使用`0.0.0.0`替换，避免启动ipv6
8. `dns`服务的`mode`指定`udp_only`，更加稳定

`dns`的udp数据包会由系统本身转发到本地的53端口处理，OpenWRT默认使用`dnsmasq`代理解析域名，即`53`端口已经被占用，所以`sslocal`启动时可能退出。检查`网络->DHCP/DNS->高级设置->DNS服务器端口`，修改成其他端口，比如`153`，保存并应用。`dns`建议使用`udp_only`，`tcp`连接在服务端获取解析信息后会自动断开连接，此时在`local-dns`内对于客户端连接有缓存，但该连接已经失效，所以会造成系统不断生成新的`tcp`连接，在一段时间后非常影响系统性能。但此外，注意部署的`server`需要配置`"mode":"tcp_and_udp"`，同时检查iptables或者nftables已经打开了对应端口的`udp`协议。可以通过本地命令检查`udp`是否可以连接

```shell
dig @127.0.0.1 www.google.com -p 5333
```

## 透明代理 TProxy

路由器从网线硬件上接收到数据包后，会根据`netfilter`，也就是防火墙规则转发数据包。利用用户层面工具`iptables`与`ip6tables`，配置数据转发的规则。透明代理就是截取流过路由器的数据包，转发给上面运行的`redir`代理程序。

注意此处已经将路由器建立成`dns`服务器，重点是`tproxy`配合`redir`，针对`dns`部分数据包，不再需要配置转发，而普通请求数据，根据`ip`地址，在`netfilter`中转发。

首先确定已经安装`iptables-mod-tproxy`，一般都是已经安装`opkg install iptables-mod-tproxy`，或者nftables下安装`kmod-nft-tproxy`与`kmod-nft-socket`，参考[配置透明代理规则](https://guide.v2fly.org/app/tproxy.html#%E9%85%8D%E7%BD%AE%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86%E8%A7%84%E5%88%99)

如果在`sslocal`进程中开启`debug`日志后，发现还是没有接收到数据，可能的两个配置是
1. `cat /etc/sysctl.d/10-default.conf`查看配置`net.ipv4.ip_forward=1`，ipv6同理
2. `cat /etc/sysctl.d/11-br-netfilter.conf`查看配置`net.bridge.bridge-nf-call-iptables=0`

尤其是第二个配置，`net.bridge.bridge-nf-call-iptables=1`或者`net.bridge.bridge-nf-call-ip6tables=1`会导致数据在流过`PREROUTING`后，转向`FORWARD`，而正确的情况应该是转入`INPUT`链表，最后进入代理应用程序。具体这也是个2009年的旧[bug](https://bugzilla.redhat.com/show_bug.cgi?id=512206)。

此外，可以执行`netstat -anptu | grep sslocal`，若是打开的连接数量过多，可能导致无法创建新的连接，或者上网卡顿，此时可以修改以下参数
1. `/etc/sysctl.d/31-optmize-proxy.conf`内配置`net.ipv4.tcp_fin_timeout = 5`
2. `/etc/sysctl.d/31-optmize-proxy.conf`内配置`net.ipv4.tcp_keepalive_time = 15`

原有参数关闭`tcp`连接较慢，产生大量连接后无法及时释放文件，导致路由器卡顿。`/etc/sysctl.d/10-default.conf`内也有该配置参数，可以一并修改，修改配置之后重启生效即可

## 关于shadowsock-rust.ipk

可以看到在`shadowsocks-rust`页面内推荐参考项目[openwrt-shadowsocks-rust](https://github.com/honwen/openwrt-shadowsocks-rust)，实际上如果不是分发便于安装，可以不需要打包`ipk`格式文件，跳过这个过程至少节约了`openwrt sdk`下载配置编译过程。再是搭配[luci-app-shadowsocks-rust](https://github.com/honwen/luci-app-shadowsocks-rust)，在查看该项目内的`ss-rule`构建透明代理的过程，可以借鉴优化，直接引入`iptables_tproxy.sh`，而`/etc/init.d/shadowsocks`文件过于臃肿，也可以适当简化。这里如果有可视化配置页面，就可以极大减少普通用户的学习曲线。

记录一下自己配置的另外一套管理脚本`/etc/init.d/shadow`

```shell
#!/bin/sh /etc/rc.common

START=90
STOP=10

USE_PROCD=1

start_service() {
  procd_open_instance shadow
  procd_set_param command /root/shadow/shadow.sh
  procd_set_param respawn ${respawn_threshold:-3600} ${respawn_timeout:-5} ${respawn_retry:-5}
  procd_set_param stdout 1
  procd_set_param stderr 1
  procd_set_param pidfile /var/run/shadow.pid
  procd_close_instance
}

```

此处调用脚本`/root/shadow/shadow.sh`，用于切换工作目录

```shell
#!/bin/bash

DIR=$( cd $( dirname $0 ) && pwd )
cd "$DIR"

./sslocal -c config.json
```

## Summary

事实上，`shadowsocks-rust`是非常优秀的项目，作者在日常的`issues`维护中，不断耐心地帮助他人，项目本身代码设计，除了兼容老版本的`shadowsocks`外，架构也设计得非常优美，模块化实现不同代理配置功能，能够快速支持开发新的功能。

当然，直接使用现在发布的`shadowsocks-rust`包可能无法顺利访问一些网站，一般是需要额外配置插件，以及`ssserver`开启`security.replay_attack`重放攻击保护。

Reference:
1. [Linux之iptables防火墙](https://juejin.cn/post/7090181016135925796)
2. [记一次OpenWrt TPROXY透明代理踩坑之旅](http://www.hackdig.com/02/hack-597366.htm)
3. [bridge-nf](https://feisky.gitbooks.io/sdn/content/linux/params.html#bridge-nf)
3. [第一篇万字长文：围绕透明代理的又一次探究](https://moecm.com/something-about-v2ray-with-tproxy/)
4. [为 V2Ray 配置透明代理](https://guide.v2fly.org/app/tproxy.html#%E9%85%8D%E7%BD%AE%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86%E8%A7%84%E5%88%99)

