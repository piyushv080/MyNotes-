# PowerShell and Shell Scripting Guide (Automation Engineer)
 
Scope: PowerShell fundamentals, objects and the pipeline, error handling, remoting with **WinRM** (and SSH remoting), calling **REST APIs** (ServiceNow, AAP, AWS), working with JSON/CSV/files, Windows administration tasks, Bash/zsh essentials, and how all of this connects to Ansible (Windows hosts over WinRM, `win_*` modules).
 
How to use: PowerShell 7 (`pwsh`) runs on macOS/Linux/Windows, so you can practise most of section 1-4 and 6 locally. WinRM sections (5, 8) need a Windows machine or VM as a target. Examples are not run against your environment; test on a lab VM first.
 
> Version note: Windows PowerShell 5.1 (`powershell.exe`, ships with Windows) and PowerShell 7+ (`pwsh`) differ in a few cmdlet parameters (e.g. `Invoke-RestMethod -SkipCertificateCheck` exists only in 7+). Where it matters, it is called out.
 
---
 
## 0. Mental model
 
- **Bash** passes **text** between commands; **PowerShell** passes **objects** (with properties and methods). You filter and select by property, not by `awk` column. - Cmdlets are named **Verb-Noun** (`Get-Service`, `Restart-Service`). `Get-Command`, `Get-Help`, `Get-Member` are how you discover anything. - In your role: PowerShell is the glue for **Windows servers and Microsoft ecosystem** (AD, IIS, services, registry, patching); Bash is the glue for **Linux, CI runners, containers and quick pipelines**. Ansible wraps both: Linux over SSH, Windows over **WinRM** (or SSH).
 
---
 
## 1. PowerShell fundamentals
 
### Discovery (use these constantly)
 
```powershell Get-Command *service*                 # find cmdlets Get-Help Get-Service -Examples        # examples; Update-Help to refresh Get-Service | Get-Member              # what properties/methods does the object have? $PSVersionTable                       # version info Get-Alias ls                          # aliases (ls = Get-ChildItem); avoid aliases in scripts ```
 
### Variables, types, strings
 
```powershell $name   = "web01" $count  = 3                           # [int] [string[]]$servers = "web01", "web02" $hash   = @{ env = "dev"; port = 8080 }       # hashtable $arr    = @(1, 2, 3)                           # array $obj    = [pscustomobject]@{ Name = "web01"; Env = "dev" }
 
"Server: $name"                       # double quotes expand variables 'Server: $name'                       # single quotes are literal "Count: $($servers.Count)"            # subexpression for properties/expressions @" multi-line here-string with $name "@
 
$env:COMPUTERNAME                     # environment variable $env:PATH -split [IO.Path]::PathSeparator ```
 
### Operators (not `==`, `>` !)
 
| Task | Operator | Example | |---|---|---| | Equal / not equal | `-eq` `-ne` | `$a -eq 5` | | Greater / less | `-gt` `-ge` `-lt` `-le` | `$cpu -gt 90` | | Wildcard match | `-like` `-notlike` | `$n -like "web*"` | | Regex match | `-match` `-notmatch` | `$s -match '^\d+$'` (captures in `$Matches`) | | Contains (collection) | `-contains` `-in` | `$list -contains "a"`, `"a" -in $list` | | Logical | `-and` `-or` `-not` `!` | | | Replace / split / join | `-replace` `-split` `-join` | `"a,b" -split ","` |
 
String comparisons are **case-insensitive** by default; use `-ceq`, `-clike` for case-sensitive.
 
### Control flow
 
```powershell if ($cpu -gt 90) { "high" } elseif ($cpu -gt 70) { "medium" } else { "ok" }
 
foreach ($s in $servers) { Write-Host $s } $servers | ForEach-Object { $_.ToUpper() }        # $_ = current item for ($i = 0; $i -lt 3; $i++) { $i } while ($true) { break }
 
switch -Regex ($line) {     '^ERROR' { "err" }     '^WARN'  { "warn" }     default  { "info" } } ```
 
### Functions with parameters (advanced function pattern)
 
```powershell function Restart-AppService {     [CmdletBinding(SupportsShouldProcess)]          # gives -WhatIf and -Confirm for free     param(         [Parameter(Mandatory, ValueFromPipeline)]         [ValidateNotNullOrEmpty()]         [string[]]$ComputerName,
 
        [ValidateSet('nginx', 'w3svc', 'spooler')]         [string]$ServiceName = 'w3svc',
 
        [ValidateRange(1, 10)]         [int]$Retries = 3     )     process {         foreach ($c in $ComputerName) {             if ($PSCmdlet.ShouldProcess($c, "Restart $ServiceName")) {                 Invoke-Command -ComputerName $c -ScriptBlock {                     param($svc) Restart-Service -Name $svc -ErrorAction Stop                 } -ArgumentList $ServiceName             }         }     } }
 
"web01","web02" | Restart-AppService -WhatIf ```
 
### Modules and scripts
 
```powershell Get-Module -ListAvailable Install-Module Pester -Scope CurrentUser         # PSGallery Import-Module ActiveDirectory Set-StrictMode -Version Latest                    # catch typos/uninitialised variables $ErrorActionPreference = 'Stop'                   # make errors terminating (script-wide) ```
 
Script template:
 
```powershell #Requires -Version 7.0 [CmdletBinding()] param(     [Parameter(Mandatory)][string]$Environment ) Set-StrictMode -Version Latest $ErrorActionPreference = 'Stop' try {     # main logic     exit 0 } catch {     Write-Error $_     exit 1 } ```
 
---
 
## 2. Objects and the pipeline
 
```powershell Get-Service |     Where-Object { $_.Status -eq 'Running' -and $_.Name -like 'w3*' } |     Select-Object Name, Status, StartType |     Sort-Object Name
 
Get-Process | Sort-Object CPU -Descending | Select-Object -First 5 Name, CPU, WS Get-ChildItem C:\Logs -Recurse -Filter *.log | Where-Object Length -gt 10MB Get-Content .\app.log -Tail 100 -Wait                        # like tail -f Select-String -Path .\*.log -Pattern 'ERROR' | Group-Object Filename | Sort-Object Count -Desc
 
# measure / group Get-ChildItem | Measure-Object Length -Sum -Average $records | Group-Object Env | Select-Object Name, Count
 
# calculated properties Get-Process | Select-Object Name, @{n='MemMB'; e={[math]::Round($_.WS/1MB,1)}}
 
# formatting is for DISPLAY only; do not pipe Format-* into other cmdlets Get-Service | Format-Table Name, Status -AutoSize ```
 
Filter **left** (`-Filter`, `Where-Object` early) for performance. Use `-Filter` over `Where-Object` when a cmdlet supports it (server-side filtering).
 
### Export/convert
 
```powershell $data | ConvertTo-Json -Depth 10 | Set-Content out.json       # -Depth matters (default is 2!) Get-Content out.json -Raw | ConvertFrom-Json $data | Export-Csv out.csv -NoTypeInformation Import-Csv servers.csv | ForEach-Object { $_.Name } Import-Clixml / Export-Clixml                                    # round-trip PowerShell objects (secure strings tied to user/machine) ```
 
---
 
## 3. Error handling
 
PowerShell has **terminating** and **non-terminating** errors. `try/catch` only catches terminating errors, so set `-ErrorAction Stop` (or `$ErrorActionPreference = 'Stop'`).
 
```powershell try {     $svc = Get-Service -Name "doesnotexist" -ErrorAction Stop     Restart-Service -InputObject $svc -ErrorAction Stop } catch [Microsoft.PowerShell.Commands.ServiceCommandException] {     Write-Warning "Service not found: $($_.Exception.Message)" } catch {     Write-Error "Unexpected: $($_.Exception.Message)`n$($_.ScriptStackTrace)"     throw                                         # rethrow so the caller/CI fails } finally {     Write-Verbose "cleanup"                       # always runs }
 
# $? last command success, $LASTEXITCODE for native executables & ping.exe -n 1 nothere.invalid if ($LASTEXITCODE -ne 0) { throw "ping failed with $LASTEXITCODE" }
 
# error variable and stream handling Get-Item C:\nope -ErrorAction SilentlyContinue -ErrorVariable e $Error[0]                                         # most recent error ```
 
Streams: `Write-Output` (success), `Write-Error`, `Write-Warning`, `Write-Verbose`, `Write-Debug`, `Write-Information`. Use `Write-Host` only for human-facing display (it bypasses the pipeline).
 
Retry with backoff:
 
```powershell function Invoke-WithRetry {     param([scriptblock]$Script, [int]$Max = 5, [int]$DelaySec = 2)     for ($i = 1; $i -le $Max; $i++) {         try { return & $Script }         catch {             if ($i -eq $Max) { throw }             Write-Warning "Attempt $i failed: $($_.Exception.Message); retrying..."             Start-Sleep -Seconds ($DelaySec * [math]::Pow(2, $i - 1))         }     } } Invoke-WithRetry { Invoke-RestMethod -Uri $url -TimeoutSec 15 } ```
 
Logging: `Start-Transcript -Path "C:\Logs\run-$(Get-Date -f yyyyMMdd-HHmmss).log"` and `Stop-Transcript` for audit evidence.
 
---
 
## 4. REST APIs with PowerShell
 
`Invoke-RestMethod` (IRM) returns **parsed objects** (JSON → PSCustomObject); `Invoke-WebRequest` (IWR) returns the raw response (status code, headers, content).
 
### Basics
 
```powershell $base = "https://devXXXXX.service-now.com"
 
# GET with query parameters (build safely) $query = [System.Web.HttpUtility]::UrlEncode("active=true^priority=1")      # PS 7: [uri]::EscapeDataString $uri   = "$base/api/now/table/incident?sysparm_query=$query&sysparm_limit=5" $resp  = Invoke-RestMethod -Uri $uri -Headers @{ Authorization = "Bearer $token"; Accept = "application/json" } -Method Get -TimeoutSec 30 $resp.result | Select-Object number, short_description, state
 
# POST JSON $body = @{ short_description = "Deployment failed on web01"; impact = "2"; urgency = "2" } | ConvertTo-Json $new  = Invoke-RestMethod -Uri "$base/api/now/table/incident" -Method Post `         -Headers @{ Authorization = "Bearer $token" } -ContentType "application/json" -Body $body $new.result.sys_id
 
# PATCH Invoke-RestMethod -Uri "$base/api/now/table/incident/$($new.result.sys_id)" -Method Patch `     -Headers @{ Authorization = "Bearer $token" } -ContentType "application/json" `     -Body (@{ state = "2"; work_notes = "Investigating" } | ConvertTo-Json) ```
 
### Authentication variants
 
```powershell # Basic auth (dev only) $cred = Get-Credential Invoke-RestMethod -Uri $uri -Credential $cred -Authentication Basic     # PS 7+ (requires HTTPS unless -AllowUnencryptedAuthentication)
 
# Classic Basic header (5.1 compatible) $pair = "{0}:{1}" -f $user, $pass $hdr  = @{ Authorization = "Basic " + [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes($pair)) }
 
# Bearer token (OAuth) $secure = ConvertTo-SecureString $token -AsPlainText -Force Invoke-RestMethod -Uri $uri -Authentication Bearer -Token $secure       # PS 7+
 
# Windows integrated auth (Kerberos/NTLM) Invoke-RestMethod -Uri $uri -UseDefaultCredentials ```
 
### OAuth token (ServiceNow password grant) and token reuse
 
```powershell function Get-SnowToken {     param([string]$Instance, [string]$ClientId, [string]$ClientSecret, [pscredential]$Credential)     $form = @{         grant_type    = "password"         client_id     = $ClientId         client_secret = $ClientSecret         username      = $Credential.UserName         password      = $Credential.GetNetworkCredential().Password     }     $r = Invoke-RestMethod -Uri "$Instance/oauth_token.do" -Method Post -Body $form `          -ContentType "application/x-www-form-urlencoded" -TimeoutSec 30       # hashtable body => form-encoded     [pscustomobject]@{         AccessToken = $r.access_token         ExpiresAt   = (Get-Date).AddSeconds([int]$r.expires_in - 30)     } } ```
 
### Error handling for REST (status codes and response bodies)
 
```powershell try {     $r = Invoke-RestMethod -Uri $uri -Headers $hdr -Method Get -ErrorAction Stop } catch {     # PowerShell 7+: status code and body are on the exception/ErrorDetails     $status = $_.Exception.Response.StatusCode.value__     $detail = $_.ErrorDetails.Message                      # response body (JSON error from ServiceNow)     switch ($status) {         401 { throw "Auth failed (token expired?)" }         403 { throw "Forbidden: check ACLs/roles. $detail" }         404 { return $null }         429 { Start-Sleep 5; <# retry #> }         default { throw "HTTP $status : $detail" }     } } ```
 
Use `-StatusCodeVariable sc` (PS 7) to capture the status code on success, and `-SkipHttpErrorCheck` if you want to handle non-2xx without exceptions.
 
### Pagination and rate limits
 
```powershell $all = @(); $offset = 0; $limit = 100 do {     $page = (Invoke-RestMethod -Uri "$base/api/now/table/incident?sysparm_limit=$limit&sysparm_offset=$offset" -Headers $hdr).result     $all += $page     $offset += $limit } while ($page.Count -eq $limit) ```
 
### Launching an AAP job template
 
```powershell $aap = "https://aap.example.com" $job = Invoke-RestMethod -Uri "$aap/api/controller/v2/job_templates/42/launch/" -Method Post `     -Headers @{ Authorization = "Bearer $env:AAP_TOKEN" } -ContentType "application/json" `     -Body (@{ extra_vars = @{ server_name = "app01"; env = "dev" } } | ConvertTo-Json -Depth 5)
 
# poll status do {     Start-Sleep 10     $j = Invoke-RestMethod -Uri "$aap/api/controller/v2/jobs/$($job.id)/" -Headers @{ Authorization = "Bearer $env:AAP_TOKEN" } } while ($j.status -in 'pending','waiting','running') if ($j.status -ne 'successful') { throw "AAP job $($job.id) ended: $($j.status)" } ```
 
### AWS from PowerShell
 
```powershell Install-Module AWS.Tools.Common, AWS.Tools.EC2 -Scope CurrentUser Set-AWSCredential -ProfileName dev Get-EC2Instance -Region eu-west-1 -Filter @{Name="tag:Env";Values="dev"} |     ForEach-Object Instances | Select-Object InstanceId, PrivateIpAddress, @{n='Name';e={($_.Tags | ? Key -eq 'Name').Value}} ```
 
### TLS and certificates
 
- PS 7: `-SkipCertificateCheck` for **lab only** (self-signed). Production: trust the internal CA (`-CertificateThumbprint`, OS trust store). - Windows PowerShell 5.1 may default to old TLS: `[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12`. - Corporate proxy: `-Proxy http://proxy:8080 -ProxyUseDefaultCredentials`.
 
### Secrets handling
 
```powershell $cred = Get-Credential                                           # interactive $secure = Read-Host -AsSecureString # SecretManagement module (vault-agnostic) Install-Module Microsoft.PowerShell.SecretManagement, Microsoft.PowerShell.SecretStore -Scope CurrentUser Register-SecretVault -Name Local -ModuleName Microsoft.PowerShell.SecretStore Set-Secret -Name SnowPassword -Secret "..." $pw = Get-Secret -Name SnowPassword -AsPlainText:$false # In CI/AAP: read from environment variables supplied by the credential store $env:SN_PASSWORD ```
 
Never put secrets in the script or in `Write-Host`. `ConvertTo-SecureString -AsPlainText -Force` only wraps a string you already exposed; it is not protection by itself.
 
---
 
## 5. WinRM and PowerShell remoting
 
### What WinRM is
 
**WinRM (Windows Remote Management)** is Microsoft's implementation of WS-Management: a SOAP-over-HTTP(S) protocol and service. **PowerShell Remoting** (PSRP) runs on top of it. Ansible also uses WinRM for Windows targets (`ansible_connection=winrm`).
 
| Item | Value | |---|---| | Service name | `WinRM` | | Ports | **5985** (HTTP, message-level encrypted with Kerberos/NTLM) and **5986** (HTTPS) | | Auth | **Kerberos** (domain, preferred), **NTLM** (workgroup/IP access), **CredSSP** (double-hop, risky), **Basic** (only over HTTPS, local accounts, avoid), **Certificate** | | Config | `winrm get winrm/config`, `Get-WSManInstance`, `WSMan:` drive | | Authorisation | Remote Management Users group / local Administrators; session configurations (JEA) for constrained endpoints |
 
Default: enabled on Windows Server (2012+), **disabled** on Windows clients until `Enable-PSRemoting`.
 
### Enable and configure on the target (run as admin)
 
```powershell # Quick enable (HTTP listener on 5985, firewall rule, service start). Domain/private networks. Enable-PSRemoting -Force
 
# Check Test-WSMan -ComputerName localhost Get-Service WinRM winrm enumerate winrm/config/listener Get-NetFirewallRule -Name "WINRM*" | Select Name, Enabled, Profile ```
 
### HTTPS listener (recommended; required for Basic or non-domain scenarios)
 
```powershell # 1. Certificate (lab: self-signed; production: issued by your CA with the host's FQDN in SAN) $cert = New-SelfSignedCertificate -DnsName $env:COMPUTERNAME, "web01.corp.example.com" `         -CertStoreLocation Cert:\LocalMachine\My -NotAfter (Get-Date).AddYears(2)
 
# 2. HTTPS listener New-Item -Path WSMan:\LocalHost\Listener -Transport HTTPS -Address * `          -CertificateThumbPrint $cert.Thumbprint -Force
 
# 3. Firewall New-NetFirewallRule -DisplayName "WinRM HTTPS" -Direction Inbound -Protocol TCP -LocalPort 5986 -Action Allow
 
# 4. Verify winrm enumerate winrm/config/listener Test-WSMan -ComputerName web01 -UseSSL ```
 
Hardening checklist: prefer **Kerberos over HTTPS**, disable `Basic` and `AllowUnencrypted` (`winrm get winrm/config/service` should show `AllowUnencrypted = false`), restrict source IPs with firewall rules, restrict who is in *Remote Management Users*, avoid **CredSSP** unless needed (it delegates credentials), use **JEA** for least privilege, audit with Script Block Logging/Transcription.
 
### Connect from PowerShell (client side)
 
```powershell # One-off command Invoke-Command -ComputerName web01.corp.example.com -ScriptBlock { Get-Service w3svc }
 
# With credentials, HTTPS and (lab) skip CA/CN checks if using a self-signed cert $cred = Get-Credential $opt  = New-PSSessionOption -SkipCACheck -SkipCNCheck          # lab only Invoke-Command -ComputerName 10.0.1.15 -Credential $cred -UseSSL -SessionOption $opt -ScriptBlock { hostname }
 
# Persistent session (reuse for several commands) $s = New-PSSession -ComputerName web01 -Credential $cred -Authentication Kerberos Invoke-Command -Session $s -ScriptBlock { Get-Process | Sort CPU -Desc | Select -First 3 } Enter-PSSession -Session $s                    # interactive; Exit-PSSession to leave Copy-Item -Path .\app.zip -Destination C:\Temp\ -ToSession $s      # file copy over the session Copy-Item -Path C:\Logs\app.log -Destination .\ -FromSession $s Remove-PSSession $s
 
# Many hosts in parallel Invoke-Command -ComputerName (Get-Content hosts.txt) -ThrottleLimit 32 -ScriptBlock {     [pscustomobject]@{ Host = $env:COMPUTERNAME; Uptime = (Get-Date) - (gcim Win32_OperatingSystem).LastBootUpTime } } -ErrorAction SilentlyContinue | Select-Object PSComputerName, Host, Uptime
 
# Passing local variables into the remote scope $svc = "w3svc" Invoke-Command -ComputerName web01 -ScriptBlock { Restart-Service -Name $using:svc } Invoke-Command -ComputerName web01 -ScriptBlock { param($n) Get-Service $n } -ArgumentList $svc ```
 
Gotchas to know: - **Non-domain / IP address**: Kerberos needs a hostname (SPN). When connecting by IP or from a workgroup machine you need `TrustedHosts` and NTLM, or HTTPS:
 
```powershell Set-Item WSMan:\localhost\Client\TrustedHosts -Value "10.0.1.15" -Concatenate -Force    # client side, admin ```
 
- **Double-hop problem**: from a remote session you cannot reuse your credentials to access a second system (file share). Solutions: Kerberos constrained delegation / resource-based delegation, pass explicit `-Credential` inside the block, or CredSSP (less secure). - **Objects come back deserialised**: remote output has properties but not methods. Do work on the remote side, return data. - `Invoke-Command` errors per host appear as non-terminating unless `-ErrorAction Stop`; use `-ErrorVariable`. - **Execution policy** is not a security boundary; set via `Set-ExecutionPolicy RemoteSigned` and sign scripts in regulated environments. - Time skew > 5 min breaks Kerberos. - Linux/macOS clients connecting to WinRM: PowerShell 7 uses **PSRP over WSMan** with NTLM/Kerberos via `gss-ntlmssp`/`krb5` packages (client-side config required). On those platforms, **SSH remoting** is often easier (below).
 
### PowerShell remoting over SSH (cross-platform)
 
```powershell # Target: install OpenSSH Server, add to sshd_config:  Subsystem powershell pwsh -sshs -NoLogo -NoProfile Invoke-Command -HostName user@web01 -KeyFilePath ~/.ssh/id_ed25519 -ScriptBlock { Get-Service } New-PSSession -HostName user@web01 -SSHTransport ```
 
### Testing connectivity from any OS (troubleshooting ladder)
 
```bash nc -zv web01.corp.example.com 5985        # TCP reachable? (5986 for HTTPS) curl -k https://web01:5986/wsman          # HTTPS listener responds (401 is a good sign) ```
 
```powershell Test-NetConnection web01 -Port 5985 Test-WSMan web01 -Authentication Kerberos klist                                     # Kerberos tickets winrm get winrm/config                    # on the target Get-WinEvent -LogName Microsoft-Windows-WinRM/Operational -MaxEvents 20   # on the target ```
 
| Error | Meaning / fix | |---|---| | `WinRM cannot process the request... Kerberos` / `The target principal name is incorrect` | Use the FQDN (SPN match), or HTTPS+NTLM, or fix SPN/DNS | | `Access is denied` | User not in Administrators/Remote Management Users; UAC remote restrictions for local accounts (`LocalAccountTokenFilterPolicy`); wrong creds | | `The client cannot connect to the destination` | Service not running, firewall, listener missing, wrong port/SSL flag | | `The SSL certificate is signed by an unknown certificate authority` | Trust CA, or lab-only `-SkipCACheck` | | `The WinRM client cannot process the request... TrustedHosts` | Non-domain target; add to TrustedHosts or use HTTPS | | Works interactive but not from a script/CI | Different identity; use explicit credentials and check profile/exec policy |
 
### Windows via Ansible (ties to your core skill)
 
Inventory/group vars:
 
```yaml # group_vars/windows.yml ansible_connection: winrm ansible_port: 5986 ansible_winrm_scheme: https ansible_winrm_transport: kerberos           # or ntlm / credssp / basic ansible_user: svc-ansible@CORP.EXAMPLE.COM ansible_password: "{{ vault_win_password }}" ansible_winrm_server_cert_validation: validate     # 'ignore' only in labs ansible_winrm_read_timeout_sec: 60 ansible_winrm_operation_timeout_sec: 50 ```
 
Control node prerequisites:
 
```bash pip install pywinrm requests-credssp "pywinrm[kerberos]"      # kerberos needs krb5 libs + /etc/krb5.conf ansible windows -i inventory -m ansible.windows.win_ping ```
 
```yaml - name: Configure IIS   hosts: windows   tasks:     - ansible.windows.win_feature:         name: Web-Server         state: present         include_management_tools: true     - ansible.windows.win_service:         name: W3SVC         state: started         start_mode: auto     - ansible.windows.win_copy:         src: files/site.conf         dest: C:\inetpub\site.conf     - ansible.windows.win_powershell:         script: |           Get-Service W3SVC | Select-Object Name, Status | ConvertTo-Json       register: ps     - ansible.windows.win_reboot:         when: ps.changed ```
 
Ansible-on-Windows points: modules prefixed `win_`/`ansible.windows.*`, `community.windows.*`; `ansible.windows.win_shell`/`win_command` are the non-idempotent escape hatches (use `creates`/`changed_when`); SSH is also supported (`ansible_connection: ssh` + `ansible_shell_type: powershell`); in AAP use **Machine credentials** and an EE containing `pywinrm`; mesh/execution nodes must reach 5985/5986.
 
---
 
## 6. Windows administration snippets (frequently asked)
 
```powershell # Services Get-Service -Name W3SVC | Restart-Service -PassThru Set-Service -Name Spooler -StartupType Disabled
 
# Processes, disk, memory, uptime Get-CimInstance Win32_LogicalDisk -Filter "DriveType=3" |     Select DeviceID, @{n='FreeGB';e={[math]::Round($_.FreeSpace/1GB,1)}}, @{n='Pct';e={[math]::Round(100*$_.FreeSpace/$_.Size,1)}} (Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
 
# Event logs Get-WinEvent -FilterHashtable @{ LogName='System'; Level=2; StartTime=(Get-Date).AddHours(-24) } | Select -First 20 TimeCreated, Id, Message
 
# Files and ACLs Get-Acl C:\App | Format-List New-Item -ItemType Directory -Path C:\App\logs -Force Compress-Archive -Path C:\App\logs\* -DestinationPath C:\Backup\logs.zip Get-FileHash .\setup.exe -Algorithm SHA256
 
# Registry Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' | Select ProductName, CurrentBuild
 
# Network Test-NetConnection db01 -Port 1433 Resolve-DnsName web01.corp.example.com Get-NetIPAddress -AddressFamily IPv4
 
# Scheduled task $action  = New-ScheduledTaskAction -Execute 'pwsh.exe' -Argument '-File C:\Scripts\cleanup.ps1' $trigger = New-ScheduledTaskTrigger -Daily -At 2am Register-ScheduledTask -TaskName 'Cleanup' -Action $action -Trigger $trigger -RunLevel Highest
 
# Updates (module PSWindowsUpdate), AD (RSAT ActiveDirectory module) Get-ADUser -Filter "Enabled -eq 'True'" -Properties LastLogonDate | Where LastLogonDate -lt (Get-Date).AddDays(-90) ```
 
Use **CIM** (`Get-CimInstance`) instead of legacy WMI (`Get-WmiObject`, removed in PS 7).
 
### Idempotent pattern (same principle as Ansible)
 
```powershell if (-not (Get-WindowsFeature Web-Server).Installed) {     Install-WindowsFeature Web-Server -IncludeManagementTools } ```
 
Test with Pester:
 
```powershell Describe "Web server" {     It "has W3SVC running" { (Get-Service W3SVC).Status | Should -Be 'Running' }     It "responds on 80"    { (Test-NetConnection localhost -Port 80).TcpTestSucceeded | Should -BeTrue } } Invoke-Pester -Output Detailed ```
 
Lint: `Install-Module PSScriptAnalyzer; Invoke-ScriptAnalyzer -Path . -Recurse`.
 
---
 
## 7. Bash and shell scripting (Linux/macOS, CI, containers)
 
### Script skeleton
 
```bash #!/usr/bin/env bash set -Eeuo pipefail IFS=$'\n\t'
 
readonly SCRIPT_NAME="${0##*/}" log()  { printf '%s [%s] %s\n' "$(date -u +%FT%TZ)" "$1" "${*:2}" >&2; } die()  { log ERROR "$*"; exit 1; } usage(){ printf 'usage: %s -e <env> [-n]\n' "$SCRIPT_NAME" >&2; exit 2; }
 
cleanup() { [[ -n "${TMP:-}" ]] && rm -rf "$TMP"; } trap cleanup EXIT trap 'log ERROR "failed at line $LINENO"' ERR
 
DRY_RUN=0 while getopts ":e:n" opt; do   case $opt in     e) ENVIRONMENT=$OPTARG ;;     n) DRY_RUN=1 ;;     *) usage ;;   esac done [[ -n "${ENVIRONMENT:-}" ]] || usage command -v jq >/dev/null || die "jq is required" TMP=$(mktemp -d) ```
 
`set -E` makes the `ERR` trap inherit into functions; `-e` exit on error; `-u` unset variables error; `pipefail` pipeline fails if any stage fails.
 
### Variables, quoting, arrays, parameter expansion
 
```bash name="web 01"; echo "$name"                      # ALWAYS quote files=(a.log b.log); echo "${files[@]}"; echo "${#files[@]}" declare -A env_map=([web01]=dev [db01]=prod); echo "${env_map[web01]}"   # bash 4+ (macOS default bash is 3.2; use brew bash) ${var:-default}  ${var:?error if unset}  ${var%.log}  ${var##*/}  ${var^^}  ${#var} $(command)  $((1+2)) [[ $a == b* ]]  [[ -f file ]]  [[ -d dir ]]  [[ -z "$s" ]]  [[ $n -gt 3 ]] ```
 
### Text processing toolbox
 
```bash grep -E 'ERROR|WARN' app.log | awk '{print $1, $NF}' | sort | uniq -c | sort -rn | head awk -F, 'NR>1 && $3>100 {sum+=$3} END{print sum}' data.csv sed -E 's/password=[^ ]+/password=***/g' app.log                 # redact sed -i.bak 's/old/new/g' file                                    # GNU/BSD portable form (keeps .bak) cut -d: -f1,3 /etc/passwd tr '[:lower:]' '[:upper:]' xargs -P4 -n1 ssh                                                # parallel fan-out find /var/log -name '*.log' -mtime +7 -print0 | xargs -0 rm -f   # NUL-safe ```
 
### JSON with jq (essential for APIs)
 
```bash curl -s "$URL" | jq '.result[] | {number, state}' jq -r '.result[] | [.number, .short_description] | @tsv' resp.json jq -r --arg env "$ENVIRONMENT" '.hosts[] | select(.env == $env) | .name' inventory.json jq -n --arg n "$NAME" --argjson c "$COUNT" '{server_name:$n, count:$c}'      # build JSON safely ```
 
### REST with curl (equivalent of section 4)
 
```bash # OAuth token TOKEN=$(curl -fsS -X POST "$INSTANCE/oauth_token.do" \   -d grant_type=password -d client_id="$CLIENT_ID" -d client_secret="$CLIENT_SECRET" \   -d username="$SN_USER" --data-urlencode password="$SN_PASS" | jq -r .access_token)
 
# Create incident, capture status and body separately resp=$(curl -sS -w '\n%{http_code}' -X POST "$INSTANCE/api/now/table/incident" \   -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \   -d "$(jq -n --arg d "Deployment failed on $HOST" '{short_description:$d, impact:"2", urgency:"2"}')") code=${resp##*$'\n'}; body=${resp%$'\n'*} [[ $code == 201 ]] || die "HTTP $code: $body" echo "$body" | jq -r .result.number
 
# Retries built into curl (idempotent GETs) curl -fsS --retry 5 --retry-delay 2 --retry-connrefused --max-time 30 "$URL" ```
 
Flags: `-f` fail on HTTP errors, `-s` silent, `-S` show errors, `-L` follow redirects, `-k` skip TLS verify (lab only), `--cacert` trust a CA, `-w '%{http_code}'` status.
 
### Retry function
 
```bash retry() {   local -r max=$1; shift   local n=1 delay=2   until "$@"; do     (( n >= max )) && { log ERROR "failed after $n attempts: $*"; return 1; }     log WARN "attempt $n failed; sleeping ${delay}s"     sleep "$delay"; (( n++, delay*=2 ))   done } retry 5 curl -fsS "$URL" -o out.json ```
 
### SSH and remote execution
 
```bash ssh -o BatchMode=yes -o StrictHostKeyChecking=yes -i ~/.ssh/id_ed25519 user@host 'systemctl is-active nginx' ssh user@host 'bash -s' < local_script.sh                 # run a local script remotely scp -r ./dist user@host:/opt/app/ ; rsync -az --delete ./dist/ user@host:/opt/app/ ssh -J bastion user@private-host                          # jump host for h in $(<hosts.txt); do ssh -n "$h" uptime & done; wait  # parallel with wait ```
 
SSH hygiene: keys (not passwords), `ssh-agent`, short-lived certificates or SSM Session Manager in AWS, `~/.ssh/config` host aliases, `known_hosts` management.
 
### AWS CLI patterns
 
```bash aws sts get-caller-identity aws ec2 describe-instances --filters "Name=tag:Env,Values=dev" "Name=instance-state-name,Values=running" \   --query 'Reservations[].Instances[].{id:InstanceId,ip:PrivateIpAddress,name:Tags[?Key==`Name`]|[0].Value}' --output table aws ec2 stop-instances --instance-ids i-0123 --dry-run aws ssm send-command --document-name AWS-RunShellScript --targets Key=tag:Env,Values=dev \   --parameters 'commands=["systemctl restart nginx"]' ```
 
### Common pitfalls (shellcheck catches many)
 
- Unquoted variables (word splitting, globbing); use `"$var"` and `"${arr[@]}"`. - `cd dir` without `|| exit`; with `set -e` still check. - Parsing `ls` output; use `find -print0`/globs. - `cat file | while read` loses variables (subshell); use `while read ...; done < file`. - `[ ]` vs `[[ ]]`; `==` in `[ ]` is not POSIX. - Using `echo` with arbitrary strings; prefer `printf`. - Secrets in command-line args are visible in `ps`; use env vars/files with `chmod 600`/stdin. - Portability: macOS ships BSD tools and bash 3.2/zsh; GNU-only flags (`sed -i` without suffix, `date -d`, `readlink -f`) break there.
 
Run `shellcheck script.sh` and `shfmt -d`; test with `bats-core`.
 
---
 
## 8. PowerShell vs Bash cheat sheet
 
| Task | PowerShell | Bash | |---|---|---| | List files | `Get-ChildItem` (`ls`, `dir`) | `ls -la` | | Filter | `Where-Object { $_.Length -gt 1MB }` | `find . -size +1M` / `awk` | | Select columns | `Select-Object Name, Id` | `awk '{print $1,$2}'` / `cut` | | Sort / count | `Sort-Object`, `Group-Object`, `Measure-Object` | `sort | uniq -c`, `wc -l` | | Search text | `Select-String -Pattern` | `grep` | | JSON parse | `ConvertFrom-Json` | `jq` | | HTTP | `Invoke-RestMethod` | `curl` | | Env var | `$env:NAME` | `$NAME` / `export NAME=` | | Exit/error | `try/catch`, `$LASTEXITCODE`, `$ErrorActionPreference='Stop'` | `$?`, `set -e`, `trap` | | Remote | `Invoke-Command` (WinRM/SSH) | `ssh host cmd` | | Parallel | `ForEach-Object -Parallel -ThrottleLimit 8` (PS 7), `Invoke-Command` to many hosts | `xargs -P`, `&` + `wait`, GNU parallel | | Service | `Restart-Service` | `systemctl restart` | | Package | `Install-WindowsFeature`, `winget`, `choco` | `apt`/`dnf`/`brew` | | Test a script | Pester, PSScriptAnalyzer | bats, shellcheck |
 
---
 
## 9. Putting it together: remediation script (Windows service) with ServiceNow logging
 
```powershell #Requires -Version 7.0 [CmdletBinding(SupportsShouldProcess)] param(     [Parameter(Mandatory)][string]$ComputerName,     [string]$ServiceName = 'W3SVC',     [Parameter(Mandatory)][string]$SnowInstance ) Set-StrictMode -Version Latest $ErrorActionPreference = 'Stop'
 
function Add-SnowWorkNote {     param($Instance, $Token, $SysId, $Note)     Invoke-RestMethod -Uri "$Instance/api/now/table/incident/$SysId" -Method Patch `         -Headers @{ Authorization = "Bearer $Token" } -ContentType 'application/json' `         -Body (@{ work_notes = $Note } | ConvertTo-Json) }
 
$token = $env:SN_TOKEN                      # injected by the pipeline/AAP credential $incident = Invoke-RestMethod -Uri "$SnowInstance/api/now/table/incident" -Method Post `     -Headers @{ Authorization = "Bearer $token" } -ContentType 'application/json' `     -Body (@{ short_description = "[AUTO] $ServiceName down on $ComputerName"; impact = '1'; urgency = '1' } | ConvertTo-Json) $sysId = $incident.result.sys_id
 
try {     if ($PSCmdlet.ShouldProcess($ComputerName, "Restart $ServiceName")) {         $state = Invoke-Command -ComputerName $ComputerName -UseSSL -ErrorAction Stop -ScriptBlock {             param($svc)             Restart-Service -Name $svc -ErrorAction Stop             Start-Sleep 5             (Get-Service -Name $svc).Status.ToString()         } -ArgumentList $ServiceName         if ($state -ne 'Running') { throw "Service state after restart: $state" }     }     Add-SnowWorkNote $SnowInstance $token $sysId "Restarted $ServiceName on $ComputerName; status Running."     exit 0 } catch {     Add-SnowWorkNote $SnowInstance $token $sysId "Auto-remediation FAILED: $($_.Exception.Message)"     Write-Error $_     exit 1 } ```
 
The same pattern in Ansible is `block/rescue/always` with `ansible.windows.win_service` + `servicenow.itsm.incident`.
 
---
 
## 10. Interview quick-fire
 
1. **What is WinRM and how does it differ from SSH?** A WS-Management service (HTTP 5985 / HTTPS 5986) that hosts PowerShell Remoting and is used by Ansible for Windows; SSH is the Linux standard and now also supported on Windows. WinRM uses Kerberos/NTLM/CredSSP/Basic/cert auth and has the double-hop problem. 2. **How do you secure WinRM?** HTTPS listener with proper certificates, Kerberos auth, no Basic/unencrypted, firewall source restrictions, least-privilege accounts or JEA, logging and auditing, avoid CredSSP. 3. **Troubleshoot "can't connect via WinRM"?** Port test (5985/5986), service and listener (`winrm enumerate winrm/config/listener`), firewall, auth method (FQDN/SPN for Kerberos, TrustedHosts for NTLM), certificate trust, time sync, account permissions, WinRM operational event log. 4. **What is the double-hop problem?** Credentials delegated to the first remote host cannot be reused to reach a second resource; solve with Kerberos delegation, explicit credentials, or (carefully) CredSSP. 5. **`Invoke-RestMethod` vs `Invoke-WebRequest`?** IRM parses the response into objects (JSON/XML); IWR returns the raw response with status/headers. Both need `-ErrorAction Stop` to be caught; status codes live on the exception response. 6. **Terminating vs non-terminating errors?** Only terminating errors trigger `catch`; set `-ErrorAction Stop` or `$ErrorActionPreference = 'Stop'`. 7. **Why objects instead of text?** Properties and methods can be filtered/selected reliably without parsing columns; remote output is serialised, so methods are lost. 8. **`set -euo pipefail` and its limits?** Hardens scripts, but `-e` has edge cases (conditions, subshells, `||` lists); still check critical commands and use traps and `shellcheck`. 9. **How do you call a REST API robustly in Bash?** `curl -fsS --max-time --retry`, capture HTTP status with `-w`, parse with `jq`, build JSON with `jq -n`, secrets from env, non-zero exit on failure. 10. **How does Ansible manage Windows?** `ansible_connection=winrm` (or SSH), `pywinrm`/Kerberos on the controller, `ansible.windows.*` and `community.windows.*` modules, `win_shell` as a last resort, credentials via vault/AAP machine credential.
 
---
 
## 11. Revision checklist
 
- [ ] Install PowerShell 7; explore with `Get-Command`, `Get-Help`, `Get-Member`; write three one-liners using `Where-Object`/`Select-Object`/`Group-Object` - [ ] Write an advanced function with `[CmdletBinding(SupportsShouldProcess)]`, validation attributes, pipeline input and `-WhatIf` - [ ] try/catch/finally with `-ErrorAction Stop`; retry/backoff helper; transcript logging - [ ] ServiceNow OAuth + create/patch incident with `Invoke-RestMethod`; handle 401/403/404/429; pagination loop - [ ] Launch an AAP job template and poll to completion from PowerShell and from Bash/curl - [ ] Stand up a Windows VM: `Enable-PSRemoting`, create an HTTPS listener, open 5986, verify with `Test-WSMan -UseSSL` - [ ] `Invoke-Command` to 2+ hosts with `-ThrottleLimit`, `$using:`, persistent `PSSession`, file copy with `-ToSession` - [ ] Reproduce and fix three WinRM failures: Kerberos/SPN (use FQDN), TrustedHosts/NTLM, certificate trust - [ ] Ansible: `win_ping` over WinRM (HTTPS, NTLM then Kerberos); a play with `win_feature`, `win_service`, `win_copy` - [ ] Pester tests and PSScriptAnalyzer on your script; add them to the GitHub Actions workflow (windows-latest runner) - [ ] Bash script with `set -Eeuo pipefail`, `getopts`, trap, `jq`, `curl` retries; clean `shellcheck` run - [ ] Convert one script between PowerShell and Bash to practise the cheat sheet
