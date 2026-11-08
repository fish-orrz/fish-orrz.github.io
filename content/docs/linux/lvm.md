---
date: '2026-11-08T17:04:06+08:00'
draft: false
title: 'Lvm'
---
# LVM (Logical Volume Manager) 完整操作指南

本指南包含使用 **`parted`** 與 **`fdisk`** 兩種磁碟分割工具建立與擴充 LVM 的標準流程、完整指令與終端機輸出結果範例。

---

## 一、 Quick Start 快速指令總覽

在詳細講解之前，可透過以下快速清單掌握兩種方式的全套執行指令。

### 方式 A：`parted` 完整流程 (適用 GPT / 大於 2TB 磁碟 / 推薦)

```bash
# 1. 重新掃描磁碟 (線上擴充/新增磁碟時)
echo 1 > /sys/class/block/sdb/device/rescan

# 2. 建立 GPT 分割區與 LVM 標記
parted -s /dev/sdb mklabel gpt
parted -s /dev/sdb mkpart primary 0% 100%
parted -s /dev/sdb set 1 lvm on

# 3. 建立 LVM 架構
pvcreate /dev/sdb1
vgcreate datavg /dev/sdb1
lvcreate -n datalv -L +50G datavg

# 4. 格式化與掛載
mkfs.xfs /dev/mapper/datavg-datalv
mkdir -p /data
echo "/dev/mapper/datavg-datalv /data xfs defaults 0 0" >> /etc/fstab
systemctl daemon-reload
mount -a
```

### 方式 B：`fdisk` 完整流程 (適用 MBR / 小於 2TB 磁碟)

```bash
# 1. 重新掃描磁碟
echo 1 > /sys/class/block/sdb/device/rescan

# 2. 建立 MBR 分割區並設定 LVM (8e)
fdisk /dev/sdb
# 互動介面輸入：n -> p -> 1 -> [Enter] -> [Enter] -> t -> 8e -> w

# 3. 建立 LVM 架構
pvcreate /dev/sdb1
vgcreate datavg /dev/sdb1
lvcreate -n datalv -L +50G datavg

# 4. 格式化與掛載
mkfs.xfs /dev/mapper/datavg-datalv
mkdir -p /data
echo "/dev/mapper/datavg-datalv /data xfs defaults 0 0" >> /etc/fstab
systemctl daemon-reload
mount -a
```

---

## 二、常用檢查與掃描指令

在進行磁碟變更前後，可使用以下指令確認狀態：

```bash
# 查看目前的掛載狀態與檔案系統容量 (顯示 HR 單位與 FS 類型)
df -TH

# 查看區塊裝置樹狀結構與檔案系統資訊
lsblk -f

# 查看區塊裝置 UUID 與格式
blkid

# 免重開機，觸發 Kernel 重新掃描指定磁碟 (將 sdb 替換為實際裝置名稱)
echo 1 > /sys/class/block/sdb/device/rescan
```

---

## 三、Parted 方式介紹 (優先推薦)

`parted` 支援 GPT 分割格式，適用於大於 2TB 的硬碟或自動化腳本。

### 1. 使用 `parted` 建立新分割區與 LVM

#### 互動模式指令操作

```bash
parted /dev/sdb
```

進入互動介面後的指令與預期輸出結果如下：

```text
GNU Parted 3.5
Using /dev/sdb
Welcome to GNU Parted! Type 'help' to view a list of commands.

(parted) mklabel gpt
# 設定磁碟標籤為 GPT

(parted) mkpart primary 0% 100%
# 建立一個名稱為 primary、使用 100% 空間的分割區

(parted) set 1 lvm on
# 將 1 號分割區設定為 LVM 屬性

(parted) print
# 查看目前分割區結果
Model: VMware Virtual disk (scsi)
Disk /dev/sdb: 53.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name     Flags
 1      1049kB  53.7GB  53.7GB               primary  lvm

(parted) quit
Information: You may need to update /etc/fstab.
```

#### 建置 LVM、格式化與掛載

```bash
# 1. 建立 Physical Volume 與 Volume Group
pvcreate /dev/sdb1
vgcreate datavg /dev/sdb1

# 2. 建立 Logical Volume (劃分 50G)
lvcreate -n datalv -L +50G datavg

# 3. 格式化為 XFS 檔案系統
mkfs.xfs /dev/mapper/datavg-datalv

# 4. 掛載與設定自動掛載
mkdir -p /data
echo "/dev/mapper/datavg-datalv /data xfs defaults 0 0" >> /etc/fstab
systemctl daemon-reload
mount -a
```

---

### 2. 使用 `parted` 延伸擴充空間 (Resize / Extend)

假設 `/dev/sdb` 原為 50G，在底層/虛擬機擴充至 100G 後，需放大原本的分割區與 LVM。

#### Step 1. 使用 `parted` 調整分割區大小

```bash
parted /dev/sdb
```

```text
(parted) print
Model: VMware Virtual disk (scsi)
Disk /dev/sdb: 107.4GB               ← 注意：磁碟總容量已變大為 100G
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name     Flags
 1      1049kB  53.7GB  53.7GB               primary  lvm   ← 目前 1 號分割區仍為 50G

(parted) resizepart 1 100%
# 將 1 號分割區延伸至硬碟 100% 的空間

(parted) print
Model: VMware Virtual disk (scsi)
Disk /dev/sdb: 107.4GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size     File system  Name     Flags
 1      1049kB  107.4GB 107.4GB              primary  lvm   ← 分割區成功放大至 100G

(parted) quit
```

#### Step 2. 更新 LVM 與檔案系統容量

```bash
# 1. 重新掃描並放大 Physical Volume
pvresize /dev/sdb1

# 2. 擴充 Logical Volume (選擇其中一種方式)
# 方式 A：分配剩餘的全部空間
lvextend -l +100%FREE /dev/mapper/datavg-datalv

# 方式 B：指定增加 50G 容量
# lvresize -L +50G /dev/mapper/datavg-datalv

# 3. 擴充檔案系統容量 (根據實際使用的格式擇一執行)
# 【XFS 檔案系統】
xfs_growfs /dev/mapper/datavg-datalv

# 【EXT4 / EXT3 / EXT2 檔案系統】
resize2fs /dev/mapper/datavg-datalv
```

---

## 四、fdisk 方式介紹

`fdisk` 主要用於 MBR 分割格式，適合小於 2TB 的硬碟或習慣傳統介面的操作者。

### 1. 使用 `fdisk` 建立新分割區與 LVM

#### 互動模式指令操作

```bash
fdisk /dev/sdb
```

進入互動介面後的操作與預期輸出結果如下：

```text
Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-104857599, default 2048): [直接按 Enter]
Last sector, +sectors or +size{K,M,G,T,P} (2048-104857599, default 104857599): [直接按 Enter]

Created a new partition 1 of type 'Linux' and of size 50 GiB.

Command (m for help): t
Selected partition 1
Hex code (type L to list all codes): 8e
Changed type of partition 'Linux' to 'Linux LVM'.

Command (m for help): p
Disk /dev/sdb: 50 GiB, 53687091200 bytes, 104857600 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x12345678

Device     Boot Start       End   Sectors Size Id Type
/dev/sdb1        2048 104857599 104855552  50G 8e Linux LVM

Command (m for help): w
The partition table has been altered.
Syncing disks.
```

#### 建置 LVM、格式化與掛載

```bash
# 1. 建立 PV 與 VG
pvcreate /dev/sdb1
vgcreate datavg /dev/sdb1

# 2. 建立 LV
lvcreate -n datalv -L +50G datavg

# 3. 格式化與掛載
mkfs.xfs /dev/mapper/datavg-datalv
mkdir -p /data
echo "/dev/mapper/datavg-datalv /data xfs defaults 0 0" >> /etc/fstab
systemctl daemon-reload
mount -a
```

---

### 2. 使用 `fdisk` 延伸擴充空間 (Resize / Extend)

由於 MBR / `fdisk` 原生沒有像 `parted` 的 `resizepart` 指令，對於使用 `fdisk` 的硬碟延伸擴充，標準作法通常為：

#### 方案 A：刪除原本分割區並重新以相同起始 Sectors 建立大分割區

> ⚠️ **注意**：操作前請務必確認起始 Sector 必須與原本一致，且不能刪除 LVM 數據。

```bash
fdisk /dev/sdb
```

```text
Command (m for help): p
# 先上記錄 /dev/sdb1 的 Start Sector (例如：2048)

Command (m for help): d
Selected partition 1
Partition 1 has been deleted.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-209715199, default 2048): 2048   ← 確保與原本的 Start 一致！
Last sector, +sectors or +size{K,M,G,T,P} (2048-209715199, default 209715199): [按 Enter 拿滿空間]

Do you want to remove the signature? N   ← 極度重要！務必選擇 N (保留 LVM 標籤)

Command (m for help): t
Selected partition 1
Hex code (type L to list all codes): 8e

Command (m for help): w
```

#### 方案 B：直接新增第二個分割區 (`/dev/sdb2`) 並加入原 VG (較安全且推薦)

如果不想刪除重建 `/dev/sdb1`，可以使用 `fdisk` 把新增的空間建為 `/dev/sdb2`：

```bash
# 1. 建立 /dev/sdb2 分割區
fdisk /dev/sdb
# 輸入：n -> p -> 2 -> [Enter] -> [Enter] -> t -> 2 -> 8e -> w

# 2. 將新的分割區轉為 PV 並加入原本的 VG
pvcreate /dev/sdb2
vgextend datavg /dev/sdb2

# 3. 擴充 LV
lvextend -l +100%FREE /dev/mapper/datavg-datalv

# 4. 擴充檔案系統
xfs_growfs /dev/mapper/datavg-datalv
# 若為 ext4 請使用: resize2fs /dev/mapper/datavg-datalv
```