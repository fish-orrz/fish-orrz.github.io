---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'nmcli'
---

NetworkManager 的命令列工具，本頁用來查看網路設定。

## 指令總覽

```bash
nmcli device status
nmcli connection show
nmcli connection show --active
nmcli connection show eth0
nmcli device show enp0s3
nmcli connection show enp0s3 | grep ipv4
```

{{< callout type="info" >}}
`device` 是實體網卡；`connection` 是連線設定檔，名稱不一定等於網卡名稱。名稱有空白要加引號。
{{< /callout >}}

## `nmcli device status`

列出所有網卡與狀態。

```text
DEVICE  TYPE      STATE      CONNECTION
enp0s3  ethernet  connected  enp0s3
lo      loopback  unmanaged  --
```

## `nmcli connection show`

列出所有連線設定檔。加上 `--active` 只列出啟用中的：

```bash
nmcli connection show --active
```

```text
NAME    UUID                                  TYPE      DEVICE
enp0s3  xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  ethernet  enp0s3
```

## `nmcli connection show eth0`

列出指定連線的詳細設定，把 `eth0` 換成你的連線名稱。

## `nmcli device show enp0s3`

列出指定網卡目前實際的狀態，包含 IPv4 / IPv6、Gateway、DNS。

## `nmcli connection show enp0s3 | grep ipv4`

只看 IPv4 的設定值。

```text
ipv4.method:      manual
ipv4.addresses:   192.168.1.10/24
ipv4.gateway:     192.168.1.1
ipv4.dns:         8.8.8.8
```

`ipv4.*` 是設定值，`IP4.*`（出現在 `device show`）是目前實際生效的值。
