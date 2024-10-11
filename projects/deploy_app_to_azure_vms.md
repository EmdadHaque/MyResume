
## Automate the deployment of applications to Azure Windows Virtual Machines
_Date: 10 Oct 2024_

**Aim**: To install applications on Windows VMs in Azure via scripts in the absence of any configuration management tools.   

**Scenario**: The environment has multiple Windows VMs in Azure which are not being managed by any Configuration Management tools, most are also not connected to any AD. 

**Requirement**: Deploy an agent (with an MSI installer) to these VMs in an automated way instead of logging into each VM and manually installing.

**Research**: There are several ways to do this:

   - **Run Commands** => this is the quickest and the modern way at the time of writing.

   - **Custom Script Extension** => this is an older way of achieving the same at the time of writing.

   - **Azure Automation Runbooks** => this is the same as using Run Commands but your script is running in Azure's Automation account instead of from your own VM.

   - **A Configuration Management tool** such as Intune or SCCM or Ansible etc. => needs time to set up the tool itself and the tools have their own limitations as well.

   - **Group Policy** => needs VMs to be added to AD.


&nbsp;
## Using Run Commands 

   - Set up a VM to run the PowerShell script that will trigger the Run Command scripts in required VMs
   
   - Set up a Storage Account to hold the installer file as a blob 

   - Create PowerShell script to trigger Run Command

   - Create Script that will download the installer from the Storage account and run the installer

