Windows Server OS System Requirement Pre-flight Checker

A lightweight PowerShell one-liner script designed to validate local system requirements on Windows Server OS before deploying Invicti, WSL2, Hyper-V, or Docker containers. It checks hardware specs, CPU virtualization extensions, local port availability with process mapping, Windows Server features, and service statuses.

---

WHAT IT CHECKS

* Operating System & Hardware:
* OS Caption and Build Number (Windows Server Edition).
* Hardware Manufacturer and Model.
* CPU Model Name.
* Cores: Available logical processors vs. target (Target: 6 Cores).
* RAM: Total physical memory in GB vs. target (Target: 20 GB).
* Disk Space: Available space on drive C: in GB vs. target (Target: 100 GB).


* Virtualization & WSL Setup:
* WSL Version (installed version string).
* Hardware Virtualization Flags: Hardware Virtualization (VT-x/AMD-V), Second Level Address Translation (SLAT), and Hyper-V Hypervisor presence.


* Port Availability & Process Mapping:
* Port 443 (HTTPS)
* Port 8088
* Displays whether the port is free or identifies the process name currently binding it.


* Windows Server Optional Features:
* Microsoft-Hyper-V
* Microsoft-Hyper-V-Management-PowerShell
* VirtualMachinePlatform
* Microsoft-Windows-Subsystem-Linux
* Microsoft-Hyper-V-Online
* Microsoft-Hyper-V-Offline
* RSAT-Hyper-V-Tools-Feature


* Service Status & Startup Modes:
* Virtual Disk Service (vds)
* Hyper-V Virtual Machine Management Service (vmms)



---

PREREQUISITES

* OS: Windows Server (Windows Server 2019, 2022, or newer)
* Execution Context: Run from an elevated PowerShell prompt (Run as Administrator) to query Windows Server features and active service configurations.

---

HOW TO RUN

1. Open PowerShell as Administrator on your Windows Server machine.
2. Copy and paste the command below into the terminal window:

$os=Get-CimInstance Win32_OperatingSystem;$cs=Get-CimInstance Win32_ComputerSystem;$cpu=@(Get-CimInstance Win32_Processor)[0];"OS    : $($os.Caption) build$($os.BuildNumber)";"HW    : $($cs.Manufacturer)/$($cs.Model)";"CPU   : $($cpu.Name.Trim())";"Cores : $($cs.NumberOfLogicalProcessors) / 6";"RAM   : $([math]::Round($cs.TotalPhysicalMemory/1GB,1))GB / 20";"Disk  : $([math]::Round((Get-PSDrive C).Free/1GB,1))GB / 100";"WSL   : $((wsl --version 2>&1\vert{}Select -First 1))";"VT-x  : $($cpu.VirtualizationFirmwareEnabled)  SLAT:$($cpu.SecondLevelAddressTranslationExtensions)  HV:$((Get-ComputerInfo).HyperVisorPresent)";443,8088|%{"Port $_`: $(if($b=Get-NetTCPConnection -LocalPort$_ -State Listen -EA 0){"IN USE by $((Get-Process -Id$b[0].OwningProcess -EA 0).ProcessName)"}else{'free'})"};'Microsoft-Hyper-V','Microsoft-Hyper-V-Management-PowerShell','VirtualMachinePlatform','Microsoft-Windows-Subsystem-Linux','Microsoft-Hyper-V-Online','Microsoft-Hyper-V-Offline','RSAT-Hyper-V-Tools-Feature'|%{"$_ :$((Get-WindowsOptionalFeature -Online -FeatureName $_ -EA 0).State)"};'vds','vmms'\vert{}%{$s=Get-Service $_ -EA 0;"$_ : $(if($s){"$($s.Status) / $((Get-CimInstance Win32_Service -Filter "Name='$_'").StartMode)"}else{'NOT INSTALLED'})"}

---

EXAMPLE OUTPUT

OS    : Microsoft Windows Server 2022 Datacenter build 20348
HW    : Dell Inc./PowerEdge R640
CPU   : Intel(R) Xeon(R) Silver 4210 CPU @ 2.20GHz
Cores : 20 / 6
RAM   : 64GB / 20
Disk  : 250.4GB / 100
WSL   : WSL version: 2.1.5.0
VT-x  : True  SLAT: True  HV: True
Port 443: IN USE by inetinfo
Port 8088: free
Microsoft-Hyper-V : Enabled
Microsoft-Hyper-V-Management-PowerShell : Enabled
VirtualMachinePlatform : Enabled
Microsoft-Windows-Subsystem-Linux : Enabled
Microsoft-Hyper-V-Online : Disabled
Microsoft-Hyper-V-Offline : Disabled
RSAT-Hyper-V-Tools-Feature : Enabled
vds : Running / Auto
vmms : Running / Auto

Note on Ports & Services: If Port 443 or 8088 displays "IN USE", check the identified process name to avoid port conflicts prior to deployment.