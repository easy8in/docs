```shell
# 列出所有时区
timedatectl list-timezones
# 设置时区为东8区
timedatectl set-timezone Asia/Shanghai
# 同步到硬件时钟
timedatectl set-local-rtc 1
```