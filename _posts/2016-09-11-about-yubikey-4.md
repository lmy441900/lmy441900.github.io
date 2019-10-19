---
layout: post
title:  "关于 Yubikey 4 的一些玄学"
date:   2016-09-10
categories: cn
---

大概在两个月前自行入了一枚 [Yubikey 4][yk]，本来想着就是当 [OpenPGP Card][pgpc] 和 [U2F][u2f] 用的，结果发现 [Yubikey 4][yk] 的各种功能和用法简直是魔幻般的玄学。网上的一些长者的人生经验，包括官方钦定的文档，好像都没怎么系统地讲清楚 [Yubikey 4][yk] 的功能，这里我想把我~~两个月来~~折腾的心得分享一下。

<!-- 实际上已经更新很多次了 -->

## 功能

Yubikey 4 好就好在它集多种功能为一体，它们的关系如下：

```
Yubikey
  +- HID
  |   +- U2F
  |   +- OTP (2 Slots)
  |       +- Yubico OTP
  |       +- OATH TOTP / HOTP
  |       +- Static Password
  |
  +- Challenge-Response (Share Slots w/ OTP)
  |    +- Yubico OTP
  |    +- HMAC-SHA1
  |
  +- Smart Card
      +- PIV (18 Slots, 4 main slots)
      +- OpenPGP (3 Slots)
```

### U2F

Universal 2FA 是 [FIDO Alliance][fido] 制定的新一代基于 Challenge-Response 的二步验证机制，[Yubico][yubico] 作为 FIDO 的成员之一当然是不遗余力地推出了多款 U2F USB Token，[Yubikey 4][yk] 就是一款集成了 [U2F][u2f] 的产品。[Yubikey 4][yk] 的 U2F 功能~~是开箱即用的~~需要安装 `libu2f-host`（这种的）包，安装对应库及应用 udev 规则来使用。插入 Yubikey，打开网站的设备注册，当绿灯闪烁的时候轻触按钮，便可提高账户安全性。

啊，当然也可以用于系统登录。在 Windows 下可以安装 [Yubikey for Windows Hello][yk-windows-hello]，配置好以后将 Yubikey 插入电脑即可解锁。

- 说句实话，我本人不觉得这是个很好的设定……连触摸都不需要（虽然的确没这个必要但这一点都不 2F 诶）

啊，Linux 当然也可以。安装 `pam-u2f`，就可以利用 U2F 进行 PAM 认证。

1. 执行 `pam-u2f` 附带的 `pamu2fcfg`，触摸 Yubikey 按钮，可以获得一行~~神秘代码~~ U2F 配置，复制这行配置；
2. 将上面得到的配置放到 `$XDG_CONFIG_HOME/Yubico/u2f_keys`（`$XDG_CONFIG_HOME` 一般是 `~/.config`）里；
3. 编辑 `/etc/pam.d/`（按需求），在合适位置添加一行 `auth required pam_u2f.so`
  - 除了 `required` 也可以填其它……详见 `man pam.d`。
  - 接着这一行可以给 PAM 模块一些参数，比如 `cue` 会打印一行字提醒你触摸设备。详见 `man pam_u2f`。

### OTP

[Yubikey 4][yk] 的 OTP 功能并不仅仅指其 [Yubico OTP][yubico-otp] 功能，它包括：

- [Yubico OTP][yubico-otp]
- [OATH HOTP][oath-hotp]
- [静态密码][static-pass]
- [Challenge-Response][chalresp]

在 OTP 功能中，共有 **2 个 Slot**，可以设定以上任意两个功能。**注意这里的 Slot 与 Challenge-Response 共用。**

#### Yubico OTP

> ~~最近出现一种人，贴出一串不明字符串来炫耀自己拥有某种硬件。~~

[Yubico OTP][yubico-otp] 以 [YubiCloud][ycloud] 作为中心验证由 Token 传送到服务器的一次性密码。这一功能是默认在 [Yubikey 4][yk] 内的 Slot 1 开启的，所以购买 Yubikey 后插上电脑，第一件事就是轻触按钮发送一串不明字符串来炫耀啦。

#### OATH HOTP / TOTP

HOTP 是基于 HMAC（散列消息认证码)的一次性密码（OTP）算法，与 TOTP（基于时间的一次性密码）相比，HOTP 使用一个计数器来达到一次性密码的目的。HOTP 的优点是不需要时钟（因为 Yubikey 里头也没有时钟），但是如果不小心按到了，这个计数器就和服务器不同步了，就需要重新同步。

不过，**Yubikey 也是可以使用 TOTP 的。**
只是因为 Yubikey 并没有硬件时钟，所以需要借助电脑本身的帮助。Yubico [的这款 TOTP 应用][otp-app]，可以用来计算 TOTP 码。

#### 静态密码

顾名思义，输出一串固定的字符串作为密码。看起来一点卵用都没有（用个记事本一装不就出来了嘛），但是这货在任何有密码输入框的地方都能使用，因此可以借助 Yubikey 输出一串非常难记而且预先随机生成的字符串来增强密码复杂度。

但是如果你觉得可以靠这个功能免手输密码，**呵呵**，只要 Yubikey 遗失，后果不堪设想。所以静态密码的优雅且正确用法是：

- 输入一段自己的常规密码
- 在任何地方再插入 Yubikey 的随机密码（可以是在自己的密码前面、中间、后面；如果不想 Yubikey 自动按下回车键可以在 Yubikey Personalization Tool 里面的 Settings 取消 Yubikey 在最后输出 `\n` 的功能）

### Challenge-Response

Challenge-Response 在 Yubikey 4 中有两种模式：Yubico OTP 和 HMAC-SHA1。Yubico OTP 可以在基于 Challenge-Response 的情况下以 Yubico OTP 的方式来做验证，需要网络连接到 YubiCloud；而 HMAC-SHA1 则是用于离线验证。

一般的应用可以是 Linux PAM；可以安装 [Yubico 提供的 Linux PAM 模块][yubico-pam]来启用验证，用法大概和上面 [U2F](#U2F) 的一样，这里不再赘述。

嗯对了，最近在用 [KeePassXC][keepassxc]，这款 [KeePassX][keepassx] 的“社区升级版本”（KeePass Cross-Platform Community Edition）能利用到 Yubikey 的此 Challenge-Response 功能，可以配置打开密码数据库的时候同时使用 Challenge-Response 来加强数据库安全性。

- 稍微提一下，在 Android 上有一款 KeePassX(C) 兼容的 App，[Keepass2Android](keepass2android)，它同样支持 Challenge-Response 功能，不过它要求一个种子文件（我不很清楚这是什么操作）。KeePassXC 不支持这种操作，但是 KeePassXC 希望这些应用采用他们的解决方案。

### PIV

[Yubikey 4][yk] 可以用作 PIV (FIPS 201-1) 规范的智能卡，可以用于一般智能卡用途，例如 SSH、Windows Active Directory 计算机登录，等等。但是，**Yubikey 的 PIV 并不是一个完全的 PIV 实现**，它与 OpenSC 兼容却不能完全使用某些功能，比如说不可以往卡里放文件。

使用 [Yubikey PIV Manager][ykpiv] 来管理 Yubikey 中的证书。[Yubikey 4][yk] 在 PIV 功能中共有 **18 个 Slot**：9a, 9c, 9d, 9e, 82-95, f9。**这些 Slot 和 OTP 中的 Slot 是不同的（互不影响）**。其中：

- 9a 用于验证持卡人身份；
- 9c 数字签名；
- 9d Key Management“密钥管理”，用于加密；
- 9e ~~不知道~~ Yubico 在[官方文档](https://developers.yubico.com/PIV/Introduction/Certificate_slots.html)中介绍 9e 中的密钥用于支持别的一些诸如 PIV 门锁这样的物理应用（也就是说，不是在电脑上操作 Yubikey）；
  - 网友 Yanru Huang（竟然有人跟我互动诶）反馈存取 9e 中的证书和密钥是唯一一个**不需要输入 PIN** 的 Slot。反正我是楞没搞明白，不设 PIN 和咸鱼有什么区别……。
- 82-95 ~~是过时的证书的 Slot~~ 用于存放 9d Slot 中不再使用（吊销了或者是怎么的）的证书，这部分是用来保证不会出现无法再解密之前用吊销了的证书加密的内容这种尴尬的事情。
- f9 (>=4.3) 不知道

最近玩 [PKI][pki] 用到了 PIV，回头再写……

### OpenPGP Card

OpenPGP 卡配合 GPG 使用，用来存放 GPG 私钥（**拿不出来的**），并用作加密、签名用途。在 [Yubikey 4][yk] 此功能中，有 **3 个 Slot** 存放密钥：

- Signature（签名用途）
- Encryption（加 / 解密用途）
- Authentication（验证用途）

目前 Yubikey 4 只支持 RSA GPG Key。在 `gpg --card-status` 里可能会看到三个 Slot 的 Key attributes 均为 `rsa2048`，实际上它们都可以装上 4096 位的密钥。

优雅的方法是生成不同用途的子钥，然后将子钥扔进卡里。**不要将主钥放进卡里**，因为主钥还承担人际关系的作用，一旦遗失 Yubikey，直接吊销子钥完事，而不必要吊销主钥然后重新建立 Web of Trust。

<!-- 果然这种脚注 URL 不能一整篇文章的放在一起……简直是灾难。 -->

[yk]:           https://yubi.co/4
[pgpc]:         https://en.wikipedia.org/wiki/OpenPGP_card
[u2f]:          https://en.wikipedia.org/wiki/Universal_2nd_Factor
[fido]:         https://fidoalliance.org/
[yubico]:       https://yubico.com
[yubico-otp]:   https://developers.yubico.com/OTP/
[oath-hotp]:    https://developers.yubico.com/OATH/#_hotp
[otp-app]:      https://github.com/Yubico/yubioath-desktop
[yubico-pam]:   https://github.com/Yubico/yubico-pam
[keepassxc]:    https://keepassxc.org/
[keepassx]:     https://www.keepassx.org/
[keepass2android]: https://play.google.com/store/apps/details?id=keepass2android.keepass2android
[static-pass]:  https://yubi.co/4
[chalresp]:     https://yubi.co/4
[ycloud]:       https://www.yubico.com/products/services-software/yubicloud/
[ykpiv]:        https://developers.yubico.com/PIV/
[pki]:          https://en.wikipedia.org/wiki/Public_key_infrastructure
[yk-windows-hello]: https://www.microsoft.com/en-us/store/p/yubikey-for-windows-hello/9nblggh511m5
