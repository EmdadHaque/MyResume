

## Automate JIT RDP Access and Approvals to Azure VMs with MS Forms and Azure Logic Apps  
_Date: 6 Dec 2024_

**Scenario**: In this environment, end users often require RDP access to a specific Azure VMs that are publicly available i.e. acting as bastion hosts. Moreover, the access requirement is temporary. 

**Requirement**: For security hardening, only Just-In-Time (JIT) RDP access needs to be provided in order to lock down the access to a specific public IP, for a specific duration, and to ensure the request is approved first.

**Aim**: 
Currently, users create a ticket for RDP access to a specific Azure VM and provide their pubic IP that needs to be allow-listed. 

This ticket gets triaged to the Azure admin team who evaluate the request and create the JIT access to the VM from the Azure Portal or via CLI. 

As you can imagine, the manual process is slow as it it done manually and dependent on an Azure admin's availability.

The goal is to automate this so that end users can request the JIT access that forces them to provide the required information like the public IP they will access from, and an approver (who can be a non-admin user) can allow it, so that the JIT access is provided without need for an admin user to get involved. 

**Background**: 

JIT access to Azure VMs is a feature of MS Defender for Cloud. You can learn more about it from [MS Learn - Understanding just-in-time (JIT) VM access](https://learn.microsoft.com/en-us/azure/defender-for-cloud/just-in-time-access-overview?tabs=defender-for-container-arch-aks).

The process to enable JIT is described in this [MS Learn - Enable just-in-time access on VMs](https://learn.microsoft.com/en-us/azure/defender-for-cloud/just-in-time-access-usage). 

In short, an Azure admin will set up the Azure VM to be protected by JIT and create rules for the JIT access policy. For example, a rule for port 3389 will be set up for RDP access. 

After that, roles with the correct permission can provide JIT access by using the rule and specifying the parameters like source IP, protocol, request time (duration).

This creates a temporary rule on the Network Security Group (NSG) attached to the specified VM allowing traffic for the requested parameters in the JIT request. 
&nbsp; 
Below, I have discussed in details how I have automated the process so that this JIT access is enabled when an end user's JIT request is received and approved.

A summary of the steps is listed:
   - M365 Forms is used to gather the parameters for the JIT access from end-users. 
   - Form submission triggers an Azure Logic App
   - The Logic App sends Approval email notifications to approvers
   - Approvers can approve or reject directly from the email itself
   - Based on the approval decision and the JIT parameters, the Logic App creates the JIT access and sends a notification email of the outcome to the end-user/requestor.

&nbsp;

## Set up MS Forms 

- Create a M365 Form. 

For demo purposes, I am using only 2 VMs and only asking for the public IP that needs to be allowlisted. 

We can ask for other details such as RDP or SSH or some other port, time and duration etc. 

   ![](/assets/img/projects/jit_access/ms-form-fields.png)

For security, I have allowed only people from my Entra ID tenant to access the form i.e. users will require to sign-in to access the form. 

This can be further locked down to a group of users who can access the form. 

   ![](/assets/img/projects/jit_access/ms-form-settings.png)

&nbsp; 

## Set up a Logic App

Create a Logic App and use the Logic App Designer to create the app logic as follows:

- Trigger the Logic App when a form response is submitted. 

![](/assets/img/projects/jit_access/logic-app-trigger.png)

- Initialise the parameters we are collecting from the Form into variables.

![](/assets/img/projects/jit_access/logic-app-vars.png)

- Depending on the VM name, select the Resource Group. 
   The resource group of the VM needs to mentioned in the HTTP call when requesting JIT access as you can see in the later steps.

![](/assets/img/projects/jit_access/logic-app-set-rg.png)

- Send an Approval Email to an approver or a group of approvers.

![](/assets/img/projects/jit_access/logic-app-approval.png)

![](/assets/img/projects/jit_access/logic-app-approval-details.png)

- When Approval is received, use an HTTP request to create the JIT RDP Access and send a confirmation email to the requester.

![](/assets/img/projects/jit_access/logic-app-http.png)

**URI**:
```
https://management.azure.com/subscriptions/<sub-id>/resourceGroups/@{variables('var-rg')}/providers/Microsoft.Security/locations/<az-region>/jitNetworkAccessPolicies/default/initiate?api-version=2020-01-01
```

**Method**: POST

**Body**:
```
{
   "virtualMachines": [
      {
         "id": "/subscriptions/<sub-id>/resourceGroups/@{variables('var-rg')}/providers/Microsoft.Compute/virtualMachines/@{variables('var-vm')}",
         "ports": [
            {
               "number": 3389,
               "duration": "PT24H",
               "allowedSourceAddressPrefix": "@{variables('var-ip')}"
            }
         ]
      }
   ]  
}
```

- If rejected by approver, send a rejection email to requester.    
The confirmation and rejection emails are just simple notification emails without any options unlike the Approval email.

![](/assets/img/projects/jit_access/logic-app-rejection.png)

&nbsp;

## Securing the solution

   - The **MS Forms** has been secured using settings such that only users who are part of the Entra ID Tenant can access the Form.

     This can be further restrcited to a specific group of users in the Entra ID.

   - The **Logic App** has been assigned a System-Assigned Managed Identity.
     
     A custom role has been created that only allows for requesting JIT access to a VM as described in [MS Learn - Enable just-in-time access on VMs - Prerequisites](https://learn.microsoft.com/en-us/azure/defender-for-cloud/just-in-time-access-usage#prerequisites):

      ```
      - Microsoft.Security/locations/jitNetworkAccessPolicies/initiate/action
      - Microsoft.Security/locations/jitNetworkAccessPolicies/*/read
      - Microsoft.Compute/virtualMachines/read
      - Microsoft.Network/networkInterfaces/*/read
      - Microsoft.Network/publicIPAddresses/read
      ```

      The manged identity has been assigned to this role at the Landing Zone Management Group that encompasses all subscriptions with business workload VMs.   
 
---
&nbsp;    

I hope this was informational and helps you build your own secure solution based on the steps demonstrated here. 

&nbsp;

[Back to Project List](../projects) &emsp; &emsp; &emsp; [Back to Top](#top)