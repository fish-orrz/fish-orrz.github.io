---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'rsync'
---

檔案同步，只傳送差異的部分。預設透過 SSH（port 22）；`873` 是 rsync daemon 模式才會用到的 port。

## 指令總覽

```bash
dnf install -y rsync
rsync -avhn /data/src/ user@dest_host:/data/dest/
rsync -avhi --log-file=/tmp/rsync.log /data/src/ user@dest_host:/data/dest/
rsync -avh -e "ssh -p 2222" /data/src/ user@dest_host:/data/dest/
```

## `dnf install -y rsync`

來源端與目的端都必須安裝。

## `rsync -avhn`

先演練，不會實際傳送，確認會同步哪些檔案。

```bash
rsync -avhn /data/src/ user@dest_host:/data/dest/
```

```text
sending incremental file list
./
a.txt
```

| 參數 | 說明 |
|---|---|
| `-a` | 封存模式（保留權限、擁有者、時間、子目錄） |
| `-v` | 顯示過程 |
| `-h` | 大小用易讀格式 |
| `-n` | 演練，不實際傳送 |
| `-i` | 列出每個檔案的變更項目 |

## `rsync -avhi --log-file`

確認無誤後正式同步，並寫入 log。

```bash
rsync -avhi --log-file=/tmp/rsync.log /data/src/ user@dest_host:/data/dest/
```

## `rsync -e "ssh -p 2222"`

SSH 不是 22 port 時指定連線方式。

{{< callout type="warning" >}}
來源路徑結尾的斜線有差異：`/data/src/` 複製目錄「內的內容」；`/data/src` 會連同 `src` 目錄本身一起複製。

`--delete` 會刪除目的端多出來的檔案，務必先用 `-n` 確認。
{{< /callout >}}
