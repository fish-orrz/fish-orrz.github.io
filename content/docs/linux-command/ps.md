---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'ps'
---

查看目前的 process。

## 指令總覽

```bash
ps aux
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
pgrep -af nginx
ps -p <PID> -o pid,etime,cmd
```

## `ps aux`

列出所有 process 的完整資訊。

## `ps -eo ... --sort`

自訂欄位並排序，找出吃最多記憶體或 CPU 的 process。

```bash
# 依記憶體使用率排序，取前 10 筆
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head

# 依 CPU 使用率排序
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

```text
    PID    PPID CMD                         %MEM %CPU
   1021       1 /usr/sbin/sshd -D            0.3  0.0
```

- `--sort=-%mem` 的 `-` 代表由大到小；不加 `-` 則由小到大。
- `head` 預設只顯示前 10 行（含標題）。
- `%cpu` 是 process 生命週期的平均值，要看即時 CPU 請用 `top`。

## `pgrep -af`

依名稱找 process，比 `ps aux | grep` 乾淨（不會把 grep 自己列出來）。

```bash
pgrep -af nginx
```

## `ps -p`

查看指定 PID 的執行時間。

```bash
ps -p <PID> -o pid,etime,cmd
```
