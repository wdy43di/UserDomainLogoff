Import-Module ActiveDirectory

# User to log off
$targetUser = "user"

# Log setup
$LogPath = "C:\Logs"

if (!(Test-Path $LogPath)) {
    New-Item -Path $LogPath -ItemType Directory -Force | Out-Null
}

$LogFile = Join-Path $LogPath "$($targetUser)_Logoff_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"

function Write-Log {
    param([string]$Message)

    $TimeStamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $Entry = "$TimeStamp - $Message"

    Write-Host $Entry
    Add-Content -Path $LogFile -Value $Entry
}

# Counters
$ComputersScanned = 0
$SkippedCount = 0
$OfflineCount = 0
$SessionCount = 0
$SuccessCount = 0
$FailureCount = 0

# Computers where user was found
$LoggedOffComputers = @()

Write-Log "======================================================="
Write-Log "STARTING DOMAIN-WIDE LOGOFF"
Write-Log "Target User: $targetUser"
Write-Log "======================================================="

# Get all AD computers
$computers = Get-ADComputer -Filter * -Property Name |
    Select-Object -ExpandProperty Name

foreach ($computer in $computers) {

    $ComputersScanned++

    # Skip likely servers
    if (
        $computer -match "^(DC|SRV|SQL|FS|PRT|FILE|VM|APP|TDC|EXCH|WEB)" -or
        $computer -like "*-SVR" -or
        $computer -like "*SRV*"
    ) {
        $SkippedCount++
        Write-Log "SKIPPED: $computer"
        continue
    }

    # Check reachability
    if (-not (Test-Connection -ComputerName $computer -Count 1 -Quiet)) {
        $OfflineCount++
        Write-Log "OFFLINE: $computer"
        continue
    }

    Write-Log "CHECKING: $computer"

    try {

        $sessions = quser /server:$computer 2>$null

        if ($sessions) {

            $sessions = $sessions | Select-Object -Skip 1

            foreach ($line in $sessions) {

                $line = $line.Trim()

                if ($line -match "^\>?\s*$targetUser\s") {

                    $SessionCount++

                    $columns = $line -split '\s+'

                    if ($columns.Count -ge 3) {

                        $sessionId = $columns[2]

                        try {

                            logoff $sessionId /server:$computer

                            $SuccessCount++
                            $LoggedOffComputers += $computer

                            Write-Log "SUCCESS: Logged off $targetUser from $computer (Session ID $sessionId)"

                        }
                        catch {

                            $FailureCount++

                            Write-Log "FAILED: Could not log off session $sessionId on $computer - $_"

                        }
                    }
                }
            }
        }

    }
    catch {

        $FailureCount++

        Write-Log "ERROR: Failed to query $computer - $_"

    }
}

Write-Log ""
Write-Log "======================================================="
Write-Log "SUMMARY"
Write-Log "======================================================="
Write-Log "Target User       : $targetUser"
Write-Log "Computers Scanned : $ComputersScanned"
Write-Log "Servers Skipped   : $SkippedCount"
Write-Log "Offline Systems   : $OfflineCount"
Write-Log "Sessions Found    : $SessionCount"
Write-Log "Successful Logoffs: $SuccessCount"
Write-Log "Failures          : $FailureCount"
Write-Log ""

Write-Log "SYSTEMS WHERE USER WAS FOUND"

if ($LoggedOffComputers.Count -gt 0) {

    $LoggedOffComputers |
    Sort-Object -Unique |
    ForEach-Object {
        Write-Log " - $_"
    }

}
else {

    Write-Log " - No active sessions found"

}

Write-Log ""
Write-Log "Log File: $LogFile"
Write-Log "======================================================="

Write-Host ""
Write-Host "Completed. Log saved to $LogFile" -ForegroundColor Green
