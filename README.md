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
