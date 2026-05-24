@echo off
:: ===========================================================================
::
::   KARIUSOPTIMIZER v2.1 (English Edition)
::   Windows 10 / 11 Advanced Performance Suite
::   Author  : Karius
::   Fixed   : Encoding crash, WMIC fallbacks, immediate-close bug, Registry loop
::
::   ARCHITECTURE (Future GUI Map):
::   [MODULE]  -> UI Panel/Page   [FUNC]   -> Service/Utility
::   [GUARD]   -> Middleware      [DIAG]   -> Read-only diagnostic
::   [CLEANUP] -> Destructor      [LOG]    -> Logging service
::
:: ===========================================================================

:: ---------------------------------------------------------------------------
::  GLOBAL SETUP
::  NOTE: chcp 65001 is intentionally REMOVED - it caused UTF-8 encoding
::  conflicts with batch echo statements on most Windows configurations,
::  resulting in immediate window closure. CP437 (default OEM) is used
::  instead, which natively supports all box-drawing characters below.
:: ---------------------------------------------------------------------------
setlocal EnableDelayedExpansion EnableExtensions

:: Console window appearance
mode con cols=72 lines=45
title  KariusOptimizer v2.1  ^|  Windows Performance Suite

:: ---------------------------------------------------------------------------
::  [FUNC] Global Constants
::  Centralised — maps directly to config file / app settings in future GUI
:: ---------------------------------------------------------------------------
set "APP_NAME=KariusOptimizer"
set "APP_VER=2.1.0"
set "APP_BUILD=2025"
set "LOG_FILE=%USERPROFILE%\Desktop\KariusOptimizer_Log.txt"
set "ULTIMATE_GUID=e9a42b02-d5df-448d-aa00-03f14749eb61"
set "MODULES_RAN=0"
set "RESTORE_DONE=NO"

:: ---------------------------------------------------------------------------
::  [GUARD] Administrator privilege check
::  Strategy: attempt to read a protected registry key.
::  If ERRORLEVEL != 0 the process token lacks elevation -> abort gracefully.
:: ---------------------------------------------------------------------------
:CHECK_ADMIN
    color 0F
    cls
    net session >nul 2>&1
    if %ERRORLEVEL% NEQ 0 (
        color 4F
        cls
        echo.
        echo  +------------------------------------------------------------------+
        echo  ^|                                                                  ^|
        echo  ^|   [ ACCESS DENIED ]  Administrator rights required.             ^|
        echo  ^|                                                                  ^|
        echo  ^|   HOW TO FIX:                                                    ^|
        echo  ^|   1. Close this window                                          ^|
        echo  ^|   2. Right-click KariusOptimizer.bat                            ^|
        echo  ^|   3. Select "Run as administrator"                              ^|
        echo  ^|   4. Click YES on the UAC prompt                                ^|
        echo  ^|                                                                  ^|
        echo  +------------------------------------------------------------------+
        echo.
        echo  Press any key to exit...
        pause >nul
        exit /b 1
    )
    goto :INIT

:: ---------------------------------------------------------------------------
::  [FUNC] Initialise log file for this session
::  Always called after admin check. Creates fresh log on Desktop.
:: ---------------------------------------------------------------------------
:INIT
    :: Write log header
    (
        echo ================================================================
        echo  %APP_NAME% v%APP_VER% -- Session Log
        echo  Date    : %DATE%
        echo  Time    : %TIME%
        echo  User    : %USERNAME%
        echo  Machine : %COMPUTERNAME%
        echo ================================================================
        echo.
    ) > "%LOG_FILE%" 2>nul

    :: Collect system info silently before splash
    call :COLLECT_SYSINFO
    goto :SPLASH

:: ---------------------------------------------------------------------------
::  [DIAG] Collect hardware and OS info into variables
::  Uses PowerShell as primary (works on all Win10/11 including WMIC-less).
::  Falls back gracefully if PS is restricted.
:: ---------------------------------------------------------------------------
:COLLECT_SYSINFO
    :: OS info via PowerShell (WMIC deprecated on Win11 22H2+)
    for /f "usebackq delims=" %%A in (
        `powershell -NoProfile -Command "(Get-WmiObject Win32_OperatingSystem).Caption" 2^>nul`
    ) do set "SYS_OS=%%A"

    for /f "usebackq delims=" %%A in (
        `powershell -NoProfile -Command "(Get-WmiObject Win32_OperatingSystem).BuildNumber" 2^>nul`
    ) do set "SYS_BUILD=%%A"

    :: CPU
    for /f "usebackq delims=" %%A in (
        `powershell -NoProfile -Command "(Get-WmiObject Win32_Processor).Name" 2^>nul`
    ) do set "SYS_CPU=%%A"

    for /f "usebackq delims=" %%A in (
        `powershell -NoProfile -Command "(Get-WmiObject Win32_Processor).NumberOfCores" 2^>nul`
    ) do set "SYS_CORES=%%A"

    for /f "usebackq delims=" %%A in (
        `powershell -NoProfile -Command "(Get-WmiObject Win32_Processor).NumberOfLogicalProcessors" 2^>nul`
    ) do set "SYS_THREADS=%%A"

    :: RAM (total visible in GB)
    for /f "usebackq delims=" %%A in (
        `powershell -NoProfile -Command "[math]::Round((Get-WmiObject Win32_OperatingSystem).TotalVisibleMemorySize/1MB,1)" 2^>nul`
    ) do set "SYS_RAM_GB=%%A"

    for /f "usebackq delims=" %%A in (
        `powershell -NoProfile -Command "[math]::Round((Get-WmiObject Win32_OperatingSystem).FreePhysicalMemory/1MB,1)" 2^>nul`
    ) do set "SYS_RAM_FREE=%%A"

    :: GPU
    for /f "usebackq delims=" %%A in (
        `powershell -NoProfile -Command "(Get-WmiObject Win32_VideoController | Select-Object -First 1).Name" 2^>nul`
    ) do set "SYS_GPU=%%A"

    :: Disk C: free
    for /f "usebackq delims=" %%A in (
        `powershell -NoProfile -Command "[math]::Round((Get-PSDrive C).Free/1GB,0)" 2^>nul`
    ) do set "SYS_DISK_FREE=%%A"

    for /f "usebackq delims=" %%A in (
        `powershell -NoProfile -Command "[math]::Round(((Get-PSDrive C).Used+(Get-PSDrive C).Free)/1GB,0)" 2^>nul`
    ) do set "SYS_DISK_TOTAL=%%A"

    :: Active power plan name
    for /f "tokens=4*" %%A in ('powercfg /getactivescheme 2^>nul') do set "SYS_POWER=%%B"

    :: Set defaults if PS failed or restricted
    if not defined SYS_OS      set "SYS_OS=Windows (detection failed)"
    if not defined SYS_BUILD   set "SYS_BUILD=Unknown"
    if not defined SYS_CPU     set "SYS_CPU=Unknown CPU"
    if not defined SYS_CORES   set "SYS_CORES=?"
    if not defined SYS_THREADS set "SYS_THREADS=?"
    if not defined SYS_RAM_GB  set "SYS_RAM_GB=?"
    if not defined SYS_GPU     set "SYS_GPU=Unknown GPU"
    if not defined SYS_DISK_FREE  set "SYS_DISK_FREE=?"
    if not defined SYS_DISK_TOTAL set "SYS_DISK_TOTAL=?"
    if not defined SYS_POWER   set "SYS_POWER=(unknown)"

    echo [SYSINFO] OS: %SYS_OS% Build %SYS_BUILD% >> "%LOG_FILE%" 2>nul
    echo [SYSINFO] CPU: %SYS_CPU% ^| GPU: %SYS_GPU% >> "%LOG_FILE%" 2>nul
    echo [SYSINFO] RAM: %SYS_RAM_GB%GB total, %SYS_RAM_FREE%GB free >> "%LOG_FILE%" 2>nul
    exit /b 0

:: ---------------------------------------------------------------------------
::  SPLASH SCREEN
::  Simulated loading bar using timeout.
::  Pure ASCII - no encoding issues.
:: ---------------------------------------------------------------------------
:SPLASH
    color 0A
    cls
    echo.
    echo.
    echo.
    echo.
    echo        ##    ##    ###    ######   ####  ##    ##  ######
    echo        ##   ##    ## ##   ##   ## ##  ## ##    ## ##
    echo        #####    ##   ##  ######  ##  ## ##    ##  ######
    echo        ##  ##   #######  ##   ## ##  ## ##    ##       ##
    echo        ##   ##  ##   ##  ##   ##  ####   ######  ######
    echo.
    echo            O P T I M I Z E R   v 2 . 1   [ 2025 ]
    echo        ================================================
    echo           Windows 10 / 11  Advanced Performance Suite
    echo.
    echo.
    <nul set /p "=        Loading  ["
    <nul set /p "="
    timeout /t 1 /nobreak >nul
    <nul set /p "=######"
    timeout /t 1 /nobreak >nul
    <nul set /p "=######"
    timeout /t 1 /nobreak >nul
    <nul set /p "=######"
    timeout /t 1 /nobreak >nul
    echo =]  Ready.
    echo.
    timeout /t 1 /nobreak >nul
    goto :DISCLAIMER

:: ---------------------------------------------------------------------------
::  DISCLAIMER SCREEN
::  Must be explicitly accepted.
::  N -> EXIT_CLEAN (no changes made).
:: ---------------------------------------------------------------------------
:DISCLAIMER
    color 0E
    cls
    echo.
    echo  +==================================================================+
    echo  ^|        K A R I U S O P T I M I Z E R   v 2 . 1                  ^|
    echo  +==================================================================+
    echo  ^|                                                                  ^|
    echo  ^|  DISCLAIMER - Please read carefully before continuing             ^|
    echo  ^|                                                                  ^|
    echo  ^|  This script will modify the following system configurations:     ^|
    echo  ^|  - Registry values and system parameters                        ^|
    echo  ^|  - Power management profiles                                     ^|
    echo  ^|  - Temporary file directories and caches                         ^|
    echo  ^|  - Network configurations and timer optimizations                ^|
    echo  ^|  - Windows visual effects and animations                        ^|
    echo  ^|  - Selected background system services                          ^|
    echo  ^|                                                                  ^|
    echo  ^|  GUARANTEES:                                                     ^|
    echo  ^|  [+] No actions are performed without your explicit consent      ^|
    echo  ^|  [+] All system modifications are logged to your Desktop        ^|
    echo  ^|  [+] Option to create a System Restore Point is provided        ^|
    echo  ^|                                                                  ^|
    echo  ^|  User     : %-20s                              ^|
    echo  ^|  Machine  : %-20s                              ^|
    echo  ^|  Log File : Desktop\KariusOptimizer_Log.txt                    ^|
    echo  ^|                                                                  ^|
    echo  +==================================================================+
    echo.
    echo   Do you agree to terms and wish to proceed?
    echo.
    choice /c YN /n /m "  [Y] Yes, proceed    [N] Exit  >  "
    if %ERRORLEVEL% EQU 2 goto :EXIT_CLEAN
    color 0A
    goto :RESTORE_OFFER

:: ---------------------------------------------------------------------------
::  [MODULE] Restore Point Offer
::  PowerShell Checkpoint-Computer is used (safer than WMI direct call).
::  Skipped cleanly if System Protection is disabled on C:.
:: ---------------------------------------------------------------------------
:RESTORE_OFFER
    cls
    color 0B
    echo.
    echo  +==================================================================+
    echo  ^|                 S Y S T E M   R E S T O R E                      ^|
    echo  +------------------------------------------------------------------+
    echo  ^|                                                                  ^|
    echo  ^|  Would you like to create a System Restore Point before          ^|
    echo  ^|  proceeding? (Highly recommended for first-time use)            ^|
    echo  ^|                                                                  ^|
    echo  ^|   -> Automated creation, may take 30-60 seconds                ^|
    echo  ^|   -> Automatically skipped if System Protection is disabled    ^|
    echo  ^|                                                                  ^|
    echo  +==================================================================+
    echo.
    choice /c YN /n /m "  [Y] Yes, create point    [N] Skip  >  "
    if %ERRORLEVEL% EQU 2 goto :MAIN_MENU

    echo.
    echo  +------------------------------------------------------------------+
    echo  ^|  Creating System Restore Point, please wait...                   ^|
    echo  +------------------------------------------------------------------+
    echo.

    powershell -NoProfile -ExecutionPolicy Bypass -Command ^
        "try { Enable-ComputerRestore -Drive 'C:\' -EA SilentlyContinue; Checkpoint-Computer -Description 'KariusOptimizer v2.1 Pre-Optimization' -RestorePointType MODIFY_SETTINGS -EA Stop; Write-Host '  [OK] Restore point created successfully.' } catch { Write-Host '  [WARN] Could not create restore point. Enable System Protection on C: to fix this.' }"

    set "RESTORE_DONE=YES"
    echo [RESTORE] Checkpoint attempted: %DATE% %TIME% >> "%LOG_FILE%"
    echo.
    timeout /t 2 /nobreak >nul
    goto :MAIN_MENU

:: ===========================================================================
::  MAIN MENU
::  Central hub for all modules.
:: ===========================================================================
:MAIN_MENU
    color 0B
    cls
    echo.
    echo  +==================================================================+
    echo  ^|      K A R I U S O P T I M I Z E R   v 2 . 1                     ^|
    echo  ^|      Windows 10 / 11  Advanced Performance Suite                ^|
    echo  +==================================================================+
    echo  ^|  Restore Point: %-4s    Modules Applied: %MODULES_RAN%                       ^|
    echo
