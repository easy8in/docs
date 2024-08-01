## 安装前准备

```bash
#添加被虚拟用户挂载的ftp用户
groupadd ftpadmin
useradd ftpadmin -m -s /bin/bash -d /home/ftpadmin -g ftpadmin
passwd ftpadmin

ssh ftpadmin@127.0.0.1
mkdir ftp
exit
```

## 安装

```shell
yum install vsftpd
```

## 编辑配置文件替换内容

```bash
cp /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.bak
rm /etc/vsftpd/vsftpd.conf
vim /etc/vsftpd/vsftpd.conf
:set paste
#输入 i 粘贴
```

```tex
# Example config file /etc/vsftpd/vsftpd.conf
#
# The default compiled in settings are fairly paranoid. This sample file
# loosens things up a bit, to make the ftp daemon more usable.
# Please see vsftpd.conf.5 for all compiled in defaults.
#
# READ THIS: This example file is NOT an exhaustive list of vsftpd options.
# Please read the vsftpd.conf.5 manual page to get a full idea of vsftpd's
# capabilities.
#
# Allow anonymous FTP? (Beware - allowed by default if you comment this out).
anonymous_enable=NO
#
# Uncomment this to allow local users to log in.
# When SELinux is enforcing check for SE bool ftp_home_dir
local_enable=YES
#
# Uncomment this to enable any form of FTP write command.
write_enable=YES
#
# Default umask for local users is 077. You may wish to change this to 022,
# if your users expect that (022 is used by most other ftpd's)
local_umask=022
#
# Uncomment this to allow the anonymous FTP user to upload files. This only
# has an effect if the above global write enable is activated. Also, you will
# obviously need to create a directory writable by the FTP user.
# When SELinux is enforcing check for SE bool allow_ftpd_anon_write, allow_ftpd_full_access
#anon_upload_enable=YES
#
# Uncomment this if you want the anonymous FTP user to be able to create
# new directories.
#anon_mkdir_write_enable=YES
#
# Activate directory messages - messages given to remote users when they
# go into a certain directory.
dirmessage_enable=YES
#
# Activate logging of uploads/downloads.
xferlog_enable=YES
#
# Make sure PORT transfer connections originate from port 20 (ftp-data).
connect_from_port_20=YES
#
# If you want, you can arrange for uploaded anonymous files to be owned by
# a different user. Note! Using "root" for uploaded files is not
# recommended!
#chown_uploads=YES
#chown_username=whoever
#
# You may override where the log file goes if you like. The default is shown
# below.
#xferlog_file=/var/log/xferlog
#
# If you want, you can have your log file in standard ftpd xferlog format.
# Note that the default log file location is /var/log/xferlog in this case.
xferlog_std_format=YES
#
# You may change the default value for timing out an idle session.
#idle_session_timeout=600
#
# You may change the default value for timing out a data connection.
#data_connection_timeout=120
#
# It is recommended that you define on your system a unique user which the
# ftp server can use as a totally isolated and unprivileged user.
#nopriv_user=ftpsecure
#
# Enable this and the server will recognise asynchronous ABOR requests. Not
# recommended for security (the code is non-trivial). Not enabling it,
# however, may confuse older FTP clients.
#async_abor_enable=YES
#
# By default the server will pretend to allow ASCII mode but in fact ignore
# the request. Turn on the below options to have the server actually do ASCII
# mangling on files when in ASCII mode. The vsftpd.conf(5) man page explains
# the behaviour when these options are disabled.
# Beware that on some FTP servers, ASCII support allows a denial of service
# attack (DoS) via the command "SIZE /big/file" in ASCII mode. vsftpd
# predicted this attack and has always been safe, reporting the size of the
# raw file.
# ASCII mangling is a horrible feature of the protocol.
#ascii_upload_enable=YES
#ascii_download_enable=YES
#
# You may fully customise the login banner string:
#ftpd_banner=Welcome to blah FTP service.
#
# You may specify a file of disallowed anonymous e-mail addresses. Apparently
# useful for combatting certain DoS attacks.
#deny_email_enable=YES
# (default follows)
#banned_email_file=/etc/vsftpd/banned_emails
#
# You may specify an explicit list of local users to chroot() to their home
# directory. If chroot_local_user is YES, then this list becomes a list of
# users to NOT chroot().
# (Warning! chroot'ing can be very dangerous. If using chroot, make sure that
# the user does not have write access to the top level directory within the
# chroot)
#chroot_local_user=YES
#chroot_list_enable=YES
# (default follows)
#chroot_list_file=/etc/vsftpd/chroot_list
#
# You may activate the "-R" option to the builtin ls. This is disabled by
# default to avoid remote users being able to cause excessive I/O on large
# sites. However, some broken FTP clients such as "ncftp" and "mirror" assume
# the presence of the "-R" option, so there is a strong case for enabling it.
#ls_recurse_enable=YES
#
# When "listen" directive is enabled, vsftpd runs in standalone mode and
# listens on IPv4 sockets. This directive cannot be used in conjunction
# with the listen_ipv6 directive.
listen=YES
#
# This directive enables listening on IPv6 sockets. By default, listening
# on the IPv6 "any" address (::) will accept connections from both IPv6
# and IPv4 clients. It is not necessary to listen on *both* IPv4 and IPv6
# sockets. If you want that (perhaps because you want to listen on specific
# addresses) then you must run two copies of vsftpd with two configuration
# files.
# Make sure, that one of the listen options is commented !!
listen_ipv6=NO

pam_service_name=vsftpd
userlist_enable=YES
tcp_wrappers=YES
chroot_local_user=YES
allow_writeable_chroot=YES

guest_enable=YES
guest_username=ftpadmin
virtual_use_local_privs=YES
user_config_dir=/etc/vsftpd/vconf

#使用被动模式
pasv_enable=YES
#被动模式端口范围
pasv_min_port=11240
pasv_max_port=22480
#需要加入外部ip，否则被动模式会失败
#pasv_address=39.96.38.179
#pasv_addr_resolve=YES
#取消DNS反向解析
reverse_lookup_enable=NO
```

## 创建虚拟用户配置存放路径

```bash
mkdir /etc/vsftpd/vconf
```

## 创建虚拟用户名单文件，“奇数行用户名，偶数行口令”。

```bash
vim /etc/vsftpd/vconf/virtusers
#输入下面内容保存退出
sxwt
sxwt@123
jdcall
jdcall@123

#生成用户数据库
db_load -T -t hash -f /etc/vsftpd/vconf/virtusers /etc/vsftpd/virtusers.db
```

注意：每次添加修改用户后都需要重新生成

## 修改 vsftpd pam 文件

```bash
#备份
cp /etc/pam.d/vsftpd /etc/pam.d/vsftpd.bak
vim /etc/pam.d/vsftpd
#注释掉其他的配置、只保留自己的配置
auth    required /lib64/security/pam_userdb.so db=/etc/vsftpd/virtusers
account required /lib64/security/pam_userdb.so db=/etc/vsftpd/virtusers
```

## 创建 ftp 用户配置模板

```bash
vim /etc/vsftpd/vconf/vconf.tmp
#输入下面内容
#虚拟用户存放内容路径
local_root=/home/ftpadmin/ftp
anonymous_enable=NO
write_enable=YES
local_umask=022
anon_upload_enable=NO
anon_mkdir_write_enable=NO
idle_session_timeout=600
data_connection_timeout=120
max_clients=20
max_per_ip=20
local_max_rate=50000

#创建用户配置
cp vconf.tmp sxwt #sxwt是用户名,只能是在用户文件中存在的用户名
```

## 启动并测试

```bash
#启动
systemctl start vsftpd
#查看状态
systemctl status vsftpd
#开机启动
systemctl enable vsftpd
#关闭selinux
vim /etc/selinux/config
SELINUX=disable
#开放防火墙端口
firewall-cmd --add-service=ftp --permanent
#重启(生效关闭selinux否则连接会失败)
reboot

#测试
yum install -y ftp
ftp 127.0.0.1
sxwt
sxwt@123
```