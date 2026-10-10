---
title: parted
weight: 1
---

磁碟分割工具,分割表統一使用 **GPT**。**指令立即生效**,操作前先用 `lsblk`、`blkid` 確認磁碟與檔案系統,並備份。

## 選哪種方式?

虛擬機磁碟 `/dev/sda` 已加大,要把空間分給 `/var`:

| 情況 | 方式 |
|---|---|
| LVM 分割(`sda3`)是**最後一個**分割 | 一、延伸分割 |
| LVM 分割**不是最後一個**,或不想動既有分割 | 二、新增分割加入 VG |

## 一、延伸最後一個分割(LVM,RHEL)

### 全部指令

```bash
# 1. 檢查現況
lsblk
blkid
df -hT /var
pvs; vgs; lvs

# 2. 延伸分割(只能延伸最後一個分割)
parted /dev/sda
print
resizepart 3 100%
quit
partprobe /dev/sda

# 3. 延伸 LVM
pvresize /dev/sda3
vgdisplay
lvextend -l +100%FREE /dev/mapper/rhel-var

# 4. 延伸檔案系統(二選一)
xfs_growfs /var                      # XFS
resize2fs /dev/mapper/rhel-var       # ext4

# 5. 確認
df -hT /var
```

{{< callout type="warning" >}}
`resizepart` 只能延伸**最後一個**分割。第 4 步依檔案系統選擇指令,選錯會失敗。
{{< /callout >}}

### 步驟說明

{{% steps %}}

### 檢查現況

**`lsblk`**:`sda` 已是 100G 但 `sda3` 仍是 59G,代表分割還沒延伸。

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

**`blkid`**:`TYPE` 決定第 4 步用哪個指令;`LVM2_member` 代表該分割是 PV。

```text
/dev/sda2: UUID="a1b2c3d4-..." TYPE="xfs" PARTUUID="..."
/dev/sda3: UUID="xYz123-..." TYPE="LVM2_member" PARTUUID="..."
/dev/mapper/rhel-var: UUID="e5f6a7b8-..." TYPE="xfs"
```

**`df -hT /var`**:`Type` 欄位直接顯示檔案系統。

```text
Filesystem           Type  Size  Used Avail Use% Mounted on
/dev/mapper/rhel-var xfs    10G  2.1G  7.9G  21% /var
```

**`pvs; vgs; lvs`**:LVM 三層總覽。`PFree` 為 0 表示 PV 沒有空間。

```text
  PV         VG   Fmt  Attr PSize   PFree
  /dev/sda3  rhel lvm2 a--  <59.00g    0

  VG   #PV #LV #SN Attr   VSize   VFree
  rhel   1   3   0 wz--n- <59.00g    0

  LV   VG   Attr       LSize
  root rhel -wi-ao----  40.00g
  swap rhel -wi-ao----   9.00g
  var  rhel -wi-ao----  10.00g
```

### 延伸分割

**`parted /dev/sda`** → **`print`**

```text
Warning: Not all of the space available to /dev/sda appears to be used, you can
fix the GPT to use all of the space (an extra 83886080 blocks) or continue with
the current setting?
Fix/Ignore? Fix

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  1076MB  1074MB  xfs
 3      1076MB  64.4GB  63.3GB                     lvm
```

出現 `Fix/Ignore?` 輸入 `Fix`。記下 LVM 分割編號(此例 `3`)。

**`resizepart 3 100%`** → **`quit`**

```text
Warning: Partition /dev/sda3 is being used. Are you sure you want to continue?
Yes/No? Yes
```

輸入 `Yes`,不會中斷服務。

**`partprobe /dev/sda`**:通知核心重讀分割表,成功時沒有輸出。

### 延伸 PV

**`pvresize /dev/sda3`**

```text
  Physical volume "/dev/sda3" changed
  1 physical volume(s) resized or updated / 0 physical volume(s) not resized
```

**`vgdisplay`**:看 `Free PE / Size` 有空間才能繼續。

```text
  VG Name               rhel
  VG Size               <99.00 GiB
  Free  PE / Size       10239 / <40.00 GiB
```

### 延伸 LV

**`lvextend -l +100%FREE /dev/mapper/rhel-var`**

```text
  Size of logical volume rhel/var changed from 10.00 GiB (2560 extents) to <50.00 GiB (12799 extents).
  Logical volume rhel/var successfully resized.
```

只配固定大小:**`lvextend -L +20G /dev/mapper/rhel-var`**

### 延伸檔案系統

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

**`df -hT /var`**

```text
Filesystem           Type  Size  Used Avail Use% Mounted on
/dev/mapper/rhel-var xfs    50G  2.1G   48G   5% /var
```

{{% /steps %}}

**一步完成**:**`lvextend -l +100%FREE -r /dev/mapper/rhel-var`**,`-r` 會自動依檔案系統呼叫 `xfs_growfs` 或 `resize2fs`。

## 二、新增分割加入 VG(LVM)

磁碟加大後,在磁碟尾端的**未配置空間**建立新分割 `sda4`,加入 VG 後再延伸 LV。不會動到既有分割。

### 全部指令

```bash
# 1. 檢查現況
lsblk
pvs; vgs

# 2. 在未配置空間建立新分割
parted /dev/sda
print free
mkpart lvm 64.4GB 100%
set 4 lvm on
print
quit
partprobe /dev/sda

# 3. 建立 PV 並加入 VG
pvcreate /dev/sda4
vgextend rhel /dev/sda4
vgs

# 4. 延伸 LV 與檔案系統(-r 自動處理 XFS/ext4)
lvextend -l +100%FREE -r /dev/mapper/rhel-var

# 5. 確認
lsblk
blkid /dev/sda4
df -hT /var
```

{{< callout type="info" >}}
`mkpart` 的起點(`64.4GB`)必須是 `print free` 顯示的 **Free Space 的 Start**。GPT 的 `lvm` 是**分割名稱**,不是檔案系統。
{{< /callout >}}

### 步驟說明

{{% steps %}}

### 檢查現況

**`lsblk`**:`sda` 100G,但分割只用到約 60G,後面還有空間。

```text
NAME           MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda              8:0    0  100G  0 disk
├─sda1           8:1    0    1M  0 part
├─sda2           8:2    0    1G  0 part /boot
└─sda3           8:3    0   59G  0 part
  └─rhel-var   253:2    0   10G  0 lvm  /var
```

### 建立新分割

**`parted /dev/sda`** → **`print free`**:找到最後的 `Free Space`。

```text
Number  Start   End     Size    File system  Name  Flags
        17.4kB  1049kB  1031kB  Free Space
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  1076MB  1074MB  xfs
 3      1076MB  64.4GB  63.3GB                     lvm
        64.4GB  107GB   42.9GB  Free Space
```

**`mkpart lvm 64.4GB 100%`**:從 `64.4GB` 到磁碟尾端,成功時沒有輸出。

**`set 4 lvm on`**:設定 LVM 旗標,**`print`** 確認:

```text
Number  Start   End     Size    File system  Name  Flags
 3      1076MB  64.4GB  63.3GB                     lvm
 4      64.4GB  107GB   42.9GB                     lvm
```

**`quit`** 離開後執行 **`partprobe /dev/sda`**。

### 建立 PV、加入 VG

**`pvcreate /dev/sda4`**

```text
  Physical volume "/dev/sda4" successfully created.
```

**`vgextend rhel /dev/sda4`**

```text
  Volume group "rhel" successfully extended
```

**`vgs`**:`#PV` 變 2,`VFree` 出現空間。

```text
  VG   #PV #LV #SN Attr   VSize   VFree
  rhel   2   3   0 wz--n- 101.99g <40.00g
```

### 延伸 LV 與檔案系統

**`lvextend -l +100%FREE -r /dev/mapper/rhel-var`**

```text
  Size of logical volume rhel/var changed from 10.00 GiB (2560 extents) to <50.00 GiB (12799 extents).
  Logical volume rhel/var successfully resized.
meta-data=/dev/mapper/rhel-var   isize=512    agcount=4, agsize=655360 blks
data blocks changed from 2621440 to 13106176
```

沒加 `-r` 時,再依檔案系統執行 **`xfs_growfs /var`** 或 **`resize2fs /dev/mapper/rhel-var`**。

### 確認

**`blkid /dev/sda4`**:`LVM2_member` 代表已是 PV。

```text
/dev/sda4: UUID="Pq9rSt-..." TYPE="LVM2_member" PARTLABEL="lvm" PARTUUID="..."
```

**`df -hT /var`**

```text
Filesystem           Type  Size  Used Avail Use% Mounted on
/dev/mapper/rhel-var xfs    50G  2.1G   48G   5% /var
```

{{% /steps %}}

## 三、新增磁碟(GPT + XFS/ext4)

### 全部指令

```bash
lsblk                                                        # 確認新磁碟名稱(例 sdb)
parted -s /dev/sdb mklabel gpt                               # 建立 GPT(會清空磁碟)
parted -s -a optimal /dev/sdb mkpart data 0% 100%            # 使用整顆磁碟
partprobe /dev/sdb

mkfs.xfs /dev/sdb1                                           # XFS
mkfs.ext4 /dev/sdb1                                          # ext4(二選一)

mkdir -p /data
mount /dev/sdb1 /data
blkid /dev/sdb1                                              # 取得 UUID
echo 'UUID=<uuid> /data xfs defaults,nofail 0 2' >> /etc/fstab   # ext4 把 xfs 改成 ext4
mount -a                                                     # 驗證 fstab
df -hT /data
```

- **`parted -s /dev/sdb mklabel gpt`**:`-s` 為非互動模式;超過 2TB 必須使用 GPT。
- **`parted -s -a optimal /dev/sdb mkpart data 0% 100%`**:`data` 是**分割名稱**;`-a optimal` 避免對齊警告。
- **`blkid /dev/sdb1`**:格式化後才有 UUID。

```text
/dev/sdb1: UUID="9c8d7e6f-1a2b-4c3d-8e9f-0a1b2c3d4e5f" TYPE="xfs" PARTLABEL="data" PARTUUID="..."
```

- **`mount -a`**:fstab 寫錯會在這裡報錯,**重開機前務必先執行**。

## 四、其他常用指令

```bash
parted -l                              # 列出所有磁碟
parted /dev/sdb unit GB print free     # 顯示未配置空間
parted /dev/sdb name 1 data            # 設定分割名稱
parted /dev/sdb set 1 lvm on           # 設定旗標
parted /dev/sdb rm 1                   # 刪除分割
echo 1 > /sys/class/block/sda/device/rescan    # 磁碟加大後,通知核心重新掃描
```

## 常見問題

| 狀況 | 處理 |
|---|---|
| `lsblk` 看不到磁碟變大 | 執行 `rescan` 指令 |
| `resizepart` 無法延伸 | 該分割不是最後一個,改用「二、新增分割」 |
| 延伸後 `df` 沒變大 | 漏了 `xfs_growfs` 或 `resize2fs`,或 `lvextend` 沒加 `-r` |
| `vgdisplay` 的 Free PE 為 0 | 確認 `pvresize` 或 `vgextend` 是否成功 |
| `xfs_growfs` 找不到裝置 | 參數要用**掛載點**,不是裝置路徑 |
| `mkpart` 起點算錯 | `rm 4` 刪除後,用 `print free` 重新確認 Start |
| 想縮小 XFS | 不支援,需備份後重建 |