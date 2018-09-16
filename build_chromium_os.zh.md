# 构建 Chromium OS

这篇文档试图记录我构建 Chromium OS，最后作为虚拟机启动的过程。

我使用的笔记本电脑的主要配置：
CPU：4核, 2.7GHz
内存：8G
网络状况：看 Youtube 保持在 20Mbits/S 的速度。

我的电脑没有太多的剩余空间，这是我的第一个大问题，我不得不用了一个外接硬盘。以我的整个构建过程来看，磁盘占用最多的时候接近 120G，所以你用来构建 Chromium OS 的硬盘分区最好有 **120G 的空间**。

8G 内存其实不太够了，最好能有更大的内存，更多的内存能够加速构建过程。

虽然官方的文档里没有对硬盘做出特别的要求，但是我想最好还是用 SSD 硬盘——剩余空间超过 120G 的 SSD 硬盘，因为我不得不用的那块外接硬盘不是 SSD 的，写入速度大约在 130 ~140 M/s，编译主要靠 CPU，但是编译过程中会有大量的临时小文件的读写，所以 SSD 会有优势。

以我的 4核 CPU， 8G 内存和外接 HDD 硬盘的配置，在 `build_packages` 这一步耗费了 6 个小时，是官方文档里说的 90 分钟的 4 倍！非常让人崩溃。

另外有个小建议，最好在 `tmux` 里进行整个过程，防止万一把终端窗口关闭，中断某个命令。如果有些命令跑的时间比较长，而且没有输出，可以通过 `top`, `nethogs` 等命令查看相关的进程是否还在活动着。


## 准备工作

首先要说的是，我使用的是 zsh，所以一些环境的修改是修改 `$HOME/.zshrc`，如果你用的是其他的 shell 比如 bash，那么你应该修改的是 `$HOME/.bashrc`。

### 安装需要的工具或依赖

#### 安装 `git` 和 `curl`

```
sudo apt-get install git-core gitk git-gui curl lvm2 thin-provisioning-tools \
     python-pkg-resources python-virtualenv python-oauth2client
```

#### 安装 `depot_tools`

在 `$HOME/tools` 目录下

```
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```

你当然可以选择其他任何你喜欢的路径，重要的是要把正确的 `depot_tools` 路径添加到 `PATH` 环境变量里，在 `$HOME/.zshrc` 文件添加这一行：

```
export PATH=$PATH:$HOME/tools/depot_tools
```

之后 `source $HOME/.zshrc`，重新加载 `$HOME.zshrc` 文件，对环境变量的修改生效。

### 修改 sudo 行为

把下面的内容添加到 `/etc/sudoers.d/relax_requirements` 文件里

```
Defaults !tty_tickets
Defaults timestamp_timeout=400
```

这样做的效果是：在一个终端里执行 `sudo` 命令了，输了密码后，再新开的终端里执行 `sudo` 命令不需要再输入密码，直到 400 分钟后，再执行 `sudo` 时才需要输入密码。

官方的文档里写的是 180 分钟的超时时间，但是实际上我后面的构建过程远远超过了 180 分钟，所以如果你用的是普通的电脑，我建议还是把这个时间设置长一些。

### 验证 CPU 架构和系统的默认权限

* 执行 `uname -m` 验证系统 CPU 架构是 `X86_64`。
* 执行 `umask` 确认默认权限没问题，结果应该是 `022` 或者 `002`，保证文件和目录对外是可读的。

### 配置 git 用户信息

你很可能已经早就做好了这一步

```
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

### 获取源码

由于我用的是一个外接硬盘，所以我在外接硬盘的分区里创建 `chromiumos` 目录，软链接到 `HOME` 目录。

```
mkdir -p /media/noodles/Tatooine/playground/chromiumos
ln -s /media/noodles/Tatooine/playground/chromiumos ${HOME}/chromiumos
```

在拉去源码之前，需要按照 [http://www.chromium.org/chromium-os/developer-guide/gerrit-guide](http://www.chromium.org/chromium-os/developer-guide/gerrit-guide) 的指示设置Gerrit 鉴权，你会得到一串脚本，把它追加到 `.zshrc`，之后 `source .zshrc`。然后执行

```
git ls-remote https://chromium.googlesource.com/a/chromiumos/manifest.git
```

如果没有鉴权过程，直接输出了 git 相关的内容，说明设置成功。后面可以顺利地获取源码了。

下一步 `cd ${HOME}/chromiumos` 进入 `chromiumos` 目录后执行
```
repo init -u https://chromium.googlesource.com/chromiumos/manifest.git --repo-url https://chromium.googlesource.com/external/repo.git
```

最后会有个小问题

```
Enable color display in this user account (y/N)?
```

我选了 `Y`。

然后执行

```
repo sync -j4
```

获取源码, `-j4` 表示 4 个并发任务去同步，同时同步 4 个仓库。平均速度大概 38Mbits/s，大概 08:05 开始，10:20 结束，比我想象中的更耗时。

### API KEY

在同步源码的两个小时内没有别的什么可以做的。如果你希望构建出来的 Chromium OS 要用 Google 的 API，提供登录、翻译等功能，这期间可以去
[http://www.chromium.org/developers/how-tos/api-keys](http://www.chromium.org/developers/how-tos/api-keys) 页面看一下，按照指示获取 `API KEY`, `client ID` 和 `client Secret`，保存在 `$HOME/.googleapikeys` 文件里。

```
google_api_key = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
google_default_client_id = "yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy"
google_default_client_secret = "zzzzzzzzzzzzzzzzzzzzzzzzzz"
```

### chroot

代码同步完成后。你现在应该仍然在 `${HOME}/chromiumos` 目录，执行

```
cros_sdk
```

这个命令从 Google 拉过来一个 chroot 环境，然后进入这个 chroot 环境，花了我大概十几分钟完成，进入 chroot 环境。除非特殊说明，**后面的命令都是在这个 chroot 环境里进行的**。

## 编译构建

### 准备

首先要指定你要构建给哪个平台使用，执行 `export BOARD=amd64-generic`，后面进行真正的编译配置和编译过程时，会用到 `BOARD` 这个环境变量作为参数。

然后执行 `./setup_board --board=${BOARD}` 为编译工作做好准备，这一步会下载一点内容，耗时在几分钟到二十分钟左右。

Chromium OS 会提供一个名为 `chronos` 的共享帐号，可以在命令行使用这个帐号登录系统。在执行 `./build_packages --board=${BOARD}` 开始编译之前，你可能会想为这个共享帐号设置一个密码，执行 `./set_shared_user_password.sh` 来进行设置。如果你像我一样，构建的是 `test` 模式的镜像，这里设置的密码会被忽略的，`test` 模式的密码是默认的 `test0000`。

### 编译包

然后执行 `./build_packages --board=${BOARD}` 开始编译各个所有包，我感觉这一步占据了整个过程所用时间的 80% 到 90%，非常让人无奈，唯一的安慰是你可以看到进度，在这个过程的输出里，你可以看到类似这样的信息：

```
Pending 481/648, Building 4/286, [Time 15:36:15 | Elapsed 127m16.6s | Load 10.86 11.22 10.68]
```

这个例子里 `Pending 481/648` 中 `481` 是还剩下要编译的包的数量，`Building 4/286` 中 `4` 是正在编译的包的数量。通过 `Elapsed 127m16.6s` 可以看出这一步已经花了多长时间。 

整理一下，从准备到编译包的完整步骤命令：

```
export BOARD=amd64-generic
./setup_board --board=${BOARD}
./set_shared_user_password.sh ## optional
./build_packages --board=${BOARD}
```

### 遇到的问题

我一开始设置的 `BOARD` 是 `x86-generic`，结果在 `build_packages` 这一步遇到了错误，具体是进行到这一步骤时

```
INFO    : Checking package dependencies are correct: virtual/target-os virtual/target-os-dev virtual/target-os-factory virtual/target-os-factory-shim virtual/target-os-test chromeos-base/autotest-all
```

出现以下错误：

```
!!! The ebuild selected to satisfy "chromeos-base/chromeos-chrome" for /build/x86-generic/ has unmet requirements.
- chromeos-base/chromeos-chrome-71.0.3544.0_rc-r1::chromiumos USE="accessibility autotest build_tests buildcheck cfi chrome_debug chrome_remoting cups debug_fission evdev_gestures fonts gold highdpi libcxx nacl opengles runhooks smbprovider thinlto v4l2_codec vaapi xkbcommon -afdo_use -app_shell -asan (-authpolicy) -build_native_assistant -chrome_debug_tests -chrome_internal -chrome_media -clang -clang_tidy -component_build -goma -hardfp -internal_gles_conform -jumbo -lld -mojo (-neon) -oobe_config -opengl -v4lplugin -verbose -vtable_verify" OZONE_PLATFORM="default_gbm gbm -caca -cast -default_caca -default_cast -default_egltest -default_test -egltest -test" OZONE_PLATFORM_DEFAULT="gbm -caca -cast -egltest -test"

  The following REQUIRED_USE flag constraints are unsatisfied:
    libcxx? ( clang ) thinlto? ( clang )

  The above constraints are a subset of the following complete expression:
    asan? ( clang ) at-most-one-of ( gold lld ) cfi? ( thinlto ) clang_tidy? ( clang ) libcxx? ( clang ) thinlto? ( clang any-of ( gold lld ) ) afdo_use? ( clang ) exactly-one-of ( ozone_platform_default_gbm ozone_platform_default_cast ozone_platform_default_test ozone_platform_default_egltest ozone_platform_default_caca )

(dependency required by "chromeos-base/autotest-chrome-0.0.1-r7313::chromiumos" [ebuild])
(dependency required by "chromeos-base/autotest-all-0.0.1-r46::chromiumos[-chromeless_tests,-chromeless_tty]" [ebuild])
(dependency required by "chromeos-base/autotest-all" [argument])
ERROR   : emerge detected broken ebuilds. See error message above.
```

尝试执行 `eclean -d packages` 之后，重新执行 `./build_packages --board=${BOARD}`，仍然报同样的错误。

然后继续尝试，回到上一步，执行 `./setup_board --board=${BOARD} --force` 强制重建 `board root`，之后重新执行 `build_packages` 命令，问题依然。

最后还是以 `amd64-generic` 为参数进行编译。

更多相关信息：https://groups.google.com/a/chromium.org/forum/#!topic/chromium-os-discuss/0wJ0v9XBIu8

### 构建镜像文件

```
./build_image --board=${BOARD}  --noenable_rootfs_verification test
```

过程中有个跟执行 `build_package` 命令时类似的进度展示

```
Pending 56/86, Building 4/57, [Time 19:48:34 | Elapsed 0m34.6s | Load 4.5 5.67 7.74]
```

最终耗时

```
INFO    : Elapsed time (build_image): 22m9s
```

结尾处有这段内容，明确地描述了该怎么用这个镜像文件。

```
INFO    : Test image created as chromiumos_test_image.bin
INFO    : To copy the image to a USB key, use:
INFO    :   cros flash usb:// ../build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1/chromiumos_test_image.bin
INFO    : To convert it to a VM image, use:
INFO    :   ./image_to_vm.sh --from=../build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1 --board=amd64-generic --test
```

### 转换为 KVM 镜像

按照上一部最后的输出给的命令执行

```
./image_to_vm.sh --from=../build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1 --board=amd64-generic --test
```

在转换的过程中，我查看了以下硬盘空间，当前硬盘占用 `116G`。

转换完成后，有这样的一段内容输出：

```
Created image at /mnt/host/source/src/build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1
You can start the image with:
cros_vm --start --image-path /mnt/host/source/src/build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1/chromiumos_qemu_image.bin
```

## 启动虚拟机

### 权限问题

按照提示，执行 `cros_vm --start --image-path /mnt/host/source/src/build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1/chromiumos_qemu_image.bin` 报错:

```
IOError: [Errno 13] Permission denied: '/tmp/cros_vm_9222/kvm.monitor.serial'
```

起初我以为是 chroot 环境有什么问题（因为编译包和构建镜像的时间远远超过了我一开始设置的 sudo 超时时间 180 分钟），所以尝试在 chroot 外部环境执行 `cros_sdk --unmount`，之后再次 `cros_sdk` 进入 chroot 环境，执行 `cros_vm`，还是同样的权限问题。

之后我在 chroot 环境里修改 `/tmp/cros_vm_9222` 目录权限为 777（非常粗暴的做法），但是执行 `cros_vm` 仍然报同样的错误，目录权限又变成了 775。说明 `cros_vm` 执行过程中会修改这个目录的权限。
`/tmp/cros_vm_9222` 目录权限是 `775`，根据 `cros_vm` 报错堆栈信息，在 `/mnt/host/source/chromite` 目录 grep 775 查到 `/mnt/host/source/chromite/lib/osutils.py` 这个文件，修改了下面两个函数的默认参数，`mode=0o775` 改成了 `mode=0o777`。

```
def SafeMakedirsNonRoot(path, mode=0o777, user=None)
def SafeMakedirs(path, mode=0o777, sudo=False, user='root')
```

OK，现在能继续下去了，但是 `777` 的权限实在丑陋，所以我改回了 `/mnt/host/source/chromite/lib/osutils.py`，用 `sudo` 来执行 `cros_vm`。

```
sudo /mnt/host/source/chromite/bin/cros_vm --start --image-path /mnt/host/source/src/build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1/chromiumos_qemu_image.bin
```

### 内存大小问题

上一步的权限问题解决后，还是没能启动起来，又有新的错误

```
21:09:41: INFO: RunCommand: sudo -- /mnt/host/source/chroot/usr/bin/qemu-system-x86_64 -m 8G -smp 8 -vga virtio -daemonize -usbdevice tablet -pidfile /tmp/cros_vm_9222/kvm.pid -chardev 'pipe,id=control_pipe,path=/tmp/cros_vm_9222/kvm.monitor' -serial file:/tmp/cros_vm_9222/kvm.monitor.serial -mon 'chardev=control_pipe' -cpu SandyBridge,-invpcid,-tsc-deadline,check -device 'virtio-net,netdev=eth0' -netdev 'user,id=eth0,net=10.0.2.0/27,hostfwd=tcp:127.0.0.1:9222-:22' -drive 'file=/mnt/host/source/src/build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1/chromiumos_qemu_image.bin,index=0,media=disk,cache=unsafe,format=raw' -enable-kvm
qemu-system-x86_64: cannot set up guest memory 'pc.ram': Cannot allocate memory
```

看这个内容，应该是指定了太大的内存，而系统无法分配。根据报错的堆栈信息，找到相关的目录，grep 8G 这个关键字，修改了 `/mnt/host/source/chromite/scripts/cros_vm.py` 里的这一行，把 `8G` 改成了 `4G`。

```
 parser.add_argument('--qemu-m', type=str, default='4G',
                        help='Memory argument that will be passed to qemu.')
```

执行 OK，虚拟机启动起来了。

```
21:12:56: INFO: RunCommand: sudo -- /mnt/host/source/chroot/usr/bin/qemu-system-x86_64 -m 4G -smp 8 -vga virtio -daemonize -usbdevice tablet -pidfile /tmp/cros_vm_9222/kvm.pid -chardev 'pipe,id=control_pipe,path=/tmp/cros_vm_9222/kvm.monitor' -serial file:/tmp/cros_vm_9222/kvm.monitor.serial -mon 'chardev=control_pipe' -cpu SandyBridge,-invpcid,-tsc-deadline,check -device 'virtio-net,netdev=eth0' -netdev 'user,id=eth0,net=10.0.2.0/27,hostfwd=tcp:127.0.0.1:9222-:22' -drive 'file=/mnt/host/source/src/build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1/chromiumos_qemu_image.bin,index=0,media=disk,cache=unsafe,format=raw' -enable-kvm
VNC server running on '127.0.0.1;5900'
```

### 登录虚拟机

可以从上面的信息里看出，虚拟机的 ssh 22 端口是映射到本机 127.0.0.1:9222 端口的。

```
ssh root@localhost -p 9222
```
使用密码 `test0000` 进入系统。也可以用前面提到的 `chronos` 帐号进入系统。

```
localhost ~ # uname -r
4.4.155-15233-g439d815e28b7
localhost ~ # cat /etc/lsb-release 
CHROMEOS_RELEASE_BOARD=amd64-generic
CHROMEOS_DEVSERVER=http://Noodles-ThinkPad-T470:8080
GOOGLE_RELEASE=11065.0.2018_09_15_1932
CHROMEOS_RELEASE_BUILD_NUMBER=11065
CHROMEOS_RELEASE_BRANCH_NUMBER=0
CHROMEOS_RELEASE_CHROME_MILESTONE=71
CHROMEOS_RELEASE_PATCH_NUMBER=2018_09_15_1932
CHROMEOS_RELEASE_TRACK=testimage-channel
CHROMEOS_RELEASE_DESCRIPTION=11065.0.2018_09_15_1932 (Test Build - noodles) developer-build amd64-generic
CHROMEOS_RELEASE_NAME=Chromium OS
CHROMEOS_RELEASE_BUILD_TYPE=Test Build - noodles
CHROMEOS_RELEASE_VERSION=11065.0.2018_09_15_1932
CHROMEOS_AUSERVER=http://Noodles-ThinkPad-T470:8080/update
```

### 遗留问题

使用 VNC 客户端连接 127.0.0.1:5900，成功连接，但是没有显示内容。
