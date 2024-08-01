**一、上传木马的过程**

1、默认端口 22 弱口令暴力破解；

2、21 端口或者 3306 端口弱口令暴力破解；

3、webshell 进行 shell 反弹提权；

4、木马传入服务器的上面并且执行，通过木马的方式来控制你的服务器进行非法的操作。

**二、常见操作**

1、切入/tmp；

2、wget 下载木马；

3、木马加载权限；

4、执行木马；

5、后门，支持木马复活。

**三、清除木马**

1、网络连接，过滤掉正常连接；

\# netstat -nalp | grep "tcp" | grep -v "22" | grep "ESTABLISHED"

2、判断一些异常连接，通过 PID 找到进程名称；

\# ps -ef | grep "27368"

3、通过名字，找到原文件，删除掉原文件。

**四、清除后门**

1、检查/etc/rc.local；

2、检查计划任务 crontab -l；

3、检查/root/.bashrc 和普通用户下的.bashrc；

4、检查/etc/profile 文件定期进行 md5 校验。

**五、安全加固**

1、了解常见的扫描和提权端口

 -22 端口暴力破解

 -21 端口提权

 -3306 端口提权

 -webshell 反弹

2、如何对 linux 进行安全加固

 2.1 进程数量监控及对比

  2.1.1、进程数量

  2.1.2、进程异常的名称及 PID 号

2.1.3、根据 PID 号进行查询网络连接异常

写一个脚本:

将服务器正常的进程号，导入到一个目录，取个名字叫做原始.log，提取实时进程名称>实时.log，通过 diff 去对比原始和实时的 log 区别，一旦发现对比不一样，通过名称得到 PID 号，然后通过 PID 查找网络连接和监听的端口号及 IP 地址。将这些信息发送告警到管理员的手机或者邮箱邮件里面，让管理员进行判断和分析。

 2.2、计划任务列表监控

2.2.1、查看计划任务

2.2.2、监控/var/log/cron 日志

 2.3、用户登录监控

2.3.1、什么用户登录的？在什么时候登录的？

2.3.2、用户登录 IP 是否合法？

2.3.3、用户登录的用户名是否合法？

2.3.4、用户登录时间是否合法？

 2.4、/etc/passwd、/etc/shadow MD5

2.4.1、MD5 校验防止有人更改 passwd 和 shadow 文件

2.4.2、passwd 文件可以进行加锁 chattr 权限

2.4.3、passwd 定期进行备份，进行内容 diff 对比

 2.5、非有效用户登录 shell 权限

2.5.1、除了运维常用的维护账号以外，其他账户不能拥有登录系统的 shell

2.5.2、针对/etc/passwd 文件统计 bash 结尾的有多少个？将不用的改成/sbin/nologin

2.5.3、将不必要的账户删除或者锁定

 2.6、安全日志分析与监控/var/log/secure

2.6.1、定期或者实时分析/var/log/secure 文件，是否有暴力破解和试探

2.6.2、过滤 Accepted 关键字，分析对应的 IP 是否为运维常用 IP 及端口号和协议。否则视为已经被入侵。

2.6.3、定期备份/var/log/secure 防止此人入侵后，更改和删除入侵目录

 2.7、/etc/sudoers 监控

2.7.1、防止对方通过 webshell 反弹的方式，增加普通用户到/etc/sudoers

2.7.2、定期备份/etc/sudoers 和监控，发现特殊的用户写入此文件，视为已被入侵。

2.7.3 此配置文件，普通用户可绕过 root 密码直接 sudo 到 root 权限

 2.8、网络连接数的异常

2.8.1、经常统计 TCP 连接，排除正常的连接以外，分析额外的 TCP 长连接，找出非正常的连接进程

2.8.2、发现异常进程后，通过进程名使用 top 的方式，或者 find 命令搜索木马所在的位置

2.8.3、找到连接所监听的端口号以及 IP，将此 IP 拉入黑名单，KILL 掉进程，删除木马源文件

2.8.4、监控连接监听，防止木马复活

 2.9、/etc/profile 定期巡检

2.9.1、检查/etc/profile 文件防止木马文件路径写入环境变量，防止木马复活

2.9.2、防止/etc/profile 调用其他的命令或者脚本进行后门连接

2.9.3、此文件进行 MD5 校验，定期备份与 diff 如发现异常则视为被入侵

 2.10、/root/.bashrc 定期巡检

2.10.1、检查/root/.bashrc 文件，防止随着 root 用户登录，执行用户变量，导致木马复活

2.10.2、防止/root/.bashrc 通过此文件进行后门创建与连接

2.10.3、此文件进行 MD5 校验，定期备份与 diff，如发现异常，则视为入侵

 2.11、常用端口号加固及弱口令

2.11.1、修改 22 默认端口号

2.11.2、修改 root 密码为复杂口令或禁止 root 用户登录，使用 key 方式登录

2.11.3、FTP 要固定 chroot 目录，只能在当前目录，不随意切换目录

2.11.4、mysql 注意修改 3306 默认端口号，授权的时候不允许使用% 号进行授权。

2.11.5、mysql 用户及 IP 授权请严格进行授权，除了 DBA,其他开发人员不应该知道 JDBC 文件对应的用户名和密码

 2.12、/tmp 目录的监控

2.12.1、由于/tmp 目录的特殊性，很多上传木马的第一目标就是/tmp

2.12.2、/tmp 进行文件和目录监控，发现变动及时警告

 2.13、WEB 层面的防护

  所有 WEB 层的安全成为首要，要定期进行 WEB 程序漏洞扫描，发现之后及时通知开人员修补，对于金融行业的有必要邀请第三方定期进行渗透测试。

**六、常见的安全网站**

乌云漏洞：[http://www.wooyun.org](http://www.wooyun.org/)

盒子漏洞：[https://www.vulbox.com](https://www.vulbox.com/)

国家信息安全漏洞平台：[http://www.cnvd.org.cn](http://www.cnvd.org.cn/)

Freebuf：[http://www.freebuf.com](http://www.freebuf.com/)

STACKOVERFLOW：[http://stackoverflow.com](http://stackoverflow.com/)

CVE 漏洞：[http://cve.mitre.org](http://cve.mitre.org/)

360BLOG：[http://blogs.360.cn](http://blogs.360.cn/)

OSR：[http://www.osronline.com](http://www.osronline.com/)

漏洞库：[https://www.exploit-db.com](https://www.exploit-db.com/)

CODEPROJECT：[http://www.codeproject.com](http://www.codeproject.com/)

**七、运维安全审计**

1、环境安全

2、物理链路的安全

3、网络安全

4、前端程序的安全

5、系统的安全

6、数据的安全

7、内部人员的安全