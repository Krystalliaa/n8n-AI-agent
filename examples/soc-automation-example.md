Setting up a SOC Automation Home Lab
This document outlines the setup of a home lab focused on Security Operations Center (SOC) automation. This lab aims to provide practical experience with tools and techniques used in modern SOC environments, particularly focusing on automation and orchestration. This documentation is intended for junior cybersecurity professionals looking to enhance their skills in this area.

Lab Components
This initial phase of the lab involves setting up a Windows 10 virtual machine, which will serve as a representative endpoint within our simulated network.

Windows 10 Pro VM: This virtual machine will simulate a typical user workstation and be used for testing automation scripts and tools. We will be installing the Professional edition of Windows 10 for its added features relevant to domain joining and management.
What I Did (Step-by-Step)
1. Creating the Windows 10 Pro Virtual Machine
The first step was to create the virtual environment using Oracle VirtualBox. I downloaded and installed Oracle VirtualBox from the official website (https://www.virtualbox.org/) and obtained a Windows 10 Pro ISO file.

Within VirtualBox, I created a new virtual machine for Windows 10 Pro, following these steps:

Open VirtualBox and Click "New": I launched the VirtualBox application and clicked the "New" button in the toolbar to begin creating a new virtual machine.
Name and Operating System Selection: In the "Create Virtual Machine" wizard, I provided a descriptive name for the VM (e.g., "Win10-Automation"). I then selected the operating system type as "Microsoft Windows" and the version as "Windows 10 (64-bit)".
Memory Size (RAM): I allocated an appropriate amount of RAM to the VM. 4GB (4096 MB) is usually sufficient for Windows 10, but you can adjust this based on your host system's resources.
Hard Disk: I selected "Create a virtual hard disk now" and clicked "Create". In the "Hard disk file type" window, I chose "VDI (VirtualBox Disk Image)". In the "Storage on physical hard disk" window, I selected "Dynamically allocated". This allows the virtual hard disk to grow as needed, saving space on the host system. I then allocated a reasonable size for the virtual hard disk (e.g., 50GB).
Virtual Machine Settings (Optional but Recommended): After creating the VM, I accessed its settings to make some optional but recommended adjustments:
System -> Processor: If your host system has multiple cores, you can allocate more than one core to the VM for better performance.
Display -> Video Memory: Increasing the video memory can improve graphics performance within the VM.
Storage -> Controller: IDE: I clicked on the empty disc icon and selected "Choose a disk file..." to select the Windows 10 Pro ISO file I had downloaded. This makes the ISO available to the VM as a bootable installation medium.
Network: I will configure the network settings later, after the OS installation is complete.
Start the Virtual Machine: After configuring the settings, I selected the newly created VM and clicked "Start". This launched the VM and began the Windows 10 Pro installation process.
The Windows 10 Pro installation process then proceeded as usual. I followed the on-screen prompts, selected my preferred settings, and created a local user account with a strong password.

Installing Sysmon on the Windows 10 Client
Sysmon is a powerful tool that enhances endpoint visibility by capturing detailed system activity. We will install Sysmon on our Windows 10 client machine (e.g., "Win10-Automation") to generate security events that will be collected by Wazuh.

Prerequisites
Windows 10 client machine with administrative privileges.
Downloaded Sysmon archive (e.g., `sysmon.zip`) from the official Microsoft Sysinternals website (https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon).
Downloaded Sysmon configuration file (e.g., `sysmonconfig.xml`) from a trusted source like the Olaf Hartong Sysmon configuration project on GitHub (https://github.com/olafhartong/sysmon-modular). Note: Exercise caution when using third-party configurations. Ensure you understand the contents of the configuration file before deploying it in your environment.
Steps
Extract the Sysmon Archive: Right-click the downloaded `sysmon.zip` file and select "Extract All...". Choose a convenient location for the extracted files (e.g., C:\Sysmon).
Obtain and Review the Sysmon Configuration:
Navigate to the Sysmon configuration file download page (e.g., GitHub repository linked above).
Carefully review the configuration file (e.g., `sysmonconfig.xml`) to understand the enabled monitoring options. This file defines what system activities Sysmon will capture.
Optional: You can customize the configuration file to tailor Sysmon's behavior to your specific needs. This might involve enabling or disabling certain monitoring options.
Copy the Configuration File: Copy the Sysmon configuration file (e.g., `sysmonconfig.xml`) to the extracted Sysmon folder (e.g., C:\Sysmon).
Open an Elevated PowerShell Window: Right-click the Windows Start menu button, select "Windows PowerShell (Admin)", or search for "PowerShell" and right-click "Run as administrator".
Navigate to the Sysmon Directory: Use the `cd` command to change the directory to the folder containing the extracted Sysmon files. For example, if you extracted the files to `C:\Sysmon`, run:
cd C:\Sysmon
Install Sysmon with Configuration: Run the following command to install Sysmon as a Windows service and apply the configuration file:
.\sysmon64.exe -i sysmonconfig.xml
Replace sysmonconfig.xml with the actual filename of your configuration file if it differs.

This command accepts the End-User License Agreement (EULA) for Sysmon.

**Note:** If you are using a 32-bit system, replace `sysmon64.exe` with `sysmon.exe` in the command.

Verify Installation (Optional): To confirm successful installation, you can:
Open the Windows Services window (search for "Services").
Locate the service named "Sysmon64".
Right-click the service and select "Properties". Verify that the service is running and the startup type is set to "Automatic".
Additional Notes
Sysmon logs events to the Windows Event Viewer under "Applications and Services Logs/Microsoft/Windows/Sysmon/Operational". You can use Event Viewer to inspect these logs and monitor system activity.

Consider scheduling regular reviews of the Sysmon configuration to ensure it aligns with your security requirements.

Setting up the Wazuh Manager on a DigitalOcean Droplet (Free Trial)
For this lab, I'm utilizing the DigitalOcean free trial to host the Wazuh Manager. While local VMs are a cost-effective long-term solution, using a cloud environment for initial exploration can be helpful. This section details the steps to create a Droplet (virtual server) on DigitalOcean.

Prerequisites
A DigitalOcean account (you will need to provide payment information for the free trial).
Steps
Sign Up for a DigitalOcean Account: Navigate to the DigitalOcean website and sign up for a new account using one of the available methods.
Provide Payment Information: After signing up, you will be redirected to a page prompting you to enter your credit card or other payment details. This is required even for the free trial.
Access the Control Panel: Once your account is set up, click "Explore our control panel" (or a similar button/link) usually located in the top left or center of the page.
Create a New Droplet: In the DigitalOcean control panel, click the "Create" button in the top right corner and select "Droplets".
Choose a Region: Select the region that is geographically closest to you. This can improve network latency.
Select an Operating System: Choose "Ubuntu" as the operating system for your Droplet.
Select a Plan: Choose the "Basic" plan.
Select CPU Options: Choose "Premium Intel" for better performance.
Select Droplet Size (Memory): Select a Droplet size with at least 8 GB of RAM. Wazuh can be resource-intensive, and 8GB is a good starting point for a lab environment.
Choose an Authentication Method: You have two options for connecting to your Droplet:
SSH Key Pair: This is the more secure method. If you are familiar with SSH keys, it is highly recommended you use this option.
Password: For simplicity in this lab setup, I chose to connect with a password. If you choose this option:
Create a strong, unique password.
**Crucially:** Store this password in a secure location (e.g., a password manager). Losing this password will make it very difficult to access your Droplet.
Choose a Hostname: Give your Droplet a descriptive hostname. I used "Wazuh" in this case.
Create the Droplet: Click the "Create Droplet" button to initiate the Droplet creation process. This may take a few minutes.
Once the Droplet is created, DigitalOcean will display its IP address. You will need this IP address to connect to your Wazuh Manager.

Configuring the DigitalOcean Firewall
To secure the Wazuh Manager Droplet and allow only necessary traffic, I configured a firewall in the DigitalOcean control panel. This ensures that only authorized connections from my public IP address can reach the Droplet.

Steps
Access the Firewall Section: From the left-hand menu in the DigitalOcean control panel, I selected "Manage" -> "Networking" -> "Firewalls".
Create a New Firewall: Clicked the "Create Firewall" button.
Name the Firewall: Provided a descriptive name for the firewall (e.g., "Wazuh Firewall").
Configure Inbound Rules (TCP): Configured the inbound rules for TCP traffic:
Changed the first "Type" option to "All TCP".
In the "Sources" section, deleted any existing entries.
Added my public IP address as a source. To find my public IP, I used a service like whatismyipaddress.com.
Configure Inbound Rules (UDP): Repeated the same process for UDP traffic:
Changed the "Type" option to "All UDP".
In the "Sources" section, deleted any existing entries.
Added my public IP address as a source.
Create the Firewall: Once both TCP and UDP rules were configured with my public IP as the source, I scrolled to the bottom of the page and clicked "Create Firewall".
Apply the Firewall to the Droplet: To associate the newly created firewall with the Wazuh Droplet:
From the left-hand menu, selected "Droplets".
Selected the "Wazuh" Droplet.
Selected the "Networking" tab.
Scrolled down to the "Firewalls" section and clicked "Edit".
Selected the newly created firewall (e.g., "Wazuh Firewall").
In the "Droplets" section of the Firewall configuration, clicked "Add Droplets".
Started typing the name of the Droplet ("Wazuh") and selected it from the suggestions.
Clicked "Add droplet".
By configuring the firewall in this way, I ensured that only traffic originating from my specified public IP address can reach the Wazuh Droplet on all TCP and UDP ports. This significantly enhances the security of the server.

Connecting to and Installing Wazuh on the Droplet
Now that the Droplet is set up and the firewall is configured, we can connect to it and install the Wazuh server software.

Connecting to the Droplet
**Method 1: Using the Droplet Console (Recommended)**

Access the Droplet: In the DigitalOcean control panel, navigate to "Droplets" and select the "Wazuh" Droplet.
Launch the Console: Click on the "Access" tab and then "Launch Droplet Console". This will open a terminal window within your browser where you can interact with the Droplet directly.
**Method 2: Using an SSH Client (Alternative)**

Download an SSH Client: If the Droplet Console doesn't work, download an SSH client like PuTTY for your local machine.
Connect with SSH: Use the provided IP address (from DigitalOcean) and port 22 (default SSH port) in your SSH client to connect to the Droplet. You will be prompted for the password you created during the Droplet setup.
Updating the System
Once connected to the Droplet, either through the console or SSH, run the following command to update the system's package lists and install any available updates:

apt-get update && apt-get upgrade -y
Explanation:

apt-get update: Updates the list of available packages from the repositories.

apt-get upgrade -y: Upgrades all installed packages to the latest versions. The `-y` flag automatically accepts any prompts during the upgrade process.

Important: When prompted to confirm which services to restart after the update, you can simply press Enter to accept the defaults.

Installing Wazuh
After the system is updated, run the following command to download and execute the Wazuh installation script:

curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
Explanation:

curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh: Downloads the Wazuh installation script from the official repository. The `-s` flag silences output during the download.

sudo bash ./wazuh-install.sh -a: Executes the downloaded script with root privileges (`sudo`) and the `-a` flag which selects the "all" installation option (including the Wazuh manager, worker, and API).

Crucially: During the installation, the script will **generate** a password for the Wazuh admin user. **Make absolutely sure to copy and save this generated password in a secure location (e.g., a password manager). You will need this password to access the Wazuh web interface.**

Accessing the Wazuh Web Interface
Once the installation completes, you can access the Wazuh web interface in a web browser. Open a new browser window and navigate to the following URL, replacing <your_public_ip_address> with the actual public IP address of your Wazuh Droplet:

https://<your_public_ip_address>
Login: Use the username "wazuh_admin" and the **generated** password (that you saved during the installation) to log in to the Wazuh web interface.

This will take you to the Wazuh dashboard where you can begin managing your security information and event management (SIEM) system.

Setting up TheHive on a DigitalOcean Droplet
Following the setup of the Wazuh Manager, we will now create a separate Droplet to host TheHive, our case management platform. We will use a similar procedure as with the Wazuh Droplet and configure the firewall to allow access.

Steps to Create TheHive Droplet
Access the Control Panel: Log in to the DigitalOcean control panel.
Create a New Droplet: Click the "Create" button in the top right corner and select "Droplets".
Choose a Region: Select the same region you used for the Wazuh Droplet for better performance and lower latency.
Select an Operating System: Choose "Ubuntu" as the operating system.
Select a Plan: Choose the "Basic" plan.
Select CPU Options: Choose "Premium Intel".
Select Droplet Size (Memory): Select a Droplet size with at least 8 GB of RAM.
Choose an Authentication Method: Choose the same authentication method (Password or SSH Key) you used for the Wazuh Droplet to maintain consistency. If using a password, ensure you create a strong password and save it securely.
Choose a Hostname: Give your Droplet a descriptive hostname, such as "TheHive".
Create the Droplet: Click the "Create Droplet" button.
Adding TheHive Droplet to the Firewall
Now, we will add the newly created TheHive Droplet to the existing firewall to allow access:

Access Firewalls: In the left menu, select "Networking" and then "Firewalls".
Select Your Firewall: Click on the name of the firewall you created for the Wazuh Droplet (e.g., "Wazuh Firewall").
Add TheHive Droplet:
In the firewall configuration page, locate the "Droplets" section.
Click "Add Droplets".
Start typing the hostname of your TheHive Droplet ("TheHive") and select it from the suggestions.
Click "Add droplet".
By adding TheHive to the existing firewall, we ensure that the same security rules apply to both the Wazuh and TheHive Droplets, simplifying management and maintaining a consistent security posture.

Installing TheHive on Your Droplet
This section details the installation of TheHive 5 and its necessary dependencies (Java, Cassandra, and Elasticsearch) on your DigitalOcean Droplet after you have connected via SSH.

1. Updating System Packages
Before installing any new software, it's crucial to update the system's package lists and upgrade existing packages to their latest versions. You can run the following command:

apt-get update && apt-get upgrade -y
Explanation:

apt-get update: Retrieves the latest package information from the configured repositories.

apt-get upgrade -y: Upgrades all upgradable packages on the system. The `-y` flag automatically answers "yes" to any prompts, automating the process.

Note: The && allows you to run multiple commands sequentially. The second command will only execute if the first command is successful.

2. Installing Dependencies
TheHive relies on several dependencies. Install them using the following command:

apt install wget gnupg apt-transport-https git ca-certificates ca-certificates-java curl software-properties-common python3-pip lsb-release
3. Installing Java
TheHive requires Java. We will install Amazon Corretto 11. You can run the following commands one by one, or you can copy and paste the entire block to execute them sequentially:

# Import the Amazon Corretto Java 11 signing key
wget -qO- https://apt.corretto.aws/corretto.key | sudo gpg --dearmor -o /usr/share/keyrings/corretto.gpg

# Add the Amazon Corretto Java 11 repository
echo "deb [signed-by=/usr/share/keyrings/corretto.gpg] https://apt.corretto.aws stable main" | sudo tee -a /etc/apt/sources.list.d/corretto.sources.list

# Update package lists
sudo apt update

# Install Java
sudo apt install java-common java-11-amazon-corretto-jdk

# Set JAVA_HOME environment variable
echo 'JAVA_HOME="/usr/lib/jvm/java-11-amazon-corretto"' | sudo tee -a /etc/environment

# Export the JAVA_HOME environment variable (optional but recommended)
export JAVA_HOME="/usr/lib/jvm/java-11-amazon-corretto"
Explanation: These commands add the Amazon Corretto repository, install the JDK, and set the JAVA_HOME environment variable.

Note: You can run all these commands at once by copying and pasting the entire block into your terminal.

4. Installing Cassandra
Cassandra is used as the database for TheHive. Install it using these commands. You can copy and paste the entire block to run them sequentially:

# Import the Apache Cassandra signing key
wget -qO- https://downloads.apache.org/cassandra/KEYS | sudo gpg --dearmor -o /usr/share/keyrings/cassandra-archive.gpg

# Add the Apache Cassandra repository
echo "deb [signed-by=/usr/share/keyrings/cassandra-archive.gpg] https://debian.cassandra.apache.org 40x main" | sudo tee -a /etc/apt/sources.list.d/cassandra.sources.list

# Update package lists
sudo apt update

# Install Cassandra
sudo apt install cassandra
Explanation: These commands add the Cassandra repository and install the Cassandra database.

Note: You can run all these commands at once by copying and pasting the entire block into your terminal.

5. Installing Elasticsearch
Elasticsearch is used for indexing and searching data within TheHive. Install it with the following commands. You can copy and paste the entire block to run them sequentially:

# Import the ElasticSearch signing key
wget -qO- https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

# Install apt-transport-https (if not already installed)
sudo apt-get install apt-transport-https

# Add the ElasticSearch repository
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list

# Update package lists
sudo apt update

# Install Elasticsearch
sudo apt install elasticsearch

# **Optional Elasticsearch Configuration**

# Create a jvm.options file (optional) and add the following lines to configure memory allocation:

# -Dlog4j2.formatMsgNoLookups=true
# -Xms2g
# -Xmx2g

# (Replace the memory values with your desired allocation)
Explanation: These commands add the Elastic repository, install Elasticsearch, and provide optional settings for the JVM.

Note: You can run all these commands at once by copying and pasting the entire block into your terminal.

6. Installing TheHive
Finally, install TheHive itself. You can copy and paste the entire block to run them sequentially:

# Import the Strangebee signing key
wget -O- https://archives.strangebee.com/keys/strangebee.gpg | sudo gpg --dearmor -o /usr/share/keyrings/strangebee-archive-keyring.gpg

# Add the Strangebee repository
echo 'deb [signed-by=/usr/share/keyrings/strangebee-archive-keyring.gpg] https://deb.strangebee.com thehive-5.2 main' | sudo tee -a /etc/apt/sources.list.d/strangebee.list

# Update package lists
sudo apt-get update

# Install TheHive
sudo apt-get install -y thehive
Explanation: These commands add the Strangebee repository and install TheHive.

Note: You can run all these commands at once by copying and pasting the entire block into your terminal.

Configuring TheHive: Cassandra and Elasticsearch
This section details the configuration of Cassandra and Elasticsearch, the database and search engine used by TheHive, on your DigitalOcean Droplet.

1. Configuring Cassandra
Cassandra serves as the database for TheHive. We need to modify its configuration file to ensure it listens on the correct IP address.

Edit the Cassandra Configuration File: Open the Cassandra configuration file using `nano`:
nano /etc/cassandra/cassandra.yaml
Modify Listen Addresses:
Press Ctrl + W to open the search function in `nano`.
Search for listen_address and replace the existing value with the public IP address of your TheHive Droplet.
Repeat the search and replace for rpc_address, also using your TheHive Droplet's public IP.
Update Seed Nodes:
Search for seeds.
Replace the existing seed addresses with your TheHive Droplet's public IP address followed by :7000.
Important: Ensure you leave the :7000 port specification unchanged.
Save and Exit `nano`:
Press Ctrl + X to exit.
Type y and press Enter to confirm saving the changes.
Stop and Restart Cassandra:
Stop the Cassandra service:
systemctl stop cassandra.service
Important: Because we installed TheHive using their package, we need to remove old data files:
rm -rf /var/lib/cassandra/*
Start the Cassandra service again:
systemctl start cassandra.service
Verify Cassandra Status:
Check the Cassandra service status:
systemctl status cassandra.service
Exit the status viewer by pressing q.
2. Configuring Elasticsearch
Elasticsearch is used for indexing and searching data within TheHive. We need to configure it to work correctly with our setup.

Edit the Elasticsearch Configuration File: Open the Elasticsearch configuration file with `nano`:
nano /etc/elasticsearch/elasticsearch.yml
Set Cluster Name:
Uncomment the cluster.name line (remove the #).
Change its value to "thehive".
Define Node Name:
Uncomment the node.name line.
Leave its value as node-1 (assuming this is your only Elasticsearch node).
Configure Network Settings:
Uncomment the network.host line.
Change its value to the public IP address of your TheHive Droplet.
Uncomment the http.port line and leave it set to 9200.
Set Cluster Discovery:
Uncomment the cluster.initial_master_nodes line.
Remove node-2 from within the brackets (if present, as we only have one node).
Save and Exit `nano`:
Press Ctrl + X to exit.
Type y and press Enter to confirm saving the changes.
Start and Enable Elasticsearch:
Start the Elasticsearch service:
systemctl start elasticsearch.service
Enable the Elasticsearch service to start on boot:
systemctl enable elasticsearch.service
Verify Elasticsearch Status:
Check the Elasticsearch service status:
systemctl status elasticsearch.service
Exit the status viewer by pressing q.
Configuring TheHive
This section guides you through configuring TheHive, ensuring it uses the correct IP address and is accessible from your browser.

1. Granting Ownership to TheHive User
Before modifying TheHive's configuration file, ensure the thehive user and group have ownership of the TheHive directory.

Verify Current Permissions:
Use ls -la /opt/thp to check the access permissions for the /opt/thp directory (TheHive directory). The output Permissions should show root as the owner. We need to change this.

Change Ownership:
Run the following command to grant ownership of /opt/thp and its contents to the thehive user and group:

chown -R thehive:thehive /opt/thp
The -R flag ensures ownership changes are applied recursively to all files and subdirectories within /opt/thp.

Verify Ownership Change:
Run ls -la /opt/thp again. The output should now show thehive as the owner.

2. Modifying TheHive Configuration File
TheHive's main configuration is stored in application.conf. We need to adjust some settings to match our setup.

Edit the Configuration File:
Open the TheHive configuration file with nano:

nano /etc/thehive/application.conf
Update Hostnames:
Scroll down and locate settings for storage.hostname and index.search.hostname.
Change these values to your TheHive Droplet's public IP address.
Update Base URL:
Scroll down and locate application.baseUrl.
Delete the existing IP address and insert the public IP of your TheHive Droplet, keeping the port http://<your_public_ip>:9000 intact.
Save and Exit `nano`:
Press Ctrl + X to exit.
Type y and press Enter to confirm saving the changes.
3. Starting and Enabling TheHive Service
Now that the configuration is updated, we can start TheHive service.

Start TheHive:
Use systemctl start thehive to start the TheHive service.

systemctl start thehive
Enable TheHive Service:
Use systemctl enable thehive to ensure TheHive starts automatically on system boot.

systemctl enable thehive
Verify Service Status:
Check the TheHive service status with systemctl status thehive.

systemctl status thehive
4. Accessing TheHive
You should now be able to access TheHive by opening your web browser and navigating to your TheHive Droplet's public IP address with port 9000 (e.g., http://<your_public_ip>:9000).

Login: Use the following credentials to log in:

Username: admin@thehive.local
Password: secret
Remember to replace <your_public_ip> with the actual public IP address of your TheHive Droplet.

Configuring Wazuh and Installing the Wazuh Agent
This section details accessing the Wazuh dashboard, retrieving the admin credentials (if needed), and installing the Wazuh agent on the Windows 10 client.

1. Accessing the Wazuh Dashboard
Open a web browser on your Windows 10 virtual machine and navigate to the public IP address of your Wazuh Droplet:

http://<your_wazuh_public_ip>
You will be prompted to log in.

2. Retrieving Wazuh Credentials (If Needed)
If you did not save the Wazuh admin password during the installation process, you can retrieve it by connecting to your Wazuh Droplet via SSH and doing the following:

List the files in the current directory:
ls
This will show you if the wazuh-install-files.tar is present.

Extract the files from the tar archive:
tar -xvf wazuh-install-files.tar
Navigate into the extracted directory:
cd wazuh-install-files/
List the files in the directory:
ls
Display the contents of the password file:
cat wazuh-passwords.txt
This file contains the Wazuh admin password and the Wazuh API user credentials. Copy and save these credentials in a safe place.

3. Installing the Wazuh Agent on Windows
Once you've accessed the Wazuh dashboard, you can install the agent on your Windows 10 client.

Navigate to "Add Agent": In the Wazuh dashboard, click on the "Add agent" button (the location may vary slightly depending on the Wazuh version).
Select Agent Type: Choose "Windows" as the agent type.
Configure Agent Settings:
For the "Server address", enter the public IP address of your Wazuh Droplet.
You can assign a custom name to the agent (optional).
Leave the "Select one or more existing groups" as default.
Copy the Installation Command: The Wazuh dashboard will generate a command for installing the agent. Copy this command.
Open an Elevated PowerShell Window: On your Windows 10 VM, open PowerShell as an administrator (right-click PowerShell and select "Run as administrator").
Paste and Execute the Command: Paste the copied command into the PowerShell window and press Enter to execute it.
Start the Wazuh Service: After the installation completes, start the Wazuh service:
net start Wazuhsvc
Verify Agent Connection: Go back to the Wazuh dashboard and wait a minute. The newly installed agent should appear as "Active".
Configuring Wazuh to Ingest Sysmon Logs on Windows 10
This section details how to configure Wazuh on your Windows 10 client to ingest logs from Sysmon, enabling you to monitor for suspicious activities like mimikatz usage.

1. Backing Up the Configuration File
Before making any changes, it's crucial to back up the original Wazuh agent configuration file.

Locate the Configuration File:
Navigate to the Wazuh agent configuration directory:

C:\Program Files (x86)\ossec-agent
Locate the ossec.conf file.

Create a Backup:
Right-click the ossec.conf file and select "Copy."

Paste the copied file into the same directory and rename it to ossec-backup.conf.

2. Opening the Configuration File (as Administrator)
Open the ossec.conf file with administrative privileges.

Open with Notepad (Admin):
Right-click the ossec.conf file and select "Open with > Notepad."

If prompted for administrator privileges, click "Yes."

3. Understanding Log Analysis Configuration
The "Log Analysis" section in the ossec.conf file defines which logs the Wazuh agent monitors.

4. Adding Sysmon Log Ingestion
We will now add a new <localfile> block to ingest Sysmon logs.

Copy an Existing <localfile> Block:
Locate an existing <localfile> block within the "Log Analysis" section and copy the entire block. Log Analysis

Paste the Block:
Paste the copied block below the original.

Configure Sysmon Log Source:
Change the <location> Tag:
Change the value within the <location> tags to the Sysmon channel name.

Obtain the Sysmon Channel Name:
Open the Event Viewer.
Expand "Applications and Services Logs" -> "Microsoft-Windows" -> "Sysmon."
Right-click "Operational" and select "Properties."
Copy the value under 
Copy the value displayed under "Channel Name" (e.g., "Microsoft-Windows-Sysmon/Operational").
Paste the Channel Name:
Paste the copied channel name between the <location> tags in your ossec.conf file.

Disabling Unnecessary Log Sources (Optional):
To prevent Wazuh from forwarding other logs (e.g., application, security, system), you can comment out the corresponding <localfile> blocks by adding # at the beginning of each line within the block.

5. Saving the Configuration File
Save the File:
Try saving the ossec.conf file in Notepad.

If you encounter permission issues:

Minimize Notepad.
Right-click the ossec.conf file again.
Select "Open with > Notepad (Admin)" to ensure you have administrator privileges.
Repeat the configuration changes and save the file.
6. Restarting the Wazuh Agent Service
Restart the Service:
Open the Services window (search for "Services").

Locate the "Wazuh Agent" service.

Right-click on the service and select "Restart."

7. Verifying Sysmon Log Ingestion
Check the Wazuh Dashboard:
Go to the Wazuh dashboard and navigate to the "Events" section.

Ensure you're viewing the correct index (e.g., "wazuh-alerts*").

Search for "Sysmon" in the search bar. It might take some time for Sysmon events to appear.

8. Downloading Mimikatz (For Testing Purposes Only)
Disclaimer: Downloading and running mimikatz on a production system is highly discouraged. This step is for demonstration purposes only to test your Sysmon configuration.

Disable Windows Defender or Exclude Downloads Folder:
Temporarily disable Windows Defender or create an exclusion for your downloads folder.

Downloading and Running Mimikatz, Configuring Wazuh to Detect It
This section explains how to download and run mimikatz for testing purposes and configure Wazuh to detect its execution.

1. Downloading and Extracting Mimikatz
Search for Mimikatz:
Search online for "mimikatz gentilkiwi."

Find Releases:
On the appropriate website (e.g., GitHub), locate the "Releases" section.

Download Mimikatz:
Download the "mimikatz_trunk" zip file for version 2.2.0 20220919.

Extract the Files:
Extract the downloaded zip file to your Downloads folder (which should already be excluded from Windows Defender scans).

2. Running Mimikatz
Open PowerShell as Administrator:
Open PowerShell with administrative privileges.

Navigate to the Mimikatz Directory:
Use the cd command to navigate to the x64 directory within the extracted mimikatz folder:

cd C:\Users\Win-10-Client\Downloads\mimikatz_trunk\x64
Execute Mimikatz:
Run the mimikatz executable:

.\mimikatz.exe
3. Checking for Events in the Wazuh Dashboard
After running mimikatz, check the Wazuh dashboard for any triggered events.

Search for Mimikatz Events:
In the Wazuh dashboard, search for "mimikatz."

If no events are present, it might be because no default Wazuh rules trigger on the specific Sysmon events generated by mimikatz. The next steps will address this.

4. Configuring Wazuh to Log All Events (for Testing)
To ensure all events are logged (for testing and rule development), you can modify the Wazuh manager's configuration.

Connect to the Wazuh Manager via SSH:
Connect to your Wazuh server via SSH as root.

Backup the Configuration File:
Create a backup of the ossec.conf file:

cp /var/ossec/etc/ossec.conf ~/ossec-backup.conf
Edit the Configuration File:
Open the ossec.conf file with nano:

nano /var/ossec/etc/ossec.conf
Enable Log All:
Change the <logall> and <logall_json> settings to yes:

Nano Ossec Config
<logall>yes</logall>
<logall_json>yes</logall_json>
Save and Exit `nano`:
Save and exit the file (Ctrl + X, y, Enter).

Restart the Wazuh Manager:
Restart the Wazuh manager service:

systemctl restart wazuh-manager.service
5. Configuring Filebeat to Ingest Archives
To ingest the archived logs, you need to configure Filebeat.

Edit the Filebeat Configuration File:
Open the Filebeat configuration file:

nano /etc/filebeat/filebeat.yml
Enable Archives:
Change the archives.enabled setting to true.

Save and Exit `nano`:
Save and exit the file.

Restart Filebeat:
Restart the Filebeat service:

systemctl restart filebeat
6. Creating an Index Pattern for Archives
Create an index pattern in Kibana to search the archived logs.

Navigate to Index Patterns:
In the Wazuh dashboard, go to "Stack Management" -> "Index Patterns."

Index Patterns
Create Index Pattern:
Click "Create index pattern."

Name the Index Pattern:
Name the index pattern wazuh-archives-*.

Configure Time Field:
Select timestamp as the time field and create the index pattern.

7. Creating a Custom Wazuh Rule to Detect Mimikatz
Now, we will create a custom Wazuh rule to specifically detect mimikatz execution.

Navigate to Rules Management:
In the Wazuh dashboard, go to "Management" -> "Rules."

Rules
Manage Rule Files:
Click "Manage rule files."

Edit local_rules.xml:
Click the pencil icon to edit the local_rules.xml file.

Copy and Modify a Sysmon Rule:
Copy an existing Sysmon rule (e.g., one related to Event ID 1) as a template. Copy Rule

Paste the copied rule into local_rules.xml, below the existing rule, and adjust the indentation.

Modify the Rule Details:
Make the following changes to the copied rule:

Change the rule id to 100002 (or another unique ID).
Set the level to 15.
Modify the <field name="win.eventdata.originalFileName" type="pcre2"> to detect mimikatz.exe:
<field name="win.eventdata.originalFileName" type="pcre2">(?i)mimikatz\.exe</field>
Remove the <options>no_full_log</options> tag.
Change the <description> to Mimikatz Usage Detected</description>.
Set the MITRE ATT&CK <id> to T1003 (Credential Dumping).
Save the Rule:
Save the local_rules.xml file. This will automatically restart the Wazuh manager.

8. Triggering and Verifying the Custom Rule
Run Mimikatz Again:
If you didn't see any events, exit the mimikatz prompt in PowerShell and run .\mimikatz.exe again. Mimikatz Exit

Check Wazuh Security Events:
Go back to the Wazuh dashboard and navigate to "Security Events." Refresh the page. You should now see the mimikatz detection event. Mimikatz Detected

Integrating Wazuh with Shuffle.io for Alert Automation
This guide details how to integrate Wazuh with Shuffle.io to automate workflows triggered by Wazuh security alerts.

1. Create a Shuffle.io Workflow
Sign Up for Shuffle.io:
Create an account at https://shuffler.io/.

Create a Workflow:
Go to "Workflows" -> "Create Workflow."

Name your workflow "SOC Automation Project."

Select any two use cases (it doesn't matter for this example).

Click "Create."

Add Webhook Trigger (Shuffle.png):
From the left menu, select the "Webhook Trigger" and drag it next to the "Change Me" icon.

Rename the webhook to "Wazuh-agent."

Copy the Webhook URI displayed (Webhook-URI.png).

Add Repeat Back Action:
Click the "Change Me" icon next to "Find Actions."

Select "Repeat back to me."

In the "Call" field, click the "+" sign and select "Execution Argument."

Save the action.

2. Configure Wazuh Manager for Shuffle Integration
Connect to your Wazuh manager via SSH and configure the integration using the `ossec.conf` file.

Edit `ossec.conf` (nano):
Type `nano /var/ossec/etc/ossec.conf` to open the configuration file.

Add Integration Block:
Scroll to the end of the file, locate the closing `` tag, and add the following integration block, replacing `#THE_URL_WE_COPIED_BEFORE#` with the copied Webhook URI:



  shuffle
  #THE_URL_WE_COPIED_BEFORE#
  100002
  json

Ensure proper indentation to match existing blocks.

Save and Exit `nano`:
Save the changes (Ctrl + X, Y, Enter).

Restart Wazuh Manager:
Restart the Wazuh manager service:

systemctl restart wazuh-manager.service
Verify Service Status:
Check the service status:

systemctl status wazuh-manager.service
3. Triggering the Workflow (Mimikatz Test)
Run Mimikatz on the Client:
On your Windows 10 client, open PowerShell and exit any running mimikatz instance with `exit`.

Run mimikatz again with the command `.\mimikatz.exe`.

Start Shuffle Workflow (Shuffle-start.png):
In your Shuffle.io instance, click "Start" on the created workflow.

Verify Results (results.png):
Click the "Person" button at the bottom. If successful, you should see results from the mimikatz execution.

You can expand the results for more details.

This demonstrates how to integrate Wazuh with Shuffle.io to automate workflows triggered by specific Wazuh rules (e.g., rule ID 100002 in this example).

4. Advanced Automation and Integration
Our Workflow for today:

Mimikatz Alert sent to Shuffle
Shuffle Receives Mimikatz alerts and extracts SHA256 Hash from the file
Check the reputation score with VirusTotal
Send details to thehive to create alerts
Send email to soc analyst to begin investigation.
When we examine the returned hash values, we notice they are appended with their hash type. For example, Hash Example shows a value appended with SHA1= followed by the actual hash. To automate further processing, we need to extract only the hash value itself.

If we don't extract the hash, the entire string (including SHA1=) would be sent to services like VirusTotal, which is not the desired behavior. We only want to send the raw hash value to VirusTotal for analysis.

Let's proceed with the extraction. Close the execution argument and click the "Change Me" icon. In the "Find actions" field, type regex and select regex capture group. For the input data, click the + button, then select "Execution argument" -> "hashes," which will populate the field like this: Input Data.

If you're unfamiliar with writing regular expressions (regex), you can copy the hash value and use an AI solution like ChatGPT to generate the regex code. An example prompt might look like this: ChatGPT Regex Request. Then, paste the generated regex code into the regex section in Shuffle, as shown here: Shuffle Regex Configuration.

Save the workflow. Click the "Person" icon, then click the refresh button or "rerun workflow" next to "details." Expanding the results will show that the SHA256 hash has been correctly parsed. Now we can automate sending this information to VirusTotal for scoring.

Rename the "Change Me" icon to sha256_Regex. Next, we'll use VirusTotal's API to automate hash checks and retrieve results. To use their API, you'll need a VirusTotal account. Sign up on their website, and once you have an account, copy your API key and return to Shuffle.

In Shuffle, click "Apps," search for VirusTotal, and press Enter. Drag and drop the VirusTotal app into your workflow. Rename the action to VirusTotal. In the "Find actions," we don't need to check an IP address; we want to check a hash. Click the dropdown menu; if you only see one option, wait a few minutes and refresh the page. Select Get a hash report. You can either paste your API key directly into the Apikey field or click the orange + Authenticate VirusTotal v3, paste your API key, and click "Submit."

In the "id" field, delete the existing content, click the + button, hover over "regex," select "group," and click. Save the workflow again. Click the "Person" icon, select the middle workflow, and rerun it. Let's examine the VirusTotal output. Expanding the results, we see a status of 200. Expanding further (0: -> body -> data -> attributes -> last_analysis_stats), we see malicious 64, indicating that 64 scanners detected the file as malicious Malicious VirusTotal Result.

To recap: We configured our SOAR platform to receive Wazuh alerts, used regex to extract the SHA256 hash, and used VirusTotal to check its reputation. Next, we'll send the details to thehive to create an alert for case management. In the application tab (bottom left), search for thehive, click it, and drag and drop it into your workflow. Access thehive at http://*thehiveip*:9000 with the credentials admin@thehive.local and password secret.

Creating a New Organization and Users in thehive
This guide walks you through creating a new organization and users in thehive:

Creating a New Organization
Click the plus button in the top corner.
Enter your desired organization name (e.g., "Krysta") in the Name field.
Add a description (e.g., "SOC Automation Project") in the Description field.
Click Confirm to create the organization.
Adding Users
Click the new organization name to access it.
Click the plus button New Users Button to add users.
For a Regular User (Krysta):
Leave Type as Normal.
Enter the user's login credentials:
Login: krysta@test.com
Name: Krysta
Profile: Analyst
Click Confirm to create the user.
For a Service User (SOAR):
Select Service for the Type.
Enter the service account details:
Login: shuffle@test.com
Name: SOAR
Profile: Analyst (Ideally a custom profile with least privilege)
Click Confirm to create the user.
Setting Up User Access
For Krysta:
Hover over Krysta's username and select Preview.
Scroll down and click Set a New Password.
Create a password and click Confirm.
Click outside the window.
For SOAR User (API Key):
Select preview for the SOAR user.
Click Create API key.
Copy the generated API key and store it securely.
Logging In and Configuring thehive Integration in Shuffle
Log out of the admin account.
Log in with Krysta's credentials (krysta@test.com and password).
Authenticate thehive in Shuffle:
Click the plus button beside Authenticate.
Paste your thehive API key into the designated field.
Enter thehive's public IP address and port number in the URL section.
Click Submit.
Connect thehive to Your Workflow:
In the workflow editor, click on the Virustotal icon.
Drag the blue dot on the Virustotal icon and connect it to the thehive icon.
Configure thehive Alert Creation:
Configure thehive Alert Creation:
Click on the date section (if available). If not, proceed to step 6.
Click the plus button and select Execution argument -> utc time.
Configure the remaining fields for thehive alert creation according to the provided details in the image  
You can use the provided JSON code in the advanced tab (body section) if you don't have the individual fields (
If you don't have the individual date section and other fields, you can use the provided JSON code in the advanced tab (body section):

{
  description": "Mimikatz detected on Computer: $exec.text.win.system.computer from User: $exec.text.win.eventdata.user",
  "externallink": "${externallink}",


  "flag": false,


  "pap": 2,


  "time": "$exec.text.win.eventdata.utcTime",


  "severity": "2",


  "source": "Wazuh",


  "sourceRef": "Rule: 100002",


  "status": "New",


  "summary": "Mimikatz detected on Computer: $exec.text.win.system.computer, ProcessID: $exec.text.win.system.processID and CommandLine: $exec.text.win.eventdata.commandLine",


  "tags": [


    "T1003"


  ],


  "title": "$exec.title",


  "tlp": 2,


  "type": "Internal"


} 

Save the Workflow.
Modifying the Cloud Firewall (Temporary for Testing)
Before running the workflow, we need to temporarily modify the cloud firewall to allow inbound traffic on port 9000, where our thehive instance resides. This rule will be removed after testing.

Navigate to your cloud provider's firewall settings (e.g., DigitalOcean):
In DigitalOcean, go to Networking -> Firewalls.
Select the relevant firewall.
Create a new firewall rule:
Set the rule type to Custom.
Specify port 9000.
Configure allowed IP addresses:
Remove all IPv6 addresses.
Keep all IPv4 addresses (allowing access from any source).
Save the firewall rule. This will temporarily allow any source to access your machine on port 9000.
Verifying the Integration
Returning to Shuffle, we can now rerun the workflow. Click the "Person" icon and then "Rerun Workflow."

Scrolling down in the workflow execution details, we should see confirmation that thehive integration was successful. Let's verify this in our thehive instance.

Access your thehive instance. You should see that an alert has been automatically created. Refreshing the thehive instance will display the new alert. Opening the alert provides more details about the event Successful thehive Alert Creation.

Adding more information to the "Summary" field in the workflow configuration will provide even more detailed context within thehive alert, further streamlining investigations.

Sending an Email Notification
The next step is to send an email notification to our analyst using Shuffle containing relevant information. To achieve this:

Click on "Apps" in the bottom left corner.
Drag and drop the "Email" application into the workflow.
Connect the VirusTotal action to the Email action.
Configure the email settings:
Enter the recipient's email address (any valid email address can be used).
Set the subject to "Mimikatz detected!".
Configure the email body:
"Time: " Execution argument -> utcTime
"Title:" Execution argument -> title
"Host:" Execution argument -> computer
Save the workflow and rerun it.

Success! The email notification should be received, completing the home lab setup.

Lab Details
This lab focused on building a basic Security Operations Center (SOC) environment using several key tools:

Sysmon: Deployed Sysmon on a Windows endpoint to capture detailed system activity, providing enhanced visibility into endpoint events.
Wazuh: Configured Wazuh as a Security Information and Event Management (SIEM) system to collect, analyze, and correlate security events from the Sysmon agent. This included configuring Wazuh to ingest Sysmon logs and creating a custom rule to detect mimikatz activity.
TheHive: Set up TheHive as a case management platform to track and manage security incidents generated by Wazuh alerts.
Mimikatz (Testing): Used mimikatz in a controlled environment to generate test events and validate the effectiveness of the custom Wazuh rule.
Key skills and concepts covered in this lab include:

Endpoint monitoring with Sysmon.
Log collection and analysis with Wazuh.
Rule creation and customization in Wazuh.
Incident management with TheHive.
Understanding of common attack techniques (e.g., credential dumping with mimikatz).
Integration of different security tools to create a more comprehensive security posture.
By completing this lab, I gained practical experience in setting up and configuring a basic SOC environment, improving my understanding of endpoint security, log analysis, and incident response workflows.