```
<#
.SYNOPSIS
    Проверка и удаление списка ПО средствами PowerShell (без чтения реестра) с единым логом на ПК.
#>

[CmdletBinding()]
param (
    # Список названий или wildcard-шаблонов ПО для удаления
    [Parameter(Mandatory = $false)]
    [string[]]$TargetSoftware = @(
        "*AnyDesk*",
        "*Zoom*"
    ),

    # Каталог для логов (имя файла всегда <COMPUTERNAME>.log, сохраняется только последний)
    [Parameter(Mandatory = $false)]
    [string]$LogDirectory = "C:\ProgramData\GPO_SoftwareCleanup\Logs"
)

# 1. НАДЁЖНОСТЬ (ROBUSTNESS)
Set-StrictMode -Version Latest
$ErrorActionPreference = "Stop"

# Поддержка кириллицы (UTF-8)
[Console]::InputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8

$computerName = [string]$env:COMPUTERNAME
$logFilePath  = Join-Path -Path $LogDirectory -ChildPath "$computerName.log"
$script:LogReady = $false

# Обработчик непредвиденных сбоев с выводом номера строки и текста ошибки
trap {
    $line = $_.InvocationInfo.ScriptLineNumber
    $cmd  = $_.InvocationInfo.Line.Trim()
    $err  = $_.Exception.Message
    $text = "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] [$computerName] [ОШИБКА] Сбой на строке $line | Команда: '$cmd' | Вывод ошибки: $err"
    Write-Host $text -ForegroundColor Red
    if ($script:LogReady) {
        Add-Content -LiteralPath $logFilePath -Value $text -Encoding UTF8
    }
    exit 1
}

function Write-StatusLog {
    param (
        [Parameter(Mandatory = $true)][ValidateSet("ИНФО", "УДАЛЕНО", "УЖЕ НЕТ", "ОШИБКА")][string]$Status,
        [Parameter(Mandatory = $true)][string]$Message
    )
    $stamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line  = "[$stamp] [$computerName] [$Status] $Message"

    switch ($Status) {
        "ИНФО"    { Write-Host $line -ForegroundColor Cyan }
        "УДАЛЕНО" { Write-Host $line -ForegroundColor Green }
        "УЖЕ НЕТ" { Write-Host $line -ForegroundColor Yellow }
        "ОШИБКА"  { Write-Host $line -ForegroundColor Red }
    }

    if ($script:LogReady) {
        Add-Content -LiteralPath $logFilePath -Value $line -Encoding UTF8
    }
}

function Assert-AdminRights {
    $currentIdentity = [Security.Principal.WindowsIdentity]::GetCurrent()
    $principal       = New-Object Security.Principal.WindowsPrincipal($currentIdentity)
    if (-not $principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
        throw "Требуются права Администратора или SYSTEM. Запустите скрипт с повышенными привилегиями."
    }
}

function Initialize-SingleLog {
    param ([Parameter(Mandatory = $true)][string]$DirPath, [Parameter(Mandatory = $true)][string]$FilePath)

    if (-not (Test-Path -LiteralPath $DirPath)) {
        New-Item -Path $DirPath -ItemType Directory -Force | Out-Null
        if (-not (Test-Path -LiteralPath $DirPath)) {
            throw "Не удалось создать каталог логов: $DirPath"
        }
    }

    # Перезаписываем файл, чтобы на ПК оставался только один актуальный лог
    Set-Content -LiteralPath $FilePath -Value "" -Encoding UTF8
    if (-not (Test-Path -LiteralPath $FilePath)) {
        throw "Не удалось инициализировать лог-файл: $FilePath"
    }
    $script:LogReady = $true
}

# --- ШАГ 0: ПРОВЕРКА ПРАВ И ПОДГОТОВКА ЛОГА ---
Assert-AdminRights
Initialize-SingleLog -DirPath $LogDirectory -FilePath $logFilePath

if (-not $TargetSoftware -or $TargetSoftware.Count -eq 0) {
    throw "Список целевого ПО (\$TargetSoftware) пуст."
}

Write-StatusLog -Status "ИНФО" -Message "Шаг 1: проверка наличия ПО на ПК. Шаблонов: $($TargetSoftware.Count)"

# --- ШАГ 1: ПРОВЕРКА НАЛИЧИЯ ПО СРЕДСТВАМИ POWERSHELL ---
try {
    $installedPackages = @(Get-Package -ErrorAction Stop)
}
catch {
    $errOut = $_.Exception.Message
    Write-StatusLog -Status "ОШИБКА" -Message "Не удалось получить список пакетов через Get-Package. Вывод ошибки: $errOut"
    exit 1
}

$matchedPackages = New-Object System.Collections.Generic.List[object]
$unmatchedPatterns = New-Object System.Collections.Generic.List[string]
$seenKeys = @{}

foreach ($pattern in $TargetSoftware) {
    $foundForPattern = @(
        $installedPackages | Where-Object {
            $_.Name -and ($_.Name -like $pattern)
        }
    )

    if ($foundForPattern.Count -eq 0) {
        $unmatchedPatterns.Add($pattern)
        Write-StatusLog -Status "УЖЕ НЕТ" -Message "Пакет по шаблону '$pattern' на ПК отсутствует."
    }
    else {
        foreach ($pkg in $foundForPattern) {
            $key = "$($pkg.ProviderName)|$($pkg.Name)|$($pkg.Version)"
            if (-not $seenKeys.ContainsKey($key)) {
                $seenKeys[$key] = $true
                $matchedPackages.Add([PSCustomObject]@{
                    Pattern = $pattern
                    Package = $pkg
                })
            }
        }
    }
}

# Если целевого ПО нет — не выполняем действия по удалению
if ($matchedPackages.Count -eq 0) {
    Write-StatusLog -Status "УЖЕ НЕТ" -Message "Ни один из указанных пакетов не установлен. Этап удаления пропущен."
    Write-Host ""
    Write-Host "=== ИТОГ ($computerName) ===" -ForegroundColor White
    Write-Host "Лог: $logFilePath"
    Write-Host "Найдено к удалению: 0 | Удалено: 0 | Отсутствовало: $($unmatchedPatterns.Count) | Ошибок: 0"
    exit 0
}

# --- ШАГ 2: НЕИНТЕРАКТИВНОЕ УДАЛЕНИЕ НАЙДЕННЫХ ПАКЕТОВ ---
Write-StatusLog -Status "ИНФО" -Message "Шаг 2: найдено пакетов для удаления: $($matchedPackages.Count). Запуск удаления."

$removedCount = 0
$errorCount   = 0

foreach ($item in $matchedPackages) {
    $pkg = $item.Package
    $pkgLabel = "$($pkg.Name) (версия: $($pkg.Version), провайдер: $($pkg.ProviderName))"

    try {
        # Неинтерактивный вызов без подтверждений и перезагрузки
        $null = Uninstall-Package -InputObject $pkg -Force -ErrorAction Stop

        # Контрольная проверка, что пакет действительно удалён
        $stillPresent = @(
            Get-Package -Name $pkg.Name -ErrorAction SilentlyContinue |
            Where-Object { $_.Version -eq $pkg.Version -and $_.ProviderName -eq $pkg.ProviderName }
        )

        if ($stillPresent.Count -eq 0) {
            $removedCount++
            Write-StatusLog -Status "УДАЛЕНО" -Message "$pkgLabel"
        }
        else {
            $errorCount++
            Write-StatusLog -Status "ОШИБКА" -Message "$pkgLabel | Вывод ошибки: пакет всё ещё присутствует в системе после Uninstall-Package."
        }
    }
    catch {
        $errorCount++
        $errDetail = $_.Exception.Message
        if ($_.ErrorDetails -and -not [string]::IsNullOrWhiteSpace($_.ErrorDetails.Message)) {
            $errDetail = "$errDetail | Детали: $($_.ErrorDetails.Message)"
        }
        Write-StatusLog -Status "ОШИБКА" -Message "$pkgLabel | Вывод ошибки: $errDetail"
    }
}

# --- ШАГ 3: ИТОГОВЫЙ СТАТУС ---
Write-Host ""
Write-Host "=== ИТОГОВЫЙ СТАТУС ($computerName) ===" -ForegroundColor White
Write-Host "Лог-файл:          $logFilePath"
Write-Host "Найдено пакетов:   $($matchedPackages.Count)"
Write-Host "Удалено:           $removedCount" -ForegroundColor Green
Write-Host "Уже нет (шаблонов):$($unmatchedPatterns.Count)" -ForegroundColor Yellow
Write-Host "Ошибок:            $errorCount" -ForegroundColor ($(if ($errorCount -gt 0) { "Red" } else { "Gray" }))

if ($errorCount -gt 0) {
    exit 1
}

exit 0
```

```
powershell.exe -NoProfile -NonInteractive -ExecutionPolicy Bypass -File "\\domain\SYSVOL\domain\scripts\Remove-Software.ps1" -TargetSoftware "*AnyDesk*","*Zoom*"
```


