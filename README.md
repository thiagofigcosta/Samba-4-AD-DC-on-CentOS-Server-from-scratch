# How to install Samba 4 AD DC on CentOS 8 and connect Windows and Linux clients #


How to install Active Directory on CentOS 8 Server and connect Windows and Linux machines as clients
We also are going to show how to install and setup DNS, DHCP and NTP (required for AD)

## Requirements ##

>- [CentOS 8](http://miroir.univ-lorraine.fr/centos/8.0.1905/isos/x86_64/CentOS-8-x86_64-1905-dvd1.iso)
>- [Samba 4](https://download.samba.org/pub/samba/stable/samba-4.11.2.tar.gz)
>- [Windows 10](https://www.microsoft.com/software-download/windows10)


## My environment ##

Im testing on virtual environment with VirtualBox](https://www.virtualbox.org/wiki/Downloads)

Every machine is attached to 'NAT Network' on virtual box, this network is 192.168.0.0/24

Machine adserver = 2048 MB of RAM
Machine clientlinux = 3096 MB of RAM
Machine client_win = 3096 MB of RAM

My domain will be `intra.it`

Linux username = toto
Linux password = toto
Linux root password = toto
samba administrator password = Toto123

**On virtual box with NAT network the gateway is the first address of the network, in this scenario is 192.168.0.1**

## Configuring the Server ##

>- Install virtual box, and create one machine for the server, one for the Linux client and one for Windows client
>- Configue the network to NAT Network
>- Enable bidirecional clipboard on the machines, it can be very useful
>- Install install CentOS and Windows on its own machines


### Starting ###
1. Fill safer by taking snapshots of the machine after important steps
2. Login as root on a terminal **every following step is made on root user**

```
sudo su
```

2. Add the EPEL repository:

```
yum install -y epel-release
```

3. Update all packets (optional):

```
yum update
```

### Configuring the computer ###

1. Configure your network card to have fixed IP, in my case I'll use **192.168.0.10/24** as my IP, and as I said before, **192.168.0.1/24** as gateway
tip: on Linux usually the network file is on /etc/network/interfaces, but in centos 8 its on /etc/sysconfig/network-scripts/ifcfg-`interface_name`

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

3. Add the network range that you want to allow on chrony files, you may need to add other networks in this file for other scenarios. The format is `$network_ip/$network_mask`

```
printf "\nallow 192.168.0.0/16\n" >> /etc/chrony.conf # i used /16 to allow the hole private range
```

4. Make sure that your server is going to serve the clients even if now other sync providing time to it, to ensure that every computer have the same clock.

```
printf "\n\nlocal stratum 10\n" >> /etc/chrony.conf
```

5. Restart chrony service

```
systemctl restart chronyd
```

6. Permit chrony on firewall and then reload it

```
firewall-cmd --permanent --add-service=ntp # open ntp on firewall
firewall-cmd --reload # reload firewall
```

### Setting up DHCP server ###

1. Install DHCP
   
```
yum install -y dhcp-server
```

2. Make a backup of your `/etc/dhcp/dhcpd.conf` file and delete the old one:

```
mv /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.raw
```

3. Use the following configuration file, raplacing the range, network, gateway, mask, domain, and dns server

```
printf "option domain-name \"$(hostname |cut -d. -f2-)\"; # domain\noption domain-name-servers $(hostname); # dns server\ndefault-lease-time 36000; # voluntary duration of IP in seconds\nmax-lease-time 86400; # max duration of IP in seconds\nauthoritative; # official server\n\nsubnet 192.168.0.0 netmask 255.255.255.0 {\n   option routers 192.168.0.1; # gateway\n\n   range 192.168.0.11 192.168.0.254; # range distributed to clients\n}\n" > /etc/dhcp/dhcpd.conf
```

>>- This is how the `/etc/dhcp/dhcpd.conf` file looks like: 
```
option domain-name "intra.it"; # domain
option domain-name-servers adserver.intra.it; # dns server
default-lease-time 36000;    # voluntary duration of IP in seconds
max-lease-time 86400; # max duration of IP in seconds
authoritative; # official server

subnet 192.168.0.0 netmask 255.255.255.0 {
    option routers 192.168.0.1; # gateway

    range 192.168.0.11 192.168.0.254; # range distributed to clients
}
```

4. Start and enable it:
   
```
systemctl start dhcpd
systemctl enable dhcpd
```

5. Allow it on firewall:

```
firewall-cmd --add-port=67/udp --permanent
firewall-cmd --reload
```

### Instaling samba ###

1. Disable SElinux and reboot

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
yum install -y bind cups-devel docbook-style-xsl gcc gdb gnutls-devel gpgme-devel jansson-devel keyutils-libs-devel krb5-workstation libacl-devel libaio-devel libarchive-devel libattr-devel libblkid-devel libtasn1 libtasn1-tools libxml2-devel libxslt lmdb-devel openldap-devel pam-devel perl perl-ExtUtils-MakeMaker perl-Parse-Yapp popt-devel python3-cryptography python3-dns python3-gpg python36-devel readline-devel rpcgen systemd-devel tar zlib-devel 
```

3. Download samba, uncompress it and enter its folder

```
wget https://download.samba.org/pub/samba/stable/samba-4.11.2.tar.gz # download
tar -zxvf samba-4.11.2.tar.gz # uncompress
cd samba-4.11.2 # enter
```

4. Compile and install it. **This might take a while**

```
./configure --enable-debug 
make && make install
```

5. Add samba binaries to you PATH variable, so that it'll to be able to run its commands on terminal, this is done adding `export PATH=$PATH:/usr/local/samba/bin/:/usr/local/samba/sbin/` to the end of the file .bash_profile of our user (works for terminals opened after this command), and running it once (works for the current session).

```
printf "\nexport PATH=$PATH:/usr/local/samba/bin/:/usr/local/samba/sbin/\n" >> /root/.bash_profile # for root futher sessions
export PATH=$PATH:/usr/local/samba/bin/:/usr/local/samba/sbin/ # for this session
```

6. Make a backup of your samba file

```
cp /etc/samba/smb.conf /etc/samba/smb.conf.raw
```

7. Edit /etc/samba/smb.conf setting:


```
PCNAME=$(echo $(hostname | cut -d. -f1))
WORKNAME=$(echo $(hostname | cut -d. -f2))
DOMNAME=$(echo $(hostname |cut -d. -f2-))

printf "[global]\n   server role = active directory domain controller\n   netbios name = ${PCNAME^^}\n   workgroup = ${WORKNAME^^}\n   realm = ${DOMNAME^^}\n   security = user\n   passdb backend = tdbsam\n   printing = cups\n   printcap name = cups\n   load printers = yes\n   cups options = raw\n   kerberos method = system keytab\n   server services = -dns\n   idmap_ldb:use rfc2307 = yes\n   tls enabled = yes\n   tls keyfile = tls/key.pem\n   tls cafile = tls/ca.pem\n   tls certfile = tls/cert.pem\n\n[netlogon]\n   comment = Logon Service\n   path = /home/samba/netlogon\n   write list = root\n\n\n[homes]\n   comment = Home Directories\n   valid users = %%S, %%D%%w%%S\n   browseable = No\n   read only = No\n   inherit acls = Yes\n\n[printers]\n   comment = All Printers\n   path = /var/tmp\n   printable = Yes\n   create mask = 0600\n   browseable = No\n\n[print\$]\n   comment = Printer Drivers\n   path = /var/lib/samba/drivers\n   write list = @printadmin root\n   force group = @printadmin\n   create mask = 0664\n" > /etc/samba/smb.conf
```

   <!-- verificar: https://blog.godatadriven.com/samba-configuration -->
   >- After [global] session:
   >>- Set workgroup equal to your SLD (first part of domain) **in uppercase**
   >>- Add the following line `server role = active directory domain controller`
   >>- Add the following line `netbios name = ADSERVER` which has the format `netbios name = $COMPUTERNAME` **in uppercase**
   >>- Add the following line `realm = INTRA.IT` which has the format `realm = $DOMAIN` **in uppercase**
   >>- Add the following line `kerberos method = system keytab`
   >>- Add the following line `server services = -dns` **to work with bind dns instead of samba internal** 
   >>- Add the following line `idmap_ldb:use rfc2307 = yes`
   >>- Add the following line `tls enabled = yes`
   >>- Add the following line `tls keyfile = tls/key.pem`
   >>- Add the following line `tls cafile = tls/ca.pem`
   >>- Add the following line `tls certfile = tls/cert.pem`
   >>- Add the following line `[netlogon]`
   >>- Add the following line `comment = Logon Service`
   >>- Add the following line `write list = root`

After this step the [global] session should look like:

```
[global]
        server role = active directory domain controller
        netbios name = ADSERVER
        workgroup = INTRA
        realm = INTRA.IT
        security = user
        passdb backend = tdbsam
        printing = cups
        printcap name = cups
        load printers = yes
        cups options = raw
        kerberos method = system keytab
        server services = -dns
        idmap_ldb:use rfc2307 = yes
        tls enabled = yes
        tls keyfile = tls/key.pem
        tls cafile = tls/ca.pem
        tls certfile = tls/cert.pem

[netlogon]
        comment = Logon Service
        path = /home/samba/netlogon
        write list = root
```


8. Make a backup of your configured file

```
cp /etc/samba/smb.conf /etc/samba/smb.conf.$(dnsdomainname)
```

9. Make a backup of you dns file and delete the original one:
```
mv /etc/named.conf /etc/named.conf.raw
```

10. Use my file on the DNS, the changed lines from the original file are commented explaining their behaviour:

```
printf "options {\n   listen-on port 53 { 192.168.0.10; 127.0.0.1; }; // Here we are setting the server to listen on port 53 on ips: 192.168.0.10 and 127.0.0.1. USE YOUR IP\n   listen-on-v6 { none; }; // Disable IPv6\n   directory   \"/var/named\";\n   dump-file   \"/var/named/data/cache_dump.db\";\n   statistics-file \"/var/named/data/named_stats.txt\";\n   memstatistics-file \"/var/named/data/named_mem_stats.txt\";\n   secroots-file   \"/var/named/data/named.secroots\";\n   recursing-file   \"/var/named/data/named.recursing\";\n   allow-query    { 192.168.0.0/16 ;localhost; }; // Here we are saying that we accept requests of the network 192.168.0.0/16 THIS IS VERY IMPORTANT\n\n   recursion yes; // We keep it. since we are going to solve internal requests and forward external ones \n\n   empty-zones-enable no;\n\n   forwarders { 8.8.8.8; 8.8.4.4; 130.190.190.2; 130.190.190.3; }; // List of dns servers to forward requests that couldn't be solved internally (external requests) YOU MAY WANT TO FORWARD TO ANOTHER DNS LOCAL SERVER\n\n   tkey-gssapi-keytab \"/usr/local/samba/private/dns.keytab\"; // This line is needed for samba\n   minimal-responses yes; // This line is needed for samba\n\n   pid-file \"/run/named/named.pid\";\n   session-keyfile \"/run/named/session.key\";\n   include \"/etc/crypto-policies/back-ends/bind.config\";\n};\n\nlogging {\n      channel default_debug {\n            file \"data/named.run\";\n            severity dynamic;\n      };\n};\n\nzone \".\" IN {\n   type hint;\n   file \"named.ca\";\n};\n\ninclude \"/etc/named.rfc1912.zones\";" > /etc/named.conf
```

>>- Here is how the `/etc/named.conf` file should look like:

```
options {
   listen-on port 53 { 192.168.0.10; 127.0.0.1; }; // Here we are setting the server to listen on port 53 on ips: 192.168.0.10 and 127.0.0.1. USE YOUR IP
   listen-on-v6 { none; }; // Disable IPv6
   directory   "/var/named";
   dump-file   "/var/named/data/cache_dump.db";
   statistics-file "/var/named/data/named_stats.txt";
   memstatistics-file "/var/named/data/named_mem_stats.txt";
   secroots-file   "/var/named/data/named.secroots";
   recursing-file   "/var/named/data/named.recursing";
   allow-query    { 192.168.0.0/16 ;localhost; }; // Here we are saying that we accept requests of the network 192.168.0.0/16 THIS IS VERY IMPORTANT

   recursion yes; // We keep it. since we are going to solve internal requests and forward external ones 

   empty-zones-enable no;

   forwarders { 8.8.8.8; 8.8.4.4; 130.190.190.2; 130.190.190.3; }; // List of DNS servers to forward requests that couldn't be solved internally (external requests) YOU MAY WANT TO FORWARD TO ANOTHER DNS LOCAL SERVER

   tkey-gssapi-keytab "/usr/local/samba/private/dns.keytab"; // This line is needed for samba
   minimal-responses yes; // This line is needed for samba

   pid-file "/run/named/named.pid";
   session-keyfile "/run/named/session.key";
   include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
      channel default_debug {
            file "data/named.run";
            severity dynamic;
      };
};

zone "." IN {
   type hint;
   file "named.ca";
};

include "/etc/named.rfc1912.zones";
```

11. Lets include needed records for AD
>>- First we include a zone file on named file
```
printf "\nzone \"$(hostname -d)\" IN {\n   type master;\n   file \"ad.$(hostname -d)\";\n   allow-update { 192.168/16; }; // to allow DDNS on network 192.168/16\n   check-names ignore; // to not be restrictive with DDNS\n};\n" >> /etc/named.conf
```

We are addind the following to the `/etc/named.conf`:
```
zone "intra.it" IN {
   type master;
   file "ad.intra.it";
   allow-update { 192.168/16; }; // to allow DDNS on network 192.168/16
   check-names ignore; // to not be restrictive with DDNS
};
```

>>- Second we create the zone file with records needed to AD:

```
SERVERIP="192.168.0.10" # SET YOUR SERVER IP HERE

printf ";;; Original source: https://cromwell-intl.com/open-source/samba-active-directory/dns.html\n;;; Ref: https://technet.microsoft.com/en-us/library/dd316373.aspx https://blogs.msdn.microsoft.com/servergeeks/2014/07/12/dns-records-that-are-required-for-proper-functionality-of-active-directory/\n;;; SRV format:\n;;; _service._protocol.DNSdomain IN SRV priority weight port target\n\n\n\n\$TTL 86400 ; 24 hours could have been written as 24h or 1d\n\$ORIGIN $(hostname -d).\n@  1D  IN  SOA $(hostname). thiagofigcosta.$(hostname -d). (\n               20191126 ; serial. i used a date yyyyMMdd\n               3H ; refresh\n               15 ; retry\n               1w ; expire\n               3h ; nxdomain ttl\n              )\n       IN  NS     $(hostname). ; in the domain\n$(hostname -s)        IN   A       $SERVERIP\ndc              IN   CNAME   $(hostname -s)\n\n\n_ldap._tcp       IN   SRV   0 0 389 $(hostname -s)\n_ldap._udp       IN   SRV   0 0 389 $(hostname -s)\n_kerberos._tcp  IN   SRV   0 0  88 $(hostname -s)\n_kerberos._udp  IN   SRV   0 0  88 $(hostname -s)\n_kpasswd._tcp   IN   SRV   0 0 464 $(hostname -s)\n_kpasswd._udp   IN   SRV   0 0 464 $(hostname -s)\n_kerberos-master._tcp  IN   SRV   0 0  88 $(hostname -s)\n_kerberos-master._udp  IN   SRV   0 0  88 $(hostname -s)\n\n\n; The following are used to support kadmin remotely.  Not needed if we will only do local administration.\n_kerberos-adm._ucp   IN   SRV   0 0 749 $(hostname -s)\n_kerberos-adm._tcp   IN   SRV   0 0 749 $(hostname -s)\n\n; Probably the following exists to be compliant with old\n_ldap._tcp.pdc._msdcs   IN   SRV   0 0 389 $(hostname -s)\n_ldap._tcp.gc._msdcs   IN   SRV   0 0 389 $(hostname -s)\n_ldap._tcp.dc._msdcs   IN   SRV   0 0 389 $(hostname -s)\ngc._msdcs              IN   A   $SERVERIP\n" > /var/named/ad.$(hostname -d)
```


>>- This is how the `/var/named/ad.intra.it` file should look like:
```
;;; Original source: https://cromwell-intl.com/open-source/samba-active-directory/dns.html
;;; Ref: https://technet.microsoft.com/en-us/library/dd316373.aspx https://blogs.msdn.microsoft.com/servergeeks/2014/07/12/dns-records-that-are-required-for-proper-functionality-of-active-directory/
;;; SRV format:
;;; _service._protocol.DNSdomain IN SRV priority weight port target


$TTL 86400 ; 24 hours could have been written as 24h or 1d
$ORIGIN intra.it.
@  1D  IN  SOA adserver.intra.it. thiagofigcosta.intra.it. (
			      20191126 ; serial. i used a date yyyyMMdd
			      3H ; refresh
			      15 ; retry
			      1w ; expire
			      3h ; nxdomain ttl
			     )
       IN  NS     adserver.intra.it. ; in the domain

adserver        IN	A	    192.168.0.10
dc              IN	CNAME	adserver


_ldap._tcp	    IN	SRV	0 0 389 adserver
_ldap._udp	    IN	SRV	0 0 389 adserver
_kerberos._tcp  IN	SRV	0 0  88 adserver
_kerberos._udp  IN	SRV	0 0  88 adserver
_kpasswd._tcp   IN	SRV	0 0 464 adserver
_kpasswd._udp   IN	SRV	0 0 464 adserver
_kerberos-master._tcp  IN   SRV   0 0  88 adserver
_kerberos-master._udp  IN   SRV   0 0  88 adserver

; The following are used to support kadmin remotely.  Not needed if we will only do local administration.
_kerberos-adm._ucp	IN	SRV	0 0 749 adserver
_kerberos-adm._tcp	IN	SRV	0 0 749 adserver

; Probably the following exists to be compliant with old
_ldap._tcp.pdc._msdcs	IN	SRV	0 0 389 adserver
_ldap._tcp.gc._msdcs	IN	SRV	0 0 389 adserver
_ldap._tcp.dc._msdcs	IN	SRV	0 0 389 adserver
gc._msdcs		        IN	A	192.168.0.10
```


12. Make a backup of your configured file:

```
cp /etc/named.conf /etc/named.conf.$(dnsdomainname)
```

13. Fix some permissions: **THIAGO explicitly set other files permissions**

```
chmod 770 /usr/local/samba/bind-dns
chmod 644 /etc/krb5.conf
```

14. Enable bind at boot and start it:

```
systemctl enable named
systemctl start named
```


15. Use samba-tool to configure your samba ad dc, with the following info. **You can do it passing values as arguments or typing then in a interactive way**

>>- Interactive:

```
samba-tool domain provision --use-rfc2307 --interactive

#Realm: INTRA.IT **uppercase**
#Domain: INTRA **uppercase**
#Server role: dc # domain controller
#DNS backend: BIND9_DLZ
``` 

>>- As arguments:

```
samba-tool domain provision --use-rfc2307 --realm=INTRA.IT --domain=INTRA --server-role=dc --dns-backend=BIND9_DLZ --adminpass='Toto123'
``` 

16. Check that everything's worked

```
samba-tool domain level show

# result:
#Domain and forest function level for domain 'DC=intra,DC=it'
#
#Forest function level: (Windows) 2008 R2
#Domain function level: (Windows) 2008 R2
#Lowest function level of a DC: (Windows) 2008 R2
```

17.   Allow samba on your firewall and then reload it

```
firewall-cmd --add-port=53/tcp --permanent;firewall-cmd --add-port=53/udp --permanent; #enable DNS
firewall-cmd --add-port=88/tcp --permanent;firewall-cmd --add-port=88/udp --permanent; #enable Kerberos_auth(88
firewall-cmd --add-port=135/tcp --permanent; # enable Microsoft EPMAP
firewall-cmd --add-port=137-138/udp --permanent;firewall-cmd --add-port=139/tcp --permanent; # enable NETBIOS
firewall-cmd --add-port=389/tcp --permanent;firewall-cmd --add-port=389/udp --permanent; # enable LDAP
firewall-cmd --add-port=445/tcp --permanent; # enable Active Directory shares
firewall-cmd --add-port=464/tcp --permanent;firewall-cmd --add-port=464/udp --permanent; # enable Kerberos passwd
firewall-cmd --add-port=636/tcp --permanent; # enable LDAP crypt
firewall-cmd --add-port=3268-3269/tcp --permanent # enable microsoft  global catalog
firewall-cmd --reload # reload
```

18.  Create a samba service file, fix its permissions, set it to run on boot and start it

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

The file `samba_ad.service` should look like this:

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

19. **Optional**: since I'm using virtual machines, and I have low RAM memory I would like to disable the graphical mode of my server. Linux systems have the following init values: (after this step if needed you can low down the server RAM to 1536 MB)

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

### Starting ###
1. Fill safer by taking snapshots of the machine after important steps
2. Login as root on a terminal **every following step is made on root user**

```
sudo su
```

2. Add the EPEL repository:

```
yum install -y epel-release
```

3. Update all packets (optional):

```
yum update
```

### Configuring the computer ###

1. Configure your network card to have a fixed IP (e.g. **192.168.0.11/24**) or leave it as DHCP, **set the DNS to your server IP, in my scenario 192.168.0.10** and as I said before, **192.168.0.1/24** as gateway
   
2. Set your computer name and your domain, following the patern `hostnamectl set-hostname --static "$COMPUTERNAME.$DOMAIN"`, **choose a DNS valid name for the computer**

```
hostnamectl set-hostname --static "clientlinux.intra.it"
```

3. Modify /etc/hosts, adding a line with your ad server in the format: `$IP $SERVERNAME.$DOMAIN $SERVERNAME` and remove the line that starts with `127.0.0.1` and add instead the a line with your localhost in the format: `127.0.0.1 $COMPUTERNAME.$DOMAIN $COMPUTERNAME`:

```
sed -i '/^127.0.0.1/ d' /etc/hosts
sed -i '1s/^/192.168.0.10 adserver.intra.it adserver\n127.0.0.1 clientlinux.intra.it localhost clientlinux\n/' /etc/hosts
```

The `/etc/hosts` file should look like this:

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

2. Edit the file /etc/chrony.conf to add an entry to the ad server in the format `Server $SERVERIP_OR_NAME` and to remove other servers commenting the line `pool 2.centos.pool.ntp.org iburst`

```
echo "Server adserver.intra.it" >> /etc/chrony.conf 
sed -i -e '/pool 2.centos.pool.ntp.org iburst/ s/^#*/#/' /etc/chrony.conf
```

3. Force the synchronization, stopping the chrony service first and then sync with the command following the pattern `chronyd -q 'server $SERVERIP iburst'`:
```
systemctl stop chronyd
chronyd -q 'server adserver.intra.it iburst'
systemctl start chronyd
```

4. Enable chrony to launch at boot and start it

```
systemctl enable chronyd # enable chrony
systemctl start chronyd # start chrony
```

5. Check if your server is on the sources of chrony

```
chronyc sources

# result:
#210 Number of sources = 1
#MS Name/IP address         Stratum Poll Reach LastRx Last sample               
#===============================================================================
#^? adserver.intra.it            10   6     1     2   -222ms[ -222ms] +/-  307us
```

### Joinning the domain using Linux machine ###

1. Install Samba and Kerberos:

``` 
yum install -y samba samba-winbind-krb5-locator samba-common samba-common-tools samba-winbind krb5-workstation samba-winbind-clients oddjob-mkhomedir
```

2. Make a backup of your Kerberos file and delete the original file

```
mv /etc/krb5.conf /etc/krb5.conf.raw
```

3. Create a /etc/krb5.conf file with the contents of the file bellow, change every `INTRA.IT` by your domain and in **uppercase**, change every `intra.it` by your domain in lowercase, and change every `adserver` by your ad server name

```
printf "[logging]\ndefault = FILE:/var/log/krb5libs.log\nkdc = FILE:/var/log/krb5kdc.log\nadmin_server = FILE:/var/log/kadmind.log\n\n[libdefaults]\ndns_lookup_realm = true\ndns_lookup_kdc = true\nticket_lifetime = 24h\nrenew_lifetime = 7d\nforwardable = true\nrdns = false\ndefault_realm = INTRA.IT\n\n[realms]\n# Uncomment following if DNS lookups are not working\n# INTRA.IT = {\n# kdc = adserver.intra.it\n# master_kdc = adserver.intra.it\n# admin_server = adserver.intra.it\n# }\n\n[domain_realm]\n# Uncomment following if DNS lookups are not working\n# .intra.it = INTRA.IT\n# intra.it = INTRA.IT\n" > /etc/krb5.conf
```

The file `/etc/krb5.conf`:

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

4. Test your kerberos communication, replace `@INTRA.IT` by `@$YOURDOMAIN` **UPPERCASE**, the administrator password will be required

```
KRB5_TRACE=/dev/stdout kinit -V administrator@INTRA.IT

# last line result:
#Authenticated to Kerberos v5
```

And check again:
```
klist

# result:
#Ticket cache: FILE:/tmp/krb5cc_0
#Default principal: administrator@INTRA.IT
#
#Valid starting       Expires              Service principal
#11/26/2019 15:48:50  11/27/2019 01:48:50  krbtgt/INTRA.IT@INTRA.IT
#	renew until 12/03/2019 15:48:47
```

5. Make a backup of your configured file

```
cp /etc/krb5.conf /etc/krb5.conf.$(dnsdomainname)
``` 

6. Make a backup of your samba file and delete the original file

```
mv /etc/samba/smb.conf /etc/samba/smb.conf.raw
```

7. Create a /etc/samba/smb.conf file with the contents of the file bellow, change every `INTRA.IT` by your domain and in **uppercase**, change every `INTRA` by your SLD (first part of domain) and in **uppercase**

```
printf "[global]\n   security = ads\n   realm = INTRA.IT\n   password server = adserver.intra.it\n   workgroup = INTRA\n   idmap uid = 10000-20000\n   idmap gid = 10000-20000\n   winbind enum users = yes\n   winbind enum groups = yes\n   template homedir = /home/%%D/%%U\n   template shell = /bin/bash\n   client use spnego = yes\n   client ntlmv2 auth = yes\n   encrypt passwords = yes\n   winbind use default domain = yes\n   restrict anonymous = 2\n   domain master = no\n   local master = no\n   preferred master = no\n   os level = 0\n" > /etc/samba/smb.conf
```

The file `/etc/samba/smb.conf`: 

```
[global]
   security = ads
   realm = INTRA.IT
   password server = adserver.intra.it
   workgroup = INTRA
   idmap uid = 10000-20000
   idmap gid = 10000-20000
   winbind enum users = yes
   winbind enum groups = yes
   template homedir = /home/%D/%U
   template shell = /bin/bash
   client use spnego = yes
   client ntlmv2 auth = yes
   encrypt passwords = yes
   winbind use default domain = yes
   restrict anonymous = 2
   domain master = no
   local master = no
   preferred master = no
   os level = 0
```

8. Make a backup of your configured file

```
cp /etc/samba/smb.conf /etc/samba/smb.conf.$(dnsdomainname)
```

9. Join your domain with the command following the pattern: `net ads join -U administrator@$DOMAIN -S $SERVERNAME.$DOMAIN` **domain in UPPERCASE**

```
net ads join -U administrator@INTRA.IT -S adserver.intra.it

# result:
#Enter administrator@INTRA.IT's password:
#Using short domain name -- INTRA
#Joined 'CLIENTLINUX' to dns domain 'INTRA.IT'
```

10. Check the network users available:

```
wbinfo -u

# result:
#INTRA\dns-adserver
#INTRA\guest
#INTRA\krbtgt
#INTRA\administrator
```

```
net ads info 

# result:
#LDAP server: 192.168.0.10
#LDAP server name: adserver.intra.it
#Realm: INTRA.IT
#Bind Path: dc=INTRA,dc=IT
#LDAP port: 389
#Server time: Sat, 30 Nov 2019 14:01:20 CET
#KDC server: 192.168.0.10
#Server time offset: 11
#Last machine account password change: Sat, 30 Nov 2019 13:37:13 CET
```

You can leave the domain by using `net ads leave -U administrator@intra.it -S adserver.intra.it`

11. Restart samba and winbind, then ensure that then start at boot:

```
systemctl stop smb
systemctl restart winbind.service
systemctl start smb
systemctl enable winbind.service
systemctl enable smb
```

12. Create a /etc/sssd/sssd.conf file with the contents of the file bellow, change every `intra.it` by your domain in lowercase, change every `INTRA.IT` by your domain in **uppercase**, and fix its permissions
    
```
printf "[sssd]\nconfig_file_version = 2\ndomains = intra.it\nservices = nss, pam, pac\n\n[domain/adserver.intra.it]\n# Uncomment if you need offline logins\n# cache_credentials = true\n\nid_provider = adserver\nauth_provider = adserver\naccess_provider = adserver\n\n# Uncomment if service discovery is not working\n# ad_server = adserver.intra.it\n\n# Uncomment if you want to use POSIX UIDs and GIDs set on the AD side\n# ldap_id_mapping = False\n\n# Uncomment if the trusted domains are not reachable\n#ad_enabled_domains = intra.it\n\n# Comment out if the users have the shell and home dir set on the AD side\ndefault_shell = /bin/bash\nfallback_homedir = /home/%%d/%%u\n\n# Uncomment and adjust if the default principal SHORTNAME$@REALM is not available\n# ldap_sasl_authid = host/client.intra.it@INTRA.IT\n\n# Comment out if you prefer to use shortnames.\nuse_fully_qualified_names = True\n\n# Uncomment if the child domain is reachable, but only using a specific DC\n# [domain/intra.it/child.intra.it]\n# ad_server = dc.child.intra.it\n" > /etc/sssd/sssd.conf

chmod 0600 /etc/sssd/sssd.conf
```

The `/etc/sssd/sssd.conf` file:

```
[sssd]
config_file_version = 2
domains = intra.it
services = nss, pam, pac

[domain/adserver.intra.it]
# Uncomment if you need offline logins
# cache_credentials = true

id_provider = adserver
auth_provider = adserver
access_provider = adserver

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

13. Make a backup of your configured file

```
cp /etc/sssd/sssd.conf /etc/sssd/sssd.conf.$(dnsdomainname)
```

14. Start and enable it:

```
systemctl start sssd.service
systemctl enable sssd.service
```

15. Make a copy of your `/etc/nsswitch.conf` file:
```
cp /etc/nsswitch.conf /etc/nsswitch.conf.raw
```

16.  Change Name Service Switch file `/etc/nsswitch.conf` to retrieve data about users, groups and passwords, adding `compat winbind` after lines with: `passwd:` and `passwd:`, and adding `compat` after lines with: `shadow:`:

<!-- ```
printf "\n\npasswd: compat winbind\ngroup: compat winbind\nshadow: compat\n" >> /etc/nsswitch.conf
``` -->

>>- Partial content of `/etc/nsswitch.conf`:

```
passwd: compat winbind
group: compat winbind
shadow: compat
```

17. Check if it worked:

```
getent passwd | grep administrator && getent group | grep "domain users"

# result:
#administrator:*:10003:10006::/home/INTRA/administrator:/bin/bash
#domain users:x:10006:
```


18. **WARNING: WORK IN PROGRESS** Change the login file `/etc/pam.d/login` to handle authentication, adding the following lines:

**GUI LOGIN FILE: /etc/pam.d/gdm-password**

```
printf "\n\n\nauth sufficient   pam_winbind.so\nauth sufficient   pam_unix.so nullok_secure use_first_pass\nauth required   pam_deny.so\naccount sufficient   pam_winbind.so\naccount required   pam_unix.so\nsession required   pam_unix.so\nsession required   pam_mkhomedir.so umask=0022 skel=/etc/skel\n" >> /etc/pam.d/login
```

>>- The lines:
```
auth sufficient      pam_winbind.so
auth sufficient      pam_unix.so nullok_secure use_first_pass
auth required        pam_deny.so
account sufficient   pam_winbind.so
account required     pam_unix.so
session required     pam_unix.so
session required     pam_mkhomedir.so umask=0022 skel=/etc/skel
```

19. **WARNING: WORK IN PROGRESS** Change the sudo file `/etc/pam.d/sudo` to handle permissions. adding the following lines:

```
printf "\n\nauth sufficient   pam_winbind.so\nauth sufficient   pam_unix.so use_first_pass\nauth required   pam_deny.so\n\n@include common-account\n" >> /etc/pam.d/sudo
```

>>- The lines:
```
auth sufficient   pam_winbind.so
auth sufficient   pam_unix.so use_first_pass
auth required     pam_deny.so

@include common-account
```

20. Make your user home folder in the format `/home/$DOMAIN/$USER`, **domain in UPPERCASE** **FIX TO BE AUTOMATIC**

```
mkdir /home/INTRA/
mkdir /home/INTRA/administrator
```

21. Login. Togin with administrator requires root privileges, you can create another user and login on it without being root at first place. **WARNING: GUI LOGIN NOT WORKING YET, FIX PAM FILES, TO ALLOW GUI LOGIN**

```
su -l aministrator@intra.it
```

```
su -l toto@intra.it
```



### Joinning the domain using Windows machine ###
 
1. Go to `Control Panel\Network and Internet\Network Connections`, and find your nertwork card adapter on Windows 

2. Right click on your adapter and them click on `Properties`

3. Select `Internet Protocol Version 4 (TCP/IPv4)` and then click `Properties`

4. Configure the IP manually or leave it as DHCP (automatic)

5. Set the DNS with the IP of adserver (192.168.0.10 in this scenario)

6. Press `OK` -> `Close` -> `Close`

7. Ping on `adserver.intra.it` and it should work

8. Go to `Control Panel\Clock and Region`

9.  Click on `Set the time and date`

10.  Go to tab `Internet Time` and press `Change settings...`

11. On Server put `adserver.intra.it` (the AD server name or OP) and click `Update now`

12. Click on `OK` and then on `OK`

13. Go to `Control Panel\All Control Panel Items\System`

14. Inside `Computer name, domain, and workgroup settings` click on `Change settings`

15. Click on `Change..` and then define:
>- The computer name (`clientwin` for this scenario)
>- The DNS suffix (`intra.it` for this scenario), to find this option click on `More..`
>- The domain (`intra.it` for this scenario)

16. Press `OK` and give the password for the `Administrator` user

17. Restart the computer

18. You are now on the domain
   
19. Login :D

20. Install RSAT and go to `Control Panel\System and Security\Administrative Tools` to manage it. **`Active Directory administrative Center` not working yet, but the others do**
    
## Managing the server ##

**You can install Windows Administrative tools [RSAT](https://www.microsoft.com/en-us/download/details.aspx?id=45520) on clients to manage it**

**Use `samba-tool -h` to learn the tool**, you can also do it 'recursively' like `samba-tool domain -h` or `samba-tool user create -h`

### Password security requirements ###

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

>- `samba-tool user create USERNAME PASSWORD` to create a user with a pre-given password **WARNING unsafe, but scriptable**

>- `samba-tool user create USERNAME` to create a user, you need to type the password after

>- `samba-tool user list` to list users

### To bind users to groups ###

>- `samba-tool group addmembers GROUPNAME USERNAME` to bind a user to a group

>- `samba-tool group remove members GROUPNAME USERNAME` to remove a user from a group

>- `samba-tool group listmembers GROUPNAME` to list users from a group

### [Changing DNS backend](https://wiki.samba.org/index.php/Changing_the_DNS_Back_End_of_a_Samba_AD_DC) ###

#### Changing From the Samba Internal DNS Server to the BIND9_DLZ Back End ####

1. Install bind with: 

```
yum install -y named
``` 

2. configure bind (**named.conf shown above**)

3. add `-dns` on `server services` to your /etc/samba/smb.conf, to be something like `server services = -dns` and shutdown samba

```
systemctl stop samba_ad
```

4. Start bind and enable it at boot 

```
systemctl enable named
systemctl start named
```
5. Run the following command
   
```
samba_upgradedns --dns-backend=BIND9_DLZ
```

6. Start samba

```
systemctl start samba_ad
```

#### Changing From the BIND9_DLZ Back End to the Samba Internal DNS Server ####

1. Stop bind and samba, them disable bind at boot 

```
systemctl stop samba_ad
systemctl stop named
systemctl disable named
```

2. Add `dns` on `server services` to your /etc/samba/smb.conf, to be something like `server services = dns`

3. Run the following command
   
```
samba_upgradedns --dns-backend=SAMBA_INTERNAL
```

4. Start samba

```
systemctl start samba_ad
```
