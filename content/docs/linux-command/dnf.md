---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'dnf'
---

RHEL / Rocky / AlmaLinux / Fedora 的套件管理工具。

## 指令總覽

```bash
dnf install -y epel-release
dnf repolist
dnf search htop
dnf install -y htop
dnf remove htop
dnf provides '*/ncat'
dnf group list
dnf group install "Server with GUI"
systemctl set-default graphical.target
```

## `dnf install -y epel-release`

安裝 EPEL（額外套件庫），許多常用工具（如 htop）在裡面。`-y` 代表自動回答 yes。

{{< callout type="info" >}}
Rocky / AlmaLinux 可直接安裝。RHEL 沒有這個套件，需用官方網址安裝：
`dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm`（9 換成你的版本）
{{< /callout >}}

## `dnf repolist`

列出目前啟用的套件庫，用來確認 EPEL 是否生效。

## `dnf search` / `dnf install` / `dnf remove`

搜尋、安裝、移除套件。

```bash
dnf search htop
dnf install -y htop
dnf remove htop
```

## `dnf provides`

查詢某個指令或檔案由哪個套件提供。

```bash
dnf provides '*/ncat'
```

## `dnf group install`

安裝套件群組，例如 GUI 桌面環境。

```bash
dnf group list
dnf group install "Server with GUI"
```

## `systemctl set-default graphical.target`

安裝 GUI 後必須設定，否則開機仍是文字模式。設定後重開機生效。
