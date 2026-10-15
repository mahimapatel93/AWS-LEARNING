# Linux / DevOps Command Cheatsheet

Quick reference for the Linux commands that actually come up day-to-day when managing a server — grouped by what you're trying to do, not just alphabetically.

---

## 👤 User Management

| Command | Easy Meaning | When You'll Actually Use It |
|---|---|---|
| `sudo adduser devops` | Create a new user | A new teammate joins, so you create an account for them. |
| `sudo usermod -aG sudo devops` | Give admin access | So the user can install Docker, restart Nginx, update the server, etc. |
| `groups devops` | Show user's groups | Check if the user has admin (sudo) access. |
| `id devops` | Show user information | Check the user's ID and which groups they belong to. |
| `whoami` | Who am I? | Check which user you're currently logged in as. |
| `su - devops` | Switch to another user | Test if that user's account works properly. |

---

## 🔑 SSH

| Command | Easy Meaning | When You'll Actually Use It |
|---|---|---|
| `ssh -i key.pem ubuntu@IP` | Login to server | Connect to your AWS EC2 server. |
| `scp app.zip ubuntu@IP:/home/ubuntu` | Copy file to server | Upload your application to the server. |
| `scp -r project ubuntu@IP:/home/ubuntu` | Copy folder | Upload your whole project. |

---

## 📂 Files & Folders

| Command | Easy Meaning | When You'll Actually Use It |
|---|---|---|
| `pwd` | Show current folder | You forgot where you are. |
| `ls -la` | Show all files | Check if `.env` or hidden files exist. |
| `cd project` | Go into folder | Move to your application folder. |
| `mkdir logs` | Create folder | Make a folder for logs or backups. |
| `touch .env` | Create file | Create an environment file. |
| `cp nginx.conf nginx.conf.backup` | Copy file | Backup config before editing. |
| `mv app.conf default.conf` | Rename file | Rename a config file. |
| `rm -rf old-app` | Delete folder | Remove an old deployment. |

---

## 📄 Reading Files

| Command | Easy Meaning | When You'll Actually Use It |
|---|---|---|
| `cat .env` | Show file | Check environment variables. |
| `less app.log` | Read big file | Open a large log file. |
| `tail -f app.log` | Watch logs live | See errors while the app is running. |
| `head app.log` | First few lines | Check how a log starts. |

---

## 🔍 Searching

| Command | Easy Meaning | When You'll Actually Use It |
|---|---|---|
| `find . -name Dockerfile` | Find a file | You don't know where the Dockerfile is. |
| `grep ERROR app.log` | Search for "ERROR" | Find out why the app crashed. |
| `which docker` | Find Docker location | Check if Docker is installed correctly. |

---

## ⚙️ Services

| Command | Easy Meaning | When You'll Actually Use It |
|---|---|---|
| `systemctl status nginx` | Check if service is running | Website isn't opening, so check Nginx. Same command works for `ssh`/`sshd` if you can't connect and want to rule out the service itself. |
| `systemctl restart nginx` | Restart service | After changing Nginx config. |
| `systemctl start docker` | Start Docker | Docker stopped after a reboot. |
| `systemctl stop nginx` | Stop service | Before maintenance. |
| `systemctl enable docker` | Start Docker on boot | Make Docker start automatically after restart. |

---

## 📜 Logs

| Command | Easy Meaning | When You'll Actually Use It |
|---|---|---|
| `journalctl -u nginx` | Show Nginx logs | Nginx failed to start. Find the reason. Swap in `-u ssh --since "10 minutes ago"` to confirm which IP actually reached the server during an SSH timeout. |
| `journalctl -f` | Watch logs live | See new errors as they happen. |
| `dmesg` | Show kernel logs | Check hardware or boot problems. |

---

## 🌐 Networking

| Command | Easy Meaning | When You'll Actually Use It |
|---|---|---|
| `ip a` | Show IP address | Find your server's IP. |
| `ping google.com` | Check internet | Test if the server has internet access. |
| `curl http://localhost:5000` | Test API | Check if your backend is working. |
| `curl -4 ifconfig.me` | Show my public IPv4 | Check what IP your machine is currently using before updating a Security Group rule. |
| `wget URL` | Download file | Download software or scripts. |
| `ss -tulnp` | Show open ports | Check if your app is listening on port 5000. |
| `lsof -i :80` | Find who is using port 80 | Another app is blocking Nginx. |
| `dig google.com` | Check DNS | Domain name isn't working. |

---

## 💻 Processes

| Command | Easy Meaning | When You'll Actually Use It |
|---|---|---|
| `ps -ef` | Show running programs | Check if your app is running. |
| `top` | Show CPU & RAM usage | Server feels slow. |
| `htop` | Better system monitor | Find which app uses the most resources. |
| `kill PID` | Stop program | Stop a stuck application. |
| `kill -9 PID` | Force stop | App isn't stopping normally. |
| `pgrep nginx` | Find process ID | Need Nginx's process ID. |

---

## 💾 Disk & Memory

| Command | Easy Meaning | When You'll Actually Use It |
|---|---|---|
| `df -h` | Show disk space | Check if the server is out of storage. |
| `du -sh logs` | Show folder size | Find which folder is using lots of space. |
| `free -h` | Show RAM usage | Check if memory is full. |
| `lsblk` | Show disks | Verify a new disk is attached. |

---

## 🔒 Permissions

| Command | Easy Meaning | When You'll Actually Use It |
|---|---|---|
| `chmod +x deploy.sh` | Make script executable | Run your deployment script. |
| `chmod 755 file` | Change permissions | Give correct access to a file. |
| `chown ubuntu:ubuntu app` | Change owner | Fix "Permission denied" errors. |

---

## 📦 Software Installation

| Command | Easy Meaning | When You'll Actually Use It |
|---|---|---|
| `sudo apt update` | Refresh package list | Before installing anything. |
| `sudo apt install docker.io` | Install Docker | Set up a new server. |
| `sudo apt install nginx` | Install Nginx | Host a website. |
| `sudo apt upgrade -y` | Update software | Keep the server secure. |

---

## 🐞 Debugging SSH Connection Errors (Timeouts, Refused, etc.)

| Command | Easy Meaning | When You'll Actually Use It |
|---|---|---|
| `Test-NetConnection <IP> -Port 22` | Check if a port is reachable (Windows) | See whether SSH (port 22) is reachable from your PC before blaming your key or username. |
| `nc -zv <IP> 22` | Check if a port is open (Linux/Mac) | Same check as `Test-NetConnection`, from a Linux or Mac terminal. |
| `ssh -v -i key.pem ubuntu@IP` | SSH with verbose output | See exactly where the connection hangs or fails (network, key, or auth). |
| `sudo apt install wtmpdb` then `last` | Install and check login history | Get a clean list of past successful logins and the IPs they came from, when `last` isn't available out of the box. |
| `sudo ufw status` | Check host firewall rules | Rule out the Ubuntu firewall itself blocking port 22 (uncommon on a fresh AMI, but worth ruling out). |
| `sudo iptables -L -n` | Check low-level firewall rules | Same idea as `ufw status`, for servers managing iptables directly. |
| `aws ec2 describe-security-groups --group-ids sg-xxxx` | Show a Security Group's current rules | Double-check the source IP/CIDR currently allowed, without opening the console. |
| `aws ec2 authorize-security-group-ingress ...` | Update a Security Group rule | Script the fix once you know your real IP, instead of clicking through the console. |

**How these fit together, plus `curl -4 ifconfig.me`, `systemctl status ssh`, and `journalctl -u ssh` from above:**
1. `Test-NetConnection` / `nc -zv` tells you *whether* port 22 is reachable at all.
2. `curl -4 ifconfig.me` gives you a first guess at your own IP — treat it as a guess, not the truth.
3. If you're locked out entirely, temporarily open the Security Group to `0.0.0.0/0`, SSH in, then check `last` (via `wtmpdb`) or `journalctl -u ssh --since "10 minutes ago"` from *inside* the server to see the real IP that actually connected.
4. `systemctl status ssh` and `sudo ufw status` rule out the instance itself (service down, or host firewall blocking) as separate from a Security Group/IP mismatch.
5. Once you know the real IP, update the Security Group rule (console or `aws ec2 authorize-security-group-ingress`) and re-test.

---

## ⭐ The Commands You'll Actually Type Every Day

| Task | Go-to Commands |
|---|---|
| Connect to a server | `ssh` |
| Upload project files | `scp` |
| Check if the app is running | `systemctl status`, `ps -ef` |
| Read application logs | `tail -f`, `journalctl` |
| Search for errors | `grep` |
| Check server health | `top`, `free -h`, `df -h` |
| Test the application | `curl` |
| Check open ports | `ss -tulnp` |
| Fix permissions | `chmod`, `chown` |
| Create users and give admin access | `adduser`, `usermod -aG sudo` |