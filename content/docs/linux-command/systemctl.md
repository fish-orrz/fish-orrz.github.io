---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'systemctl'
---

管理 systemd 的服務與開機模式。

## 指令總覽

```bash
systemctl status sshd
systemctl start sshd
systemctl stop sshd
systemctl restart sshd
systemctl enable --now sshd
systemctl disable --now sshd
systemctl list-units --type=service
systemctl list-units --type=service --state=running
systemctl daemon-reload
systemctl get-default
systemctl set-default multi-user.target
systemctl isolate multi-user.target
systemctl set-default graphical.target
```

## `systemctl status` / `start` / `stop` / `restart`

查看狀態、啟動、停止、重啟服務。

```bash
systemctl status sshd
systemctl restart sshd
```

## `systemctl enable --now` / `disable --now`

`enable` 設定開機自動啟動，`--now` 同時立即啟動；`disable --now` 則是取消自動啟動並立即停止。

## `systemctl list-units`

列出服務。

```bash
# 所有服務
systemctl list-units --type=service

# 只列出執行中的
systemctl list-units --type=service --state=running
```

## `systemctl daemon-reload`

修改 unit 設定檔後執行，讓 systemd 重新讀取，再 `restart` 服務才會生效。

## `systemctl get-default`

查看開機預設模式。

```text
graphical.target
```

## `systemctl set-default` / `isolate`

切換文字模式與圖形模式。

```bash
# 預設改為文字模式（下次開機生效）
systemctl set-default multi-user.target

# 立即切換到文字模式（不重開機，只有當次有效）
systemctl isolate multi-user.target

# 預設改為圖形模式
systemctl set-default graphical.target
```

`set-default` 管「下次開機」，`isolate` 管「現在」，兩者作用不同，可搭配使用。
