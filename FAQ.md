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
