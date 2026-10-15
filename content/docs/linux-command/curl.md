---
date: '2026-10-15T00:00:00+08:00'
draft: false
title: 'curl'
---

命令列 HTTP 工具，用於下載檔案與呼叫 API。

## 指令總覽

```bash
curl -O https://example.com/files/document.pdf
curl -L -o file.zip https://example.com/download
curl -I https://example.com
curl -u username -H "Content-Type: application/json" -d '{"key":"value"}' https://example.com/api
curl -H "Content-Type: application/json" --data-binary @config.json https://api.example.com/upload
curl -k https://self-signed.example.com
```

## 常用參數

| 參數 | 說明 |
|---|---|
| `-O` | 下載，使用遠端檔名儲存 |
| `-o 檔名` | 下載，自訂檔名 |
| `-L` | 跟隨轉址（redirect） |
| `-I` | 只取得回應標頭 |
| `-H` | 加入自訂標頭 |
| `-d` | 傳送資料（自動使用 POST） |
| `--data-binary @檔案` | 原樣傳送檔案內容 |
| `-u 帳號` | 帳密驗證，不寫密碼會提示輸入 |
| `-k` | 略過 SSL 憑證檢查（僅測試用） |
| `-v` | 顯示詳細過程 |

## `curl -O` / `curl -o`

下載檔案。`-O` 用遠端檔名，`-o` 自訂檔名；網址會轉址時加上 `-L`。

```bash
curl -O https://example.com/files/document.pdf
curl -L -o file.zip https://example.com/download
```

## `curl -I`

只看回應標頭，常用來檢查網站是否正常。

```text
HTTP/2 200
content-type: text/html
```

## `curl -d`

POST JSON。`-d` 已隱含 POST，不需再加 `-X POST`。

```bash
curl -u username -H "Content-Type: application/json" -d '{"key":"value"}' https://example.com/api
```

## `curl --data-binary`

POST 檔案內容。

```bash
curl -H "Content-Type: application/json" --data-binary @config.json https://api.example.com/upload
```

`-d @檔案` 會去掉換行，傳送檔案請用 `--data-binary`。

## `curl -k`

連線自簽憑證的網站時略過憑證檢查。

```bash
curl -k https://self-signed.example.com
```

{{< callout type="warning" >}}
`-k` 會失去連線的安全驗證，正式環境不要使用。`-u 帳號:密碼` 會留在 history，建議只寫帳號，讓 curl 提示輸入密碼。
{{< /callout >}}
