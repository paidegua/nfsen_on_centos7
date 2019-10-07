### NFSEN CENTOS GENERAL SETUP:
## Start with a basic minimal installation of CentOS 7

yum update -y 

vi /etc/selinux/config   	

#change "SELINUX=enforcing" to --> "SELINUX=disabled"

reboot  					#to apply SELINUX changes

sestatus					#to VERIFY status; it should stay: "SELinux status: disabled"

# Stop iptables firewall:

systemctl stop firewalld
systemctl disable firewalld

# verify:

firewall-cmd --state		#to VERIFY status; it should stay: "not running"

### -----START NFSEN APPLICATION SETUP--------

# We will need to install a number of packages for CentOS 7

yum install -y httpd php wget gcc make rrdtool-devel rrdtool-perl perl-MailTools perl-Socket6 flex byacc perl-Sys-Syslog perl-Data-Dumper autoconf automake apache php perl-MailTools rrdtool-perl perl-Socket6 perl-Sys-Syslog.x86_64 policycoreutils-python tcpdump

echo "date.timezone = America/Denver" > /etc/php.d/timezone.ini
yum update -y 

# Create user account and add it to proper group

useradd netflow
usermod -a -G apache netflow

# Create directories which we will specify later in configuration file

mkdir -p /data/nfsen
mkdir -p /var/www/html/nfsen

# Download latest nfdump and nfsen packages at this time nfdump-1.6.13.tar.gz and nfsen-1.3.6p1.tar.gz

cd /opt/
wget http://downloads.sourceforge.net/project/nfdump/stable/nfdump-1.6.13/nfdump-1.6.13.tar.gz
wget http://downloads.sourceforge.net/project/nfsen/stable/nfsen-1.3.6p1/nfsen-1.3.6p1.tar.gz

# Start httpd service

service httpd start

# Install nfdump
 
# Untar downloaded nfdump package
# in /opt/

tar -zxvf nfdump-1.6.13.tar.gz
cd nfdump-1.6.13

# Compile nfdump:
# in /opt/nfdump-1.6.13

./configure --prefix=/opt/nfdump --enable-nfprofile --enable-nftrack --enable-sflow
autoreconf
make && sudo make install

## Install and configure nfsen

# Untar nfsen:
# in /opt/

cd ..
tar -zxvf nfsen-1.3.6p1.tar.gz
cd nfsen-1.3.6p1
cd etc
cp nfsen-dist.conf nfsen.conf
vi nfsen.conf

# Edit the nfsen.conf file with at least the changes below, make sure all data path variables are set correctly:

$BASEDIR= "/data/nfsen";

$HTMLDIR = "/var/www/nfsen";  'change to' --> $HTMLDIR = "/var/www/html/nfsen";

$PREFIX  = '/usr/local/bin'; 'change to' --> '$PREFIX  = '/opt/nfdump/bin';

$WWWUSER  = "www"; 'change to' --> $WWWUSER  = "apache";
$WWWGROUP = "www"; 'change to' --> $WWWGROUP = "apache";

# You can change the following if you know all the hosts:
%sources = (
    'router-1'    => { 'port' => '9030', 'col' => '#0000ff', 'type' => 'netflow' },
    'firewall-1'    => { 'port' => '9031', 'col' => '#9093ff', 'type' => 'netflow' },
);

@plugins = (
    # profile # module
    # [ '*', 'demoplugin' ],
    [ '*', 'flowdoh' ],
)

# save the changes above changes

## We will now run perl installation script to install nfsen (change directory)

cd ..
./install.pl etc/nfsen.conf

** Perl to use: [/usr/bin/perl] 	
#Press enter to accept default path. You may get Errors since we did not configure any flows at this point.

# Lets now create a startup script for the service

vi /etc/init.d/nfsen

-----------------COPY BELOW TO NEXT LINE INDICATOR------------------------
#!/bin/bash
#!
#chkconfig: - 50 50
#description: nfsen

DAEMON=/data/nfsen/bin/nfsen

case "$1" in
start)
$DAEMON start
;;
stop)
$DAEMON stop
;;
status)
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
-----------------COPY ABOVE TO NEXT LINE INDICATOR------------------------

##save the changes above changes

# make sure the script is executable

chmod +x /etc/init.d/nfsen

# Start nfsen deamon

/etc/init.d/./nfsen start

# use the following to restart nfsen when necessary:

/etc/init.d/nfsen restart

## At this point you should be able to access nfsen at http://127.0.0.1/nfsen/nfsen.php
 
** If you see the following message when you hit up your NfSen URL @ http://x.x.x.x/nfsen/nfsen.php
** "Backend version missmatch!" 
** This error can be safely ignored.

## ADDING ADDITIONAL NETFLOW SENDERS:
# On the nfsen server, add the following to nfsen.conf file:
# set the color to something you would like: use this site for reference: 
# https://www.color-hex.com/

vi /data/nfsen/etc/nfsen.conf

%sources = (
'CiscoRouter'    => { 'port' => '2055', 'col' => '#0000ff', 'type' => 'netflow' },
'LinuxServer'    => { 'port' => '9666', 'col' => '#ff5a00', 'type' => 'netflow' },
);

# To Rebuild NFSEN after settings changes:

cd /data/nfsen/bin/
./nfsen reconfig

/etc/init.d/nfsen restart

 
## TROUBLESHOOTING:

1. Use tcpdump and verify that flows are being received on the specified port.
	LIST NICs:
		#ip link show

		#tcpdump -i eno16777984 port 9030
		
		##Make sure that you see traffic on this port from required host.

2. With nfdump you can read flow collection files from command line

	cd /data/nfsen/profiles-data/live/
	/opt/nfdump/bin/nfdump -r  "your file name"
	
3. Make sure that your system data and php date set correctly. You may need to edit /etc/php.ini and adjust your date.timezone = "US/Eastern"

4. Check running fcapd processes: 

	ps axo command | grep '[n]fcapd'

5. Check which ports nfcapd is listening on:

	ss -nutlp

## INSTALLING PLUGINS:

# FLOWDOH:
cd /opt/
sudo wget https://sourceforge.net/projects/flowdoh/files/FlowDoh_1.0.2.tar.gz
sudo tar -zxvf FlowDoh_1.0.2.tar.gz

#Backend: (/data/nfsen/plugins/)
cd /opt/flowdoh/backend/
cp -avr flowdoh/ /data/nfsen/plugins/
cp flowdoh.pm /data/nfsen/plugins/
cd /data/nfsen/plugins/flowdoh
cp flowdoh.conf.defaults flowdoh.conf
cd /data/nfsen/plugins
chmod 777 flowdoh
chmod 777 flowdoh.pm

#FrontEnd: (/var/www/html/nfsen/plugins/)
cd /opt/flowdoh/frontend/
cp flowdoh.php /var/www/html/nfsen/plugins/
cp -avr flowdoh/ /var/www/html/nfsen/plugins/
cd /var/www/html/nfsen/plugins
chmod 777 flowdoh
chmod 777 flowdoh.php

/etc/init.d/nfsen restart
