# docker 中运行 openwrt

[toc]

# 思路
利用 `macvlan` 方式创建虚拟接口进行配置。
有感于来自恩山 betterman 及 rightwifi2017 两位大佬斐讯 N1 的玩法，也获得两位大佬的帮助，在此感谢两位大佬。
由于 N1 为单网卡，所以配置只能为单臂路由，本案为双网卡 `opewnrt`

机器拥有双网卡: `enp1s0` 及 `enp3s0` ，本案将 `enp3s0` 用作 `LAN` 口，`enp1s0` 用作 `WAN` 口。
# 0. 安装 docker
```
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```
# 1. 配置系统环境
```
#打开网卡混杂模式
ip link set enp1s0 promisc on
ip link set enp3s0 promisc on

#加载 PPPOE 内核模块
modprobe pppoe
````
# 2.docker 网络配置
```
# 为 docker 创建 macvlan 虚拟接口，并链接到 host 网卡
# LAN 口
docker network create -d macvlan \
    --subnet=10.1.1.0/24 --gateway=10.1.1.254 \
    --ipv6 --subnet=fe80::/16 --gateway=fe80::1 \
    -o parent=enp3s0 \
    -o macvlan_mode=bridge \
    macvLan
# WAN 口
docker network create -d macvlan \
    --subnet=192.168.254.0/24 --gateway=192.168.254.1 \
    --ipv6 --subnet=fe81::/16 --gateway=fe81::1 \
    -o parent=enp1s0 \
    -o macvlan_mode=bridge \
    macvWan
```
# 3. 创建容器
```
#导入镜像
docker import https://downloads.openwrt.org/releases/19.07.0/targets/x86/64/openwrt-19.07.0-x86-64-generic-rootfs.tar.gz openwrt:19.07.0

#创建并启动容器
docker run -d \
    --restart unless-stopped \
    --network macvLan \
    --privileged \
    --name openwrt \
    openwrt:19.07.0 \
    /sbin/init

#将第二网卡的 macvlan 挂接到 openwrt
docker network connect macvWan openwrt
```
# 4. 配置 openwrt
```
#进入容器
docker exec -it openwrt /bin/sh

#编辑 / etc/config/network
config interface 'lan'
        option type 'bridge'
        option ifname 'eth0'   # 需要与 docker netwrok 中的虚拟接口匹配（macvLan）
        option proto 'static'
        option ipaddr '10.1.1.254'
        option netmask '255.255.255.0'
        option ip6assign '60'

config interface 'wan'
        option ifname 'eth1'  # 需要与 docker netwrok 中的虚拟接口匹配（macvWan）
        option proto 'dhcp'
        option ip6assign '60'

#重启 openwrt 网络
/etc/init.d/network restart
```
# 5. 宿主机出口
由于 `docker` 网络采用 `macvlan` 的 `bridge` 模式，即使宿主机与容器在同一网段，相互之间也是无法通信的。
为了解决这个问题，需利用多个 `macvlan` 接口之间是互通的原理，在 `LAN` 口新建一个 `macvlan` 虚拟接口：

```
# 使用 ip 命令
ip link add link enp3s0 mac30 type macvlan mode bridge # 在 enp3s0 接口下添加一个 macvlan 虚拟接口
ip addr add 10.1.1.2/24 brd + dev mac30 # 为 mac30 分配 ip 地址
ip link set mac30 up
ip route del default #删除默认路由
ip route add default via 10.1.1.254 dev mac30 # 设置静态路由
echo "nameserver 10.1.1.1" > /etc/resolv.conf # 设置静态 dns 服务器

# 或者使用 nmcli
nmcli connection add type macvlan dev enp3s0 mode bridge ifname mac30 autoconnect yes save yes
```

或者，若是在 debian 中可以编辑 `/etc/network/interface` 并加入：
```
auto mac30
iface mac30 inet manual
  pre-up ip link add link enp3s0 mac30 type macvlan mode bridge
  pre-up ip addr add 10.1.1.250/24 brd + dev mac30
  up ip link set mac30 up
  post-up ip route del default
  post-up ip route del 10.1.1.0/24 dev enp3s0
  post-up ip route add default via 10.1.1.254 dev mac30
  post-down ip link del dev mac30
```
# 6. 配置客户端 IP&enjoy
后续就是 openwrt 的玩法，除了没有 wifi，其他基本一致。有一点，若需要加载内核模块，则需要在 host 中事先加载
