---
title: Windows系统常用开发环境配
categories:
  - - android
  - - android studio
  - - flutter
tags:
  - - Windows
  - - PowerShell
  - - 环境配置
  - - android
date: 2026-02-06 14:59:55
---

> **前言**：换新电脑或重装系统后的环境配置总是繁琐且耗时。本文记录了基于 `winget` 快速部署常用软件，并通过 PowerShell 脚本实现 Java、Flutter、Node.js 多版本切换及代理管理的方案。

## 常用软件安装

在开始安装前，请以 **管理员模式** 运行 `PowerShell`，执行以下初始化命令以解锁脚本执行权限和注册包管理器。

```powershell
#允许执行自定义的脚本
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
#允许执行远程签名的脚本
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned

#winget 注册
Add-AppxPackage -RegisterByFamilyName -MainPackage Microsoft.DesktopAppInstaller_8wekyb3d8bbwe
```





### 卸载预装的软件

```powershell
winget uninstall Microsoft.BingWeather_8wekyb3d8bbwe
winget uninstall Microsoft.BingNews_8wekyb3d8bbwe
winget uninstall Microsoft.People_8wekyb3d8bbwe
winget uninstall Microsoft.MicrosoftSolitaireCollection_8wekyb3d8bbwe
winget uninstall Microsoft.MinecraftEducationEdition_8wekyb3d8bbwe
```

### choco

```powershell
winget install --id=Chocolatey.Chocolatey -e 
$env:chocolateyUseWindowsCompression = 'true'
```

#### 常用软件

```powershell
#安装浏览器
#winget install Google.Chrome

#压缩软件
winget install --id=Bandisoft.Bandizip -e 

# 微信
#winget install Tencent.WeChat

# 用于自定义 Windows 的实用工具
winget install Microsoft.PowerToys

# 看图
winget install xnviewmp

# 视频播放器
winget install Daum.PotPlayer

# 网易云音乐
winget install NetEase.CloudMusic

# notion
winget install --id=Notion.Notion -e 
```



#### 常用的开发软件

```powershell
#Git
winget install Git.Git	

#配置代理，新的终端窗口执行，为了等环境变量生效
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890

#VScode
winget install Microsoft.VisualStudioCode

#sourcetree
winget install sourcetree

#Toolbox
winget install --id=JetBrains.Toolbox -i 

```



## 环境变量



## java 多版本管理

自己新建`JAVA_HOME11`、`JAVA_HOME17`的环境变量值，利用下面的脚本改变`JAVA_HOME`指向不同的环境变量。最终`jdk11`指令切换到java11、jdk17切换到java17

需要利用 **PowerShell Profile（配置文件）**每次启动都自动加载这些脚本。

`jdk11`

```powershell
[System.Environment]::SetEnvironmentVariable("JAVA_HOME11", "D:\Java\jdk-11", "User")
```

`jdk17`

```powershell
[System.Environment]::SetEnvironmentVariable("JAVA_HOME17", "D:\Java\jdk-17", "User")
```



**检查配置文件是否存在**：
在终端输入以下命令，如果没有该文件，它会自动创建一个；如果有，则会直接打开：

```powershell
if (!(Test-Path -Path $PROFILE)) { New-Item -ItemType File -Path $PROFILE -Force }
notepad $PROFILE
```

打开配置文件

```powershell
notepad $PROFILE
```

配置导入自定义的脚本

```powershell
# 将脚本里的函数导入当前会话 . 空格要注意
. "D:\tools\JavaVersionSwitcher.ps1"
. "D:\tools\proxy.ps1"
```



编写切换脚本`JavaVersionSwitcher.ps1`

```PowerShell
# JavaVersionSwitcher.ps1 - 独立的Java版本切换脚本
# 切换到JDK 11的函数
function jdk11 {
    # 读取系统级环境变量JAVA_HOME11（若为用户级，改为"User"）
    $jdk11Path = [Environment]::GetEnvironmentVariable("JAVA_HOME11", "User")
    if (-not $jdk11Path) {
        Write-Host "错误：未找到系统环境变量 JAVA_HOME11，请检查配置！" -ForegroundColor Red
        return
    }

    # 设置系统级永久环境变量JAVA_HOME（若为用户级，改为"User"）
    [Environment]::SetEnvironmentVariable("JAVA_HOME", $jdk11Path, "User")

    # 刷新当前会话的环境变量
    $env:JAVA_HOME = $jdk11Path
    $newPath = $env:Path -split ';' | Where-Object {
        $_ -notlike "*Java*" -and $_ -notlike "*jdk*" -and $_ -ne ""
    }
    $env:Path = "$jdk11Path\bin;" + ($newPath -join ';')

    # 验证版本
    Write-Host "`n✅ 已将JAVA_HOME指向JAVA_HOME11，当前Java版本：" -ForegroundColor Green
    java -version
}

# 切换到JDK 17的函数
function jdk17 {
    # 读取系统级环境变量JAVA_HOME17（若为用户级，改为"User"）
    $jdk17Path = [Environment]::GetEnvironmentVariable("JAVA_HOME17", "User")
    if (-not $jdk17Path) {
        Write-Host "错误：未找到系统环境变量 JAVA_HOME17，请检查配置！" -ForegroundColor Red
        return
    }

    # 设置系统级永久环境变量JAVA_HOME（若为用户级，改为"User"）
    [Environment]::SetEnvironmentVariable("JAVA_HOME", $jdk17Path, "User")

    # 刷新当前会话的环境变量
    $env:JAVA_HOME = $jdk17Path
    $newPath = $env:Path -split ';' | Where-Object {
        $_ -notlike "*Java*" -and $_ -notlike "*jdk*" -and $_ -ne ""
    }
    $env:Path = "$jdk17Path\bin;" + ($newPath -join ';')

    # 验证版本
    Write-Host "`n✅ 已将JAVA_HOME指向JAVA_HOME17，当前Java版本：" -ForegroundColor Green
    java -version
}
```

## Android 开发环境

为了保护 C 盘，建议将 SDK 和缓存移动到数据盘。

```powershell
# GRADLE_USER_HOME
[System.Environment]::SetEnvironmentVariable("GRADLE_USER_HOME", "D:\.gradle", "User")

# 设置 SDK 安装目录的路径
[System.Environment]::SetEnvironmentVariable("ANDROID_HOME", "D:\Android\Sdk", "User")

# 改变  ANDROID_AVD_HOME 
[System.Environment]::SetEnvironmentVariable("ANDROID_AVD_HOME", "D:\Android\avd", "User") 
```



## Flutter版本管理

使用`fvm`管理

```
fvm config --cache-path D:\FlutterSdk
[System.Environment]::SetEnvironmentVariable("FVM_CACHE_PATH", "D:\FlutterSdk", "User")
fvm use 3.3.10
```

国内镜像加速配置

```powershell
[System.Environment]::SetEnvironmentVariable("FLUTTER_STORAGE_BASE_URL", "https://storage.flutter-io.cn", "User")
[System.Environment]::SetEnvironmentVariable("PUB_HOSTED_URL", "https://pub.flutter-io.cn", "User")
```



## node版本管理

使用`nvm` 管理,先安装

```
winget install -i --id=CoreyButler.NVMforWindows
```

环境比变灵设置

```
# =========NVM 
[System.Environment]::SetEnvironmentVariable("NVM_HOME", "D:\nvm", "User")
[System.Environment]::SetEnvironmentVariable("NVM_SYMLINK", "D:\nodejs", "User")
```



## 代理配置

同Java提供开启和关闭的函数

`proxy_on` 开启

`proxy_off` 关闭

```powershell
# proxy.ps1 - 简洁版代理管理脚本
# 仅设置/取消环境变量代理

# 开启代理 7890端口
function proxy_on {
    [CmdletBinding()]
    param(
        [string]$ProxyServer = "http://127.0.0.1:7890"
    )
    
    # 设置当前会话的环境变量
    $Env:http_proxy = $ProxyServer
    $Env:https_proxy = $ProxyServer
    
    
    Write-Host "Proxy enabled: $ProxyServer" -ForegroundColor Green
    
    # 验证设置
    Write-Host "Current proxy settings:" -ForegroundColor Cyan
    Write-Host "  http_proxy:  $Env:http_proxy" -ForegroundColor Gray
    Write-Host "  https_proxy: $Env:https_proxy" -ForegroundColor Gray
}

# 关闭代理
function proxy_off {
    # 清除当前会话的环境变量
    $Env:http_proxy = $null
    $Env:https_proxy = $null
    
    
    Write-Host "Proxy disabled" -ForegroundColor Green
    
    # 验证清除
    Write-Host "Current proxy settings:" -ForegroundColor Cyan
    Write-Host "  http_proxy:  " -NoNewline -ForegroundColor Gray
    if ($Env:http_proxy) {
        Write-Host $Env:http_proxy -ForegroundColor Yellow
    } else {
        Write-Host "(not set)" -ForegroundColor Gray
    }
    
    Write-Host "  https_proxy: " -NoNewline -ForegroundColor Gray
    if ($Env:https_proxy) {
        Write-Host $Env:https_proxy -ForegroundColor Yellow
    } else {
        Write-Host "(not set)" -ForegroundColor Gray
    }
}

# 显示当前代理状态
function proxy_status {
    Write-Host "Proxy Status" -ForegroundColor Cyan
    Write-Host "============" -ForegroundColor DarkGray
    
    Write-Host "Current session:" -ForegroundColor Yellow
    Write-Host "  http_proxy:  " -NoNewline
    if ($Env:http_proxy) {
        Write-Host $Env:http_proxy -ForegroundColor Green
    } else {
        Write-Host "(not set)" -ForegroundColor Gray
    }
    
    Write-Host "  https_proxy: " -NoNewline
    if ($Env:https_proxy) {
        Write-Host $Env:https_proxy -ForegroundColor Green
    } else {
        Write-Host "(not set)" -ForegroundColor Gray
    }
    
    Write-Host "`nPermanent settings (User):" -ForegroundColor Yellow
    $userHttpProxy = [Environment]::GetEnvironmentVariable("http_proxy", "User")
    $userHttpsProxy = [Environment]::GetEnvironmentVariable("https_proxy", "User")
    
    Write-Host "  http_proxy:  " -NoNewline
    if ($userHttpProxy) {
        Write-Host $userHttpProxy -ForegroundColor Green
    } else {
        Write-Host "(not set)" -ForegroundColor Gray
    }
    
    Write-Host "  https_proxy: " -NoNewline
    if ($userHttpsProxy) {
        Write-Host $userHttpsProxy -ForegroundColor Green
    } else {
        Write-Host "(not set)" -ForegroundColor Gray
    }
}

# 如果直接运行脚本，显示使用说明
if ($MyInvocation.InvocationName -ne '.') {
    Write-Host "`nProxy Management Script" -ForegroundColor Yellow
    Write-Host "=======================" -ForegroundColor DarkGray
    Write-Host "Load script first, then use functions:" -ForegroundColor Cyan
    Write-Host "  . .\proxy.ps1                   # Load script" -ForegroundColor Gray
    Write-Host "  proxy_on                        # Enable proxy" -ForegroundColor Gray
    Write-Host "  proxy_off                       # Disable proxy" -ForegroundColor Gray
    Write-Host "  proxy_status                    # Show status" -ForegroundColor Gray
    Write-Host ""
    Write-Host "Optional parameters:" -ForegroundColor Cyan
    Write-Host "  proxy_on -ProxyServer 'http://localhost:8080'" -ForegroundColor Gray
}
```



## Path配置

最后，运行此脚本一次性将所有开发路径注入系统 `Path`，保存为一个ps1脚本执行吧。

```powershell
# ========================= Path 追加====================
# 定义要设置的环境变量路径
$envPaths = @(
    "${env:JAVA_HOME}\bin",
    "${env:ANDROID_HOME}\platform-tools",
    "${env:ANDROID_HOME}\tools",
    "${env:ANDROID_HOME}\tools\bin",
    "$env:NVM_HOME",
    "$env:NVM_SYMLINK",
    "$env:FVM_CACHE_PATH",
    "${env:FVM_CACHE_PATH}\default\bin"
)

# 获取当前用户的Path环境变量
$envPath = [System.Environment]::GetEnvironmentVariable("Path", "User")

# 添加新路径到用户的Path环境变量中
foreach ($path in $envPaths) {
    if (-not ($envPath -split ";" | Select-String -Pattern "^$([regex]::Escape($path))\$")) {
        $envPath += ";$path"
    }
}

# 设置更新后的Path环境变量到用户级别
[System.Environment]::SetEnvironmentVariable("Path", $envPath, "User")

# 输出设置完成信息
Write-Output "环境变量设置完成"
```

