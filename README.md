# Dancing-HTB
Dancing is a very easy Windows machine which introduces the Server Message Block (SMB) protocol, its enumeration and its exploitation when misconfigured to allow access without a password.

what is SMB(servewr message block) ?
Server Message Block (SMB) is a communication protocol[1] used to share files, printers, serial ports, and miscellaneous communications between nodes on a network. On Windows, the SMB implementation consists of two vaguely named Windows services: "Server" (ID: LanmanServer) and "Workstation" (ID: LanmanWorkstation).[2] It uses NTLM or Kerberos protocols for user authentication. It also provides an authenticated inter-process communication (IPC) mechanism. 

Functionality: SMB is a client-server, request-response protocol. A client sends a request to a server, which processes it and sends a response.
Use Cases: It is widely used in corporate networks for centralized file access, mapping network drives, printer sharing, and inter-process communication.
Operating Systems: deeply integrated into Windows
Connectivity: It operates over TCP/IP, usually on port 445, but can also use QUIC for secure internet access without a VPN.
<img width="800" height="438" alt="image" src="https://github.com/user-attachments/assets/11b542bf-c378-4110-9c91-bdc7a5aa0623" />

How does the SMB protocol work?
The client sends an SMB request to the server to initiate the connection. When the server receives the request, it sends an SMB response back to the client, establishing the communication channel necessary for a two-way conversation. Once it is granted access, the client can access the required resource for reading, writing, executing and so on. Since the network server has a resource that it shares with one or more clients, the protocol is also known as a server-client protocol.
The SMB protocol operates at the application layer but relies on lower network levels for transport. At one time, SMB ran on top of Network BIOS over TCP/IP or, to a lesser degree, legacy protocols, such as Internetwork Packet Exchange or NetBIOS Extended User Interface.
When SMB was using NBT, it relied on ports 137, 138 and 139 for transport. Now, SMB runs directly over TCP/IP and uses port 445. Port 445 supports data encryption and digital signing of SMB packets, providing a more secure means of communication than port 139.

**#Write-up: HTB Fawn - FTP Exploitation**
Target Information

Machine Name: Fawn (HTB Starting Point) 
IP Address: 10.129.26.231
Difficulty: Very Easy 
Goal: identification and exploitation of an incorrectly configured Server Message Block (SMB) service on a Windows machine
--By leveraging a Null Session, I was able to enumerate network shares, access sensitive user directories, and exfiltrate credentials and flag data

 **Phase 1: Reconnaissance**
An initial Nmap scan was performed to identify open ports and services.
  $ nmap -sC -sV 10.129.26.231
       _ Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-01 08:21 -0500
Stats: 0:00:19 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 97.30% done; ETC: 08:21 (0:00:00 remaining)
Nmap scan report for 10.129.26.231
Host is up (0.63s latency).
Not shown: 996 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 3h59m54s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-03-01T17:22:23
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 89.94 seconds_

Key Findings:

   Port 139/445 (SMB): Open. Version 3.1.1. Message signing is enabled but not required.
   Port 5985 (WinRM): Open. Indicates the machine is set up for remote management.
   OS: Windows.

   **Phase 2: Enumeration**
I attempted to list the available SMB shares using a Null Session (no password).
  $ smbclient -L //10.129.26.231/ -N 
   Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        WorkShares      Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.26.231 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available

Discovered Shares:

   ADMIN$, C$, IPC$ (Standard Windows shares)
   WorkShares (Custom Share - High Interest)

Phase 3: Exploitation & Exfiltration
I connected to the WorkShares directory to browse the filesystem.
 $ smbclient //10.129.26.231/WorkShares -N
 
  Internal Structure:
  Amy.J/
  James.P/
Inside James.P, I located and downloaded the machine flag. Inside Amy.J, I found a file named worknotes.txt containing administrative "to-do" items regarding Apache and WinRM setup.

Extracted Flag:
5f61c10dffbc77a704d76016a22f1664

**Remediation Recommendations**
1. Disable Guest Access: The WorkShares directory should not allow Null Sessions or Anonymous logins.
2. Enforce SMB Signing: To prevent SMB Relay attacks, set "Digitally sign communications" to Required.
3. Principle of Least Privilege: User directories (Amy/James) should have restricted NTFS permissions so they are not viewable by unauthorized users.

Lessons Learned

- Defaults are Dangerous: Many services come with "Guest" or "Anonymous" access enabled by default. In this case, an administrator created a share (WorkShares) but didn't restrict who could see it, allowing any unauthorized user on the network to browse its contents.

- Information Leakage leads to Escalation: Even a simple text file like worknotes.txt can be a security risk. By reading it, an attacker gains insight into the environment (usernames, machine names, and active projects like WinRM), which helps them plan a more targeted attack.

- SMB Signing is a Critical Guardrail: When SMB signing is "enabled but not required," it leaves the door cracked open for Man-in-the-Middle (MITM) and Relay attacks.

  **Remediation (How to Fix It)**
1. Disable Anonymous/Null Sessions

To prevent users from listing shares without a password, you should modify the Windows Security Policy:

  - Path: Computer Configuration > Windows Settings > Security Settings > Local Policies > Security Options

  - Action: Set "Network access: Do not allow anonymous enumeration of SAM accounts and shares" to Enabled.

   - Action: Set "Network access: Restrict anonymous access to Named Pipes and Shares" to Enabled.

2. Enforce SMB Signing

This prevents attackers from "intercepting" a login and relaying it to another machine.

  - Path: Same as above.

  - Action: Set "Microsoft network server: Digitally sign communications (always)" to Enabled.

3. Apply the Principle of Least Privilege

    Action: Audit the WorkShares folder permissions. Ensure that only specific users (like Amy.J or James.P) have access. Remove "Everyone," "Guest," and "Anonymous Logon" from the Access Control List (ACL).

4. Disable Legacy Protocols

    Action: Ensure SMBv1 is completely disabled via PowerShell to prevent ancient exploits like EternalBlue:
         Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol

  ** Tools Used**

  1. Nmap: Service discovery and script scanning.
  2. smbclient: SMB share enumeration and file transfer.
  3. Linux/Bash: Local file analysis

    .
