---
title: SourceTree 自定义 Git Actions
categories:
  - - Git
  - - 工具
tags:
  - - Git
  - - SourceTree
  - - Windows
date: 2026-09-27 14:10:00
---

本文配置 3 个常用的 SourceTree Custom Actions，用于在 Windows 下管理 Git 的 `assume-unchanged` 文件，并在文末附带 Windows 鼠标右键快速用 SourceTree 打开项目的配置。

## 一、功能说明

| Action | 功能 | 命令 |
|---|---|---|
| `Assume Unchanged` | 对选中的文件设置 `assume-unchanged` | `git update-index --assume-unchanged <file>` |
| `List Assume Unchanged` | 查看所有标记为 `assume-unchanged` 的文件 | `git ls-files -v \| findstr /b "h"` |
| `Restore All Assume Unchanged` | 一键恢复所有 `assume-unchanged` 文件 | `git update-index --no-assume-unchanged <file>` |

> **提示**：`assume-unchanged` 仅作用于本地 Git Index，只对已被 Git 跟踪的文件生效，不修改 `.gitignore`，不影响远程仓库。

---

## 二、脚本目录

统一存放目录：

```text
D:\tools\ASoureTreeActions\
├── git-assume-unchanged.bat
├── git-list-assume-unchanged.bat
└── git-restore-assume-unchanged.bat
```

---

## 三、Action 1：Assume Unchanged

### 1. BAT 脚本

文件：`D:\tools\ASoureTreeActions\git-assume-unchanged.bat`

```bat
@echo off
setlocal

if "%~1"=="" (
    echo No file selected.
    pause
    exit /b 1
)

git update-index --assume-unchanged -- "%~1"

if errorlevel 1 (
    echo Failed: %~1
    pause
    exit /b 1
)

echo Assume unchanged:
echo %~1
```

### 2. SourceTree 配置

路径：`Tools → Options → Custom Actions`

- **Menu Caption**: `Assume Unchanged`
- **Script to run**: `D:\tools\ASoureTreeActions\git-assume-unchanged.bat`
- **Parameters**: `"$FILE"`
- **选项**: 勾选 `Open in a separate window` 与 `Show Full Output`

### 3. 使用

在 SourceTree 文件列表中：`选中文件` → `右键/顶部 Actions` → `Custom Actions` → `Assume Unchanged`

---

## 四、Action 2：List Assume Unchanged

### 1. BAT 脚本

文件：`D:\tools\ASoureTreeActions\git-list-assume-unchanged.bat`

```bat
@echo off
setlocal

echo ========================================
echo Assume Unchanged Files
echo ========================================
echo.

git ls-files -v | findstr /b "h"

if errorlevel 1 (
    echo No assume-unchanged files.
)

echo.
echo ========================================
pause
```

### 2. SourceTree 配置

- **Menu Caption**: `List Assume Unchanged`
- **Script to run**: `D:\tools\ASoureTreeActions\git-list-assume-unchanged.bat`
- **Parameters**: 留空
- **选项**: 勾选 `Open in a separate window` 与 `Show Full Output`

### 3. 使用

`Actions` → `Custom Actions` → `List Assume Unchanged`

输出以 `h` 开头即为当前忽略的文件。

---

## 五、Action 3：Restore All Assume Unchanged

### 1. BAT 脚本

文件：`D:\tools\ASoureTreeActions\git-restore-assume-unchanged.bat`

```bat
@echo off
setlocal

echo ========================================
echo Restore Assume Unchanged Files
echo ========================================
echo.

for /f "tokens=1,*" %%a in ('git ls-files -v ^| findstr /b "h"') do (
    echo Restoring: %%b
    git update-index --no-assume-unchanged -- "%%b"
)

echo.
echo ========================================
echo Done.
echo ========================================
pause
```

### 2. SourceTree 配置

- **Menu Caption**: `Restore All Assume Unchanged`
- **Script to run**: `D:\tools\ASoureTreeActions\git-restore-assume-unchanged.bat`
- **Parameters**: 留空
- **选项**: 勾选 `Open in a separate window` 与 `Show Full Output`

### 3. 使用

`Actions` → `Custom Actions` → `Restore All Assume Unchanged`

---

## 六、三个 Action 总配置表

| Action | Script to run | Parameters | 窗口选项 |
|---|---|---|---|
| `Assume Unchanged` | `D:\tools\ASoureTreeActions\git-assume-unchanged.bat` | `"$FILE"` | 独立窗口打开、显示输出 |
| `List Assume Unchanged` | `D:\tools\ASoureTreeActions\git-list-assume-unchanged.bat` | *(留空)* | 独立窗口打开、显示输出 |
| `Restore All Assume Unchanged` | `D:\tools\ASoureTreeActions\git-restore-assume-unchanged.bat` | *(留空)* | 独立窗口打开、显示输出 |

---

## 七、直接导入 customactions.xml

关闭 SourceTree，打开配置文件：

```text
%LOCALAPPDATA%\Atlassian\SourceTree\customactions.xml
```

将以下 XML 内容写入或追加：

```xml
<?xml version="1.0" encoding="utf-8"?>
<ArrayOfCustomAction xmlns:xsd="http://www.w3.org/2001/XMLSchema"
                     xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <CustomAction>
    <Caption>Assume Unchanged</Caption>
    <OpenInSeparateWindow>true</OpenInSeparateWindow>
    <ShowFullOutput>true</ShowFullOutput>
    <IsSilent>false</IsSilent>
    <Target>D:\tools\ASoureTreeActions\git-assume-unchanged.bat</Target>
    <Parameters>"$FILE"</Parameters>
  </CustomAction>

  <CustomAction>
    <Caption>List Assume Unchanged</Caption>
    <OpenInSeparateWindow>true</OpenInSeparateWindow>
    <ShowFullOutput>true</ShowFullOutput>
    <IsSilent>false</IsSilent>
    <Target>D:\tools\ASoureTreeActions\git-list-assume-unchanged.bat</Target>
    <Parameters></Parameters>
  </CustomAction>

  <CustomAction>
    <Caption>Restore All Assume Unchanged</Caption>
    <OpenInSeparateWindow>true</OpenInSeparateWindow>
    <ShowFullOutput>true</ShowFullOutput>
    <IsSilent>false</IsSilent>
    <Target>D:\tools\ASoureTreeActions\git-restore-assume-unchanged.bat</Target>
    <Parameters></Parameters>
  </CustomAction>

</ArrayOfCustomAction>
```

---

## 八、日常使用流程

1. **临时忽略**：修改本地配置后，选中文件执行 `Assume Unchanged`，文件不再出现在修改列表中；
2. **查看清单**：执行 `List Assume Unchanged`，查看当前被忽略的文件；
3. **全部恢复**：更新或切换分支前，执行 `Restore All Assume Unchanged`，恢复正常 Git 追踪。

---

## 九、注意事项

1. **仅适用于已跟踪文件**：无法忽略未跟踪的新文件（新文件请用 `.gitignore`）；
2. **仅影响本地**：不产生 Commit，不修改远程，不影响其他团队成员；
3. **适合个人差异配置**：适合本地调试配置、环境差异文件，不要长期用于替代工程化的环境配置文件。

---

## 十、Windows 右键配置：使用 SourceTree 打开指定目录

在 Windows 资源管理器中右键任意文件夹或空白处，直接用 SourceTree 打开该仓库。

### 核心命令

```cmd
"你的SourceTree.exe路径" -f "%V"
```

> 其中 `-f` 指定仓库路径，`"%V"` 代表右键选中的目录。

### 一键注册表脚本（.reg）

新建 `OpenWithSourceTree.reg`，将以下内容保存后**双击导入**（注意替换你的 `SourceTree.exe` 实际路径，路径中的反斜杠需写成双反斜杠 `\\`）：

```reg
Windows Registry Editor Version 5.00

; 1. 文件夹上右键
[HKEY_CLASSES_ROOT\Directory\shell\SourceTree]
@="Open with SourceTree"
"Icon"="C:\\Users\\dong\\AppData\\Local\\SourceTree\\SourceTree.exe"

[HKEY_CLASSES_ROOT\Directory\shell\SourceTree\command]
@="\"C:\\Users\\dong\\AppData\\Local\\SourceTree\\SourceTree.exe\" -f \"%V\""

; 2. 文件夹内部空白处右键
[HKEY_CLASSES_ROOT\Directory\Background\shell\SourceTree]
@="Open with SourceTree"
"Icon"="C:\\Users\\dong\\AppData\\Local\\SourceTree\\SourceTree.exe"

[HKEY_CLASSES_ROOT\Directory\Background\shell\SourceTree\command]
@="\"C:\\Users\\dong\\AppData\\Local\\SourceTree\\SourceTree.exe\" -f \"%V\""
```
