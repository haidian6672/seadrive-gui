# Setting up build enviroment for seafile windows client

Seafile windows client release package is built on windows XP with msys2/mingw enviroment.

- Prepare a windows XP machine
- Download and install msys2
- Install the required packages

If you're preparing a development environment, you can also use Windows 7 or above. In this case, you can use a 64-bit environment.

## Download and install MSYS2

- Visit https://msys2.github.io/ , download the 32bit vesion: http://sourceforge.net/projects/msys2/files/Base/i686/msys2-i686-20150202.exe/download . Download the 64bit version if you're using Windows 7 or above and a 64bit platform for developement.
- Run the installer, install to `C:\msys` directory
- Create a shortcur on desktop for `C:\msys2\mingw32_shell.bat`. We would always start the shell by double clicking it. Use mingw64_shell if you're using 64bit environment.

## Install required packages

Note that in msys2, there are two set of toolchains, one for 32-bit (i686) and one for 64-bit (x86_64). In the follwing, we'll install packages for both archtectures. If you're only interested in setting up a release package building environment, i686 architecture is the only one required.

### Update the packages index

- Open the shell, and run the `pacman -Sy` command.
- After it finishes, close the terminal and restart it.

#### Update msys-runtime

- Run `pacman -Su msys-runtime`, and restart the terminal when it finishes.

#### Update all pacakges

- Run `pacman -Su`, and restart the terminal when it finishes. This may take a while.

#### Install necessary tools

- Run `pacman -S git tree wget vim p7zip`

#### Install base-devel and mingw32/mingw64/cross toolchain

- Run `pacman -S base-devel mingw-w64-{i686,x86_64,cross}-toolchain`. This may take a while.

The cross toolchain is required to build the 64-bit seafile shell extension DLL on 32bit machines. You can skip it when setting up a development environment.

#### Install seafile client dependent packages

```sh
pacman -S mingw-w64-{i686,x86_64}-{glib2,vala,jansson,libevent,cmake}
pacman -S mingw-w64-{i686,x86_64}-{qtbinpatcher,jasper,libjpeg,libmng,libpng,libtiff,libxml2,libxslt,libwebp,pcre,sqlite3,xpm-nox}
```

#### Build customized libcurl and qt5

We want to build the customized version because:

- For libcurl, we need to build a customized version with ipv6 support enabled. 

- For qt5, the customized version can reduce the final seafile windows installer size by 10MB (mainly because we removed the dependency on `libicu`). You can skip it for a development environment, in which case you can just install it by `pacman -Ss mingw32/mingw-w64-qt5`.

Download the `MINGW-packages.tar.gz` from the share link https://seacloud.cc/d/ebbe7e3813/ (password L1yS3jEH), then:

```
gpg --keyserver hkp://pgp.mit.edu --recv-keys 78E11C6B279D5C91
tar xf MINGW-packages.tar.gz
cd MINGW-packages/mingw-w64-curl && rm -f mingw-w64-*.xz && makepkg-mingw -s && pacman -U mingw-w64-i686-curl-*-any.pkg.tar.xz
cd MINGW-packages/mingw-w64-qt5 && rm -f mingw-w64-*.xz && makepkg-mingw -s && pacman -U mingw-w64-i686-qt5-*.pkg.tar.xz
```
We first add the PGP key for libcurl source code. Then build curl and qt5. There are two things to note:

- The "-s" option to makepkg-mingw tell the tool the fetch dependent packages.
- If you're only interested in building one architecture, you can set environment variable "MINGW_INSTALLS=mingw64" or "MINGW_INSTALLS=mingw32". See msys2's package building instructions for details.


#### Install breakpad and bpwrapper (optional)

- breakpad:
```sh
pacman -S mingw-w64-i686-breakpad-svn
```
- bpwrapper:
```sh
git clone https://github.com/haiwen/bpwrapper.git
cd bpwrapper
./autogen.sh
./configure
make
make install
```

#### Install wix and .net framework

.Net framework installer and other tools like wix can be downloaded from https://seacloud.cc/d/ebbe7e3813/ , password is L1yS3jEH

- Download `dotnetfx35setup.exe` from the above share link, and install it.
- Download `wix310-binaries.zip` and uncompress to `c:\wix`
- Download (and uncompress if necessary) `Paraffin.exe`, `depends.zip`, also to `c:\wix`

### Update .bashrc

Edit `~/.bashrc`, add the following lines to it:

```
export PATH=/c/wix/:/opt/bin/:$PATH
export PKG_CONIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH
```

Save the file, and restart the shell.
