---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'mount'
---

掛載檔案系統。本頁以掛載 Windows 分享資料夾（CIFS / SMB）為例。

## 指令總覽

```bash
dnf install -y cifs-utils
mkdir -p /mnt/nas
# 建立憑證檔 /root/.smbcred（內容見下方）
chmod 600 /root/.smbcred
mount -t cifs //nas/folder /mnt/nas -o credentials=/root/.smbcred,vers=3.0
df -h /mnt/nas
umount /mnt/nas
```

## `dnf install -y cifs-utils`

安裝掛載 CIFS 所需的工具，沒裝會出現 `wrong fs type`。

## `mkdir -p /mnt/nas`

建立掛載點。本機目錄必須先存在。

## `cat > /root/.smbcred`

建立憑證檔，避免把密碼寫在指令中而留在 history。

```bash
cat > /root/.smbcred <<'EOF'
username=xxx
password=xxx
domain=DOMAIN
EOF
chmod 600 /root/.smbcred
```

`chmod 600` 讓只有 root 能讀取。

## `mount -t cifs`

```bash
mount -t cifs //nas/folder /mnt/nas -o credentials=/root/.smbcred,vers=3.0
```

| 參數 | 說明 |
|---|---|
| `-t cifs` | 檔案系統類型 |
| `credentials=` | 帳密檔路徑 |
| `vers=` | SMB 版本（2.0 / 2.1 / 3.0），依對方伺服器調整 |

成功時不會有任何輸出。臨時測試也可直接寫帳密：`-o username=xxx,password=xxx,domain=DOMAIN,vers=3.0`。

{{< callout type="warning" >}}
帳密直接寫在指令會留在 shell history，正式使用請用憑證檔。
{{< /callout >}}

## `df -h /mnt/nas`

確認是否掛載成功。

```text
Filesystem     Size  Used Avail Use% Mounted on
//nas/folder   500G  120G  380G  24% /mnt/nas
```

## `umount /mnt/nas`

卸載。若顯示 `target is busy`，代表有程式正在使用該目錄，先離開該目錄（`cd /`）再卸載。

## 開機自動掛載

重開機後掛載會消失，需寫入 `/etc/fstab`：

```text
//nas/folder  /mnt/nas  cifs  credentials=/root/.smbcred,vers=3.0,_netdev  0 0
```

`_netdev` 表示等網路起來後再掛載。寫入後用 `mount -a` 測試，沒報錯才代表設定正確。
