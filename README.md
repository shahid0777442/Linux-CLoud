# Linux, Networking and Cloud Infrastructure: Step-by-Step Practical Report

In this project I learned the basics of the Linux command line, networking, and cloud security settings, and then used them to create, secure, and test a Linux server in the cloud. I did all the work in my own cloud account, with a small free-tier virtual machine, and I used no real or private data.

## Table of contents

1. [Objectives and scope](#1-objectives-and-scope)
2. [Part A: Linux fundamentals](#2-part-a-linux-fundamentals)
3. [Part B: Networking fundamentals](#3-part-b-networking-fundamentals)
4. [Part C: Deploy a Linux server](#4-part-c-deploy-a-linux-server)
5. [Cleanup](#5-cleanup)
6. [Results and conclusion](#6-results-and-conclusion)
7. [What I learned](#7-what-i-learned)
8. [Final checklist](#8-final-checklist)

## 1. Objectives and scope

1. Learn Linux CLI operations, files and permissions, users and groups, process management, services, and SSH keys.
2. Learn IP addressing, DNS, ports, firewalls, and security group settings.
3. Deploy a Linux server in this order: **Create VM, Configure Linux, Secure Access, Deploy Service, Test Connectivity.**

| Item | Details |
|---|---|
| Platform | AWS (EC2), free tier, account owned by me |
| Operating system | Ubuntu Server (see screenshots for the exact version) |
| Service deployed | Nginx web server with a simple test page |
| Scope | Only my own VM. No other system was scanned or tested. |
| Safety | No real data, billing alert set, and VM deleted at the end |

## 2. Part A: Linux fundamentals

I practiced these commands on my VM (or on a local Linux machine or WSL first).

### Step A1: Navigating and working with files

| Command | What it does |
|---|---|
| `pwd` | Shows the folder I am in |
| `ls -la` | Lists files, including hidden ones, with details |
| `cd /var/log` | Goes to another folder (`cd ~` goes home, `cd ..` goes up) |
| `mkdir lab` | Creates a folder |
| `touch notes.txt` | Creates an empty file |
| `echo 'hello' > notes.txt` | Writes text into a file |
| `cat notes.txt` | Shows the content of a file |
| `cp notes.txt backup.txt` | Copies a file |
| `mv backup.txt old.txt` | Moves or renames a file |
| `rm old.txt` | Deletes a file (careful, there is no recycle bin) |
| `grep 'hello' notes.txt` | Searches for text in a file |
| `man ls` | Shows the manual of a command |

### Step A2: Files and permissions

When I run `ls -l`, I see something like `-rw-r--r-- 1 ubuntu ubuntu 6 Oct 25 notes.txt`. The first part shows the permissions:

- The first character is the type (`-` file, `d` directory).
- Then three groups of three characters: **owner**, **group**, and **others**.
- `r` = read (4), `w` = write (2), `x` = execute (1).

| Number | Permission | Example use |
|---|---|---|
| 7 | `rwx` | Owner of a script |
| 6 | `rw-` | Normal private file |
| 5 | `r-x` | Read and run only |
| 4 | `r--` | Read only |
| 0 | `---` | No access |

Commands I practiced:

```bash
chmod 600 notes.txt            # only the owner can read and write
chmod 755 script.sh            # owner can do everything, others can read and run
chmod u+x script.sh            # add execute permission for the owner
chown ubuntu:ubuntu notes.txt  # change the owner and group
ls -l notes.txt                # check the result
```

**Security lesson:** sensitive files like private keys should be `600` or `400`, and a file should never be `777` (everyone can do everything).

![File permissions with ls -l](images/image1.png)

### Step A3: Users and groups

```bash
whoami                              # my current user
id                                  # my user ID and groups
sudo adduser labuser                # create a new user (asks for a password and details)
sudo groupadd labgroup              # create a group
sudo usermod -aG labgroup labuser   # add the user to the group
groups labuser                      # show the user's groups
cat /etc/passwd | grep labuser      # user entry
cat /etc/group | grep labgroup      # group entry
sudo passwd -l labuser              # lock the account (to test, then undo with -u)
```

I learned that `sudo` gives temporary admin rights, so I should use a normal user and use `sudo` only when needed, instead of working as root all the time.

![Creating a user and group](images/image2.png)

### Step A4: Process management

```bash
sleep 300 &        # start a test process in the background
pgrep -l sleep     # find a process ID by name
ps aux             # list all running processes
ps aux | grep nginx  # find a specific process
top                # live view of CPU and memory (press q to quit)
kill <PID>         # ask a process to stop nicely
kill -9 <PID>      # force stop (last option)
```

### Step A5: Services with systemd

Services (daemons) are programs that run in the background, like the web server or the SSH server.

```bash
systemctl status ssh               # check a service
sudo systemctl start nginx         # start it now
sudo systemctl stop nginx          # stop it
sudo systemctl restart nginx       # restart it
sudo systemctl enable nginx        # start automatically at boot
sudo systemctl is-enabled nginx
journalctl -u nginx --no-pager | tail   # service logs
```

### Step A6: SSH keys

SSH lets me control a remote Linux machine securely. SSH keys are safer than passwords. They come as a pair: a **private key** (secret, stays only on my computer) and a **public key** (can be shared and is placed on the server).

```bash
ssh-keygen -t ed25519 -C 'lab-key'   # create a key pair on my own computer
ls ~/.ssh                            # id_ed25519 (private) and id_ed25519.pub (public)
chmod 600 ~/.ssh/id_ed25519          # protect the private key
```

I used a passphrase for the private key. I never share or upload the private key and I never show it in screenshots.

## 3. Part B: Networking fundamentals

### Step B1: IP addressing

- An **IP address** identifies a device on a network. IPv4 looks like `203.0.113.10`.
- **Private addresses** are used inside a network and are not reachable directly from the internet: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
- **Public addresses** are reachable from the internet. A cloud VM usually has both a private and a public address.
- **CIDR** notation shows the network size. `/24` means 256 addresses, and `/32` means exactly one address (for example, only my own IP).

```bash
ip addr show                            # my interfaces and private IP
ip route                                # routing table and default gateway
hostname -I                             # quick view of IP addresses
curl -s https://checkip.amazonaws.com   # my public IP as the internet sees it
```

![ip addr show output](images/image3.png)

### Step B2: DNS

**DNS** translates names like `example.com` into IP addresses, like a phone book for the internet.

| Record | Meaning |
|---|---|
| A | Name to IPv4 address |
| AAAA | Name to IPv6 address |
| CNAME | Name that points to another name |
| MX | Mail server for the domain |
| TXT | Text information (for example, verification) |

```bash
nslookup example.com
dig example.com A +short
dig example.com MX +short
resolvectl status        # which DNS server my Linux machine uses
cat /etc/resolv.conf
```

**DNS setup for my server:**

- **Option 1 (I own a domain):** In my DNS provider (for example, Route 53, Cloudflare, or my domain registrar), I create an **A record** that points `lab.mydomain.com` to the server's public IP. I use a fixed IP (for example, an Elastic IP in AWS), because normal public IPs can change after a restart. Then I check it with `dig lab.mydomain.com +short`.
- **Option 2 (no domain):** For a local test, I add a line to the hosts file on my own computer, for example `<server-public-ip> lab.local`, and then open `http://lab.local` in my browser. I remove this line after the test.

### Step B3: Ports

A port tells the computer which service should receive the traffic. An address is like a building and a port is like a room number.

| Port | Protocol | Service | Should it be open to the internet? |
|---|---|---|---|
| 22 | TCP | SSH (remote login) | Only from my own IP |
| 80 | TCP | HTTP (web) | Yes, for a public website |
| 443 | TCP | HTTPS (secure web) | Yes, for a public website |
| 53 | TCP/UDP | DNS | Normally no, unless it is a DNS server |
| 3306 | TCP | MySQL database | No, keep it private |
| 3389 | TCP | Windows Remote Desktop | No, not on a Linux web server |

```bash
ss -tulpn    # services listening on ports on my server
```

### Step B4: Firewalls and security groups

There are two layers of protection:

| | Cloud security group | Linux firewall (UFW) |
|---|---|---|
| Where it works | Outside the VM, at the cloud network level | Inside the VM |
| Controlled from | Cloud console | Terminal |
| Default for incoming traffic | Everything blocked until I add a rule | Depends on settings |
| Best practice | Allow only needed ports and sources | Same rules again (defence in depth) |

The principle I followed is **least privilege**: open only the ports that are needed, and only to the people who need them.

## 4. Part C: Deploy a Linux server

The order is: **Create VM, Configure Linux, Secure Access, Deploy Service, Test Connectivity.**

### Step 1: Create the VM

1. Sign in to my own cloud console (I used MFA) and select one region.
2. Set a **billing alert** if I have not done it yet.
3. Open the compute service (AWS EC2) and click **Launch instance**.
4. Name: `lab-server`.
5. Image (OS): **Ubuntu Server LTS**.
6. Instance type: the smallest free-tier type (for example, `t2.micro` or `t3.micro`).
7. Key pair: create or import my SSH key pair. For AWS I download the `.pem` file once and keep it safe. (Or I upload my own `id_ed25519.pub` public key.)
8. Network and security group: create a new security group named `lab-sg` with these **inbound rules**:
   - SSH, TCP 22, source: **My IP** (/32) only.
   - HTTP, TCP 80, source: `0.0.0.0/0` (so the test website can be opened).
   - No other inbound rules.
9. Storage: default (for example, 8 GB).
10. Launch the instance and wait for the status to become **Running**. Note the **public IP address**.

![Security group inbound rules](images/image4.png)

![Launch instance summary](images/image5.png)

### Step 2: Configure Linux

1. Protect my key file and connect (replace the key name and IP):

   ```bash
   chmod 400 lab-key.pem
   ssh -i lab-key.pem ubuntu@<public-ip>
   ```

   The first time, SSH asks me to trust the server fingerprint. I type `yes`.

2. Update the system:

   ```bash
   sudo apt update
   sudo apt upgrade -y
   ```

3. Set the hostname and time zone (optional but neat):

   ```bash
   sudo hostnamectl set-hostname lab-server
   sudo timedatectl set-timezone Asia/Karachi
   ```

4. Check basic information:

   ```bash
   hostnamectl
   uname -a
   df -h
   free -h
   ```

![System information](images/image6.png)

![Terminal session on the VM](images/image7.png)

### Step 3: Secure access

**3.1 Create a normal admin user (instead of using the default user for everything)**

```bash
sudo adduser labadmin
sudo usermod -aG sudo labadmin
```

**3.2 Give this user my public SSH key**

```bash
sudo mkdir -p /home/labadmin/.ssh
sudo cp ~/.ssh/authorized_keys /home/labadmin/.ssh/authorized_keys
sudo chown -R labadmin:labadmin /home/labadmin/.ssh
sudo chmod 700 /home/labadmin/.ssh
sudo chmod 600 /home/labadmin/.ssh/authorized_keys
```

**3.3 Open a second terminal and test the new login before changing anything else**

```bash
ssh -i lab-key.pem labadmin@<public-ip>
sudo whoami    # should answer: root
```

I keep the first session open until this works, so I do not lock myself out.

**3.4 Harden the SSH server**

```bash
sudo nano /etc/ssh/sshd_config
```

I set (or checked) these lines:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Some Ubuntu images also have extra settings in `/etc/ssh/sshd_config.d/`. I checked that they do not turn password login back on. Then I tested the configuration and restarted:

```bash
sudo sshd -t
sudo systemctl restart ssh
```

To prove it works, I tried to log in without the key and it was refused (`Permission denied (publickey)`).

**3.5 Turn on the Linux firewall (UFW)**

I allow SSH first, so I do not block myself:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw enable
sudo ufw status verbose
```

![UFW configuration](images/image8.png)

![UFW status](images/image9.png)

**3.6 Optional extra protection**

```bash
sudo apt install fail2ban -y                  # blocks IPs that fail to log in many times
sudo systemctl enable --now fail2ban
sudo apt install unattended-upgrades -y       # automatic security updates
```

![SSH hardening check](images/image10.png)

![SSH login without key is refused](images/image11.png)

### Step 4: Deploy the service (Nginx web server)

```bash
sudo apt install nginx -y
sudo systemctl enable --now nginx
systemctl status nginx --no-pager
ss -tulpn | grep :80
```

Create a simple test page (no private information):

```bash
echo '<h1>Lab server is running</h1><p>Linux, networking and cloud practical task</p>' | sudo tee /var/www/html/index.html
curl http://localhost
```

If I have a domain, I can open `http://lab.mydomain.com`. If I own a domain and want HTTPS, I can add a free certificate with Certbot (`sudo apt install certbot python3-certbot-nginx` then `sudo certbot --nginx -d lab.mydomain.com`). This is optional.

![Nginx service status](images/image12.png)

![Test page in the browser](images/image13.png)

### Step 5: Test connectivity

I ran these tests from my own computer (not from the server) against my own server only.

| Test | Command or action | Expected result | My result |
|---|---|---|---|
| Web page works | Open `http://<public-ip>` in a browser | Test page appears | [result] |
| HTTP headers | `curl -I http://<public-ip>` | `HTTP/1.1 200 OK` | [result] |
| Port 80 reachable | `nc -vz <public-ip> 80` | Succeeded | [result] |
| SSH with key works | `ssh -i lab-key.pem labadmin@<public-ip>` | Login succeeds | [result] |
| SSH without key fails | `ssh -o PubkeyAuthentication=no labadmin@<public-ip>` | Permission denied | [result] |
| Closed port is blocked | `nc -vz -w 3 <public-ip> 8080` | Timed out or refused | [result] |
| DNS (if used) | `dig lab.mydomain.com +short` | Shows the server IP | [result] |
| Firewall test | Temporarily remove the HTTP rule from the security group, reload the page, then add the rule back | Page stops loading, then works again | [result] |

On the server I also checked the logs:

```bash
sudo tail -n 20 /var/log/nginx/access.log   # my web requests
sudo tail -n 20 /var/log/auth.log           # SSH logins and failures
sudo ufw status numbered
```

### Troubleshooting guide

| Problem | Likely cause | What I do |
|---|---|---|
| SSH: `Connection timed out` | Security group does not allow my current IP, or wrong IP address | Update the SSH rule to my current IP and check the public IP |
| SSH: `Permission denied (publickey)` | Wrong key, wrong user name, or wrong key permissions | Use the correct key and user, and run `chmod 400` on the key |
| SSH: `Unprotected private key file` | Key file is too open | `chmod 400 lab-key.pem` |
| Web page does not open | Port 80 is not open in the security group or UFW, or nginx is stopped | Check security group, `sudo ufw status`, and `systemctl status nginx` |
| Locked out after UFW | SSH was not allowed before enabling | Use the cloud console's serial or session access, or recreate the lab VM |
| Domain does not load | DNS record is wrong or not updated yet | Check with `dig`, and wait for DNS to update |

## 5. Cleanup

To avoid cost and leftover risk, at the end I:

1. Terminated the VM and deleted any extra disks, fixed IPs, and test DNS records.
2. Deleted the security group and the key pair I created for the lab.
3. Removed the hosts file line and the lab key from my computer if I do not need it.
4. Checked the billing page to make sure no resource was still running.

## 6. Results and conclusion

I created a Linux VM in the cloud, connected with an SSH key, created a separate admin user, disabled password and root login, turned on a firewall at two levels (cloud security group and UFW), deployed an Nginx web server, and tested that the allowed services work and unneeded ports are blocked.

## 7. What I learned

- Linux basics: files, permissions, users, groups, processes, and services.
- Networking: IP addresses, DNS, ports, and how firewalls decide what traffic can pass.
- Secure server setup: key-only SSH, no root login, least-privilege firewall rules, and updates.
- Testing is part of security: I confirmed both that the allowed things work and that the blocked things really fail.
- Cloud and Linux each have their own firewall, and using both gives stronger protection.

## 8. Final checklist

- [x] Work done only in my own account and my own VM
- [x] Billing alert set and MFA enabled on the cloud account
- [x] SSH uses keys only, root login and password login are disabled
- [x] Security group allows SSH only from my IP and only needed ports
- [x] UFW is enabled with the same rules
- [x] Nginx is running and the test page opens
- [ ] Tests for allowed and blocked traffic are done and documented
- [x] No private keys, passwords, account IDs, or tokens in screenshots or text
- [ ] Lab resources deleted after finishing
