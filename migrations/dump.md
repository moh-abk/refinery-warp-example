# OICv1 Upgrade and Pre-Flight Health Validation Guide

> ⚠️ **Important Architectural Notice: Upgrade Path**  
> Direct upgrade from version 1.14 to 1.16 is **NOT** supported.
> 
> In Google Distributed Cloud air-gapped (GDC-AG) infrastructure, OICv1 upgrades are strictly sequential:  
> **Version 1.14.x ⟶ Version 1.15.x ⟶ Version 1.16.x**
> 
> * **1.14 → 1.15:** Requires minimum base version `1.14.3` to target `1.15.2+`.
> * **1.15 → 1.16:** Requires minimum base version `1.15.3` to target `1.16.1+`.
> 
> **Key Reason:** Configuration schemas (`config.ps1`), DSC resource modules, GPO structures, and MCM (v2403 → v2509 rebuild via runbook `IT-R0019`) rely on incremental transitions. Attempting to skip 1.15 will result in invalid MOF compilations and inconsistent DSC state.

---

## 1. Pre-Flight Health Validation Guide

Run these checks prior to initiating any upgrade procedure to ensure Active Directory, DSC, and Hyper-V subsystems are healthy.

### 1.1 Prerequisites & Access Verification

| Component | Target State | Verification Method |
| :--- | :--- | :--- |
| **Operator Account** | Domain/System Admin | Logged in with active `-SA` account. |
| **Admin Groups** | Validated | Member of `OC IT System Admins` and `MECM Server Admins`. |
| **Service Accounts** | Unexpired | Confirm passwords for `-SA`, `caadmin`, and `Marvin` are active. |
| **Host Disk Space** | $\ge 20\%$ free | Check root volumes on `CONFIG1` (`C:`) and Hyper-V hosts (`C:`, `H:`, `Z:`). |

### 1.2 Health Check PowerShell Scripts

Log into `CONFIG1` using an elevated PowerShell session with `-SA` credentials.

```powershell
# ==========================================
# 1. Load Current Configuration & Credentials
# ==========================================
. c:\config\config.ps1

$da_creds = Get-Credential -Message "Enter Domain Admin Credentials"
$sa_creds = Get-Credential -Message "Enter System Admin Credentials"

# ==========================================
# 2. Active Directory Replication Summary
# ==========================================
Write-Host "=== Active Directory Replication Summary ===" -ForegroundColor Cyan
repadmin /replsummary

# ==========================================
# 3. PDC Time Synchronization Status
# ==========================================
$pdc = $config.AllNodes.DomainConfig.GpoOwner
Write-Host "=== PDC Time Sync Status ($pdc) ===" -ForegroundColor Cyan
Invoke-Command -ComputerName $pdc -Credential $da_creds -ScriptBlock {
    w32tm /query /status /verbose
}

# ==========================================
# 4. DSC Health Status Across All Nodes
# ==========================================
Write-Host "=== Testing DSC Health Across All Nodes ===" -ForegroundColor Cyan
$dcs = $config.AllNodes | Where-Object { $_.Role -eq 'domain_controller' }
$non_dcs = $config.AllNodes | Where-Object { $_.Role -ne 'domain_controller' -and $_.Role -ne 'ca_root' -and $_.NodeName -ne '*' }

$dcs | ForEach-Object {
    $session = New-CimSession -ComputerName $_.NodeName -Credential $da_creds
    Get-DscConfigurationStatus -CimSession $session | 
        Select-Object HostName, Status, NumberOfResources, ResourcesNotInDesiredState
}

$non_dcs | ForEach-Object {
    $session = New-CimSession -ComputerName $_.NodeName -Credential $sa_creds
    Get-DscConfigurationStatus -CimSession $session | 
        Select-Object HostName, Status, NumberOfResources, ResourcesNotInDesiredState
}
```

### 1.3 Hyper-V Storage & VM Replication Health

Execute on `CONFIG1`:

```powershell
# Check Disk Space on Hyper-V Bare Metal Hosts (C:, H:, Z:)
$hypers = $config.AllNodes | Where-Object { $_.Role -eq 'hyper_v' }
$hypers | ForEach-Object {
    Invoke-Command -ComputerName $_.NodeName -Credential $sa_creds -ScriptBlock {
        Get-PSDrive -PSProvider FileSystem | Where-Object { $_.Name -in 'C', 'H', 'Z' } | 
            Select-Object @{N="Host";E={$env:COMPUTERNAME}}, Name, 
                          @{N="Free_GB";E={[math]::Round($_.Free/1GB,2)}}, 
                          @{N="Used_GB";E={[math]::Round($_.Used/1GB,2)}}
    }
}

# Check VM Replication Health
$hypers | ForEach-Object {
    Invoke-Command -ComputerName $_.NodeName -Credential $sa_creds -ScriptBlock {
        Get-VMReplication | Select-Object VMName, Health, State, PrimaryServer, ReplicaServer
    }
}
```

### 🛑 Go / No-Go Decision Criteria

1. `repadmin /replsummary` shows 0 failures.
2. `w32tm /query /status` shows `Last Sync Error: 0`.
3. All bare metal and VM nodes are reachable over WinRM and CIM.

---

## 2. OICv1 Step-by-Step Upgrade Execution (1.15 to 1.16)

### Phase A: Backup & VM Checkpoints

#### On CONFIG1 (Elevated Administrator Session):

```powershell
# 1. Load configuration
. c:\config\config.ps1

# 2. Take VM snapshots across all Hyper-V hosts
$vms = $config.AllNodes | Where-Object { $_.Role -ne 'hyper_v' -and $_.NodeName -ne '*' }
$vms | ForEach-Object {
    Checkpoint-VM -VMName $_.NodeName -SnapshotName "Checkpoint_$($_.NodeName)_$(Get-Date -Format 'yyyyMMddHHmmss')" -ComputerName $_.HyperVHost
}

# 3. Create local backup directories and isolate existing files
mkdir c:\oic_backup
mkdir c:\temp
Move-Item c:\dsc -Destination c:\oic_backup
Move-Item c:\config -Destination c:\oic_backup
Move-Item C:\release\operations_center -Destination c:\oic_backup

Copy-Item C:\oic_backup\config\config.ps1 -Destination c:\temp\config.backup.ps1
Copy-Item C:\oic_backup\config\certs -Destination C:\temp -Recurse
Copy-Item C:\oic_backup\config\creds -Destination C:\temp -Recurse

# 4. Enable long file path support
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem' -Name 'LongPathsEnabled' -Value 1

# 5. Move backup off C: drive to BM01 Z: drive
# (Replace <site> with your site code, e.g. DC1)
Move-Item c:\oic_backup "\\<site>-BM01\z$\oic_backup"
```

#### On BM01 (Elevated Administrator Session):

```powershell
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem' -Name 'LongPathsEnabled' -Value 1
Rename-Item -Path H:\operations_center -NewName operations_center_backup
```

---

### Phase B: Staging Release 1.16 Files

#### On BM01:

```powershell
Set-Location H:\
tar -zxvf prod_IT_component_bundle.tar.gz

Move-Item -Path H:\release\operations_center -Destination H:\
Remove-Item -Path 'H:\release' -Recurse -Force

# Stage files to CONFIG1 (Replace DC1-CONFIG1 with your config host name)
$config = "DC1-CONFIG1"
Copy-Item -Path H:\operations_center -Destination "\\$config\c$\release" -Recurse -Force
```

#### On CONFIG1:

```powershell
# Initialize config files and DSC modules
C:\release\operations_center\dsc\Initialize-ConfigHostFiles.ps1
C:\dsc\Install-PowerShellModules.ps1 -Path C:\dsc\modules -ErrorAction SilentlyContinue

# Restore previous certificates, credentials, and backup configuration
Move-Item C:\temp\config.backup.ps1 -Destination c:\config\config.backup.ps1
Move-Item C:\temp\certs -Destination C:\config
Move-Item C:\temp\creds -Destination C:\config

# Extract ETL tools
Expand-Archive -LiteralPath 'C:\dsc\etl-to-evtx.zip' -DestinationPath 'C:\config\third_party\' -Force
```

---

### Phase C: Configuration Merge & Compilation (Under Marvin)

#### 1. Launch Session as Marvin
1. Open PowerShell using **"Run as different user"** $\rightarrow$ User: `.\Marvin`.
2. Inside that window, elevate to Administrator: `Start-Process powershell.exe -Verb runas`
3. Run `whoami` to verify you are executing under `.\marvin`. Close all other open PowerShell sessions.

#### 2. Configuration & MOF Build

```powershell
# Copy new template
Copy-Item C:\dsc\config.example.ps1 -Destination c:\config\config.ps1

# Manually merge site parameters from c:\config\config.backup.ps1 into c:\config\config.ps1 using VS Code.
# Ensure you do not overwrite new schema items.

# Validate configuration
. c:\config\config.ps1

# Unblock yaml modules if required
Get-ChildItem -Path "C:\Program Files\WindowsPowerShell\Modules\powershell-yaml" -Recurse | Unblock-File

# Compile MECM files and MOFs
C:\dsc\Build-MecmFiles.ps1
C:\dsc\Build-Mof.ps1
```

#### 3. Initialize Update Arguments

```powershell
. 'c:\config\config.ps1'

$da_creds = Get-Credential -Message "Provide domain admin credentials"
$sa_creds = Get-Credential -Message "Provide system admin credentials"

$sa_args = @{
    Credential     = $sa_creds
    SetLcm         = $true
    RemoveExisting = $true
    CopyModules    = $true
}

$da_args = @{
    Credential     = $da_creds
    SetLcm         = $true
    RemoveExisting = $true
    CopyModules    = $true
}
```

---

### Phase D: Node-by-Node Upgrade Sequence

```
┌────────────────────────────────────────────────────────┐
│  1. Primary DC  ──►  2. Remaining DCs                  │
│       │                                                │
│  3. CA-ISSUING1 ──►  4. CA-WEB ──► 5. CA-ROOT1         │
│       │                                                │
│  6. ADFS1       ──►  7. Jumphosts                      │
│       │                                                │
│  8. Fileservers ──►  9. DHCP Servers (1, then 2)       │
│       │                                                │
│ 10. Userlock    ──► 11. Nessus                         │
│       │                                                │
│ 12. Toolboxes   ──► 13. Hyper-V Hosts (BM01, etc.)     │
│       │                                                │
│ 14. Splunk VMs  ──► 15. CONFIG1 Host                   │
│       │                                                │
│ 16. MCM Rebuild ──► 17. Re-enable Domain GPOs          │
└────────────────────────────────────────────────────────┘
```

#### Step 1: Domain Controllers

```powershell
$pdc = $config.AllNodes.DomainConfig.GpoOwner

# Clear legacy GPOs on PDC
Invoke-Command -Computername $pdc -Credential $da_creds -ScriptBlock {
    Remove-DscConfigurationDocument -Stage Current,Pending,Previous
    Get-GPO -All | Where-Object { $_.DisplayName -like "OIC*" } | Remove-GPO
    Get-GPO -All | Where-Object { $_.DisplayName -like "SITE*" -and $_.DisplayName -notlike "*SCCM*" } | Remove-GPO
    Get-Item "C:\config\domain_controller\oic_gpos" | Remove-Item -Recurse -Force
    Get-Item "C:\config\domain_controller\site_gpo*" | Remove-Item -Recurse -Force
}

# Update Primary DC
.\Update-RemoteHost.ps1 @da_args -ComputerName $pdc

# Update Remaining DCs
$dcs = $config.AllNodes | Where-Object { $_.Role -eq 'domain_controller' }
$dcs | Where-Object { $_.Nodename -ne "$pdc" } | ForEach-Object {
    c:\dsc\Update-RemoteHost.ps1 @da_args -ComputerName $_.NodeName
}

# Verify AD Replication
repadmin /replsummary
```

#### Step 2: Certificate Authority Subsystem

```powershell
# 1. Update CA-ISSUING1
$ca_iss = $config.AllNodes | Where-Object { $_.Role -eq "ca_issuing" }
c:\dsc\Update-RemoteHost.ps1 @sa_args -ComputerName $ca_iss.NodeName

# 2. Update CA-WEB
$ca_web = $config.AllNodes | Where-Object { $_.Role -eq "ca_web" }
c:\dsc\Update-RemoteHost.ps1 @sa_args -Computername $ca_web.NodeName

# 3. Update Offline CA-ROOT1
$ca_root = $config.AllNodes | Where-Object { $_.Role -eq "ca_root" }
$hvsession = New-CimSession -ComputerName $ca_root.HyperVHost -Credential $sa_creds
Start-VM -CimSession $hvsession -Name $ca_root.NodeName

$caroot_cred = Get-GeccoCredential -Name "$($ca_root.NodeName)\caadmin" -CredStore "c:\config\creds"
c:\dsc\Update-RemoteHost.ps1 -Computername $ca_root.NodeName -RemoteHost $ca_root.Ipv4Addr -Credential $caroot_cred

# Verify time peer sync on CA-ROOT1 before shutdown
Invoke-Command -ComputerName $ca_root.IPv4Addr -Credential $caroot_cred -ScriptBlock { w32tm /query /peers }
Stop-VM -CimSession $hvsession -Name $ca_root.NodeName
```

#### Step 3: Core Member Servers

```powershell
# ADFS
$config.AllNodes | Where-Object { $_.Role -eq 'adfs' } | ForEach-Object {
    c:\dsc\Update-RemoteHost.ps1 @sa_args -Computername $_.NodeName
}

# Jumphosts
$config.AllNodes | Where-Object { $_.Role -eq 'jumphost' } | ForEach-Object {
    c:\dsc\Update-RemoteHost.ps1 @sa_args -Computername $_.NodeName
}

# Fileservers
$config.AllNodes | Where-Object { $_.Role -eq 'file' } | ForEach-Object {
    c:\dsc\Update-RemoteHost.ps1 @sa_args -Computername $_.NodeName
}

# DHCP Primary & Failover
$dhcp1 = $config.AllNodes | Where-Object { $_.Role -eq 'dhcp_primary' }
c:\dsc\Update-RemoteHost.ps1 @sa_args -Computername $dhcp1.NodeName

$dhcp2 = $config.AllNodes | Where-Object { $_.Role -eq 'dhcp_failover' }
c:\dsc\Update-RemoteHost.ps1 @sa_args -Computername $dhcp2.NodeName

# Userlock (Remove lockfile logs prior to push)
$ulock1 = $config.AllNodes | Where-Object { $_.Role -eq 'userlock_primary' }
$ulock2 = $config.AllNodes | Where-Object { $_.Role -eq 'userlock_backup' }
Invoke-Command -ComputerName $ulock1.NodeName -Credential $sa_creds -Scriptblock {
    Remove-DscConfigurationDocument -Stage Current,Pending,Previous
    Remove-Item "c:\config\userlock_primary\ServiceImpersonation.log" -ErrorAction SilentlyContinue
}
Invoke-Command -ComputerName $ulock2.NodeName -Credential $sa_creds -Scriptblock {
    Remove-DscConfigurationDocument -Stage Current,Pending,Previous
    Remove-Item "c:\config\userlock_backup\ServiceImpersonation.log" -ErrorAction SilentlyContinue
}
c:\dsc\Update-RemoteHost.ps1 @sa_args -Computername $ulock1.NodeName
c:\dsc\Update-RemoteHost.ps1 @sa_args -Computername $ulock2.NodeName

# Nessus
$config.AllNodes | Where-Object { $_.Role -match 'nessus_' } | ForEach-Object {
    c:\dsc\Update-RemoteHost.ps1 @sa_args -Computername $_.NodeName
}

# Toolboxes
$config.AllNodes | Where-Object { $_.Role -eq 'toolbox' } | ForEach-Object {
    c:\dsc\Update-RemoteHost.ps1 @sa_args -Computername $_.NodeName
}
```

#### Step 4: Hyper-V Infrastructure Hosts

```powershell
$config.AllNodes | Where-Object { $_.Role -eq 'hyper_v' } | ForEach-Object {
    c:\dsc\Update-RemoteHost.ps1 @sa_args -Computername $_.NodeName
}
# Note: If BM01 reboots, the session on CONFIG1 will drop. Wait for host reboot, re-login as Marvin, re-populate credentials, and continue.
```

#### Step 5: Splunk Instances

```powershell
$sitecode = "DC1" # Set to your local site code
Set-Location c:\dsc
$splunk_roles = @("HEAVYFWD", "INDEXER1", "INDEXER2", "INDEXER3", "SPLUNKMGR", "SEARCHHEAD")
foreach ($sr in $splunk_roles) {
    .\Update-RemoteHost.ps1 -Computername "$sitecode-$sr" -Credential $sa_creds -SetLcm $true -RemoveExisting $true
}

# Restart Splunk services
$servers = ($config.AllNodes | Where-Object { $_.Role -match "splunk_" }).NodeName
Invoke-Command -ComputerName $servers -Credential $sa_creds -ScriptBlock { & c:\splunk\bin\splunk.exe restart } -ErrorAction Continue
```

#### Step 6: CONFIG1 Host

```powershell
Start-DscConfiguration -ComputerName $env:COMPUTERNAME -Path c:\config\mofs -Verbose -Wait -Force
```

#### Step 7: MCM Rebuild (v2509)

In-place MCM upgrades within OIC are unsupported for 1.16. Execute runbook `IT-R0019` to redeploy MECM to version 2509.

#### Step 8: Re-Enable Domain Group Policy Objects

```powershell
# Enable links in GPO mapping
$gpolinks = (Get-Content C:\dsc\data\GpoLinkMapping.yaml -Raw).Replace("LinkEnabled: 'No'", "LinkEnabled: 'Yes'")
$gpolinks | Out-File C:\dsc\data\GpoLinkMapping.yaml -Force

# Push updated GPOs to PDC
c:\dsc\Update-RemoteHost.ps1 -Computername $config.AllNodes.DomainConfig.GpoOwner -Credential $da_creds
```

---

### Phase E: Post-Upgrade Verification & Snapshot Cleanup

1. **Validate All Systems Operational:**  
   Confirm AD replication, DNS, DHCP, Web/ADFS endpoints, and Splunk ingestion are responding without error.

2. **Remove Temporary Checkpoints:**  
   *(Run **ONLY** after all roles and nodes have passed end-to-end verification)*

```powershell
$config.AllNodes | Where-Object { $_.Role -eq "hyper_v" } | ForEach-Object {
    Invoke-Command -ComputerName $_.NodeName -Credential $sa_creds -Scriptblock {
        Get-VM | Get-VMSnapshot | Where-Object { $_.Name -like "Checkpoint_*" } | Remove-VMSnapshot -Verbose
    }
}
```
