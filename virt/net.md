# net
## 网络模型
总的来说，目前有四种常见的网络模型：
- 桥接（Bridge Adapter）

	使用虚拟交换机（Linux Bridge）, 将虚拟机和物理机连接起来，它们处于同一个网段, IP 地址是一样的.

	在这种网络模型下，虚拟机和物理机都处在一个二层网络里面，所以有：
	- 虚拟机之间彼此互通
    - 虚拟机与主机彼此可以互通
    - 只要物理机可以上网，那么虚拟机也可以

    桥接网络的好处是简单方便，但也有一个很明显的问题，就是一旦虚拟机太多，广播就会很严重. 所以, 桥接网络一般也只适用于桌面虚拟机或者小规模网络这种简单的形式.
- NAT

	网络地址转换（Network Address Translatation）, 这种模型严格来讲，又可以分为 NAT 和 NAT 网络两种: 
	- NAT：主机上的虚拟机之间是互相隔离的，彼此不能通信（它们有独立的网络栈，独立的虚拟 NAT 设备）

	                                    |->虚拟nat设备-> vm(10.0.2.15)
    	物理网卡(192.168.108.34)->switch -> 虚拟dhcp
    									|->虚拟nat设备-> vm(10.0.3.15)

    - NAT 网络：虚拟机之间共享虚拟 NAT 设备，彼此互通

                                           |-> vm(10.0.2.15)
    	物理网卡(192.168.108.34)->虚拟nat设备-> 虚拟dhcp
    									   |-> vm(10.0.2.4)

	根据 NAT 的原理，虚拟机所在的网络和物理机所在的网络不在同一个网段，虚拟机要访问物理所在网络必须经过一个地址转换的过程，也就是说在虚拟机网络内部需要内置一个虚拟的 NAT 设备来做这件事.

	NAT 网络模式中一般还会内置一个虚拟的 DHCP 服务器来进行 IP 地址的管理.

	总结一下，以上两种 NAT 模式，如果不做其他配置，那么有：
    - 虚拟机可以访问主机，反之不行
    - 如果主机可以上外网，那么虚拟机也可以
    - 对于 NAT，同主机上的虚拟机之间不能互通
    - 对于 NAT 网络，虚拟机之间可以互通

	PS：如果做了 端口映射 配置，那么主机也可以访问虚拟机

- 主机（Host-only Adapter）

	只限于主机内部访问的网络，虚拟机之间彼此互通，虚拟机与主机之间彼此互通。但是默认情况下虚拟机不能访问外网（注意：这里说的是默认情况下，如果稍作配置，也是可以的）.

	它的网络模型是相对比较复杂的，可以说前面几种模式实现的功能，在这种模式下，都可以通过虚拟机和网卡的配置来实现，这得益于它特殊的网络模型.

	```bash
											 |-> vm(192.168.56.103)
	host网卡---(两者默认断开)虚拟网卡(192.168.56.1) -> switch-
											 |-> vm(192.168.56.104)
	```

	主机网络模型会在主机中模拟出一块虚拟网卡供虚拟机使用，所有虚拟机都连接到这块网卡上，这块网卡默认会使用网段 192.168.56.x（在主机的网络配置界面可以看到这块网卡).

	默认情况下，虚拟机之间可以互通，虚拟机只能和主机上的虚拟网卡互通，不能和不同网段的网卡互通，更不能访问外网. 如果vm想访问外网，那么需要将按上图将host网卡和虚拟网卡桥接或共享即可.
- 内部网络（Internal）

	这种模型是相对最简单的一种，虚拟机与外部环境完全断开，只允许虚拟机之间互相访问，这种模型一般不怎么用，所以在 VMware 虚拟机中是没有这种网络模式的.

VirtualBox 支持的四种模型，对于 VMware，则只有前三种.

可访问性:
<table>
<thead>
<tr>
<th style="text-align: center">Model</th>
<th style="text-align: center">VM -&gt; host</th>
<th style="text-align: center">host -&gt; VM</th>
<th style="text-align: center">VM &lt;-&gt; VM</th>
<th style="text-align: center">VM -&gt; Internet</th>
<th style="text-align: center">Internet -&gt; VM</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: center">Bridged</td>
<td style="text-align: center">+</td>
<td style="text-align: center">+</td>
<td style="text-align: center">+</td>
<td style="text-align: center">+</td>
<td style="text-align: center">+</td>
</tr>
<tr>
<td style="text-align: center">NAT</td>
<td style="text-align: center">+</td>
<td style="text-align: center">Port Forwarding</td>
<td style="text-align: center">-</td>
<td style="text-align: center">+</td>
<td style="text-align: center">Port Forwarding</td>
</tr>
<tr>
<td style="text-align: center">NAT Network</td>
<td style="text-align: center">+</td>
<td style="text-align: center">Port Forwarding</td>
<td style="text-align: center">+</td>
<td style="text-align: center">+</td>
<td style="text-align: center">Port Forwarding</td>
</tr>
<tr>
<td style="text-align: center">Host-only</td>
<td style="text-align: center">+</td>
<td style="text-align: center">+</td>
<td style="text-align: center">+</td>
<td style="text-align: center">-</td>
<td style="text-align: center">-</td>
</tr>
<tr>
<td style="text-align: center">Internal</td>
<td style="text-align: center">-</td>
<td style="text-align: center">-</td>
<td style="text-align: center">+</td>
<td style="text-align: center">-</td>
<td style="text-align: center">-</td>
</tr>
</tbody>
</table>