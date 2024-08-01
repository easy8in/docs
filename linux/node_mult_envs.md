# Nodejs 环境搭建

## nodejs 多版本管理使用 nvm 来管理

[Linux仓库](https://github.com/nvm-sh/nvm) [Window仓库](https://github.com/coreybutler/nvm-windows)

## Npm 安装加速方法

### 方法一

```bash
npm install -s [package-name] --registry=https://registry.npm.taobao.org
```

### 方法二

```bash
npm config ls
npm config set registry https://registry.npm.taobao.org
```

## Mac OS M1 搭建 Nodejs with brew

```shell
# 官方安装brew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
# git 仓库安装
git clone https://github.com/Homebrew/install.git
/bin/bash install/install.sh
brew help
# 安装nvm
brew install nvm
# vim .zshrc
export NVM_NODEJS_ORG_MIRROR=http://npm.taobao.org/mirrors/node
export NVM_IOJS_ORG_MIRROR=http://npm.taobao.org/mirrors/iojs
# 生效
source .zshrc
nvm install V14.15.4
## m1 芯片 nvm 会下载源码构建
```