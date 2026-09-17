Windows Client OS System Requirement Pre-flight Checker

A lightweight PowerShell one-liner script designed to validate local system requirements on Windows Client OS (Windows 10/11) before deploying Invicti, WSL2, or Docker containers. It checks hardware limits, required Windows optional features, local port availability, and network connectivity to required registries.

---

WHAT IT CHECKS

* Operating System: OS Caption and Build Number (Windows 10/11 Client).
* Hardware Specs:
* CPU Cores: Checks logical processor count (Target: 6 Cores).
* RAM: Total physical memory in GB (Target: 20+ GB).
* Disk Space: Available space on drive C: (Target: 100 GB).


* Virtualization & Windows Features:
* Hyper-V Presence
* VirtualMachinePlatform Feature Status
* Microsoft-Windows-Subsystem-Linux Feature Status
* Installed WSL version details


* Port Availability:
* Port 443 (HTTPS)
* Port 8088


* Network Connectivity:
* [https://platform-registry.invicti.com/v2/](https://www.google.com/search?q=https://platform-registry.invicti.com/v2/)
* [https://registry-1.docker.io/v2/](https://www.google.com/search?q=https://registry-1.docker.io/v2/)
* [https://activation.invicti.com](https://www.google.com/search?q=https://activation.invicti.com)



---

PREREQUISITES

* OS: Windows 10 or Windows 11 (Desktop/Client edition)
* Execution Context: Run from an elevated PowerShell prompt (Run as Administrator) to query optional feature status.

---

HOW TO RUN

1. Open PowerShell as Administrator on your Windows 10/11 machine.
2. Copy and paste the command below into the terminal window:

; $os=Get-CimInstance Win32_OperatingSystem; $cs=Get-CimInstance Win32_ComputerSystem; "OS       : $($os.Caption) build$($os.BuildNumber)"; "Cores    : $($cs.NumberOfLogicalProcessors)  (need 6)"; "RAM      : $([math]::Round($cs.TotalPhysicalMemory/1GB,1)) GB  (need 20+)"; "Free C:  : $([math]::Round((Get-PSDrive C).Free/1GB,1)) GB  (need 100)"; "HyperV OK: $((Get-ComputerInfo).HyperVisorPresent)"; foreach($f in 'VirtualMachinePlatform','Microsoft-Windows-Subsystem-Linux'){ "$f :$((Get-WindowsOptionalFeature -Online -FeatureName $f).State)" }; foreach($p in 443,8088){ $busy = Get-NetTCPConnection -LocalPort$p -State Listen -ErrorAction SilentlyContinue; "Port $p  :$(if($busy){'IN USE'}else{'free'})" }; wsl --version 2>&1 \vert{} Select-Object -First 2; foreach($u in '[https://platform-registry.invicti.com/v2/','https://registry-1.docker.io/v2/','https://activation.invicti.com](https://platform-registry.invicti.com/v2/','https://registry-1.docker.io/v2/','https://activation.invicti.com)'){ $c=try{(Invoke-WebRequest $u -UseBasicParsing -TimeoutSec 10).StatusCode}catch{$_.Exception.Response.StatusCode.value__}; "$u ->$(if($c){$c}else{'UNREACHABLE'})" };

---

EXAMPLE OUTPUT

OS       : Microsoft Windows 11 Enterprise build 22631
Cores    : 8  (need 6)
RAM      : 32 GB  (need 20+)
Free C:  : 142.5 GB  (need 100)
HyperV OK: True
VirtualMachinePlatform : Enabled
Microsoft-Windows-Subsystem-Linux : Enabled
Port 443  : free
Port 8088  : free
WSL version: 2.1.5.0
Kernel version: 5.15.153.1-2
[https://platform-registry.invicti.com/v2/](https://www.google.com/search?q=https://platform-registry.invicti.com/v2/) -> 200
[https://registry-1.docker.io/v2/](https://www.google.com/search?q=https://registry-1.docker.io/v2/) -> 401
[https://activation.invicti.com](https://www.google.com/search?q=https://activation.invicti.com) -> 200

Note on Status Codes: A status code return of 200 or 401 indicates that the URL is successfully reachable across the network. A 401 Unauthorized response from Docker registry endpoints is expected when connecting without authentication tokens.