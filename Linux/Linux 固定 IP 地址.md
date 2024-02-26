# Linux 系统配置固定 IP 地址

## 前言
正常来说，很多时候使用虚拟机或者本地服务器，默认情况下使用DHCP。
这样会有一个困扰，机器关机，隔天再开机，IP 可能会发生变化。
因此会在创建机器后，配置一个静态 IP 地址，不需要每次连接都改变 IP。

## 环境


操作系统：centos 7.9-VMware

## 配置过程

1. 查找网络配置文件

    ```shell
    # ls /etc/sysconfig/network-scripts/ifcfg-*
    
    /etc/sysconfig/network-scripts/ifcfg-ens33  /etc/sysconfig/network-scripts/ifcfg-lo
    ```

2. 修改 `/etc/sysconfig/network-scripts/ifcfg-ens33` 文件，`ifcfg-lo` 是本地回环地址的配置文件

    ```shell
    TYPE=Ethernet #类型
    PROXY_METHOD=none
    BROWSER_ONLY=no
    #是否启动DHCP：none为禁用DHCP；static为使用静态ip地址；设置DHCP为使用DHCP服务
    #如果要设定多网口绑定bond的时候，必须设成none
    BOOTPROTO=static
    DEFROUTE=yes #就是default route，是否把这个网卡设置为ipv4默认路由
    IPV4_FAILURE_FATAL=no # 如果ipv4配置失败禁用设备
    IPV6INIT=yes #是否使用IPV6地址：yes为使用；no为禁用
    IPV6_AUTOCONF=yes
    #就是default route，是否把这个网卡设置为ipv6默认路由
    IPV6_DEFROUTE=yes
    # 如果ipv6配置失败禁用设备
    IPV6_FAILURE_FATAL=no
    IPV6_ADDR_GEN_MODE=stable-privacy
    #网络连接的名字
    NAME=ens33
    #唯一标识
    UUID=b4701c26-8ea8-46a5-b738-1d4d0ca5b5a9
    
    # 网卡名称
    #设备名，不要自己乱改，和文件ifcfg-** 里的**要一致
    #一般不需要修改
    DEVICE=ens33  
    
    #启动或者重启网络时是否启动该设备：yes是启用；no是禁用
    ONBOOT=yes
    
    #添加如下配置信息
    IPADDR=192.168.29.136      #IP地址
    GATEWAY=192.168.29.2       #网关
    DNS1=192.168.29.2          #NDS
    DNS2=8.8.8.8               #NDS                     
    PREFIX=24                  #centos子网掩码长度：24--> 255.255.255.0    
    
    # 子网掩码 RedHat，不同版本的Linux的配置是不一样的 
    # NETMASK=255.255.255.0 
    
    # 地址 ipv6 配置信息，如果不使用ipv6 可以不用配置
    IPV6_PEERDNS=yes
    IPV6_PEERROUTES=yes 
    IPV6_PRIVACY=no 
    ```

3.  重启网络生效

    ```shell
    service network restart
    ```

    