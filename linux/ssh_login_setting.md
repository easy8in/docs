```shell
# client端
ssh-keygen
ssh-copy-id -i ~/.ssh/id_rsa.pub ${username}@{server-host}
# server端口
cat ~/.ssh/authorized_keys
```