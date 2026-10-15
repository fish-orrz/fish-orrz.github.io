---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'ncat'
---

測試遠端 port 是否連通。

## 指令總覽

```bash
dnf install -y nmap-ncat
ncat -zvw3 10.136.225.3 22
echo $?
```

## `dnf install -y nmap-ncat`

安裝 `ncat`（`nc` 指令也會一併提供）。

## `ncat -zvw3`

測試 `10.136.225.3` 的 22 port。

| 參數 | 說明 |
|---|---|
| `-z` | 只測試連線，不傳送資料 |
| `-v` | 顯示詳細過程 |
| `-w3` | 逾時 3 秒 |

結果：

```text
Ncat: Connected to 10.136.225.3:22.
```

| 訊息 | 意義 |
|---|---|
| `Connected to` | port 通 |
| `Connection refused` | 主機可達，但該 port 沒有服務在監聽 |
| `Connection timed out` | 被防火牆擋住或網路不通 |

## `echo $?`

查看上一個指令的結果，`0` 代表連通，非 `0` 代表不通，適合寫在腳本中。
