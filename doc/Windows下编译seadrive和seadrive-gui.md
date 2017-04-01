* 下载代码:

```
git clone git@github.com:seafileltd/seadrive.git
git clone git@github.com:seafileltd/seadrive-gui.git
```

* 下载安装 Dokan Library: https://github.com/dokan-dev/dokany/releases/download/v1.0.0/DokanSetup_redist-1.0.0.5000.exe, **安装时一定要勾选 "install developement files"**
* 安装后, 在 mingw 的命令行中, 把 /C/Program\ Files/Dokan Library/1.0.0 下的 include/dokan 目录拷贝到 /usr/local/include 下, 把 x86/lib/dokan1.lib 拷贝到 seadrive/src/ 目录下

* 编译 seadrive:

```
cd seadrive
./autogen.sh
./configure.sh
make
make install
```

* 编译 seadrive-gui

```
cd seadrive-gui
cmake .
make
```
