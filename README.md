# openwrt-in-docker
# docker中运行openwrt

#### 思路
利用macvlan方式创建虚拟接口进行配置。
有感于来自恩山betterman、及rightwifi2017两位大佬斐讯N1的玩法，也获得两位大佬的帮助，在此感谢两位大佬。
由于N1为单网卡，所以配置只能为单臂路由，本案为双网卡opewnrt。
机器拥有双网卡:enp1s0及enp3s0，本案将enp1s0用作openwrt的LAN口，enp3s0用作openwrt的LAN口，
#### 0.安装docker
```
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```
#### 1.配置系统环境
```
#打开网卡混杂模式
ip link set enp1s0 promisc on
ip link set enp3s0 promisc on

#加载PPPOE内核模块
modprobe pppoe

#为enp1s0网卡(LAN口)配置macvlan接口，用于和openwrt进行互联。
#利用同一网卡下bridge模式的macvlan为联通的原理，因此，hMACvLAN将与后面的dMACvLAN联通。
#目的是配置完openwrt后，hMACvLAN接口会从openwrt获取一个ip，使得内网机器能够从此IP访问HOST
nmcli connection add type macvlan dev enp1s0 mode bridge ifname hMACvLAN autoconnect yes save yes
````
#### 2.docker网络配置
```
#为docker创建macvlan虚拟接口，并链接到host网卡
docker network create -d macvlan --subnet=10.1.1.0/24 --gateway=10.1.1.1 -o parent=enp1s0 dMACvLAN
docker network create -d macvlan --subnet=10.1.2.0/24 --gateway=10.1.2.1 -o parent=enp3s0 dMACvWAN
```
#### 3.创建容器
```
#导入镜像
docker import https://downloads.openwrt.org/releases/18.06.1/targets/x86/64/openwrt-18.06.1-x86-64-generic-rootfs.tar.gz openwrt:18.06.1

#创建并启动容器
docker run --restart always -d --network dMACvLAN --privileged --name openwrt openwrt:18.06.1 /sbin/init

#将第二网卡的macvlan挂接到openwrt
docker network connect dMACvWAN openwrt
```
#### 4.配置openwrt
```
#进入容器
docker exec -it openwrt /bin/sh

#编辑/etc/config/network
config interface 'lan'
        option type 'bridge'
        option ifname 'eth0'   #需要与docker netwrok中的虚拟接口匹配（dMACvLAN）
        option proto 'static'
        option ipaddr '10.1.1.1'
        option netmask '255.255.255.0'
        option ip6assign '60'

config interface 'wan'
        option ifname 'eth1'  ##需要与docker netwrok中的虚拟接口匹配（dMACvWAN）
        option proto 'static'
        option ipaddr '10.1.2.1'
        option netmask '255.255.255.0'
        option ip6assign '60'

#重启openwrt网络
/etc/init.d/network restart
```
#### 5.配置客户端IP&enjoy
后续就是openwrt的玩法，除了没有wifi，其他基本一致。有一点，若需要加载内核模块，则需要在host中事先加载