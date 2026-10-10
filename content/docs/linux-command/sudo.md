---
title: sudo
weight: 2
---

讓一般帳號以 root 或其他使用者身分執行指令。環境:RHEL,主機名稱 `host01`。

{{< callout type="warning" >}}
- 一律用 **`visudo -f`** 編輯,不要用 `vi` 直接改,語法錯誤會讓 sudo 全部失效。
- `/etc/sudoers.d/` 內的檔名**不能含 `.`**,也不能以 `~` 結尾,否則會被忽略。
- `NOPASSWD: ALL` 等同免密碼 root,只給信任的帳號。
{{< /callout >}}

## 情境一:新帳號給完整 sudo(免密碼)

### 全部指令

```bash
useradd -m user1
passwd user1
visudo -f /etc/sudoers.d/user1
ls -l /etc/sudoers.d/user1
visudo -c
sudo -l -U user1
su - user1
sudo whoami
```

### 步驟

{{% steps %}}

### 建立帳號

**`useradd -m user1`**:成功時**沒有輸出**。

**`passwd user1`**

```text
Changing password for user user1.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

### 建立 sudoers 檔案

**`visudo -f /etc/sudoers.d/user1`**:在編輯器輸入下列內容,按 `Esc` 後輸入 `:wq` 存檔離開。

```text
user1 ALL=(ALL) NOPASSWD: ALL
```

格式:`帳號 主機=(可切換的身分) 選項: 可執行的指令`。

**`ls -l /etc/sudoers.d/user1`**:權限必須是 `-r--r-----`(0440),擁有者 `root`。

```text
-r--r----- 1 root root 30 Oct 10 14:20 /etc/sudoers.d/user1
```

**`visudo -c`**:檢查所有 sudoers 檔案語法。

```text
/etc/sudoers: parsed OK
/etc/sudoers.d/user1: parsed OK
```

### 驗證

**`sudo -l -U user1`**:查看 user1 的 sudo 權限。

```text
Matching Defaults entries for user1 on host01:
    !visiblepw, always_set_home, env_reset, ...

User user1 may run the following commands on host01:
    (ALL) NOPASSWD: ALL
```

**`su - user1`**:切換成 user1,成功時只有提示字元改變。

```text
[user1@host01 ~]$
```

**`sudo whoami`**:不需輸入密碼,顯示 `root` 即成功。

```text
root
```

{{% /steps %}}

## 情境二:只允許特定指令

user2 只能重啟與查看 nginx,其他指令一律拒絕。

### 全部指令

```bash
which systemctl
visudo -f /etc/sudoers.d/user2
visudo -c
sudo -l -U user2
su - user2
sudo systemctl restart nginx
sudo systemctl stop nginx
```

### 步驟

{{% steps %}}

### 確認指令的完整路徑

**`which systemctl`**:sudoers 規則必須寫**完整路徑**。

```text
/usr/bin/systemctl
```

### 建立 sudoers 檔案

**`visudo -f /etc/sudoers.d/user2`**:輸入下列內容(同一行,多個指令用逗號分隔),`:wq` 存檔離開。

```text
user2 ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
```

**`visudo -c`**

```text
/etc/sudoers: parsed OK
/etc/sudoers.d/user2: parsed OK
```

### 驗證

**`sudo -l -U user2`**

```text
User user2 may run the following commands on host01:
    (root) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
```

**`su - user2`** → **`sudo systemctl restart nginx`**:允許的指令,成功時**沒有輸出**。

**`sudo systemctl stop nginx`**:不在清單內,被拒絕。

```text
[sudo] password for user2:
Sorry, user user2 is not allowed to execute '/usr/bin/systemctl stop nginx' as root on host01.
```

{{% /steps %}}

## 情境三:用群組給 sudo

RHEL 的 `wheel` 群組預設可使用 sudo(需輸入密碼)。Ubuntu 則是 `sudo` 群組。

### 全部指令

```bash
useradd -m user3
usermod -aG wheel user3
id user3
sudo -l -U user3
```

### 步驟

{{% steps %}}

### 建立帳號並加入群組

**`useradd -m user3`**:成功時**沒有輸出**。

**`usermod -aG wheel user3`**:`-a` 是附加,**不可省略**,否則會把使用者從其他群組移除。成功時**沒有輸出**。

### 驗證

**`id user3`**:`groups` 要有 `wheel`。

```text
uid=1002(user3) gid=1002(user3) groups=1002(user3),10(wheel)
```

**`sudo -l -U user3`**

```text
User user3 may run the following commands on host01:
    (ALL) ALL
```

{{% /steps %}}

## 情境四:移除 sudo 權限

### 全部指令

```bash
rm /etc/sudoers.d/user1
sudo -l -U user1
gpasswd -d user3 wheel
sudo -l -U user3
```

### 步驟

{{% steps %}}

### 移除 sudoers 檔案

**`rm /etc/sudoers.d/user1`**:成功時**沒有輸出**。

**`sudo -l -U user1`**

```text
User user1 is not allowed to run sudo on host01.
```

### 移出群組

**`gpasswd -d user3 wheel`**

```text
Removing user user3 from group wheel
```

**`sudo -l -U user3`**

```text
User user3 is not allowed to run sudo on host01.
```

{{% /steps %}}

## 情境五:日常常用

### 全部指令

```bash
sudo -l
sudo -i
sudo -u nobody whoami
sudo !!
sudo -k
```

### 步驟

{{% steps %}}

### 查看自己的權限

**`sudo -l`**

```text
User user3 may run the following commands on host01:
    (ALL) ALL
```

### 切換成 root 互動 shell

**`sudo -i`**:提示字元變成 `#`,輸入 `exit` 離開。

```text
[root@host01 ~]#
```

### 以其他使用者身分執行

**`sudo -u nobody whoami`**

```text
nobody
```

### 補上 sudo 重跑上一個指令

**`sudo !!`**:忘記加 sudo 時使用,會先顯示完整指令再執行。

```text
[user3@host01 ~]$ cat /var/log/secure
cat: /var/log/secure: Permission denied
[user3@host01 ~]$ sudo !!
sudo cat /var/log/secure
[sudo] password for user3:
Oct 10 14:30:01 host01 sshd[1234]: Accepted publickey for user3 ...
```

### 清除密碼快取

**`sudo -k`**:成功時**沒有輸出**,下一次 sudo 必須重新輸入密碼。

{{% /steps %}}