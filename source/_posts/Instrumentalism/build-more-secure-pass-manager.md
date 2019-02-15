---
title: 打造基于git的跨平台密码管理器
date: 2019-02-15 09:29:40
tags:
  - git
  - gunpg
  - password manager
  - pass
---





现在互联网的发达程度，导致我们对需要管理并维护很多属于自己的账号信息。很多人为了自己方便起见，将自己的所有账户的密码设置为同一个。这样虽然方便，但一旦一个账户的密码泄露，所有的账号都会被一锅端。因此密码管理工具应运而生。

当前最流行的密码管理工具应该就是1Password与LastPassword。然而作为商业软件，这种软件不仅仅需要付费使用，同时用户需要完全信任这种软件运营开发公司，由于其代码封闭，难以审计，很难说在你将自己的密码交给他之后会被做什么。

因此，结合现有的开源方案，在这里提出一套可以媲美1Password等商业方案的密码管理解决方案。

总体来说这套方案适用于如下需求：

- 希望免费拥有一套免费密码管理方案；
- 不信任1Password，LastPassword这种中心化密码管理方案；
- 有一定的技术能力。

<!--more-->

## 使用工具

这套方案需要使用如下几个工具相互配合来实现：

- git - 分布式版本管理软件，用于管理加密后的密码库，同时负责跨平台的同步密码库。

- pass - 管理并组织密码库的软件，提供多种跨平台客户端。
- gunpg - OpenPGP的实现，负责加密认证密码库。
- github - 作为密钥库存储的中央仓库，github现在提供免费的私有仓库，也可以采用其他的托管仓库。

 ## 安装与配置

由于不同操作系统之间的差异，因此在安装、配置的时候需要根据不同的操作系统区别进行。

首先需要拥有一个github仓库或者其他支持git协议的托管仓库。

### Linux

区别于不同的发行版，Linux可以选择并采用不同的方式去安装软件。需要安装的软件包括如下：

- [pass](https://www.passwordstore.org/) - 原始版本的pass密码管理器
- [gnupg2](https://www.gnupg.org/) - GPG2，必须
- [git](https://git-scm.com/) - 版本管理工具，必须
- [qtpass](http://qtpass.org/) - 图形化pass密码管理器，与原始版本pass二选其一即可，也可以共存。

首先你需要创建一个gpg密钥，可以生成一个新的，也可以将你已有的gpg密钥导入当前电脑上。

如果你采用图形化密码管理器qtpass，参见Windows系统下的配置方法；如果采用命原始的pass密码管理器，首先需要初始化你的密码存储库：

```shell
$ pass init "<你的gpg公钥ID>"
```

之后你可以利用git初始化这个密钥仓库

```shell
$ pass git init
```

之后添加托管仓库

```shell
$ pass git remote add origin <仓库的远程地址>
```

Linux下默认的密码库会被存储在`~/.password-store`目录下。

### Windows

windows系统下可以避免在命令行下工作，使用图形化界面完成操作，除了git之外。

首先你需要下载并安装如下软件：

- [gpg4win](https://www.gpg4win.org/) - windows上的gpg加密软件（可选图形化界面）；
- [TortoiseGit](https://tortoisegit.org/) - windows上的git软件（图形界面）；
- [qtpass](http://qtpass.org/) - 图形化pass密码管理器。

在安装好之后，首先配置git，如果你之前使用过git，并且完成过配置可以忽略此步骤；

之后使用gpg4win提供的图形化密钥管理软件Kleopatra生成或导入属于自己的gpg密钥；

使用TortoiseGit在需要放置密码库的位置初始化git仓库，并添加托管仓库的远程地址；

最后运行QtPass，在设置页面的用用户选项卡中设置密码库。

### Android

安卓系统下使用，只需要两个软件即可，分别是：

- Password Store - pass密码管理器的安卓版，自带git版本管理；
- OpenKeyChain - gpg的安卓版本，管理私钥。

安卓上配置则较为简单，只需要生成或导入gpg密钥，同时在Password Store中配置远端托管仓库即可。

### iOS

iOS端可以使用pass-password-store来管理并同步属于你的密码。

## 生成密码

不同平台的采用的客户端不相同，则生成新密码的方式各不相同。pass将密码库中的密码以文件夹的形式进行管理，你可以自由的选择密码的组织方式。

### pass 命令

如果你使用pass命令来生成新密码，则在shell中执行：

```shell
$ pass generate Email/aliyun.com 15
```

这个命令会在Email目录下生成一个名为aliyun.com的文件，同时会在命令行输出一个长度为15的随机密码。整个文件会采用gpg加密存储。当然你需要在生成的时候输入gpg的口令。

在生成新密码之后，你需要手动使用git管理密码库：

```shell
$ pass git add -A
$ pass git commit -m "add password"
$ pass git push -u origin master
```

### QtPass

如果你使用qtpass来进行图形化的管理，则只需要点击新密码按钮，之后生成你需要长度的密码，点击生成即可生成新密码。当你点下确认保存密码之后，QtPass会自动帮你管理提交密码库到远端托管仓库中。

### Password Store

Password Store可以非常轻易的创建新的密码，生成密码之后只需选择功能菜单中的git pull与git push就可以与托管仓库进行同步。

## OTP

OTP多用于两步验证过程，pass也是可以支持OTP，根据不同的实现，支持程度略有不同。pass命令需要安装otp扩展，Password Store可以直接支持，而QtPass则完全不支持。

在添加OTP时，网站上往往会给出一个二维码，这个二维码中记录了一个字符串，用于描述OTP的具体细节。在pass中使用时，由于pass管理的密码库本质上就是一个经过加密的文本文件，因此只需将二维码中的字符串写在文件中独立的一行即可。凡是可以使用谷歌手机验证器进行的两步验证都可以在pass中管理。

pass命令使用的扩展可以使用如下命令添加新的OTP：

```shell
$ pass otp insert Email/aliyun.com # 创建一个新的密码文件用于记录otp，存储在Email/aliyun.com
$ pass otp append Email/aliyun.com # 添加otp到密码文件Email/aliyun.com中
```

生成新的OTP码只需要：

```shell
$ pass otp Email/aliyun.com
```

## Reference

- [pass](https://www.passwordstore.org/)
- [pass-otp](https://github.com/tadfisher/pass-otp)
- [GPG入门教程](http://www.ruanyifeng.com/blog/2013/07/gpg.html)

