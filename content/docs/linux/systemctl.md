---
date: '2026-11-14T10:53:49+08:00'
draft: false
title: 'Systemctl'
---

環境：RHEL 8（systemd 239）。需 root 或 `sudo`。

## 1. 新增一個 Service

流程：寫 unit 檔 → `daemon-reload` → 啟動 → 設定開機自啟。

### 1.1 Unit 檔位置

| 路徑 | 用途 |
|---|---|
| `/etc/systemd/system/` | 管理員自訂（優先，自己的 service 放這） |
| `/run/systemd/system/` | 執行期產生，重開機消失 |
| `/usr/lib/systemd/system/` | 套件安裝，不要直接改 |

### 1.2 步驟

```bash
# 1. 建立 unit 檔（檔名 .service 結尾）
vi /etc/systemd/system/myapp.service

# 2. 讓 systemd 重新讀取
systemctl daemon-reload

# 3. 啟動並設為開機自啟
systemctl enable --now myapp

# 4. 確認
systemctl status myapp
```

> 每次修改 unit 檔都要 `systemctl daemon-reload`，再 `restart` 才生效。

### 1.3 Unit 檔三大區塊

| 區塊 | 說明 |
|---|---|
| `[Unit]` | 描述與依賴順序（`Description`、`After`、`Requires`、`Wants`） |
| `[Service]` | 如何啟動／停止／重啟 |
| `[Install]` | `enable` 時掛到哪個 target（通常 `multi-user.target`） |

`After` 只管順序，`Wants`/`Requires` 才管依賴：

| 欄位 | 說明 |
|---|---|
| `After=` | 在指定 unit 之後啟動（只管順序） |
| `Wants=` | 弱依賴，對方失敗不影響自己 |
| `Requires=` | 強依賴，對方失敗自己也失敗 |

## 2. Service 範例（啟動／停止／重啟）

### 2.1 範例程式（測試用）

`/usr/local/bin/myapp.sh`：

```bash
#!/bin/bash
trap 'echo "reload: config reloaded"' HUP
echo "myapp started, pid=$$"
while true; do
  echo "myapp running $(date +%T)"
  sleep 10 &
  wait $!
done
```

```bash
chmod +x /usr/local/bin/myapp.sh
```

> 腳本放 `/usr/local/bin` 或 `/opt`，確保有執行權限。若 SELinux 阻擋，`restorecon -Rv /usr/local/bin/myapp.sh`。

### 2.2 Unit 檔

`/etc/systemd/system/myapp.service`：

```ini
[Unit]
Description=My App
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/tmp
Environment=APP_ENV=prod
# 啟動
ExecStart=/usr/local/bin/myapp.sh
# 重載設定（systemctl reload）
ExecReload=/bin/kill -HUP $MAINPID
# 停止（省略時預設送 SIGTERM）
ExecStop=/bin/kill -TERM $MAINPID
# 重啟策略
Restart=on-failure
RestartSec=5
# 停止等待時間，逾時送 SIGKILL
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

### 2.3 操作

```bash
systemctl daemon-reload
systemctl start myapp        # 啟動
systemctl stop myapp         # 停止
systemctl restart myapp      # 重啟（stop + start）
systemctl reload myapp       # 重載設定，不中斷（觸發 ExecReload）
systemctl enable myapp       # 開機自啟
systemctl disable myapp      # 取消開機自啟
```

### 2.4 常用 `[Service]` 欄位

| 欄位 | 說明 |
|---|---|
| `Type=` | 啟動型態，見下表 |
| `ExecStart=` | 啟動指令（必須絕對路徑） |
| `ExecStartPre=` / `ExecStartPost=` | 啟動前／後執行 |
| `ExecStop=` | 停止指令 |
| `ExecReload=` | 重載指令 |
| `Restart=` | 重啟策略，見下表 |
| `RestartSec=` | 重啟前等待秒數 |
| `User=` / `Group=` | 以何種身分執行 |
| `WorkingDirectory=` | 工作目錄 |
| `Environment=` | 環境變數 |
| `EnvironmentFile=` | 從檔案讀環境變數 |
| `PIDFile=` | PID 檔（`Type=forking` 用） |
| `TimeoutStartSec=` / `TimeoutStopSec=` | 啟動／停止逾時 |
| `LimitNOFILE=` | 開檔數上限 |
| `StandardOutput=` | 輸出去向，預設 journal |

`Type` 選擇：

| Type | 說明 |
|---|---|
| `simple` | 預設，ExecStart 即主程序（前景執行） |
| `forking` | 程序自行 fork 到背景，父程序退出（搭配 `PIDFile=`） |
| `oneshot` | 執行完就結束，常搭配 `RemainAfterExit=yes` |
| `notify` | 程序用 sd_notify 通知就緒 |
| `exec` | 類似 simple，exec 成功才算啟動 |

`Restart` 選擇：

| 值 | 說明 |
|---|---|
| `no` | 預設，不重啟 |
| `on-failure` | 非正常結束才重啟（推薦） |
| `always` | 一律重啟（含 `systemctl stop` 以外的任何退出） |
| `on-abnormal` | 被訊號殺掉、逾時才重啟 |

### 2.5 Forking 型範例（傳統背景程式）

```ini
[Service]
Type=forking
PIDFile=/var/run/myapp.pid
ExecStart=/opt/myapp/bin/start.sh
ExecStop=/opt/myapp/bin/stop.sh
ExecReload=/opt/myapp/bin/reload.sh
TimeoutStopSec=60
```

### 2.6 不改原 unit，只覆寫設定（drop-in）

```bash
systemctl edit myapp
```

```ini
[Service]
Restart=always
RestartSec=10
```

產生 `/etc/systemd/system/myapp.service.d/override.conf`，存檔後自動 `daemon-reload`。清單型欄位要先清空：

```ini
[Service]
ExecStart=
ExecStart=/usr/local/bin/myapp.sh --new-option
```

## 3. 常見檢查指令

### 3.1 單一服務

| 指令 | 說明 |
|---|---|
| `systemctl status myapp` | 狀態、PID、cgroup、最近日誌 |
| `systemctl is-active myapp` | 是否執行中（`active`/`inactive`/`failed`） |
| `systemctl is-enabled myapp` | 是否開機自啟 |
| `systemctl is-failed myapp` | 是否失敗 |
| `systemctl cat myapp` | 顯示實際 unit 內容（含 drop-in） |
| `systemctl show myapp` | 所有屬性 |
| `systemctl show myapp -p MainPID,ExecStart,ActiveState` | 指定屬性 |
| `systemctl list-dependencies myapp` | 依賴樹 |
| `systemctl list-dependencies --reverse myapp` | 誰依賴它 |
| `systemctl reset-failed myapp` | 清除 failed 狀態 |
| `systemd-analyze verify /etc/systemd/system/myapp.service` | 檢查 unit 語法 |

`status` 重點：

| 欄位 | 說明 |
|---|---|
| `Loaded` | unit 檔路徑；`enabled`/`disabled`/`masked` |
| `Active` | `active (running)`、`active (exited)`、`inactive (dead)`、`failed` |
| `Main PID` | 主程序 PID 與結束碼 |
| `CGroup` | 該服務所有程序 |

### 3.2 全系統

| 指令 | 說明 |
|---|---|
| `systemctl --failed` | 列出失敗的 unit |
| `systemctl list-units --type=service` | 已載入的 service |
| `systemctl list-units --type=service --state=running` | 執行中的 service |
| `systemctl list-unit-files --type=service` | 所有 service 檔與啟用狀態 |
| `systemctl list-unit-files --state=enabled` | 開機自啟清單 |
| `systemctl list-timers` | timer 與下次觸發時間 |
| `systemctl list-sockets` | socket unit |
| `systemctl is-system-running` | 系統整體狀態（`running`/`degraded`） |
| `systemd-analyze blame` | 各 unit 開機耗時排序 |
| `systemd-analyze critical-chain` | 開機關鍵路徑 |

### 3.3 常見錯誤碼

| 訊息 | 原因 |
|---|---|
| `Unit xxx.service not found` | 檔名錯，或沒 `daemon-reload` |
| `status=203/EXEC` | ExecStart 路徑錯、無執行權限、SELinux 阻擋 |
| `status=200/CHDIR` | WorkingDirectory 不存在 |
| `status=217/USER` | User 不存在 |
| `start-limit-hit` | 短時間重啟太多次，`reset-failed` 後再啟動 |
| `Unit is masked` | 被 mask，需 `systemctl unmask` |

SELinux 排查：

```bash
ausearch -m avc -ts recent
restorecon -Rv /usr/local/bin/myapp.sh
```

## 4. Target

Target 是 unit 的群組，取代舊式 runlevel，決定開機要啟動哪些服務。

### 4.1 常見 Target

| Target | 說明 | 舊 runlevel |
|---|---|---|
| `poweroff.target` | 關機 | 0 |
| `rescue.target` | 單人維護模式（掛載檔案系統、基本服務） | 1 |
| `emergency.target` | 緊急模式（僅 root 唯讀掛載） | - |
| `multi-user.target` | 多使用者、文字模式（伺服器預設） | 3 |
| `graphical.target` | 圖形介面 | 5 |
| `reboot.target` | 重開機 | 6 |
| `default.target` | 預設 target（符號連結） | - |
| `network-online.target` | 網路已連線 | - |
| `timers.target` | 所有 timer | - |
| `sockets.target` | 所有 socket | - |

### 4.2 指令

| 指令 | 說明 |
|---|---|
| `systemctl get-default` | 查看預設 target |
| `systemctl set-default multi-user.target` | 設預設為文字模式 |
| `systemctl set-default graphical.target` | 設預設為圖形模式 |
| `systemctl isolate multi-user.target` | 立即切換（不重開機） |
| `systemctl rescue` | 進入 rescue 模式 |
| `systemctl emergency` | 進入 emergency 模式 |
| `systemctl list-units --type=target` | 目前啟用的 target |
| `systemctl list-unit-files --type=target` | 所有 target |
| `systemctl list-dependencies multi-user.target` | 該 target 包含哪些 unit |

### 4.3 `[Install]` 與 Target 的關係

```ini
[Install]
WantedBy=multi-user.target
```

`systemctl enable myapp` 會在 `/etc/systemd/system/multi-user.target.wants/` 建立指向 unit 檔的符號連結；`disable` 則移除。開機進入 `multi-user.target` 時，就會連帶啟動 myapp。

```bash
ls -l /etc/systemd/system/multi-user.target.wants/
```

### 4.4 開機進入 Rescue（忘記密碼／修復）

GRUB 開機選單按 `e`，在 `linux` 行尾加：

```text
systemd.unit=rescue.target
```

按 `Ctrl+X` 開機。

## 5. Journalctl

systemd 的日誌由 `systemd-journald` 收集，包含服務 stdout/stderr、syslog、核心訊息。格式為二進位，必須用 `journalctl` 讀取。

### 5.1 常用查詢

| 指令 | 說明 |
|---|---|
| `journalctl` | 全部日誌（舊到新） |
| `journalctl -r` | 新到舊 |
| `journalctl -n 50` | 最後 50 行 |
| `journalctl -f` | 即時追蹤（類似 `tail -f`） |
| `journalctl -u myapp` | 指定服務 |
| `journalctl -u myapp -f` | 追蹤指定服務 |
| `journalctl -u myapp -u nginx` | 多個服務 |
| `journalctl -u myapp -n 100 --no-pager` | 不分頁，方便輸出到檔案 |
| `journalctl -xe` | 跳到結尾並附說明（排錯用） |
| `journalctl -b` | 本次開機 |
| `journalctl -b -1` | 上一次開機 |
| `journalctl --list-boots` | 列出所有開機紀錄 |
| `journalctl -k` | 核心訊息（同 `dmesg`） |
| `journalctl -p err` | err 以上等級 |
| `journalctl -p warning..err` | 指定等級範圍 |
| `journalctl _PID=1234` | 指定 PID |
| `journalctl _UID=1000` | 指定使用者 |
| `journalctl /usr/sbin/sshd` | 指定執行檔 |
| `journalctl -g "error"` | 搜尋關鍵字（grep，RHEL 8 支援） |

優先等級：`emerg(0)` `alert(1)` `crit(2)` `err(3)` `warning(4)` `notice(5)` `info(6)` `debug(7)`

### 5.2 時間篩選

```bash
journalctl --since "1 hour ago"
journalctl --since "2026-11-14 09:00" --until "2026-11-14 10:00"
journalctl --since today
journalctl --since yesterday --until today
journalctl -u myapp --since "30 min ago"
```

### 5.3 輸出格式

| 指令 | 說明 |
|---|---|
| `journalctl -o short-iso` | ISO 時間格式 |
| `journalctl -o verbose` | 顯示所有欄位 |
| `journalctl -o json-pretty` | JSON（格式化） |
| `journalctl -o cat` | 只顯示訊息本體 |
| `journalctl -u myapp > myapp.log` | 匯出成文字檔 |

### 5.4 日誌存放位置

| 路徑 | 說明 |
|---|---|
| `/run/log/journal/` | 揮發性（記憶體），重開機消失 |
| `/var/log/journal/` | 持久性（磁碟），重開機保留 |
| `/etc/systemd/journald.conf` | journald 設定檔 |
| `/etc/systemd/journald.conf.d/*.conf` | 設定 drop-in（推薦） |

**RHEL 8 預設**：`Storage=auto`，只有 `/var/log/journal/` 目錄存在才會持久保存。預設該目錄不存在，所以重開機後日誌消失。

```bash
# 確認目前是否持久化
ls -ld /var/log/journal
journalctl --list-boots
```

### 5.5 設定持久化日誌

方法一：建立目錄（auto 模式下自動切換）

```bash
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald
```

方法二：明確指定

```bash
mkdir -p /etc/systemd/journald.conf.d
cat > /etc/systemd/journald.conf.d/persistent.conf <<'EOF'
[Journal]
Storage=persistent
SystemMaxUse=1G
SystemKeepFree=2G
MaxRetentionSec=1month
EOF
systemctl restart systemd-journald
```

| 設定 | 說明 |
|---|---|
| `Storage=` | `volatile`（記憶體）、`persistent`（磁碟）、`auto`、`none` |
| `SystemMaxUse=` | 磁碟日誌最大佔用 |
| `SystemKeepFree=` | 磁碟至少保留的剩餘空間 |
| `SystemMaxFileSize=` | 單一日誌檔大小上限 |
| `RuntimeMaxUse=` | 記憶體日誌最大佔用 |
| `MaxRetentionSec=` | 最長保留時間 |
| `Compress=` | 是否壓縮（預設 yes） |
| `ForwardToSyslog=` | 是否轉送 rsyslog（RHEL 8 預設 yes） |

### 5.6 管理與清理

| 指令 | 說明 |
|---|---|
| `journalctl --disk-usage` | 日誌佔用空間 |
| `journalctl --vacuum-size=500M` | 縮減到 500M |
| `journalctl --vacuum-time=7d` | 只保留 7 天 |
| `journalctl --vacuum-files=5` | 只保留 5 個檔 |
| `journalctl --verify` | 檢查日誌完整性 |
| `journalctl --rotate` | 立即輪替日誌檔 |
| `journalctl --flush` | 把 `/run` 的日誌寫入 `/var/log/journal` |

### 5.7 與傳統 /var/log 的關係

RHEL 8 同時跑 `rsyslog`，journald 會轉送給它，所以傳統文字日誌仍存在：

| 檔案 | 內容 |
|---|---|
| `/var/log/messages` | 一般系統訊息 |
| `/var/log/secure` | 登入、sudo、SSH 認證 |
| `/var/log/cron` | cron 執行紀錄 |
| `/var/log/maillog` | 郵件 |
| `/var/log/boot.log` | 開機訊息 |
| `/var/log/dmesg` | 核心開機訊息 |
| `/var/log/audit/audit.log` | auditd / SELinux 紀錄 |

### 5.8 權限

預設只有 root 能看全部。其他使用者加入下列群組後可讀取：

```bash
usermod -aG systemd-journal <user>   # 讀所有日誌
usermod -aG adm <user>               # 同樣可讀
usermod -aG wheel <user>             # wheel 也可讀
```

### 5.9 把應用程式輸出寫進指定檔案（不走 journal）

```ini
[Service]
StandardOutput=append:/var/log/myapp/out.log
StandardError=append:/var/log/myapp/err.log
```

`append:` 為 systemd 240 才加入；RHEL 8 的 systemd 239 上游版本不支援（部分小版本可能有 backport，用前先測試），保險做法：

```ini
[Service]
StandardOutput=journal
SyslogIdentifier=myapp
```

再於 rsyslog 設定依 `$programname == 'myapp'` 分流到自訂檔案：

```bash
cat > /etc/rsyslog.d/myapp.conf <<'EOF'
if $programname == 'myapp' then /var/log/myapp.log
& stop
EOF
systemctl restart rsyslog
```
