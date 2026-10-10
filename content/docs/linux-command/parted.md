---
title: parted
weight: 1
---

磁碟分割工具,分割表使用 **GPT**。**指令立即生效**,操作前請先備份或建立快照,並用 `lsblk` 確認磁碟名稱。

環境:RHEL,磁碟 `/dev/sda` 已在虛擬化平台加大到 100G,要把空間分給 `/var`(LV:`rhel-var`)。

## 先了解:空間怎麼層層延伸

LVM 的空間由下往上分層,**每一層都要依序延伸**,少一層上面就看不到新空間:

```text
磁碟 sda  →  分割 sda3  →  PV  →  VG (rhel)  →  LV (rhel-var)  →  檔案系統 (/var)
(已加大)     parted        pvresize   自動變大     lvextend         xfs_growfs / resize2fs
```

## 情境一:延伸最後一個分割

適用:LVM 的分割 `sda3` 是磁碟上**最後一個**分割。

{{< callout type="warning" >}}
`resizepart` 只能延伸**最後一個**分割。檔案系統依 `df -hT` 顯示的類型,**XFS 與 ext4 二選一**,不要兩個都執行。
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

{{% steps %}}

### `lsblk`

**作用**:列出所有磁碟與分割的大小、類型、掛載位置。

**範例輸出**:

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

**怎麼看**:`sda` 已是 100G,但 `sda3` 仍是 59G,代表新增的空間還沒分配給分割。

### `blkid`

**作用**:顯示每個分割或 LV 的檔案系統類型與 UUID。

**範例輸出**:

```text
/dev/sda2: UUID="a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d" TYPE="xfs" PARTUUID="1f2e3d4c-02"
/dev/sda3: UUID="xYz123-AbCd-EfGh-IjKl-MnOp-QrSt-UvWxYz" TYPE="LVM2_member" PARTUUID="1f2e3d4c-03"
/dev/mapper/rhel-var: UUID="e5f6a7b8-1234-4abc-9def-0123456789ab" TYPE="xfs"
```

**怎麼看**:`LVM2_member` 代表 `sda3` 是 LVM 的 PV;`TYPE` 是 `xfs` 或 `ext4`,決定最後一步用哪個指令。

### `df -hT /var`

**作用**:顯示 `/var` 所在檔案系統的類型與使用量。

**範例輸出**:

```text
Filesystem           Type  Size  Used Avail Use% Mounted on
/dev/mapper/rhel-var xfs    10G  2.1G  7.9G  21% /var
```

**怎麼看**:`Type` 欄就是檔案系統類型,記下來(此例 `xfs`)。

### `pvs`

**作用**:列出 LVM 的 PV(實體磁區)與剩餘空間。

**範例輸出**:

```text
  PV         VG   Fmt  Attr PSize   PFree
  /dev/sda3  rhel lvm2 a--  <59.00g    0
```

**怎麼看**:`PFree` 為 0,代表這個 PV 沒有可用空間。

### `vgs`

**作用**:列出 VG(磁區群組)的總容量與剩餘空間。

**範例輸出**:

```text
  VG   #PV #LV #SN Attr   VSize   VFree
  rhel   1   3   0 wz--n- <59.00g    0
```

### `lvs`

**作用**:列出 LV(邏輯磁區)及大小。

**範例輸出**:

```text
  LV   VG   Attr       LSize
  root rhel -wi-ao----  40.00g
  swap rhel -wi-ao----   9.00g
  var  rhel -wi-ao----  10.00g
```

### `parted /dev/sda`

**作用**:進入 `sda` 的互動式分割模式,之後的指令都在 `(parted)` 提示下輸入。

**範例輸出**:

```text
GNU Parted 3.5
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
```

### `print`

**作用**:顯示 `sda` 的分割表。

**範例輸出**:

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

**注意**:出現 `Fix/Ignore?` 時輸入 `Fix`(讓 GPT 使用整顆新磁碟)。記下 LVM 分割的編號(此例 `3`)。

### `resizepart 3 100%`

**作用**:把第 3 個分割延伸到磁碟尾端(100%)。

**範例輸出**:

```text
Warning: Partition /dev/sda3 is being used. Are you sure you want to continue?
Yes/No? Yes
```

**注意**:輸入 `Yes`。分割使用中是正常的,延伸不會中斷服務。

### `quit`

**作用**:離開 parted。

**範例輸出**:

```text
Information: You may need to update /etc/fstab.
```

**注意**:這行訊息可忽略,LVM 掛載不需要改 fstab。

### `partprobe /dev/sda`

**作用**:通知核心重新讀取 `sda` 的分割表,讓系統認得新大小。

**範例輸出**:成功時**沒有輸出**。可用 `lsblk` 確認 `sda3` 已變大。

### `pvresize /dev/sda3`

**作用**:讓 LVM 的 PV 跟著分割變大。

**範例輸出**:

```text
  Physical volume "/dev/sda3" changed
  1 physical volume(s) resized or updated / 0 physical volume(s) not resized
```

### `vgdisplay`

**作用**:顯示 VG 的詳細資訊,確認有可用空間。

**範例輸出**:

```text
  --- Volume group ---
  VG Name               rhel
  VG Size               <99.00 GiB
  PE Size               4.00 MiB
  Total PE              25343
  Alloc PE / Size       15104 / 59.00 GiB
  Free  PE / Size       10239 / <40.00 GiB
```

**怎麼看**:`Free PE / Size` 有數值(此例約 40GiB)才能繼續,為 0 代表上一步沒成功。

### `lvextend -l +100%FREE /dev/mapper/rhel-var`

**作用**:把 VG 全部的剩餘空間加給 `rhel-var` 這個 LV。

**範例輸出**:

```text
  Size of logical volume rhel/var changed from 10.00 GiB (2560 extents) to <50.00 GiB (12799 extents).
  Logical volume rhel/var successfully resized.
```

### `xfs_growfs /var`(XFS)

**作用**:讓 XFS 檔案系統使用 LV 增加的空間。

**範例輸出**:

```text
meta-data=/dev/mapper/rhel-var   isize=512    agcount=4, agsize=655360 blks
data     =                       bsize=4096   blocks=2621440, imaxpct=25
data blocks changed from 2621440 to 13106176
```

**注意**:檔案系統是 **XFS** 才執行;參數是**掛載點**(`/var`)。ext4 請改用下一步。

### `resize2fs /dev/mapper/rhel-var`(ext4)

**作用**:讓 ext4 檔案系統使用 LV 增加的空間。

**範例輸出**:

```text
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mapper/rhel-var is mounted on /var; on-line resizing required
The filesystem on /dev/mapper/rhel-var is now 13106176 (4k) blocks long.
```

**注意**:檔案系統是 **ext4** 才執行;參數是 **LV 路徑**。XFS 請勿執行。

### `df -hT /var`(確認)

**作用**:再次確認 `/var` 的容量。

**範例輸出**:

```text
Filesystem           Type  Size  Used Avail Use% Mounted on
/dev/mapper/rhel-var xfs    50G  2.1G   48G   5% /var
```

**怎麼看**:`Size` 從 10G 變成 50G,即完成。

{{% /steps %}}

## 情境二:新增分割並加入 VG

適用:不想動既有分割,或 `sda3` 不是最後一個分割。在磁碟尾端的**未配置空間**建立新分割 `sda4`,加入 VG 後延伸 LV。

{{< callout type="warning" >}}
`mkpart` 的起點必須使用 `print free` 顯示的**最後一個 Free Space 的 Start**,不要自己推算。檔案系統依 `df -hT` 顯示的類型,**XFS 與 ext4 二選一**。
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

{{% steps %}}

### `lsblk`

**作用**:列出所有磁碟與分割的大小、類型、掛載位置。

**範例輸出**:

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

**怎麼看**:`sda` 是 100G,但分割只用到約 60G,後面是未配置空間。

### `blkid`

**作用**:顯示每個分割或 LV 的檔案系統類型與 UUID。

**範例輸出**:

```text
/dev/sda2: UUID="a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d" TYPE="xfs" PARTUUID="1f2e3d4c-02"
/dev/sda3: UUID="xYz123-AbCd-EfGh-IjKl-MnOp-QrSt-UvWxYz" TYPE="LVM2_member" PARTUUID="1f2e3d4c-03"
/dev/mapper/rhel-var: UUID="e5f6a7b8-1234-4abc-9def-0123456789ab" TYPE="xfs"
```

### `df -hT /var`

**作用**:顯示 `/var` 所在檔案系統的類型與使用量。

**範例輸出**:

```text
Filesystem           Type  Size  Used Avail Use% Mounted on
/dev/mapper/rhel-var xfs    10G  2.1G  7.9G  21% /var
```

**怎麼看**:記下 `Type`(`xfs` 或 `ext4`),決定後面用哪個指令。

### `pvs`

**作用**:列出 LVM 的 PV 與剩餘空間。

**範例輸出**:

```text
  PV         VG   Fmt  Attr PSize   PFree
  /dev/sda3  rhel lvm2 a--  <59.00g    0
```

### `vgs`

**作用**:列出 VG 的總容量與剩餘空間。

**範例輸出**:

```text
  VG   #PV #LV #SN Attr   VSize   VFree
  rhel   1   3   0 wz--n- <59.00g    0
```

### `lvs`

**作用**:列出 LV 及大小。

**範例輸出**:

```text
  LV   VG   Attr       LSize
  root rhel -wi-ao----  40.00g
  swap rhel -wi-ao----   9.00g
  var  rhel -wi-ao----  10.00g
```

### `parted /dev/sda`

**作用**:進入 `sda` 的互動式分割模式。

**範例輸出**:

```text
GNU Parted 3.5
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
```

### `print free`

**作用**:顯示分割表,**並列出未配置的空間**(Free Space)。

**範例輸出**:

```text
Number  Start   End     Size    File system  Name  Flags
        17.4kB  1049kB  1031kB  Free Space
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  1076MB  1074MB  xfs
 3      1076MB  64.4GB  63.3GB                     lvm
        64.4GB  107GB   42.9GB  Free Space
```

**注意**:找**最後一行** `Free Space`,記下它的 **Start**(此例 `64.4GB`)。出現 `Fix/Ignore?` 時輸入 `Fix`。

### `mkpart lvm 64.4GB 100%`

**作用**:建立新分割,範圍是從 `64.4GB`(Free Space 的 Start)到磁碟尾端。`lvm` 只是分割的名稱。

**範例輸出**:成功時**沒有輸出**。

### `set 4 lvm on`

**作用**:把第 4 個分割標記為 LVM 用途。

**範例輸出**:成功時**沒有輸出**。

### `print`

**作用**:確認新分割已建立。

**範例輸出**:

```text
Number  Start   End     Size    File system  Name  Flags
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  1076MB  1074MB  xfs
 3      1076MB  64.4GB  63.3GB                     lvm
 4      64.4GB  107GB   42.9GB                     lvm
```

**怎麼看**:出現第 4 個分割,大小約 42.9GB。

### `quit`

**作用**:離開 parted。

**範例輸出**:

```text
Information: You may need to update /etc/fstab.
```

**注意**:這行訊息可忽略。

### `partprobe /dev/sda`

**作用**:通知核心重新讀取分割表,讓系統認得 `sda4`。

**範例輸出**:成功時**沒有輸出**。

### `pvcreate /dev/sda4`

**作用**:把 `sda4` 初始化成 LVM 的 PV。

**範例輸出**:

```text
  Physical volume "/dev/sda4" successfully created.
```

### `vgextend rhel /dev/sda4`

**作用**:把 `sda4` 加進 `rhel` 這個 VG,VG 就多出新空間。

**範例輸出**:

```text
  Volume group "rhel" successfully extended
```

### `vgs`(確認)

**作用**:確認 VG 已有剩餘空間。

**範例輸出**:

```text
  VG   #PV #LV #SN Attr   VSize    VFree
  rhel   2   3   0 wz--n- <101.99g <42.00g
```

**怎麼看**:`#PV` 變成 2,`VFree` 約 42G,可以繼續。

### `lvextend -l +100%FREE /dev/mapper/rhel-var`

**作用**:把 VG 全部的剩餘空間加給 `rhel-var` 這個 LV。

**範例輸出**:

```text
  Size of logical volume rhel/var changed from 10.00 GiB (2560 extents) to <52.00 GiB (13311 extents).
  Logical volume rhel/var successfully resized.
```

### `xfs_growfs /var`(XFS)

**作用**:讓 XFS 檔案系統使用 LV 增加的空間。

**範例輸出**:

```text
meta-data=/dev/mapper/rhel-var   isize=512    agcount=4, agsize=655360 blks
data     =                       bsize=4096   blocks=2621440, imaxpct=25
data blocks changed from 2621440 to 13630464
```

**注意**:檔案系統是 **XFS** 才執行;參數是**掛載點**(`/var`)。ext4 請改用下一步。

### `resize2fs /dev/mapper/rhel-var`(ext4)

**作用**:讓 ext4 檔案系統使用 LV 增加的空間。

**範例輸出**:

```text
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mapper/rhel-var is mounted on /var; on-line resizing required
The filesystem on /dev/mapper/rhel-var is now 13630464 (4k) blocks long.
```

**注意**:檔案系統是 **ext4** 才執行;參數是 **LV 路徑**。XFS 請勿執行。

### `lsblk`(確認)

**作用**:確認新分割已加入 LV。

**範例輸出**:

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

**怎麼看**:`rhel-var` 同時出現在 `sda3` 和 `sda4` 底下,代表這個 LV 橫跨兩個分割,大小 52G。

### `blkid /dev/sda4`

**作用**:確認 `sda4` 的類型。

**範例輸出**:

```text
/dev/sda4: UUID="Pq9rSt-UvWx-YzAb-CdEf-GhIj-KlMn-OpQrSt" TYPE="LVM2_member" PARTLABEL="lvm" PARTUUID="7a8b9c0d-4e5f-4a1b-9c2d-3e4f5a6b7c8d"
```

**怎麼看**:`LVM2_member` 代表 `sda4` 已是 PV。

### `df -hT /var`(確認)

**作用**:再次確認 `/var` 的容量。

**範例輸出**:

```text
Filesystem           Type  Size  Used Avail Use% Mounted on
/dev/mapper/rhel-var xfs    52G  2.1G   50G   4% /var
```

**怎麼看**:`Size` 從 10G 變成 52G,即完成。

{{% /steps %}}