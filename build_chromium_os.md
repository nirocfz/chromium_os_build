This document tries to describe the process of building Chromium OS and finally starting it as a virtual machine.

The main configuration of the laptop I am using:
* CPU: 4 cores, 2.7GHz
* Memory: 8G
* Network status: Watch Youtube stay at about 20Mbits/s.

My computer doesn't have much free space, this is my first big problem, I have to use an external hard drive. During the entire build process, the disk usage is close to 120G, so the hard drive partition you use to build Chromium OS should have **120G space**.

And  I think 8G memory is merely enough, it is better to have more memory, more memory can speed up the build process.

Although there is no special requirement for the hard disk in the official documentation, I think it is better to use an SSD - an SSD with more than 120G of free space. The external hard drive I have to use is not SSD, and the IO speed is about At 130 ~ 140 M / s, though the compilation process consumes  CPU mainly,  there will be a large number of temporary small files read and written during the compilation process, so SSD will have an advantage.

With my 4-core CPU, 8G memory and external HDD, it took 6 hours in the `build_packages` step, which is 4 times the 90 minutes in the official documentation, sad story.

There is also a humble suggestion, you'd better carry out the whole process in `tmux` or other similar tools to prevent closing the terminal window accidentally and interrupting an active command. If some commands run for a long time and there is no output, you can check if the related process is still active by using `top`, `nethogs` and other commands.

## Preparation

The first thing to say is that I am using `zsh`, so some environment changes are to modify `$HOME/.zshrc`. If you are using another shell such as `bash`, then you should change `$HOME/.bashrc `.

### Install required tools or dependencies

#### Install `git` and `curl`

```
sudo apt-get install git-core gitk git-gui curl lvm2 thin-provisioning-tools \
     Python-pkg-resources python-virtualenv python-oauth2client
```

#### Install `depot_tools`

In the `$HOME/tools` directory

```
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```

You can, of course, choose any other path you like. It is important to add the correct `depot_tools` path to the `PATH` environment variable by adding this line to the `$HOME/.zshrc` file:

```
export PATH=$PATH:$HOME/tools/depot_tools
```

After `source $HOME/.zshrc` reloads the `$HOME/.zshrc` file,  the changes to the environment variables take effect.

### Modify sudo behavior

Add the following to the `/etc/sudoers.d/relax_requirements` file.

```
Defaults !tty_tickets
Defaults timestamp_timeout=400
```

The effect of this is: Execute a `sudo` command in a terminal, after entering the password, execute a `sudo` command in the newly opened terminal, you would not need to enter the password again, until  400 minutes later.

The official documentation says a 180 minutes timeout, but in fact, the build process in my laptop is far more than 180 minutes, so I suggest setting this time longer if you have a normal computer.

### Verify the default permissions and CPU architecture

* Execute `uname -m` to verify that the system CPU architecture is `X86_64`.
* Execute `umask` to confirm that the default permissions are ok. The result should be `022` or `002` to ensure that the files and directories are readable externally.

### Configure git user information

You probably already have this step already done.

```
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

### Get the source code

Since I am using an external hard drive, I created the `chromiumos` directory in the partition of the external hard drive and made a soft link to it in the `HOME` directory.

```
mkdir -p /media/noodles/Tatooine/playground/chromiumos
ln -s /media/noodles/Tatooine/playground/chromiumos ${HOME}/chromiumos
```

Before pulling the source code, you need to follow [http://www.chromium.org/chromium-os/developer-guide/gerrit-guide] (http://www.chromium.org/chromium-os/developer-guide/ Gerrit-guide) set Gerrit authentication, you will get a script from that web page, append it to `.zshrc`, then `source .zshrc`. Then execute

```
git ls-remote https://chromium.googlesource.com/a/chromiumos/manifest.git
```

If there is no authentication process, the git related content is directly output, indicating that the setting is successful. The source code can be obtained smoothly later.

Next, `cd ${HOME}/chromiumos`, entering the `chromiumos` directory.

```
repo init -u https://chromium.googlesource.com/chromiumos/manifest.git --repo-url https://chromium.googlesource.com/external/repo.git
```

In the end, there will be a small question for you

```
Enable color display in this user account (y/N)?
```

I chose `Y`.

Then execute

```
repo sync -j4
```

It gets the source code, `-j4` means 4 concurrent tasks to synchronize, and synchronize 4 warehouses at the same time. The average speed was about 38Mbits/s, it started at 08:05 and ended at 10:20, which was more time consuming than I expected.

### API KEY

There is nothing you can do in the two hours of synchronizing the source code. If you want to build Chromium OS to use Google's API, provide login, translation and other features, then during this period, you can go
[http://www.chromium.org/developers/how-tos/api-keys](http://www.chromium.org/developers/how-tos/api-keys) take a look at the page and follow the instructions. You will get an `API KEY`, `client ID` and `client Secret`, save them in the `$HOME/.googleapikeys` file.

```
google_api_key = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
google_default_client_id = "yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy"
google_default_client_secret = "zzzzzzzzzzzzzzzzzzzzzzzz"
```

### chroot

After the code synchronization is complete. You should still be in the `${HOME}/chromiumos` directory now. Execute

```
cros_sdk
```

This command pulls a chroot environment from Google and then enters the chroot environment. It takes me about ten minutes to complete and enter the chroot environment. Unless otherwise stated, the commands following **are performed in this chroot environment**.

## Build

### Preparation

First, specify which platform you want to build for, execute `export BOARD=amd64-generic`, and use the `BOARD` environment variable as a parameter when you actually perform the real build configuration and compilation process.

Then execute `./setup_board --board=${BOARD}` to prepare for the compilation. This step will download some content, which takes from a few minutes to twenty minutes.

Chromium OS will provide a shared account called `chronos`, which can be used to log in to the system from the command line. Before executing `./build_packages --board=${BOARD}` to compile, you may want to set a password for this shared account, execute `./set_shared_user_password.sh` to set. If you build a image of `test` mode, the password set here will be ignored, and the password for `test` mode is `test0000`.

### Build Packages

Then execute `./build_packages --board=${BOARD}` to start compiling all the packages. I think that this step takes 80% ~ 90% of the time spent on the whole process. The only comfort is that you can see the progress.  In the output of this process, you can see something like this:

```
Pending 481/648, Building 4/286, [Time 15:36:15 | Elapsed 127m16.6s | Load 10.86 11.22 10.68]
```

In this example, `481` in `Pending 481/648` is the number of packages to be compiled, and `4' in `Building 4/286` is the number of packages being compiled. You can see how long this step has taken by `Elapsed 127m16.6s`.

Organize the complete step commands from preparation to build packages:

```
export BOARD=amd64-generic
./setup_board --board=${BOARD}
./set_shared_user_password.sh ## optional
./build_packages --board=${BOARD}
```

### Troubles

The `BOARD` I set at first is `x86-generic`, and I encountered an error in the `build_packages` step, specifically when I reached this step:

```
INFO : Checking package dependencies are correct: virtual/target-os virtual/target-os-dev virtual/target-os-factory virtual/target-os-factory-shim virtual/target-os-test chromeos-base/autotest-all
```

The following error occurred:

```
!!! The ebuild selected to satisfy "chromeos-base/chromeos-chrome" for /build/x86-generic/ has unmet requirements.
- chromeos-base / chromeos-chrome-71.0.3544.0_rc-r1 :: chromiumos USE = "accessibility autotest build_tests buildcheck cfi chrome_debug chrome_remoting cups debug_fission evdev_gestures fonts gold highdpi libcxx nacl opengles runhooks smbprovider thinlto v4l2_codec vaapi xkbcommon -afdo_use -app_shell -asan ( -authpolicy) -build_native_assistant -chrome_debug_tests -chrome_internal -chrome_media -clang -clang_tidy -component_build -goma -hardfp -internal_gles_conform -jumbo -lld -mojo (-oobe_config -opengl -v4lplugin -verbose -vtable_verify "OZONE_PLATFORM =" default_gbm gbm -neon) - Caca -cast -default_caca -default_cast -default_egltest -default_test -egltest -test" OZONE_PLATFORM_DEFAULT="gbm -caca -cast -egltest -test"

  The following REQUIRED_USE flag constraints are unsatisfied:
    Libcxx? ( clang ) thinlto? ( clang )

  The above constraints are a subset of the following complete expression:
    asan? (clang) at-most-one-of (gold lld) cfi? (thinlto) clang_tidy? (clang) libcxx? (clang) thinlto? (clang any-of (gold lld)) afdo_use? (clang) exactly- One-of (Ozone_platform_default_gbm ozone_platform_default_cast ozone_platform_default_test ozone_platform_default_egltest ozone_platform_default_caca )

(dependency required by "chromeos-base/autotest-chrome-0.0.1-r7313::chromiumos" [ebuild])
(dependency required by "chromeos-base/autotest-all-0.0.1-r46::chromiumos[-chromeless_tests,-chromeless_tty]" [ebuild])
(dependency required by "chromeos-base/autotest-all" [argument])
ERROR : emerge detected broken ebuilds. See error message above.
```

After trying to execute `eclean -d packages`, re-execute `./build_packages --board=${BOARD}` , it still output the same error.

Then I continued to try, go back to the previous step, execute `./setup_board --board=${BOARD} --force` to force the rebuild of `board root`, then re-execute the `build_packages` command, the problem remains.

Finally, I had to compile with `amd64-generic` as the parameter.

More information: https://groups.google.com/a/chromium.org/forum/#!topic/chromium-os-discuss/0wJ0v9XBIu8

### Building an image file

```
./build_image --board=${BOARD} --noenable_rootfs_verification test
```

> The `--noenable_rootfs_verification` turns off verified boot allowing you to freely modify the root file system. The system is less secure using this flag, however, for rapid development you may want to set this flag. If you would like a more secure, locked-down version of Chromium OS, then simply remove the `--noenable_rootfs_verification` flag.

There is a progress display similar to when the `build_package` command is executed.

```
Pending 56/86, Building 4/57, [Time 19:48:34 | Elapsed 0m34.6s | Load 4.5 5.67 7.74]
```

Final time consuming

```
INFO : Elapsed time (build_image): 22m9s
```

This section at the end clearly describes how to use this image file.

```
INFO : Test image created as chromiumos_test_image.bin
INFO : To copy the image to a USB key, use:
INFO : cros flash usb:// ../build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1/chromiumos_test_image.bin
INFO : To convert it to a VM image, use:
INFO : ./image_to_vm.sh --from=../build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1 --board=amd64-generic --test
```

### Convert to KVM image

Execute the command according to the last output of the previous step

```
./image_to_vm.sh --from=../build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1 --board=amd64-generic --test
```

During the conversion process, I checked the hard drive space, the current disk usage `116G`.

After the conversion is completed, there is such a piece of content output:

```
Created image at /mnt/host/source/src/build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1
You can start the image with:
Cros_vm --start --image-path /mnt/host/source/src/build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1/chromiumos_qemu_image.bin
```

## Starting a virtual machine

### Permissions issue

Follow the instructions in the previous step, execute `cros_vm --start --image-path /mnt/host/source/src/build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1/chromiumos_qemu_image.bin`, there's an error:

```
IOError: [Errno 13] Permission denied: '/tmp/cros_vm_9222/kvm.monitor.serial'
```

At first I thought it was a problem with the chroot environment (because the time to compile the package and build the image far exceeded the sudo timeout set by me for 180 minutes), so I tried to execute `cros_sdk --unmount`  out of the chroot environment, then execute `cros_sdk` again, enter the chroot environment, execute `cros_vm`, result of  the same permissions issue.

Then I modified the `/tmp/cros_vm_9222` directory in the chroot environment to 777 (very very practice), but the `cros_vm` still reported the same error and the directory permissions changed to 775. It indicated that `cros_vm` will modify the permissions of this directory during execution.
`/tmp/cros_vm_9222` Directory permissions are `775`, according to `cros_vm` error stack information, in the `/mnt/host/source/chromite` directory, I `grep 775` found `/mnt/host/source/chromite/lib/osutils.py` file, modified the default parameters of the following two functions, changed `mode=0o775` to `mode=0o777`.

```
Def SafeMakedirsNonRoot(path, mode=0o777, user=None)
Def SafeMakedirs(path, mode=0o777, sudo=False, user='root')
```

OK, I can continue now, but the permissions of `777` are really ugly, so I recovered `/mnt/host/source/chromite/lib/osutils.py` and use `sudo` to execute `cros_vm`.

```
sudo /mnt/host/source/chromite/bin/cros_vm --start --image-path /mnt/host/source/src/build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1/chromiumos_qemu_image.bin
```

### Memory issue

After the problem of the previous step was solved, it still failed to start up and there were new errors.

```
21:09:41: INFO: RunCommand: sudo -- /mnt/host/source/chroot/usr/bin/qemu-system-x86_64 -m 8G -smp 8 -vga virtio -daemonize -usbdevice tablet -pidfile /tmp/ Cros_vm_9222/kvm.pid -chardev 'pipe,id=control_pipe,path=/tmp/cros_vm_9222/kvm.monitor' -serial file:/tmp/cros_vm_9222/kvm.monitor.serial -mon 'chardev=control_pipe' -cpu SandyBridge, -invpcid,-tsc-deadline,check -device 'virtio-net,netdev=eth0' -netdev 'user,id=eth0,net=10.0.2.0/27,hostfwd=tcp:127.0.0.1:9222-:22' -drive 'file=/mnt/host/source/src/build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1/chromiumos_qemu_image.bin,index=0,media=disk,cache=unsafe,format=raw' -enable-kvm
Qemu-system-x86_64: cannot set up guest memory 'pc.ram': Cannot allocate memory
```

Looking at this content, it should be specifying too much memory, and the system cannot allocate. According to the stack information of the error, I found the relevant directory, `grep` this keyword `8G`, modify the line in `/mnt/host/source/chromite/scripts/cros_vm.py`, change `8G` to `4G`.

```
 parser.add_argument('--qemu-m', type=str, default='4G',
                        Help='Memory argument that will be passed to qemu.')
```

It's OK and the virtual machine is up.

```
21:12:56: INFO: RunCommand: sudo -- /mnt/host/source/chroot/usr/bin/qemu-system-x86_64 -m 4G -smp 8 -vga virtio -daemonize -usbdevice tablet -pidfile /tmp/ Cros_vm_9222/kvm.pid -chardev 'pipe,id=control_pipe,path=/tmp/cros_vm_9222/kvm.monitor' -serial file:/tmp/cros_vm_9222/kvm.monitor.serial -mon 'chardev=control_pipe' -cpu SandyBridge, -invpcid,-tsc-deadline,check -device 'virtio-net,netdev=eth0' -netdev 'user,id=eth0,net=10.0.2.0/27,hostfwd=tcp:127.0.0.1:9222-:22' -drive 'file=/mnt/host/source/src/build/images/amd64-generic/R71-11065.0.2018_09_15_1932-a1/chromiumos_qemu_image.bin,index=0,media=disk,cache=unsafe,format=raw' -enable-kvm
VNC server running on '127.0.0.1;5900'
```

### Log in to the virtual machine

As you can see from the above information, the virtual machine's ssh 22 port is mapped to the local 127.0.0.1:9222 port.

```
ssh root@localhost -p 9222
```
Enter the system with the password `test0000`. You can also use the `chronos` account mentioned above to enter the system.

```
localhost ~ # uname -r
4.4.155-15233-g439d815e28b7
Localhost ~ # cat /etc/lsb-release
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

### Other issues

Using the VNC client connect to 127.0.0.1:5900, the connection was successful, but no content was displayed, just a black window.
