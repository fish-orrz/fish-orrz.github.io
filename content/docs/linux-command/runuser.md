---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'runuser'
---

由 root 切換成其他帳號執行指令，不需輸入密碼，一般使用者無法使用。常用於腳本與服務啟動。

## 指令總覽

```bash
runuser -l user1
runuser -l user1 -c 'whoami'
runuser -u user1 -- whoami
runuser -s /bin/bash user1
```

## `runuser -l`

切換成 `user1`（login shell，載入該帳號的環境），輸入 `exit` 離開。

```bash
runuser -l user1
```

加上 `-c` 則只執行單一指令後離開：

```bash
runuser -l user1 -c 'whoami'
```

```text
user1
```

## `runuser -u`

以 `user1` 執行指令，不載入登入環境。`--` 之後是要執行的指令。

```bash
runuser -u user1 -- whoami
```

## `runuser -s`

帳號的 shell 是 `nologin` 時，臨時指定 shell。

```bash
runuser -s /bin/bash user1
```

## 與 su / sudo 的差異

| 指令 | 說明 |
|---|---|
| `runuser` | root 切換成其他帳號，不問密碼 |
| `su` | 任何人切換帳號，需輸入對方密碼 |
| `sudo` | 以自己的身分借用權限，依 sudoers 設定 |
