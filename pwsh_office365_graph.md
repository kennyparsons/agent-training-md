# AI Agent Guide: PowerShell, Exchange Online & Microsoft Graph

This guide outlines best practices for an AI agent interacting with Microsoft 365 services via PowerShell Core (`pwsh`) in a stateless CLI environment.

## 1. Core Concepts

### Stateless Execution
*   **Constraint:** Every `run_shell_command` invocation spawns a fresh shell process.
*   **Consequence:** Authentication tokens, variables, and session states **do not persist** between tool calls.
*   **Solution:** You must **chain** the connection command and the action command in a single execution string.
    *   ❌ **Bad:** 
        1. `pwsh -c "Connect-ExchangeOnline ..."` 
        2. `pwsh -c "Get-EXOMailbox ..."` (Fails: "You must call Connect-...")
    *   ✅ **Good:** 
        `pwsh -c "Connect-ExchangeOnline ...; Get-EXOMailbox ..."`

### Module Management
Always check for module availability before use. Install in the `CurrentUser` scope to avoid permission issues.
```powershell
# Check
Get-Module -ListAvailable ExchangeOnlineManagement
Get-Module -ListAvailable Microsoft.Graph.Authentication

# Install
Install-Module ExchangeOnlineManagement -Scope CurrentUser -Force -AllowClobber
Install-Module Microsoft.Graph.Authentication -Scope CurrentUser -Force -AllowClobber
```

## 2. Exchange Online (EXO)

**Primary Module:** `ExchangeOnlineManagement`
**Use Case:** Mailboxes, permissions, forwarding, shared mailbox conversion.

### Connection Pattern
```powershell
Connect-ExchangeOnline -UserPrincipalName admin@domain.com
```

### Common Commands

**Get Mailbox Status:**
```powershell
Get-EXOMailbox -Identity target@domain.com | Select-Object DisplayName, RecipientTypeDetails, PrimarySmtpAddress
```

**Convert to Shared Mailbox:**
```powershell
Set-Mailbox -Identity target@domain.com -Type Shared
```

**Add Delegate Permissions:**
```powershell
Add-MailboxPermission -Identity target@domain.com -User delegate@domain.com -AccessRights FullAccess -InheritanceType All
```

**Check Storage/Usage:**
```powershell
Get-EXOMailboxStatistics -Identity target@domain.com | Select-Object TotalItemSize, ItemCount
```

## 3. Microsoft Graph (MgGraph)

**Primary Modules:** `Microsoft.Graph.Authentication`, `Microsoft.Graph.Users`
**Use Case:** Licensing, user account management, Azure AD attributes.

### Connection Pattern
Requires defining scopes explicitly.
```powershell
Connect-MgGraph -Scopes User.ReadWrite.All, Directory.ReadWrite.All
```
*Note: If a cached token exists on the system, this may work without interactive login, but always anticipate needing a device code flow if running locally/interactively.*

### Common Commands

**Check Assigned Licenses:**
⚠️ **Critical:** `Get-MgUser` does NOT return the `AssignedLicenses` property by default. You must request it explicitly using `-Property`.

```powershell
$user = Get-MgUser -UserId target@domain.com -Property DisplayName, AssignedLicenses
Write-Host "License Count:" $user.AssignedLicenses.Count
```

**Remove a License:**
You need the `SkuId` of the license to remove it.
```powershell
# 1. Get SkuId (Part of a larger script logic)
$user = Get-MgUser -UserId target@domain.com -Property AssignedLicenses
$skuId = $user.AssignedLicenses[0].SkuId

# 2. Remove License (Requires Set-MgUserLicense)
Set-MgUserLicense -UserId target@domain.com -RemoveLicenses @($skuId) -AddLicenses @{}
```

## 4. Troubleshooting Tips

*   **"Authentication needed":** You forgot to chain `Connect-...` in the same command string.
*   **Empty Output:** 
    *   The command might have succeeded silently (standard for `Set-` commands).
    *   You might be selecting a property that wasn't retrieved (common with Graph objects). Verify with `Select-Object *` or `-Property`.
*   **Object Parsing:** PowerShell objects returned to the CLI text output can be messy. Use `| Select-Object Prop1, Prop2` or `| Format-List` to make the output readable for the LLM.
