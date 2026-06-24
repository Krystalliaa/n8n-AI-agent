Active Directory Home Lab Setup
Hello! I'm a Junior Cybersecurity Analyst building this home lab to gain practical experience with Active Directory, Splunk, Sysmon, and other essential tools. This guide documents my process, providing a step-by-step walkthrough for anyone looking to replicate this setup. This will serve as a proof that I have hands on experience with the systems.

Network Topology
Network Topology Diagram

This diagram illustrates the network setup for our lab. We'll be using four virtual machines (VMs):

ADDC01 (Windows Server 2022): Our Domain Controller, responsible for authentication and authorization.
Target-PC (Windows 10): A client machine joined to the domain, used for testing and attacks.
Splunk (Ubuntu Server): Our Security Information and Event Management (SIEM) server, collecting and analyzing logs.
Kali (Kali Linux): An attacker machine used for simulating attacks and testing defenses.
Setting up the Virtual Environment
We'll be using Oracle VirtualBox for virtualization. You can download it from https://www.virtualbox.org/.

Creating the Virtual Machines
Create four VMs:

ADDC01
Target-PC
Splunk
Kali
Use the appropriate ISO images for each operating system. I used the evaluation version of Windows Server 2022.

Installing Operating Systems
Install each operating system on its respective VM.

Configuring Network Settings
Configuring NAT in Oracle VirtualBox
NAT configuration was performed in the Oracle VirtualBox settings. The steps are as follows:

Open the Oracle VirtualBox application.
Go to Tools > Properties > NAT Network
Name it AD-Project and the IPv4 Prefix to 192.168.10.0/24 according to the diagram.
Enable DCHP is checked.
Select each virtual machine and navigate to Settings > Network.
Set the Attached to option to NAT Network for internet connectivity.
Ensure the same NAT network is assigned to all machines for internal communication.
After configuring NAT in VirtualBox, proceed to configure network settings for each machine within the lab.

Creating and Installing Ubuntu Server 24.04.1 VM (Splunk Server)
In VirtualBox, click "New".
Name the VM (e.g., "Splunk"). Select "Linux" as the type and "Ubuntu (64-bit)" as the version.
Allocate RAM (e.g., 4GB).
Create a virtual hard disk (VHD) (VDI, dynamically allocated, e.g., 50GB).
In the VM settings, go to "Storage" and add the Ubuntu Server 24.04.1 ISO file.
Go to "Network" and configure the network settings (NAT or Bridged Adapter).
Start the VM.
Select "Try or Install Ubuntu Server".
The installer will guide you through the process. Select "Done" for most options until you reach the profile setup.
Enter your name, the server's name ("Splunk"), a username, and a strong password.
Do *not* select Ubuntu Pro. Continue with the installation.
After installation, the server will prompt for your Splunk login and password.
Log in and update the system:
sudo apt-get update && sudo apt-get upgrade -y
Confirm the password when prompted.
If asked which services to restart, select "OK".
Configuring Ubuntu Server (Splunk) - Post Installation
Setting a Static IP
To match the network topology, I set a static IP address for the Splunk server. I edited the netplan configuration file:

sudo nano /etc/netplan/00-installer-config.yaml
Changing IP in netplan

After making the necessary changes (addresses: "192.168.10.10/24" , dns: "8.8.8.8" , to: "default" , via: "192.168.10.1") I applied the changes and verified the IP address:

sudo netplan apply
ip a
Installing Splunk and Setting up Shared Folders
I signed up for Splunk and downloaded the .deb package. To easily transfer this file to the VM, I set up a shared folder using VirtualBox Guest Additions.

First, I installed the Guest Additions:

sudo apt-get install virtualbox-guest-additions-iso
VirtualBox Guest Additions Installation

Then, I configured the shared folder in VirtualBox settings:

Shared Folders Settings

After rebooting the Ubuntu VM:

sudo reboot
I attempted to add my user to the `vboxsf` group, but encountered an error:

Adduser Problem

sudo adduser mydfir vboxsf
I resolved this by installing the `virtualbox-guest-utils` package:

sudo apt-get install virtualbox-guest-utils
sudo reboot
sudo adduser mydfir vboxsf
Adding User to vboxsf

Next, I created a directory to mount the shared folder:

mkdir share
sudo mount -t vboxsf Active_Directory_Project share/
cd share/
ls -la
Installing and Configuring Splunk
Now that the Splunk .deb package was accessible in the shared folder, I installed it:

sudo dpkg -i splunk-9.3.2-d8bb32809498-linux-2.6-amd64.deb
cd /opt/splunk
ls -la
Users and Groups

I noticed that the files and directories belonged to the `splunk` user and group. To start Splunk, I switched to the `splunk` user:

sudo -u splunk bash
cd bin
./splunk start
I accepted the license agreement and set an administrator username and password ("mydfir" and a password of my choice). After the installation completed, I exited the `splunk` user's shell and enabled Splunk to start on boot:

exit
cd bin
sudo ./splunk enable boot-start -user splunk
Configuring Target-PC (Windows 10)
Setting Static IP and Hostname
I set the static IP to 192.168.10.100 and renamed the PC to "target-PC".

Installing Splunk Universal Forwarder and Sysmon
I downloaded and installed the Splunk Universal Forwarder, configuring it to send data to 192.168.10.10:9997.

I downloaded Sysmon and the Olaf Sysmon configuration:

Sysmon Configuration

cd C:\Users\bob\Download\Sysmon
.\Sysmon64.exe -i ..\sysmonconfig.xml
CD Sysmon

Configuring Splunk Forwarder Inputs
I created an `inputs.conf` file in `C:\Program Files\SplunkUniversalForwarder\etc\system\local`:

Inputs.conf File

[WinEventLog://Application]
index = endpoint
disabled = false
[WinEventLog://Security]
index = endpoint
disabled = false


[WinEventLog://System]
index = endpoint
disabled = false


[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = endpoint
disabled = false
renderXml = true
source = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational

I restarted the Splunk Forwarder service.

Configuring Splunk Server to Receive Data
I created an index named "endpoint" in Splunk and configured a receiving port on 9997.

I verified data ingestion by searching `index=endpoint` in Splunk.

Setting up Active Directory (ADDC01)
This section details the setup of Active Directory Domain Services (AD DS) on the Windows Server 2022 VM, promoting it to a Domain Controller (DC). This provides us with the capability to perform authentication and authorization using the Kerberos protocol.

Initial Server Configuration
Before installing AD DS, I performed some initial configuration:

Changing the Computer Name: I changed the computer name to "ADDC01", following the same procedure used for the Target-PC.
Installing Sysmon and Splunk Forwarder: I installed Sysmon and the Splunk Universal Forwarder on ADDC01 using the same instructions as for the Target-PC. This allows us to collect security logs from the DC as well.
Splunk Two Hosts

Setting a Static IP Address
A static IP is crucial for a Domain Controller. I set the IP address to 192.168.10.7 using the following steps:

Right-clicked the network icon in the system tray and selected "Open Network & Internet settings".
Clicked "Change adapter options".
Right-clicked the Ethernet adapter and selected "Properties".
Selected "Internet Protocol Version 4 (TCP/IPv4)" and clicked "Properties".
Selected "Use the following IP address" and entered the following:
IP address: 192.168.10.7
Subnet mask: 255.255.255.0
Default gateway: 192.168.10.1
Preferred DNS server: 192.168.10.7 (Crucially, the DC should point to itself for DNS)
Clicked "OK" on all dialog boxes.
Verified the configuration using the command prompt:
ipconfig
Installing and Configuring Active Directory Domain Services
With the static IP configured, I proceeded to install and configure AD DS:

Opened Server Manager and clicked "Manage" -> "Add Roles and Features".
Clicked "Next" on the "Before you begin" page.
Selected "Role-based or feature-based installation" and clicked "Next".
Selected the server (ADDC01) and clicked "Next".
Selected "Active Directory Domain Services" and clicked "Add Features" in the pop-up window.
Clicked "Next" through the remaining pages until the "Install" button appeared.
Clicked "Install".
Installation Succeded

After the installation completed, a yellow flag appeared in Server Manager. I clicked it and selected "Promote this server to a domain controller".
Selected "Add a new forest".
Entered the root domain name: "mydfir.local".
Clicked "Next".
Entered a Directory Services Restore Mode (DSRM) password and confirmed it. This password is crucial for recovery purposes.
Clicked "Next" through the DNS Options, Additional Options, Paths, and Prerequisites Check pages.
Clicked "Install".
The server rebooted automatically. After rebooting, the login screen showed the domain name: `MYDFIR\Administrator`, confirming the successful installation and promotion to a Domain Controller.
Creating Users and Organizational Units (OUs)
After logging in as `MYDFIR\Administrator`, I created Organizational Units (OUs) and user accounts:

In Server Manager, clicked "Tools" -> "Active Directory Users and Computers".
Right-clicked the domain ("mydfir.local") and selected "New" -> "Organizational Unit".
Named the OU "IT" and clicked "OK".
Inside the "IT" OU, right-clicked and selected "New" -> "User".
Entered the following information for the first user:
First name: "Jenny"
Last name: "Smith"
User logon name: "jsmith"
Clicked "Next".
Set the password to "Password1" (in a real-world scenario, use a much stronger password). Unchecked "User must change password at next logon".
Clicked "Next" and then "Finish".
Add New User Server

I repeated the process to create another OU named "HR" and a user named "Terry Smith" with the username "tsmith" and the same password.

Add New User Server 1

Now, with the domain set up and users created, I was ready to join the Target-PC to the domain.

Joining the Target-PC to the Domain
With the Active Directory domain now configured on ADDC01, the next step is to join the Target-PC to the domain. This allows users on the Target-PC to authenticate using domain credentials and access domain resources.

Accessing System Properties: On the Target-PC, I started by accessing the system properties. There are a few ways to do this, but I typically search for "PC" in the Windows search bar and then select "Properties" from the context menu.
Navigating to Advanced System Settings: In the System window, I scrolled down and clicked on "Advanced system settings". This opens the System Properties dialog box.
Changing Computer Name/Domain: In the System Properties dialog box, I selected the "Computer Name" tab. Then, I clicked the "Change..." button.
Joining the Domain: In the "Computer Name/Domain Changes" dialog box, I selected the "Domain" radio button and entered the domain name: "MYDFIR.LOCAL".
Join Domain
Providing Domain Credentials: Clicking "OK" prompted a window asking for an account with permissions to join computers to the domain. I entered the domain administrator credentials:
Username: "administrator"
Password: "Password1"
Successful Domain Join: After entering the correct credentials, a pop-up window appeared saying: "Welcome to the MYDFIR.LOCAL domain." I clicked "OK" on this message and then "OK" on the "Computer Name/Domain Changes" dialog box.
Restarting the Target-PC: A message appeared stating that I needed to restart the computer for the changes to take effect. I clicked "Close" and then "Restart Now".
Logging in with a Domain User: After the Target-PC restarted, I wanted to log in with the newly created domain user, Jenny Smith. On the login screen, I selected "Other user".
Entering Domain User Credentials: I entered the following credentials:
Username: "jsmith"
Password: "Password1"
Successful Domain Login: After entering the credentials, I was successfully logged in to the Target-PC as the domain user "jsmith". This confirmed that the Target-PC had successfully joined the "MYDFIR.LOCAL" domain.
Configuring Kali Linux and Preparing for the Attack
This section covers the configuration of the Kali Linux VM, including setting a static IP address, installing necessary tools, and preparing a password list for the brute-force attack.

Starting the Kali Linux VM: First, I started the Kali Linux virtual machine.
Setting a Static IP Address: To ensure consistent network communication, I assigned a static IP address of 192.168.10.250 to the Kali VM. The method for setting a static IP in Kali can vary slightly depending on the version and desktop environment. Here's the general process:
Right-click the network/internet icon at the top right of the screen and select "Edit Connections...".
Select the network profile (usually named "Wired connection 1") and click the cogwheel/settings icon.
Kali IP Configuration
Go to the "IPv4 Settings" tab.
Change the "Method" to "Manual".
Click the "Add" button and enter the following information:
Address: 192.168.10.250
Netmask: 24
Gateway: 192.168.10.1
DNS servers: 8.8.8.8
Click "Save".
Close the network settings window.
To verify the IP change, I opened a terminal (right-click on the desktop and select "Open Terminal Here") and used the following command:

ip a
If the IP address hadn't changed, I disconnected and reconnected the network interface within Kali by clicking on the network icon and selecting disconnect and then reconnect the profile we just edited. Then I checked the IP again using the same command:

ip a
Updating the System: To ensure I had the latest packages and security updates, I updated the Kali system:
sudo apt-get update && sudo apt-get upgrade -y
I entered my password when prompted and pressed Enter. I allowed the update process to complete.

Creating a Working Directory: I created a directory to store the files related to the attack:
mkdir ad-project
Installing Crowbar: I installed the `crowbar` tool, which is used for brute-force attacks against various services:
sudo apt-get install -y crowbar
I entered my password and pressed Enter to confirm the installation.

Preparing the Password List (rockyou.txt): I used the `rockyou.txt` wordlist, which is a common password list included in Kali.
Navigated to the wordlist directory:
cd /usr/share/wordlists/
Listed the files to confirm `rockyou.txt.gz` was present:
ls
Unzipped the compressed wordlist:
sudo gunzip rockyou.txt.gz
Listed the files again to confirm `rockyou.txt` is now extracted:
ls
Rockyou
Copied the `rockyou.txt` file to my working directory:
cp rockyou.txt ~/Desktop/ad-project
Changed to my working directory:
cd ~/Desktop/ad-project
To manage the size of the password list for this lab, I decided to use only the first 20 lines of the `rockyou.txt` file. I created a new file called `passwords.txt` containing these 20 lines:
head -n 20 rockyou.txt > passwords.txt
I viewed the contents of the new `passwords.txt` file:
cat passwords.txt
Since I knew the password I had set for the test users ("Password1"), I added it to the `passwords.txt` file to ensure the brute-force attack would succeed:
nano passwords.txt
(Inside the nano editor, I added "Password1" on a new line, saved the file by pressing Ctrl+X, then Y, then Enter)

Finally, I checked the contents of the `passwords.txt` file again to confirm the addition:
cat passwords.txt
Enabling Remote Desktop on Target-PC
To prepare the Target-PC for remote access and a simulated brute-force attack, I enabled Remote Desktop. Here are the detailed steps I followed:

Accessing System Properties: I started by opening the System Properties window. I searched for "PC" in the Windows search bar and selected "Properties".
Opening Advanced System Settings: In the System window, I clicked on "Advanced system settings" in the left-hand menu. This opened the System Properties dialog box.
Navigating to the Remote Tab: In the System Properties dialog box, I selected the "Remote" tab.
Enabling Remote Connections: In the "Remote Desktop" section, I selected the radio button for "Allow remote connections to this computer". This enables Remote Desktop on the Target-PC.
Selecting Allowed Users: To specify which users are allowed to connect remotely, I clicked the "Select Users..." button.
Adding Domain Users: In the "Remote Desktop Users" dialog box, I clicked "Add...". I then added the domain users "jsmith" and "tsmith":
I typed "jsmith" in the "Enter the object names to select" field.
I clicked "Check Names" to resolve the name against the domain. This ensures that the correct domain user is selected.
I repeated the process for "tsmith".
I clicked "OK" to close the "Select Users or Groups" dialog box.
Applying the Changes: I clicked "OK" on the "Remote Desktop Users" dialog box, "Apply" and then "OK" on the System Properties dialog box to save the changes.
Performing the Brute-Force Attack from Kali Linux
With Remote Desktop enabled on the Target-PC, I switched back to the Kali Linux VM to perform the brute-force attack using the `crowbar` tool.

Checking Crowbar Help: I first checked the help menu for `crowbar` to review the available options:
crowbar -h
Executing the Brute-Force Attack: I then executed the following command to perform the brute-force attack against the "tsmith" user on the Target-PC:
crowbar -b rdp -u tsmith -C passwords.txt -s 192.168.10.100/32
Crowbar Brute-Force Attack
This command specifies the following:

-b rdp: Use the RDP (Remote Desktop Protocol) service.
-u tsmith: Target the user "tsmith".
-C passwords.txt: Use the password list contained in the "passwords.txt" file.
-s 192.168.10.100/32: Target the specific IP address of the Target-PC (192.168.10.100). The `/32` indicates a single host.
Analyzing Logs in Splunk
After the brute-force attack, I returned to the Target-PC and accessed the Splunk web interface to analyze the security logs and see if the attack was detected.

Accessing Splunk Web Interface: I opened a web browser on the Target-PC and navigated to 192.168.10.10:8000. I logged in using the Splunk administrator credentials.
Navigating to Search & Reporting: From the Splunk home page, I selected "Search & Reporting" from the app menu.
Searching for Events: In the search bar, I entered the following search query to find events related to the "tsmith" user:
index=endpoint tsmith
I also set the time range to "Last 15 minutes" to focus on recent events.

Interpreting the Results: After running the search, I examined the results. The key information for analyzing the attack was the Event Code ID. Each event in Windows logs has a unique ID that describes the event.
Event Code IDs in Splunk
To understand the meaning of each Event Code ID, I would typically search online using resources like Microsoft's documentation or other online event ID databases. For example, Event ID 4625 indicates a failed login attempt. By correlating these Event Code IDs with the time of the brute-force attack, I could confirm that the attack was logged and identify the specific events generated.

This concludes the walkthrough of setting up a basic Active Directory environment, configuring Splunk for log collection, and simulating a brute-force attack. This lab provided valuable hands-on experience with these technologies and demonstrated the importance of security monitoring and strong password practices.

What I Learned
Here's a summary of my key learnings:

Setting up a Virtualized Lab Environment: I gained practical skills in using Oracle VirtualBox to create and manage multiple virtual machines, configuring their network settings for both internal communication and external access.
Active Directory Domain Services Installation and Configuration: I learned the process of installing AD DS on a Windows Server, promoting it to a Domain Controller, and creating a new domain. This included understanding the importance of DNS configuration during domain promotion.
User and Group Management in Active Directory: I gained experience in creating user accounts, organizing them into Organizational Units (OUs), and managing user permissions and group memberships. I also learned about the security best practice of creating dedicated administrator accounts.
Configuring Network Services (DHCP and NAT): I learned how to configure DHCP to automatically assign IP addresses to clients within the internal network, simplifying network administration. I also configured NAT to enable internet access for the internal network through the Domain Controller.
Splunk Deployment and Configuration: I gained hands-on experience installing and configuring Splunk on a Linux server, including setting up shared folders for file transfer and configuring Splunk to start on boot.
Splunk Universal Forwarder and Sysmon Integration: I learned how to deploy and configure the Splunk Universal Forwarder on Windows machines (both the Target-PC and the Domain Controller) to collect logs. I also integrated Sysmon for enhanced event logging and configured Splunk to ingest and index these logs.
Simulating a Brute-Force Attack and Log Analysis: I simulated a brute-force attack against the Target-PC using Kali Linux and the `crowbar` tool. This practical exercise demonstrated the importance of security monitoring and log analysis. I learned how to use Splunk to search for relevant events, including failed login attempts, and how to interpret Event Code IDs to understand the nature of security events.
Cross-Platform Integration: This lab involved integrating different operating systems (Windows Server, Windows 10, and Kali Linux) and different software applications (VirtualBox, Splunk, Sysmon, and Crowbar), providing valuable experience in managing a heterogeneous environment.
This hands-on lab significantly strengthened my understanding of Active Directory, network services, security monitoring, and attack simulation. It provided practical skills that are directly applicable to cybersecurity and system administration roles, and serves as demonstrable proof of my practical experience with these technologies.

credits to MyDFIR