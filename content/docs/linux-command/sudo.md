---
title: sudo
weight: 2
---

讓一般帳號暫時以 root(或其他使用者)的身分執行指令。環境:RHEL,主機名稱 `host01`。

{{< callout type="warning" >}}
- 一律用 **`visudo -f`** 編輯,不要用 `vi` 直接改。sudoers 語法錯誤會讓所有人都無法使用 sudo。
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

{{% steps %}}

### `useradd -m user1`

**作用**:建立帳號 `user1`,`-m` 表示同時建立家目錄 `/home/user1`。

**範例輸出**:成功時**沒有輸出**。

### `passwd user1`

**作用**:設定 `user1` 的登入密碼。

**範例輸出**:

```text
Changing password for user user1.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

**注意**:輸入密碼時畫面不會顯示任何字元,是正常的。

### `visudo -f /etc/sudoers.d/user1`

**作用**:用安全的方式建立 `user1` 專屬的 sudo 設定檔。存檔時會自動檢查語法,有錯會提示,不會直接寫入錯誤內容。

**範例內容**:在編輯器輸入下列這一行,按 `Esc` 後輸入 `:wq` 存檔離開。

```text
user1 ALL=(ALL) NOPASSWD: ALL
```

**這行的意思**:`user1`(帳號)`ALL`(任何主機)`=(ALL)`(可切換成任何身分)`NOPASSWD:`(不用輸入密碼)`ALL`(可執行任何指令)。

### `ls -l /etc/sudoers.d/user1`

**作用**:確認檔案的擁有者與權限。

**範例輸出**:

```text
-r--r----- 1 root root 30 Oct 10 14:20 /etc/sudoers.d/user1
```

**怎麼看**:擁有者是 `root`,權限是 `-r--r-----`(0440)才正確。

### `visudo -c`

**作用**:檢查所有 sudoers 檔案的語法。

**範例輸出**:

```text
/etc/sudoers: parsed OK
/etc/sudoers.d/user1: parsed OK
```

**怎麼看**:每個檔案都是 `parsed OK` 才安全。

### `sudo -l -U user1`

**作用**:列出 `user1` 被允許的 sudo 權限。

**範例輸出**:

```text
Matching Defaults entries for user1 on host01:
    !visiblepw, always_set_home, env_reset, ...

User user1 may run the following commands on host01:
    (ALL) NOPASSWD: ALL
```

### `su - user1`

**作用**:切換成 `user1`,實際測試這個帳號。

**範例輸出**:成功時提示字元會改變。

```text
[user1@host01 ~]$
```

### `sudo whoami`

**作用**:以 sudo 執行 `whoami`,確認是否變成 root。

**範例輸出**:

```text
root
```

**怎麼看**:沒有要求輸入密碼,且顯示 `root`,代表設定成功。

{{% /steps %}}

## 情境二:只允許特定指令

`user2` 只能重啟與查看 nginx,其他指令一律拒絕。

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

{{% steps %}}

### `which systemctl`

**作用**:查出 `systemctl` 的**完整路徑**。sudoers 規則一定要寫完整路徑。

**範例輸出**:

```text
/usr/bin/systemctl
```

### `visudo -f /etc/sudoers.d/user2`

**作用**:建立 `user2` 專屬的 sudo 設定檔,只開放指定的指令。

**範例內容**:輸入下列一行(多個指令用逗號分隔),`:wq` 存檔離開。

```text
user2 ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
```

**這行的意思**:`user2` 可以用 root 身分、免密碼,執行 `restart nginx` 與 `status nginx` 這兩個指令,其他都不行。

### `visudo -c`

**作用**:檢查所有 sudoers 檔案的語法。

**範例輸出**:

```text
/etc/sudoers: parsed OK
/etc/sudoers.d/user2: parsed OK
```

### `sudo -l -U user2`

**作用**:列出 `user2` 被允許的 sudo 權限。

**範例輸出**:

```text
User user2 may run the following commands on host01:
    (root) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
```

### `su - user2`

**作用**:切換成 `user2`,實際測試這個帳號。

**範例輸出**:

```text
[user2@host01 ~]$
```

### `sudo systemctl restart nginx`

**作用**:測試**允許**的指令,重啟 nginx。

**範例輸出**:成功時**沒有輸出**。

### `sudo systemctl stop nginx`

**作用**:測試**不允許**的指令,應該被拒絕。

**範例輸出**:

```text
[sudo] password for user2:
Sorry, user user2 is not allowed to execute '/usr/bin/systemctl stop nginx' as root on host01.
```

**怎麼看**:出現 `not allowed`,代表限制生效。

{{% /steps %}}

## 情境三:用群組給 sudo

RHEL 的 `wheel` 群組預設就能使用 sudo(需輸入密碼)。Ubuntu 則是 `sudo` 群組。

### 全部指令

```bash
useradd -m user3
usermod -aG wheel user3
id user3
sudo -l -U user3
```

{{% steps %}}

### `useradd -m user3`

**作用**:建立帳號 `user3`,同時建立家目錄。

**範例輸出**:成功時**沒有輸出**。

### `usermod -aG wheel user3`

**作用**:把 `user3` 加入 `wheel` 群組。

**範例輸出**:成功時**沒有輸出**。

**注意**:`-a` 是「附加」,**不可省略**。少了 `-a` 會把使用者從其他群組移除。

### `id user3`

**作用**:顯示帳號所屬的群組。

**範例輸出**:

```text
uid=1002(user3) gid=1002(user3) groups=1002(user3),10(wheel)
```

**怎麼看**:`groups` 裡有 `10(wheel)` 即加入成功。

### `sudo -l -U user3`

**作用**:列出 `user3` 被允許的 sudo 權限。

**範例輸出**:

```text
User user3 may run the following commands on host01:
    (ALL) ALL
```

**怎麼看**:`(ALL) ALL` 表示可執行任何指令,但每次需要輸入密碼(沒有 `NOPASSWD`)。

{{% /steps %}}

## 情境四:移除 sudo 權限

### 全部指令

```bash
rm /etc/sudoers.d/user1
sudo -l -U user1
gpasswd -d user3 wheel
sudo -l -U user3
```

{{% steps %}}

### `rm /etc/sudoers.d/user1`

**作用**:刪除 `user1` 的 sudo 設定檔,權限立即失效。

**範例輸出**:成功時**沒有輸出**。

### `sudo -l -U user1`

**作用**:確認 `user1` 已沒有 sudo 權限。

**範例輸出**:

```text
User user1 is not allowed to run sudo on host01.
```

### `gpasswd -d user3 wheel`

**作用**:把 `user3` 從 `wheel` 群組移除。

**範例輸出**:

```text
Removing user user3 from group wheel
```

### `sudo -l -U user3`

**作用**:確認 `user3` 已沒有 sudo 權限。

**範例輸出**:

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

{{% steps %}}

### `sudo -l`

**作用**:查看**自己**有哪些 sudo 權限。

**範例輸出**:

```text
User user3 may run the following commands on host01:
    (ALL) ALL
```

### `sudo -i`

**作用**:切換成 root 的互動式 shell,連續執行多個管理指令時使用。

**範例輸出**:

```text
[root@host01 ~]#
```

**注意**:提示字元變成 `#`,代表現在是 root。完成後輸入 `exit` 離開。

### `sudo -u nobody whoami`

**作用**:以**指定的使用者**身分執行指令(`-u` 後面接使用者名稱)。

**範例輸出**:

```text
nobody
```

### `sudo !!`

**作用**:忘記加 sudo 時,自動在上一個指令前補上 sudo 再執行。

**範例輸出**:

```text
[user3@host01 ~]$ cat /var/log/secure
cat: /var/log/secure: Permission denied
[user3@host01 ~]$ sudo !!
sudo cat /var/log/secure
[sudo] password for user3:
Oct 10 14:30:01 host01 sshd[1234]: Accepted publickey for user3 ...
```

**怎麼看**:第二次執行前會先顯示完整指令 `sudo cat /var/log/secure`。

### `sudo -k`

**作用**:清除 sudo 的密碼快取,下一次使用 sudo 必須重新輸入密碼。

**範例輸出**:成功時**沒有輸出**。

{{% /steps %}}