# How to install Samba 4 AD DC on CentOS 8 and connect windows and linux clients #


How to install AD in CentOS server and connect windows and linux machines as clients
Im testing on virtual environment with virtual box

## Requirements ##

>- [CentOS 8](http://miroir.univ-lorraine.fr/centos/8.0.1905/isos/x86_64/CentOS-8-x86_64-1905-dvd1.iso)
>- [Samba 4](https://download.samba.org/pub/samba/stable/samba-4.11.2.tar.gz)
>- [Windows 10](https://www.microsoft.com/software-download/windows10)

## My environment ##

>- [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

Every machine is attached to 'NAT Network' on virtual box, this network is 192.168.0.0/24

Machine adserver = 2048 MB of RAM
Machine clientlinux = 3096 MB of RAM
Machine client_win = 3096 MB of RAM

My domain will be `intra.it`

linux username = toto
linux password = toto
linux root password = toto
samba administrator password = Toto123

**On virtual box with NAT network the gateway is the first address of the network, in this scenario is 192.168.0.1**

## Configuring Server ##

>- Install virtual box, and create one machine for the server, one for the linux client and one for windows client
>- Configue the network to NAT Network
>- Enable bidirecional clipboard on the machines, it can be very useful
>- Install install centOS and windows on its machines

## Configuring the server ##

### Updating ###
1. Fill safe to take snapshots of the machine after important steps
2. Login as root **every following step is made on root user**

```
sudo su
```

2. Open a terminal and add the EPEL repository:

```
yum install -y epel-release
```

3. Update all packets:

```
yum update
```

### Configuring the computer ###

1. Configure you network card to have fixed IP, in my case I'll use **192.168.0.10/24** as my IP, and as I said, **192.168.0.1/24** as gateway
tip: on linux usually the network file is on /etc/network/interfaces, but in centos 8 its on /etc/sysconfig/network-scripts/ifcfg-`interface_name`

2. Set your computer name and your domain, following the patern `hostnamectl set-hostname --static "$COMPUTERNAME.$DOMAIN"`, **choose a DNS valid name for the computer**

```
hostnamectl set-hostname --static "adserver.intra.it"
```

3. Reboot your computer

```
reboot
```

4. Check the configurations

```
ifconfig

# parcial result:
#enp0s3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
#inet 192.168.0.10  netmask 255.255.255.0  broadcast 192.168.0.255
```

```
hostname && dnsdomainname 

# result:
#adserver.intra.it
#intra.it
```
        

### Setting up NTP server ###
1. In order to have the network working without problems, we need to make sure that every machine have the same datetime, so we are going to syncrhonize them, install chrony:

```
yum install -y chrony
```   

2. Enable chrony at boot:

```
systemctl enable chronyd
```  

3. Add your network to chrony files, you may need to add other networks in this file for other scenarios. The format is `$network_ip/$network_mask`

```
printf "\nallow 192.168.0.0/24\n" >> /etc/chrony.conf
```

4. Restart chrony service

```
systemctl restart chronyd
```

5. Permit chrony on firewall and then reload it

```
firewall-cmd --permanent --add-service=ntp # open ntp on firewall
firewall-cmd --reload # reload firewall
```

### Instaling samba ###

0. Disable SElinux and reboot

```
sed -i -e 's/SELINUX=enforcing/SELINUX=disabled/g;s/SELINUX=permissive/SELINUX=disabled/g' /etc/selinux/config
reboot
```

1. Add PowerTools repository

```
yum install -y dnf-plugins-core
yum config-manager --set-enabled PowerTools
```

2. Install dependencies

```
yum install -y cups-devel docbook-style-xsl gcc gdb gnutls-devel gpgme-devel jansson-devel keyutils-libs-devel krb5-workstation libacl-devel libaio-devel libarchive-devel libattr-devel libblkid-devel libtasn1 libtasn1-tools libxml2-devel libxslt lmdb-devel openldap-devel pam-devel perl perl-ExtUtils-MakeMaker perl-Parse-Yapp popt-devel python3-cryptography python3-dns python3-gpg python36-devel readline-devel rpcgen systemd-devel tar zlib-devel 
```

3. Download samba, uncompress it and enter its folder

```
wget https://download.samba.org/pub/samba/stable/samba-4.11.2.tar.gz # download
tar -zxvf samba-4.11.2.tar.gz # uncompress
cd samba-4.11.2 # enter
```

4. Compile and install it **this might take a while**

```
./configure --enable-debug 
make && make install
```

5. Add samba binaries to you PATH variable to be able to run its commands on terminal, this is done adding `export PATH=$PATH:/usr/local/samba/bin/:/usr/local/samba/sbin/` to the end of the file .bash_profile of our user (works for terminals opened after this command), and runing it one (works for the current session)

```
printf "\nexport PATH=$PATH:/usr/local/samba/bin/:/usr/local/samba/sbin/\n" >> /root/.bash_profile # for futher sessions
export PATH=$PATH:/usr/local/samba/bin/:/usr/local/samba/sbin/ # for this session
```

6. Make a backup of your samba file

```
cp /etc/samba/smb.conf /etc/samba/smb.conf.raw
```

7. Edit /etc/samba/smb.conf setting:
   <!-- verificar: https://blog.godatadriven.com/samba-configuration -->
   <!-- `server services = rpc, nbt, wrepl, ldap, cldap, kdc, drepl, winbind, ntp_signd, kcc, dnsupdate, dns, s3fs` isso deveria ser adicionado em global? -->
   >- Inside [global] session:
   >>- Set workgroup equal to your SLD (first part of domain) **in uppercase**
   >>- Add the following line `kerberos method = system keytab`
   >>- Add the following line `realm = INTRA.IT` which has the format `realm = $DOMAIN` **in uppercase**
   <!-- >>- Add the following line `dns forwarder = 8.8.8.8` **dont add this line if you want to forward to a local dns** -->
   >>- Add the following line `idmap_ldb:use rfc2307 = yes`
   >>- Add the following line `tls enabled = yes`
   >>- Add the following line `tls keyfile = tls/key.pem`
   >>- Add the following line `tls cafile = tls/ca.pem`
   >>- Add the following line `tls certfile = tls/cert.pem`

After this step my [global] session looks like: <!--  removed: dns forwarder = 8.8.8.8 -->

```
[global]
        workgroup = INTRA
        realm = INTRA.IT
        security = user
        passdb backend = tdbsam
        printing = cups
        printcap name = cups
        load printers = yes
        cups options = raw
        kerberos method = system keytab
        idmap_ldb:use rfc2307 = yes
        tls enabled = yes
        tls keyfile = tls/key.pem
        tls cafile = tls/ca.pem
        tls certfile = tls/cert.pem
```


8. Make a backup of you configured file

```
cp /etc/samba/smb.conf /etc/samba/smb.conf.$(dnsdomainname)
```

9. Use samba-tool to configure your samba ad dc

Realm: INTRA.IT **uppercase**
Domain: INTRA **uppercase**
Server role: dc # domain controller
DNS backend: SAMBA_INTERNAL 
DNS forwarder ip address: 8.8.8.8 # **you maybe want to forward another local dns**

```
samba-tool domain provision --use-rfc2307 --interactive
``` 

10.  Check you everything worked

```
samba-tool domain level show

# result:
#Domain and forest function level for domain 'DC=intra,DC=it'
#
#Forest function level: (Windows) 2008 R2
#Domain function level: (Windows) 2008 R2
#Lowest function level of a DC: (Windows) 2008 R2
```

11. Allow samba on your firewall and then reload it

```
firewall-cmd --add-port=53/tcp --permanent;firewall-cmd --add-port=53/udp --permanent; #enable DNS
firewall-cmd --add-port=88/tcp --permanent;firewall-cmd --add-port=88/udp --permanent; #enable kerberos_auth(88
firewall-cmd --add-port=135/tcp --permanent; # enable Microsoft EPMAP
firewall-cmd --add-port=137-138/udp --permanent;firewall-cmd --add-port=139/tcp --permanent; # enable NETBIOS
firewall-cmd --add-port=389/tcp --permanent;firewall-cmd --add-port=389/udp --permanent; # enable LDAP
firewall-cmd --add-port=445/tcp --permanent; # enable Active Directory shares
firewall-cmd --add-port=464/tcp --permanent;firewall-cmd --add-port=464/udp --permanent; # enable kerberos passwd
firewall-cmd --add-port=636/tcp --permanent; # enable LDAP crypt
firewall-cmd --add-port=3268-3269/tcp --permanent # enable microsoft  global catalog
firewall-cmd --reload # reload
```

12. Create a samba service file, fix its permissions, set it to run on boot and start it

```
printf "[Unit]\nDescription= Samba 4 Active Directory\nAfter=syslog.target\nAfter=network.target\n\n[Service]\nType=forking\nPIDFile=/usr/local/samba/var/run/samba.pid\nExecStart=/usr/local/samba/sbin/samba\n\n[Install]\nWantedBy=multi-user.target" >> /etc/systemd/system/samba_ad.service # create
chmod 664 /etc/systemd/system/samba_ad.service # fix permissions
systemctl enable samba_ad # enable at boot
systemctl start samba_ad # start
```

You can check if samba is running and its listening ports by using:

```
netstat -tulpn | egrep 'smbd|samba'
```

The file `samba_ad.service` looks like this:

```
[Unit]
Description= Samba 4 Active Directory
After=syslog.target
After=network.target

[Service]
Type=forking
PIDFile=/usr/local/samba/var/run/samba.pid
ExecStart=/usr/local/samba/sbin/samba

[Install]
WantedBy=multi-user.target
```

13. **Optional** since i'm using virtual machines, and i have low RAM memory i would like to disable the graphical mode of my server, linux systems have the following init values:

>- 0 – Halt.
>- 1 – Single-user text mode.
>- 2 – Not used (user-definable)
>- 3 – Full multi-user text mode.
>- 4 – Not used (user-definable)
>- 5 – Full multi-user graphical mode (with an X-based login screen)
>- 6 – Reboot.

The following command is used to set the boot to be only in text mode:

```
systemctl set-default multi-user.target # boot only in text mode
```

You can revert this action using the following command:

```
systemctl set-default graphical.target # boot with graphics
```

Useful commands:
>- `startx` to start the graphical mode
>- `init 1` -> **give root password** -> `init 2` to kill the graphical mode


## Connect with a Linux client ##

### Configuring the computer ###

1. Configure you network card to have a fixed ip (e.g. **192.168.0.11/24**) or leave it as DHCP, **set the DNS to your server ip, in my scenario 192.168.0.10** and as I said, **192.168.0.1/24** as gateway
   
2. Set your computer name and your domain, following the patern `hostnamectl set-hostname --static "$COMPUTERNAME.$DOMAIN"`, **choose a DNS valid name for the computer**

```
hostnamectl set-hostname --static "clientlinux.intra.it"
```

3. Modify /etc/hosts, adding a line with your ad server in the format: `$IP $SERVERNAME.$DOMAIN $SERVERNAME` and remove the line that starts with `127.0.0.1` and add instead the a line with your localhost in the format: `127.0.0.1 $COMPUTERNAME.$DOMAIN $COMPUTERNAME`:

```
sed -i '/^127.0.0.1/ d' /etc/hosts
sed -i '1s/^/192.168.0.10 adserver.intra.it adserver\n127.0.0.1 clientlinux.intra.it localhost clientlinux\n/' /etc/hosts
```

My /etc/hosts looks like:

```
192.168.0.10 adserver.intra.it adserver
127.0.0.1 clientlinux.intra.it localhost clientlinux
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```

4. Reboot your computer

```
reboot
```

5. Check the configurations

```
ifconfig

# parcial result:
#enp0s3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
#inet 192.168.0.11  netmask 255.255.255.0  broadcast 192.168.0.255
```

```
hostname && dnsdomainname 

# result:
#clientlinux.intra.it
#intra.it
```

```
ping -c4 adserver.intra.it

# result:
#PING adserver.intra.it (192.168.122.1) 56(84) bytes of data.
#64 bytes from localhost.localdomain (192.168.122.1): icmp_seq=1 ttl=64 time=0.045 ms
#64 bytes from localhost.localdomain (192.168.122.1): icmp_seq=2 ttl=64 time=0.104 ms
#64 bytes from localhost.localdomain (192.168.122.1): icmp_seq=3 ttl=64 time=0.096 ms
#64 bytes from localhost.localdomain (192.168.122.1): icmp_seq=4 ttl=64 time=0.061 ms
```

```
cat /etc/resolv.conf

# result:
## Generated by NetworkManager
#search intra.it
#nameserver 192.168.0.10
```

### Synchronize clock with server ###
1. Install chrony

```
sudo yum install chrony
```

2. Edit the file /etc/chrony.conf to add an entry to the ad server in the format `Server $SERVERIP` and to remove other servers commenting the line `pool 2.centos.pool.ntp.org iburst`

```
echo "Server 192.168.0.10" >> /etc/chrony.conf
sed -i -e '/pool 2.centos.pool.ntp.org iburst/ s/^#*/#/' /etc/chrony.conf
```

3. Enable chrony to launch at boot and start it

```
systemctl enable chronyd # enable chrony
systemctl start chronyd # start chrony
```

4. Check if your server is on the sources of chrony

```
chronyc sources

# result:
#210 Number of sources = 1
#MS Name/IP address         Stratum Poll Reach LastRx Last sample               
#===============================================================================
#^? adserver.intra.it             0   6     0     -     +0ns[   +0ns] +/-    0ns
```

### Joinning the domain using linux machine ###

1. Add EPEL repository

```
yum install -y epel-release
```

2. Install SSSD, realm, samba and kerberos:

```
yum install -y sssd realmd oddjob oddjob-mkhomedir adcli samba-winbind-krb5-locator samba-common samba-common-tools samba-winbind krb5-workstation openldap-clients
```

3. Make a backup of your kerberos file and delete the original file

```
mv /etc/krb5.conf /etc/krb5.conf.raw
```

4. Create a /etc/krb5.conf file with the contents of the file bellow, change every `INTRA.IT` by your domain and in **uppercase**, change every `intra.it` by your domain in lowercase, and change every `adserver` by your ad server name

```
printf "[logging]\ndefault = FILE:/var/log/krb5libs.log\nkdc = FILE:/var/log/krb5kdc.log\nadmin_server = FILE:/var/log/kadmind.log\n\n[libdefaults]\ndns_lookup_realm = true\ndns_lookup_kdc = true\nticket_lifetime = 24h\nrenew_lifetime = 7d\nforwardable = true\nrdns = false\ndefault_realm = INTRA.IT\n\n[realms]\n# Uncomment following if DNS lookups are not working\n# INTRA.IT = {\n# kdc = adserver.intra.it\n# master_kdc = adserver.intra.it\n# admin_server = adserver.intra.it\n# }\n\n[domain_realm]\n# Uncomment following if DNS lookups are not working\n# .intra.it = INTRA.IT\n# intra.it = INTRA.IT\n" > /etc/krb5.conf
```

The file `/etc/krb5.conf`: **THIAGO apenas a versão sem os comentarios está funcionado, realmente tive um problema com dns?**

```
[logging]
default = FILE:/var/log/krb5libs.log
kdc = FILE:/var/log/krb5kdc.log
admin_server = FILE:/var/log/kadmind.log

[libdefaults]
dns_lookup_realm = true
dns_lookup_kdc = true
ticket_lifetime = 24h
renew_lifetime = 7d
forwardable = true
rdns = false
default_realm = INTRA.IT

[realms] 
# Uncomment following if DNS lookups are not working
# INTRA.IT = {
# kdc = adserver.intra.it
# master_kdc = adserver.intra.it
# admin_server = adserver.intra.it
# }

[domain_realm] 
# Uncomment following if DNS lookups are not working
# .intra.it = INTRA.IT
# intra.it = INTRA.IT
```

5. Test your kerberos communication, replace `@intra.it` by `@$YOURDOMAIN`, the administrator password will be required

```
KRB5_TRACE=/dev/stdout kinit -V administrator@intra.it

# last line result:
#Authenticated to Kerberos v5
```

6. Make a backup of you configured file

```
cp /etc/krb5.conf /etc/krb5.conf.$(dnsdomainname)
``` 

7. Make a backup of your samba file and delete the original file

```
mv /etc/samba/smb.conf /etc/samba/smb.conf.raw
```

8. Create a /etc/samba/smb.conf file with the contents of the file bellow, change every `INTRA.IT` by your domain and in **uppercase**, change every `INTRA` by your SLD (first part of domain) and in **uppercase**

```
printf "[global]\n\nsecurity = ads\nrealm = INTRA.IT\nworkgroup = INTRA\nlog file = /var/log/samba/%%m.log\nkerberos method = secrets and keytab\nclient signing = yes\nclient use spnego = yes\n" > /etc/samba/smb.conf
```

The file `/etc/samba/smb.conf`: 

```
[global]

security = ads
realm = INTRA.IT
workgroup = INTRA
log file = /var/log/samba/%m.log
kerberos method = secrets and keytab
client signing = yes
client use spnego = yes
```

9. Make a backup of you configured file

```
cp /etc/samba/smb.conf /etc/samba/smb.conf.$(dnsdomainname)
```

10. Create a /etc/sssd/sssd.conf file with the contents of the file bellow, change every `intra.it` by your domain in lowercase, change every `INTRA.IT` by your domain in **uppercase**
    
```
printf "[sssd]\nconfig_file_version = 2\ndomains = intra.it\nservices = nss, pam\n\n[domain/intra.it]\n# Uncomment if you need offline logins\n# cache_credentials = true\n\nid_provider = ad\nauth_provider = ad\naccess_provider = ad\n\n# Uncomment if service discovery is not working\n# ad_server = adserver.intra.it\n\n# Uncomment if you want to use POSIX UIDs and GIDs set on the AD side\n# ldap_id_mapping = False\n\n# Uncomment if the trusted domains are not reachable\n#ad_enabled_domains = intra.it\n\n# Comment out if the users have the shell and home dir set on the AD side\ndefault_shell = /bin/bash\nfallback_homedir = /home/%%d/%%u\n\n# Uncomment and adjust if the default principal SHORTNAME$@REALM is not available\n# ldap_sasl_authid = host/client.intra.it@INTRA.IT\n\n# Comment out if you prefer to use shortnames.\nuse_fully_qualified_names = True\n\n# Uncomment if the child domain is reachable, but only using a specific DC\n# [domain/intra.it/child.intra.it]\n# ad_server = dc.child.intra.it\n" > /etc/sssd/sssd.conf
```

The `/etc/sssd/sssd.conf` file:

```
[sssd]
config_file_version = 2
domains = intra.it
services = nss, pam

[domain/intra.it]
# Uncomment if you need offline logins
# cache_credentials = true

id_provider = ad
auth_provider = ad
access_provider = ad

# Uncomment if service discovery is not working
# ad_server = adserver.intra.it

# Uncomment if you want to use POSIX UIDs and GIDs set on the AD side
# ldap_id_mapping = False

# Uncomment if the trusted domains are not reachable
#ad_enabled_domains = intra.it

# Comment out if the users have the shell and home dir set on the AD side
default_shell = /bin/bash
fallback_homedir = /home/%d/%u

# Uncomment and adjust if the default principal SHORTNAME$@REALM is not available
# ldap_sasl_authid = host/client.intra.it@INTRA.IT

# Comment out if you prefer to use shortnames.
use_fully_qualified_names = True

# Uncomment if the child domain is reachable, but only using a specific DC
# [domain/intra.it/child.intra.it]
# ad_server = dc.child.intra.it
```

11. Make a backup of you configured file

```
cp /etc/sssd/sssd.conf /etc/sssd/sssd.conf.$(dnsdomainname)
```

12. Join your domain with the command following the pattern: `net ads join -U administrator@$DOMAIN -S $SERVERNAME.$DOMAIN`

```
net ads join -U administrator@intra.it -S adserver.intra.it

# result:
#Enter administrator@intra.it's password:
#Using short domain name -- INTRA
#Joined 'CLIENTLINUX' to dns domain 'intra.it'
```

You can leave the domain by using `net ads leave -U administrator@intra.it -S adserver.intra.it`

### Joinning the domain using windows machine ###

0. **warning** Control panel paths and button names might not be exactly the same for this session
   
1. Find your nertwork card adapter on windows (probably on `Control Panel\Network and Internet\network connections`)

2. Right click on your adapter and them click on `properties`

3. Select `Protocol IP v4` and then click `properties`

4. Configure a manual IP or leave it as DHCP (automatic)

5. Set the DNS with the ip of adserver (192.168.0.10 in this scenario)

6. Press `OK` -> `Close` -> `Close`

7. Ping on `adserver.intra.it` and it must work

8. Go to `Control Panel\Clock and Region`

9. Click on `Define date and time`

10.  Go to tab `Internet clock` and press `Change configurations`

11. On Server type the ip of adserver (192.168.0.10 in this scenario) and click `sync now`

12. Click on `Ok` and then on `Ok`

13. Go to windows explorer, right click on the computer and then click `Properties`

14. Click on `Change settings`

15. Click on `change` and them define:
>- The computer name (`clientwin` for this scenario)
>- The dns suffix (`intra.it` for this scenario)
>- The domain (`intra.it` for this scenario)

15. Press `Ok` and give the password for the `Administrator` user

16. Restart the computer

17. You are now on the domain
    
## Managing the server ##

**You can install windows administrative tools [RSAT](https://www.microsoft.com/en-us/download/details.aspx?id=45520) on clients to manage it**

**Use `samba-tool -h` to learn the tool**, you can also do it 'recursively' like `samba-tool domain -h` or `samba-tool user create -h`

### Password securiy requirements ###

>- You can see the password requirements by using `samba-tool domain passwordsettings show`
>- You can drop every requirement by using the followings **WARNING unsafe**

```
samba-tool domain passwordsettings set --complexity=off
samba-tool domain passwordsettings set --history-length=0
samba-tool domain passwordsettings set --min-pwd-age=0
samba-tool domain passwordsettings set --max-pwd-age=0
samba-tool domain passwordsettings set --min-pwd-length=0
```

### Manage groups ###

>- `samba-tool group add GROUP_NAME` to create a group
>- `samba-tool group list` to list groups

### Manage users ###

>- `samba-tool user create USERNAME PASSWORD` to create a user with a password **WARNING unsafe** but scriptable

>- `samba-tool user create USERNAME` to create a user, you need to type the password after

>- `samba-tool user list` to list users

### To bind users and groups ###

>- `samba-tool group addmembers GROUPNAME USERNAME` to bind a user to a group

>- `samba-tool group remove members GROUPNAME USERNAME` to remove a user from a group

>- `samba-tool group listmembers GROUPNAME` to list users from a group

