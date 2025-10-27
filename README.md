Import-Module ActiveDirectory

$targetUser = "*UserName*"  # Replace with the actual username
$computers = Get-ADComputer -Filter * -Property Name | Select-Object -ExpandProperty Name

foreach ($computer in $computers) {
    # Skip if computer name suggests it's a server
    if ($computer -match "^(DC|SRV|SQL|FS|PRT|FILE|VM|APP|TDC|EXCH|WEB)" -or $computer -like "*-SVR*" -or $computer -like "*SRV*") {
        Write-Host "Skipping server-like computer: $computer"
        continue
    }

    if (Test-Connection -ComputerName $computer -Count 1 -Quiet) {
        try {
            $sessions = quser /server:$computer 2>$null

            if ($sessions) {
                # Skip the header line
                $sessions = $sessions | Select-Object -Skip 1

                foreach ($line in $sessions) {
                    $line = $line.Trim()
                    if ($line -match $targetUser) {
                        # Quser output can have variable spacing; extract the session ID properly
                        $columns = $line -split '\s+'
                        $sessionId = $columns[2]  # Usually the session ID column
                        
                        try {
                            logoff $sessionId /server:$computer
                            Write-Host ("✅ Logged off " + $targetUser + " from " + $computer + " (Session " + $sessionId + ")")
                        } catch {
                            Write-Warning ("⚠️ Failed to log off session " + $sessionId + " on " + $computer + ": " + $_)
                        }
                    }
                }
            } else {
                Write-Host "No sessions found on $computer"
            }
        } catch {
            Write-Warning ("⚠️ Failed to query sessions on " + $computer + ": " + $_)
        }
    } else {
        Write-Host "❌ $computer is offline or unreachable."
    }
}
