# UserDomainLogoff
Input user, logs off user on any pc in domain


!! Need to add it as a Powershell Code.. But..

Import-Module ActiveDirectory

$targetUser = "*User*"  # Replace with the actual username
$computers = Get-ADComputer -Filter * -Property Name | Select-Object -ExpandProperty Name

foreach ($computer in $computers) {
    # Skip if computer name suggests it's a server
    if ($computer -match "^(DC|SRV|SQL|FS|PRT|FILE|VM|APP|TDC|EXCH|WEB)" -or $computer -like "*-SVR*" -or $computer -like "*SRV*") {
        Write-Verbose "Skipping server-like computer: $computer"
        continue
    }

    if (Test-Connection -ComputerName $computer -Count 1 -Quiet) {
        try {
            $sessions = quser /server:$computer 2>$null
            if ($sessions) {
                foreach ($line in $sessions) {
                    if ($line -match $targetUser) {
                        $sessionId = ($line -split '\s+')[2]
                        logoff $sessionId /server:$computer
                        Write-Host "✅ Logged off $targetUser from $computer (Session $sessionId)"
                    }
                }
            } else {
                Write-Verbose "No sessions found on $computer"
            }
        } catch {
            Write-Warning "⚠️ Failed to query or log off on ${computer}: $_"
        }
    } else {
        Write-Verbose "❌ $computer is offline or unreachable."
    }
}
