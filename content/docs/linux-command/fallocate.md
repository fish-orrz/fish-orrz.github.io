---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'fallocate'
---

瞬間建立指定大小的檔案（預先配置空間，不實際寫入資料）。

## 指令總覽

```bash
fallocate -l 8G /data/bigfile
ls -lh /data/bigfile
df -h /data
rm /data/bigfile
```

## `fallocate -l`

建立 8GB 的檔案。

```bash
fallocate -l 8G /data/bigfile
```

`-l` 指定大小，可用 `M`、`G` 等單位。幾乎瞬間完成。

{{< callout type="info" >}}
路徑請放在空間足夠的分割區，避免放在 `/` 造成系統空間滿。若檔案系統不支援 `fallocate`，改用 [dd](../dd)。
{{< /callout >}}

## `ls -lh` / `df -h`

確認檔案大小與磁碟使用量。

```text
-rw-r--r--. 1 root root 8.0G Oct 15 09:00 /data/bigfile
```

## `rm`

測試完記得刪除，釋放空間。
