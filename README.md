# How-To-Clear-ImmutableID-AzureAD-User

This is a quick PowerShell draft to show the process of clearing ImmutableID attribute of Azure AD users to re-sync these in a new OnPrem Hybrid environment that you might have rebuilt after a disaster...

## Quick summary how it works

- the user object will have a new ImmutableID value based on its ObjectGUID, here's a quick summ-up how it works out: 
  
  - the user object in OnPRem AD is soft-matched with the object in Azure AD (soft-march = UPN Onprem matches UPN on Azure AD, if not, e-mail address onprem marches e-mail address on Azure AD). 
  - When there is a match between OnPrem AD user and AAD user, the OnPrem user's ObjectGUID is converted to a base64 value and stamped into the mS-DS-ConsistencyGuid attribute
  - That ms-DS-ConsistencyGuid value is copied on the "ImmutableID" attribute of the user on Azure AD.
  - Then on subsequent synchros, ImmutableID will match with the ms-DS-ConsistencyGuid, this time it is called a "Hard match" (as opposed to Soft-match above)

> NOTE: the **ImmutableID** attribute is also called the **sourceAnchor**.

All Details are in this article here: [Hybrid identity getting objects synced](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/hybrid-identity-getting-users-aligned/ba-p/2274690#:~:text=The%20immutable%20ID%20attribute%20in%20AAD%20is%20ObjectId%3B,the%20immutable%20ID%20is%20what%20represents%20object%20uniqueness.)

## PowerShell sequence to check and set the ImmutableID of a user if you need to

```powershell
# CLEARING the ImmutableID attribute from one or more users

## Purpose : re-sync cloud users with AADConnect OnPrem

### Use Case : hybrid is destroyed, you rebuild OnPrem hybrid, and you want to sync back Exchange Online mailbox-enabled users as Mail EnabledUsers OnPrem.

#### Example for one user

##### Prerequisites:
<#
1. We must stop OnPrem AD Sync [put AADConnect in Staging mode](https://learn.microsoft.com/en-us/azure/active-directory/hybrid/connect/how-to-connect-sync-staging-server)

2. then we must stop AD Sync from Azure


#>

#Once OnPrem AD Sync is put on Staging, check Azure DirSync status (showing 2 ways below)

#First you must Connect to Azure
Connect-MsolService

(Get-MSOLCompanyInformation).DirectorySynchronizationEnabled
Get-MSOLCompanyInformation | select DirectorySynchronizationStatus

#then stop/disable AD Sync from Azure
Set-MsolDirSyncEnabled -EnableDirSync $false

#Then recheck AD Sync status on Azure
(Get-MSOLCompanyInformation).DirectorySynchronizationEnabled
Get-MSOLCompanyInformation | select DirectorySynchronizationStatus

#NOTE: once you set Dirsync to $False and back to $True on Azure AD, you must wait between 12 and 72 hours before being able to change it again.

#Store user UPN in a variable - you can use an array of users or get all users to remove all ImmutableIDs
$AZUserUPN = "Anne.Orak@CanadaDrey.ca"

# if not already done, Connect to Azure
Connect-MsolService

# Check the user ImmutableID(you can export it if you change your mind later and want to put it back)
Get-MSOlUser -UserPrincipalName $AZUserUPN | ft DisplayName, ImmutableID

#NOTE: to export all immurableIDs:

Get-MsolUser -All | Select-Object UserprincipalName,ImmutableID,WhenCreated,LastDirSyncTime| Export-Csv c:\temp\UsersImmutableIDs.csv -NoTypeInformation

# Set ImmutableID attribute to $null
Set-MsolUser -UserPrincipalName $AZUserUPN -ImmutableId $null

#Check the user ImmutableID is cleared
Get-MSOLUser -UserPrincipalName $AZUserUPN | ft DisplayName, ImmutableID


# AT THIS POINT you can re-enable Azure AD side DirSync, and Onprem side Dirsync, force a sync and see the MEU corresponding to the user we cleared ImmutableID that will appear.

## To enable Dirsync at the Azure AD level:

### Check
(Get-MSOLCompanyInformation).DirectorySynchronizationEnabled
Get-MSOLCompanyInformation | select DirectorySynchronizationStatus

### Enable
Set-MsolDirSyncEnabled -EnableDirSync $true

### Recheck
(Get-MSOLCompanyInformation).DirectorySynchronizationEnabled
Get-MSOLCompanyInformation | select DirectorySynchronizationStatus



# Then run the following Command(s) from your AADCONNECT server:
  
## For a Delta Sync (most common, and used for most situations):

Start-ADSyncSyncCycle -PolicyType Delta

## For a Full Sync (only necessary in some situations):

Start-ADSyncSyncCycle -PolicyType Initial

```
