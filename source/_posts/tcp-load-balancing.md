---
layout: post
title: tcp连接负载均衡
date: 2016-06-29 10:57:41
tags: linux
categories: 编程
---

**1.   ****LVS** **性能最强**

<span style="line-height: 1.5;">1\. LVS ：Linux Virtual Server（VS），请求对于RealServer（RS）都是透明的（不修改Src），使用内核转发，相比应用实现性能要高很多。</span>

分为3种模式：

l  DR：性能最好

在VS 和RS 使用相同的VIP，RS上抑制ARP。请求通过VS 分发，应答绕过直接回复。

**优点**：性能最高、流量小

**缺点**：机器都需要在同一个Vlan下，Real Server 之间无法通过VIP通信。需要抑制VIP ARP 有风险。

l  IP Tunnel 性能适中

使用IPIP封包，解决跨Vlan问题，相比DR 就多了一层IP封装，也是只处理请求。

**优点**：跨Vlan

**缺点**：虽然应用最为广泛，但windows 不支持IPIP。

l  NAT

与前两种相比，不需要绑定VIP，但也要修改路由，RealServer 所有流量都需要经过，需要处理很多无用流量，一般实用性很差。

**优点：**不需要要绑定VIP、跨VLAN

支持RS 通过VIP访问RS本身，需要对RS做抑制路由操作，所有流量直接发给网关。

**缺点**：对于请求为SNAT，应答为DNAT。为实现DNAT，需要将RS 的默认网关指向VS

VS 被当成网关处理，RealServer所有的流量都将发给VS，VS的带宽压力会很大。

**配置工具**：如果使用ReadHat版本，可以使用[**Piranha**](http://www.linuxvirtualserver.org/docs/ha/piranha.html) 提供的WEB UI工具配置，配置为需要手动同步一下主板配置。

　　　　如果需要部署多个实例，而且经常改动，Piranha 将提供很友好的配置环境。

**2\. HaProxy 性能高，处理高并发、高连接**

**优点：**

l  性能高，20w连接下，处理10w/s请求

 （单进程处理性能1w/s样子，为更好利用多核，多进程模式提升空间很少，需要开启多个实例，这样一来前端必须上LVS负载多个实例端口）

l  7层、4层都行

l  不限于VLan ，只要网络策略通即可

l  可配性高

l  监控完善

** 注意点：**

l Src IP被修改，业务中publicIP应该从 x-forward-for中获取（Haproxy也支持透明代理，就像LVS的NAT模式，存在问题）

l 流量double，千M网卡最多也只能支持500M流量

<span style="line-height: 1.5;"> 连接数超过65535时，需要考虑绑定多个IP</span>

Tcp 转发性能要比nginx要好。

HaProxy 是默认单线程、也是最高效方式；也可以支持多线程（进程），但监控会随机请求到的业务、部分功能无法正常使用。

<span style="line-height: 1.5;"> </span>

**3\. Nginx 专注于7层高并发，但不善于处理1w+连接数**

要支持4层需要另外的patch，而且高连接数时性能无法和haproxy相比。

默认1个master+ 多个（核心）Worker，可充分利用多核资源。

Master和worker之间通过进程通信实现，有点像IIS5，相比II6 性能要差些。

<span style="line-height: 1.5;"> </span>

**不管HaProxy还是Nginx 都存在一个问题，就是只有一个线程在做Receive，也就是无法均衡软中断。

 对于主要的四层负载，用户态执行时间很短，性能瓶颈在于主要软中断的处理、多个work其实也无济于事；

 当然也可以使用redis那样，开启多实例方式。

**5\. Windows Server 内置NBL 负载均衡功能**

原理和LVS的DR相似，但是相比更加暴力，请求使用广播方式，同一个交换机下RS都将受到请求，RS通过一致性算法确认自己改处理请求还是丢弃请求。

