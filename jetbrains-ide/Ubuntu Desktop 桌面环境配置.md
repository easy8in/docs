为了破解方便，切记不可用 snap 的方式安装，因为 snap 安装目录 /snap 是只读文件系统，禁止该目录下的程序修改任何文件。

[破解](https://3.jetbra.in/ )

将官方程序安装到自己设计的目录内 例如: /${HOME}/jetbrains/idea-IU

配置 idea-IU 的桌面文件

```bash
cd /usr/share/applications/

# jetbrains 其他 IDE 将 idea 替换对应名字 例如: clion --> jetbrains-clion.desktop
touch jetbrains-idea.desktop

# 输入一下内容
[Desktop Entry]
Name=IntelliJ IDEA
Comment=IntelliJ IDEA
GenericName=IDEA
Exec=/home/easy/dev-env/jetbrains/idea-IU/bin/idea.sh
Icon=/home/easy/dev-env/jetbrains/idea-IU/bin/idea.svg
Terminal=false
Type=Application
Categories=Application;Development;IDE;
StartupWMClass=jetbrains-idea
StartupNotify=true

# 注意事项
Exec= 替换实际执行文件
Icon= 替换实际图标位置，找不到图片系统采用默认图标 OpenJDK 图标
StartupWMClass=jetbrains-idea 这里不能修改，且必须是这个值，识别不到进程名，图标不生效
```

CLion 配置备注

```bash

sudo vim /usr/share/applications/jetbrains-clion.desktop

# 输入下方内容
[Desktop Entry]
Name=CLion
Comment=Clion
GenericName=Clion
Exec=/home/easy/dev-env/jetbrains/clion-2023.2.2/bin/clion.sh
Icon=/home/easy/dev-env/jetbrains/clion-2023.2.2/bin/clion.svg
Terminal=false
Type=Application
Categories=Application;Development;
Categories=Application;Development;IDE;
StartupWMClass=jetbrains-clion
StartupNotify=false
```

图标不能识别的情况下，例如 dock 启动的 IDE 图标不正确，可以将鼠标移到程序上，看看 tip 显示的是什么，之后在 desktop 文件中修改 StartupWMClass=${tip 显示的内容}