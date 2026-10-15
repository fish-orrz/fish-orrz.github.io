---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'cockpit'
---

用瀏覽器管理 Linux 的 Web UI。

## 指令總覽

```bash
dnf install -y cockpit
systemctl enable --now cockpit.socket
firewall-cmd --permanent --add-service=cockpit
firewall-cmd --reload
systemctl status cockpit.socket
ss -tlnp | grep 9090
```

## `dnf install -y cockpit`

安裝 Cockpit。

## `systemctl enable --now cockpit.socket`

設定開機自動啟動並立即啟動。Cockpit 以 socket 方式運作，有人連線時才會拉起服務，所以不需要另外 `start`。

## `firewall-cmd`

放行預設的 9090 port。

```bash
firewall-cmd --permanent --add-service=cockpit
firewall-cmd --reload
```

## 登入

瀏覽器開啟 `https://<主機IP>:9090`，使用系統帳號登入。憑證為自簽，瀏覽器會出現警告，確認後繼續即可。

{{< callout type="info" >}}
預設禁止 root 登入，請用一般帳號。
{{< /callout >}}

## 設定檔

| 檔案 | 說明 |
|---|---|
| `/etc/cockpit/cockpit.conf` | 主設定檔（預設不存在，需自行建立） |
| `/etc/cockpit/disallowed-users` | 禁止登入的帳號，一行一個（預設含 root） |

## 變更 port（以 7777 為例）

### `listen.conf`

建立 systemd drop-in 設定：

```bash
mkdir -p /etc/systemd/system/cockpit.socket.d
cat > /etc/systemd/system/cockpit.socket.d/listen.conf <<'EOF'
[Socket]
ListenStream=
ListenStream=7777
EOF
```

第一行空的 `ListenStream=` 是清除預設的 9090，不能省略。

### `semanage port`

SELinux 為 Enforcing（`getenforce` 可查）時，必須允許新 port，否則 Cockpit 無法啟動：

```bash
dnf install -y policycoreutils-python-utils
semanage port -a -t websm_port_t -p tcp 7777
```

### `firewall-cmd`

```bash
firewall-cmd --permanent --add-port=7777/tcp
firewall-cmd --permanent --remove-service=cockpit
firewall-cmd --reload
```

### `systemctl restart`

套用並確認：

```bash
systemctl daemon-reload
systemctl restart cockpit.socket
ss -tlnp | grep 7777
```

{{< callout type="warning" >}}
只綁定特定 IP 時寫成 `ListenStream=192.168.1.1:7777`，並另起一行加上 `FreeBind=yes`。

systemd 設定檔不支援行尾註解，註解必須獨立一行並以 `#` 開頭。
{{< /callout >}}
