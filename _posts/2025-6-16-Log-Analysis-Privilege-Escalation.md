# Log Analysis - Privilege Escalation

Lab link:  [BTLO](https://blueteamlabs.online/home/challenge/log-analysis-privilege-escalation-65ffe8df12)

- Challenges
    
    Having to infer without a full view of what happened
    
- Pros
    
    I had fun mapping out the attackers moves with the limited information I had
    

Scenario

A server with sensitive data was accessed by an attacker and the files were posted on an underground forum. This data was only available to a privileged user, in this case the ‘root’ account. Responders say ‘www-data’ would be the logged in user if the server was remotely accessed, and this user doesn’t have access to the data. The developer stated that the server is hosting a PHP-based website and that proper filtering is in place to prevent php file uploads to gain malicious code execution. The bash history is provided to you but the recorded commands don’t appear to be related to the attack. Can you find what happened?

### Step1

Unzip and open the bash_history file in a text editor. I will be using Sublime Text.

![image.png](/assets/Img/05/image.png)

Step2 

Let's try to make sense of the bash history.

![image.png](/assets/Img/05/image%201.png)

I suspect that the hacker found an upload vulnerability; therefore, on line 10, he navigated to the uploads directory and performed his attack, which I also suspect was successful due to the reconnaissance activity that followed on line 11

![image.png](/assets/Img/05/image%202.png)

Note: The absence of the command running the Linux exploit suggester that was seemingly downloaded on line 32 on the log suggests that the download of the linux exploit suggester failed.

![image.png](/assets/Img/05/4559dc8a-8c2b-4586-a93e-7a1f9637039a.png)

What do we know?

- The attacker used x.phtml to exploit an upload vulnerability.
- They tried downloading the Linux exploit suggester, which failed.
- There was a ton of reconnaissance activity.
- Eventually the attacker searched for files with SUID bit set and found a python binary
- The attacker eventually escalated to root using the python binary with SUID bit set.
- The attacker removed the x.phtml file

### Solutions

![image.png](/assets/Img/05/image%203.png)