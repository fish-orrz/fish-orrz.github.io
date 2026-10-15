---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'tar'
---

壓縮與解壓縮。

## 指令總覽

```bash
tar -czvf backup.tar.gz /path/to/dir
tar -tvf backup.tar.gz
mkdir -p folder
tar -xvf backup.tar.gz -C folder
```

## `tar -czvf`

把目錄壓縮成 `.tar.gz`。

```bash
tar -czvf backup.tar.gz /path/to/dir
```

| 參數 | 說明 |
|---|---|
| `c` | 建立壓縮檔 |
| `z` | 使用 gzip |
| `v` | 顯示過程 |
| `f` | 指定檔名（`f` 後面要緊接檔名） |

## `tar -tvf`

只列出內容，不解壓。解壓前先確認內容與路徑。

```text
drwxr-xr-x root/root         0 2026-10-15 09:00 dir/
-rw-r--r-- root/root      1024 2026-10-15 09:00 dir/a.txt
```

## `tar -xvf -C`

解壓到指定目錄。

```bash
mkdir -p folder
tar -xvf backup.tar.gz -C folder
```

| 參數 | 說明 |
|---|---|
| `x` | 解壓 |
| `-C` | 解壓到指定目錄，該目錄必須先存在 |

解壓時 tar 會自動判斷壓縮格式（gz / xz / bz2），不用再加 `z`。
