---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'chsh'
---

修改帳號登入後使用的 shell。

## 指令總覽

```bash
dnf install -y util-linux-user
chsh -l
chsh -s /bin/bash user1
chsh -s /bin/bash
getent passwd user1
usermod -s /bin/bash user1
```

## `dnf install -y util-linux-user`

RHEL 8 以後若找不到 `chsh` 才需要安裝。

## `chsh -l`

列出可用的 shell，內容等同 `cat /etc/shells`。

```text
/bin/sh
/bin/bash
/sbin/nologin
```

## `chsh -s`

修改 shell。`-s` 後面要接 shell 的完整路徑。

```bash
# root 修改 user1 的 shell
chsh -s /bin/bash user1

# 修改自己的 shell
chsh -s /bin/bash
```

## `getent passwd`

確認結果，最後一欄就是 shell。

```text
user1:x:1001:1001::/home/user1:/bin/bash
```

## `usermod -s`

root 也可用 `usermod` 達到相同效果：

```bash
usermod -s /bin/bash user1
```

{{< callout type="info" >}}
shell 必須是 `/etc/shells` 內列出的路徑。設為 `/sbin/nologin` 代表禁止該帳號登入。
{{< /callout >}}
