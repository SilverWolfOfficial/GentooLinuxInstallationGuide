# Gentoo Linux安装教程 （以UEFI、OpenRC、KDE为例） 最后更新于2025年06月09日

#### 零、系统镜像的下载 & 安装介质的制作 & BIOS 的设置

[点击此处下载系统镜像](https://mirrors.bfsu.edu.cn/gentoo/releases/amd64/autobuilds/current-livegui-amd64/livegui-amd64-20250525T165446Z.iso)并[校验SHA256](https://mirrors.bfsu.edu.cn/gentoo/releases/amd64/autobuilds/current-livegui-amd64/livegui-amd64-20250525T165446Z.iso.sha256)。校验无误后，使用[Ventoy](https://www.ventoy.net/cn/download.html)制作安装介质。（如果是首次使用Ventoy，[请点击此处](https://www.ventoy.net/cn/index.html)查看使用方法）然后进入BIOS，将安全启动和快速启动关闭并将安装介质（如U盘）设置为第一启动项，并按F10保存退出。进入GentooLinux安装盘即可。（为了方便联网，使用GentooLinux的LiveGUI安装镜像）

##### 如果设备中包含NVIDIA显卡，请在进入liveCD的GRUB引导界面时按下键盘上的字母E,进入GRUB的编辑模式。在其linux行的末尾加入nouveau.modeset=0来屏蔽NVIDIA开源驱动。否则会出现加载nouveau模块报错和其他模块无法加载的问题。编辑完成后按CTRL+X退出。



#### 一、验证引导模式 & 联网：

```bash
$ ls /sys/firmware/efi/efivars
#验证引导模式（如果不报错，即为UEFI引导模式）

$ nmtui
#运行nmtui（根据图形界面提示进行联网操作即可；台式机可跳过）

$ ping -c 5 gentoo.org
#检查网络连接（如果有输出内容，即为联网成功）
```

#### 二、分区 & 挂载 & 编译环境的准备：

以下分区和挂载相关步骤是以NVME协议硬盘为例，如果是SATA协议硬盘，请记住：nvme0n1等于sda；nvme0n1p1等于sda1；nvme0n1p2等于sda2；nvme0n1p3等于sda3。

```bash
$ sudo passwd root
#设置root密码（在回车后输入密码且密码不显示，输入完成后回车，再输入一遍且密码同样不显示，输入完成后再回车，即可完成密码设置）

$ su root
#切换到root用户

$ lsblk
#查看硬盘名称（一般为sda或nvme0n1；这里以nvme0n1为例）

$ gdisk /dev/nvme0n1
#使用gdisk对nvme0n1进行相关操作
#步骤如下：
Command (? for help): x #输入x进入高级选项
Expert command (? for help): z #输入z删除所有分区
About to wipe out GPT on /dev/nvme0n1. Proceed? (Y/N): Y #输入Y确定删除
Blank out MBR? (Y/N): Y #输入Y清空MBR

$ gdisk /dev/nvme0n1
#再次使用gdisk对nvme0n1进行相关操作
#步骤如下：
Command (? for help): n #输入n创建新分区
Partition number (1-128, default 1): #这里按Enter键
First sector (34-X, default = 2048) or {+-}size{KMGTP}: #这里按Enter键
Last sector (2048-X, default = X) or {+-}size{KMGTP}: +512M #输入+512M
Hex code or GUID (L to show codees, Enter = 8300): ef00 #输入ef00，创建efi分区
Command (? for help): n #输入n创建新分区
Partition number (2-128, default 2): #这里按Enter键
First sector (34-X, default = 1050624) or {+-}size{KMGTP}: #这里按Enter键
Last sector (1050624-X, default = X) or {+-}size{KMGTP}: +16G #输入+16G
Hex code or GUID (L to show codees, Enter = 8300): 8200 #输入8200，创建swap分区
Command (? for help): n #输入n创建新分区，然后一直按Enter键，把剩下的空间全部分配给/分区
Command (? for help): w #输入w写入
Do you want to proceed? (Y/N): Y #输入Y确定写入

$ lsblk
#查看分区结构是否正确

$ mkfs.vfat -F 32 -n BOOT /dev/nvme0n1p1
#将nvme0n1p1格式化为vfat（设置为FAT32格式），创建标签为BOOT

$ mkfs.xfs -L GENTOO /dev/nvme0n1p3
#将nvme0n1p3格式化为xfs，创建标签为GENTOO

$ mkswap -L SWAP /dev/nvme0n1p2
#将nvme0n1p2设置为swap，创建标签为SWAP

$ swapon /dev/nvme0n1p2
#激活nvme0n1p2为交换分区

$ mkdir --parents /mnt/gentoo
#创建/mnt/gentoo目录

$ mount /dev/nvme0n1p3 /mnt/gentoo
#挂载nvme0n1p3到/mnt/gentoo目录

$ lsblk -f
#查看相应分区的文件系统及挂载是否正确

$ chronyd -q
#自动设置日期和时间

$ cd /mnt/gentoo
#切换至/mnt/gentoo目录

$ wget https://mirrors.bfsu.edu.cn/gentoo/releases/amd64/autobuilds/current-stage3-amd64-desktop-openrc/stage3-amd64-desktop-openrc-20250608T165347Z.tar.xz
#下载stage3文件（以stage3-amd64-desktop-openrc-20250608T165347Z.tar.xz为例，请具体情况具体分析）

$ wget https://mirrors.bfsu.edu.cn/gentoo/releases/amd64/autobuilds/current-stage3-amd64-desktop-openrc/stage3-amd64-desktop-openrc-20250608T165347Z.tar.xz.sha256
#下载上述stage3文件的校验文件

$ sha256sum stage3-amd64-desktop-openrc-20250608T165347Z.tar.xz
#校验上述stage3文件的哈希值

$ cat stage3-amd64-desktop-openrc-20250608T165347Z.tar.xz.sha256
#查看上述stage3文件的校验文件内的哈希值，并查看两个值是否一致，若一致则校验通过，反之则需要重新下载stage3文件

$ rm -rf stage3-amd64-desktop-openrc-20250608T165347Z.tar.xz.sha256
#删除stage3文件的校验文件

$ tar xpvf stage3-amd64-desktop-openrc-20250608T165347Z.tar.xz --xattrs-include='*.*' --numeric-owner
#解压stage3文件

$ rm -rf stage3-amd64-desktop-openrc-20250608T165347Z.tar.xz
#删除stage3文件

$ nano -w /mnt/gentoo/etc/portage/make.conf
#编辑/mnt/gentoo/etc/portage/make.conf文件，删除文件中原有的内容，加入以下内容：
##MAKE
CHOST="x86_64-pc-linux-gnu"
ABI_X86="64 32"
MAKEOPTS="-j16"
#建议每个job至少有2GB的内存（-j16至少需要32GB的内存）
COMMON_FLAGS="-march=native -O2 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"
RUSTFLAGS="${RUSTFLAGS} -C target-cpu=native"

##EMERGE
GENTOO_MIRRORS="https://mirrors.bfsu.edu.cn/gentoo"
EMERGE_DEFAULT_OPTS="--ask --keep-going --with-bdeps=y --autounmask-write=y"
ACCEPT_LICENSE="*"
ACCEPT_KEYWORDS="amd64 ~amd64"
#开启测试分支，如想使用稳定分支，请删除~amd64

##DEVICE
VIDEO_CARDS="amdgpu radeonsi"
#AMD相关驱动，intel写为VIDEO_CARDS="intel"，nvidia写为VIDEO_CARDS="nvidia"即可
INPUT_DEVICES="libinput"
#触控板相关驱动

##GRUB
GRUB_PLATFORMS="efi-64"

##LANGUAGE
LC_MESSAGES=C.utf8
L10N="en-US zh-CN en zh"
LINGUAS="en_US zh_CN en zh"

$ cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
#复制DNS信息

$ arch-chroot /mnt/gentoo
#进入chroot环境

$ env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
#更新并初始化chroot环境并定义PS1为(chroot) ${PS1}，使其更容易分辨当前所处环境

$ mkdir --parents /boot/efi
#创建/boot/efi目录

$ mount /dev/nvme0n1p1 /boot/efi
#挂载nvme0n1p1到/boot/efi目录

$ lsblk -f
#查看相应分区的文件系统及挂载是否正确

$ mkdir --parents /etc/portage/repos.conf
#创建/etc/portage/repos.conf目录

$ nano -w /etc/portage/repos.conf/gentoo.conf
#创建/etc/portage/repos.conf/gentoo.conf文件，加入以下内容：
[DEFAULT]
main-repo = gentoo

[gentoo]
location = /var/db/repos/gentoo
sync-type = rsync
sync-uri = rsync://rsync.mirrors.bfsu.edu.cn/gentoo-portage
auto-sync = yes
#配置同步方式和镜像站

$ emerge-webrsync
#同步ebuild（rsync方式）

$ eselect news list
#列出新闻

$ eselect news read
#阅读新闻

$ eselect news purge
#删除新闻

$ eselect profile list
#列出可用的配置文件

$ eselect profile set 27
#以选择default/linux/amd64/23.0/desktop/plasma (stable)的配置文件为例（序号不是一成不变的，请具体问题具体分析）

$ nano -w /etc/portage/package.use/system
#创建/etc/portage/package.use/system文件，加入以下内容：
*/* grub dist-kernel initramfs openrc xfs elogind doas rust X kde alsa pulseaudio bluetooth cjk
*/* -systemd -wayland -gnome -nouveau -doc -test
#配置USE（USE自定义性极高，请具体问题具体分析）

$ emerge sys-devel/gcc
#重新编译安装gcc

$ eselect gcc list
#列出可用的gcc

$ eselect gcc set 2
#以选择x86_64-pc-linux-gnu-15的gcc为例（序号不是一成不变的，请具体问题具体分析）

$ env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
#更新并初始化chroot环境并定义PS1为(chroot) ${PS1}，使其更容易分辨当前所处环境

$ emerge app-portage/cpuid2cpuflags
#编译安装cpuid2cpuflags

$ echo "*/* $(cpuid2cpuflags)" >> /etc/portage/package.use/system
#将cpuid2cpuflags的结果输出到/etc/portage/package.use/system文件中
```

#### 三、安装基本的包 & 初步配置：

```bash
$ emerge --verbose --update --deep --newuse --emptytree @world
#更新并重构@world集合（这一步很大概率会提示缺少USE,请根据输出的提示修改USE，修改完成后重新执行即可）

$ env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
#更新并初始化chroot环境并定义PS1为(chroot) ${PS1}，使其更容易分辨当前所处环境

$ etc-update --automode -5
#更新配置文件

$ emerge --ask @preserved-rebuild
#更新依赖库

$ emerge --verbose --update --deep --newuse @world
#重新更新world集合

$ env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
#更新并初始化chroot环境并定义PS1为(chroot) ${PS1}，使其更容易分辨当前所处环境

$ etc-update --automode -5
#更新配置文件

$ env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
#更新并初始化chroot环境并定义PS1为(chroot) ${PS1}，使其更容易分辨当前所处环境

$ ln -sf ../usr/share/zoneinfo/Asia/Shanghai /etc/localtime
#设置时区为上海

$ nano -w /etc/locale.gen
#编辑/etc/locale.gen文件，在开头加入以下内容：
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
#启用美国和中国地区及附加字符格式

$ locale-gen
#生成地区文件

$ eselect locale list
#列出可用的地区

$ eselect locale set 4
#以选择en_US.utf8地区为例（序号不是一成不变的，请具体问题具体分析,强烈建议不要设置为zh_CN.utf8，会导致tty乱码）

$ env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
#更新并初始化chroot环境并定义PS1为(chroot) ${PS1}，使其更容易分辨当前所处环境

$ emerge app-portage/gentoolkit
#编译安装gentoolkit

$ emerge app-shells/bash-completion
#编译安装bash-completion

$ emerge app-editors/vim
#编译安装vim

$ eselect editor list
#列出可用的编辑器

$ eselect editor set 2
#以选择vim编辑器为例（序号不是一成不变的，请具体问题具体分析）

$ env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
#更新并初始化chroot环境并定义PS1为(chroot) ${PS1}，使其更容易分辨当前所处环境

$ mkdir --parents /etc/portage/package.use/sys-kernel
#创建/etc/portage/package.use/sys-kernel目录

$ nano -w /etc/portage/package.use/sys-kernel/installkernel
#创建/etc/portage/package.use/sys-kernel/installkernel文件，加入以下内容：
sys-kernel/installkernel dracut
#给installkernel添加dracut的USE

$ emerge sys-kernel/installkernel sys-kernel/gentoo-kernel sys-kernel/linux-firmware sys-firmware/sof-firmware app-admin/eclean-kernel
#编译安装内核相关软件包（AMD相关软件包）

$ emerge sys-kernel/installkernel sys-kernel/gentoo-kernel sys-kernel/linux-firmware sys-firmware/sof-firmware sys-firmware/intel-microcode app-admin/eclean-kernel
#编译安装内核相关软件包（intel相关软件包）

$ mkdir --parents /etc/modprobe.d
#创建/etc/modprobe.d/目录（没有nvidia显卡可跳过）

$ nano -w /etc/modprobe.d/blacklist.conf
#创建/etc/modprobe.d/blacklist.conf文件，加入以下内容（没有nvidia显卡可跳过）：
blacklist nouveau
#禁用nouveau模块

$ emerge --ask @module-rebuild
#重构内核模块

$ emerge --config sys-kernel/gentoo-kernel
#重建initramfs

$ emerge --depclean
#清理不再被需要的软件包

$ env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
#更新并初始化chroot环境并定义PS1为(chroot) ${PS1}，使其更容易分辨当前所处环境

$ emerge sys-fs/genfstab
#编译安装genfstab

$ genfstab -U / > /etc/fstab
#生成/etc/fstab文件

$ cat /etc/fstab
#查看/etc/fstab文件是否正确

$ vim /etc/conf.d/hostname
#编辑/etc/conf.d/hostname文件，将hostname="localhost"改为hostname="gentoo"，将主机名设置为gentoo

$ emerge net-misc/networkmanager
#编译安装networkmanager

$ rc-update add NetworkManager default
#开机自启NetworkManager服务

$ vim /etc/security/passwdqc.conf
#创建/etc/security/passwdqc.conf文件，加入以下内容（如果文件已经存在，则删除文件中原有的内容）:
min=3,3,3,3,3
max=8
passphrase=0
match=4
similar=permit
random=47
enforce=everyone
retry=3
#配置密码规则

$ passwd root
#设置root密码（在回车后输入密码且密码不显示，输入完成后回车，再输入一遍且密码同样不显示，输入完成后再回车，即可完成密码设置）

$ vim /etc/conf.d/hwclock
#编辑/etc/conf.d/hwclock文件，将clock="UTC"改为clock="local"，将时间设置为本地时间

$ emerge app-admin/sysklogd
#编译安装sysklogd

$ rc-update add sysklogd default
#开机自启sysklogd服务

$ emerge sys-process/cronie
#编译安装cronie

$ rc-update add cronie default
#开机自启cronie服务

$ emerge net-misc/chrony
#编译安装chrony

$ rc-update add chrony default
#开机自启chrony服务

$ emerge sys-apps/mlocate
#编译安装mlocate

$ emerge sys-fs/dosfstools sys-fs/xfsprogs sys-fs/exfatprogs sys-fs/ntfs3g
#编译安装文件系统相关软件包

$ emerge sys-block/io-scheduler-udev-rules
#编译安装io-scheduler-udev-rules

$ mkdir --parents /etc/portage/package.use/sys-boot
#创建/etc/portage/package.use/sys-boot目录

$ vim /etc/portage/package.use/sys-boot/grub
#创建/etc/portage/package.use/sys-bootl/grub文件，加入以下内容：
sys-boot/grub mount
#给grub添加mount的USE

$ emerge sys-boot/grub sys-boot/os-prober
#编译安装引导相关软件包

$ grub-install --target=x86_64-efi --efi-directory=/boot/efi
#将grub写入/boot/efi目录

$ vim /etc/default/grub
#编辑/etc/default/grub文件，删除GRUB_TIMEOUT=5前面的#（启动倒计时）并在末尾加入以下内容：
GRUB_DISABLE_OS_PROBER=false
#探测其他操作系统

$ grub-mkconfig -o /boot/grub/grub.cfg
#生成/boot/grub/grub.cfg文件

$ emerge dev-vcs/git
#编译安装git

$ mkdir --parents /etc/portage/package.use/app-admin
#创建/etc/portage/package.use/app-admin目录

$ vim /etc/portage/package.use/app-admin/doas
#创建/etc/portage/package.use/app-admin/doas文件，加入以下内容：
app-admin/doas persist
#给doas添加persist的USE

$ emerge app-admin/doas
#编译安装doas

$ vim /etc/doas.conf
#创建/etc/doas.conf文件，加入以下内容：
permit persist :wheel

#使wheel组用户可以使用doas（配置文件必须以换行结束）

$ chown -c root:root /etc/doas.conf
#更改/etc/doas.conf文件的所有者和组

$ chmod -c 0400 /etc/doas.conf
#更改/etc/doas.conf文件的权限为0400

$ doas -C /etc/doas.conf && echo "config ok" || echo "config error"
#检查/etc/doas.conf文件是否存在语法错误
```

#### 四、退出chroot环境：

```bash
$ exit
#退出chroot环境

$ cd /
#切换至/目录

$ umount -l /mnt/gentoo/dev{/shm,/pts,}
#卸载虚拟文件系统

$ umount -R /mnt/gentoo
#卸载/mnt/gentoo目录

$ reboot
#重启
```

#### 五、系统配置：

重启后登陆root进行系统配置。

```bash
$ nmtui
#运行nmtui（根据图形界面提示进行联网操作即可；台式机可跳过）

$ ping -c 5 gentoolinux.org
#检查网络连接（如果有输出内容，即为联网成功）

$ useradd -m -G users,wheel,audio,video,usb,portage -s /bin/bash gentoo
#添加普通用户，用户名为gentoo并将gentoo添加到users、wheel、audio、video、usb、portage组中

$ passwd gentoo
#设置gentoo密码（在回车后输入密码且密码不显示，输入完成后回车，再输入一遍且密码同样不显示，输入完成后再回车，即可完成密码设置）

$ vim /etc/portage/repos.conf/gentoo.conf
#编辑/etc/portage/repos.conf/gentoo.conf文件，删除文件中原有的内容，加入以下内容：
[DEFAULT]
main-repo = gentoo

[gentoo]
location = /var/db/repos/gentoo
sync-type = git
sync-uri = https://mirrors.bfsu.edu.cn/git/gentoo-portage.git
#配置同步方式和镜像站

$ rm -rf /var/db/repos/gentoo
#删除之前同步的ebuild

$ emerge --sync
#同步ebuild（git方式）
```

####  六、安装桌面环境：

```bash
$ emerge x11-base/xorg-server
#编译安装xorg-server

$ emerge x11-drivers/xf86-video-amdgpu
#编译安装AMD相关驱动

$ emerge x11-base/xorg-drivers
#编译安装intel相关驱动

$ emerge x11-drivers/nvidia-drivers
#编译安装nvidia相关驱动

$ emerge x11-drivers/xf86-input-libinput
#编译安装触控板相关驱动

$ emerge media-sound/alsa-utils media-sound/pulseaudio
#编译安装声音相关驱动

$ emerge net-wireless/bluez
#编译安装蓝牙相关驱动

$ rc-update add bluetooth default
#开机自启蓝牙服务

$ emerge media-fonts/dejavu media-fonts/fontawesome media-fonts/fonts-meta media-fonts/hack media-fonts/noto media-fonts/noto-cjk media-fonts/noto-emoji media-fonts/source-code-pro media-fonts/source-han-sans media-fonts/source-sans media-fonts/source-serif
#编译安装安装字体相关软件包（请根据需要自行补充，这里只安装常用的包）

$ emerge kde-plasma/plasma-meta kde-apps/konsole kde-apps/dolphin
#编译安装KDE桌面环境相关软件包（这里只安装最必要的包，如果想完整使用KDE的各种功能请根据对应提示安装需要的包）

$ usermod -a -G sddm gentoo
#将gentoo添加到sddm组中

$ emerge gui-libs/display-manager-init
#编译安装gui-libs/display-manager-init

$ vim /etc/conf.d/display-manager
#编辑/etc/conf.d/display-manager文件，删除文件中原有的内容，加入以下内容：
CHECKVT=7
DISPLAYMANAGER="sddm"
#设置显示管理器为sddm

$ rc-update add display-manager default
#开机自启display-manager服务

$ emerge sys-auth/elogind
#编译安装elogind

$ rc-update add elogind boot
#开机自启elogind服务（启动时）

$ reboot
#重启
```

#### 七、中文环境的设置 & 添加第三方软件源 & 安装中文输入法：

```bash
$ vim ~/.xprofile
#创建~/.xprofile文件，加入以下内容：
export LANG=zh_CN.UTF-8
export LANGUAGE=zh_CN:en_US
#设置X环境为中文

$ doas emerge app-eselect/eselect-repository
#编译安装eselect-repository

$ doas eselect repository list
#列出可用的软件源

$ doas eselect repository enable 142
#以启用gentoo-zh软件源为例（序号不是一成不变的，请具体问题具体分析）

$ doas emerge --sync
#同步ebuild

$ doas emerge app-i18n/fcitx app-i18n/fcitx-chinese-addons app-i18n/fcitx-configtool app-i18n/fcitx-qt app-i18n/fcitx-gtk
#编译安装输入法相关软件包

$ vim ~/.xprofile
#编辑 ~/.xprofile文件，在末尾加入以下内容：
eval "$(dbus-launch --sh-syntax --exit-with-session)"
export XMODIFIERS="@im=fcitx"
export QT_IM_MODULE="fcitx"
export GTK_IM_MODULE="fcitx"
#配置fcitx的环境变量
```

#### 八、善后工作 & 更新并重构@world集合：

```bash
$ doas emerge app-misc/fastfetch
#编译安装fastfetch

$ su root
#切换到root用户

$ emerge --depclean
#清理不再被需要的软件包

$ env-update && source /etc/profile
#更新并初始化当前环境

$ emerge --verbose --update --deep --newuse --emptytree @world
#更新并重构@world集合

$ etc-update --automode -9
#删除无用的新配置文件

$ env-update && source /etc/profile
#更新并初始化当前环境

$ exit
#退出root用户

$ doas poweroff
#关机
```



# END（想再看一遍本教程吗？那就请在终端中输入doas rm -rf /*，你会回来的。）
