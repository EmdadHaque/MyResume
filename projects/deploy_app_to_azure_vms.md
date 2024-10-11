
## Automate the deployment of applications to Azure Windows Virtual Machines
_Date: 10 Oct 2024_


**Scenario**: The environment has multiple Windows VMs in Azure which are not being managed by any Configuration Management tools, most are also not connected to any AD. 

**Requirement**: I needed to deploy an agent (with an MSI installer) to these VMs in an automated way instead of logging into each VM and manually installing.

**Research**: I found several ways to do this, however I had to choose one that best suited my scenario.
   1. Use Run Commands => this was the quickest and the modern way of doing it at the time of writing.
   2. Use Custom Script Extension => this is the same as using older way of doing it.
   3. Use Azure Automation Runbooks => this is the same as using Run Commands but your script is running in Azure's Automation account instead of from your own VM.
   4. Use a Configuration Management tool such as Intune or SCCM or Ansible etc. => needs time to set up the tool itself.
   5. Use Group Policy => needs VMs to be added to AD.



Aim: 




- **[Serverless Web App](https://eh-serverless-webapp-proj1.s3.ap-southeast-2.amazonaws.com/index.html){:target="_blank"}** - This is a demo for hosting a web app using serverless AWS services: 
   1. S3 for hosting HTML website that uses JavaScript for API calls
   2. API Gateway for handling API calls
   3. Lambda for app logic coded in Python
   4. Dynamo DB for storing data