---
title: PVE usage
date: 2023-09-06 23:03:34
tags: PVE
---

### LXC安装Docker

为了避免污染pve宿主系统，在lxc内套娃docker

> 从alpine模版创建一个lxc容器，注意去掉`无特权的容器`的勾选；

> 编辑 `/etc/pve/lxc/{id}.conf`，在末尾加入以下开启权限

```bash
## 磁盘映射
mp0: /mnt/volume1,mp=/volume1,replicate=0
mp1: /mnt/volume2,mp=/volume2,replicate=0
mp2: /mnt/volume3,mp=/volume3,replicate=0
mp3: /mnt/volume4,mp=/volume4,replicate=0
#从另一个zfs数据区映射数据给容器，部署的docker compose文件和config都在这里
mp4: /mnt/Seagate1t-files/data/home,mp=/home/yourname,replicate=0 
## pct set {id} -mp0 /mnt/down,mp=/data 和 pct set {id} -mp0 /dev/sda1,mp=/data

## 指定绑定核心
lxc.cgroup2.cpuset.cpus: 2-5
lxc.apparmor.profile: unconfined
lxc.cap.drop:
# lxc.cgroup2.devices.allow: a  #表示全部允许，一般不用开
## 核显
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
## tun
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

> 安装ssh、Docker等必要软件

```bash
$ apk add --no-cache --update vim bash openssh-server docker docker-compose openrc tzdata sudo shadow 
# 如果需要部署samba、nfs、webdav还需要安装，不建议，后面用docker实现
$ apk add --no-cache --update samba nfs-utils nginx-mod-http-dav-ext
#开机启动
$ echo "sshd docker samba rpcbind" | xargs -n1 rc-update add boot
# 重启或者单独开启Docker
$ service docker start
# 设置时区
$ ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# 修改/etc/sudoers，去除sudo组前面的# 注意文件只读：chmod +w /etc/sudoers
# 添加用户
$ useradd -d /home/yourname -u 1026 -g 100 -G root,wheel,docker -m -p 1 yourname
$ sudo adduser -H -D -s /sbin/nologin yourname2
```

> 后续重启后用普通用户进入，用compose文件部署容器。

### LXC安装普通镜像，模版制作

> 调整img镜像磁盘大小

```bash
$ qemu-img info name.img
$ qemu-img resize name.img 20G
```

> 挂载提取rootfs

```bash
$ modprobe nbd
$ qemu-nbd -c /dev/nbd0 -f raw name.img
# 然后查看rootfs所在分区
$ lsblk -f /dev/nbd0
$ mount /dev/nbd0p2 /mnt
# rootfs就在/mnt了
```

> 制作模板

```bash
$ cd /mnt
$ tar zcf /var/lib/vz/template/cache/name.tar.gz *
# 这样在pve管理页面的CT模板里就可以看到这个tar
```

> 新建LXC

```bash
$ pct create 103 local:vztmpl/istoreOS.tar.gz --rootfs local:1 --ostype unmanaged --hostname istoreOS --arch amd64 --cores 2 --memory 2048 --swap 0 -net0 bridge=vmbr4,name=eth0
```

> 完成后卸载

```bash
$ umount /mnt
$ qemu-nbd -d /dev/nbd0
```

#### 容器中alpine的ssh无法启动，需要安装openrc

```bash
apk add openssh-server --no-cache
还不行，试试如下：
ssh-keygen -A
rc-status
touch /run/openrc/softlevel
/etc/init.d/sshd start
```

### 虚拟机安装群晖RR引导

参考：

-	[Proxmox VE(PVE)8.0安装黑群晖NAS并直通硬盘 – 分享 (iminling.com)](https://www.iminling.com/2024/pve-install-nas)
-	[2024年PVE8最新安装使用指南|安装黑群晖｜img格式镜像安装_NAS存储_什么值得买 (smzdm.com)](https://post.smzdm.com/p/al82z7ee/)

```bash
【1】通用半白码（任何群晖型号都可以激活半白效果）:
sn=V8YQ2J3B5IIZJ
mac=021132222536
【2】通用半白码（任何群晖型号都可以激活半白效果）:
sn=V9ETLA32CTYYX
mac=0211322601B7
SA6400半白码
SN=227IUMR8HWA07
MAC1=9009D02A95D1
MAC2=9009D02A95D2
```

> 群晖调整存储池名称

```bash
synospace --meta -e
synospace --meta -s -i reuse_2 /dev/md4
synospace --meta -s -i reuse_1 /dev/md2
synospace --meta -s -i reuse_4 /dev/vg1

对于7.0以后的需要同时修改
/var/lib/space/space_table
lvm lvrename vg1 volume_3 volume_1
```

#### 备份RR

```bash
# 备份为 disk.img.gz, 自行导出.
$ dd if=`blkid | grep 'LABEL="RR3"' | cut -d3 -f1` | gzip > disk.img.gz
# 结合 transfer.sh 直接导出链接
$ curl -skL --insecure -w '\n' --upload-file disk.img.gz https://transfer.sh
```

#### 挂载引导盘

```bash
$ sudo -i
$ echo 1 > /proc/sys/kernel/syno_install_flag
$ ls /dev/synoboot*    # 正常会有 /dev/synoboot  /dev/synoboot1  /dev/synoboot2  /dev/synoboot3
# 挂载第1个分区
$ mkdir -p /tmp/synoboot1 
$ mount /dev/synoboot1 /tmp/synoboot1 
$ ls /tmp/synoboot1/
# 挂载第2个分区
$ mkdir -p /tmp/synoboot2
$ mount /dev/synoboot2 /tmp/synoboot2
$ ls /tmp/synoboot2/
```

#### 如果希望一些不具备管理员权限的用户能够直接执行docker，则需要单独建立一个群组，用于权限的分配。

```bash
$ sudo synogroup --add docker
$ sudo synogroup --memberadd docker <username1> <username2> ...
$ sudo synogroup --get docker
$ sudo chgrp docker /var/run/docker.sock
# 或者简单的：
$ sudo chgrp administrators /var/run/docker.sock
```

#### ABB本地过验证

```bash
https://192.168.2.248:5001/webapi/auth.cgi?api=SYNO.API.Auth&version=3&method=login&account=jervis&passwd=Gangdu%4088&format= cookie
https://192.168.2.248:5001/webapi/entry.cgi?api=SYNO.ActiveBackup.Activation&method=set&version=1&activated=true&serial_number=227IUMR8HWA07
```

### 虚拟机安装绿联OS

参考： [绿联DXP系列新品 PVE底层+UGOS Pro虚拟机 傻瓜式教程_网络存储_什么值得买 (smzdm.com)](https://post.smzdm.com/p/aeq9n86m/)
但是目前新版本系统无法识别内置硬盘，即使直通控制器也不行。

### 其他应用

> rustdesk
> [使用Docker自定义配置部署RustDesk Server - 独思则滞而不通 - 博客园 (cnblogs.com)](https://www.cnblogs.com/HeisenbergUncertainty/p/17908858.html)

> 查看周期性运行进程

```bash
apt install bpftrace && execsnoop.bt
```

> 常用命令

```
$ intel_gpu_top -d drm:/dev/dri/renderD128
$ watch -n 1 "cat /proc/cpuinfo | grep MHz"
$ mpstat -P ALL
$ apt install --no-install-recommends --no-install-suggests
```

```bash
# 编辑systemctl服务
$ systemctl edit lvm2-monitor.service
[Service]
ExecStartPre=/usr/local/bin/lvm_scan.sh pre
ExecStartPost=/usr/local/bin/lvm_scan.sh post
```

> 一些备份维护定时任务，根据需求调整

```bash
# 定期开关硬盘休眠服务
0 0 * * * systemctl start hd-idle
0 7 * * * systemctl stop hd-idle
# 备份配置
00 12 * * 5 rsync -avz --relative /etc/ugreen-leds.conf /usr/local/bin/lvm_scan.sh /etc/fstab /etc/pve /etc/network/interfaces /etc/ugreen-leds.conf /etc/fancontrol /usr/local/bin/aggregate_temp.sh /etc/modules /etc/default/grub /etc/default/hd-idle /etc/lvm/lvm.conf /etc/systemd/system/lvm_scan_p* /etc/systemd/system/fake_temp_fs.* /mnt/volume4/PVE/pve_cfg/
# 个别盘不需要跟随hd-idle休眠的话，可以定时查看下保持唤醒
*/50 7-23 * * * hdparm -C /dev/sdc|grep 'standby' || ls -l /mnt/volume3
```
