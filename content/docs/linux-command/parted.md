---
title: parted
weight: 1
---

磁碟分割工具,分割表使用 **GPT**。**指令立即生效**,操作前請先備份或建立快照,並用 `lsblk` 確認磁碟名稱。

環境:RHEL,磁碟 `/dev/sda` 已在虛擬化平台加大到 100G,要把空間分給 `/var`(LV:`rhel-var`)。

## 情境一:延伸最後一個分割

LVM 分割 `sda3` 是磁碟上**最後一個**分割時使用。

{{< callout type="warning" >}}
`resizepart` 只能延伸**最後一個**分割。檔案系統要依 `df -hT` 顯示的類型,選擇 XFS 或 ext4 其中一個指令。
{{< /callout >}}

### 全部指令

```bash
lsblk
blkid
df -hT /var
pvs
vgs
lvs

parted /dev/sda
print
resizepart 3 100%
quit
partprobe /dev/sda

pvresize /dev/sda3
vgdisplay
lvextend -l +100%FREE /dev/mapper/rhel-var

xfs_growfs /var                      # XFS
resize2fs /dev/mapper/rhel-var       # ext4

df -hT /var
```

### 步驟

{{% steps %}}

### 檢查現況

**`lsblk`**:`sda` 是 100G,但 `sda3` 仍是 59G,代表分割還沒延伸。

```text
NAME           MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda              8:0    0  100G  0 disk
├─sda1           8:1    0    1M  0 part
├─sda2           8:2    0    1G  0 part /boot
└─sda3           8:3    0   59G  0 part
  ├─rhel-root  253:0    0   40G  0 lvm  /
  ├─rhel-swap  253:1    0    9G  0 lvm  [SWAP]
  └─rhel-var   253:2    0   10G  0 lvm  /var
```

**`blkid`**:`LVM2_member` 代表該分割是 LVM 的 PV。

```text
/dev/sda2: UUID="a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d" TYPE="xfs" PARTUUID="1f2e3d4c-02"
/dev/sda3: UUID="xYz123-AbCd-EfGh-IjKl-MnOp-QrSt-UvWxYz" TYPE="LVM2_member" PARTUUID="1f2e3d4c-03"
/dev/mapper/rhel-var: UUID="e5f6a7b8-1234-4abc-9def-0123456789ab" TYPE="xfs"
```

**`df -hT /var`**:`Type` 欄位就是檔案系統類型(`xfs` 或 `ext4`),第 5 步要用。

```text
Filesystem           Type  Size  Used Avail Use% Mounted on
/dev/mapper/rhel-var xfs    10G  2.1G  7.9G  21% /var
```

**`pvs`**:`PFree` 為 0 代表 PV 沒有剩餘空間。

```text
  PV         VG   Fmt  Attr PSize   PFree
  /dev/sda3  rhel lvm2 a--  <59.00g    0
```

**`vgs`**

```text
  VG   #PV #LV #SN Attr   VSize   VFree
  rhel   1   3   0 wz--n- <59.00g    0
```

**`lvs`**

```text
  LV   VG   Attr       LSize
  root rhel -wi-ao----  40.00g
  swap rhel -wi-ao----   9.00g
  var  rhel -wi-ao----  10.00g
```

### 延伸分割

**`parted /dev/sda`**:進入互動模式。

```text
GNU Parted 3.5
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
```

**`print`**:出現 `Fix/Ignore?` 時輸入 `Fix`。記下 LVM 分割編號(此例 `3`)。

```text
Warning: Not all of the space available to /dev/sda appears to be used, you can
fix the GPT to use all of the space (an extra 83886080 blocks) or continue with
the current setting?
Fix/Ignore? Fix

Model: VMware Virtual disk (scsi)
Disk /dev/sda: 107GB
Partition Table: gpt

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  1076MB  1074MB  xfs
 3      1076MB  64.4GB  63.3GB                     lvm
```

**`resizepart 3 100%`**:出現確認時輸入 `Yes`,不會中斷服務。

```text
Warning: Partition /dev/sda3 is being used. Are you sure you want to continue?
Yes/No? Yes
```

**`quit`**:離開 parted。

```text
Information: You may need to update /etc/fstab.
```

**`partprobe /dev/sda`**:通知核心重讀分割表,成功時**沒有輸出**。

### 延伸 PV

**`pvresize /dev/sda3`**

```text
  Physical volume "/dev/sda3" changed
  1 physical volume(s) resized or updated / 0 physical volume(s) not resized
```

**`vgdisplay`**:`Free PE / Size` 有空間才能繼續。

```text
  --- Volume group ---
  VG Name               rhel
  VG Size               <99.00 GiB
  PE Size               4.00 MiB
  Total PE              25343
  Alloc PE / Size       15104 / 59.00 GiB
  Free  PE / Size       10239 / <40.00 GiB
```

### 延伸 LV

**`lvextend -l +100%FREE /dev/mapper/rhel-var`**:把 VG 全部剩餘空間給 `rhel-var`。

```text
  Size of logical volume rhel/var changed from 10.00 GiB (2560 extents) to <50.00 GiB (12799 extents).
  Logical volume rhel/var successfully resized.
```

### 延伸檔案系統

依 `df -hT` 的類型**二選一**,不要兩個都執行。

**XFS:`xfs_growfs /var`**(參數是**掛載點**)

```text
meta-data=/dev/mapper/rhel-var   isize=512    agcount=4, agsize=655360 blks
data     =                       bsize=4096   blocks=2621440, imaxpct=25
data blocks changed from 2621440 to 13106176
```

**ext4:`resize2fs /dev/mapper/rhel-var`**(參數是 **LV 路徑**)

```text
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mapper/rhel-var is mounted on /var; on-line resizing required
The filesystem on /dev/mapper/rhel-var is now 13106176 (4k) blocks long.
```

### 確認

**`df -hT /var`**:`Size` 變大即完成。

```text
Filesystem           Type  Size  Used Avail Use% Mounted on
/dev/mapper/rhel-var xfs    50G  2.1G   48G   5% /var
```

{{% /steps %}}

## 情境二:新增分割並加入 VG

在磁碟尾端的**未配置空間**建立新分割 `sda4`,加入 VG 後延伸 LV。不會更動既有分割。

{{< callout type="warning" >}}
`mkpart` 的起點必須使用 `print free` 顯示的**最後一個 Free Space 的 Start**,不要自己推算。檔案系統要依 `df -hT` 顯示的類型,選擇 XFS 或 ext4 其中一個指令。
{{< /callout >}}

### 全部指令

```bash
lsblk
blkid
df -hT /var
pvs
vgs
lvs

parted /dev/sda
print free
mkpart lvm 64.4GB 100%
set 4 lvm on
print
quit
partprobe /dev/sda

pvcreate /dev/sda4
vgextend rhel /dev/sda4
vgs
lvextend -l +100%FREE /dev/mapper/rhel-var

xfs_growfs /var                      # XFS
resize2fs /dev/mapper/rhel-var       # ext4

lsblk
blkid /dev/sda4
df -hT /var
```

### 步驟

{{% steps %}}

### 檢查現況

**`lsblk`**:`sda` 是 100G,但分割只用到約 60G,後面有未配置空間。

```text
NAME           MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda              8:0    0  100G  0 disk
├─sda1           8:1    0    1M  0 part
├─sda2           8:2    0    1G  0 part /boot
└─sda3           8:3    0   59G  0 part
  ├─rhel-root  253:0    0   40G  0 lvm  /
  ├─rhel-swap  253:1    0    9G  0 lvm  [SWAP]
  └─rhel-var   253:2    0   10G  0 lvm  /var
```

**`blkid`**

```text
/dev/sda2: UUID="a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d" TYPE="xfs" PARTUUID="1f2e3d4c-02"
/dev/sda3: UUID="xYz123-AbCd-EfGh-IjKl-MnOp-QrSt-UvWxYz" TYPE="LVM2_member" PARTUUID="1f2e3d4c-03"
/dev/mapper/rhel-var: UUID="e5f6a7b8-1234-4abc-9def-0123456789ab" TYPE="xfs"
```

**`df -hT /var`**:記下檔案系統類型(`xfs` 或 `ext4`),第 5 步要用。

```text
Filesystem           Type  Size  Used Avail Use% Mounted on
/dev/mapper/rhel-var xfs    10G  2.1G  7.9G  21% /var
```

**`pvs`**

```text
  PV         VG   Fmt  Attr PSize   PFree
  /dev/sda3  rhel lvm2 a--  <59.00g    0
```

**`vgs`**

```text
  VG   #PV #LV #SN Attr   VSize   VFree
  rhel   1   3   0 wz--n- <59.00g    0
```

**`lvs`**

```text
  LV   VG   Attr       LSize
  root rhel -wi-ao----  40.00g
  swap rhel -wi-ao----   9.00g
  var  rhel -wi-ao----  10.00g
```

### 建立新分割

**`parted /dev/sda`**:進入互動模式。

```text
GNU Parted 3.5
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
```

**`print free`**:找最後一個 `Free Space`,記下它的 **Start**(此例 `64.4GB`)。出現 `Fix/Ignore?` 時輸入 `Fix`。

```text
Number  Start   End     Size    File system  Name  Flags
        17.4kB  1049kB  1031kB  Free Space
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  1076MB  1074MB  xfs
 3      1076MB  64.4GB  63.3GB                     lvm
        64.4GB  107GB   42.9GB  Free Space
```

**`mkpart lvm 64.4GB 100%`**:從 Free Space 的 Start 到磁碟尾端。`lvm` 是 GPT 的**分割名稱**,成功時**沒有輸出**。

**`set 4 lvm on`**:設定 LVM 旗標,成功時**沒有輸出**。

**`print`**:確認出現第 4 個分割。

```text
Number  Start   End     Size    File system  Name  Flags
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  1076MB  1074MB  xfs
 3      1076MB  64.4GB  63.3GB                     lvm
 4      64.4GB  107GB   42.9GB                     lvm
```

**`quit`**:離開 parted。

```text
Information: You may need to update /etc/fstab.
```

**`partprobe /dev/sda`**:通知核心重讀分割表,成功時**沒有輸出**。

### 建立 PV 並加入 VG

**`pvcreate /dev/sda4`**

```text
  Physical volume "/dev/sda4" successfully created.
```

**`vgextend rhel /dev/sda4`**

```text
  Volume group "rhel" successfully extended
```

**`vgs`**:`#PV` 變成 2,`VFree` 出現可用空間。

```text
  VG   #PV #LV #SN Attr   VSize    VFree
  rhel   2   3   0 wz--n- <101.99g <42.00g
```

### 延伸 LV

**`lvextend -l +100%FREE /dev/mapper/rhel-var`**

```text
  Size of logical volume rhel/var changed from 10.00 GiB (2560 extents) to <52.00 GiB (13311 extents).
  Logical volume rhel/var successfully resized.
```

### 延伸檔案系統

依 `df -hT` 的類型**二選一**,不要兩個都執行。

**XFS:`xfs_growfs /var`**(參數是**掛載點**)

```text
meta-data=/dev/mapper/rhel-var   isize=512    agcount=4, agsize=655360 blks
data     =                       bsize=4096   blocks=2621440, imaxpct=25
data blocks changed from 2621440 to 13630464
```

**ext4:`resize2fs /dev/mapper/rhel-var`**(參數是 **LV 路徑**)

```text
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mapper/rhel-var is mounted on /var; on-line resizing required
The filesystem on /dev/mapper/rhel-var is now 13630464 (4k) blocks long.
```

### 確認

**`lsblk`**:`sda4` 出現,且 `rhel-var` 同時使用 `sda3`、`sda4`。

```text
NAME           MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda              8:0    0  100G  0 disk
├─sda1           8:1    0    1M  0 part
├─sda2           8:2    0    1G  0 part /boot
├─sda3           8:3    0   59G  0 part
│ ├─rhel-root  253:0    0   40G  0 lvm  /
│ ├─rhel-swap  253:1    0    9G  0 lvm  [SWAP]
│ └─rhel-var   253:2    0   52G  0 lvm  /var
└─sda4           8:4    0   40G  0 part
  └─rhel-var   253:2    0   52G  0 lvm  /var
```

**`blkid /dev/sda4`**:`LVM2_member` 代表已是 PV。

```text
/dev/sda4: UUID="Pq9rSt-UvWx-YzAb-CdEf-GhIj-KlMn-OpQrSt" TYPE="LVM2_member" PARTLABEL="lvm" PARTUUID="7a8b9c0d-4e5f-4a1b-9c2d-3e4f5a6b7c8d"
```

**`df -hT /var`**:`Size` 變大即完成。

```text
Filesystem           Type  Size  Used Avail Use% Mounted on
/dev/mapper/rhel-var xfs    52G  2.1G   50G   4% /var
```

{{% /steps %}}