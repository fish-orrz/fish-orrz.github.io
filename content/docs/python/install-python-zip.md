---
date: '2026-10-15T08:23:38+08:00'
draft: false
title: '手動安裝 Python(zip 免安裝版)'
---

不使用安裝程式,下載 zip 版 Python 解壓縮後自行配置環境,並安裝 pip。

## 指令總覽

```powershell
python --version
python get-pip.py
pip --version
python -m pip install --upgrade pip
```

## 步驟

### 1. 下載並解壓縮 Python

到 <https://www.python.org/downloads/windows/> 下載 **Windows embeddable package (64-bit)**,解壓縮到 `D:\Python`。

### 2. 設定環境變數 PATH

在系統環境變數 `Path` 新增:

```
D:\Python
D:\Python\Scripts
```

### 3. python --version

重新開啟 PowerShell,確認 Python 可執行。

```powershell
python --version
```

```
Python 3.10.11
```

### 4. 編輯 python310._pth

用文字編輯器開啟 `D:\Python\python310._pth`,新增 `Lib\site-packages`,並取消 `import site` 前面的 `#`。

檔名會依版本而不同,例如 3.11 為 `python311._pth`。

```
python310.zip
.
Lib\site-packages

import site
```

### 5. 下載 get-pip.py

下載 <https://bootstrap.pypa.io/get-pip.py>,放到 `D:\Python`。

### 6. python get-pip.py

安裝 pip,完成後 `D:\Python` 會多出 `Lib` 與 `Scripts` 目錄。

```powershell
cd D:\Python
python get-pip.py
```

```
Collecting pip
Installing collected packages: pip
Successfully installed pip-25.0
```

### 7. pip --version

確認 pip 可使用。

```powershell
pip --version
```

```
pip 25.0 from D:\Python\Lib\site-packages\pip (python 3.10)
```

### 8. python -m pip install --upgrade pip

之後要更新 pip 時使用。已是最新版會顯示 `Requirement already satisfied`。

```powershell
python -m pip install --upgrade pip
```

```
Requirement already satisfied: pip in d:\python\lib\site-packages (25.0)
```