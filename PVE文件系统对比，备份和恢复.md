---
title: PVE文件系统对比，备份和恢复
date: 2025-01-02 22:01:09
tags: pre, zfs
---

### 关于分区格式

在 Proxmox VE (PVE) 中，**ZFS 的使用方式确实可以更高效地共享同一块磁盘上的存储空间**，而不像传统的文件系统（如 ext4 和 XFS 配合 LVM）那样容易出现空间割裂的问题。以下是详细解释：

------

#### **1. LVM 的限制**

当使用 LVM 配置 `ext4` 或 `xfs` 时：

- **`local`（指向 LVM 卷格式化为 ext4 的目录）** 和 **`local-thin`（基于 LVM-thin 提供虚拟磁盘存储）** 是两个独立的逻辑卷。

- 空间隔离问题

  ：两者各自分配固定的空间，不能动态调整。例如：

  - 如果 `local` 的 ext4 卷空间用尽，而 `local-thin` 仍有空闲空间，这些空间无法给 `local` 使用。
  - 导致磁盘空间管理不灵活，特别是在需要灵活分配给虚拟机和容器时。

------

#### **2. ZFS 的优势**

使用 ZFS 存储时：

- ZFS 池（如 `rpool` 或自定义池）是一种基于块的动态文件系统。

- 在 ZFS 中，**`local` 和 `local-zfs`（或其他子文件系统）共用同一个存储池（ZFS pool）**。

- 共享空间

  ：ZFS 不对子文件系统或虚拟磁盘预分配空间，而是根据实际使用动态分配。

  - 比如，`local` 用于存放容器的文件和模板，`local-zfs` 用于虚拟机磁盘，这两者共享整个 ZFS 池的可用空间。

##### **示例结构**:

```plaintext
rpool
├── local (用于存放模板、ISO等)
├── local-zfs (用于虚拟机磁盘和容器数据)
```

**空间分配灵活性**：

- 如果 `local` 使用了 50GB，而 `local-zfs` 使用了 100GB，整个存储池剩余空间仍然可以自由分配给任何子文件系统。
- 不会因为固定的卷大小限制某一部分的存储使用。

------

#### **3. ZFS 的 thin provisioning 特性**

ZFS 天然支持 thin provisioning：

- 虚拟机磁盘（如 `vm-101-disk-0`）实际上是 ZFS 的子卷（zvol），占用的空间是动态分配的。
- 容器的存储（如 `subvol-100-disk-0`）也是 ZFS 子文件系统的一部分，空间同样动态管理。

这意味着，存储的实际使用量是基于实际写入数据量，而不是预先分配的容量。

------

#### **4. 比较总结**

| **存储类型**       | **特点**                                                     | **缺点**                                    |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------- |
| **LVM + ext4/XFS** | - 空间固定分配，`local` 和 `local-thin` 独立- 容器和虚拟机存储隔离 | - 空间利用率低- 扩展或调整卷大小复杂        |
| **ZFS**            | - 所有存储共享一个 ZFS 池- 动态空间分配- 自动快照、压缩和校验特性- 支持 thin provisioning | - 对 CPU 和内存有一定开销- 配置需要一定经验 |

------

#### **5. 建议**

- 如果你希望实现更灵活的空间分配和更强的存储管理功能，**ZFS 是更优的选择**。
- 如果硬件资源允许（如有足够的内存），ZFS 还可以开启压缩和数据完整性校验功能，进一步提升存储效率和可靠性。

在 PVE 中使用 ZFS，可以更高效地利用存储空间，并减少传统分区方案中的割裂问题。

---

### 分区备份和恢复

#### 注意事项

1. **备份内容确认**：
   - `rpool/ROOT/pve-1` 是 PVE 的核心系统，必须备份。
   - 如果你的虚拟机数据存储在 `rpool/data` 或 `rpool/var-lib-vz` 中，也需要纳入备份。

#### ZFS快照保存系统

要成功发送递归快照，**每个子数据集**都需要有相应的快照。以下是完整的备份步骤：

---

#### 1. 创建递归快照

为 `rpool` 及其所有子数据集创建一个一致的快照：

```bash
zfs snapshot -r rpool@backup
```

- `-r`：递归地为 `rpool` 和所有子数据集创建同名快照。

你可以通过以下命令验证快照是否已正确创建：

```bash
zfs list -t snapshot
```

确保所有相关子数据集（如 `rpool/ROOT/pve-1`、`rpool/data` 和 `rpool/var-lib-vz`）都存在 `@backup` 快照。

---

#### 2. 递归发送快照

如果有外部 USB 硬盘或 NFS 挂载点，可以将快照导出：

```bash
zfs send -R rpool@backup | gzip > /path/to/backup/rpool-backup.gz
```

重新运行递归发送命令，将快照发送到目标数据集：

```bash
zfs send -R rpool@backup | zfs receive -u Seagate1t-zfs/backup
```

- `-R`：递归发送 `rpool@backup` 及其所有子数据集的快照。
- 目标数据集 `Seagate1t-zfs/backup` 将接收到完整的 ZFS 层级结构。

---

#### 3. 验证目标数据集

确认快照和数据已经正确接收到目标数据集中：

```bash
zfs list -t snapshot Seagate1t-zfs/backup
```

检查 `Seagate1t-zfs/backup` 是否包含 `rpool` 的所有子数据集和快照。

#### 4.**增量备份**：

- 在初次备份后，可通过增量快照实现差异备份，例如：

  ```bash
  zfs snapshot -r rpool@backup2
  zfs send -RI rpool@backup rpool@backup2 | zfs receive Seagate1t-zfs/backup
  ```

- `-I`：发送两个快照间的增量数据。

#### 5.**特别注意**

`备份数据区必须设置不自动挂载，否则重启后一起挂载容易冲突，系统虚拟机可能崩溃`

```bash
zfs set canmount=off Seagate1t-zfs/backup/* #其中*要替换了每个执行，否则每次重启后会自动挂载
zfs list -o name,mountpoint,mounted,canmount
```

#### 6.恢复系统和数据

##### **1. 恢复系统盘**

- 如果备份使用了 ZFS 快照：

  1. 启动到 Live 环境，导入目标磁盘的 ZFS 池：

     ```bash
     zpool import rpool
     ```

  2. 恢复快照：

     ```bash
     zfs recv -F rpool < /path/to/backup/rpool-backup.gz
     或者
     zfs send -R Seagate1t-zfs/backup@backup | zfs receive -F rpool
     ```

  3. 配置引导程序：

     ```bash
     proxmox-boot-tool refresh
     update-initramfs -u
     update-grub
     ```

- 如果使用了镜像工具（如 Clonezilla 或 `dd`），直接写回系统盘：

  ```bash
  dd if=/path/to/backup.img of=/dev/system-disk bs=4M
  ```

##### **2. 恢复数据盘**

按照迁移时的反向操作，将数据从备份中恢复到新池。

#### 7. **dd备份与恢复**

```bash
$ dd bs=4096 if=/dev/sda status=progress | gzip -9 > /mnt/smb/pve.img.gz
$ gzip -c -d /mnt/smb/pve.img.gz | dd of=/dev/sda status=progress
```

#### 8. **注意事项**

1. **确保数据安全**：备份前核对文件系统的完整性，迁移前确保目标磁盘足够空间。
2. **引导修复**：如果更换了系统盘，可能需要修复引导程序。
3. **更新 PVE 配置**：所有存储路径（如 `/etc/pve/storage.cfg`）都需要根据新的存储池调整。
4. **测试恢复流程**：在操作前，可以通过测试环境验证备份和恢复流程的可行性。

---

### 虚拟机虚拟磁盘移动位置

1. 手动zfs命令

   ```bash
   zfs rename local-zfs/vm-104-disk-0 local-zfs/data/vm-104-disk-0
   ```

2. pve命令

   在 `/etc/pve/storage.cfg` 中添加一个新的 ZFS 存储池配置，比如 `local-zfs`：

   ```bash
   zfspool: local-zfs
           pool Seagate1t-zfs/data
           sparse
           content images,rootdir
   ```

   ```bash
   qm move_disk 101 scsi1 local-zfs
   ```

### 单独zfs分区存放文件（不要直接在zfs pool顶层直接存文件)

   #### 1. **创建独立数据集**

   在根数据集下创建一个专用数据集（如 `files`），用于存放通用文件：

   ```bash
zfs create Seagate1t-zfs/files
   ```

   #### 2. **优化数据集配置**

   为文件存储的数据集设置合适的参数：

   ```bash
zfs set recordsize=16K Seagate1t-zfs/files
zfs set compression=lz4 Seagate1t-zfs/files
zfs set atime=off Seagate1t-zfs/files
   ```

   #### 3. **设置挂载点**

   为该数据集指定一个独立的挂载路径，避免混淆：

   ```bash
zfs set mountpoint=/mnt/Seagate1t-files Seagate1t-zfs/files
   ```
