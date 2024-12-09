

## Automate JIT RDP Access and Approvals to Azure VMs with MS Forms and Azure Logic Apps  
_Date: 6 Dec 2024_

**Scenario**: In this environment, end users often require RDP access to a specific Azure VMs that are publicly available i.e. acting as bastion hosts. Moreover, the access requirement is temporary. 

**Requirement**: For security hardening, only Just-In-Time (JIT) RDP access needs to be provided in order to lock down the access to a specific public IP, for a specific duration, and to ensure the request is approved first.

**Aim**: Currently, users create a ticket for RDP access to a specific Azure VM and provide their pubic IP that needs to be allow-listed. This ticket gets triaged to the Azure admin team who evaluate the request and create the JIT access to the VM from the Azure Portal or via CLI. As you can imagine, the manual process is slow as it it done manually and dependent on an Azure admin's availability.

The goal is to automate this so that end users can request the JIT access that forces them to provide the required information like the public IP they will access from, and an approver (who can be a non-admin user) can allow it, so that the JIT access is provided without need for an admin user to get involved. 

**Background**: 

JIT access to Azure VMs is a feature of MS Defender for Cloud. You can learn more about it from [MS documentation](https://learn.microsoft.com/en-us/azure/defender-for-cloud/just-in-time-access-overview?tabs=defender-for-container-arch-aks).

&nbsp; 

The process to enable JIT is described in this [MS documentation](https://learn.microsoft.com/en-us/azure/defender-for-cloud/just-in-time-access-usage). 
In short, an Azure admin will set up the Azure VM to be protected by JIT and create rules for the JIT access policy. For example, a rule for port 3389 will be set up for RDP access. After that, roles with the correct permission can provide JIT access by using the rule and specifying the parameters like source IP, protocol, request time (duration).

This create a temporary rule on the Network Security Group (NSG) attached to the specified VM allowing traffic for the requested parameters in the JIT request. 

&nbsp; 

Below, I have discussed in details how I have automated the process so that this JIT access is enabled when an end user's JIT request is received and approved.

&nbsp;

## Using VM Applications

   Prerequisites: 

   - Create a **Storage Account** where we store the application installation source files

   Steps:
   
   1. From the Azure Portal, browse to your Storage Account and upload the MSI file to a container as a Block Blob.
   
   2. Generate a **SAS** for the MSI file's blob with "Read" permissions and an appropriate time duration. Take a note of the generated SAS URL.

   3. Open **Azure Compute Gallery** and create a Compute Gallery. The Compute Gallery is used to contain the application definition and versions.

   4. In your Compute Gallery, add a **VM Application Definition**. Enter the app's name, region (use the same region as your VMs) and OS type (Windows).

   5. In your App Definition, add an **Application Version** for example "2.4.08".
      - For the **version number** use the version in the dot format (e.g. 2.4.08). 
      - For the **source application package**, use the SAS URL for the MSI file from the Storage Account. 
      - For the **install script**, use the sample command below. The move command is used to rename the downloaded MSI file as it is downloaded as the application name instead of the MSI file name:

      ```shell
      move .\\7zip .\\7zip_2408.msi & start /wait %windir%\\system32\\msiexec.exe /i 7zip_2408.msi /quiet /norestart /L*V 7zip_2408_install.log
      ```

      - For the **uninstall script** use the sample command below:

      ```shell
      msiexec.exe /x {23170F69-40C1-2702-2408-000001000000} /quiet /norestart /L*V 7zip_2408_uninstall.log
      ```

   6. Once the App Version is created, browse to a running VM in the same region as the Application.  
      - Select _Extensions and Applications_ > _VM Applications_ > _Add Application_ > Select the app (7zip and it's version) and click Save.
   
---

You will see that when the first VM app is added, the _VMAppExtension_ VM Extension is installed. You can log into the VM and check the _C:\Packages\Plugins\Microsoft.CPlat.Core.VMApplicationManagerWindows_ for the VM Application Extension installation. 

Under the same path, you will find the _Downloads_ folder which holds the downloaded MSI files and corresponding log files.

This manual triggering is fine if you wish to deploy apps to a handful of VMs, however for a larger deployment you can use a PowerShell script to assign the VM App to as many VMs as you desire. 

```powershell

#############################################################################################
# Assign a VM app to all VMs in a Sub  
# VM Apps are region locked so need different app in different region
#############################################################################################

$filename = "Assign_App"
$timestamp = Get-Date -Format "yyyyMMddHHmmss"


$outfile = "$pwd\$($filename)-output-$($timestamp).csv"
Add-Content -Path $outfile -Value '"Subscription","VM Name","Comment"'
$errorfile = "$pwd\$($filename)-error-$($timestamp).csv"


# Set Context to Sub where Compute Gallery is
Set-AzContext -Subscription "SUBSCRIPTION_NAME"

# App details
$gallery_name = 'GALLERY_NAME'
$gallery_rg = 'GALLERY_RESOURCE_GROUP'
$app_name = 'APP_DEF_NAME'
$app_version = 'APP_VER_NAME'


# App details have been moved out of FOR loop
$app = Get-AzGalleryApplicationVersion `
        -GalleryApplicationName $app_name `
        -GalleryName $gallery_name `
        -Name $app_version `
        -ResourceGroupName $gallery_rg

$packageid = $app.Id

$app_instance = New-AzVmGalleryApplication -PackageReferenceId $packageid



# Get all subscriptions
$Subscriptions = Get-AzSubscription

ForEach ($Subscription in $Subscriptions) 
{
    if($Subscription.State -eq "Enabled")
    {
              
        # Change to current subscription to query resources under the subs
        $null = Set-AzContext -Subscription $Subscription.Name

        Write-Host "set context to sub" $Subscription.Name

        # Get all VMs in the Sub
        $vms = Get-AzVM

        ForEach ($vm in $vms)
        {

            Try
            {

                Add-AzVmGalleryApplication -VM $vm -GalleryApplication $app_instance -TreatFailureAsDeploymentFailure

                # Apply the VM application
                $vm | Update-AzVM

                write-host "installing app for" $vm.Name "under" $Subscription.Name "subs"
                Add-Content -Path $outfile -Value "$($Subscription.Name),$($vm.Name),'App Assignment successful'"
            }
            Catch
            {
                $errorMessage = $_.Exception.Message
                $failedItem = $_.Exception.ItemName
                    
                Add-content -Path $errorfile -Value "$($Subscription.Name),$($vm.Name),'Error -' $errorMessage $failedItem"
        
                Continue # Go back to top of loop for next user
            } 
        }
    }
}
```

---

&nbsp;

## Using Run Commands 

Prerequisites: 

   - Create a **Storage Account** where we store the application installation source files

   - Set up a **VM** to run the PowerShell script that will trigger the Run Command scripts in required VMs


Steps:

   1. Run this PowerShell cmdlet below to trigger a Run Command on a VM:
   

```powershell

#############################################################################################
# Set Run Cmd that will run script in VM
# Ideally run the script from an AZ VM that has a managed ID to access the SA files 
#############################################################################################

Set-AzVMRunCommand `
-ResourceGroupName "myRg" `
-VMName "myVM" `
-RunCommandName "RunCommandName" `
-Location "VMRegion"

# Script can be local on the PC running the AZ cmd
-ScriptLocalPath "C:\MyScriptsDir\MyScript.ps1"

# OR 

# Script can be on a Blob
-SourceScriptUri "<SAS_URI_of_a_storage_blob_with_read_access_or_public_URI>" `
-OutputBlobUri "<SAS_URI_of_a_storage_append_blob_with_read_add_create_write_access>" `
-ErrorBlobUri "<SAS_URI_of_a_storage_append_blob_with_read_add_create_write_access>"

# OR 

# Script can be added inline
-SourceScript "id; echo HelloWorld"
```


   2. The Run Command will run the below PowerShell script which downloads the installer from the Storage account and runs the installer.

```powershell

#############################################################################################
# Install MSI and uses time-limited SAS token 
# Ideally run the script from an AZ VM that has a managed ID to access the SA files 
#############################################################################################

# Variables
$storageAccountName = "STORAGE_ACCOUNT_NAME" 
$containerName = "CONTAINER_NAME" 
$msiBlobName = "7zip-2408.msi" 
$localMsiPath = "C:\Temp\$($msiBlobName)" # Path where the MSI file will be downloaded to on the the VM

# Generated time-limited SAS token on SA Container
$SasToken = "SAS_TOKEN" # Choose SAS Token, not URL

# Create the context for the storage account
$context = New-AzStorageContext -StorageAccountName $storageAccountName -SasToken $SasToken

# Download the MSI file from the storage account
Get-AzStorageBlobContent -Container $containerName -Blob $msiBlobName -Destination $localMsiPath -Context $context

# Install the MSI file
Start-Process msiexec.exe -ArgumentList "/i $localMsiPath /quiet /norestart" -Wait
```


---

&nbsp;

Hope this was useful. 

I aim to add information on using Azure Automation Runbooks and Custom Script Extensions to deploy apps in a future update.

&nbsp;

[Back to Project List](../projects) &emsp; &emsp; &emsp; [Back to Top](#top)