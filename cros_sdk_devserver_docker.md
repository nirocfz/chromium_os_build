## devserver

Reference [Dev server](https://www.chromium.org/chromium-os/how-tos-and-troubleshooting/using-the-dev-server)

In the local machine `cd $HOME/chromiumos`, execute `cros_sdk` to enter the chroot environment and execute `start devserver`, which works fine.

```
$ start_devserver
[16/Sep/2018:14:39:00] DEVSERVER Using cache directory /var/lib/devserver/static/cache
[16/Sep/2018:14:39:00] DEVSERVER Serving from /var/lib/devserver/static
[16/Sep/2018:14:39:00] XBUDDY Using shadow config file stored at /mnt/host/source/src/platform/dev/shadow_xbuddy_config.ini
[16/Sep/2018:14:39:00] ENGINE Listening for SIGHUP.
[16/Sep/2018:14:39:00] ENGINE Listening for SIGTERM.
[16/Sep/2018:14:39:00] ENGINE Listening for SIGUSR1.
[16/Sep/2018:14:39:00] ENGINE Bus STARTING
[16/Sep/2018:14:39:00] ENGINE Started monitor thread '_TimeoutMonitor'.
[16/Sep/2018:14:39:00] ENGINE Serving on :::8080
[16/Sep/2018:14:39:00] ENGINE Bus STARTED
```

Requesting port 8080 that `devserver` listens locally

```
$ curl localhost:8080
Welcome to the Dev Server!<br>
<br>
Here are the available methods, click for documentation:<br>
<br>
<a href=doc/api/fileinfo>api/fileinfo</a><br>
<a href=doc/api/hostinfo>api/hostinfo</a><br>
<a href=doc/api/hostlog>api/hostlog</a><br>
<a href=doc/api/setnextupdate>api/setnextupdate</a><br>
<a href=doc/build>build</a><br>
<a href=doc/check_health>check_health</a><br>
<a href=doc/collect_cros_au_log>collect_cros_au_log</a><br>
<a href=doc/controlfiles>controlfiles</a><br>
<a href=doc/cros_au>cros_au</a><br>
<a href=doc/get_au_status>get_au_status</a><br>
<a href=doc/handler_cleanup>handler_cleanup</a><br>
<a href=doc/is_staged>is_staged</a><br>
<a href=doc/kill_au_proc>kill_au_proc</a><br>
<a href=doc/latestbuild>latestbuild</a><br>
<a href=doc/list_image_dir>list_image_dir</a><br>
<a href=doc/list_suite_controls>list_suite_controls</a><br>
<a href=doc/locate_file>locate_file</a><br>
<a href=doc/post_au_status>post_au_status</a><br>
<a href=doc/setup_telemetry>setup_telemetry</a><br>
<a href=doc/stage>stage</a><br>
<a href=doc/symbolicate_dump>symbolicate_dump</a><br>
<a href=doc/update>update</a><br>
<a href=doc/xbuddy>xbuddy</a><br>
<a href=doc/xbuddy_capacity>xbuddy_capacity</a><br>
<a href=doc/xbuddy_list>xbuddy_list</a><br>
<a href=doc/xbuddy_translate>xbuddy_translate</a>
```

## Make docker image of cros_sdk(devserver)

### Install Docker

```
$ sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

$ sudo apt-key fingerprint 0EBFCD88
pub   rsa4096 2017-02-22 [SCEA]
      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
sub   rsa4096 2017-02-22 [S]

$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

$ sudo apt-get update
$ sudo apt-get install docker-ce
```

The `docker` service communicates with the `docker` tool via a local UNIX SOCKET. This UNIX SOCKET belongs to the `root` user, and the `docker` group is readable and writable.

```
srw-rw---- 1 root docker 0 9月  16 17:29 /var/run/docker.sock
```

Adding current user to the `docker` group, the current user has the privileges of this UNIX SOCKET. Then I can run `docker` command without `sudo`.

```
$ sudo groupadd docker
$ sudo usermod -aG docker $USER
```

### Create a Docker image

```
$ mkdir /path/to/cros_sdk_docker
$ cd /path/to/cros_sdk_docker
$ vim Dockerfile
```

Write the following in `Dockerfile`

```
FROM ubuntu:16.04
RUN apt-get update && apt-get install -y git-core gitk git-gui curl lvm2 thin-provisioning-tools python-pkg-resources python-virtualenv python-oauth2client subversion ca-certificates python xz-utils sudo
RUN useradd cros --create-home
RUN echo 'cros ALL=(ALL:ALL) NOPASSWD: ALL' >> /etc/sudoers
USER cros
RUN git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git /home/cros/depot_tools
ENV PATH /home/cros/depot_tools:$PATH
RUN mkdir /home/cros/chromiumos
WORKDIR /home/cros/chromiumos
RUN repo init -u https://chromium.googlesource.com/chromiumos/manifest.git --repo-url https://chromium.googlesource.com/external/repo.git -g minilayout
RUN repo sync -j4
EXPOSE 8080
ENTRYPOINT ["/home/cros/depot_tools/cros_sdk"]
```

### Make the image

Execute in the directory where `Dockerfile` is located (`/path/to/cros_sdk_docker`)

```
$ docker build -t cros_sdk .
```
Then check the list of images of `docker`

```
$ docker image list
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
cros_sdk            latest              f5b6b9be5fe5        11 seconds ago      1.97GB
ubuntu              16.04               b9e15a5d1e1a        10 days ago         115MB
```

### Run the container

```
docker run cros_sdk
```

There's an error:

```
cros_sdk: Unhandled exception:
Traceback (most recent call last):
  File "/home/cros/chromiumos/chromite/bin/cros_sdk", line 169, in <module>
    DoMain()
  File "/home/cros/chromiumos/chromite/bin/cros_sdk", line 165, in DoMain
    commandline.ScriptWrapperMain(FindTarget)
  File "/home/cros/chromiumos/chromite/lib/commandline.py", line 912, in ScriptWrapperMain
    ret = target(argv[1:])
  File "/home/cros/chromiumos/chromite/scripts/cros_sdk.py", line 844, in main
    _ReExecuteIfNeeded([sys.argv[0]] + argv)
  File "/home/cros/chromiumos/chromite/scripts/cros_sdk.py", line 676, in _ReExecuteIfNeeded
    cgroups.Cgroup.InitSystem()
  File "/home/cros/chromiumos/chromite/lib/memoize.py", line 32, in wrapper
    val = functor(obj)
  File "/home/cros/chromiumos/chromite/lib/cgroups.py", line 148, in InitSystem
    _EnsureMounted(cls.CGROUP_ROOT, cgroup_root_args)
  File "/home/cros/chromiumos/chromite/lib/cgroups.py", line 137, in _EnsureMounted
    osutils.SafeMakedirs(mnt, sudo=True)
  File "/home/cros/chromiumos/chromite/lib/osutils.py", line 254, in SafeMakedirs
    os.makedirs(path, mode)
  File "/usr/lib/python2.7/os.py", line 157, in makedirs
    mkdir(name, mode)
OSError: [Errno 30] Read-only file system: '/sys/fs/cgroup/cpuset/cros'
```

I tried deleting the line `ENTRYPOINT` in `Dockerfile`, make a new image, execute `docker run -it cros_sdk` to run the new container interactively, and manually execute `cros_sdk` in the container environment, there's the same `Read-only file system` problem.

## Troubleshooting

1. How to solve the problem of `Read-only file system`?

Add `--privileged` parameter when executing `docker run`.

```
docker run --privileged cros_sdk
```

Reference [run chroot within docker](https://stackoverflow.com/questions/33235395/run-chroot-within-docker).

2. How to automate `start_devserver`?

`ENTRYPOINT ["/home/cros/depot_tools/cros_sdk"]`, this line tells docker to run `/home/cros/depot_tools/cros_sdk` entering the chroot environment when running the container. How to automatically execute `start_devserver after entering the chroot environment? ` command?

```
docker run --privileged cros_sdk -- start_devserver
```

Run the `cros_sdk` container by the above command.

But actually, the executing of `/home/cros/depot_tools/cros_sdk` is still a problem.

3. Problems creating and entering the chroot environment in the container

Executing the above command, new error occurred

```
cros_sdk: Unhandled exception:
Traceback (most recent call last):
  File "/home/cros/chromiumos/chromite/bin/cros_sdk", line 169, in <module>
    DoMain()
  File "/home/cros/chromiumos/chromite/bin/cros_sdk", line 165, in DoMain
    commandline.ScriptWrapperMain(FindTarget)
  File "/home/cros/chromiumos/chromite/lib/commandline.py", line 912, in ScriptWrapperMain
    ret = target(argv[1:])
  File "/home/cros/chromiumos/chromite/scripts/cros_sdk.py", line 949, in main
    if not cros_sdk_lib.MountChroot(options.chroot, create=True):
  File "/home/cros/chromiumos/chromite/lib/cros_sdk_lib.py", line 291, in MountChroot
    cros_build_lib.SudoRunCommand(cmd, capture_output=True, print_cmd=False)
  File "/home/cros/chromiumos/chromite/lib/cros_build_lib.py", line 283, in SudoRunCommand
    return RunCommand(cmd, **kwargs)
  File "/home/cros/chromiumos/chromite/lib/cros_build_lib.py", line 647, in RunCommand
    raise RunCommandError(msg, cmd_result)
chromite.lib.cros_build_lib.RunCommandError: return code: 5; command: lvcreate -q -L499G -T cros_home+cros+chromiumos+chroot_001/thinpool -V500G -n chroot
  /run/lvm/lvmetad.socket: connect failed: No such file or directory
  WARNING: Failed to connect to lvmetad. Falling back to internal scanning.
  /dev/cros_home+cros+chromiumos+chroot_001/lvol0: not found: device not cleared
  Aborting. Failed to wipe start of new LV.

cmd=['lvcreate', '-q', '-L499G', '-T', 'cros_home+cros+chromiumos+chroot_001/thinpool', '-V500G', '-n', 'chroot']
```

Suspected to be a problem with hard disk space, so I transferred `/var/lib/docker` to an external hard drive.

```
# run as `root` here
service docker stop
mv /var/lib/docker /media/noodles/Tatooine/playground/docker
ln -s /media/noodles/Tatooine/playground/docker /var/lib/docker
service docker start
```

Execute `docker run --privileged cros_sdk -- start_devserver` again and there's another error.

```
sudo: effective uid is not 0, is /usr/bin/sudo on a file system with the 'nosuid' option set or an NFS file system without root privileges?
```

This is the error reported when run `sudo` in the container. Follow the instruction above, I remounted the external hard drive without `nosuid` option.  Return to the previous question, so it should not be a problem related to hard drive space.

~~Perhaps, the next step I should create a full image with `-g minilayout` in the step of `repo init`(`Dockerfile file`).~~ No, I got only a huge docker image which has the same issue.