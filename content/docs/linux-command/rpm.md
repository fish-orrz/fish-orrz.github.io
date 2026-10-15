---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'rpm'
---

查詢已安裝的套件。安裝與移除建議使用 [dnf](../dnf)。

## 指令總覽

```bash
rpm -qa | grep gzip
rpm -qi gzip
rpm -ql gzip
rpm -qf /etc/fstab
rpm -V setup
dnf reinstall -y setup
```

## `rpm -qa | grep`

列出所有已安裝套件並過濾，用來確認某套件是否已安裝。

## `rpm -qi` / `rpm -ql`

`-qi` 查看套件資訊（版本、說明），`-ql` 列出套件包含哪些檔案。

## `rpm -qf`

查詢某個檔案屬於哪個套件。

```bash
rpm -qf /etc/fstab
```

```text
setup-2.13.7-10.el8.noarch
```

## `rpm -V`

驗證套件的檔案是否被修改過。沒有輸出代表完全正常。

```bash
rpm -V setup
```

```text
S.5....T.  c /etc/fstab
```

開頭的字母代表被改的項目（`S` 大小、`5` 內容、`T` 時間），`c` 代表這是設定檔。

## `dnf reinstall`

檔案損壞時，用 `rpm -qf` 找出所屬套件後重裝。

```bash
dnf reinstall -y setup
```

{{< callout type="warning" >}}
`/etc` 下的設定檔重裝時通常不會覆蓋現有檔案，新版會另存成 `.rpmnew`，需自行比對與還原。
{{< /callout >}}
