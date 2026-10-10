---
title: parted
weight: 1
---

磁碟分割工具,支援 GPT 與 MBR。**指令立即生效,沒有「寫入」步驟**,操作前務必確認磁碟名稱並備份。

## 情境:磁碟擴大後,延伸 LVM 空間(RHEL)

虛擬機的磁碟 `/dev/sda` 已在虛擬化平台加大,要把新增空間配給 `/var`。

### 步驟總覽

```bash
parted /dev/sda
print
resizepart 3 100%        # 只能延伸最後一個分割
quit

partprobe /dev/sda
pvresize /dev/sda3
vgdisplay
lvextend -l +100%FREE /dev/mapper/rhel-var
xfs_growfs /var          # XFS;ext4 改用 resize2fs /dev/mapper/rhel-var
```

{{< callout type="warning" >}}
`resizepart` 只能延伸**最後一個**分割。要延伸的分割後面還有其他分割時無法操作。
{{< /callout >}}

### 0. 確認磁碟已變大

若 `lsblk` 還是舊大小,先要求核心重新掃描:

```bash
echo 1 > /sys/class/block/sda/device/rescan
lsblk
```

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

`sda` 已是 100G,但 `sda3` 仍是 59G,表示分割尚未延伸。

### 1. 延伸分割

```bash
parted /dev/sda
```

```text
GNU Parted 3.5
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print
Warning: Not all of the space available to /dev/sda appears to be used, you can
fix the GPT to use all of the space (an extra 83886080 blocks) or continue with
the current setting?
Fix/Ignore? Fix
Model: VMware Virtual disk (scsi)
Disk /dev/sda: 107GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  1076MB  1074MB  xfs
 3      1076MB  64.4GB  63.3GB                     lvm
```

- 出現 `Fix/Ignore?` 時輸入 `Fix`(GPT 備份表移到新磁碟尾端)。MBR 不會有此提示。
- 記下 LVM 分割的編號(此例為 `3`)。

```text
(parted) resizepart 3 100%
Warning: Partition /dev/sda3 is being used. Are you sure you want to continue?
Yes/No? Yes
(parted) print
...
Number  Start   End     Size    File system  Name  Flags
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  1076MB  1074MB  xfs
 3      1076MB  107GB   106GB                      lvm

(parted) quit
```

分割正在使用中的警告可放心輸入 `Yes`,延伸不會中斷服務。

### 2. 通知核心並延伸 PV

```bash
partprobe /dev/sda
pvresize /dev/sda3
```

`partprobe` 成功時沒有輸出。

```text
  Physical volume "/dev/sda3" changed
  1 physical volume(s) resized or updated / 0 physical volume(s) not resized
```

### 3. 確認 VG 有可用空間

```bash
vgdisplay
```

```text
  --- Volume group ---
  VG Name               rhel
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  6
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                3
  Open LV               3
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <99.00 GiB
  PE Size               4.00 MiB
  Total PE              25343
  Alloc PE / Size       15104 / 59.00 GiB
  Free  PE / Size       10239 / <40.00 GiB
  VG UUID               xxxxxx-xxxx-xxxx-xxxx-xxxx-xxxx-xxxxxx
```

看 `Free PE / Size`,有剩餘空間才能繼續。要看每個 LV 名稱與大小用 `lvs`。

### 4. 延伸 LV

```bash
lvextend -l +100%FREE /dev/mapper/rhel-var
```

```text
  Size of logical volume rhel/var changed from 10.00 GiB (2560 extents) to <50.00 GiB (12799 extents).
  Logical volume rhel/var successfully resized.
```

只配固定大小:`lvextend -L +20G /dev/mapper/rhel-var`。

### 5. 延伸檔案系統(不做的話 `df` 不會變大)

```bash
xfs_growfs /var
```

```text
meta-data=/dev/mapper/rhel-var   isize=512    agcount=4, agsize=655360 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=2621440, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 2621440 to 13106176
```

```bash
df -h /var
```

```text
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/rhel-var   50G  2.1G   48G   5% /var
```

**一步完成的寫法**:`lvextend -l +100%FREE -r /dev/mapper/rhel-var`,`-r` 會自動依檔案系統類型擴充。

**替代 parted 的工具**:`growpart /dev/sda 3`(套件 `cloud-utils-growpart`)可免互動延伸分割。

## 安裝

```bash
sudo apt install parted     # Debian/Ubuntu
sudo dnf install parted     # RHEL/Rocky
```

## 查看

```bash
lsblk -f                                  # 磁碟、分割、檔案系統總覽
sudo parted -l                            # 列出所有磁碟
sudo parted /dev/sdb print                # 單一磁碟
sudo parted /dev/sdb unit GB print free   # 含未配置空間
```

## 新增磁碟並分割(非互動)

```bash
sudo parted -s /dev/sdb mklabel gpt                           # 建立 GPT 分割表(會清空)
sudo parted -s -a optimal /dev/sdb mkpart data xfs 0% 100%    # 使用整顆磁碟
sudo mkfs.xfs /dev/sdb1                                       # 格式化(ext4 用 mkfs.ext4)
sudo mkdir -p /data
sudo mount /dev/sdb1 /data
sudo blkid /dev/sdb1                                          # 取得 UUID
```

- 超過 2TB 必須用 `gpt`。
- GPT 的 `mkpart` 第一個參數是**分割名稱**,不是類型。

開機自動掛載,編輯 `/etc/fstab`:

```text
UUID=<uuid> /data xfs defaults,nofail 0 2
```

```bash
sudo mount -a        # 驗證 fstab 無誤
```

## 其他操作

```bash
sudo parted /dev/sdb name 1 data          # 設定分割名稱
sudo parted /dev/sdb set 1 lvm on         # 設定旗標(lvm、boot、esp)
sudo parted /dev/sdb rm 1                 # 刪除分割
sudo partprobe /dev/sdb                   # 通知核心重新讀取分割表
```

## 常見問題

| 狀況 | 處理 |
|---|---|
| `not properly aligned` 警告 | 加 `-a optimal`,起點用 `0%` 或 `1MiB` |
| 新分割在 `lsblk` 看不到 | `sudo partprobe /dev/sdb` |
| `lsblk` 看不到磁碟變大 | `echo 1 > /sys/class/block/sda/device/rescan` |
| 延伸後 `df` 空間沒增加 | 漏了 `xfs_growfs` 或 `resize2fs` |
| `vgdisplay` 的 Free PE 為 0 | 先確認 `pvresize` 有成功 |
| 想縮小 | XFS 不支援縮小,需備份後重建 |