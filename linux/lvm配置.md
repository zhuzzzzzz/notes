#### lvm逻辑卷管理

##### 基本管理命令

```shell
# 查看文件系统使用情况
df -h
# 确定文件系统类型
df -T

# 查看磁盘物理结构
lsblk

# 查看 LVM 逻辑结构
vgs
vgdisplay
lvs
lvdisplay
```

##### 添加磁盘至虚拟卷组

```shell
# 如果是对整块磁盘操作
pvcreate /dev/sdb

# 如果是对分区操作 (例如 /dev/sdb1)，需先用 fdisk 或 parted 创建分区
# fdisk /dev/sdb (创建分区后)
pvcreate /dev/sdb1

# 将 PV 加入卷组 (VG)
# 语法: vgextend <卷组名> <物理卷设备>
vgextend cl /dev/sdb
# 或
vgextend vg_data /dev/sdb1
```

##### 扩容逻辑卷

```shell
# 语法: lvextend -L <+增加的大小> <逻辑卷路径>
# 示例1: 增加10G空间
lvextend -L +10G /dev/mapper/cl-root

# 示例2: 使用VG中所有剩余空间
lvextend -l +100%FREE /dev/vg_data/lv_data

# 刷新文件系统（XFS）
xfs_growfs /
# 或
xfs_growfs /data

# 刷新文件系统（ext4）
resize2fs /dev/mapper/cl-root
# 或
resize2fs /dev/vg_data/lv_data
```

