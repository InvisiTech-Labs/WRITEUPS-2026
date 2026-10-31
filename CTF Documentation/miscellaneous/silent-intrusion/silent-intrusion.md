# Silent Intrusion Challenge
A public Ubuntu server shows a burst of SSH failures against user `devops` followed by a successful login. Soon after, a hidden binary appears and a new systemd service/timer begins executing it on a schedule. Your job is to confirm the intrusion timeline and persistence using the provided artifacts.

File Location: ChallengeFile\uac-num1-linux-20260130195304.tar.gz

## Questions
#### 1. 
What is the first timestamp of failed SSH attempts in the suspicious window?
Answer Format: YYYY-MM-DD HH:MM:SS UTC
![alt text](image-1.png)
 
Hint:auth.log, first "Failed password" after scope start. You can use grep-awk commands on powershell.
#### 2. 
What is the source IP of that first failed attempt?
![alt text](image-2.png)
 
Hint: Same line as Q1, look after "from"
#### 3.
How many failed login attempts occur before the first successful login?
 ![alt text](image-3.png)
#### 4.
What is the name of the unit file that defines the persistence service?
 ![alt text](image-4.png)
Hint: /etc/systemd/system

#### 5. 

What binary is executed by the persistence mechanism?
 ![alt text](image-5.png)
Hint: ExecStart line in systemd service file

## FLAGS
1.	flag{inm_2026-01-30 19:34:45 UTC}
2.	flag{inm_146.190.237.83}
3.	flag{inm_123}
4.	flag{inm_system-update.service}
5.	flag{inm_/usr/local/lib/sys/.upd}

## Walkthrough
#### 1.
Open the var/log/auth.log file and check for the first failed login attempt for the user devops
![alt text](image-6.png)
 
#### 2. 
Same line as above check the IP address 
#### 3. 
Search for "Failed password for devops from 146.190.237.83" and count. You can utilize notepad++ for this
 ![alt text](image-7.png)
Since the question specifically asked for the count before the first successful login, the answer becomes 123 since the other one happened after the first successful login.
#### 4.
Attackers need to establish persistence or "backdoor" after gaining access to ensure they can access the said system whenever they want even when the machine reboots. 
For this, one needs to check the service files under \[root]\etc\systemd\system. In this case, the persistence mechanism uses a systemd timer to schedule recurring execution of a service. The timer configuration references system-update.service through the Unit= directive. This service unit defines the actual command or process to be executed, while system-update.timer controls when it runs.
 ![alt text](image-8.png)
OnBootSec=5min schedules the service to run 5 minutes after the system boots.
OnUnitActiveSec=15min schedules it to run again 15 minutes after the service was last activated.
#### 5.
Now that we know the unit file, the next step is to check the contents of that service file to understand what it does. 
 ![alt text](image-9.png)
The system-update.service unit file specifies the binary to be executed through the ExecStart directive under the [Service] section. The configured executable is /usr/local/lib/sys/.upd, which is launched whenever the service is triggered.
Therefore, the answer is /usr/local/lib/sys/.upd
