---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'lsof'
---

列出系統中被開啟的檔案（包含 socket、目錄、裝置），可查「誰在使用這個檔案或 port」。

## 指令總覽

```bash
dnf install -y lsof
lsof -p <PID>
lsof -u account
lsof /var/log/messages
lsof -nP -i tcp
lsof -nP -i :22
lsof -nP -iTCP -sTCP:LISTEN
```

## `lsof -p` / `lsof -u`

```bash
# 指定 PID 使用的所有檔案
lsof -p <PID>

# 指定帳號開啟的所有檔案
lsof -u account
```

## `lsof /path/file`

查看誰正在使用這個檔案，例如無法卸載、無法刪除時用來找出原因。

## `lsof -nP -i`

查看網路連線。

```bash
# 所有 TCP 連線
lsof -nP -i tcp

# 誰佔用 22 port
lsof -nP -i :22
```

```text
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd    1021 root    3u  IPv4  23456      0t0  TCP *:22 (LISTEN)
```

| 參數 | 說明 |
|---|---|
| `-n` | 不解析主機名稱（較快） |
| `-P` | 不把 port 轉成服務名稱（顯示 22 而非 ssh） |
| `-i` | 網路連線 |

## `lsof -nP -iTCP -sTCP:LISTEN`

列出所有 listen 中的 TCP port。

{{< callout type="info" >}}
查看其他使用者的 process 需要 root 權限。
{{< /callout >}}
