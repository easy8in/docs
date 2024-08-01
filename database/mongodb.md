# MongoDB 操作手册

## 安装

### mongodb 版本 4.x

https://repo.mongodb.org/yum/redhat/8/mongodb-org/4.2/x86_64/

 https://www.mongodb.org/static/pgp/

编写 yum repo 文件 mongodb-org-4.2.repo （可以参考 https://repo.mongodb.org/yum/redhat/ 中的 repo 文件，$releasever指的是redhat版本，$releasever 会自己识别）

```bash
[mongodb-org-4.2]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.2/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.2.asc
```

保存到服务器这个目录下

```bash
/etc/yum.repos.d/
```

 执行如下命令安装 mongodb：

```bash
sudo yum install -y mongodb-org
```

禁用自动升级，修改/etc/yum.conf 文件，加入如下信息

```
exclude=mongodb-org,mongodb-org-server,mongodb-org-shell,mongodb-org-mongos,mongodb-org-tools
```

 查看安装版本

```bash
mongod --version
```

添加 linux 用户

```bash
useradd mongodb
passwd mongodb
```

切换用户

```bash
su mongodb
```

创建目录

```bash
mkdir data
mkdir log
mkdir run
mkdir etc
```

拷贝配置文件

```bash
cp /etc/mongod.conf ./etc/
```

修改配置文件

```bash
vim etc/mongod.conf
---
 path: /home/mongodb/logs/mongod.log
 dbPath: /home/mongodb/data
 pidFilePath: /home/mongodb/run/mongod.pid
 bindIp: 0.0.0.0
```

```bash

path: /root/mongodata/logs/mongod.log
dbPath: /root/mongodata/data
pidFilePath: /root/mongodata/run/mongod.pid
bindIp: 0.0.0.0
```

开启身份验证：

```bash
security: 
  authorization: enabled
```

编写启动脚本 start.sh

```bash
#!/bin/sh
mongod --config /home/mongodb/etc/mongod.conf
```

编写停止脚本 shutdown.sh

```bash
#!/bin/sh
mongod --config /home/mongodb/etc/mongod.conf --shutdown
```

添加开机启动

```bash
su root
vim /etc/rc.local
/home/mongodb/start.sh
```

## 开启 Mongodb 副本集

* 开启 Mongodb 副本集前，一定要先关闭安全验证，配成功后可能无法启动副本集，正确启动副本集后再开启安全验证。

配置副本集

```
vi /etc/mongod.conf
---
replication:
  replSetName: rs0
```

启动副本集

```bash
rs.initiate({_id:'rs0',members:[{_id:1,host:'192.168.1.130:27017'}]})
```

副本集读取用户配置，先在 admin 数据库中添加用户

db.addUser();

再

 db.grantRolesToUser( “solr_mongo”, [{ role: “read”, db: “local”}])