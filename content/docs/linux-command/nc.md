---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'nc'
---

測試 port 是否連通。RHEL 8 以後的 `nc` 就是 `ncat`，用法相同，詳見 [ncat](../ncat)。

## 指令總覽

```bash
nc -zvw3 localhost 80
echo $?
timeout 3 bash -c '</dev/tcp/10.136.225.3/22' && echo open || echo closed
```

## `nc -zvw3`

檢查 `localhost` 的 80 port 是否有服務在監聽。

## `echo $?`

`0` 代表有在監聽，非 `0` 代表沒有。

## 沒有 nc 時的替代寫法

使用 bash 內建的 `/dev/tcp`，不需額外安裝任何工具：

```bash
timeout 3 bash -c '</dev/tcp/10.136.225.3/22' && echo open || echo closed
```
