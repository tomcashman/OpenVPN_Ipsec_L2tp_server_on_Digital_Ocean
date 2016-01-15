# OpenVPN and Ipsec L2tp server
The steps I take when setting up VPN server on Digital Ocean

###Create SSH Keys on client computer
Check for existing SSH keys

```bash
    ls -al ~/.ssh
```

Generate new SSH key

```bash
ssh-keygen -t rsa -b 4096 -C your_email@example.com
```

The public key is now located in `/home/demo/.ssh/id_rsa.pub` The private key (identification) is now located in `/home/demo/.ssh/id_rsa`. Add your keys to the droplet when creating it

###Login after creating droplet
Login as root

```bash
ssh root@server_ip_address
```

Create new user

```bash
adduser demo
```

Give root privileges

```bash
gpasswd -a demo sudo
```

Add public key authentication for new user

```bash
ssh-keygen -t rsa -b 4096 -C your_email@example.com
```

Manually install the key. Copy the public key by `CTRL-C` or `(cat ~/.ssh/id_rsa.pub)`

```bash
su - demo
mkdir .ssh
chmod 700 .ssh
```

Paste in public key while in nano

```bash
sudo nano .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
```

Exit will return to root

```bash
exit
```

Now you can log in as the new user

###Install OpenVPN

```bash    
wget git.io/vpn --no-check-certificate -O openvpn-install.sh && bash openvpn-install.sh
```

Copy unified .ovpn to client computer

```bash
scp -P root@server_ip_address:client.ovpn Downloads/
```

###Disable root login

```bash
sudo nano /etc/ssh/sshd_config
PermitRootLogin without-password
PasswordAuthentication no
```

To change SSH port - ALLOW NEW PORT IN UFW RULES BELOW AND RESTART UFW BEFORE RESTARTING SSH

```bash
Port 22
reload ssh
sudo restart ssh
```

###Enable UFW
    
```bash    
ufw limit 22
ufw allow 1194/udp
sudo nano /etc/default/ufw
```

Change from DROP to ACCEPT

```bash
DEFAULT_FORWARD_POLICY="ACCEPT"
sudo nano /etc/ufw/before.rules
```

Add these lines to the before.rules file

```bash
# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0]
# Allow traffic from OpenVPN client to eth0
-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
COMMIT
# END OPENVPN RULES
```
Your ufw rules should look similar to this

```bash
#Status: active
#Logging: on (low)
#Default: deny (incoming), allow (outgoing), allow (routed)
#New profiles: skip

#To                         Action      From
#--                         ------      ----
#22                         LIMIT IN    Anywhere
#1194/udp                   ALLOW IN    Anywhere
#500/udp                    ALLOW IN    Anywhere
#4500/udp                   ALLOW IN    Anywhere
#1194/udp (v6)              ALLOW IN    Anywhere (v6)
#22 (v6)                    LIMIT IN    Anywhere (v6)
#500/udp (v6)               ALLOW IN    Anywhere (v6)
#4500/udp (v6)              ALLOW IN    Anywhere (v6)
```

###Install Dnsmasq

Check your current nameserver configuration

```bash
cat /etc/resolv.conf
```

Install Dnsmasq

```bash
sudo apt-get install dnsmasq
cat /etc/resolv.conf
```

Take note of query time

```bash
dig digitalocean.com @localhost
```

Check again after cached

```bash
dig digitalocean.com @localhost
```

###Allow multiple clients to connect with same ovpn file. Better to create multiple ovpn files

```bash
sudo nano /etc/openvpn/server.conf
```

Uncomment following line:

```bash
duplicate-n
```

Restart OpenVPN service

```bash
sudo service openvpn restart
```

###Configure NTP sync

```bash
sudo apt-get update
sudo apt-get install ntp
```

Configure timezone to UTC

```bash    
sudo dpkg-reconfigure tzdata
sudo ntpdate pool.ntp.org
sudo service ntp start
```

###Enable Automatic Upgrades

```bash
sudo apt-get install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

Update the 10 periodic file. "1" means that it will upgrade every day

```bash
/etc/apt/apt.conf.d/10periodic
```

```bash
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
```

###Autostart OpenVPN on Debian client computer

```bash
sudo nano /etc/default/openvpn
```

Uncomment:

```bash
AUTOSTART=all
```

Copy client.ovpn to /etc/openvpn/client.conf by renaming file

```bash
gksu -w -u root gksu thunar
```

Reload openvpn configuration

```bash
/etc/init.d/openvpn reload /etc/openvpn/client.conf
```

Check for tun0 interface

```bash
ifconfig
```

###Install send only SSMTP service

```bash
sudo apt-get install ssmtp
sudo nano /etc/ssmtp/ssmtp.conf
```

```bash
#root=postmaster
root=your_email@example.com
#mailhub=mail
mailhub=smtp.gmail.com:587
AuthUser=your_email@example.com
AuthPass=your_password
UseTLS=YES
UseSTARTTLS=YES
#rewriteDomain=
rewriteDomain=gmail.com
#hostname=your_hostname
hostname=your_email@example.com
```

Test sstp in terminal

```bash
ssmtp recipient_email@example.com
```

Format message as below:
```bash
To: recipient_email@example.com
From: myemailaddress@gmail.com
Subject: test email

hello world!
```

Note the blank line after the subject, everything after this line is the body of the email. When you're finished, press Ctrl-D. sSMTP may take a few seconds to send the message before closing.

###Setup fail2ban

```bash
sudo apt-get install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

```bash
# "ignoreip" can be an IP address, a CIDR mask or a DNS host
ignoreip = 127.0.0.⅛
bantime  = 600
maxretry = 3
```

Setup ssmtp settings with following settings

```bash
destemail = your_email@example.com
sendername = Fail2Ban
mta = sendmail
#_mwl sends email with logs
action = %(action_mwl)s
```

Jails we can initially set to true without any errors

```bash
#ssh
#dropbear
#pam-generic
#ssh-ddos
#postfix
#couriersmtp
#courierauth
#sasl
#dovecot
```

Restart fail2ban

```bash
sudo service fail2ban stop
sudo service fail2ban start
```

Check list of banned IP for fail2ban

```bash
fail2ban-client status ssh
iptables --list -n | fgrep DROP
```

###Full system backup using rsync.

Using the -aAX set of options, all attributes are preserved

```bash
rsync -aAXv --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} root@your_hostname:/ /home/demo/backup/
```

###Configuring TripWire

```bash
sudo apt-get install tripwire
```

Set the Site-Key passphrase and the Local-Key passphrase

Create policy file

```bash
sudo twadmin --create-polfile /etc/tripwire/twpol.txt
```

Initialize database

```bash
sudo tripwire --init
sudo sh -c 'tripwire --check | grep Filename > /etc/tripwire/test_results'
```

Entries may look like this:

```bash
less /etc/tripwire/test_results
```

```bash
     Filename: /etc/rc.boot
     Filename: /root/mail
     Filename: /root/Mail
     Filename: /root/.xsession-errors
     Filename: /root/.xauth
     Filename: /root/.tcshrc
     Filename: /root/.sawfish
     Filename: /root/.pinerc
     Filename: /root/.mc
     Filename: /root/.gnome_private
     Filename: /root/.gnome-desktop
     Filename: /root/.gnome
     Filename: /root/.esd_auth
     Filename: /root/.elm
     Filename: /root/.cshrc
     Filename: /root/.bash_profile
     Filename: /root/.bash_logout
     Filename: /root/.amandahosts
     Filename: /root/.addressbook.lu
     Filename: /root/.addressbook
     Filename: /root/.Xresources
     Filename: /root/.Xauthority
     Filename: /root/.ICEauthority
     Filename: /proc/30400/fd/3
     Filename: /proc/30400/fdinfo/3
     Filename: /proc/30400/task/30400/fd/3
     Filename: /proc/30400/task/30400/fdinfo/3
```

Edit text policy in editor

```bash
sudo nano /etc/tripwire/twpol.txt
```

Search for each of the files that were returned in the test_results file. Comment out lines that match.

```bash
    {
        /dev                    -> $(Device) ;
        /dev/pts                -> $(Device) ;
        #/proc                  -> $(Device) ;
        /proc/devices           -> $(Device) ;
        /proc/net               -> $(Device) ;
        /proc/tty               -> $(Device) ;
        . . .
```

Comment out /var/run and /var/lock lines so that system does not flag normal filesystem changes by services:

```bash
    (
  rulename = "System boot changes",
  severity = $(SIG_HI)
    )
    {
        #/var/lock              -> $(SEC_CONFIG) ;
        #/var/run               -> $(SEC_CONFIG) ; # daemon PIDs
        /var/log                -> $(SEC_CONFIG) ;
    }
```

Save and close

Implement by re-creating encrypted policy file that tripwire reads

```bash
sudo twadmin -m P /etc/tripwire/twpol.txt
```

Reinitialize the database to implement policy

```bash
sudo tripwire --init
```

Warnings should be gone. If there are still warnings, continue editing /etc/tripwire/twpol.txt file until gone.

The basic syntax for a check is

```bash
sudo tripwire --check
```

Delete the test_results file that we created

```bash 
sudo rm /etc/tripwire/test_results
```
Remove plain text configuration files

```bash    
sudo sh -c 'twadmin --print-polfile > /etc/tripwire/twpol.txt'
```

Move text version to backup location and recreate it

```bash
sudo mv /etc/tripwire/twpol.txt /etc/tripwire/twpol.txt.bak
sudo sh -c 'twadmin --print-polfile > /etc/tripwire/twpol.txt'
```

Remove plain text files

```bash
sudo rm /etc/tripwire/twpol.txt
sudo rm /etc/tripwire/twpol.txt.bak
```

Send an email notifications

```bash
sudo apt-get install mailutils
```

See if we can send email

```bash
sudo tripwire --check | mail -s "Tripwire report for `uname -n`" your_email@example.com
```

Check report that was sent with the email

```bash
sudo tripwire --check --interactive
```

Remove “x” from box if not ok with change

Automate Tripwire with Cron

Check to see if root already has a crontab by issuing this command:

```bash
sudo crontab -l
```

If a crontab is present, you should pipe it into a file to back it up:

```bash
sudo sh -c 'crontab -l > crontab.bad'
```

Afterwards, we can edit the crontab by typing:

```bash
sudo crontab -e
```

To have tripwire run at 3:30am every day, we can place a line like this in our file:

```bash
30 3 * * * /usr/sbin/tripwire --check | mail -s "Tripwire report for `uname -n`" your_email@example.com
```

###Maintenance Commands

Programs holding an open network socket

```bash
lsof -i
```

Show all running processes

```bash
ps -ef
```

Who is logged on

```bash
who -u
```

Kill the process that you want

```bash
kill "pid"
```

Check SSH sessions

```bash
ps aux | egrep "sshd: [a-zA-Z]+@"
```

Check SSHD

```bash    
ps fax
```

Check last logins

```bash
last
```

Check ufw status

```bash
sudo ufw status verbose
```

Delete ufw rules

```bash
sudo ufw delete deny "port"
```

Check logs

```bash
grep -ir ssh /var/log/* 
grep -ir sshd /var/log/* 
grep -ir breakin /var/log/* 
grep -ir security /var/log/*
```

Tree directory 

```bash
http://www.cyberciti.biz/faq/linux-show-directory-structure-command-line/
```

See all files

```bash
tree -a
```

List directories only

```bash
tree -d
```
Colorized output

```bash    
tree -C
```

File management

```bash
https://www.digitalocean.com/community/tutorials/basic-linux-navigation-and-file-management
http://www.computerworld.com/article/2598082/linux/linux-linux-command-line-cheat-sheet.html
http://www.debian-tutorials.com/beginners-how-to-navigate-the-linux-filesystem
```

LSOF Commands

```bash
https://stackoverflow.com/questions/106234/lsof-survival-guide
```

How to kill zombie process

```bash
ps aux | grep 'Z'
```

Find the parent PID of the zombie

```bash
pstree -p -s 93572
```

Check IPTables traffic

```bash
sudo iptables -v -x -n -L
```
Report file system disk space

```bash
df -Th
```

Check trash size

```bash
sudo find / -type d -name '*Trash*' | sudo xargs du -h | sort
```

Check size of packages in apt

```bash
du -h /var/cache/apt/
```

Check size of log files

```bash
sudo du -h /var/log
```

Check size of lost+found folder

```bash
sudo find / -name "lost+found" | sudo xargs du -h
```

How to delete lots of text in nano

```bash
Scroll to top of text, press Alt+A, Ctrl-V to bottom of text, press Ctrl-K to cut the text, Ctrl-O to save, Ctrl-X to exit
```

How to scan top 8000 ports using nmap

```bash
nmap -vv --top-ports 8000 your_hostname
```

###Install Libreswan

```bash
https://blog.ls20.com/ipsec-l2tp-vpn-auto-setup-for-ubuntu-12-04-on-amazon-ec2/
https://github.com/hwdsl2/setup-ipsec-vpn
```

Open ports 500 and 4500 before running script

```bash
PSK:your_private_key
Username:your_username
Password:your_password
```

Change IP tables in script from port 22 to non-standard port for ssh

If problems with openvpn after install run the following iptable rules then restart ufw and openvpn

```bash
sudo iptables -I INPUT -p udp --dport 1194 -j ACCEPT
sudo iptables -I FORWARD -s 10.8.0.0/24 -j ACCEPT
sudo iptables -I FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo service ufw stop
sudo service ufw start
sudo /etc/init.d/openvpn restart
```

I remedied the above problems with iptables persistence by running the following commands. Shouldn't have to run the commands anymore.

```bash
sudo iptables-save > /etc/iptables.rules
```

Insert these lines in /etc/rc.local:

```bash
iptables-restore < /etc/iptables.rules
```
