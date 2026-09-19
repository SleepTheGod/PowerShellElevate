# PowerShellElevate

A small Windows PowerShell proof of concept for launching an elevated PowerShell session through the native Windows UAC `RunAs` execution mechanism.

PowerShellElevate is intentionally simple. The project demonstrates elevation detection, UAC invocation, and execution of a command inside the elevated PowerShell process without requiring `Add-Type`, custom native API declarations, or Base64 encoded commands.

Repository

https://github.com/SleepTheGod/PowerShellElevate

Main source file

`poc.ps`

## Overview

PowerShellElevate checks whether the current PowerShell process is running with administrator privileges.

When administrator privileges are not present, the script launches Windows PowerShell again using the `RunAs` execution verb.

Windows then presents the standard User Account Control prompt.

After elevation, the new PowerShell process executes the demonstration payload and reports that the administrator session is active.

If the original process is already elevated, the script skips the elevation request and executes the elevated demonstration directly.

## Features

* Windows PowerShell 5.1 compatible
* Windows 11 compatible
* Administrator privilege detection
* Standard Windows UAC elevation
* Uses the built in `RunAs` mechanism
* No `Add-Type`
* No custom DLL compilation
* No manually defined Win32 structures
* No Base64 command encoding
* No external dependencies
* Designed to work from an interactive PowerShell session
* Simple proof of concept structure
* Suitable as a starting point for administrative automation

## Project Structure

```text
PowerShellElevate
│
├── poc.ps
└── README.md
```

## Requirements

* Windows 10 or Windows 11
* Windows PowerShell 5.1
* User Account Control enabled
* Permission to approve the UAC elevation request

PowerShell 7 is not required.

## Usage

Clone the repository

```powershell
git clone https://github.com/SleepTheGod/PowerShellElevate.git
```

Enter the project directory

```powershell
cd PowerShellElevate
```

Run the proof of concept

```powershell
powershell.exe -NoProfile -File .\poc.ps
```

Alternatively, the contents of `poc.ps` can be pasted directly into an interactive Windows PowerShell session.

## Expected Behavior

When the current PowerShell process is not elevated, Windows displays the normal UAC administrator prompt.

After approval, a new PowerShell process opens.

The demonstration displays information similar to

```text
========================================
 WINDOWS 11 ELEVATED POWERSHELL
========================================

Computer  DESKTOP
User      Administrator

hello world!

Administrator PowerShell is active.
```

When PowerShell is already elevated, the script skips the UAC request and displays

```text
========================================
 ALREADY ELEVATED
========================================

hello world!
```

## How It Works

The script first identifies the Windows PowerShell executable located under the Windows system directory.

It then checks the current Windows identity and determines whether that identity belongs to the local Administrator role.

If administrator privileges are unavailable, the script constructs a small PowerShell command and launches Windows PowerShell with the `RunAs` execution verb.

The `RunAs` verb invokes the normal Windows UAC elevation workflow.

Once the user approves the request, Windows starts the new PowerShell process with the requested administrator token.

The elevated process then executes the demonstration command.

## Constrained Language Mode

This proof of concept was designed around a situation where PowerShell Constrained Language Mode prevented several approaches based on custom .NET types and native API declarations.

The implementation therefore avoids techniques such as

```text
Add-Type
DllImport
ShellExecuteEx declarations
custom Win32 structures
Base64 command generation
```

Instead, it relies on the PowerShell `Start-Process` cmdlet and the Windows `RunAs` execution verb.

Constrained Language Mode itself is a security control and this project does not attempt to bypass that control.

## Security Considerations

PowerShellElevate is a proof of concept demonstrating a normal Windows administrator elevation workflow.

It does not bypass UAC.

It does not exploit a Windows vulnerability.

It does not attempt to obtain administrator privileges without user authorization.

The UAC prompt remains controlled by Windows.

The project should therefore be understood as an administrative process launching technique rather than a UAC bypass.

## Why This Exists

The project was created to provide a minimal reference implementation for PowerShell scripts that need to determine whether they are elevated and request administrator privileges when they are not.

The implementation deliberately avoids unnecessary complexity.

There are no compiled binaries.

There are no external modules.

There are no third party dependencies.

There is only a small PowerShell proof of concept.

## Customizing the Elevated Payload

The demonstration command can be replaced with an administrative task appropriate for your environment.

For example, the payload could be extended to perform system administration tasks such as

```powershell
Get-Service
```

```powershell
Get-NetAdapter
```

```powershell
Get-NetIPAddress
```

```powershell
Get-Process
```

Administrative operations should only be performed on systems where you have authorization.

## Limitations

This project depends on the normal Windows UAC infrastructure.

If UAC is disabled or restricted by organizational policy, behavior may differ.

Security products and enterprise application control policies may also restrict PowerShell execution.

Constrained Language Mode can restrict other PowerShell functionality even after elevation.

Elevation does not automatically remove security policies enforced by Windows Defender Application Control, AppLocker, or other application control systems.

## Disclaimer

PowerShellElevate is provided for educational purposes, Windows administration, security research, and authorized testing.

Do not use administrative capabilities against systems or accounts without authorization.

The author does not provide any guarantee that the proof of concept will operate identically across every Windows configuration.

## License

Choose a license appropriate for your intended use.

A permissive option for this type of small proof of concept is the MIT License.

## Author

Taylor Christian Newsome

GitHub

https://github.com/SleepTheGod

Project

https://github.com/SleepTheGod/PowerShellElevate
