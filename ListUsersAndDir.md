```powershell
#requires -Version 5.1
<#
.SYNOPSIS
    Lists directories and local users on the Windows system.

.DESCRIPTION
    Read-only inventory script that:
      - Lists local user accounts
      - Lists local groups
      - Lists directories on selected drives
      - Shows directory ownership
      - Shows directory ACL permissions
      - Shows hidden/system directories
      - Exports results to CSV files

.NOTES
    Read-only. Does not create, modify, delete, or change permissions.
#>

[CmdletBinding()]
param(
    [string]$OutputDirectory = "$env:USERPROFILE\Desktop\SystemInventory",
    [string[]]$Drives = @("C:")
)

$ErrorActionPreference = "SilentlyContinue"

# ----------------------------------------------------------------------
# Output directory
# ----------------------------------------------------------------------

New-Item -ItemType Directory -Path $OutputDirectory -Force | Out-Null

Write-Host ""
Write-Host "============================================" -ForegroundColor Cyan
Write-Host " Windows System Directory/User Inventory" -ForegroundColor Cyan
Write-Host "============================================" -ForegroundColor Cyan
Write-Host ""

# ----------------------------------------------------------------------
# System information
# ----------------------------------------------------------------------

$SystemInfo = [PSCustomObject]@{
    ComputerName = $env:COMPUTERNAME
    UserName     = $env:USERNAME
    Domain       = $env:USERDOMAIN
    OS           = (Get-CimInstance Win32_OperatingSystem).Caption
    Version      = (Get-CimInstance Win32_OperatingSystem).Version
    Architecture = (Get-CimInstance Win32_OperatingSystem).OSArchitecture
    PowerShell   = $PSVersionTable.PSVersion.ToString()
}

$SystemInfo |
    Export-Csv "$OutputDirectory\system-info.csv" -NoTypeInformation -Encoding UTF8

# ----------------------------------------------------------------------
# Local users
# ----------------------------------------------------------------------

Write-Host "[+] Local users" -ForegroundColor Green

$Users = Get-CimInstance Win32_UserAccount -Filter "LocalAccount=True" |
    Select-Object `
        Name,
        FullName,
        Description,
        Disabled,
        Lockout,
        PasswordRequired,
        PasswordChangeable,
        SID

$Users |
    Format-Table -AutoSize

$Users |
    Export-Csv "$OutputDirectory\local-users.csv" -NoTypeInformation -Encoding UTF8

# ----------------------------------------------------------------------
# Local groups
# ----------------------------------------------------------------------

Write-Host ""
Write-Host "[+] Local groups" -ForegroundColor Green

$Groups = Get-CimInstance Win32_Group -Filter "LocalAccount=True" |
    Select-Object Name, Description, SID

$Groups |
    Format-Table -AutoSize

$Groups |
    Export-Csv "$OutputDirectory\local-groups.csv" -NoTypeInformation -Encoding UTF8

# ----------------------------------------------------------------------
# Group membership
# ----------------------------------------------------------------------

Write-Host ""
Write-Host "[+] Local group membership" -ForegroundColor Green

$GroupMembers = foreach ($Group in $Groups) {

    try {
        $Members = Get-LocalGroupMember -Group $Group.Name -ErrorAction Stop

        foreach ($Member in $Members) {
            [PSCustomObject]@{
                Group        = $Group.Name
                Member       = $Member.Name
                ObjectClass  = $Member.ObjectClass
                PrincipalSource = $Member.PrincipalSource
            }
        }
    }
    catch {
        # Fallback for systems without Microsoft.PowerShell.LocalAccounts
        try {
            $Members = Get-CimInstance Win32_GroupUser |
                Where-Object {
                    $_.GroupComponent -match [regex]::Escape($Group.Name)
                }

            foreach ($Member in $Members) {
                [PSCustomObject]@{
                    Group           = $Group.Name
                    Member          = $Member.PartComponent
                    ObjectClass     = "Unknown"
                    PrincipalSource = "CIM"
                }
            }
        }
        catch {}
    }
}

$GroupMembers |
    Format-Table -AutoSize

$GroupMembers |
    Export-Csv "$OutputDirectory\group-membership.csv" -NoTypeInformation -Encoding UTF8

# ----------------------------------------------------------------------
# Directory inventory
# ----------------------------------------------------------------------

Write-Host ""
Write-Host "[+] Scanning directories" -ForegroundColor Green
Write-Host ""

$Directories = foreach ($Drive in $Drives) {

    if (-not (Test-Path $Drive)) {
        Write-Warning "Drive not found: $Drive"
        continue
    }

    Write-Host "Scanning $Drive ..." -ForegroundColor Yellow

    Get-ChildItem `
        -LiteralPath "$Drive\" `
        -Directory `
        -Force `
        -Recurse `
        -ErrorAction SilentlyContinue |
    ForEach-Object {

        $Owner = $null

        try {
            $ACL = Get-Acl -LiteralPath $_.FullName -ErrorAction Stop
            $Owner = $ACL.Owner
        }
        catch {}

        [PSCustomObject]@{
            Path          = $_.FullName
            Name          = $_.Name
            Parent        = $_.Parent.FullName
            Attributes    = $_.Attributes
            CreationTime  = $_.CreationTime
            LastWriteTime = $_.LastWriteTime
            Owner         = $Owner
        }
    }
}

$Directories |
    Export-Csv "$OutputDirectory\directories.csv" -NoTypeInformation -Encoding UTF8

Write-Host ""
Write-Host "[+] Directory count: $($Directories.Count)" -ForegroundColor Cyan

# ----------------------------------------------------------------------
# Top-level directories
# ----------------------------------------------------------------------

Write-Host ""
Write-Host "[+] Top-level directories" -ForegroundColor Green

$TopLevel = foreach ($Drive in $Drives) {

    if (Test-Path $Drive) {
        Get-ChildItem `
            -LiteralPath "$Drive\" `
            -Directory `
            -Force `
            -ErrorAction SilentlyContinue |
        Select-Object `
            FullName,
            Name,
            Attributes,
            CreationTime,
            LastWriteTime
    }
}

$TopLevel |
    Format-Table -AutoSize

$TopLevel |
    Export-Csv "$OutputDirectory\top-level-directories.csv" `
        -NoTypeInformation `
        -Encoding UTF8

# ----------------------------------------------------------------------
# User profile directories
# ----------------------------------------------------------------------

Write-Host ""
Write-Host "[+] User profile directories" -ForegroundColor Green

$Profiles = Get-ChildItem "C:\Users" `
    -Directory `
    -Force `
    -ErrorAction SilentlyContinue |
    Select-Object `
        Name,
        FullName,
        CreationTime,
        LastWriteTime,
        Attributes

$Profiles |
    Format-Table -AutoSize

$Profiles |
    Export-Csv "$OutputDirectory\user-profiles.csv" `
        -NoTypeInformation `
        -Encoding UTF8

# ----------------------------------------------------------------------
# Directory ACL inventory
# ----------------------------------------------------------------------

Write-Host ""
Write-Host "[+] Collecting directory permissions" -ForegroundColor Green
Write-Host "    This can take a while on large drives."
Write-Host ""

$ACLResults = foreach ($Directory in $Directories) {

    try {

        $ACL = Get-Acl -LiteralPath $Directory.Path -ErrorAction Stop

        foreach ($Access in $ACL.Access) {

            [PSCustomObject]@{
                Path              = $Directory.Path
                Owner             = $ACL.Owner
                IdentityReference = $Access.IdentityReference
                FileSystemRights  = $Access.FileSystemRights
                AccessControlType = $Access.AccessControlType
                IsInherited        = $Access.IsInherited
                InheritanceFlags  = $Access.InheritanceFlags
                PropagationFlags  = $Access.PropagationFlags
            }
        }
    }
    catch {}
}

$ACLResults |
    Export-Csv "$OutputDirectory\directory-permissions.csv" `
        -NoTypeInformation `
        -Encoding UTF8

# ----------------------------------------------------------------------
# Summary
# ----------------------------------------------------------------------

$Summary = [PSCustomObject]@{
    ComputerName       = $env:COMPUTERNAME
    LocalUsers         = @($Users).Count
    LocalGroups        = @($Groups).Count
    GroupMemberships   = @($GroupMembers).Count
    DirectoriesFound   = @($Directories).Count
    ACLEntries         = @($ACLResults).Count
    UserProfiles       = @($Profiles).Count
    OutputDirectory    = $OutputDirectory
    Generated          = Get-Date
}

$Summary |
    Format-List

$Summary |
    Export-Csv "$OutputDirectory\summary.csv" `
        -NoTypeInformation `
        -Encoding UTF8

Write-Host ""
Write-Host "============================================" -ForegroundColor Cyan
Write-Host " Inventory complete" -ForegroundColor Green
Write-Host "============================================" -ForegroundColor Cyan
Write-Host ""
Write-Host "Results:" -ForegroundColor Yellow
Write-Host "  $OutputDirectory"
Write-Host ""
Write-Host "Files:"
Get-ChildItem $OutputDirectory -File |
    Select-Object -ExpandProperty Name
Write-Host ""
```

### Run it

Save as:

```powershell
SystemInventory.ps1
```

Then:

```powershell
Set-ExecutionPolicy -Scope Process Bypass
.\SystemInventory.ps1
```

To scan multiple drives:

```powershell
.\SystemInventory.ps1 -Drives C:,D:,E:
```

To choose another output directory:

```powershell
.\SystemInventory.ps1 -OutputDirectory "C:\SystemInventory"
```

The main results will be:

```text
SystemInventory\
├── system-info.csv
├── local-users.csv
├── local-groups.csv
├── group-membership.csv
├── directories.csv
├── top-level-directories.csv
├── user-profiles.csv
├── directory-permissions.csv
└── summary.csv
```

It is **read-only** and doesn't change users, directories, ACLs, or system configuration.
