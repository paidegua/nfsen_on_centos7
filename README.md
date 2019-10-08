# How to Install NFSEN on CENTOS 7

This procedure witll help you get NFSEN running on CentOS 7. Start with a minimal installation of CentOS 7, update it and then proceed with the below steps.

## Start with a basic minimal installation of CentOS 7

Change "SELINUX=enforcing" to --> "SELINUX=disabled"

```
vi /etc/selinux/config
```

Reboot to apply the SELINUX Changes:

```
reboot 
```

Run "sestatus" to verify selinux is disabled. The status should say the following: "SELinux status: disabled"

```
sestatus
```

Stop iptables firewall:

```
systemctl stop firewalld 
systemctl disable firewalld
```

Run the following to verify the firewall status. It should say: "not running"

```
firewall-cmd --state
```

## START the NFSEN APPLICATION SETUP

We will need to install a number of packages for CentOS 7 to get NFSEN running

```
yum install -y httpd php wget gcc make rrdtool-devel rrdtool-perl perl-MailTools perl-Socket6 flex byacc perl-Sys-Syslog perl-Data-Dumper autoconf automake apache php perl-MailTools rrdtool-perl perl-Socket6 perl-Sys-Syslog.x86_64 policycoreutils-python tcpdump

echo "date.timezone = America/Denver" > /etc/php.d/timezone.ini yum update -y
```

Create the netflow user account and add it to the apache group:

```
useradd netflow usermod -a -G apache netflow
```

Create the directories which we will specify later in the configuration file:

```
mkdir -p /data/nfsen 
mkdir -p /var/www/html/nfsen
```

Download the latest nfdump and nfsen packages. At time of this writing the latest versions are nfdump-1.6.13.tar.gz and nfsen-1.3.6p1.tar.gz

```
cd /opt/ 
wget http://downloads.sourceforge.net/project/nfdump/stable/nfdump-1.6.13/nfdump-1.6.13.tar.gz 
wget http://downloads.sourceforge.net/project/nfsen/stable/nfsen-1.3.6p1/nfsen-1.3.6p1.tar.gz
```

Start the httpd service:

```
service httpd start
```

## Install nfdump, untar the downloaded nfdump package into the "/opt/" Directory.

```
tar -zxvf nfdump-1.6.13.tar.gz cd nfdump-1.6.13
```

Compile nfdump while in the "/opt/nfdump-1.6.13" directory:

```
./configure --prefix=/opt/nfdump --enable-nfprofile --enable-nftrack --enable-sflow 
autoreconf 
make && sudo make install
```

## Install and configure nfsen

Untar nfsen into the "/opt/" directory.

```
cd .. 
tar -zxvf nfsen-1.3.6p1.tar.gz 
cd nfsen-1.3.6p1 
cd etc 
cp nfsen-dist.conf nfsen.conf 
vi nfsen.conf
```

Edit the nfsen.conf file with at least the changes below. Make sure all data path variables are set correctly: 

```
$BASEDIR= "/data/nfsen";

$HTMLDIR = "/var/www/nfsen"; 'change to' --> $HTMLDIR = "/var/www/html/nfsen";

$PREFIX = '/usr/local/bin'; 'change to' --> '$PREFIX = '/opt/nfdump/bin';

$WWWUSER = "www"; 'change to' --> $WWWUSER = "apache"; $WWWGROUP = "www"; 'change to' --> $WWWGROUP = "apache";
```

You can change the following if you know all the hosts:
Duplicate the 'router-1' line for each different host. Change the Name, Port and Color for each different line.

```
%sources = ( 
'router-1' => { 'port' => '9030', 'col' => '#0000ff', 'type' => 'netflow' }, 
'firewall-1' => { 'port' => '9031', 'col' => '#9093ff', 'type' => 'netflow' }, 
);
```

```
@plugins = ( # profile # module # [ '', 'demoplugin' ], [ '', 'flowdoh' ], )
```

Save the above changes

## We will now run the perl installation script to install nfsen (change directory):

```
cd .. ./install.pl etc/nfsen.conf
```

Perl to use: [/usr/bin/perl]

Press enter to accept the default path. You may get Errors since we did not configure any flows at this point.

Let's now create a startup script for the service

```
vi /etc/init.d/nfsen
```

Copy the below information into the new configuration file:

```
#!/bin/bash 
#! #chkconfig: - 50 50 
#description: nfsen

DAEMON=/data/nfsen/bin/nfsen

case "$1" in 
start) 
$DAEMON start 
;; 
stop) 
$DAEMON stop 
;; status) 
$DAEMON status 
;; 
restart) 
$DAEMON stop 
sleep 1 
$DAEMON start 
;; 
*) 
echo "Usage: $0 {start|stop|status|restart}" 
exit 1 
;; 
esac

exit 0 
```

Save the above file.

Make sure the script is executable

```
chmod +x /etc/init.d/nfsen
```

Start the nfsen deamon:

```
/etc/init.d/./nfsen start
```

Use the following to restart nfsen when necessary:

```
/etc/init.d/nfsen restart
```

## At this point you should be able to access nfsen at http://127.0.0.1/nfsen/nfsen.php

### You can ignore the following message when you connect to your NfSen URL @ http://x.x.x.x/nfsen/nfsen.php 

```
Backend version missmatch!
```

## Adding additional NETFLOW Senders:
On the nfsen server, add the following to nfsen.conf file: 

```
vi /data/nfsen/etc/nfsen.conf
```

You can set the color to something you would like by finding a HEX color identifyer at the following site: https://www.color-hex.com/

```
%sources = ( 
'CiscoRouter' => { 'port' => '2055', 'col' => '#0000ff', 'type' => 'netflow' }, 
'LinuxServer' => { 'port' => '9666', 'col' => '#ff5a00', 'type' => 'netflow' }, 
);
```

## Rebuild NFSEN after all settings changes:

```
cd /data/nfsen/bin/ ./nfsen reconfig

/etc/init.d/nfsen restart
```
 

## Troubleshooting: 

Use tcpdump to verify that flows are being received on the specified port. 

Run the below to list the device NICs:

```
ip link show 

tcpdump -i <NIC_IDENTIFIER> port <TCP_PORT_DEFINED_IN_SETUP>
```

Make sure that you see traffic on this port from required host.

With nfdump you can read flow collection files from command line

```
cd /data/nfsen/profiles-data/live/ /opt/nfdump/bin/nfdump -r "<your_file_name>"
```

## Make sure that your system data and php dates are set correctly. You may need to edit /etc/php.ini and adjust your date.timezone = "US/Eastern"

Check running fcapd processes:

```
ps axo command | grep '[n]fcapd'
```

Check which ports nfcapd is listening on:

```
ss -nutlp
```

## Installing Plugins:

### FLOWDOH:
```
cd /opt/ 
sudo wget https://sourceforge.net/projects/flowdoh/files/FlowDoh_1.0.2.tar.gz 
sudo tar -zxvf FlowDoh_1.0.2.tar.gz
```

Backend: (/data/nfsen/plugins/)

```
cd /opt/flowdoh/backend/ 
cp -avr flowdoh/ /data/nfsen/plugins/ 
cp flowdoh.pm /data/nfsen/plugins/ 
cd /data/nfsen/plugins/flowdoh 
cp flowdoh.conf.defaults flowdoh.conf 
cd /data/nfsen/plugins 
chmod 777 flowdoh 
chmod 777 flowdoh.pm
```

FrontEnd: (/var/www/html/nfsen/plugins/)

```
cd /opt/flowdoh/frontend/ 
cp flowdoh.php /var/www/html/nfsen/plugins/ 
cp -avr flowdoh/ /var/www/html/nfsen/plugins/ 
cd /var/www/html/nfsen/plugins 
chmod 777 flowdoh 
chmod 777 flowdoh.php
```

Restart NFSEN to add the FLOWDOH Plugin.

```
/etc/init.d/nfsen restart
```

## Authors

* **Lance Jeffery** - *Initial work* - [paidegua](https://github.com/paidegua)
