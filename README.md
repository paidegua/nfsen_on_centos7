# How to Install NFSEN on CENTOS 7

This procedure will help you get NFSEN and NFDUMP running on CentOS 7. Start with a minimal installation of CentOS 7, update it "yum update" to apply all the latest update. Then proceed with the below steps.

## Prepare the OS to allow access to NFSEN.

Change "SELINUX=enforcing" to --> "SELINUX=disabled"

```
vi /etc/selinux/config
```

Stop iptables firewall:

```
systemctl stop firewalld 
systemctl disable firewalld
```

Reboot to apply the SELINUX and the ipTables Changes:

```
reboot 
```

Run "sestatus" to verify selinux is disabled. The status should say the following: "SELinux status: disabled"

```
sestatus
```

Run the following to verify the firewall status. It should say: "not running"

```
firewall-cmd --state
```

## Start the NFSEN application setup

We will need to install a number of packages for CentOS 7 to get NFSEN running:

```
yum install -y httpd php wget gcc make rrdtool-devel rrdtool-perl perl-MailTools perl-Socket6 flex byacc perl-Sys-Syslog perl-Data-Dumper 

yum install -y autoconf automake apache php perl-MailTools rrdtool-perl perl-Socket6 perl-Sys-Syslog.x86_64 policycoreutils-python tcpdump

echo "date.timezone = America/Denver" > /etc/php.d/timezone.ini yum update -y
```

Create the netflow user account and add it to the apache group:

```
useradd netflow -G apache
```

Create the directories which we will specify later in the configuration file:

```
mkdir -p /data/nfsen 
mkdir -p /var/www/html/nfsen
```

Download the latest nfdump and nfsen packages. 
At time of this writing the latest versions are nfdump-1.6.13.tar.gz and nfsen-1.3.8.tar.gz

```
cd /opt/ 
wget http://downloads.sourceforge.net/project/nfdump/stable/nfdump-1.6.13/nfdump-1.6.13.tar.gz 
wget https://sourceforge.net/projects/nfsen/files/stable/nfsen-1.3.8/nfsen-1.3.8.tar.gz
```

Start the httpd service:

```
service httpd start
```

## Install NFDUMP

Untar the downloaded nfdump package into the "/opt/" Directory.

```
tar -zxvf nfdump-1.6.13.tar.gz 
```

Compile nfdump while in the "/opt/nfdump-1.6.13" directory:

```
cd /opt/nfdump-1.6.13
./configure --prefix=/opt/nfdump --enable-nfprofile --enable-nftrack --enable-sflow 
autoreconf 
make && sudo make install
```

## Install and configure nfsen

Untar nfsen into the "/opt/" directory.

```
cd ..
cd /opt/
tar -zxvf nfsen-1.3.8.tar.gz 
cd nfsen-1.3.8
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
Comment out lines by using "#" at the beginning of the line

```
%sources = ( 
'router-1' => { 'port' => '9030', 'col' => '#0000ff', 'type' => 'netflow' }, 
'firewall-1' => { 'port' => '9031', 'col' => '#9093ff', 'type' => 'netflow' }, 
);
```

The next section we will add the "flowdoh" plugin which we will install and configure after NFSEN and NFDUMP are up and running.

```
@plugins = ( # profile # module 
		     # [ '', 'demoplugin' ], 
			 [ '', 'flowdoh' ], )
```

Save the above changes

## Run the perl installation script to install nfsen:

```
cd .. 
./install.pl etc/nfsen.conf
```
Press enter to accept the default path. 

```
Perl to use: [/usr/bin/perl]
```

You may get Errors since we did not configure any flows at this point.

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

## At this point you should be able to access NFSEN

Go to http://127.0.0.1/nfsen/nfsen.php 
(in place of 127.0.0.1" use your server IP.)

### Ignore the following message when you connect:

```
Backend version missmatch!
```

## Installing Plugins:

### FLOWDOH:

We enabled it earlier, now we are going to installand configure it.

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

# INSTALLATION COMPLETE

The below steps will help in troubleshooting and maintaning your installation.

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

## Make sure that your system date and php dates are set correctly. 

You may need to edit /etc/php.ini and adjust your (date.timezone = "US/Eastern") settings

## Check running fcapd processes:

```
ps axo command | grep '[n]fcapd'
```

Check which ports nfcapd is listening on:

```
ss -nutlp
```

## Do the following when adding additional NETFLOW Senders:

On the nfsen server, edit the nfsen.conf file to add NetFlow sources: 

```
vi /data/nfsen/etc/nfsen.conf
```

You can set the color ( 'col' => '#xxyyxx') to something you would like by finding a HEX color identifyer at the following site: https://www.color-hex.com/
In place of ('CiscoRouter') and or ('LinuxServer') use a friendly name for your device like 'corp-firewall' or 'corp-router'

```
%sources = ( 
'CiscoRouter' => { 'port' => '2055', 'col' => '#0000ff', 'type' => 'netflow' }, 
'LinuxServer' => { 'port' => '9666', 'col' => '#ff5a00', 'type' => 'netflow' }, 
);
```

## Rebuild NFSEN after all settings changes:

```
cd /data/nfsen/bin/ 
./nfsen reconfig

/etc/init.d/nfsen restart
```
 
## Install Note: 

It takes around 5 minutes (depending on your server) to start showing data on the web pages. 
Validate with TCP Dump (Troubleshooting section above) to ensure data is being received.
If data is being received just wait it will start to populate the graphs.

## Authors

* **Initial work completed by** - [paidegua](https://github.com/paidegua)

* If I helped in anyway consider - [Donating a Beer](https://www.paypal.me/LanceJeffery)
