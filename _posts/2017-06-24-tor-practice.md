---
layout: post
title:  "Tor 实践"
date:   2017-06-24
categories: security tor
---

[Tor][] 在这里\*无论是直连还是接网桥都是已经做不到的了，尽管 Tor 的开发者一再声称 Tor 所使用的网桥协议 [obfs4][] 可以躲避任何侦测手段，然而因为实现上的小缺陷（而不是协议本身，协议是完美的[^1]）现在墙也可以侦测到 obfs4 流量并进行拦截了，所以现在一个通行的办法是用 [Shadowsocks][] 先穿出去，然后让 Tor 走 Shadowsocks 来连接洋葱网络。

~~\* 嗯。~~

[Tor]:         https://en.wikipedia.org/wiki/Tor_(anonymity_network)
[obfs4]:       https://github.com/Yawning/obfs4/blob/master/doc/obfs4-spec.txt
[Shadowsocks]: https://en.wikipedia.org/wiki/Shadowsocks

## 本机通过 SS 连接 Tor

这一点比较简单。首先在本机建立 SS 代理，这里不再赘述。如果使用的是 Tor Browser，只需要：

1. 在网络初始化界面选择 **Configure（配置）**；
2. 在 **Does your Internet Service Provider (ISP) block or otherwise censor connections to the Tor Network?（互联网服务提供商（ISP）是否对 Tor 网络连接进行了封锁或审查？）** 处，选择“否”。（因为此处是 Tor Bridges Configuration（Tor 网桥配置），选择“是”之后将可以使用网桥。）
  - 实际上，使用网桥是更安全的举措[^2]。但是如果你使用的是 Tails（见下），Tails 并没有像 Tor Browser 里有“集成的网桥”，而要完全手动输入。如果你记得住一些，那是最好，这一步你就可以选择“是”。
3. 在 **Does this computer need to use a local proxy to access the Internet?（是否需要本地代理访问互联网？）** 处，选择“是”。
4. 选择 **Proxy Type（代理类型）** 为你在 SS 里设立的本地代理类型（可以是 SOCKS5 或 HTTP(S)，默认的也是最好的是 SOCKS5，因为 HTTP(S) 会带来一些性能开销[^3]），**Address（地址）** 填写 **`127.0.0.1`** 或者 **`localhost`**，端口填写代理端口，点 **Connect（连接）** 即可通过 SS 连接到洋葱网络。

如果使用的是 Tor 客户端来建立代理服务器[^4]，你需要编辑 `/etc/tor/torrc`，添加：

<!-- 为了那行注释还是写 bash 吧 -->

```bash
# 替换 <PORT> 为代理端口
Socks5Proxy 127.0.0.1:<PORT>
```

就可以让 Tor 客户端走本地代理出站了。

## 使用 [Tails][]

Tails 在系统全局网络连接上开启了 Tor，而且系统内也没有 SS，自己带上一个也很麻烦，就需要用另一台电脑或者手机做本地代理服务器。

[Tails]: https://tails.boum.org/

### 用 PC 给 Tails 做代理

首先在第二台电脑上设立 SS 代理服务器。以 [ss-qt5][] 为例，编辑服务器选项，将“本地地址”改为电脑目前所在的 IP 地址（别的不行），注意端口不要冲突，然后连接即可在设置好的端口上建立 SS 代理。

启动 Tails，在欢迎界面里的 **Additional Settings** 添加 **Network Connection**，选择 **Bridge & Proxy**，再启动到 Tails 连接网络，按照[“本机通过 SS 连接 Tor”](#本机通过-SS-连接-Tor)章节的步骤进行配置即可。**注意在地址里要写代理服务器的 IP 地址和代理端口！**

[ss-qt5]: https://github.com/shadowsocks/shadowsocks-qt5

### 用手机给 Tails 做代理

不清楚怎么让 SS 代理热点流量……（好像不 Root 是不可以的）

## 在 Android 上使用 Orbot

Orbot 是 Tor 在安卓平台上的实现（就是有点慢）。要让 Tor 走 SS：

1. 首先要注意 SS 上的“本地端口”，SS 在这个端口上设立了本地代理，我们让 Tor 走这里。
2. 在 Orfox 里，点击右上角菜单进入 Settings（设置），在 **Outbound Network Proxy（出站网络代理）** 处，填写（出站代理类型） 为 **`Socks5`**，填写（出站代理主机）为 **`127.0.0.1`** 或 **`localhost`**，（出站代理端口）为 SS 上设置的“本地端口”。
  - 此处注意：**不需要在 SS 里为 Orbot 和 Orfox 设置分应用代理**，这样反而可能会造成连接问题。
3. 长按洋葱或者点击 Start 就可以通过 SS 连接洋葱网络了。
  - Tips：屏幕右边缘向左拉是日志窗口，可以看到连接实时进度，只要看到 _"NOTICE: Bootstrapped 100%: Done"_ 就说明已经完全建立 Tor 链路了。

要使用 Orbot，需要配合 Orfox 或者别的设计为配合 Orbot 的应用使用，不过 Orbot 也设立了一个本地代理端口（默认 9050），其它应用也可以使用本地代理（如果可以）访问 Tor 网络。[^4]

## 注

[^1]: 盖子这么说的。
[^2]: <https://www.torproject.org/download/download-easy.html.en#warning> "...by default, it does not prevent somebody watching your Internet traffic from learning that you're using Tor. If this matters to you, you can reduce this risk by configuring Tor to use a Tor bridge relay rather than connecting directly to the public Tor network."
[^3]: <https://github.com/shadowsocks/shadowsocks-qt5/wiki/%E4%BD%BF%E7%94%A8%E6%89%8B%E5%86%8C#%E5%85%B6%E4%BB%96%E8%AF%B4%E6%98%8E>
[^4]: Tor 不建议这样做。参见（需要梯子）：<https://www.torproject.org/download/download-easy.html.en#warning>
