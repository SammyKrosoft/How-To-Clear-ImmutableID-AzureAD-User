# Understanding and managing AADConnect user object match (sourceAnchor/ImmutableID)

In an Hybrid environment setup with AADConnect to synchronize OnPrem AD (AD DS) and Azure AD, objects are linked by an attribute. This attribute is called "sourceAnchor", or "ImmutableID", and it's based on the ObjectGUID. First Objects are matched using the primary mail (SMTP) address of the object, then the sourceAnchor (on Metaverse) / ImmurableID (on AzureAD) **and** the *ms-DS-ConsistencyGuid* (on AD DS Onprem) attributes are populated with the objectGUID value (converted in Base64) ==> that's how the OnPrem and the Azure objects are "hard-linked" together

This is a quick PowerShell draft to show the process of clearing ImmutableID attribute of Azure AD users to re-sync these in a new OnPrem Hybrid environment that you might have rebuilt after a disaster...

## Quick summary how it works

- the user object will have a new ImmutableID value based on its ObjectGUID, here's a quick summ-up how it works out: 
  
  - the user object in OnPRem AD is soft-matched with the object in Azure AD.

**NOTE:**
Soft match is the process used to link an object being synced from on-premises for the first time with one that already exists in the cloud (Azure AD). Soft match will first be attempted using the standard logic, based on the primary SMTP address. If a match isn't found based on primary SMTP, then a match will be attempted based on UserPrincipalName (UPN). UPN soft-match is not enabled by default, only SMTP soft-match happens. To enable UPN soft match, see [this article](https://learn.microsoft.com/en-us/powershell/module/msonline/set-msoldirsyncfeature?view=azureadps-1.0) to enable it - also note that once enabled it cannot be disabled.


  - When there is a match between OnPrem AD user and AAD user, the OnPrem user's ObjectGUID is converted to a base64 value and stamped into the mS-DS-ConsistencyGuid attribute
  - That ms-DS-ConsistencyGuid value is copied on the "ImmutableID" attribute of the user on Azure AD.
  - Then on subsequent synchros, ImmutableID will match with the ms-DS-ConsistencyGuid, this time it is called a "Hard match" (as opposed to Soft-match above)

## NOTE: the **ImmutableID** attribute is also called the **sourceAnchor** attribute.

This attribute is actually **mS-DS-ConsistencyGuid** OnPrem, **sourceAnchor** in the metaverse of AAD Connect, and **ImmutableID** on Azure AD. But as the below article states, Microsoft articles and consultants talk about ImmutableID or SourceAnchor interchangeably to name the same thing: the attribute that ties an OnPrem user object with an Azure AD user object.

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

```

Here's what you should see when disabling DirSync on the Azure AD side:

<img width="382" alt="image" src="https://github.com/SammyKrosoft/How-To-Clear-ImmutableID-AzureAD-User/assets/33433229/f89b17ad-ed1a-43d2-907a-6bf75bd2b9d7">

When it shows "Disabled" (run ```Get-MSOLCompanyInformation | select DirectorySynchronizationStatus``` to monitor the DirSync state) like the below:

<img width="365" alt="image" src="https://github.com/SammyKrosoft/How-To-Clear-ImmutableID-AzureAD-User/assets/33433229/f8f268e2-8aef-4ce9-81d6-7e1508126f58">


move on to the below steps to clear out ImmutableID from one (or more) user(s):

```powershell
#Store user UPN in a variable - you can use an array of users or get all users to remove all ImmutableIDs
$AZUserUPN = "Anne.Orak@CanadaDrey.ca"

# if not already done, Connect to Azure
Connect-MsolService

# Check the user ImmutableID(you can export it if you change your mind later and want to put it back)
Get-MSOlUser -UserPrincipalName $AZUserUPN | ft DisplayName, ImmutableID

#NOTE: to export all immurableIDs:
$FullExportCSVPath = "c:\temp\UsersImmutableIDs.csv"
Get-MsolUser -All | Select-Object UserprincipalName,ImmutableID,WhenCreated,LastDirSyncTime| Export-Csv $FullExportCSVPath -NoTypeInformation


# Set ImmutableID attribute to $null
Set-MsolUser -UserPrincipalName $AZUserUPN -ImmutableId $null
```

**NOTE:** In my tests, setting ```-ImmutableID``` to ```$null``` did not change the ImmurableID value... I had to replace the ```$null``` value into simple quotes, like this:

```
Set-MsolUser -UserPrincipalName $AZUserUPN -ImmutableId ''
```

And I was able to remove Immutble ID like this.

**NOTE 2:** If you want to clear the ```ImmutableID``` attribute for all users, you can call ```Get-MSOLUser -All | Set-MSOLUser -ImmutableID '' ``` but please **BE CAREFUL** about that if you still are configured in a hybrid infrastructure as this will "disconnect" all your Azure AD objects from your OnPrem objects.

Next back to our process in PowerShell, let's check if the users's ImmutableID is cleared...

```powershell
#Check the user's ImmutableID is cleared
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
