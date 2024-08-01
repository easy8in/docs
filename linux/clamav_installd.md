# Clamscan 杀毒安装和使用

## 安装

```bash
# 软件源
yum install epel-release

# 安装
yum install clamav-server clamav-data clamav-update clamav-filesystem clamav clamav-scanner-systemd clamav-devel clamav-lib clamav-server-systemd -y

# 更新病毒库
sudo freshclam

# 全盘扫描 病毒文件移动到/opt/tmp 目录下
sudo clamscan -i -r --move=/opt/tmp /
```

```bash
#最后 ，使用chkrootkit、clamav、rkhunter一通查杀，当然，还是重装系统最保险。

ps auxf
ps -ef | grep ${pid}
netstat -natlp | grep ${pid}
lsof -c
strace -T -tt -p ${pid}

#登录失败
cat /var/log/secure* | grep Failed
#登录成功
cat /var/log/secure* | grep Accepted

/etc/hosts.allow和/etc/hosts.deny

#定时任务配置
/etc/cron.d/
/var/spool/cron/crontabs
```

```bash
# 查找异常进程
ps auxf
root       4755  0.0  0.0   2404   104 ?        Ssl  9月09   0:08 y3wkYwSJ
root     193608  397  0.0 2443316 4140 ?        Ssl  15:06  98:28 AOj0l8HP
# 找到进程名 随机英文名的 y3wkYwSJ AOj0l8HP
kill -9 4755 193608

# 检查定时任务
# 查询有登录权用户
cat /etc/passwd | grep /bash
# 检查用户定时任务
crontab -l
crontab -l -u ?
cd /etc/cron.d
rm -rf 0system*
cd /var/spool/cron/
rm -rf *
systemctl restart crond

# 删除脚本文件
# 查看程序启动目录
ls -al /proc/${pid} | grep cwd
# cd 启动目录
ls -al 
rm -rf 
# 查看
/opt/
rm -rf 类似脚本

# 查看程序启动脚本
ls -al /proc/${pid} | grep exe
# 存在则删除
top 检查
reboot检查
```