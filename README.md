## PC Hang Diagnostic ##

### Check System Info ###
```
# ============================================================
# PC HANG DIAGNOSTIC TOOL
# Compatible: Windows 7 / 8 / 8.1 / 10 / 11
# Output: Self-contained HTML diagnostic report
# ============================================================

$ErrorActionPreference = "SilentlyContinue"

$StartTime = Get-Date
$ComputerName = $env:COMPUTERNAME
$UserName = $env:USERNAME
$ReportFolder = Join-Path $env:USERPROFILE "Desktop"
$ReportFile = Join-Path $ReportFolder ("PC_Hang_Diagnostic_{0}_{1}.html" -f `
    $ComputerName, (Get-Date -Format "yyyyMMdd_HHmmss"))

Write-Host ""
Write-Host "============================================================" -ForegroundColor Cyan
Write-Host "              PC HANG DIAGNOSTIC TOOL" -ForegroundColor Cyan
Write-Host "============================================================" -ForegroundColor Cyan
Write-Host ""
Write-Host "Computer : $ComputerName"
Write-Host "User     : $UserName"
Write-Host ""

# ------------------------------------------------------------
# DATA STORAGE
# ------------------------------------------------------------

$Data = [ordered]@{}

# ------------------------------------------------------------
# HELPER FUNCTIONS
# ------------------------------------------------------------

function Add-Data {
    param(
        [string]$Name,
        [object]$Value
    )

    $Data[$Name] = $Value
}

function Safe-Value {
    param(
        [object]$Value,
        [string]$Default = "N/A"
    )

    if ($null -eq $Value) {
        return $Default
    }

    $Text = [string]$Value

    if ([string]::IsNullOrWhiteSpace($Text)) {
        return $Default
    }

    return $Text
}

function Convert-ToHtml {
    param(
        [object]$Value
    )

    if ($null -eq $Value) {
        return ""
    }

    return [System.Net.WebUtility]::HtmlEncode([string]$Value)
}

function Format-Bytes {
    param(
        [double]$Bytes
    )

    if ($Bytes -ge 1TB) {
        return "{0:N2} TB" -f ($Bytes / 1TB)
    }

    if ($Bytes -ge 1GB) {
        return "{0:N2} GB" -f ($Bytes / 1GB)
    }

    if ($Bytes -ge 1MB) {
        return "{0:N2} MB" -f ($Bytes / 1MB)
    }

    if ($Bytes -ge 1KB) {
        return "{0:N2} KB" -f ($Bytes / 1KB)
    }

    return "{0:N0} Bytes" -f $Bytes
}

function Get-PropertyValue {
    param(
        [object]$Object,
        [string]$Property
    )

    if ($null -eq $Object) {
        return "N/A"
    }

    try {
        if ($null -ne $Object.PSObject.Properties[$Property]) {
            return Safe-Value $Object.$Property
        }
    }
    catch {}

    return "N/A"
}

function Write-Step {
    param(
        [int]$Number,
        [int]$Total,
        [string]$Text
    )

    Write-Host ("[{0}/{1}] {2}" -f $Number, $Total, $Text) -ForegroundColor Yellow
}

function Get-TableHtml {
    param(
        [object]$Items,
        [string[]]$Properties
    )

    if ($null -eq $Items) {
        return '<div class="empty">No information available.</div>'
    }

    $Array = @($Items)

    if ($Array.Count -eq 0) {
        return '<div class="empty">No information available.</div>'
    }

    $Html = '<div class="table-wrap"><table><thead><tr>'

    foreach ($Property in $Properties) {
        $Html += "<th>$(Convert-ToHtml $Property)</th>"
    }

    $Html += "</tr></thead><tbody>"

    foreach ($Item in $Array) {
        $Html += "<tr>"

        foreach ($Property in $Properties) {
            $Value = Get-PropertyValue $Item $Property
            $Html += "<td>$(Convert-ToHtml $Value)</td>"
        }

        $Html += "</tr>"
    }

    $Html += "</tbody></table></div>"

    return $Html
}

# ============================================================
# 1. SYSTEM INFORMATION
# ============================================================

Write-Step 1 15 "Collecting System Information..."

try {
    $OS = Get-CimInstance Win32_OperatingSystem
}
catch {
    try {
        $OS = Get-WmiObject Win32_OperatingSystem
    }
    catch {
        $OS = $null
    }
}

try {
    $Computer = Get-CimInstance Win32_ComputerSystem
}
catch {
    try {
        $Computer = Get-WmiObject Win32_ComputerSystem
    }
    catch {
        $Computer = $null
    }
}

try {
    $BIOS = Get-CimInstance Win32_BIOS
}
catch {
    try {
        $BIOS = Get-WmiObject Win32_BIOS
    }
    catch {
        $BIOS = $null
    }
}

try {
    $CPU = Get-CimInstance Win32_Processor
}
catch {
    try {
        $CPU = Get-WmiObject Win32_Processor
    }
    catch {
        $CPU = $null
    }
}

$SystemInfo = [ordered]@{
    ComputerName     = $ComputerName
    LoggedInUser     = $UserName
    Manufacturer     = Get-PropertyValue $Computer "Manufacturer"
    Model            = Get-PropertyValue $Computer "Model"
    OperatingSystem  = Get-PropertyValue $OS "Caption"
    OSVersion        = Get-PropertyValue $OS "Version"
    OSBuild          = Get-PropertyValue $OS "BuildNumber"
    Architecture     = Get-PropertyValue $OS "OSArchitecture"
    CPU              = Get-PropertyValue $CPU "Name"
    CPUCores         = Get-PropertyValue $CPU "NumberOfCores"
    CPUThreads       = Get-PropertyValue $CPU "NumberOfLogicalProcessors"
    BIOSManufacturer = Get-PropertyValue $BIOS "Manufacturer"
    BIOSVersion      = Get-PropertyValue $BIOS "SMBIOSBIOSVersion"
    BIOSDate         = Get-PropertyValue $BIOS "ReleaseDate"
}

Add-Data "System Information" $SystemInfo

# ============================================================
# 2. RAM
# ============================================================

Write-Step 2 15 "Checking RAM..."

try {
    $RAMModules = @(Get-CimInstance Win32_PhysicalMemory)
}
catch {
    try {
        $RAMModules = @(Get-WmiObject Win32_PhysicalMemory)
    }
    catch {
        $RAMModules = @()
    }
}

$TotalRAM = 0

foreach ($RAM in $RAMModules) {
    try {
        $TotalRAM += [double]$RAM.Capacity
    }
    catch {}
}

$RAMUsage = $null

try {
    $RAMUsage = [math]::Round(
        (($TotalRAM - [double]$OS.FreePhysicalMemory * 1KB) / $TotalRAM) * 100,
        2
    )
}
catch {}

$RAMInfo = [ordered]@{
    InstalledRAM = Format-Bytes $TotalRAM
    RAMUsage     = if ($null -ne $RAMUsage) { "$RAMUsage %" } else { "N/A" }
    AvailableRAM = if ($null -ne $OS) {
        Format-Bytes ([double]$OS.FreePhysicalMemory * 1KB)
    }
    else {
        "N/A"
    }
    Modules      = $RAMModules.Count
}

Add-Data "RAM Summary" $RAMInfo
Add-Data "RAM Modules" $RAMModules

# ============================================================
# 3. CPU
# ============================================================

Write-Step 3 15 "Checking CPU..."

$CPUUsage = $null

try {
    $Samples = @()

    for ($i = 0; $i -lt 3; $i++) {
        $Samples += @(Get-CimInstance Win32_Processor |
            Measure-Object -Property LoadPercentage -Average).Average

        Start-Sleep -Seconds 1
    }

    if ($Samples.Count -gt 0) {
        $CPUUsage = [math]::Round(
            (($Samples | Measure-Object -Average).Average),
            2
        )
    }
}
catch {}

$CPUInfo = [ordered]@{
    Processor = Get-PropertyValue $CPU "Name"
    Cores     = Get-PropertyValue $CPU "NumberOfCores"
    Threads   = Get-PropertyValue $CPU "NumberOfLogicalProcessors"
    MaxClock  = if ($CPU) {
        "{0} MHz" -f (Get-PropertyValue $CPU "MaxClockSpeed")
    }
    else {
        "N/A"
    }
    CurrentUsage = if ($null -ne $CPUUsage) {
        "$CPUUsage %"
    }
    else {
        "N/A"
    }
}

Add-Data "CPU Information" $CPUInfo

# ============================================================
# 4. UPTIME
# ============================================================

Write-Step 4 15 "Checking uptime..."

$UptimeInfo = [ordered]@{
    LastBootTime = "N/A"
    Uptime       = "N/A"
}

try {
    $BootTime = $OS.LastBootUpTime

    if ($BootTime) {
        $BootDate = [Management.ManagementDateTimeConverter]::ToDateTime(
            [string]$BootTime
        )

        $Uptime = (Get-Date) - $BootDate

        $UptimeInfo.LastBootTime = $BootDate.ToString("yyyy-MM-dd HH:mm:ss")
        $UptimeInfo.Uptime = "{0} days {1} hours {2} minutes" -f `
            $Uptime.Days,
            $Uptime.Hours,
            $Uptime.Minutes
    }
}
catch {}

Add-Data "System Uptime" $UptimeInfo

# ============================================================
# 5. DISK SPACE
# ============================================================

Write-Step 5 15 "Checking disk space..."

$DiskInfo = @()

try {
    $LogicalDisks = @(Get-CimInstance Win32_LogicalDisk -Filter "DriveType=3")

    foreach ($Disk in $LogicalDisks) {

        $FreePercent = 0

        if ([double]$Disk.Size -gt 0) {
            $FreePercent = [math]::Round(
                ([double]$Disk.FreeSpace / [double]$Disk.Size) * 100,
                2
            )
        }

        $Status = "OK"

        if ($FreePercent -lt 10) {
            $Status = "CRITICAL"
        }
        elseif ($FreePercent -lt 20) {
            $Status = "WARNING"
        }

        $DiskInfo += [pscustomobject]@{
            Drive          = $Disk.DeviceID
            FileSystem     = $Disk.FileSystem
            Total          = Format-Bytes $Disk.Size
            Free           = Format-Bytes $Disk.FreeSpace
            FreePercent    = "$FreePercent %"
            Status         = $Status
        }
    }
}
catch {}

Add-Data "Disk Space" $DiskInfo

# ============================================================
# 6. DISK HEALTH / SMART
# ============================================================

Write-Step 6 15 "Checking disk health..."

$DiskHealth = @()

try {
    $PhysicalDisks = @(Get-CimInstance Win32_DiskDrive)

    foreach ($Disk in $PhysicalDisks) {

        $Health = "Unknown"

        try {
            $Status = Get-CimInstance `
                -Namespace root\wmi `
                -ClassName MSStorageDriver_FailurePredictStatus |
                Where-Object {
                    $_.InstanceName -like "*$($Disk.PNPDeviceID)*"
                }

            if ($Status) {
                if ($Status.PredictFailure) {
                    $Health = "PREDICTIVE FAILURE"
                }
                else {
                    $Health = "OK"
                }
            }
        }
        catch {}

        $DiskHealth += [pscustomobject]@{
            Model       = Safe-Value $Disk.Model
            Interface   = Safe-Value $Disk.InterfaceType
            Size        = Format-Bytes $Disk.Size
            Status      = $Health
            Serial      = Safe-Value $Disk.SerialNumber
        }
    }
}
catch {}

Add-Data "Disk Health" $DiskHealth

# ============================================================
# 7. DEVICE / DRIVER PROBLEMS
# ============================================================

Write-Step 7 15 "Checking device and driver problems..."

$DeviceProblems = @()

try {
    $DeviceProblems = @(Get-CimInstance Win32_PnPEntity |
        Where-Object {
            $_.ConfigManagerErrorCode -ne 0
        } |
        Select-Object `
            Name,
            Manufacturer,
            PNPDeviceID,
            ConfigManagerErrorCode,
            Status)
}
catch {}

Add-Data "Device Problems" $DeviceProblems

# ============================================================
# 8. EVENT LOGS
# ============================================================

Write-Step 8 15 "Reading event logs..."

$SystemEvents = @()
$ApplicationEvents = @()
$CriticalEvents = @()

try {
    $SystemEvents = @(Get-WinEvent -FilterHashtable @{
        LogName   = "System"
        StartTime = (Get-Date).AddDays(-7)
    } -MaxEvents 100 |
    Where-Object {
        $_.LevelDisplayName -in @("Error","Critical","Warning")
    } |
    Select-Object -First 100 |
    ForEach-Object {
        [pscustomobject]@{
            Time    = $_.TimeCreated
            Level   = $_.LevelDisplayName
            Source  = $_.ProviderName
            EventID = $_.Id
            Message = $_.Message
        }
    })
}
catch {}

try {
    $ApplicationEvents = @(Get-WinEvent -FilterHashtable @{
        LogName   = "Application"
        StartTime = (Get-Date).AddDays(-7)
    } -MaxEvents 100 |
    Where-Object {
        $_.LevelDisplayName -in @("Error","Critical","Warning")
    } |
    Select-Object -First 100 |
    ForEach-Object {
        [pscustomobject]@{
            Time    = $_.TimeCreated
            Level   = $_.LevelDisplayName
            Source  = $_.ProviderName
            EventID = $_.Id
            Message = $_.Message
        }
    })
}
catch {}

try {
    $CriticalEvents = @(Get-WinEvent -FilterHashtable @{
        LogName   = "System"
        Level     = 1,2
        StartTime = (Get-Date).AddDays(-7)
    } -MaxEvents 50 |
    ForEach-Object {
        [pscustomobject]@{
            Time    = $_.TimeCreated
            Source  = $_.ProviderName
            EventID = $_.Id
            Message = $_.Message
        }
    })
}
catch {}

Add-Data "System Events" $SystemEvents
Add-Data "Application Events" $ApplicationEvents
Add-Data "Critical Events" $CriticalEvents

# ============================================================
# 9. RUNNING PROCESSES
# ============================================================

Write-Step 9 15 "Checking running processes..."

$Processes = @()

try {
    $Processes = @(Get-Process |
        Sort-Object WorkingSet64 -Descending |
        Select-Object -First 30 |
        ForEach-Object {

            $CPUValue = "N/A"

            try {
                if ($_.CPU) {
                    $CPUValue = [math]::Round([double]$_.CPU, 2)
                }
            }
            catch {}

            [pscustomobject]@{
                Name       = $_.ProcessName
                PID        = $_.Id
                CPUSeconds = $CPUValue
                MemoryMB   = [math]::Round(
                    ([double]$_.WorkingSet64 / 1MB),
                    2
                )
                Threads    = $_.Threads.Count
            }
        })
}
catch {}

Add-Data "Top Processes" $Processes

# ============================================================
# 10. STARTUP PROGRAMS
# ============================================================

Write-Step 10 15 "Checking startup programs..."

$StartupPrograms = @()

try {
    $StartupPrograms = @(Get-CimInstance Win32_StartupCommand |
        Select-Object `
            Name,
            Command,
            Location,
            User)
}
catch {}

Add-Data "Startup Programs" $StartupPrograms

# ============================================================
# 11. RUNNING SERVICES
# ============================================================

Write-Step 11 15 "Checking services..."

$Services = @()

try {
    $Services = @(Get-Service |
        Where-Object {
            $_.Status -eq "Running"
        } |
        Select-Object `
            Name,
            DisplayName,
            Status,
            StartType |
        Sort-Object DisplayName)
}
catch {}

Add-Data "Running Services" $Services

# ============================================================
# 12. PAGE FILE
# ============================================================

Write-Step 12 15 "Checking page file..."

$PageFile = @()

try {
    $PageFile = @(Get-CimInstance Win32_PageFileUsage |
        Select-Object `
            Name,
            AllocatedBaseSize,
            CurrentUsage,
            PeakUsage,
            TempPageFile |
        ForEach-Object {

            [pscustomobject]@{
                Name             = $_.Name
                AllocatedMB      = $_.AllocatedBaseSize
                CurrentUsageMB   = $_.CurrentUsage
                PeakUsageMB      = $_.PeakUsage
                Temporary        = $_.TempPageFile
            }
        })
}
catch {}

Add-Data "Page File" $PageFile

# ============================================================
# 13. CRASH DUMPS
# ============================================================

Write-Step 13 15 "Checking crash dumps..."

$CrashDumps = @()

$DumpLocations = @(
    "$env:SystemRoot\Minidump",
    "$env:SystemRoot\MEMORY.DMP"
)

foreach ($Location in $DumpLocations) {

    try {

        if (Test-Path $Location) {

            if ((Get-Item $Location).PSIsContainer) {

                $Files = @(Get-ChildItem $Location -Filter "*.dmp" -File)

                foreach ($File in $Files) {

                    $CrashDumps += [pscustomobject]@{
                        File         = $File.FullName
                        Size         = Format-Bytes $File.Length
                        Created      = $File.CreationTime
                        Modified     = $File.LastWriteTime
                    }
                }
            }
            else {

                $File = Get-Item $Location

                $CrashDumps += [pscustomobject]@{
                    File         = $File.FullName
                    Size         = Format-Bytes $File.Length
                    Created      = $File.CreationTime
                    Modified     = $File.LastWriteTime
                }
            }
        }
    }
    catch {}
}

Add-Data "Crash Dumps" $CrashDumps

# ============================================================
# 14. NETWORK
# ============================================================

Write-Step 14 15 "Checking network adapters..."

$NetworkInfo = @()

try {

    $Adapters = @(Get-CimInstance Win32_NetworkAdapter |
        Where-Object {
            $_.NetEnabled -eq $true
        })

    foreach ($Adapter in $Adapters) {

        $Configuration = $null

        try {
            $Configuration = Get-CimInstance Win32_NetworkAdapterConfiguration |
                Where-Object {
                    $_.Index -eq $Adapter.DeviceID
                }
        }
        catch {}

        $IPAddresses = "N/A"
        $Gateway = "N/A"
        $DNS = "N/A"

        if ($Configuration) {

            if ($Configuration.IPAddress) {
                $IPAddresses = $Configuration.IPAddress -join ", "
            }

            if ($Configuration.DefaultIPGateway) {
                $Gateway = $Configuration.DefaultIPGateway -join ", "
            }

            if ($Configuration.DNSServerSearchOrder) {
                $DNS = $Configuration.DNSServerSearchOrder -join ", "
            }
        }

        $NetworkInfo += [pscustomobject]@{
            Adapter     = Safe-Value $Adapter.Name
            Manufacturer = Safe-Value $Adapter.Manufacturer
            MACAddress  = Safe-Value $Adapter.MACAddress
            Speed       = if ($Adapter.Speed) {
                "{0} Mbps" -f [math]::Round(
                    ([double]$Adapter.Speed / 1MB), 2
                )
            }
            else {
                "N/A"
            }
            IPAddress   = $IPAddresses
            Gateway     = $Gateway
            DNS         = $DNS
        }
    }
}
catch {}

Add-Data "Network Adapters" $NetworkInfo

# ============================================================
# 15. TEMPERATURE / SENSORS
# ============================================================

Write-Step 15 15 "Checking temperature sensors..."

$TemperatureInfo = @()

try {

    $Temperatures = @(Get-CimInstance `
        -Namespace root/wmi `
        -ClassName MSAcpi_ThermalZoneTemperature)

    foreach ($Temperature in $Temperatures) {

        $Celsius = (
            [double]$Temperature.CurrentTemperature / 10
        ) - 273.15

        $TemperatureInfo += [pscustomobject]@{
            Sensor      = Safe-Value $Temperature.InstanceName
            Temperature = "{0:N1} C" -f $Celsius
        }
    }
}
catch {}

Add-Data "Temperature Sensors" $TemperatureInfo

# ============================================================
# AUTOMATIC DIAGNOSTIC ANALYSIS
# ============================================================

Write-Host ""
Write-Host "Running automatic diagnostic analysis..." -ForegroundColor Cyan

$Findings = @()

# RAM
if ($null -ne $RAMUsage) {

    if ($RAMUsage -ge 90) {
        $Findings += [pscustomobject]@{
            Severity = "CRITICAL"
            Area     = "RAM"
            Finding  = "RAM usage is extremely high."
            Recommendation = "Check applications consuming excessive memory. Consider additional RAM."
        }
    }
    elseif ($RAMUsage -ge 80) {
        $Findings += [pscustomobject]@{
            Severity = "WARNING"
            Area     = "RAM"
            Finding  = "RAM usage is high."
            Recommendation = "Check memory-heavy applications and background processes."
        }
    }
}

# CPU
if ($null -ne $CPUUsage) {

    if ($CPUUsage -ge 90) {
        $Findings += [pscustomobject]@{
            Severity = "CRITICAL"
            Area     = "CPU"
            Finding  = "CPU utilization is extremely high."
            Recommendation = "Check the Top Processes section for applications consuming CPU."
        }
    }
    elseif ($CPUUsage -ge 80) {
        $Findings += [pscustomobject]@{
            Severity = "WARNING"
            Area     = "CPU"
            Finding  = "CPU utilization is high."
            Recommendation = "Check running processes and background applications."
        }
    }
}

# Disk
foreach ($Disk in @($DiskInfo)) {

    if ($Disk.Status -eq "CRITICAL") {

        $Findings += [pscustomobject]@{
            Severity = "CRITICAL"
            Area     = "Disk Space"
            Finding  = "$($Disk.Drive) has less than 10% free space."
            Recommendation = "Free disk space. Low disk space can cause severe Windows performance problems."
        }
    }
    elseif ($Disk.Status -eq "WARNING") {

        $Findings += [pscustomobject]@{
            Severity = "WARNING"
            Area     = "Disk Space"
            Finding  = "$($Disk.Drive) has less than 20% free space."
            Recommendation = "Consider cleaning temporary files and unnecessary data."
        }
    }
}

# SMART
foreach ($Disk in @($DiskHealth)) {

    if ($Disk.Status -eq "PREDICTIVE FAILURE") {

        $Findings += [pscustomobject]@{
            Severity = "CRITICAL"
            Area     = "Disk Health"
            Finding  = "Disk reports a predictive failure."
            Recommendation = "Back up important data immediately and test/replace the drive."
        }
    }
}

# Device problems
if (@($DeviceProblems).Count -gt 0) {

    $Findings += [pscustomobject]@{
        Severity = "WARNING"
        Area     = "Drivers / Devices"
        Finding  = "$(@($DeviceProblems).Count) device(s) report configuration problems."
        Recommendation = "Check Device Manager and update/reinstall the affected drivers."
    }
}

# System events
$SystemErrorCount = @(
    $SystemEvents |
    Where-Object {
        $_.Level -eq "Error" -or $_.Level -eq "Critical"
    }
).Count

if ($SystemErrorCount -gt 10) {

    $Findings += [pscustomobject]@{
        Severity = "WARNING"
        Area     = "System Event Log"
        Finding  = "$SystemErrorCount serious System events were found during the last 7 days."
        Recommendation = "Review System Events for disk, driver, power, hardware and Windows errors."
    }
}

# Application events
$ApplicationErrorCount = @(
    $ApplicationEvents |
    Where-Object {
        $_.Level -eq "Error" -or $_.Level -eq "Critical"
    }
).Count

if ($ApplicationErrorCount -gt 10) {

    $Findings += [pscustomobject]@{
        Severity = "WARNING"
        Area     = "Application Event Log"
        Finding  = "$ApplicationErrorCount serious Application events were found during the last 7 days."
        Recommendation = "Review Application Events for crashing applications and software conflicts."
    }
}

# Uptime
try {

    if ($Uptime.Days -ge 7) {

        $Findings += [pscustomobject]@{
            Severity = "INFO"
            Area     = "System Uptime"
            Finding  = "System has been running for more than 7 days."
            Recommendation = "Restart Windows periodically, especially after updates."
        }
    }
}
catch {}

# Page file
foreach ($PF in @($PageFile)) {

    try {

        if ([double]$PF.CurrentUsageMB -gt 0 -and
            [double]$PF.AllocatedMB -gt 0) {

            $PFPercent = (
                [double]$PF.CurrentUsageMB /
                [double]$PF.AllocatedMB
            ) * 100

            if ($PFPercent -ge 90) {

                $Findings += [pscustomobject]@{
                    Severity = "WARNING"
                    Area     = "Page File"
                    Finding  = "Page file usage is very high."
                    Recommendation = "Investigate RAM pressure and memory-heavy applications."
                }
            }
        }
    }
    catch {}
}

if (@($Findings).Count -eq 0) {

    $Findings += [pscustomobject]@{
        Severity = "INFO"
        Area     = "Overall"
        Finding  = "No major automatic problem was detected from the collected information."
        Recommendation = "If the PC still hangs, reproduce the issue and review Event Logs, Top Processes, Drivers and Disk Health."
    }
}

Add-Data "Automatic Diagnostic Findings" $Findings

# ============================================================
# HTML GENERATION
# ============================================================

Write-Host ""
Write-Host "Creating HTML report..." -ForegroundColor Cyan

function New-Section {
    param(
        [string]$Title,
        [string]$Content,
        [string]$Badge = ""
    )

    $BadgeHtml = ""

    if ($Badge -ne "") {
        $BadgeHtml = '<span class="badge">' +
            (Convert-ToHtml $Badge) +
            '</span>'
    }

    return @"
<section class="section">
    <button class="section-header" type="button" onclick="toggleSection(this)">
        <span class="section-title">
            <span class="section-icon">+</span>
            $(Convert-ToHtml $Title)
        </span>
        $BadgeHtml
    </button>

    <div class="section-content">
        $Content
    </div>
</section>
"@
}

function New-KeyValueTable {
    param(
        [hashtable]$Object
    )

    $Html = '<div class="info-grid">'

    foreach ($Key in $Object.Keys) {

        $Value = Safe-Value $Object[$Key]

        $Html += @"
<div class="info-item">
    <div class="info-label">$(Convert-ToHtml $Key)</div>
    <div class="info-value">$(Convert-ToHtml $Value)</div>
</div>
"@
    }

    $Html += "</div>"

    return $Html
}

function New-FindingHtml {
    param(
        [object]$Finding
    )

    $Class = "info"

    if ($Finding.Severity -eq "CRITICAL") {
        $Class = "critical"
    }
    elseif ($Finding.Severity -eq "WARNING") {
        $Class = "warning"
    }

    return @"
<div class="finding $Class">
    <div class="finding-top">
        <span class="severity">$(
            Convert-ToHtml $Finding.Severity
        )</span>

        <span class="finding-area">$(
            Convert-ToHtml $Finding.Area
        )</span>
    </div>

    <div class="finding-title">$(
        Convert-ToHtml $Finding.Finding
    )</div>

    <div class="finding-recommendation">
        $(Convert-ToHtml $Finding.Recommendation)
    </div>
</div>
"@
}

# ------------------------------------------------------------
# CREATE SECTION HTML
# ------------------------------------------------------------

$Sections = New-Object System.Collections.Generic.List[string]

$Sections.Add(
    (New-Section `
        "System Information" `
        (New-KeyValueTable $SystemInfo) `
        "SYSTEM")
)

$Sections.Add(
    (New-Section `
        "RAM Summary" `
        (New-KeyValueTable $RAMInfo) `
        "RAM")
)

$Sections.Add(
    (New-Section `
        "RAM Modules" `
        (Get-TableHtml $RAMModules @(
            "Capacity",
            "Speed",
            "Manufacturer",
            "PartNumber",
            "SerialNumber"
        )) `
        "MEMORY")
)

$Sections.Add(
    (New-Section `
        "CPU Information" `
        (New-KeyValueTable $CPUInfo) `
        "CPU")
)

$Sections.Add(
    (New-Section `
        "System Uptime" `
        (New-KeyValueTable $UptimeInfo) `
        "UPTIME")
)

$Sections.Add(
    (New-Section `
        "Disk Space" `
        (Get-TableHtml $DiskInfo @(
            "Drive",
            "FileSystem",
            "Total",
            "Free",
            "FreePercent",
            "Status"
        )) `
        "STORAGE")
)

$Sections.Add(
    (New-Section `
        "Disk Health" `
        (Get-TableHtml $DiskHealth @(
            "Model",
            "Interface",
            "Size",
            "Status",
            "Serial"
        )) `
        "SMART")
)

$Sections.Add(
    (New-Section `
        "Device and Driver Problems" `
        (Get-TableHtml $DeviceProblems @(
            "Name",
            "Manufacturer",
            "ConfigManagerErrorCode",
            "Status"
        )) `
        "DRIVERS")
)

$Sections.Add(
    (New-Section `
        "System Events - Last 7 Days" `
        (Get-TableHtml $SystemEvents @(
            "Time",
            "Level",
            "Source",
            "EventID",
            "Message"
        )) `
        "EVENT LOG")
)

$Sections.Add(
    (New-Section `
        "Application Events - Last 7 Days" `
        (Get-TableHtml $ApplicationEvents @(
            "Time",
            "Level",
            "Source",
            "EventID",
            "Message"
        )) `
        "EVENT LOG")
)

$Sections.Add(
    (New-Section `
        "Critical System Events" `
        (Get-TableHtml $CriticalEvents @(
            "Time",
            "Source",
            "EventID",
            "Message"
        )) `
        "CRITICAL")
)

$Sections.Add(
    (New-Section `
        "Top Processes" `
        (Get-TableHtml $Processes @(
            "Name",
            "PID",
            "CPUSeconds",
            "MemoryMB",
            "Threads"
        )) `
        "PROCESSES")
)

$Sections.Add(
    (New-Section `
        "Startup Programs" `
        (Get-TableHtml $StartupPrograms @(
            "Name",
            "Command",
            "Location",
            "User"
        )) `
        "STARTUP")
)

$Sections.Add(
    (New-Section `
        "Running Services" `
        (Get-TableHtml $Services @(
            "Name",
            "DisplayName",
            "Status",
            "StartType"
        )) `
        "SERVICES")
)

$Sections.Add(
    (New-Section `
        "Page File" `
        (Get-TableHtml $PageFile @(
            "Name",
            "AllocatedMB",
            "CurrentUsageMB",
            "PeakUsageMB",
            "Temporary"
        )) `
        "MEMORY")
)

$Sections.Add(
    (New-Section `
        "Crash Dumps" `
        (Get-TableHtml $CrashDumps @(
            "File",
            "Size",
            "Created",
            "Modified"
        )) `
        "CRASH DUMPS")
)

$Sections.Add(
    (New-Section `
        "Network Adapters" `
        (Get-TableHtml $NetworkInfo @(
            "Adapter",
            "Manufacturer",
            "MACAddress",
            "Speed",
            "IPAddress",
            "Gateway",
            "DNS"
        )) `
        "NETWORK")
)

$Sections.Add(
    (New-Section `
        "Temperature Sensors" `
        (Get-TableHtml $TemperatureInfo @(
            "Sensor",
            "Temperature"
        )) `
        "TEMPERATURE")
)

# ------------------------------------------------------------
# DIAGNOSTIC FINDINGS
# ------------------------------------------------------------

$FindingHtml = ""

foreach ($Finding in @($Findings)) {
    $FindingHtml += New-FindingHtml $Finding
}

$Sections.Insert(
    0,
    (New-Section `
        "Automatic Diagnostic Findings" `
        $FindingHtml `
        "ANALYSIS")
)

$EndTime = Get-Date
$Duration = $EndTime - $StartTime

# ------------------------------------------------------------
# HTML DOCUMENT
# ------------------------------------------------------------

$Html = @"
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">

<title>PC Hang Diagnostic Report - $(Convert-ToHtml $ComputerName)</title>

<meta name="viewport" content="width=device-width, initial-scale=1.0">

<style>

* {
    box-sizing: border-box;
}

html {
    scroll-behavior: smooth;
}

body {
    margin: 0;
    padding: 0;
    background: #f4f6f9;
    color: #20252b;
    font-family: Arial, Helvetica, sans-serif;
    font-size: 14px;
}

.container {
    max-width: 1500px;
    margin: 0 auto;
    padding: 25px;
}

.header {
    background: linear-gradient(135deg, #172033, #273653);
    color: white;
    border-radius: 18px;
    padding: 30px;
    margin-bottom: 22px;
    box-shadow: 0 10px 30px rgba(0,0,0,0.12);
}

.header h1 {
    margin: 0 0 8px 0;
    font-size: 28px;
    font-weight: 700;
}

.header p {
    margin: 4px 0;
    color: #dbe3f0;
}

.header-meta {
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    margin-top: 20px;
}

.meta {
    background: rgba(255,255,255,0.10);
    border: 1px solid rgba(255,255,255,0.14);
    border-radius: 10px;
    padding: 9px 13px;
}

.toolbar {
    position: sticky;
    top: 10px;
    z-index: 100;
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    margin-bottom: 18px;
    padding: 10px;
    background: rgba(244,246,249,0.94);
    backdrop-filter: blur(8px);
}

.toolbar button {
    border: 0;
    border-radius: 10px;
    padding: 11px 17px;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
    transition: 0.2s ease;
}

.toolbar button:hover {
    transform: translateY(-1px);
    box-shadow: 0 5px 15px rgba(0,0,0,0.12);
}

.btn-open {
    background: #172033;
    color: white;
}

.btn-close {
    background: #ffffff;
    color: #172033;
    border: 1px solid #d7dce4 !important;
}

.btn-print {
    background: #ffffff;
    color: #172033;
    border: 1px solid #d7dce4 !important;
}

.section {
    background: white;
    border-radius: 14px;
    margin-bottom: 12px;
    overflow: hidden;
    border: 1px solid #e2e6ec;
    box-shadow: 0 3px 12px rgba(0,0,0,0.045);
}

.section-header {
    width: 100%;
    border: 0;
    background: white;
    padding: 17px 20px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: space-between;
    text-align: left;
    transition: background 0.2s ease;
}

.section-header:hover {
    background: #f8fafc;
}

.section-title {
    display: flex;
    align-items: center;
    gap: 12px;
    font-size: 16px;
    font-weight: 700;
    color: #172033;
}

.section-icon {
    width: 27px;
    height: 27px;
    border-radius: 8px;
    background: #eef2f7;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    font-size: 20px;
    line-height: 1;
    color: #4a5668;
    transition: 0.25s ease;
}

.section.open .section-icon {
    transform: rotate(45deg);
    background: #172033;
    color: white;
}

.badge {
    background: #eef2f7;
    color: #526074;
    border-radius: 20px;
    padding: 5px 10px;
    font-size: 10px;
    font-weight: 700;
    letter-spacing: 0.5px;
}

.section-content {
    max-height: 0;
    overflow: hidden;
    opacity: 0;
    padding: 0 20px;
    transition:
        max-height 0.35s ease,
        opacity 0.25s ease,
        padding 0.35s ease;
}

.section.open .section-content {
    max-height: 100000px;
    opacity: 1;
    padding: 0 20px 20px 20px;
}

.info-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(230px, 1fr));
    gap: 10px;
}

.info-item {
    background: #f7f9fb;
    border: 1px solid #e7ebf0;
    border-radius: 10px;
    padding: 12px;
}

.info-label {
    color: #6d7786;
    font-size: 11px;
    font-weight: 700;
    text-transform: uppercase;
    margin-bottom: 6px;
}

.info-value {
    color: #1f2937;
    font-size: 14px;
    font-weight: 600;
    word-break: break-word;
}

.table-wrap {
    width: 100%;
    overflow-x: auto;
    border: 1px solid #e2e6ec;
    border-radius: 10px;
}

table {
    width: 100%;
    border-collapse: collapse;
    min-width: 700px;
}

th {
    background: #f0f3f7;
    color: #3c4655;
    font-size: 11px;
    text-transform: uppercase;
    letter-spacing: 0.4px;
    text-align: left;
    padding: 11px;
    border-bottom: 1px solid #dfe4eb;
    white-space: nowrap;
}

td {
    padding: 10px 11px;
    border-bottom: 1px solid #edf0f3;
    vertical-align: top;
    line-height: 1.45;
    word-break: break-word;
}

tr:last-child td {
    border-bottom: 0;
}

tr:hover td {
    background: #fafbfc;
}

.empty {
    padding: 18px;
    background: #f7f9fb;
    border: 1px dashed #d5dae2;
    border-radius: 10px;
    color: #6b7280;
}

.finding {
    border-radius: 12px;
    padding: 16px;
    margin-bottom: 10px;
    border-left: 5px solid #7b8798;
    background: #f7f9fb;
}

.finding.critical {
    border-left-color: #c0392b;
    background: #fff5f4;
}

.finding.warning {
    border-left-color: #d68910;
    background: #fff9ed;
}

.finding.info {
    border-left-color: #52759b;
    background: #f4f8fc;
}

.finding-top {
    display: flex;
    align-items: center;
    gap: 9px;
    margin-bottom: 8px;
}

.severity {
    font-size: 10px;
    font-weight: 800;
    padding: 5px 8px;
    border-radius: 6px;
    background: #e7ebf0;
    color: #394454;
}

.finding.critical .severity {
    background: #f5d4d1;
    color: #8e2117;
}

.finding.warning .severity {
    background: #f7e3b6;
    color: #855300;
}

.finding.info .severity {
    background: #dce9f5;
    color: #315779;
}

.finding-area {
    font-size: 11px;
    font-weight: 700;
    color: #6b7280;
}

.finding-title {
    font-size: 15px;
    font-weight: 700;
    margin-bottom: 7px;
}

.finding-recommendation {
    color: #5d6674;
    line-height: 1.5;
}

.footer {
    margin-top: 25px;
    text-align: center;
    color: #7b8492;
    font-size: 12px;
    padding: 20px;
}

@media(max-width: 700px) {

    .container {
        padding: 12px;
    }

    .header {
        padding: 22px;
    }

    .header h1 {
        font-size: 23px;
    }

    .toolbar {
        position: static;
    }

    .toolbar button {
        flex: 1;
    }

    .section-header {
        padding: 15px;
    }

    .section-content {
        padding-left: 15px;
        padding-right: 15px;
    }

}

@media print {

    body {
        background: white;
    }

    .container {
        max-width: none;
        padding: 0;
    }

    .toolbar {
        display: none;
    }

    .section {
        box-shadow: none;
        break-inside: avoid;
    }

    .section-content {
        max-height: none !important;
        opacity: 1 !important;
        padding: 0 20px 20px 20px !important;
    }

    .section-icon {
        display: none;
    }

}

</style>
</head>

<body>

<div class="container">

<div class="header">

    <h1>PC Hang Diagnostic Report</h1>

    <p>Windows system performance and troubleshooting report</p>

    <div class="header-meta">

        <div class="meta">
            <strong>Computer:</strong>
            $(Convert-ToHtml $ComputerName)
        </div>

        <div class="meta">
            <strong>User:</strong>
            $(Convert-ToHtml $UserName)
        </div>

        <div class="meta">
            <strong>Generated:</strong>
            $(Get-Date -Format "yyyy-MM-dd HH:mm:ss")
        </div>

        <div class="meta">
            <strong>Scan Time:</strong>
            $([math]::Round($Duration.TotalSeconds,2)) seconds
        </div>

    </div>

</div>

<div class="toolbar">

    <button class="btn-open" type="button" onclick="openAll()">
        Expand All
    </button>

    <button class="btn-close" type="button" onclick="closeAll()">
        Collapse All
    </button>

    <button class="btn-print" type="button" onclick="window.print()">
        Print Report
    </button>

</div>

$($Sections -join "`n")

<div class="footer">
    PC Hang Diagnostic Tool<br>
    Report generated locally on the computer.
</div>

</div>

<script>

function toggleSection(button) {

    var section = button.parentElement;

    if (section.classList.contains("open")) {
        section.classList.remove("open");
    }
    else {
        section.classList.add("open");
    }
}

function openAll() {

    var sections = document.querySelectorAll(".section");

    for (var i = 0; i < sections.length; i++) {
        sections[i].classList.add("open");
    }
}

function closeAll() {

    var sections = document.querySelectorAll(".section");

    for (var i = 0; i < sections.length; i++) {
        sections[i].classList.remove("open");
    }
}

</script>

</body>
</html>
"@

# ============================================================
# SAVE REPORT
# ============================================================

try {

    [System.IO.File]::WriteAllText(
        $ReportFile,
        $Html,
        [System.Text.UTF8Encoding]::new($false)
    )

    Write-Host ""
    Write-Host "============================================================" -ForegroundColor Green
    Write-Host "                 DIAGNOSTIC COMPLETE" -ForegroundColor Green
    Write-Host "============================================================" -ForegroundColor Green
    Write-Host ""
    Write-Host "HTML Report:" -ForegroundColor Cyan
    Write-Host $ReportFile -ForegroundColor White
    Write-Host ""

    # Open report
    Start-Process $ReportFile

}
catch {

    Write-Host ""
    Write-Host "ERROR: Unable to save HTML report." -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Red
}

Write-Host ""
Write-Host "Press any key to exit..."
$null = $Host.UI.RawUI.ReadKey("NoEcho,IncludeKeyDown")
```
