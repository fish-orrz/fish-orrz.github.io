---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'locate'
---

從資料庫快速搜尋檔案，比 `find` 快很多。

## 指令總覽

```bash
dnf install -y mlocate
updatedb
locate file.txt
locate -i readme
locate -c "*.conf"
locate -l 10 "*.log"
```

## `dnf install -y mlocate`

RHEL 8 / 9 使用 `mlocate`。較新的系統若找不到，改裝 `plocate`，用法相同。

## `updatedb`

建立與更新資料庫，第一次使用必須先執行，否則查不到東西。

## `locate`

搜尋路徑中包含關鍵字的檔案。

```bash
locate file.txt
```

```text
/home/user1/file.txt
/tmp/backup/file.txt.bak
```

常用選項：

| 指令 | 說明 |
|---|---|
| `locate -i readme` | 不分大小寫 |
| `locate -c "*.conf"` | 只顯示符合的數量 |
| `locate -l 10 "*.log"` | 只顯示前 10 筆 |

{{< callout type="info" >}}
`locate` 查的是資料庫：新建的檔案要先 `updatedb` 才找得到，已刪除的檔案在更新前仍會被列出。mlocate 預設每天自動執行 `updatedb`。

含萬用字元（`*`）的關鍵字要加引號；排除的目錄（如 `/tmp`、網路檔案系統）設定在 `/etc/updatedb.conf`。
{{< /callout >}}
