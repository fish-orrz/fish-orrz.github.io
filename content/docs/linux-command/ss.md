---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'ss'
---

查看 socket、port 與連線狀態，用來取代 `netstat`。

## 指令總覽

```bash
ss -tlnp
ss -tn state established
ss -Htn state established '( sport = :443 )' | wc -l
ss -s
```

## `ss -tlnp`

列出所有 listen 中的 TCP port 與對應的 process。

```text
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
LISTEN  0       128     0.0.0.0:22          0.0.0.0:*          users:(("sshd",pid=1021,fd=3))
```

| 參數 | 說明 |
|---|---|
| `-t` / `-u` | TCP / UDP |
| `-l` | 只顯示 listen |
| `-n` | 不解析名稱，顯示數字 |
| `-p` | 顯示 process（需 root） |

## `ss -tn state established`

列出目前已建立的 TCP 連線。

## `ss -Htn ... | wc -l`

統計 443 port 目前已建立的連線數。

```bash
ss -Htn state established '( sport = :443 )' | wc -l
```

`-H` 不顯示標題列，所以 `wc -l` 的數字就是連線數。`sport` 是本機 port，`dport` 是對方 port。

{{< callout type="warning" >}}
`ss` 顯示的狀態是 `ESTAB`，不是 `ESTABLISHED`（那是 netstat 的寫法），用 `grep ESTABLISHED` 會找不到東西。
{{< /callout >}}

## `ss -s`

顯示連線統計摘要。
