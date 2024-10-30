

## RDP / Log in to Azure VM using MS Entra ID credentials
_Date: 30 Oct 2024_  

**Scenario**: The environment has Windows VMs in Azure which are not connected to AD and need to be accessed via RDP. 

**Requirement**: Security is priority-zero so the Remote Desktop Connection must occur in a secure manner. 

**Research**: 

To secure the Remote Desktop Connection we will require a Point-to-Site VPN connection. Other secure connections are possible too such as Express routes or site-to-site VPNs.

We will also use Multi-Factor Authentication as an added measure of security.

&nbsp; 

I have discussed these options in more details below as well as curated the steps for installing the 7zip app via its MSI as an example.

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