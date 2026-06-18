---
type: Concept
status: Organized
Belongs to: "[[Linux]]"
Related to:
  - "[[磁盘管理]]"
  - "[[文件系统]]"
Has:
  - "[[inode]]"
  - "[[软链接]]"
  - "[[硬链接]]"
  - "[[MBR]]"
  - "[[mount]]"
  - "[[swap]]"
  - "[[fstab]]"
source: ""
created: 2026-06-13
updated: 2026-06-13
---

# 磁盘与文件系统

磁盘要使用，一般经历分区、格式化、挂载三个阶段。

## 存储设备类型

| 类型 | 特点 |
| --- | --- |
| HDD 机械硬盘 | 磁性碟片存储，怕震动，随机读写慢 |
| SSD 固态硬盘 | 闪存颗粒存储，随机读写快 |

## 硬盘逻辑结构

| 结构 | 含义 |
| --- | --- |
| 磁道 | 盘片上的同心圆逻辑区域 |
| 扇区 | 磁道上的弧段，传统大小 512Byte |
| 柱面 | 多个盘面上编号相同的磁道组成 |

## 设备命名

| 场景 | 常见设备名 |
| --- | --- |
| IDE / SATA / SCSI / SAS | `/dev/sda`、`/dev/sdb` |
| 云服务器虚拟磁盘 | `/dev/vda`、`/dev/xvda` |

## MBR 分区

MBR 位于硬盘第一个扇区（512 字节）：主引导程序（446B） + DPT 分区表（64B） + 有效标志（2B）。

限制：
- 主分区最多 4 个
- 逻辑分区必须建立在扩展分区之上

## 文件系统类型

| 类型 | 说明 |
| --- | --- |
| Linux | ext3、ext4、xfs |
| Windows | NTFS、FAT32 |
| 网络共享 | NFS、SMB |
| 分布式 | Ceph、HDFS |

CentOS 7 默认常用 XFS。

## 分区、格式化、挂载

### 设备识别

```bash
ls /dev/sd*
fdisk -l /dev/sdb
```

### fdisk 分区

```bash
fdisk /dev/sdb
```

常用交互命令：`n` 新建分区、`d` 删除、`p` 打印、`w` 保存退出。

创建 5G 主分区：

```text
n -> p -> 1 -> 回车 -> +5G -> w
```

### 格式化与挂载

```bash
mkfs.xfs /dev/sdb1
mkdir /test
mount /dev/sdb1 /test/
df -h
umount /test/
```

### 永久挂载

```bash
blkid
vim /etc/fstab
```

```fstab
UUID="271074c0-28bd-4ccb-b8ff-6076b276b7f2" /test xfs defaults 0 0
```

## 分区规划示例

| 挂载点 | 大小 | 文件系统 | 用途 |
| --- | --- | --- | --- |
| `/boot` | 2G | xfs | 启动文件 |
| `swap` | 16G | swap | 交换空间 |
| `/` | 200G | xfs | 系统根目录 |
| `/data` | 剩余 | xfs | 业务数据 |

## fstab 挂载选项

| 选项 | 含义 |
| --- | --- |
| `defaults` | rw, suid, dev, exec, nouser, async |
| `noatime` | 不更新访问时间 |
| `noexec` | 不允许执行文件 |
| `ro` / `rw` | 只读 / 可读写 |

最后两列：dump 备份（0/1/2）和 fsck 检查顺序（0/1/2）。

## swap 扩展

### 使用分区

```bash
mkswap -f /dev/sdb6
swapon /dev/sdb6
swapoff /dev/sdb6
```

### 使用文件

```bash
dd if=/dev/zero of=swap_file bs=1M count=500
chmod 600 swap_file
mkswap swap_file
swapon swap_file
```

## df 与 du

```bash
df -h
du /root/ -sh
```

## mount 挂载

### 挂载光盘

```bash
mount -t iso9660 /dev/sr0 /mnt/cdrom/
umount /mnt/cdrom
```

### 挂载 U 盘

```bash
mount /dev/sdb1 /mnt/usb/
mount -t vfat -o iocharset=utf8 /dev/sdb1 /mnt/usb/
mount -t ntfs -o iocharset=utf8 /dev/sdb1 /mnt/usb/   # 需 ntfs-3g
```

## inode、block 与文件名

| 组成 | 作用 |
| --- | --- |
| 文件名 | 便于用户识别，关联 inode 号 |
| inode | 保存元数据：大小、属主、权限、时间、block 位置 |
| block | 真正保存文件数据 |

```bash
ls -i 文件
df -i
```

inode 在格式化时分配，小文件极多时可能出现 inode 耗尽。

## 软链接

```bash
ln -s 原地址 目标地址
```

类似 Windows 快捷方式。删除原文件后软链接变成死链接。

## 硬链接

```bash
ln 原地址 目标地址
```

- 只能链接文件
- 共享 inode，修改一个另一个同步
- 删除一个不影响另一个

## 易错点

| 场景 | 说明 |
| --- | --- |
| 修改 `/etc/fstab` 不测试 | 可能导致开机挂载失败 |
| swap 文件权限 0644 | 应改成 0600 |
| NTFS 直接挂载失败 | CentOS 7 需 `ntfs-3g` |
| 软链接目录后加 `/` 删除 | 可能删除目标目录内容 |
