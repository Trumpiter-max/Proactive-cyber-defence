# Security

## Table of content
- [Security rating](#security-rating)
- [Influenced the design of Windows](#influenced-the-design-of-windows)
- [Virtualization-based security](#virtualization-based-security)
    - [Credential Guard](#credential-guard)
    - [Device Guard](#device-guard)
- [Protecting objects](#protecting-objects)
- [The AuthZ API](#the-authz-api)
- [Account rights and privileges](#account-rights-and-privileges)
- [Access tokens of processes and threads](#access-tokens-of-processes-and-threads)
- [Security auditing](#security-auditing)
- [AppContainers](#appcontainers)
- [Logon](#logon)
- [User Account Control and virtualization](#user-account-control-and-virtualization)
- [Exploit mitigations](#exploit-mitigations)
- [Application Identification](#application-identification)
- [AppLocker](#applocker)
- [Software Restriction Policies](#Software-Restriction-Policies)
- [Kernel Patch Protection](#Kernel-Patch-Protection)
- [PatchGuard](#PatchGuard)
- [HyperGuard](#HyperGuard)
----

Influenced design and implementation by `stringent requirements` 

## Security rating

Current security rating standard: `Common Criteria (CC) - 1996`:

- More **flexible and closer** to the `ITSEC` than the `TCSEC`  
- `Security system components`: core components and databases 
- `Security reference monitor (SRM)`: in the `Windows executive - Ntoskrnl.exe`, represent a *security context, security access check, user right, security audit messages*
- `Local Security Authority Subsystem Service (Lsass)`: local system *security policy*, *user authentication*, send security audit messages to the event log, `Lsasrv.dll` - a library that `Lsass` loads
- `LSAIso.exe (Credential Guard)`: used by `Lsass`, store *users’ token hashes*
- `Lsass policy database`: under `HKLM\SECURITY`, contains the local system security policy settings, logon information used for cached *domain logons* and Windows service *user-account logons*
- `Security Accounts Manager (SAM)`:  in `Samsrv.dll`, manage the database containing the user names and groups defined on the *local machine*
- `SAM database`: in the *registry* under `HKLM\SAMstore` defined local users and groups, system’s administrator recovery account
- `Active Directory`: directory service containing a database of objects in *single entity*, implemented as `Ntdsa.dll`
-  `Authentication packages`:  include `dynamic link libraries (DLLs)` , authenticate user’s security identity
- `Interactive logon manager (Winlogon)`:  `Winlogon.exe`, response to the SAS, manage interactive logon sessions
- `Logon user interface (LogonUI)`: `LogonUI.exe`, presents users with the user interface to authenticate, query user credentials through various methods
- `Credential providers (CPs)`:  are `authui.dll`, `SmartcardCredentialProvider.dll`, `BioCredProv.Dll`, `FaceCredentialProvider.dll`, a face-detection provider
- `Network logon service (Netlogon)`: `Netlogon.dll` sets up the secure channel to a domain controller
- `Kernel Security Device Driver (KSecDD)`: kernel-mode library `(%SystemRoot%\System32\Drivers\Ksecdd.sys)`, `Encrypting File System (EFS)`, use to communicate with `Lsass`
- `AppLocker`: consists of a driver `(%SystemRoot%\System32\Drivers\AppId.sys)` and `AppIdSvc.dll`, allow administrators to specify executable files, DLLs, and scripts

---

## Influenced the design of Windows

The `Trusted Computer System Evaluation Criteria (TCSEC) - 1981` includes `levels-of-trust` ratings. Core
requirements for any secure operating system `(C2)`:
- `Secure logon facility`: access to the computer only after being been authenticated
- `Discretionary access control`: allows the owner of a resource to determine 
- `Security auditing`: detect, record *security-related events* or *resources actions*
- `Object reuse protection`: prevents users from seeing data that another user has deleted `B-level` security:
- `Trusted path functionality`: prevents Trojan from intercept user data
- `Trusted facility management`: separate account roles for administrative functions

---

## Virtualization-based security

- Common to refer to the `kernel` as *trusted*
- `Virtualization-based security (VBS)`
- `Virtual trust level (VTL)`
- `normal user-mode` and `kernel` code runs in `VTL0`; `VTL1` requires Secure Boot
- `Isolated User Mode`: cannot gain access to anything stored in `VTL1`

### Credential Guard

Some or all of these components in the memory of `Lsass`
- `LSA (Lsass.exe)` and `Isolated LSA (LsaIso.exe)`
- `Password`: primary credential 
- `NT one-way function (NT OWF)`: MD4 hash used by legacy components, using the `NT LAN Manager (NTLM)` protocol
- `Ticket-granting ticket (TGT)`: using `Kerberos`, default on `Windows Active Directory–based` domains
- `Protecting the password`:  encrypted, stored to provide `single sign-on (SSO)`. Biometric credentials  against hardware tracking application.  Similar method `Two-factor authentication (TFA)` 
- `Protecting the NTOWF/TGT key`: protecting `Lsass` - config DWORD value `RunAsPPL` in `HKLM\System\CurrentControlSet\Consol\Lsa` registry key to 1. Moreover, use `Lsaiso.exe`(`VTL 1`) alter `Lsass`is more secure
- `Secure communication`: `Advanced Local Procedure Call (ALPC)` - `Secure Kernel` supported by `NtAlpc` calls to the `Normal Kernel`. `Isolated User Mode` support for the `RPC runtime library (Rpcrt4.dll)` over the ALPC protocol allowing VTL 0 and 1 to communicate using local `RPC`
- `UEFI lock`: prevent from disabling `Credential Guard`
- `Authentication policies` and armored `Kerberos`: `VTL 1` Secure Kernel using `TPM`, two security guarantees are provided:
- The user is authenticating from a known machine
-  The `NTLM` response/user ticket is coming from isolated `LSA` and has not been manually
generated from `Lsass`
- `Future improvements`: combined with `BioIso.exe` and `FsIso.exe`
- Navigate to `Computer Configuration > Administrative Templates > System > Device Guard > Turn on Virtualization Based Security. In the "Credential Guard Configuration"` section in `gpedit.msc` (Local Group Policy Editor), set the dropdown value to "Enabled/Disabled":

### Device Guard

- `Hypervisor-Based Code Integrity (HVCI)` and `Kernel-Mode Code Integrity (KMCI)`
- Fully configurable, protect machine from different kinds of `software-based` and `hardware-based` attacks
- Moving all `code-signing` enforcement into `VTL1` (in a library called `SKCI.DLL`, or
`Secure Kernel Code Integrity`)
- Active in BIOS or  edit at regitry `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\DeviceGuard` or powershell: `(Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard).SecurityServicesRunning`

---

## Protecting objects 

Using with WinObj Sysinternals tool. The Windows integrity mechanism is used by `User Account Control (UAC)` elevations
- Access checks with `SRM` (object manager) and three inputs: the security identity of a thread, the access that the thread wants to an object, and the security settings of the object
- Security identifiers: using `Windows uses security identifiers (SIDs)`  used to uniquely identify a security principal or security group
- Integrity levels ( or `AppContainer` used by `UWP apps`): `Mandatory Integrity Control (MIC)` allows the SRM to have more detailed information, obtained with the `GetTokenInformation API`
- `Tokens` used by `SRM`  to identify the security context of a process or thread including `session ID`
- Impersonation provides access to resources, lets a server notify the `SRM`
- `Restricted tokens`  is created from a primary or `impersonation token` using the `CreateRestrictedToken` function, impersonate a client at a reduced security level, primarily for safety reasons when running `untrusted code`, also used by `UAC` to create the `filtered admin token`
- `Virtual service accounts` improves the security isolation and access control
- `Security descriptors` and `access control`
- `ACL assignment`
- `Trust ACEs`:   is part of a `token object` that exists for tokens attached to protected or `PPL processes`
- `Determining access`: `mandatory integrity check` and `discretionary access check`
- `User Interface Privilege Isolation`: prevents processes with a lower integrity level (IL) from sending messages to higher IL processes
- `Owner Rights` improves service hardening in the operating system and allow more flexibility for specific usage scenarios
- `A warning regarding the GUI security editors` shows the order of ACEs in the
DACL to know what access a particular user or group will have to an object
- `Dynamic Access Control`: flexible mechanism used to define rules based on custom attributes defined in `Active Directory`

---

## The AuthZ API

- Same security model as the `security reference monitor (SRM)` but in user mode in the `%SystemRoot%\System32\Authz.dll` library, improve subsequent checks.
- Allow applications to perform customizable access checks with better performance and more simplified development than `Low-level Access Control` and cache access checks for improved performance, to query and modify client contexts, and to define business rules that can be used to evaluate access permission dynamically
- `Conditional ACEs`: are a form of `CALLBACK ACEs` with a special format of the application data, allows a conditional expression to be evaluated when an access check

---

## Account rights and privileges
- A `privilege` allows account to perform a particular system-related operation
- An `account right` grants or denies the account to perform a particular type of logon
- A `system administrator` assigns privileges to groups and accounts using tools such as the `Active Directory Users` and `Groups MMC` snap-in for domain accounts or the `Local Security Policy editor (%SystemRoot%\System32\secpol.msc)`
- `Account rights`: the function responsible for
logon is `LsaLogonUser` - takes a parameter
that indicates the type of logon being performed
- `Privileges`: is a privilege constant
- `Super privileges`: full control over a computer, gain unauthorized access to otherwise off-limit resources and to perform unauthorized operations 

---

## Access tokens of processes and threads

![basic process
and thread security structures](https://www.oreilly.com/api/v2/epubs/9780735671294/files/httpatomoreillycomsourcemspimages1568995.png.jpg)

---

## Security auditing

- `Object manager` can generate `audit events` as a result of an access check, and Windows functions
available to user applications can generate them directly
- A process must have the `SeSecurityPrivilege` privilege to manage the security event
log and to view or set an object’s `SACL`
- The `audit policy` configuration (both the basic settings under `Local Policies` and the `Advanced Audit Policy Configuration`) is stored in the registry as a bitmapped value in the `HKEY_LOCAL_MACHINE\SECURITY\Policy\PolAdtEv`
- `Object access auditing` has two types of audit ACEs: access allowed and access denied
- `Global audit policy` is defined for the
system that enables `object-access` auditing for all `file-system objects`, all `registry keys`, or for both. Using `AuditPol` command with the
/resourceSACL option to set or query the global audit policy
- `Advanced Audit Policy settings`: the security audit policy settings under `Security Settings\Advanced Audit Policy Configuration` can help your organization audit compliance with iAppContainers
mportant business-related and security-related rules by tracking precisely defined activities

---

## AppContainers

- Security sandbox created primarily to host `UWP processes`
- `Overview of UWP apps`: `Universal Windows Platform (UWP)` use `WinRT APIs` to provide powerful UI and advanced asynchronous features
    - Produce normal executables, just like desktop apps. `Wwahost.exe` `(%SystemRoot%\System32\wwahost.exe)` is used to host `HTML/JavaScript-based` UWP apps, as those produce a DLL, not an executable
- `The AppContainer`: characteristics of packaged processes running inside
    - The process token integrity level is set to Low
    - UWP processes are always created inside a job
    - The token for UWP processes has an AppContainer SID
    - The token may contain a set of capabilities
    - Containing a set of capabilities, each represented with a SID
    - Containing privileges: `SeChangeNotifyPrivilege, SeIncrease-WorkingSetPrivilege, SeShutdownPrivilege, SeTimeZonePrivilege`, and `SeUndockPrivilege`
    - The token will contain up to four security attributes
        - WIN://PKG
        - WIN://SYSAPPID
        - WIN://PKGHOSTID
        - WIN://BGKD
- `AppContainer security environment`
- `AppContainer capabilities`
- `AppContainer and object namespace`
- `AppContainer handles`
- `Brokers`: Windows provides helper processes, called brokers, managed by the system broker process, `RuntimeBroker.exe`

---

## Logon

- `Winlogon` is a trusted process responsible for managing security-related user interactions
- Default providers are `authui.dll, SmartcardCredentialProvider.dll, and FaceCredentialProvider.dll`, listed in
`HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Authentication\Credential Providers`
- `Winlogon initialization`: when Winlogon initializes, it registers the `CTRL+ALT+DEL` `secure attention sequence (SAS)` -  a special key or key combination to be pressed on a computer keyboard before a login screen, with the system, and then creates three desktops within the `WinSta0` window station
- `User logon steps`: 
    - Begin with pressing the `SAS (Ctrl+Alt+Del)`, call `LsaLogonUser`, disable or enable at registry `HKLM\SYSTEM\CurrentControlSet\Control\Lsa` 
    - 2 standard authentication packages: `MSV1_0` and `Kerberos`
- `Assured authentication`
- `Windows Biometric Framework`
    - The Windows Biometric Service `(%SystemRoot%\System32\Wbiosrvc.dll)`
    - The Windows Biometric Driver Interface (WBDI) 
    - The Windows Biometric API
    - The fingerprint biometric service provider
    - The functional device driver for the actual biometric scanner device    
- `Windows Hello`
    - Fingerprint
    - Face
    - Iris

---

## User Account Control and virtualization

Run most applications with standard user rights, even though the user is logged in to an account with administrative rights

- File system and registry virtualization
    - Enables these legacy applications to run in standard user accounts through the help of file
system and registry namespace virtualization
    - File virtualization: `UAC File Virtualization filter driver (%SystemRoot%\System32\Drivers\Luafv.sys)`
    - Registry virtualization
        - There are numerous exceptions, such as the following:
            - HKLM\Software\Microsoft\Windows
            - HKLM\Software\Microsoft\Windows NT
            - HKLM\Software\Classes
- Elevation
    - Running with administrative rights
        - `Admin Approval Mode (AAM)` mechanism creates one with standard user rights and another with administrative rights
        -  Over-the-shoulder (OTS) elevation because it requires the
entry of credentials for an account that’s a member of the Administrators group
    - Requesting administrative rights: `Run as Administrator` context menu command and shortcut option to call the `ShellExecute API` with the runas verb
    - Auto-elevation: require executable file is Windows executable and one of several directories considered secure: `%SystemRoot%\System32` and most of its subdirectories, `%Systemroot%\Ehome`, and a small number of directories under `%ProgramFiles%`
    - Controlling UAC behavior: The UAC setting is stored in four values in the registry under `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`

---

## Exploit mitigations

- Process-mitigation policies: `MemGC` to avoid many classes of memory-corruption attacks
- Control Flow Integrity: validate that the target of an indirect
JMP or CALLinstruction is the beginning of a real function, or that a RET instruction is pointing to an expected location, or that a RET instruction is issued after the function was entered through its beginning
- Control Flow Guard: highly-optimized platform security feature that was created to combat memory corruption vulnerabilities. It is an exploit-mitigation mechanism, exists as `Kernel CFG (KCFG)` on the `Creators Update`, terminates process if the target is not at the start of a known function
    - The CFG bitmap: 
    - Strengthening CFG protection
    - Loader interaction with CFG
    - Kernel CFG
- Security assertions
- Compiler and OS support
- Fast fail/security assertion codes

---

## Application Identification

- AppID providing a single set of APIs and data structures to identify and distinguish the application from other applications, provide a consistent and recognizable way to refer to the application.
- AppID contains:
    - Fields
    - File hash
    - The partial or complete path to the file
- AppID stored in process access token

---

## AppLocker

- Allows an administrator to lock down a system to prevent unauthorized programs from being run.
- AppLocker's auditing mode allows an administrator to create an AppLocker policy and examine the results
- Two types of rule in AppLocker:
    - Allow the specified files to run, denying everything else.
    - Deny the specified files from being run, allowing everything else. Deny rules take precedence over allow rules.
- AppLocker can created rule with the following criteria:
    - Fields within a code-signing certificate embedded within the file, allowing for different combinations of publisher name, product name, file name, and version
    - Directory path, allowing only files within a particular directory tree to run
    - File hash
- AppLocker stored in registry
    - HKLM\Software\Policies\Microsoft\Windows\SrpV2
    - HKLM\SYSTEM\CurrentControlSet\Control\Srp\Gp\Exe
    - HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Group Policy
Objects\\{GUID}Machine\Software\Policies\Microsoft\Windows\SrpV2

---

## Software Restriction Policies

- Help administrators to control what images and scripts execute on their system.
- Node of Local Security editor
- Several global policy:
    + Enforcement
    + Designated File types
    + Trusted Publishers

---

## Kernel Patch Protection

- SRP is a user-mode mechanism help administrators to control what images and scripts execute on their systems
    - node of Local Security Policy Editor, with role as the management interface for a machine's code execution policies
- Some global policy setting appear from SRP node:
    - Enforcement
    - Designated File Types
    - Trusted Publishers
- Modifying the behavior Windows through device drivers can cause stability, security issues and may also be done with malicious.
- Some mechanism for reacting to unwanted operations:
    - crashing the machine with an identify kernel-mode crash dump.
    - obfuscation to make it harder for unwanted behavior to disable the detection mechanism
    - randomization and non-documentation of the detection/prevention mechanism to make it harder for attackers to exploit it
    - automatic submission of kernel mode crash dumps to Microsoft allows the company to receive telemetry on unwanted code and track malicious drivers.

---

## PatchGuard

- Also called Kernel Patch Protection (KPP), use to telemetry and exploit-crippling patch detection. 
- KPP has a variety of checks that it makes on protected systems, and documenting them all would both
be impractical
- Some supported techniques for third-party developers who use KPP deters:
    - File system (mini) filters
    - Registry filter notifications
    - Process notifications
    - Object manager filtering
    - NDIS Lightweight Fileters (LWF) and Windows Filtering Platform (WFP)
    - Event Tracing for Windows (ETW)

---

## HyperGuard

- Few attributes different from Patch Guard
    - Symbol files and function names that implement HyperGuard are available for anyone to see, and the code is no obfuscated
    - By operating deterministically, HyperGuard can cra    sh the system immediately when unwanted behavior is detected
    - Due to the preceding property, it can detect a wider variety of attacks so the malicious code does not have the chance to restore a value back to its correct value (it restore to correct value for hide its tracks)
- HyperGuard is also used to extend PatchGuard’s capabilities in certain ways, and to strengthen its
ability to run undetected by attackers trying to disable it
- No way to disable PatchGuard or HyperGuard once they are enabled.

---

## Conclusion

Windows provides an extensive array of security functions that meet the key requirements of both
government agencies and commercial installations. In this chapter, we’ve taken a brief tour of the internal
components that are the basis of these security features