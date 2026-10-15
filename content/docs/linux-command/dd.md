---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'dd'
---

複製與寫入資料，常用來建立指定大小的測試檔。

## 指令總覽

```bash
dd if=/dev/zero of=/data/bigfile bs=1M count=4 status=progress
dd if=/dev/zero of=/data/bigfile bs=1M count=4096 status=progress
ls -lh /data/bigfile
```

## `dd if=/dev/zero of=... bs=1M count=...`

| 參數 | 說明 |
|---|---|
| `if` | 來源，`/dev/zero` 會產生全 0 的資料 |
| `of` | 輸出目標 |
| `bs` | 每次寫入的區塊大小 |
| `count` | 寫入幾個區塊 |
| `status=progress` | 顯示進度 |

檔案大小 = `bs` × `count`：

```bash
# 4MB
dd if=/dev/zero of=/data/bigfile bs=1M count=4 status=progress

# 4GB
dd if=/dev/zero of=/data/bigfile bs=1M count=4096 status=progress
```

```text
4+0 records in
4+0 records out
4194304 bytes (4.2 MB, 4.0 MiB) copied, 0.00386 s, 1.1 GB/s
```

{{< callout type="warning" >}}
`of=` 若誤寫成磁碟（例如 `/dev/sda`）會直接覆蓋資料，執行前請再確認一次。
{{< /callout >}}

## `ls -lh`

確認檔案大小。與 [fallocate](../fallocate) 不同，`dd` 會真的寫入資料，檔案越大花越久。
