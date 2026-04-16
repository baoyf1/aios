# 安装包制作功能文档

## 功能概述
安装包制作模块是Windows版本AIOS系统的重要组成部分，负责创建Windows安装包，包括WiX工具集成、安装向导、注册表配置和服务安装等功能。

## 核心功能

### 1. WiX工具集成

#### 1.1 安装包配置
```xml
<!-- WiX配置文件 (Product.wxs) -->
<?xml version="1.0" encoding="UTF-8"?>
<Wix xmlns="http://schemas.microsoft.com/wix/2006/wi">
    <Product Id="*" Name="AIOS" Version="1.0.0.0" Manufacturer="AIOS Team" Language="1033" Codepage="1252" UpgradeCode="YOUR-UPGRADE-CODE-HERE">
        <Package InstallerVersion="200" Compressed="yes" InstallScope="perMachine" />
        
        <MajorUpgrade DowngradeErrorMessage="A newer version of [ProductName] is already installed." />
        <MediaTemplate />
        
        <Feature Id="ProductFeature" Title="AIOS" Level="1">
            <ComponentGroupRef Id="ProductComponents" />
        </Feature>
        
        <!-- 安装条件 -->
        <Condition Message="This application requires Windows 10 or later.">
            <![CDATA[VersionNT >= 601]]>
        </Condition>
    </Product>
    
    <Fragment>
        <Directory Id="TARGETDIR" Name="SourceDir">
            <Directory Id="ProgramFilesFolder">
                <Directory Id="INSTALLFOLDER" Name="AIOS" />
            </Directory>
        </Directory>
    </Fragment>
    
    <Fragment>
        <ComponentGroup Id="ProductComponents" Directory="INSTALLFOLDER">
            <!-- 组件定义 -->
        </ComponentGroup>
    </Fragment>
</Wix>
```

#### 1.2 构建脚本
```powershell
# 构建安装包的PowerShell脚本

# 设置变量
$WiXPath = "C:\Program Files (x86)\WiX Toolset v3.11\bin"
$ProjectDir = "$(pwd)"
$OutputDir = "$ProjectDir\output"
$ProductWxs = "$ProjectDir\Product.wxs"
$ProductWixobj = "$OutputDir\Product.wixobj"
$MsiFile = "$OutputDir\AIOS.msi"

# 创建输出目录
if (!(Test-Path $OutputDir)) {
    New-Item -ItemType Directory -Path $OutputDir | Out-Null
}

# 运行candle.exe编译WiX文件
& "$WiXPath\candle.exe" -out $ProductWixobj $ProductWxs

# 运行light.exe链接生成MSI
& "$WiXPath\light.exe" -out $MsiFile $ProductWixobj

Write-Host "安装包构建完成: $MsiFile"
```

### 2. 安装向导

#### 2.1 自定义安装界面
```xml
<!-- 自定义安装界面配置 -->
<UI>
    <UIRef Id="WixUI_Mondo" />
    <UIRef Id="WixUI_ErrorProgressText" />
    
    <!-- 自定义对话框 -->
    <Dialog Id="CustomWelcomeDlg" Width="370" Height="270" Title="[ProductName] Setup">
        <Control Id="Next" Type="PushButton" X="236" Y="243" Width="56" Height="17" Default="yes" Text="&Next">
            <Publish Event="NewDialog" Value="CustomInstallDirDlg">1</Publish>
        </Control>
        <Control Id="Cancel" Type="PushButton" X="304" Y="243" Width="56" Height="17" Cancel="yes" Text="&Cancel">
            <Publish Event="SpawnDialog" Value="CancelDlg">1</Publish>
        </Control>
        <Control Id="Bitmap" Type="Bitmap" X="0" Y="0" Width="370" Height="234" TabSkip="no" Text="WixUI_Bmp_Dialog" />
        <Control Id="WelcomeText" Type="Text" X="135" Y="70" Width="220" Height="60" Transparent="yes" NoPrefix="yes" Text="Welcome to the [ProductName] setup wizard. This wizard will guide you through the installation of [ProductName]." />
        <Control Id="Description" Type="Text" X="135" Y="150" Width="220" Height="60" Transparent="yes" NoPrefix="yes" Text="[ProductName] is an AI Agent Operating System for commercial vertical domains." />
        <Control Id="BottomLine" Type="Line" X="0" Y="234" Width="370" Height="0" />
        <Control Id="Title" Type="Text" X="135" Y="20" Width="220" Height="60" Transparent="yes" NoPrefix="yes" Text="{
#ifdef ARPHELPTEXT
[ARPHELPTEXT]
#else
Welcome to the [ProductName] Setup Wizard
#endif
}" />
    </Dialog>
</UI>
```

#### 2.2 安装进度显示
```xml
<!-- 安装进度对话框 -->
<Dialog Id="ProgressDlg" Width="370" Height="270" Title="[ProductName] Setup">
    <Control Id="Next" Type="PushButton" X="236" Y="243" Width="56" Height="17" Default="yes" Text="&Next">
        <Publish Event="EndDialog" Value="Return" Order="999">1</Publish>
    </Control>
    <Control Id="Cancel" Type="PushButton" X="304" Y="243" Width="56" Height="17" Cancel="yes" Text="&Cancel">
        <Publish Event="SpawnDialog" Value="CancelDlg">1</Publish>
    </Control>
    <Control Id="ActionText" Type="Text" X="20" Y="60" Width="330" Height="15" Transparent="yes" NoPrefix="yes" Text="[ActionText]" />
    <Control Id="ProgressBar" Type="ProgressBar" X="20" Y="80" Width="330" Height="15" ProgressBlocks="yes" />
    <Control Id="BottomLine" Type="Line" X="0" Y="234" Width="370" Height="0" />
    <Control Id="Title" Type="Text" X="135" Y="20" Width="220" Height="60" Transparent="yes" NoPrefix="yes" Text="Installing [ProductName]..." />
</Dialog>
```

### 3. 注册表配置

#### 3.1 注册表项配置
```xml
<!-- 注册表配置 -->
<Component Id="RegistryComponent" Guid="YOUR-GUID-HERE" Directory="INSTALLFOLDER">
    <RegistryKey Root="HKLM" Key="Software\AIOS" Action="createAndRemoveOnUninstall">
        <RegistryValue Type="string" Name="InstallPath" Value="[INSTALLFOLDER]" KeyPath="yes" />
        <RegistryValue Type="string" Name="Version" Value="[ProductVersion]" />
        <RegistryValue Type="string" Name="InstallDate" Value="[Date]" />
    </RegistryKey>
    <RegistryKey Root="HKLM" Key="Software\AIOS\Settings">
        <RegistryValue Type="string" Name="ApiPort" Value="8000" />
        <RegistryValue Type="string" Name="LogLevel" Value="Info" />
    </RegistryKey>
</Component>
```

### 4. 服务安装

#### 4.1 服务配置
```xml
<!-- 服务安装配置 -->
<Component Id="ServiceComponent" Guid="YOUR-GUID-HERE" Directory="INSTALLFOLDER">
    <File Id="ServiceExe" Source="$(var.ServiceExePath)" KeyPath="yes" />
    <ServiceInstall Id="AIOSService" Name="AIOSService" DisplayName="AIOS Service" Description="AIOS System Service" Start="auto" ErrorControl="normal" Type="ownProcess">
        <ServiceDependency Id="Tcpip" />
    </ServiceInstall>
    <ServiceControl Id="AIOSService" Name="AIOSService" Start="install" Stop="both" Remove="uninstall" Wait="yes" />
</Component>
```

## 技术实现

### 1. 架构设计
- **分层设计**：配置层、构建层、安装层
- **模块化**：安装组件作为独立模块
- **配置驱动**：基于WiX配置文件

### 2. 核心组件
- **WiX配置**：定义安装包结构和行为
- **构建工具**：编译和链接安装包
- **安装向导**：引导用户完成安装
- **服务安装**：配置和安装Windows服务
- **注册表配置**：设置系统注册表项

### 3. 数据流
1. **配置准备**：准备WiX配置文件
2. **构建过程**：使用WiX工具编译和链接
3. **安装过程**：用户通过向导完成安装
4. **服务配置**：安装和配置Windows服务
5. **注册表设置**：配置系统注册表
6. **安装完成**：完成安装并启动服务

## 配置选项

### 1. 安装包配置
```xml
<!-- 产品配置 -->
<Product Id="*" Name="AIOS" Version="1.0.0.0" Manufacturer="AIOS Team" Language="1033" Codepage="1252" UpgradeCode="YOUR-UPGRADE-CODE-HERE">
    <Package InstallerVersion="200" Compressed="yes" InstallScope="perMachine" />
    <MajorUpgrade DowngradeErrorMessage="A newer version of [ProductName] is already installed." />
    <MediaTemplate />
    <!-- 其他配置 -->
</Product>
```

### 2. 组件配置
```xml
<!-- 组件配置 -->
<ComponentGroup Id="ProductComponents" Directory="INSTALLFOLDER">
    <Component Id="MainExecutable" Guid="YOUR-GUID-HERE">
        <File Id="AIOS.exe" Source="$(var.AIOSExePath)" KeyPath="yes" />
    </Component>
    <Component Id="ConfigFile" Guid="YOUR-GUID-HERE">
        <File Id="config.yaml" Source="$(var.ConfigPath)" />
    </Component>
    <!-- 其他组件 -->
</ComponentGroup>
```

### 3. 安装条件
```xml
<!-- 安装条件 -->
<Condition Message="This application requires Windows 10 or later.">
    <![CDATA[VersionNT >= 601]]>
</Condition>
<Condition Message="This application requires .NET Framework 4.7.2 or later.">
    <![CDATA[NETFRAMEWORK472 >= '#394806']]>
</Condition>
```

## 错误处理

### 1. 构建错误处理
```powershell
# 构建脚本中的错误处理
try {
    # 运行candle.exe编译WiX文件
    & "$WiXPath\candle.exe" -out $ProductWixobj $ProductWxs
    if ($LASTEXITCODE -ne 0) {
        throw "Candle.exe failed with exit code $LASTEXITCODE"
    }
    
    # 运行light.exe链接生成MSI
    & "$WiXPath\light.exe" -out $MsiFile $ProductWixobj
    if ($LASTEXITCODE -ne 0) {
        throw "Light.exe failed with exit code $LASTEXITCODE"
    }
    
    Write-Host "安装包构建成功: $MsiFile"
} catch {
    Write-Host "构建失败: $($_.Exception.Message)" -ForegroundColor Red
    exit 1
}
```

### 2. 安装错误处理
```xml
<!-- 安装错误处理 -->
<CustomAction Id="RollbackInstall" Execute="rollback" BinaryKey="WixCA" DllEntry="CAQuietExecRollback" Return="ignore" />
<CustomAction Id="CleanupOnError" Execute="onerror" BinaryKey="WixCA" DllEntry="CAQuietExec" Return="ignore">
    <![CDATA[cmd /c "echo Installation failed, cleaning up... >> %TEMP%\AIOSInstall.log"]]>
</CustomAction>
```

## 性能优化

### 1. 安装包大小优化
- 压缩安装包
- 仅包含必要文件
- 使用增量更新

### 2. 安装速度优化
- 并行安装组件
- 减少文件复制时间
- 优化服务安装过程

### 3. 资源使用优化
- 减少内存使用
- 优化磁盘I/O
- 合理使用系统资源

## 安全考虑

### 1. 安装包签名
```powershell
# 签名安装包
$CertificateThumbprint = "YOUR-CERTIFICATE-THUMBPRINT"
$TimestampServer = "http://timestamp.digicert.com"

& "C:\Program Files (x86)\Windows Kits\10\bin\10.0.19041.0\x64\signtool.exe" sign /f "$CertificatePath" /t $TimestampServer /v $MsiFile
```

### 2. 权限控制
```xml
<!-- 权限控制 -->
<DirectoryRef Id="INSTALLFOLDER">
    <Component Id="Permissions" Guid="YOUR-GUID-HERE">
        <CreateFolder>
            <Permission User="Everyone" GenericAll="yes" />
        </CreateFolder>
    </Component>
</DirectoryRef>
```

### 3. 安全验证
- 验证安装包完整性
- 检查数字签名
- 防止恶意修改

## 集成与扩展

### 1. 与其他模块的集成
- 与依赖管理模块集成，确保所有依赖项被包含
- 与自动更新模块集成，支持更新检测
- 与配置管理模块集成，设置初始配置

### 2. 扩展点
- 自定义安装界面
- 自定义安装行为
- 插件和扩展安装
- 多语言支持