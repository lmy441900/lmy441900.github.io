---
layout: post
title:  "如何在单台服务器上可靠地部署 ownCloud（未完）"
date:   2016-08-06
categories: ownCloud
---

ownCloud 是一个知名的私有云解决方案，其功能不仅有基础云计算中的云存储，还能进行生活方面的管理（譬如日历、联系人、待办事项等）。关于 ownCloud 的更详细介绍，可以访问其[官网][owncloud]，在此我就不再赘述。

<!--
我研究 ownCloud 是因为我的母校，[东莞市第一中学][dgyz]，开展了“微课”（即常说的 Mook）活动，学生可利用寒、暑假选修自己喜欢的“校本课程”，并在临开学时统计学习效果，计入学分（没错，高中也是有学分的）。学校的现行方案是使用百度网盘，利用其共享功能及大容量来达到基本的 Mook 视频分享目的。但是将这些资源放置在公共空间（我们暂且把百度网盘视作公共服务器）是一种明显不负责任的行为：在前一阵子经常就有“被7秒”的事情发生。因此，自托管一个 Mook 平台就成了当务之急。

然而选定的 [OpenEdX][openedx] 平台，本是一个非常优秀的 Mook 平台，然而其价值在于自开发（并需要使用 Django）而且系统庞大，我一个人明显不能胜任（说实话我连部署都搞不定）。因此我就在寻找百度网盘的替代方案——ownCloud 明显是坠吼的。

题外话说得有点多。（逃（并不
-->

本文将完整介绍如何从零开始搭建一个稍侧重安全的 ownCloud 单台服务器，因此涉及大型网站所使用的技术时（譬如 [Redis][redis]、分布式计算方面技术等）本文将不讨论（其中一个原因是我并没有去了解过）。

测试通过的操作系统：

- [AOSC OS 3.6.2][aosc-os]
- [CentOS 7][cent-os-7]

测试使用的 ownCloud 版本：**9.1.0**

测试使用的软件：LAMP（Linux + Apache + MySQL(MariaDB) + PHP）

**注意：并不能使用 Windows，除非交钱（Enterprise 版本）。**

[owncloud]: https://woncloud.org
[dgyz]:     http://dgyz.net
[openedx]:  https://open.edx.org/
[redis]:    http://redis.io/
[aosc-os]:  https://aosc.io
[cent-os-7]: https://centos.org

# 部署

首先我们需要获得 ownCloud。如果使用的是 [CentOS][cent-os-7]（[RHEL][rhel-7] 的社区版本）一类的大发行版，ownCloud 社区是提供有软件包的，直接根据[这个页面](https://download.owncloud.org/download/repositories/stable/owncloud/)的指引安装就可以了。如果是 [AOSC OS][aosc-os] 这样的没有 ownCloud 包的发行版或者想自己动手，你有两个选择：

1. 下载压缩包解压到目标位置
2. 使用预安装 PHP 脚本

*方案 2* 可以自动化*方案 1* 的过程并自动设置好权限等问题。图省事，或者是在一些没有提供 Shell 接入的 VPS 商（比如虚拟空间），可选这个方案。下面主要阐述方案 1，因为博主一开始并没有注意到方案 2，直接啃了块大骨头 :(

[rhel-7]:    https://www.redhat.com/rhel/

## 解压并设置好权限

**注意：你必须知道自己系统以哪个用户启动 Apache（httpd）。查阅相关资料获得这个信息，或者看看 `httpd.conf` 里面的 `User` 和 `Group` 配置。**

注意谜一样用法的 `tar`,先进入目标目录，再用 `tar` 解压到当前目录。

```bash
cd /srv/www/path/to/extract
tar xf /path/to/owncloud-9.1.0.tar.bz2
# 你也可以用 v 选项装逼（显示进度，会刷屏）
# tar xvf /path/to/owncloud-9.1.0.tar.bz2
```

完成解压后，别忘了设置权限和所有者。Linux 的权限模型看上去无比复杂，实际上执行起来非常 Naive。根据 [ownCloud 官方指引](https://doc.owncloud.org/server/9.1/admin_manual/installation/installation_wizard.html#strong-perms-label)，安全的 ownCloud 目录应该是以下情况：

- 以下五个目录所有者应为 `[HTTP user]:[HTTP group]`：
  - `apps/`
  - `config/`
  - `themes/`
  - `assets/`
  - `data/`
- 以下目录和文件应为 `root:[HTTP user]`，并且权限位置于 `-rw-r--r--`（0644）：
  - `owncloud/.htaccess`
  - `owncloud/data/.htaccess`

对于更加安全的措施（比如基于 SELinux 的强制访问保护），参见文末附录。

## 配置服务器

### Apache httpd

#### 模块

于 `httpd.conf` 中检查。PHP 模块名为 `libphp5.so` 或（PHP 7）`libphp7.so`，一般位于 PHP 软件包中。

- php
- ssl
- env
- headers

#### 配置

- `ServerName`：服务器名称。设定到服务器的域名，如果没有域名，写 IP。
- `DocumentRoot [dir]`：网页服务器目录。指定网站根目录。**对应的 `<Directory>` 标签也要更改到 `DocumentRoot` 中，否则会应用全局的 Deny 规则，导致网页服务器无法正常运行。**

对于 ownCloud 所在目录，以下选项是需要的：

```xml
<Directory /path/to/owncloud/>
  Options +FollowSymlinks
  AllowOverride All

  <IfModule mod_dav.c>
    Dav off
  </IfModule>

  SetEnv HOME /var/www/owncloud
  SetEnv HTTP_HOME /var/www/owncloud

</Directory>
```

配置完成后，使用 `sudo systemctl start httpd`（CentOS 7 使用 `sudo systemctl start apache`）来启动 Apache httpd 服务器。将 `start` 替换为 `enable` 可使其自启动。

### PHP

ownCloud 对 PHP 的最低要求是版本 5.4。PHP 5.5 或以上版本本文讨论 `APCu` 作为 Memcache 模块，而 PHP 版本 5.4（CentOS 7 和 Ubuntu 等发行版提供）则需要使用 `APC` Memcache 模块，安装方法大致一样。经本人测试，PHP 7 也可以运行 ownCloud，所以推荐使用 PHP 7，性能提升比较大。

#### 模块

注意：**粗体字**表示这是一个 PHP Core 模块，一般都预先安装好了，或者可以通过包管理安装对应包。如果两者都没有，那就真没有了，**你需要重新编译 PHP 来开启 PHP Core 模块**。`php -m` 可以检查加载了的模块，查看 `php.ini` 和 PHP 模块目录可以知道已经安装了的模块，使用 `pecl`（PHP 的包管理）可以找到动态安装的模块。

以下模块是运行 ownCloud 所必需的 PHP 模块。

- **ctype**
- dom
- **GD**
- **iconv**
- JSON
- libxml
- multibyte
- **posix**
- SimpleXML
- xmlwriter
- **zip**
- **zlib**
- **curl**

对于 PHP 到数据库的连接，**以下模块三选一**。本文只讨论 MySQL(MariaDB)，所以可以只选 `pdo_mysql`。

- **pdo_mysql**
- **sqlite**
- **pgsql**

某些操作会用到以下模块，如果确认不会用到可以不开。

- **ldap**
- smbclient
- **ftp**
- **imap**
- **exif**
- **gmp**

如果开启文件预览（缩略图）会用到以下模块：

- imagick

如果开启内存缓存（Memcache），**以下模块四选一**。本文只讨论 APCu。

- APC
- APCu
- memcache
- redis

以下这个模块貌似可以让你在 php 下响应终端命令（比如说 Ctrl+C）。我也不知道有没有用。

- pcntl

#### 配置

调整 PHP 的某些设置可以让 PHP 更加安全。参见 [PHPSec][phpsec] 网站提供的信息来打造一个安全的 PHP 环境。

- `open_basedir = ...`：仅允许 PHP 在指定目录下运行。**注意：为了达到最大安全性，ownCloud 强烈建议在这之中包含 `/dev/urandom` 设备以更好地利用熵值。**
- `display_errors = Off`：不允许 PHP 在遇到错误时把错误信息发给用户。
  - `log_errors = On`：这个选项应该被开启。
- `expose_php = Off`：不允许 PHP 暴露自己。
- `allow_url_fopen = Off`：不允许老式的 fopen 操作，比较危险，应该使用 `curl` 库代替。
- `memory_limit = 128M`：设置内存使用上限。按照 PHPSec 的说法设置为 8M 只会把 ownCloud 废掉，所以 128M 就可以了，这也是 PHP 的默认值。

以下两个配置将会影响到 ownCloud 的“最大上传限制”设置，因此请将 `[size]` 替换成这个数值。

- `post_max_size = [size]`
- `upload_max_filesize = [size]`

[phpsec]: http://phpsec.org/projects/phpsecinfo/tests/

### MySQL (MariaDB)

[MariaDB][mariadb] 是一款 [MySQL][mysql] 的开源替代品，使用方法和 [MySQL][mysql] 是一样的。

[mariadb]: https://mariadb.org/
[mysql]:   https://www.mysql.com/

#### 配置实例

首先确认本机安装有 MariaDB。在 CentOS 7 中你需要安装 `xxx` 软件包来获得 MariaDB 的所有功能。然后执行：

```
sudo mysql_install_db --user=[user] --basedir=/usr --datadir=/var/lib/mysql
```

来安装数据库实例。其中 `--user=[user]` 指定执行 `mysqld` 守护进程的用户；`--basedir=/usr` 指定 [MariaDB][mariadb] 安装目录，一般就是 `/usr`；`--datadir=/var/lib/mysql` 指定数据库存放区域。

安装完数据库实例后，执行

```bash
sudo mysql_secure_installation
```

来 Harden [MariaDB][mariadb] 的安装，增强安全性。设置 root 密码后一路 `y` 即可。完成配置后使用 `sudo systemctl start mysql`（CentOS 7 使用 `sudo systemctl start mariadb`）打开 MariaDB 守护进程。将 `status` 替换为 `enable` 可以使其自启动。

#### 配置用户和数据库

下面为 ownCloud 新建一个专用用户和数据库。执行：

```bash
mysql -u root -p
# 没有配置 root 密码的话不需要 -p 开关
# mysql -u root
```

然后输入配置好的 root 密码来进入 MySQL 控制台。**没有配置数据库 root 密码的勇士请允许我向你致敬。**

执行以下 SQL 命令来创建用户、数据库以及设置权限：

```sql
CREATE USER '[user]'@'localhost' IDENTIFIED BY '[password]'
CREATE DATABASE [dbname]
GRANT ALL PRIVILEGES ON [dbname].* TO '[user]'@'localhost' IDENTIFIED BY '[password]'
```

其中除 `localhost` 是主机名，一般不需要改动外，根据实际情况填入以下字段：

- `[user]` 是 ownCloud 将使用的数据库用户名；
- `[password]` 是用户密码；
- `[dbname]` 是数据库名称；

至此服务器配置完成。

## 配置 ownCloud

进入 ownCloud 页面即可完成基本配置，若没有警告则可顺利使用 ownCloud。以下介绍各项功能优化选项。

### 缓存（Memcache）

Memcache 可以将多次请求的内容放在内存里减少 I/O 请求以达到加速后来请求的目的，对性能提升不少，建议开启。因为我们要建立的是一台简单的 ownCloud 服务器，所以我们选用 PHP 的 `APCu` Memcache 模块，配置较 ownCloud 支持的 [`memcached`][memcached] 和 [`redis`][redis] 简单。在 Ubuntu 下，安装 `php5-apcu/trusty-backports` 包。**Ubuntu 主仓库提供的 `APCu` 版本较老（4.0.2），而 ownCloud 需要 `APCu` 版本大于等于 4.0.6，所以要从 `trusty-backports` 仓库安装。**

如果源中没有 `APCu` 包（例如 CentOS 7），则需要从 `pecl` 中下载。先安装好 `pecl` 包，然后执行

```bash
pecl install APCu
# CentOS 7 使用 APC 因为 PHP 版本 5.4
pecl install APC
```

**注意：下载的 `APCu` 以及 `APC` 模块是需要编译的，因此使用这种方法请确保系统已经安装好以下软件包：**

- `autoconf`
- `gcc`

[memcached]: https://memcached.org/

安装完成 `APCu` 后，编辑 ownCloud 目录中的 `config/config.php` 文件，在其中加入一行：

```php
<?php
$CONFIG = array ( // 注意：填在这个数组内
  // ...
  'memcache.local' => '\OC\Memcache\APCu'
  // APC 则填写 APC
  // 'memcache.local' => '\OC\Memcache\APC'
  // ...
);
```

之后 ownCloud 就可以使用 Memcache 加速请求了。

### 改变后台任务执行方式

ownCloud 有一些后台程序用以执行更新缓存、清除垃圾等自维护行为。默认情况下 ownCloud 使用 Ajax 方式执行后台程序：一个请求一次。这是不被推荐的，请求数量一多，后台任务的数量就多（而且还是一模一样的），造成不必要的开销。ownCloud [在其官方指引中](https://doc.owncloud.org/server/9.1/admin_manual/configuration_server/background_jobs_configuration.html)推荐使用 Cronjob 来每 15 分钟执行一次后台任务，在此我们配置 Cronjob。

执行：

```
crontab -u [HTTP User] -e
```

这将打开一个编辑器（应该是 `nano`）。在其中输入：

```
*/15 * * * * php -f /path/to/owncloud/cron.php
```

`/path/to/owncloud` 替换为 ownCloud 所在目录。保存退出后可执行：

```
crontab -u [HTTP User] -l
```

来检查是否启用任务。之后我们需要在 ownCloud 中选择 Cron 任务执行类型，打开“管理”界面找到“后台任务”选项设置为 Cron 即可。

### 服务器端加密（不推荐开启）

ownCloud 有一个选项用于开启服务器端加密，可以对上传的文件进行加密处理。**我个人不推荐这么做，原因如下：**

- 加密之后文件比原来大 35% 左右（参见[官方资料](https://doc.owncloud.org/server/9.1/admin_manual/configuration_files/encryption_configuration.html)）；
- 使用 ownCloud 模块会给系统带来更多开销；
- 真的 Care 安全性的话~~不如跳舞~~，使用全盘加密效果会更好。

### 二步验证（2FA）

ownCloud 可以打开二步验证（双因子认证，Two-Factor Authentication）来通过验证器使得登录更加安全。要使用 2FA 首先需要在 ownCloud 的“应用”中打开这个插件。单击左上角的倒三角，选择“应用”，**并打开左下角隐藏菜单中的“使用实验性功能”**，在“工具”菜单中找到 TOTP 应用，启用即可在用户的个人页面中选择开启 2FA。

### 电子邮件

ownCloud 可以在用户分享文件、密码丢失等情况下向用户发送邮件。可以选择自建邮件服务器，但是自建服务器容易被外部邮箱判断为垃圾邮件，因其信任度低。ownCloud 支持以内建 SMTP、PHP SMTP 以及系统内置 sendmail 三种方式发送邮件。

使用域名邮箱服务可以借大服务商的邮箱发送自己的邮件。本文不介绍具体操作。

# 附录

## 启用 HTTP 安全连接（HTTPS）

### 配置 SELinux
