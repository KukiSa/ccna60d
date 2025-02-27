# 网络故障排除


本文记录网络故障排除，以备今后遇到同样问题时，快速解决。


## 交换机Telnet、SSH 无法登录奇怪故障一例

交换机上配置了 `Vlanif 9` - `10.12.9.250/24`、`Vlanif 10` - `10.12.10.250/24`, 想要通过 `10.12.10.0/23` 网段上的终端，通过 SSH/TELNET 登录交换机，交换机上 SSH/TELNET 配置已配置妥当，但无论是通过 SSH, 还是 TELNET 都无法连接上 `10.12.9.250/24`，不过可通过 `10.12.10.250/24` 可以连接上。为何会这样呢？

这是由于发起 SSH/TELNET 的数据包，是交换机上 `Vlanif 9` 收到的，但返回的数据包，则是从 `Vlanif 10` 发出的，那么就造成了混乱，导致 SSH/TELNET 连接无法建立。

## 光模块起不来的问题


光模块起不来，通常是因为所使用的光模块，与光纤不匹配的原因造成。其中，多模光模块使用 810nm 波长的激光，而单模光模块则使用 1310nm 波长的激光。有关单模与多模的区别，请参考：[Fiber Optic Cable Types: Single Mode vs Multimode Fiber Cable](https://community.fs.com/article/single-mode-cabling-cost-vs-multimode-cabling-cost.html)。排查此问题的三个命令（H3C）:


- `display transceiver interface Ten-GigabitEthernet 1/0/52`


输出：


```console
[xfoss-com-core]display transceiver interface Ten-GigabitEthernet 1/0/52
Ten-GigabitEthernet1/0/52 transceiver information:
  Transceiver Type              : 10G_BASE_SR_SFP
  Connector Type                : LC
  Wavelength(nm)                : 850
  Transfer Distance(m)          : 80(OM2),30(OM1),300(OM3)
  Digital Diagnostic Monitoring : YES
  Vendor Name                   : XFOSS-COM
  Vendor Part Number            : SFPXG-850C-D30
  Part Number                   : SFPXG-850C-D30
  Serial Number                 : 012345678901
```

- `display transceiver manuinfo interface Ten-GigabitEthernet 1/0/52`


输出：


```console
[xfoss-com-core]display transceiver manuinfo interface Ten-GigabitEthernet 1/0/51
Ten-GigabitEthernet1/0/51 transceiver manufacture information:
  Manu. Serial Number : 01234567890123456789
  Manufacturing Date  : 2017-04-01
  Vendor Name         : XFOSS-COM
```


- `display transceiver diagnosis interface Ten-GigabitEthernet 1/0/51`

输出：


```console
[xfoss-com-core]display transceiver diagnosis interface Ten-GigabitEthernet 1/0/51
Ten-GigabitEthernet1/0/51 transceiver diagnostic information:
  Current diagnostic parameters:
    Temp.(C) Voltage(V)  Bias(mA)  RX power(dBm)  TX power(dBm)
    42         3.41        6.42      -2.55          -2.71
  Alarm thresholds:
          Temp.(C) Voltage(V)  Bias(mA)  RX power(dBm)  TX power(dBm)
    High  80         3.60        15.00     0.00           0.00
    Low   -15        3.00        0.00      -10.90         -8.30
```


- `display transceiver alarm interface Ten-GigabitEthernet 1/0/52`


输出：


```console
[xfoss-com-core]display transceiver alarm interface Ten-GigabitEthernet 1/0/52
Ten-GigabitEthernet1/0/52 transceiver current alarm information:
  RX power low
  RX signal loss
```


## H3C IPv6 DHCP snooping 设置

此问题源起 [Windows DNS 解析异常的一种情况](https://snippets.xfoss.com/36_Win_SysAdmin.html#windows-dns-%E8%A7%A3%E6%9E%90%E5%BC%82%E5%B8%B8%E7%9A%84%E4%B8%80%E7%A7%8D%E6%83%85%E5%86%B5)，排查发现在防火墙（FortiGate 60E）上已经禁用 IPv6，故分析主机获取到的局域网IPv6地址，是从交换机上得到。但继续排查发现核心交换机同样没有启用 IPv6 栈，那么就可能是网络上有具备 IPv6 DHCP 功能的设备，给主机了局域网的 IPv6 地址，以及 IPv6 DNS 服务器。

尝试在核心交换机上开启 IPv6 DHCP snooping 功能：


```console
ipv6 dhcp snooping enable
```


并在接入接口上，开启 IPv6 DHCP snooping 功能，已消除上述设备 IPv6 DHCP 功能的影响：


```console
interface GigabitEthernet1/0/1
 port link-mode bridge
 port access vlan 10
 ipv6 dhcp snooping deny
```


随后发现，终端主机已经没有获取到 IPv6 DNS server 了。Windows DNS 解析异常问题解除。


*注*：H3C 交换机建立端口组的命令为：`interface range name acc`，其中 `acc` 为端口组的名字，命令后跟要加入该端口组的接口。



## FG 防火墙问题



本小节记录网络设备 FG 防火墙的有关问题。



### 开启 SSLVPN 的双因素认证


经测试，要在 FGT-60F 启用双因素认证（2FA），需要以下必要条件。


- 用户手机上要安装 FortiToken App；
- 需要采购软件 Token（“用户与认证” -> “FortiToken” -> “+新建” 导入）。



然后在 “用户与认证” -> “设置用户” 下，导入 LDAP/AD 中的用户，并与采购的软件 Token 绑定。随后 FGT-60F 将通过设置的用户邮箱，发出一封带有在 FortiToken 中建立 2FA 验证码条码密钥、二维码图片附件的邮件。用户籍由此邮件便可在 FortiToken App 中获得 2FA 验证码。在今后的 SSLVPN 拨号操作，就都需要 2FA 验证码了。



### 使用 LDAP/AD 认证 SSLVPN 时指定用户组


需要在 “远程组” 中：



- “编辑”  “远程服务器” 中的项目

- 在弹出的 “添加组匹配” 弹窗中

- 浏览到特定组后，在其上点击右键，选择 “添加已选”

- 然后点击 “OK” 确认


### 在防火墙上无法 `ping` 通 IPSec VPN 远端网络问题

在一台 FG-60F 上配置测试 LDAP 与双因素认证的 SSLVPN 时，发现无法连通 IPSec VPN 下的远端 AD 控制器，进而检查防火墙无法 `ping` 通该 AD 控制器，于是联系 FG TAC。FG TAC 排除解决后答复：


> “如远程排查结果，IPSec VPN 阶段 2 中感兴趣流严格控制了只有内网网段能访问，防火墙 LDAP 配置没有指定源，所以流量无法通过 VPN 转发，带上内网接口 IP 源后已经可以正常访问”


TAC 工程师并给出有关指令如下。

```console
# 为 ping 制定源 IP 地址
exec ping-options source <source-intf_ip>
```

```console
# 配置 LDAP 查询源 IP 地址
config user ldap
    edit <LDAP object name>
    set source-ip <IP address associated an interface>
    end
```


检查配置命令：



```console
show user ldap <LDAP object name>
```




### 利用 “策略与对象” - “QoS 限速”，实现 IPSec VPN 流量优先


此举的目的，是运用设备的流量整形特性，在发生拥塞（接入带宽耗尽）时，使 IPSec VPN 隧道流量仍能得到保证。


TAC 给的示例配置如下：


```config
config firewall shaper traffic-shaper
    edit "guarantee-ipsec-traffic"
        set guaranteed-bandwidth 50000  //保证带宽,单位kbps，确保如果发生拥塞的时候能够流量优先保证
        set maximum-bandwidth 100000 //最大带宽,单位kbps
        set per-policy enable
    next
    edit "guarantee-wan-traffic"
        set maximum-bandwidth 100000 //最大带宽,单位kbps
        set per-policy enable
    next
end
config firewall shaping-policy
    edit 0 //顺序新建一条限速策略
        set name "guarantee-ipsec-traffic"
        set service "ALL"
        set srcintf X    //流量的进入接口
        set dstintf Y    //流量的出接口（ipsec接口）
        set traffic-shaper "guarantee-ipsec-traffic"
        set traffic-shaper-reverse "guarantee-ipsec-traffic"
        set srcaddr "all"    //可以具体指定流量的IP范围
        set dstaddr "all"    //可以具体指定流量的IP范围
    next
end
config firewall shaping-policy
    edit 0 //顺序新建一条限速策略
        set name "guarantee-wan-traffic"
        set service "ALL"
        set srcintf X    //流量的进入接口
        set dstintf Z    //流量的出接口（wan接口）
        set traffic-shaper "guarantee-wan-traffic"
        set traffic-shaper-reverse "guarantee-wan-traffic"
        set srcaddr "all"    //可以具体指定流量的IP范围
        set dstaddr "all"    //可以具体指定流量的IP范围
    next
end
```

参考示例配置进行配置后，使用配置查看命令，查看到的配置如下：


```config
FG-60F-BJ-KG # show firewall shaper traffic-shaper
config firewall shaper traffic-shaper

(...)

    edit "guarantee-ipsec-traffic"
        set guaranteed-bandwidth 5000
        set maximum-bandwidth 10000
        set per-policy enable
    next
    edit "guarantee-wan-traffic"
        set maximum-bandwidth 50000
        set per-policy enable
    next
    edit "guarantee-dmz-traffic"
        set maximum-bandwidth 30000
        set per-policy enable
    next
end


FG-60F-BJ-KG # show firewall shaping-policy
config firewall shaping-policy
    edit 1
        set uuid ba271712-8b82-51ef-606c-50e63838e1a5
        set name "guarantee-ipsec-traffic"
        set service "ALL"
        set srcintf "internal"
        set dstintf "to_ks"
        set traffic-shaper "guarantee-ipsec-traffic"
        set traffic-shaper-reverse "guarantee-ipsec-traffic"
        set srcaddr "LAN"
        set dstaddr "to_ks_remote_subnet_1"
    next
    edit 2
        set uuid bddc113a-8b84-51ef-e3d2-5e0685bba2ae
        set name "guarantee-dmz-traffic"
        set service "ALL"
        set srcintf "dmz"
        set dstintf "wan1"
        set traffic-shaper "guarantee-dmz-traffic"
        set traffic-shaper-reverse "guarantee-dmz-traffic"
        set srcaddr "all"
        set dstaddr "all"
    next
    edit 3
        set uuid 0c2ac2c8-8b85-51ef-d450-c9c2b69983c5
        set name "guarantee-wan-traffic"
        set service "ALL"
        set srcintf "internal"
        set dstintf "wan1"
        set traffic-shaper "guarantee-wan-traffic"
        set traffic-shaper-reverse "guarantee-wan-traffic"
        set srcaddr "all"
        set dstaddr "all"
    next
end
```


预期这样配置后，可以解决北京 site 在接入带宽耗尽时，IPSec VPN 受到影响的问题。


### SSLVPN 中策略放行但某个网段下主机不通问题


这是因为 FG 中 SSLVPN 下，还有路由的设置，通过这个设置，控制了 SSLVPN 拨号进来的用户，总体上可以访问哪些网段。设置位于

`VPN --> SSL-VPN 门户 --> full-access --> 隧道分割 --> 隧道分离地址`


若在此设置中未加入某个网段，那么即使在 “策略” 中放行了这个网段中的某些主机，SSLVPN 拨入的用户也仍然不能连接到这些主机。同时 SSLVPN 拨号的终端上，也不会有到这个网段的路由条目（加入到隧道分离地址中的网段，在拨号终端的路由表中是有对应的路由条目的）。



### IPSec 隧道下站点 B 无法访问站点 A 特定子网问题


站点 B 可以访问到站点 A 的 `10.11.1.0/24` 子网，但无法访问到 `10.11.2.0/24` 子网，检查站点 B 上已设置正确的路由，站点 A 上也放行了 IPSec 隧道对 `10.11.2.0/24` 子网的访问。

随后联系 FG 官方支持工程师对此问题进行排查，发现是策略配置错误导致。`10.11.1.0/24` 和 `10.11.2.0/24` 两个子网，是配置在 FG 60E 防火墙的两个不同 internal 接口上，而原先的策略配置，把 IPSec 隧道对 `10.11.2.0/24` 子网的访问放行，与对 `10.11.1.0/24` 子网的放行，一并放在了 `10.11.1.0/24` 所在接口上，由此造成站点 B 对 `10.11.2.0/24` 子网的访问不通。


**解决方法**：在 Site A 的 FG 防火墙上，把 IPSec 隧道对 `10.11.2.0/24` 网段的访问放行策略，建立在 `10.11.2.0/24` 子网所对应的接口上。



### IPSec 隧道建立问题



- 建立 “接口模式” IPSec 隧道时，务必要选择物理接口，否则会报错；

- ~~特性（“Feature”）版本下，无法在网页 GUI 界面下修改IPSEC VPN 配置；成熟版本下可以~~，选择“转换未自定义隧道”后就可以了。


### “自定义” 隧道下参数配置


在不同企业、对端为非 FG 防火墙的如下 IPSec 隧道参数约定下，FG 端的配置如下面的截图所示。


```text
Phase 1 Proposal:
    IKE Version:            IKEv1
    Authentication Method:  Preshared
    Preshared Key:          somekey(根据客户不同进行事先协商确定)
    DH Group:               Group 2
    Encryption Algorithm:   3DES-CBC
    Hash Algorithm:         SHA-1
    IKE SA Lifetime:        480 Mins
    Mode:                   Main

Phase 2 Proposal:
    Perfect Forward Secrecy:    NO-PFS
    Encapsulation:
        Encryption (ESP)
            Encryption Algorithm:       3DES-CBC
            Authentication Algorithm:   SHA-1
    IPSEC Tunnel Lifetime:      60 Mins
```


*图 1、阶段 1 认证方法*

![阶段 1 认证方法](images/fg_ipsec_01.png)


*图 2、阶段 1 参数*

![阶段 1 参数](images/fg_ipsec_02.png)


*图 3、阶段 2 参数*

![阶段 2 参数](images/fg_ipsec_03.png)


### FG 固件升级


FG 防火墙固件升级，需要按照升级路径进行。访问 [Recommended Upgrade Path](https://docs.fortinet.com/upgrade-tool/fortigate)，可以找到升级路径。


在未按照升级路径升级固件时，就会出现系统无法启动，如下所示。


```console
FortiGate-60F (19:09-07.21.2021)
Ver:05000009
Serial number: FGT60FTK2209J3YG
CPU: 1200MHz
Total RAM: 2 GB
Initializing boot device...
Initializing MAC... NP6XLITE#0
Please wait for OS to boot, or press any key to display configuration menu......

Booting OS...
Initializing firewall...
failed verification on /data/datafs.tar.gz
fos_ima: System Integrity check failed...
CPU7: stopping
CPU4: stopping
CPU0: stopping
CPU2: stopping
CPU5: stopping
CPU6: stopping
CPU1: stopping

```


此时，设备可以启动到 Bootloader，可进入 `configuration menu`，选择升级前可启动的固件启动。如下所示。


```console
FortiGate-60F (19:09-07.21.2021)
Ver:05000009
Serial number: FGT60FTK2209J3YG
CPU: 1200MHz
Total RAM: 2 GB
Initializing boot device...
Initializing MAC... NP6XLITE#0
Please wait for OS to boot, or press any key to display configuration menu..

[C]: Configure TFTP parameters.
[R]: Review TFTP parameters.
[T]: Initiate TFTP firmware transfer.
[F]: Format boot device.
[I]: System information.
[B]: Boot with backup firmware and set as default.
[Q]: Quit menu and continue to boot.
[H]: Display this list of options.

Enter C,R,T,F,I,B,Q,or H:

Loading backup firmware from boot device...


Booting OS...
.Initializing firewall...

System is starting...
Starting system maintenance...
Scanning /dev/mmcblk0p1... (100%)
Scanning /dev/mmcblk0p3... (100%)
```

### `7.2.x` 后固件升级


> “7.2系列的固件在“系统”--“Fabric管理”这边上传固件。”


进入 `Firmware & Registration` 后，需要在当前运行固件版本上点击，后 “升级” 按钮由灰色变为正常可用。



## 深信服 Sangfor SSL VPN



### 解锁用户


“运行状态” -> “在线用户” -> “锁定用户：n[查看]”


### VPN 上的 DNS 服务器设置


`系统设置` -> `SSL VPN选项` -> `系统选项` -> `内网域名解析`

> **参考链接**：[VPN设备上的DNS解析顺序](https://bbs.sangfor.com.cn/forum.php?mod=viewthread&tid=9161)


（End）


