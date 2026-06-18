---
type: Concept
status: Organized
Belongs to: "[[Linux]]"
Related to:
  - "[[linux-磁盘与文件系统]]"
  - "[[磁盘管理]]"
Has:
  - "[[LVM]]"
  - "[[PV]]"
  - "[[VG]]"
  - "[[LV]]"
  - "[[RAID]]"
  - "[[tar]]"
source: ""
created: 2026-06-13
updated: 2026-06-13
---

# LVM、RAID 与 tar

## LVM 逻辑卷管理

LVM 把物理磁盘/分区抽象成 PV，再组合成 VG，最后从 VG 中划分 LV，便于后期扩容。

### 核心概念

| 概念 | 全称 | 含义 |
| --- | --- | --- |
| PV | Physical Volume | 物理卷，由分区或整块硬盘转换 |
| VG | Volume Group | 卷组，由多个 PV 组成 |
| LV | Logical Volume | 逻辑卷，供格式化和挂载 |
| PE | Physical Extent | 最小存储分配单位，默认 4MB |

建立步骤：分区 → PV → VG → LV → 文件系统 → 挂载。

### 创建 LVM 分区

```bash
fdisk /dev/sdb
# n -> p -> 1 -> 回车 -> +2G
# t -> 1 -> 8e（Linux LVM）
# w
```

### PV 物理卷

```bash
pvcreate /dev/sdb1
pvscan
pvdisplay
pvremove /dev/sdb3
```

### VG 卷组

```bash
vgcreate xvg /dev/sdb1 /dev/sdb2
vgextend xvg /dev/sdb3
vgreduce xvg /dev/sdb3
vgremove xvg
```

### LV 逻辑卷

```bash
lvcreate -L 3G -n xlv xvg
lvscan
lvdisplay
```

### 格式化、挂载与扩容

```bash
mkfs.xfs /dev/xvg/xlv
mkdir /xyj
mount /dev/xvg/xlv /xyj/

# 扩容到 5G
lvresize -L 5G /dev/xvg/xlv
xfs_growfs /dev/xvg/xlv
```

> `lvresize -L 5G` 是把逻辑卷调整到 5G，不是增加 5G。

## RAID 磁盘阵列

| 类型 | 最少磁盘 | 特点 | 风险 |
| --- | --- | --- | --- |
| RAID0 | 2 | 读写最快，数据分散写入 | 任意坏盘丢数据 |
| RAID1 | 2 | 镜像备份 | 空间利用率低 |
| RAID5 | 3 | 允许坏一块盘 | 坏盘后 IO 下降明显 |
| RAID10 | 4 | 兼顾镜像和条带 | 成本较高 |

> RAID 不是备份。RAID 只能缓解部分硬件故障风险，不能防误删、逻辑损坏。

## tar 打包压缩

### 常用选项

| 选项 | 含义 |
| --- | --- |
| `-c` | 创建归档 |
| `-v` | 显示过程 |
| `-f` | 指定归档文件 |
| `-z` | 通过 gzip 压缩或解压 |
| `-x` | 解包 |
| `-t` | 查看包内容 |
| `-C` | 指定解压目录 |
| `--exclude` | 排除文件或目录 |

### 基础用法

```bash
tar -cvf data.tar ./data                # 只打包
tar -zcvf data.tar.gz ./data            # 打包并压缩
tar -tf data.tar.gz                     # 查看内容
tar -zxvf data.tar.gz -C /root/         # 解压到指定目录
tar -zcvf hh.tar.gz --exclude=*.log .   # 打包时排除
```

### find 与 tar

```bash
find . -size +10k | xargs tar -zcvf hh.tar.gz
```

## 易错点

| 场景 | 说明 |
| --- | --- |
| LV 扩容后 `df -h` 不变 | 还需要扩文件系统（XFS 用 `xfs_growfs`） |
| `lvresize -L 5G` 理解成增加 5G | 实际是调整到 5G |
| RAID 当成备份 | RAID 不是备份 |
| tar 与 gzip 混淆 | tar 是归档，gzip 是压缩 |
| `--exclude` 路径不一致 | 相对路径和绝对路径要匹配 |
