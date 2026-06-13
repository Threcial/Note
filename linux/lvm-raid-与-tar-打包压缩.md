---
title: LVM、RAID 与 tar 打包压缩
type: knowledge
status: evergreen
created: 2026-06-13
updated: 2026-06-13
tags:
  - Linux
  - CentOS7
  - LVM
  - RAID
  - tar
  - 磁盘管理
  - 打包压缩
aliases:
  - DAY04
  - LVM 逻辑卷管理
  - RAID 磁盘阵列
  - tar 打包压缩
Belongs to:
  - "[[Linux 运维基础]]"
  - "[[磁盘管理]]"
  - "[[压缩归档]]"
Related to:
  - "[[day03 Linux 用户、权限、sudo 与磁盘文件系统基础]]"
  - "[[磁盘分区]]"
  - "[[文件系统]]"
  - "[[mount]]"
  - "[[find 命令]]"
  - "[[xargs]]"
Has:
  - "[[LVM]]"
  - "[[PV]]"
  - "[[VG]]"
  - "[[LV]]"
  - "[[PE]]"
  - "[[RAID0]]"
  - "[[RAID1]]"
  - "[[RAID5]]"
  - "[[RAID10]]"
  - "[[tar]]"
_organized: true
---
# LVM、RAID 与 tar 打包压缩

## 概览

- LVM 把物理磁盘/分区抽象成 PV，再组合成 VG，最后从 VG 中划分 LV，便于后期扩容。
- LV 扩容分两步：先扩逻辑卷，再扩文件系统。XFS 使用 `xfs_growfs`。
- RAID 解决的是磁盘性能和冗余问题，不等于备份。
- `tar` 主要负责归档打包；结合 `gzip` 后才形成常见的 `.tar.gz`。
- `find | xargs tar` 可以打包查找结果；`find -exec tar -zcvf 固定包名 {} \;` 会反复覆盖，不适合多文件归档。

## 关系表

| 条目   | 上游依赖          | 下游用途        |
| ---- | ------------- | ----------- |
| PV   | 物理磁盘或分区       | 组成 VG       |
| VG   | 一个或多个 PV      | 提供可分配容量     |
| LV   | VG 中划分出的逻辑卷   | 格式化、挂载、扩容   |
| PE   | VG/LV 的最小分配单位 | 影响 LVM 空间分配 |
| RAID | 多块物理磁盘        | 性能提升、冗余保护   |
| tar  | 文件/目录/find 结果 | 归档、备份、迁移    |

## LVM 逻辑卷管理

参考链接：

```yaml
https://cloud.tencent.com/developer/article/1937939
```

LVM 关系：

```text
物理磁盘/分区 -> PV -> VG -> LV -> 文件系统 -> 挂载点
```

| 概念 | 全称              | 含义                     |
| -- | --------------- | ---------------------- |
| PV | Physical Volume | 物理卷，由分区或整块硬盘转换而来       |
| VG | Volume Group    | 卷组，由多个 PV 组成，可动态扩容     |
| LV | Logical Volume  | 逻辑卷，从 VG 中划分出来，供格式化和挂载 |
| PE | Physical Extent | LVM 最小存储分配单位，默认常见 4MB  |

建立 LVM 的步骤：

1. 把物理硬盘变成分区，或直接使用整块硬盘。
2. 把物理分区建立成 PV。
3. 把多个 PV 组合成 VG。
4. 从 VG 中划分 LV。
5. 对 LV 创建文件系统并挂载。

## 创建 LVM 分区

进入分区工具：

```bash
fdisk /dev/sdb
```

创建 3 个 2G 主分区的交互逻辑：

```text
n -> p -> 1 -> 回车 -> +2G
n -> p -> 2 -> 回车 -> +2G
n -> p -> 3 -> 回车 -> +2G
p
```

把分区类型改为 Linux LVM：

```text
t -> 1 -> 8e
t -> 2 -> 8e
t -> 3 -> 8e
p
w
```

`8e` 表示 Linux LVM。

> **注意**\
> 原始记录中第一次创建分区时把 `+2G` 填到了 `First sector`，后面又删除重建。正确方式是起始扇区通常直接回车，结束扇区写 `+2G`。

## PV 物理卷

```bash
pvcreate /dev/sdb1
pvcreate /dev/sdb2
pvcreate /dev/sdb3
pvscan
pvdisplay
pvremove /dev/sdb3
```

| 命令          | 作用        |
| ----------- | --------- |
| `pvcreate`  | 创建物理卷     |
| `pvscan`    | 扫描物理卷     |
| `pvdisplay` | 查看物理卷详细信息 |
| `pvremove`  | 删除物理卷标签   |

## VG 卷组

创建卷组：

```bash
vgcreate xvg /dev/sdb1 /dev/sdb2
```

查看卷组：

```bash
vgscan
vgdisplay
```

扩容卷组：

```bash
vgextend xvg /dev/sdb3
```

缩减卷组：

```bash
vgreduce xvg /dev/sdb3
```

删除卷组：

```bash
vgremove xvg
```

指定 PE 大小：

```bash
vgcreate -s 4M xvg /dev/sdb1 /dev/sdb2
```

| vgdisplay 字段     | 含义          |
| ---------------- | ----------- |
| `VG Name`        | 卷组名         |
| `VG Access`      | 访问状态        |
| `VG Status`      | 是否可调整大小     |
| `Cur PV`         | 当前 PV 数量    |
| `VG Size`        | 卷组大小        |
| `PE Size`        | PE 大小       |
| `Total PE`       | PE 总数       |
| `Free PE / Size` | 空闲 PE 数量和容量 |

## LV 逻辑卷

创建逻辑卷：

```bash
lvcreate -L 3G -n xlv xvg
```

查看逻辑卷：

```bash
lvscan
lvdisplay
```

| 命令          | 作用        |
| ----------- | --------- |
| `lvcreate`  | 创建逻辑卷     |
| `lvscan`    | 扫描逻辑卷     |
| `lvdisplay` | 查看逻辑卷详细信息 |
| `lvresize`  | 调整逻辑卷大小   |

## LV 格式化、挂载与扩容

格式化：

```bash
mkfs.xfs /dev/xvg/xlv
```

挂载：

```bash
mkdir /xyj
mount /dev/xvg/xlv /xyj/
df -h
```

扩容逻辑卷到 5G：

```bash
lvresize -L 5G /dev/xvg/xlv
```

此时 `df -h` 可能仍显示旧容量，因为文件系统还没扩容。

扩容 XFS 文件系统：

```bash
xfs_growfs /dev/xvg/xlv
```

永久挂载：

```fstab
/dev/xvg/xlv /xyj xfs defaults 0 0
```

> **注意**\
> `lvresize -L 5G` 的意思是把逻辑卷调整到 5G，不是“增加 5G”。如果目标是 20G，就写 `-L 20G`。

## RAID 磁盘阵列

参考链接保留：

```text
RAID：https://zhuanlan.zhihu.com/p/77433161
RAID01 问题：https://www.zhihu.com/question/395165776
阵列卡：https://item.jd.com/10179285174555.html?spmTag=YTAyMTkuYjAwMjM1Ni5jMDAwMDQ2ODkuc2VhcmNoX2NvbmZpcm0lMkNhMDI0MC5iMDAyNDkzLmMwMDAwNDAyNy4xMSUyM3NrdV9jYXJk
```

| 类型     | 最少磁盘 | 特点               | 风险                     |
| ------ | ---- | ---------------- | ---------------------- |
| RAID0  | 2    | 读写速度最快，数据分散写入多盘  | 任意一块坏盘都会丢数据            |
| RAID1  | 2    | 镜像备份，一块盘是另一块盘的副本 | 空间利用率低                 |
| RAID5  | 3    | 允许坏一块盘不丢数据       | 坏盘状态下 IO 性能下降明显，重建风险较高 |
| RAID10 | 4    | 兼顾镜像冗余和条带性能      | 成本高于 RAID0/1           |

原始记录中的使用倾向：

- RAID0 和 RAID1 使用较多。
- 同时考虑速度和安全时，用 RAID10。
- RAID5 在坏盘情况下 IO 下降明显，实际使用要谨慎。
- 阵列卡应有缓存和电池，防止掉电导致缓存数据丢失。

> **注意**\
> RAID 不是备份。RAID 只能缓解部分硬件故障风险，不能防误删、误覆盖、勒索、逻辑损坏。

## tar 打包压缩

### 概念

`tar` 的核心是归档，把多个文件或目录打成一个包。是否压缩取决于是否配合压缩算法，例如 gzip。

| 文件名       | 含义                 |
| --------- | ------------------ |
| `.tar`    | 只打包，不压缩            |
| `.tar.gz` | 先 tar 打包，再 gzip 压缩 |

## tar 命令格式

```bash
tar [OPTION...] [FILE]...
```

常用选项：

| 选项               | 含义            |
| ---------------- | ------------- |
| `-c`             | create，创建归档   |
| `-v`             | verbose，显示过程  |
| `-f`             | file，指定归档文件   |
| `-z`             | 通过 gzip 压缩或解压 |
| `-x`             | extract，解包    |
| `-t`             | list，查看包内容    |
| `-C`             | 指定解压目录        |
| `--exclude`      | 排除文件或目录       |
| `--exclude-from` | 从文件读取排除规则     |

## 只打包

准备环境：

```bash
mkdir data
cd data/
mkdir {a..z}
touch {1..50}
cd ..
```

打包：

```bash
tar -cvf data.tar ./data
ls
```

输出文件：

```text
data.tar
```

## 打包并 gzip 压缩

```bash
tar -zcvf data.tar.gz ./data
ls
```

输出文件：

```text
data.tar.gz
```

## 查看压缩包内容

```bash
tar -tf data.tar.gz
```

## 解压到指定目录

```bash
tar -zxvf data.tar.gz -C /root/
cd /root/data
ls
```

## 打包时排除文件或目录

```bash
tar -zcvf weibo.tar.gz --exclude=*4 ./data
```

排除多个文件：

```bash
tar -zcvf hh.tar.gz --exclude=yum.log --exclude=wtmp --exclude=secure .
tar -zcvf hh.tar.gz --exclude=yum.log --exclude=wtmp --exclude=secure --exclude=aa --exclude=ppp --exclude=sa .
```

排除规则：

- `--exclude=名称` 和 `--exclude 名称` 都可用，推荐使用 `=`。
- `--exclude=文件名` 会排除所有同名文件。
- 排除目录时，目录名后面最好不要带 `/`。
- 如果归档目标使用相对路径，`--exclude` 和 `--exclude-from` 也应使用相对路径。
- 如果归档目标使用绝对路径，排除规则也应使用绝对路径。

## find 与 tar

用 `find` 找到符合条件的文件，再交给 `tar`：

```bash
find . -size +10k | xargs tar -zcvf hh.tar.gz
```

不推荐这样写：

```bash
find . -maxdepth 1 -size +10k -exec tar -zcvf kk.tar.gz {} \;
```

原因：`-exec` 会对每一个匹配文件单独执行一次 `tar -zcvf kk.tar.gz`，同一个压缩包名会被反复覆盖，最后可能只剩最后一个文件。

## 命令速查

| 目标          | 命令                                        |
| ----------- | ----------------------------------------- |
| 创建 PV       | `pvcreate /dev/sdb1`                      |
| 创建 VG       | `vgcreate xvg /dev/sdb1 /dev/sdb2`        |
| 扩容 VG       | `vgextend xvg /dev/sdb3`                  |
| 创建 LV       | `lvcreate -L 3G -n xlv xvg`               |
| 格式化 LV      | `mkfs.xfs /dev/xvg/xlv`                   |
| 扩容 LV 到 5G  | `lvresize -L 5G /dev/xvg/xlv`             |
| 扩 XFS 文件系统  | `xfs_growfs /dev/xvg/xlv`                 |
| tar 打包      | `tar -cvf data.tar ./data`                |
| tar.gz 打包压缩 | `tar -zcvf data.tar.gz ./data`            |
| 查看包内容       | `tar -tf data.tar.gz`                     |
| 解压到目录       | `tar -zxvf data.tar.gz -C /root/`         |
| 排除文件        | `tar -zcvf a.tar.gz --exclude=xxx ./data` |

## 易错点

| 场景                         | 说明                             |
| -------------------------- | ------------------------------ |
| LVM 分区时把 `+2G` 填到起始扇区      | 起始扇区通常回车，结束扇区写容量               |
| LV 扩容后 `df -h` 不变          | 还需要扩文件系统，例如 XFS 用 `xfs_growfs` |
| `lvresize -L 5G` 理解成增加 5G  | 实际是调整到 5G                      |
| RAID 当成备份                  | RAID 不是备份                      |
| tar 与 gzip 混淆              | tar 是归档，gzip 是压缩               |
| `find -exec tar -zcvf 同名包` | 会反复覆盖，应用 `xargs` 或其他方式聚合       |
| `--exclude` 路径不一致          | 相对路径和绝对路径要匹配                   |

## 关联条目

- [[LVM]]
- [[PV]]
- [[VG]]
- [[LV]]
- [[RAID]]
- [[磁盘分区]]
- [[文件系统]]
- [[tar]]
- [[find 命令]]
- [[xargs]]
