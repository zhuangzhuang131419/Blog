---
title: sftp服务器搭建
date: 2020-06-27 21:36:47
tags:
---
# 背景
## 1. Linux管理员(root)修改和查看普通用户的密码
### root修改普通用户密码
`-> sudo passwd user_name`
### root查看普通用户密码
密码是无法被查看的，即使是root也不行，因此普通用户要是遗忘了密码，可以参照上一步，让管理员使用root权限修改密码，然后再将新密码告知普通用户
### 普通用户修改自己的密码
`-> passwd`
## 2. 查看当前用户及用户组
### 可以查看所有用户的列表 
`-> cat /etc/passwd`
### 可以查看当前活跃的用户列表
`-> w`
### 查看用户组
`-> cat /etc/group`
### 查看当前登录用户名
`-> who am i`
### 查看用户属于哪一个用户组
`-> groups username`
# 开始搭建
需求：创建三个用户，其中一个为sftp管理员，其余两个分别为指定目录的访问用户。sftp管理员对其他用户的sftp根目录下的内容具有读写权限，限制其他用户只能访问其自己的根目录且仅有读权限；相关的sftp用户不能登录到Linux系统中。
## 1. 确认openssh的版本
`-> ssh -V`
![](/Users/zhuangzhuang/Blog/source/images/sftp-ssh.png)
## 2. 切换到管理员(root)
> 也可以不切换在下面的命令行前加`sudo`

`-> sudo -i`
## 3. 创建sftp管理组及用户组
> 添加管理组 bytedance_admin

`-> groupadd bytedance_admin `
> 添加用户组 bytedance

`-> groupadd bytedance `

## 4. 创建sftp管理用户及普通用户
>`/bin/false` 目的是不让用户登录 也可以使用`/bin/nologin`

`-> useradd -g bytedance -s /bin/false bob`

`-> useradd -g bytedance -s /bin/false john`

`-> useradd -g bytedance_admin -s /bin/false king`

> 为每一位用户设置密码

`-> passwd king`

`-> passwd bob`

`-> passwd john`

## 5. 分别创建对应用户的bytedance根目录并指定为其家目录

`-> mkdir -pv /usr/bytedance/{bob,john}/share`

## 6. 配置sshd_config文件

`-> vi /etc/ssh/sshd_config`

找到如下这行，用#符号注释掉，大致在文件末尾处。 

```
# Subsystem sftp /usr/libexec/openssh/sftp-server

Subsystem sftp internal-sftp     #这行指定使用sftp服务使用系统自带的internal-sftp

Match Group bytedance     #这行用来匹配bytedance组的用户，如果要匹配多个组，多个组之间用逗号分割；

	ChrootDirectory /usr/bytedance/%u        #用chroot将用户的根目录指定到%h，%h代表用户home目录，这样用户就只能在用户目录下活动。也可用%u，%u代表用户名。

	ForceCommand internal-sftp    #指定sftp命令 

	AllowTcpForwarding no

	X11Forwarding no

Match User bytedance_admin        #匹配用户了，多个用户名之间也是用逗号分割

	ChrootDirectory /usr/bytedance

	ForceCommand internal-sftp

	AllowTcpForwarding no

	X11Forwarding no

```

## 7. 设置Chroot目录的权限
### chown和chmod 命令
> chmod修改的是文件的读、写、执行。
> 
> chown修改的是文件的用户或者组的权限。

### 具体步骤

```
#修改普通用户的根目录属组

chown root:bytedance /usr/bytedance/{bob,john}
```
![](/Users/zhuangzhuang/Blog/source/images/sftp-chown.png)

第一个root表示文件所有者 第二个bytedance表示文件所在的群组

```
#修改普通用户的根目录权限
-> chmod 755 /usr/bytedance/{bob,john}  

#修改管理员的根目录属组
-> chown root:bytedance_admin /usr/bytedance/

#修改管理员根目录的权限
-> chmod 755 /usr/bytedance/

#修改各普通用户下的share目录的属主为管理员，属组为普通用户组
-> chown king:bytedance /usr/bytedance/{bob,john}/share/ 

#各share目录管理员的权限为读写，普通bytedance组仅有读权限，其他用户没有权限访问
-> chmod 750 /usr/bytedance/{bob,john}/share/

```

chmod 后面的数字含义参考

https://chmodcommand.com/

## 8. 关闭selinux
`-> vim /etc/selinux/config`

`SELINUX=permissive`

`-> setenforce 0`

如果提示 `setenforce: command not found`

解决方案:

* `apt-get install selinux-utils`

* 添加环境变量
`/usr/sbin/setenforce`

## 9. 重启sshd服务

`-> service sshd restart`

如果提示
`Job for ssh.service failed because the control process exited with error codesee systemctl status ssh.service and journalctl -xe for details.`
解决方案：

* 按照提示`systemctl status ssh.service`
	* ![](/Users/zhuangzhuang/Blog/source/images/sftp-sshd.png)
* `/usr/sbin/sshd -T`
	* 根据具体情况分析
	* ![](/Users/zhuangzhuang/Blog/source/images/sftp-sshd02.png)
	* 这里我的情况是把Match User写到了前面，导致后面的参数读取失败

**Tips:**
请务必解决上述问题，不然sshd重启出错将会导致之后本地无法通过ssh连接开发机

## 10. 验证sftp登录

```
#管理员登录，能对share目录下的文件进行读写操作

-> sftp king@127.0.0.1

Connecting to 127.0.0.1...
king@127.0.0.1's password: 
#输入之前的密码

sftp> 

#普通用户登录，对share目录下的文件只能进行读操作

-> sftp bob@127.0.0.1

bob@127.0.0.1's password: 

sftp> ls
share

```
验证登录出现`sftp>` 基本就说明了sftp服务器搭建成功了，剩下需要注意的就是权限问题了。此时也可以通过相关的ftp client 如：FileZilla FTP Client 和xftp 来连接到对应的sftp服务器了。

# 设置记录sftp服务器的登录及操作日志
 








# 参考文献
https://www.jianshu.com/p/6b588a712513






