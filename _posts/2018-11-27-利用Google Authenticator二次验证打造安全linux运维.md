---
layout:     post
title:      利用Google Authenticator二次验证打造安全linux运维
subtitle:   实践
date:       2018-11-27
author:     dbstack
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - linux安全
---
## 利用Google Authenticator二次验证打造安全linux运维
linux服务器安全至上，为了防止黑客非法暴力破解linux密码，我们可以利用Google Authenticator二次验证打造安全服务器远程登录，具体配置如下：
# 一、关闭 SELinux
````shell
[root@wyw ~]# vim /etc/selinux/config      #永久关闭。需要reboot重启后生效
SELINUX=disabled
[root@wyw ~]# setenforce 0   #临时性关闭。不需要reboot重启
````
# 二、安装Google Authenticator 
````shell
1、配置好服务器epel源和bases源后，安装相关依赖包
[root@wyw ~]# yum install pam-devel libpng-devel wget gcc make mercurial -y
2、下载libpam-google-authenticator-1.0-source.tar.bz2、qrencode-3.4.4.tar.gz软件包
官方github地址
https://github.com/google/google-authenticator-libpam
百度网盘地址：链接：https://pan.baidu.com/s/1pr7JPHjU0mMbXZuwOETTwQ   提取码：m3zp 
3、安装google authenticator PAM插件
[root@wyw ~]# cd /opt/google
[root@wyw google]# tar -jxvf libpam-google-authenticator-1.0-source.tar.bz2
[root@wyw google]# cd libpam-google-authenticator-1.0
[root@wyw libpam-google-authenticator-1.0]# make && make install
4、安装QrenCode，此工具可以在Linux命令行下生成二维码
[root@wyw ~]# cd /opt/google/
[root@wyw google]# tar -zvxf qrencode-3.4.4.tar.gz
[root@wyw google]# cd qrencode-3.4.4
[root@wyw qrencode-3.4.4]# ./configure --prefix=/usr
[root@wyw qrencode-3.4.4]# make && make install
````
# 三、配置ssh服务调用google authenticator PAM插件
````shell
[root@wyw ~]# vim /etc/pam.d/sshd       #在第一行（即auth       required pam_sepermit.so的下一行）增加以下代码    
auth required pam_google_authenticator.so
[root@wyw ~]# vim /etc/ssh/sshd_config
ChallengeResponseAuthentication yes          #修改no为yes 
[root@wyw ~]# service sshd restart 
````
# 四、使用google authenticator PAM插件为ssh登录账号生成动态验证码
````shell
注意：哪个账号需要动态验证码，请切换到该账号下操作。（可以在不同用户下执行这个命令以生成各自的二次验证码）
[root@wyw ~]# google-authenticator
[root@wyw qrencode-3.4.4]#  google-authenticator

Do you want authentication tokens to be time-based (y/n) y
https://www.google.com/chart?chs=200x200&chld=M|0&cht=qr&chl=otpauth://totp/root@wyw%3Fsecret%3DHLGYC5FTL4WKEGNE  #这个链接只能在FQ条件下才能打开                  
Your new secret key is: HLGYC5FTL4WKEGNE  #如果在手机的谷歌身份验证器上不想通过"扫描条形码"的方式添加，就输入这个key，通过"手动输入验证码的方式"。账号就是服务器主机
Your verification code is 237334
Your emergency scratch codes are:  #下面会生成5个紧急验证码(当无法获取动态验证码或验证码不能使用使用可以使用这5个)
  23037330                         #需要注意的是：这5个验证码用一个就会少一个！请保存好！
  22153353
  56825949
  98098052
  63067449

Do you want me to update your "/root/.google_authenticator" file (y/n) y  #提示是否要更新验证文件，选择y
Do you want to disallow multiple uses of the same authentication
token? This restricts you to one login about every 30s, but it increases
your chances to notice or even prevent man-in-the-middle attacks (y/n) y #禁止使用相同口令

By default, tokens are good for 30 seconds and in order to compensate for
possible time-skew between the client and the server, we allow an extra
token before and after the current time. If you experience problems with poor
time synchronization, you can increase the window from its default
size of 1:30min to about 4min. Do you want to do so (y/n) n #默认动态验证码在30秒内有效，由于客户端和服务器可能会存在时间差，可将时间增加到最长4分钟，是否要这么做：这里选择是n，继续默认30秒

If the computer that you are logging into isn't hardened against brute-force
login attempts, you can enable rate-limiting for the authentication module.
By default, this limits attackers to no more than 3 login attempts every 30s.
Do you want to enable rate-limiting (y/n) y   #是否限制尝试次数，每30秒只能尝试最多3次，这里选择y进行限制
````
# 五、手机安装Google身份验证器，通过此工具扫描上一步生成的二维码图形，获取动态验证码。
````shell
在App Store里直接可以下载Authenticator
扫码增加或者手动增加即可

[root@node1 ~]$ ssh root@10.0.52.222
The authenticity of host '[10.0.52.222]:22 ([10.0.52.222]:22)' can't be established.
RSA key fingerprint is 5c:e7:1a:05:8b:2e:66:99:20:90:1f:47:56:bf:b9:41.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[10.0.52.222]:22' (RSA) to the list of known hosts.
Verification：
Password:
[root@wyw ~]#
````
# END
