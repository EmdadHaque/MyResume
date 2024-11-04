

## RDP to an Azure VM using MS Entra ID credentials
_Date: 30 Oct 2024_  

**Scenario**: The environment has Windows VMs in Azure which are not connected to AD and need to be accessed via RDP. This is mainly going to be used by external partners.  

**Requirement**: Security is priority-zero so the Remote Desktop Connection must occur in a secure manner. 

**Research**: 

To secure the Remote Desktop Connection we will require a **Point-to-Site VPN connection**. Other secure connections are possible too such as Express routes or site-to-site VPNs.

We will also use **Multi-Factor Authentication** as an added measure of security.

This solution could be replaced by _Azure Virtual Desktops_ which would be a more robust solution, however there are times in the real world when we need a quick solution with existing resources. 

&nbsp; 

---

### Prerequisites: 

- Set up an Azure Point-to-Site VPN because the VMs will not have public IPs. Follow the steps in this [link](https://techcommunity.microsoft.com/t5/itops-talk-blog/step-by-step-creating-an-azure-point-to-site-vpn/ba-p/326264){:target="_blank"} to set up the VPN.


### Steps:

- When creating the VM, ensure that you select the _"Login with Entra ID"_ under the _Management_ tab.

    ![](/assets/img/projects/rdp_entra_id/login_with_entra_id.png)

    Once the VM is created, under the VM's "Extensions" you will notice the "AADLoginForWindows" extension, which will allow the RDP access with an Entra ID.

    ![](/assets/img/projects/rdp_entra_id/vm_ext_aadlogin.png)

- If the VM is not connected to AD, add the DNS Suffix of the network to your VM's hostname at the OS level. 
    
    ![](/assets/img/projects/rdp_entra_id/vm_dns_suffix.png)

- To control who can RDP to the VM as a standard user or as an administrator, use the built-in roles _Virtual Machine User Login_ and _Virtual Machine Administrator Login_ respectively. Go to Azure Portal and select the VM > _Access Control (IAM)_ > _Add Role Assignment_ > Select one of the above roles and the user or group you wish to have this role.

- To add MFA prompt for the RDP access, create a **Conditional Policy** that Grants access to Users for All Resources when the MFA Requirement is met. This will cover the RDP access to the VM and require MFA.

- To RDP to the VM:
    - Connect to the Azure VPN from your local device.

    - Open Remote Desktop Connection app from your local device and from the Advanced tab, select _"Use a web account to sign in to the remote computer"_ 
    ![](/assets/img/projects/rdp_entra_id/rdc_web_account.png)
    
    - Add the remote computer's FQDN (OS hostname plus the DNS Suffix)
    
    - Enter the username, password and MFA code when you the MS Entra ID modern authentication credential prompt. 
    
    - Select _Yes_ when prompted to _Allow remote desktop connection?_ 
    ![](/assets/img/projects/rdp_entra_id/rdp_prompt.png) 


You should now be able to access the Azure VM via RDP using the Entra ID credentials and MFA over a VPN connection. Hope this was helpful.  

---
&nbsp;

[Back to Project List](../projects) &emsp; &emsp; &emsp; [Back to Top](#top)