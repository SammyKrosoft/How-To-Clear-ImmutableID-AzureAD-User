# How-To-Clear-ImmutableID-AzureAD-User
Quick draft to show the process of clearing ImmutableID attribute of Azure AD users to re-sync these in a new OnPrem Hybrid environment for example

```powershell
# CLEARING the ImmutableID attribute from one or more users

## Purpose : re-sync cloud users with AADConnect OnPrem

### Use Case : hybrid is destroyed, you rebuild OnPrem hybrid, and you want to sync back Exchange Online mailbox-enabled users as Mail EnabledUsers OnPrem.

#### Example for one user

##### Prerequisites:
<#
1. We must stop OnPrem AD Sync (put AADConnect in Staging mode)
https://learn.microsoft.com/en-us/azure/active-directory/hybrid/connect/how-to-connect-sync-staging-server

2. then we must stop AD Sync from Azure


#>

#Once OnPrem AD Sync is put on Staging, check Azure DirSync status (showing 2 ways below)

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

# Connect to Azure
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

#NOTE: the user will have a new ImmutableID based on the 

```
