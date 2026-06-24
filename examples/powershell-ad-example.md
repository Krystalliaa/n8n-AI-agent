Building a Basic Active Directory Lab
This document details the process of setting up a basic Active Directory lab environment using two virtual machines. This project aims to build practical skills in Active Directory and networking, which are essential for IT and cybersecurity roles. By completing this lab, I gained a better understanding of how domain controllers function, user management within a domain, and the configuration of essential network services.

Why I Did This Project
I wanted to build practical skills in Active Directory and networking, which are critical for IT and cybersecurity roles. This lab setup helped me understand how domain controllers work, how to manage users, and how to handle essential network services.

Tools I Used
Oracle VirtualBox for virtualization
Windows Server 2019 ISO for the domain controller
Windows 10 ISO for the client machine
Lab Components
This lab consists of two virtual machines:

DC: The Domain Controller (running Windows Server 2019)
Client1: A Windows 10 client machine joined to the domain
What I Did (Step-by-Step)
1. Creating the Virtual Machines
The first step was to set up the virtual environment. I downloaded Oracle VirtualBox from the official website (https://www.virtualbox.org/) and the required ISO files for Windows Server 2019 and Windows 10.

Then, within VirtualBox, I created two new virtual machines:

A VM named "DC" for the domain controller.
A VM named "Client1" for the Windows 10 client.
For each VM, I followed these general steps:

Opened VirtualBox and clicked "New".
Provided a name for the VM (e.g., "DC" or "Client1").
Selected the appropriate operating system type and version (Microsoft Windows, Windows Server 2019 (64-bit) for DC and Microsoft Windows, Windows 10 (64-bit) for Client1).
Allocated RAM (e.g., 4GB for DC, 2GB for Client1 - adjust based on your system resources).
Created a virtual hard disk (VDI, dynamically allocated, with sufficient storage – 50GB is a good starting point).
In the VM settings, under "Storage," I attached the corresponding ISO file to the virtual DVD drive.
I then configured the network adapters for each VM:

For the DC VM, I set up two network adapters:
Adapter 1: Attached to "NAT" for internet access.
Adapter 2: Attached to "Internal Network". I created a new internal network and named it (e.g., "AD-Lab"). This adapter will be used for communication between the DC and Client1.
For the Client1 VM, I set up one network adapter:
Adapter 1: Attached to the same "Internal Network" ("AD-Lab") as the DC's second adapter. This ensures that the client can communicate with the domain controller.
This network configuration allows both VMs to communicate with each other on the internal network while the DC also has access to the internet through NAT.

Configuring the Domain Controller (DC)
With the virtual machines created, the next step involved configuring the Domain Controller (DC) VM. Here's what I did on the DC machine:

2.1 Setting Up Network Settings
First, I configured the network settings of the DC VM to ensure proper internal communication and internet access. This involved setting static IP addresses for both the internal and external network adapters:

Within the DC VM settings, I accessed the "Network" section.
For the "Adapter 1" (NAT):
Selected "Bridged Adapter" mode (if your network allows it, for internet access).
(Alternatively, you can use "NAT" mode if your network doesn't allow bridged connections.)
Obtained the IP address automatically (DHCP) or manually assigned a static IP address suitable for your network configuration.
For the "Adapter 2" (Internal Network):
Selected "Internal Network" mode and ensured it was connected to the previously created internal network (e.g., "AD-Lab").
Manually assigned a static IP address within the internal network range (e.g., 192.168.1.100). Make sure this IP address doesn't conflict with any existing devices on your network.
Set the subnet mask appropriately (e.g., 255.255.255.0) and configured the default gateway if needed (depending on your network).
2.2 Renaming the Server
Next, I renamed the server to reflect its role as a Domain Controller. Within the DC VM, I followed these steps:

Opened the "System Properties" window (search for "System Properties" in the Start menu).
Clicked on "Change settings" under "Computer name, domain, and workgroup settings".
In the "Computer name" tab, clicked "Change".
Entered the new name "DC" and confirmed the change.
(A restart might be required for the name change to take effect.)
2.3 Promoting the Server to Domain Controller
Finally, I configured the DC VM as a domain controller for the newly created domain "mydomain.com". This involved promoting the server using the Server Manager tool.

Opened Server Manager on the DC VM.
Clicked on "Add roles and features".
In the wizard, selected "Role-based or feature-based installation" and clicked "Next".
Selected the "Select a server from the server pool" option and chose the DC server.
Clicked "Next" and then "Next" again on the feature selection page.
On the "Server Roles" page, expanded "Active Directory Domain Services" and checked the box next to "Active Directory Domain Services".
Clicked "Next" on several subsequent screens, accepting default options for features and installation type.
On the "Confirm Installation Selections" page, reviewed the selections and clicked "Next".
On the "Pre-installation Tasks" page, reviewed any warnings and clicked "Install".
The installation process might take some time. Once completed, the wizard prompted for a restart. I clicked "Restart Now" to reboot the DC server.
After the restart, the DC promotion wizard launched automatically. This wizard guided me through additional configuration steps specific to creating a new Active Directory forest and domain. Here's a summary of the configuration choices I made:

Selected "Add a new forest" option.
Entered the root domain name "mydomain.com".
Provided a strong password for the Domain Administrator account (use a complex password for security).
Reviewed and confirmed the configuration details.
Completed the wizard, which finalized the promotion of the DC server to a domain controller for the "mydomain.com" domain.3. Creating a Domain Admin and Users
After promoting the server to a Domain Controller and establishing the "mydomain.com" domain, the next crucial step was to manage user accounts and organizational structure within Active Directory. This involved creating a dedicated domain administrator account, organizing the directory using Organizational Units (OUs), and creating a standard user account.

3.1 Creating a Dedicated Domain Admin Account
As a security best practice, it's highly recommended to create a separate, dedicated account for administrative tasks instead of using the built-in "Administrator" account directly. This limits the potential impact of compromised credentials. I created a new domain admin account as follows:

Opened "Active Directory Users and Computers" (dsa.msc) from the "Tools" menu in Server Manager.
Right-clicked on the domain ("mydomain.com") in the left-hand pane.
Selected "New" -> "User".
Entered the following information for the new domain admin account (e.g., "DomainAdmin"):
First name: Domain
Last name: Admin
User logon name: domainadmin
Clicked "Next".
Set a strong, complex password for the "DomainAdmin" account and unchecked "User must change password at next logon" (in a production environment, you would enforce password changes).
Clicked "Next" and then "Finish".
To grant this new user domain administrator privileges, I right-clicked the "domainadmin" user account, selected "Properties", and navigated to the "Member Of" tab.
Clicked "Add..." and typed "Domain Admins". Clicked "Check Names" to resolve the group name and then clicked "OK".
Clicked "OK" on the user's properties window to save the changes.
3.2 Creating an Organizational Unit (OU)
Organizational Units (OUs) are used to organize users, computers, and other Active Directory objects into logical groups. This simplifies administration and allows for more granular application of Group Policies. I created an OU named "Users" to contain standard user accounts:

In "Active Directory Users and Computers", right-clicked on the domain ("mydomain.com").
Selected "New" -> "Organizational Unit".
Entered "Users" as the name for the OU and clicked "OK".
3.3 Creating a Standard User and Promoting to Domain Admin (Example)
For demonstration purposes (and to exemplify the risk of promoting regular users to domain admins), I created a standard user and then promoted them to a domain administrator. In a real-world scenario, you would rarely do this and would use the dedicated admin account created earlier.

Inside the "Users" OU, right-clicked and selected "New" -> "User".
Entered the following information for the standard user (e.g., "TestUser"):
First name: Test
Last name: User
User logon name: testuser
Clicked "Next".
Set a password for the "TestUser" account.
Clicked "Next" and then "Finish".
To demonstrate the risk, I then promoted "TestUser" to a Domain Admin by following the same steps as for the "domainadmin" account (right-click, Properties, Member Of, Add "Domain Admins").
It's important to understand that granting a regular user Domain Admin privileges significantly increases the security risk. If this user's account is compromised, the attacker gains full control of the domain.

4. Setting Up DHCP and NAT
To enable network services like DHCP and provide Network Address Translation (NAT) for internet access from the internal network, I configured Routing and Remote Access Services (RRAS) on the DC. This allows the client machine (Client1) to obtain an IP address automatically and access the internet through the DC.

4.1 Enabling Routing and Remote Access (RRAS) and NAT
Opened Server Manager and clicked "Manage" -> "Add Roles and Features".
Selected "Role-based or feature-based installation" and clicked "Next".
Selected the server (DC) and clicked "Next".
On the "Server Roles" page, checked the box next to "Remote Access" and clicked "Next".
On the "Features" page, clicked "Next".
On the "Remote Access" page, clicked "Next".
On the "Role Services" page, checked the box next to "Direct Access and VPN (RAS)" and "Routing". Clicked "Add Features" in the pop-up window.
Clicked "Next" through the remaining pages and then clicked "Install".
After the installation, opened the "Routing and Remote Access" console (search for "Routing and Remote Access" in the Start menu).
Right-clicked the server name (DC) in the left-hand pane and selected "Configure and Enable Routing and Remote Access".
In the wizard, selected "Network address translation (NAT)" and clicked "Next".
Selected the network interface connected to the internet (the NAT adapter or Bridged Adapter) and clicked "Next".
Clicked "Finish" to complete the configuration.
4.2 Configuring the DHCP Server
With NAT configured, I then set up a DHCP server on the DC to automatically assign IP addresses to clients on the internal network. I used the following settings:

IP range: 172.16.0.100 - 172.16.0.200
Subnet mask: 255.255.255.0
Gateway: 172.16.0.1
DNS server: 172.16.0.1 (This is the IP address of the DC, which will also act as the DNS server for the domain.)
Here are the steps to configure the DHCP server:

In Server Manager, clicked "Manage" -> "Add Roles and Features".
Selected "Role-based or feature-based installation" and clicked "Next".
Selected the server (DC) and clicked "Next".
On the "Server Roles" page, checked the box next to "DHCP Server" and clicked "Add Features" in the pop-up window.
Clicked "Next" through the remaining pages and then clicked "Install".
After the installation, completed the DHCP post-install configuration by clicking on the yellow notification flag in Server Manager and selecting "Complete DHCP Configuration".
Authorized the DHCP server in Active Directory.
Opened the "DHCP" console (search for "DHCP" in the Start menu).
Expanded the server name (DC) and right-clicked on "IPv4". Selected "New Scope".
In the "New Scope Wizard", entered a name for the scope (e.g., "InternalNetwork").
Entered the IP address range:
Start IP address: 172.16.0.100
End IP address: 172.16.0.200
Entered the subnet mask: 255.255.255.0.
Clicked "Next" through the exclusions and lease duration settings (you can adjust these as needed).
Configured the DHCP options:
Default Gateway: 172.16.0.1
DNS Server: 172.16.0.1
Activated the scope by selecting "Yes, I want to activate this scope now" and clicked "Finish".
With these steps, the DHCP server is now configured to provide IP addresses, gateway, and DNS server information to clients on the internal network, enabling them to communicate within the network and access the internet through the DC's NAT configuration.

5. Using PowerShell to Automate User Creation
To streamline the process of adding multiple users to Active Directory, I utilized a PowerShell script. This provided valuable experience with automation and scripting within an Active Directory environment.

5.1 Automating User Creation with PowerShell
Using PowerShell scripts for user creation is a best practice for managing large numbers of users efficiently. It reduces manual effort and minimizes the risk of errors. I used the script in the following repository:

AD_PS Script: https://github.com/joshmadakor1/AD_PS
This script allowed me to define user attributes (such as username, password, department, etc.) in a structured format (e.g., CSV file) and then automatically create the users in Active Directory based on that data. This approach is significantly more efficient than manually creating each user through the Active Directory Users and Computers console.

6. Configuring the Client Machine (Client1)
With the Active Directory domain and services set up on the DC, the final step involved configuring the Client1 virtual machine to join the domain "mydomain.com". This allows users on Client1 to leverage the domain resources and centralized authentication.

6.1 Network Configuration
To ensure proper communication with the domain and internet access, I configured the network adapter settings on Client1:

Opened the "Network Connections" window (search for "Network Connections" in the Start menu).
Right-clicked on the network adapter and selected "Properties".
Double-clicked on "Internet Protocol Version 4 (TCP/IPv4)".
Selected "Obtain an IP address automatically" (since the DHCP server is running on the DC) and "Obtain DNS server address automatically" (the DC will also act as the DNS server).
Clicked "OK" to save the changes.
6.2 Joining the Domain
Once the network settings were configured, I joined Client1 to the domain using the following steps:

Opened "System Properties" (search for "System Properties" in the Start menu).
Clicked on "Change settings" under "Computer name, domain, and workgroup settings".
In the "Computer Name" tab, clicked "Change".
Selected "Join a domain" and entered the domain name "mydomain.com".
Clicked "Next" and provided the credentials of a user with domain join permissions (e.g., the "DomainAdmin" account created earlier).
Clicked "OK" through any prompts and rebooted the Client1 VM.
After the reboot, Client1 was successfully joined to the domain "mydomain.com". Logging into the Client1 machine using a domain user account was a rewarding accomplishment, signifying a functional Active Directory lab environment.

What I Learned
This project provided valuable hands-on experience with several key concepts related to networking and Active Directory administration. Here's a summary of the key takeaways:

Installing and Configuring Active Directory: I gained practical experience in installing Active Directory Domain Services (AD DS) on a Windows Server and promoting it to a Domain Controller. This included understanding the required prerequisites, configuring the server roles, and navigating the domain promotion process.
The Importance of DNS in a Domain Environment: I learned how crucial DNS is for name resolution within a domain. The Domain Controller acts as the primary DNS server for the domain, allowing clients to locate domain resources and services by name rather than IP address.
Setting up DHCP to Manage IP Addresses: Configuring a DHCP server on the DC provided insight into automatic IP address assignment. I learned how to define IP address ranges, subnet masks, default gateways, and DNS server settings within a DHCP scope, simplifying network administration.
Using PowerShell Scripts to Automate Tasks: Using PowerShell to automate user creation demonstrated the power of scripting for managing Active Directory objects efficiently. This included understanding how to use scripts to create multiple users with specific attributes, saving significant time and effort compared to manual creation.
How NAT Enables Internet Access for Internal Networks: Configuring NAT on the DC allowed the internal network to access the internet using the DC's external IP address. I learned how NAT translates internal IP addresses to external IP addresses, enabling multiple devices on a private network to share a single public IP address.
This hands-on experience has significantly enhanced my understanding of Active Directory, networking fundamentals, and automation techniques. These skills are essential for anyone working in IT or cybersecurity.

credit to [Joshmadakor]