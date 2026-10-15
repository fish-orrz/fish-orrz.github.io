---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'ldd'
---

查看執行檔依賴哪些共享函式庫（`.so`）。

## 指令總覽

```bash
ldd /usr/sbin/httpd
ldd $(which curl)
ldd /usr/sbin/httpd | grep "not found"
```

## `ldd`

需使用完整路徑，不知道路徑時搭配 `which`。

```bash
ldd /usr/sbin/httpd
ldd $(which curl)
```

```text
	libcurl.so.4 => /lib64/libcurl.so.4 (0x00007f...)
	libfoo.so.1 => not found
```

## `ldd | grep "not found"`

只找出缺少的函式庫。出現 `not found` 代表缺少該函式庫，可用 `dnf provides '*/libfoo.so.1'` 找出所屬套件。

{{< callout type="warning" >}}
不要對不信任的檔案執行 `ldd`，它可能會執行該檔案。
{{< /callout >}}
