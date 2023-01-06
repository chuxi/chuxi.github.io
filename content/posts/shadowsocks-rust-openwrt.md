+++
title = "shadowsocks rust openwrt配置"
date = "2023-01-05"
description = """在openwrt环境下配置shadowsocks-rust，实现透明代理上网"""

[taxonomies]
tags = [ "network", "shadowsocks", "rust" ]

+++

## Openwrt

在十年前便研究过嵌入式设备上运行最小化linux系统OpenWRT，当时还是在学校跟随翁恺老师上《嵌入式原理》课程，
拿着`TP-Link WR`系列的mini路由器刷OpenWRT系统，当然，玩到最后还是喜新厌旧之下，不再继续折腾。

回头如今重新开始摆弄这个玩具，倒也是有点无聊所致。之前原本一直借助`Shadowsocks`自己购买VPS顺顺利利上网，
做着一个老老实实本本份份忍受丢包延迟狗苟蝇营用着Google的普通人。但无奈最后这点生存空间也被剥夺
了，上Google的速度越来越慢，丢包也越来越严重，一怒之下，决定彻底摆脱这种困境，做起网络工程师的工作。

原本开源版本的[Shadowsocks](https://github.com/shadowsocks/shadowsocks)已经因为监管的原因，
作者喝茶之后，转行去做了计算机图形学的光学追踪，写这文章的时候应该还在游戏行业构建3D。后来新的维护人员重新
`Fork`项目[Shadowsock-libev](https://github.com/shadowsocks/shadowsocks-libev)，核心库采用
`C`语言维护，再后来因为维护这个项目实在是太累，因为C语言编程项目在模块化编程以及生态库这方面全靠代码拷贝，
所以新的作者又开发了[Shadowsocks-rust](https://github.com/shadowsocks/shadowsocks-rust)，目前该
项目已经成为主要的`Shadowsocks`版本。这长段的历史也是告诉大家一个道理，最好在国外做国内网络软件工程师的工作。

这不妨碍这篇文章的目的，只是在技术上探讨一下如何实现个人的网络自由，避免监管。当然花钱买第三方网络服务才是
最简单的解决方案，比如`Clash`， `V2Ray`， `ExpressVPN`等等。其实最刺激群众购买这些软件服务的，还是大家的
炒币需求，只要稍稍想正经一点的数字货币交易，就不可能在自己国家允许交易，兔子不吃窝边草。这件事如同国家之间的
《囚徒困境》，总有一方想要以此获益，当然，这其中韩国的韭菜最绿。

回到正题，这篇文章主要记录一下在`NanoPi`的路由器`R2C-Plus`设备上部署搭建`Shadowsocks-Rust`的过程，
也希望帮助一下其他人，避开相同的错误，毕竟很多问题在嵌入式开发里，需要搭进去不少的青春岁月。

## NanoPi

`NanoPi`是一款ARM开发板的系列产品，其中包括非常有名的`R2S`，`R4S`等等产品，就是开发板包了一个外壳当`软路由器`卖。
这个产品的公司有一个非常公益的名字《友善之家》，每次看到都会浮现出杭州路边的《城管驿站》。经常有人会困惑于如何选择
合适的`软路由器`，其实关键的核心在于成本与功耗，其他一些花里胡哨的功能，比如`docker`，真的没什么必要。
所以目标200块钱上下，满足千兆网络上下数据转发顺顺利利，也就足够家庭使用了。而功耗这方面，`x86_64`处理器，
只能说非常不适合，跑多了还得搞个风扇散热。综合之下，290块钱买了`R2C-Plus`这款路由器，跟`R2S`配置多了一个8GB固定存储，
省下再购买TF存储卡的钱。这里说一下也是觉得该公司产品非常不错，定位非常之准确，算是比较下来众多路由器产品里性价比最合适的。

另外一个需求是能够运行`Shadowsocks-Rust`，`Rust`本身的跨平台编译是支持嵌入式设备开发的，所以只需要能够运行openwrt，
以及硬件设备的CPU指令集不是特别离谱，就应该能编译，所以看`R2C-Plus`采用`ARM64`架构，也就不会有太大的问题。

在通电之后，根据[R2C-Plus wiki文档](https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R2C_Plus/zh)，
稍稍熟悉之后，便能够开始完成一些基础的配置操作。本来买个路由器，有一些预装的功能服务，但如果个人没什么必要的话也是可以
直接删除，避免占用路由器硬件资源。

整个过程可以分为：
1. 编译shadowsocks-rust `-target aarch64-unknown-linux-musl -features local-redir,local-dns`
2. 直接解压安装，启动 `local-dns`, `local-redir`，不需要打包成 `ipk`
3. 配置透明代理 iptables tproxy

## 编译openwrt可执行`shadowsocks-rust`

在配置`Rust`开发环境之后，可以参考`.github/workflows/build-release.yml`文件，编译目标平台的可执行文件。
此处针对`R2C-Plus`以及整个系列的产品，目标平台均为`aarch64-unknown-linux-musl`。

`./build/build-release -t aarch64-unknown-linux-musl -f local-redir,local-dns`

注意该脚本依赖[cross](https://github.com/cross-rs/cross)实现跨平台交叉编译打包，
该程序依赖[docker](https://docs.docker.com/engine/)

此处暂不考虑集成到OpenWRT的包管理系统内，所以可以直接使用打包好的`tar.xz`文件，解压后执行对应的`sslocal`即可。
注意在路由器上，需要配置`redir`与`dns`功能搭配使用。

在个人笔记本或者手机上，解析一个网页或者软件的网络请求时，需要先解析域名`DNS`获取`IP`，再进一步将请求发送到服务器。
国内主要无法上海外网站的原因，来自于请求解析域名`DNS`时，某个传说中的势力，返回了一个无法连接的假`IP`,即使配置了
正确的域名解析服务器，也会在中间链路层串改其中返回的`IP`信息。如果是`Socks5`代理模式，则请求会直接转发到代理程序，
包括目标的域名，所以不需要单独处理域名解析`DNS`问题。

## `local-redir`与`local-dns`

`local-redir`是将收到的数据包，转发到国外VPS上的`ssserver`，所以路由器配置之后能够直接转发数据包。但此处路由器
收到数据包时，是根据`IP`完成路由转发的。

`local-dns`根据ACL模式，接收到解析域名的请求时，提取出其中的域名部分，根据ACL列表的配置，决定本地`DNS`服务器
获取解析结果，还是转发到`ssserver`服务代理解析。

因为`Shadowsocks-Rust`支持多模块同时运行，所以两个功能可以一个进程完成启动。此处有非常多的细节，一个不小心便可能
懵逼一整晚。

创建`shadow.sh`脚本，方便重启服务

```bash
root@FriendlyWrt:~# cat shadow.sh
#!/bin/bash

DIR=$( cd $( dirname $0 ) && pwd )
cd "$DIR"

# 停止之前的sslocal进程
cat ss.pid | xargs kill -15 || true

sslocal -c shadowsocks.json -d --daemonize-pid ss.pid --acl shadowsocks.acl --outbound-fwmark 255
```

1. `-c shadowsocks.json` 配置文件
2. `-d --daemonize-pid ss.pid` 保存启动后的`sslocal`守护进程id
3. `--acl shadowsocks.acl` 配置`ACL`文件，该文件能有效配置分流
4. `--outbound-fwmark 255` 给所有`sslocal`发出的数据包标记`255`，与透明代理配置配合

注意`ACL`与`fwmark`配置需要命令行方式传入，配置文件内不生效，而`ACL`文件生成可以执行脚本
[genacl_proxy_gfw_bypass_china_ip.py](https://github.com/shadowsocks/shadowsocks-rust/blob/master/acl/genacl_proxy_gfw_bypass_china_ip.py)

此外，`shadowsocks.json`配置文件内，同时配置`redir`与`dns`本地服务

```json
{
  "locals": [
    {
      "local_address": "0.0.0.0",
      "local_port": 60080,
      "protocol": "redir",
      "tcp_redir": "tproxy",
      "udp_redir": "tproxy"
    },
    {
      "local_address": "::",
      "local_port": 53,
      "protocol": "dns",
      "local_dns_address": "114.114.114.114",
      "local_dns_port": 53,
      "remote_dns_address": "1.1.1.1",
      "remote_dns_port": 53
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
  "udp_timeout": 30,
  "udp_max_associations": 128,

  "security": {
    "replay_attack": {
      "policy": "reject"
    }
  },

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

1. `locals`里`redir`的`tcp_redir`需要配置`tproxy`，当前透明代理基本采用iptables tproxy模式
2. `locals`下`dns`的`local_port`配置端口53，替换默认的`dnsmasq`解析服务
3. `nofile`配置注意不要超过系统支持最大值，`cat /proc/sys/fs/file-max`查看
4. `security.replay_attack.policy`是必须强调的一个配置，建议在`ssserver`上启动，此处local本身没太大必要
5. `log.config_path`的配置文件[log4rs.yml](https://github.com/shadowsocks/shadowsocks-rust/blob/master/configs/log4rs.yaml)
6. `runtime.worker_count`，根据处理器最大线程数量-1配置
7. `local_address`如果绑定失败，可以使用`0.0.0.0`替换，避免启动ipv6

`dns`的udp数据包会由系统本身转发到本地的53端口处理，如果没有检查`/etc/resolv.conf`内配置。
OpenWRT默认使用`dnsmasq`代理解析域名，即`53`端口已经被占用，所以`sslocal`启动时可能退出。
检查`网络->DHCP/DNS->高级设置->DNS服务器端口`，修改成其他端口，比如`153`，保存并应用。

## 透明代理 TProxy

路由器从网线硬件上接收到数据包后，会根据`netfilter`，也就是防火墙规则转发数据包。利用用户层面工具`iptables`与
`ip6tables`，配置数据转发的规则。透明代理就是截取流过路由器的数据包，转发给上面运行的`redir`代理程序。

注意此处已经将路由器建立成`dns`服务器，重点是`tproxy`配合`redir`，针对`dns`部分数据包，不再需要配置转发，
而普通请求数据，根据`ip`地址，在`netfilter`中转发。

首先确定已经安装`iptables-mod-tproxy`，一般都是已经安装`opkg install iptables-mod-tproxy`

`shadowsocks-rust`提供的[iptables_tproxy.sh](https://github.com/shadowsocks/shadowsocks-rust/blob/master/configs/iptables_tproxy.sh)。
此处去掉一些不必要配置，提供一份简单的`iptables_tproxy.sh`路由配置稍做说明

```bash
#!/bin/bash

iptables-save | grep -v shadowsocks- | iptables-restore
ip6tables-save | grep -v shadowsocks- | ip6tables-restore

### IPv4 RULES

# Create chnip ipset，中国ip地址范围库
ipset create chnip hash:net family inet -exist
ipset restore < chnip

SHADOWSOCKS_REDIR_IP=0.0.0.0
SHADOWSOCKS_REDIR_PORT=60080

readonly IPV4_RESERVED_IPADDRS="\
0/8 \
10/8 \
100.64/10 \
127/8 \
169.254/16 \
172.16/12 \
192/24 \
192.0.2.0/24 \
192.88.99/24 \
198.18/15 \
192.168/16 \
198.51.100/24 \
203.0.113/24 \
224/4 \
240/4 \
255.255.255.255/32 \
！！！增加 shadowsocks.json 内 servers 的地址
"

## TCP+UDP
# Strategy Route
ip -4 rule del fwmark 0x01 table 803
ip -4 rule add fwmark 0x01 table 803
ip -4 route del local 0.0.0.0/0 dev lo table 803
ip -4 route add local 0.0.0.0/0 dev lo table 803

# TPROXY for LAN
iptables -t mangle -N shadowsocks-tproxy
# Skip LoopBack, Reserved
for addr in ${IPV4_RESERVED_IPADDRS}; do
   iptables -t mangle -A shadowsocks-tproxy -d "${addr}" -j RETURN
done

# UDP: Bypass CN IPs
iptables -t mangle -A shadowsocks-tproxy -m set --match-set chnip dst -p udp -j RETURN
# TCP: Bypass CN IPs
iptables -t mangle -A shadowsocks-tproxy -m set --match-set chnip dst -p tcp -j RETURN

# UDP: TPROXY UDP to 60080
iptables -t mangle -A shadowsocks-tproxy -p udp -j TPROXY --on-port ${SHADOWSOCKS_REDIR_PORT} --tproxy-mark 0x01/0x01
# TCP: TPROXY TCP to 60080
iptables -t mangle -A shadowsocks-tproxy -p tcp -j TPROXY --on-port ${SHADOWSOCKS_REDIR_PORT} --tproxy-mark 0x01/0x01


# TPROXY for Local
iptables -t mangle -N shadowsocks-tproxy-mark
# Skip LoopBack, Reserved
for addr in ${IPV4_RESERVED_IPADDRS}; do
   iptables -t mangle -A shadowsocks-tproxy-mark -d "${addr}" -j RETURN
done

# Bypass CN IPs
iptables -t mangle -A shadowsocks-tproxy-mark -m set --match-set chnip dst -j RETURN

# Bypass sslocal's outbound data
iptables -t mangle -A shadowsocks-tproxy-mark -m mark --mark 0xff/0xff -j RETURN
# UDP: Set MARK and reroute
iptables -t mangle -A shadowsocks-tproxy-mark -p udp -j MARK --set-xmark 0x01/0xffffffff
# TCP: Set MARK and reroute
iptables -t mangle -A shadowsocks-tproxy-mark -p tcp -j MARK --set-xmark 0x01/0xffffffff

# Apply TPROXY to LAN
iptables -t mangle -A PREROUTING -p udp -j shadowsocks-tproxy
iptables -t mangle -A PREROUTING -p tcp -j shadowsocks-tproxy
# Apply TPROXY for Local
iptables -t mangle -A OUTPUT -p udp -j shadowsocks-tproxy-mark
iptables -t mangle -A OUTPUT -p tcp -j shadowsocks-tproxy-mark
```

1.chnip文件是中国大陆ipv4地址库，可以由[genipset.py](https://github.com/shadowsocks/shadowsocks-rust/blob/master/configs/genipset.py)
脚本生成，通过该方式，绝大多数国内`ip`均不会走代理，而是直接路由器转发到上层，虽然`redir`已经支持`ACL`，
配置大多数ip地址，但在内核路由层面提前转发，能提高效率
2. `IPV4_RESERVED_IPADDRS` 是本地网关以及局域网配置地址，当目标地址是这个范围时，直接返回，避免转发到`redir`上
3. `shadowsocks-tproxy-mark -m mark --mark 0xff/0xff -j RETURN`这句配置十分重要，配合前面设置的`fwmark`，
能将`sslocal`产生的数据避免回传到`PREROUTING`链上，被重新路由。如果不配置，最直接的就是路由器内存耗尽。
4. 注意如果需要支持ipv6，那需要设置两套tproxy，且ipv4与ipv6的redir采用不同的端口号，例如60081，具体参考官方的
`iptables_tproxy.sh`文件，此处仅做参考

针对以上配置命令说明可以参考其他的文档，可以先熟悉`netfilter`运行方式，再理解`iptables`设计逻辑，
否则很容易绕晕在其中。

此外，可以通过命令`iptables -t mangle -vL`，跟踪数据流向，判断该路由配置是否生效。

如果在`sslocal`进程中开启`debug`日志后，发现还是没有接收到数据，可能的两个配置是
1. `cat /etc/sysctl.d/10-default.conf`查看配置`net.ipv4.ip_forward=1`，ipv6同理
2. `cat /etc/sysctl.d/11-br-netfilter.conf`查看配置`net.bridge.bridge-nf-call-iptables=0`

尤其是第二个配置，`net.bridge.bridge-nf-call-iptables=1`或者`net.bridge.bridge-nf-call-ip6tables=0`
会导致数据在流过`PREROUTING`后，转向`FORWARD`，而正确的情况应该是转入`INPUT`链表，最后进入代理应用程序。
具体这也是个2009年的旧[bug](https://bugzilla.redhat.com/show_bug.cgi?id=512206)。

## 关于shadowsock-rust.ipk

可以看到在`shadowsocks-rust`页面内推荐参考项目[openwrt-shadowsocks-rust](https://github.com/honwen/openwrt-shadowsocks-rust)，
实际上如果不是分发便于安装，可以不需要打包`ipk`格式文件，跳过这个过程至少节约了`openwrt sdk`下载配置编译过程，
毕竟尝试过之后才知道啥叫做用力过猛，再次感谢`Rust`的跨平台编译功能。

再是搭配[luci-app-shadowsocks-rust](https://github.com/honwen/luci-app-shadowsocks-rust)，在查看
该项目内的`ss-rule`构建透明代理的过程，可以借鉴优化，直接引入`iptables_tproxy.sh`，
而`/etc/init.d/shadowsocks`文件过于臃肿，也可以适当简化。这里如果有可视化配置页面，就可以极大减少普通用户的学习曲线。

## Summary

事实上，`shadowsocks-rust`是非常优秀的项目，作者在日常的`issues`维护中，不断耐心地帮助他人，项目本身代码设计，
除了兼容老版本的`shadowsocks`外，架构也设计得非常优美，模块化实现不同代理配置功能，能够快速支持开发新的功能。

当然，直接使用现在发布的`shadowsocks-rust`包可能无法顺利使用，一般是需要额外配置插件，以及`ssserver`开启
`security.replay_attack`重放攻击保护。

最后，不得不承认，中国可能是熟悉网络的程序员最多的国家。

Reference:
1. [Linux之iptables防火墙](https://juejin.cn/post/7090181016135925796)
2. [记一次OpenWrt TPROXY透明代理踩坑之旅](http://www.hackdig.com/02/hack-597366.htm)
3. [bridge-nf](https://feisky.gitbooks.io/sdn/content/linux/params.html#bridge-nf)
3. [第一篇万字长文：围绕透明代理的又一次探究](https://moecm.com/something-about-v2ray-with-tproxy/)
4. [为 V2Ray 配置透明代理](https://guide.v2fly.org/app/tproxy.html#%E9%85%8D%E7%BD%AE%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86%E8%A7%84%E5%88%99)

