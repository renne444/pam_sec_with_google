# 带有谷歌安全认证的服务器docker部署

## 基本环境部署

1. 物理机安装docker，并更改存放位置

在我的机子上，固态硬盘空间很宝贵，但机械硬盘空间却特别多。所以我更倾向于把集群装在机械硬盘上。

```bash
sudo apt-get install docker
sudo apt install docker.io
```

2. 更改docker文件存放位置

```bash
sudo vim /etc/docker/daemon.json
# 写入以下内容
{
  "registry-mirrors": ["http://hub-mirror.c.163.com"],
  "data-root": "/home/caros/personal/disk/docker"
}

# 重启两项服务
systemctl daemon-reload 
systemctl restart docker
```

3. 测试helloworld docker内容

```bash
sudo docker pull registry.docker-cn.com/library/ubuntu:18.04
```

## 编译谷歌pam认证源码

```bash
cd /root
mkdir google && cd google
# wget https://github.com/google/google-authenticator/archive/master.zip
wget https://github.com/google/google-authenticator-libpam/archive/refs/tags/1.09.zip
unzip 1.09.zip 
cd google-authenticator-libpam-1.09
./bootstrap.sh
./configure
make -j2
```

编译完成后执行安装

```bash
make install
```

其中用到以下几个文件：
- pam_google_authenticator.la，安装到/usr/local/lib/security
- .libs/pam_google_authenticator.so，安装到/usr/local/lib/security/pam_google_authenticator.so
- .libs/pam_google_authenticator.lai，安装到/usr/local/lib/security/pam_google_authenticator.la

ldconfig加入/usr/local/lib/security路径、

绑定谷歌验证器

```bash
google-authenticator
```

有个链接，显示了一个二维码，需要扫码输入里面的号。并且显示了一个secret key `YRIPZLNQ3OF4P5E4N2KSDOTGP4`。在google play上下载相应的软件`google身份验证器`，扫描上面显示的二维码即可。接下来显示了几个应急代码。

Your emergency scratch codes are:
  48852966
  21138117
  75827041
  19757626
  33947722

接着一直按y，设置了很多安全相关的选项。

## 将谷歌验证器链接到pam登陆，并在ssh开启

修改pam设置，我们需要修改ssh文件。与登陆相关的配置文件一共有3个：

|配置文件|效果|
|-|-|
|/etc/pam.d/login|终端登陆，就是直接tty这种|
|/etc/pam.d/sshd|远程登录|
|/etc/pam.d/system-auth|全局配置文件|

在/etc/pam.d/sshd写入一些内容

```config
auth required pam_google_authenticator.so
```

将外链接库地址写入/etc/ld.so.conf.d，并运行ldconfig命令


修改/etc/ssh/sshd_config，修改ChallengeResponseAuthentication，改为yes。然后重新启动服务，对于ubuntu需要重启的是openssh不是sshd，所以restart ssh。

```bash
service ssh restart
```

## P.S.

当遇到问题时，需要学会查系统日志。一共有两种比较方便的方式。

1. `journalctl -xe`。这种方式能查当前系统中的错误数据
2. `/var/log/auth`。这种方式能查和登陆相关的错误日志
