# Log Analysis – Compromised WordPress

Lab link:  [BTLO](https://blueteamlabs.online/home/challenge/log-analysis-compromised-wordpress-ce000f5b59)

### Scenario
One of our WordPress sites has been compromised but we're currently unsure how. The primary hypothesis is that an installed plugin was vulnerable to a remote code execution vulnerability which gave an attacker access to the underlying operating system of the server.

There was a WordPress site compromise, and we were given logs to find out how it happened.

### Step1

Download the zip file, extract it and then open it with a text editor. 

![image.png](/assets/Img/03/image.png)

The only other information we have is a hypothesis; let’s take note of that.

### Step2

My first step would be to look for a foothold into the attack. I search for popular tools used in this context and see whether I can get something to start with. Search for the string “wpsscan”; WPScan is a popular WordPress scanner.
Note: The default configurations of tools allow them to supply their names and version as user-agent header values, so these can be used to identify these tools

![image.png](/assets/Img/03/image%201.png)

Line 1364 shows 119.241.22.121 sending a GET request with a user-agent header of “WPScan v3.8.10”. This suggests that the IP of 1192.241.22.121 is using WPScan version 3.8.10 to scan the WordPress site, trying to discover vulnerabilities.

Let’s make a list of artefacts and collect this IP as an artefact.
Naming convention: <artefact acronym>(<artefact number>) i.e. A(1)

Artefacts

A(1) - `119.241.22.121`

### Step3

A(1)’s Time of activity: 14/Jan/2021, 05:42:34 - 06:26:53.
Using artefact A(1) as a search string to see the IP’s activity. The IP seems to be conducting a directory brute force, and it seems to be automated given how fast it’s going — you can tell by looking at the time.

![image.png](/assets/Img/03/image%202.png)

While investigating A(1), I found another IP sending unusual strings in a POST request.

![image.png](/assets/Img/03/image%203.png)

The strings in question look like SQLI strings

Let’s add this to the list of artefacts.

Artefacts

A(1) - `119.241.22.121`
A(2):`168.22.54.119`

Something of note — that aligns with the scenario’s hypothesis — involves a plugin, simple-file-list. 
On line 1714, A(1) made a POST request to an endpoint “/wp-content/plugins/simple-file-list/ee-upload-engine.php “ — based on the naming convention, this suggests a file was uploaded and then followed by a GET request on line 1715, requesting to access a png file, fr34k.png, I would assume this was the uploaded file. 

On line 1716, the POST request was to another endpoint, “/wp-content/plugins/simple-file-list/ee-file-engine.php” with referrer, “/wp-admin/admin.php?page=ee-simple-file-list&tab=file_list&eeListID=1” — the naming conventions suggest that uploaded files were viewed, then a file operation was performed on a file.

This is similar to a popular technique used by hackers; they first upload a malicious file and then execute the file to gain a shell. In most cases, they’re extension restrictions, so they use an acceptable extension and then try to change it back to the original later on

![image.png](/assets/Img/03/image%204.png)

Let’s collect artefacts from the above

**Artefacts**

A(1) - `119.241.22.121`
A(2):`168.22.54.119`
A(3): Plugin name: simple-file-list; Upload Post Path: /wp-content/plugins/simple-file-list/ee-upload-engine.php; Upload GET Path: /wp-content/uploads/simple-file-list; Uploaded File: fr34k.png; 2nd Post Path: /wp-content/plugins/simple-file-list/ee-file-engine.php; 2nd Referrer Route: /wp-admin/admin.php?page=ee-simple-file-list&tab=file_list&eeListID=1

### Step4

Another thing to reduce the likelihood of missing something is to filter the logs further strategically. Using A(1), I would filter to get only the status codes and user-agent fields.

Starting with the status codes, I want to get requests with a 200 status code so we can see what requests were successful and check whether we missed anything of note.

![image.png](/assets/Img/03/image%205.png)

*The command above uses cat to output the contents of the access.log file, then I use the pipe, “|” to feed the output into grep where I grep for the IP. I pipe the output again into cut where I use space, “ ” as a delimiter and grab the 6th to 9th fields, and then I pipe the contents one last time to only get the lines with a 200 status code.*

We could note the plugins being successfully enumerated by the attacker and Google to see what vulnerabilities their version has

contact-form-7

![image.png](/assets/Img/03/image%206.png)

simple-file-list

![image.png](/assets/Img/03/image%207.png)

According to my search, the simple-file-list plugin is most likely version 4.2.2. The exploit above also seems to be similar to the one used by A(1)

Note: Based on the exploit above, we were correct in our assumption that the file was renamed on line 1716

| Plugin name | Version as suggested by logs | Version as suggested by internet search | Vulnerability |
| --- | --- | --- | --- |
| contact- form-7 | 5.3.1 |  | Arbitrary file upload |
| simple-file-list | 5 | 4.2.2 | Arbitrary ***File*** Upload |
| better-wp-security |  |  | the logs don’t suggest the version was successfully enumerated |

Another thing to note is the successful POST request to /wp-login.php?itsec-hb-token=adminlogin
This suggests a successful login as administrator

![image.png](/assets/Img/03/image%208.png)

Let’s look at the user-agents used by the A(1)

![image.png](/assets/Img/03/image%209.png)

The command above does the same as the previously explained one, the only difference is that this one selects the fields from 12 to the end of the line.

We don't see where the renamed file was being used. 
Based on this function in the exploit code, we found

![image.png](/assets/Img/03/image%2010.png)

The file retains its original name; it only changes its extension, so we should be looking for “fr34k.php”

**Artefacts**

A(1) - `119.241.22.121`
A(2):`168.22.54.119`
A(3): Plugin name: simple-file-list; Upload Post Path: /wp-content/plugins/simple-file-list/ee-upload-engine.php; Upload GET Path: /wp-content/uploads/simple-file-list; Uploaded File: fr34k.png; 2nd Post Path: /wp-content/plugins/simple-file-list/ee-file-engine.php; 2nd Referrer Route: /wp-admin/admin.php?page=ee-simple-file-list&tab=file_list&eeListID=1; Renamed-to: fr34k.php

Let’s keep investigating.

### Step5

Continue the investigation by using artefact A(2): 168.22.54.119

The IP was found sending unusual strings in a POST request

Time of activity: 14/Jan/2021, between 06:03:41 and 06:14:03

![image.png](/assets/Img/03/image%2011.png)

Scrolling through to get a high-level overview of what’s going on, you could notice automated requests using different methods: POST, GET, HEAD
You’ll also notice SQLI strings

![image.png](/assets/Img/03/image%2012.png)

We could look more strategically. Since we know that some tools leave footprints in their user-agent fields, we could filter on that and get unique user-agents

![image.png](/assets/Img/03/image%2013.png)

*The command above greps for the IP in question, then pipes the result to cut where we grab specific fields using space as our delimiter, then we pipe the results to sort to arrange the lines alphabetically, finally, we pipe the results to uniq to remove duplicate lines.
Note: sort was used before uniq in the commands below because uniq only removes duplicates that are together.*

Something of note is sqlmap. SQL map is a popular tool for SQLI. 

We could also look at successful requests.

![image.png](/assets/Img/03/image%2014.png)

There are successful POST requests to /wp-login.php?itsec-hb-token=adminlogin which suggest a login as an admin.

![image.png](/assets/Img/03/image%2015.png)

![image.png](/assets/Img/03/image%2016.png)

A successful admin login with SQLI string embedded in it.
We could look at what the embedded SQLI string is using cyberchef

![image.png](/assets/Img/03/image%2017.png)

An attempt at RCE, which we’re not sure was successful.

### Step6

Lets take a look at A(03)
A(3): `Plugin name: simple-file-list; Upload Post Path: /wp-content/plugins/simple-file-list/ee-upload-engine.php; Upload GET Path: /wp-content/uploads/simple-file-list; Uploaded File: fr34k.png; 2nd Post Path: /wp-content/plugins/simple-file-list/ee-file-engine.php; 2nd Referrer Route: /wp-admin/admin.php?page=ee-simple-file-list&tab=file_list&eeListID=1`

We can start by searching for the plugin name. Let’s see what activity surrounds it
Let’s grep to see what IPs are related to it

![image.png](/assets/Img/03/image%2018.png)

Then we can look through the full log to understand the different activity

![image.png](/assets/Img/03/image%2019.png)

172.21.0.1 seem to have activated the plugin on line 165.

![image.png](/assets/Img/03/image%2020.png)

on the same line, the referrer suggest that the plugin was also uploaded by the same IP.  We can assume that this is the administrator of the site.

103.212.94.19’s activity suggests probing due to the presence of multiple 404s.
Time of activity: 14/Jan/2021 06:08:31- 06:17:46

![image.png](/assets/Img/03/image%2021.png)

Scrolling through we also notice the activity of A(1) and 103.69.55.212
103.69.55.212 Time of activity: 14/Jan/2021, 06:19:04 - 06:30:11

![image.png](/assets/Img/03/image%2022.png)

In the screenshot, we already noted the activity of A(1) regarding this plugin.
On line 1716, the file fr34k.png was renamed to fr34k.php and then accessed by the IP 103.69.55.212

Let’s add to our list of artefacts

**Artefacts**

A(1) - `119.241.22.121`
A(2):`168.22.54.119`
A(3): `Plugin name: simple-file-list; Upload Post Path: /wp-content/plugins/simple-file-list/ee-upload-engine.php; Upload GET Path: /wp-content/uploads/simple-file-list; Uploaded File: fr34k.png; 2nd Post Path: /wp-content/plugins/simple-file-list/ee-file-engine.php; 2nd Referrer Route: /wp-admin/admin.php?page=ee-simple-file-list&tab=file_list&eeListID=1; Renamed-to: fr34k.php`
A(4): `103.69.55.212` 

Subsequently, various POST and GET requests are sent to the fr34k.php.

![image.png](/assets/Img/03/image%2023.png)

We could conduct a search for the fr34k.php file to see if there are other activities surrounding it
I found none. It’s safe to assume the remote code execution happened from line 1723 - line 1735 by A(4) using “fr34k.php”

Before line 1736, the file in question was discovered by security and removed, this resulted in a 404.

![image.png](/assets/Img/03/image%2024.png)