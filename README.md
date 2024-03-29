# Setup FreeSWITCH
---
In this project FreeSWITCH is used as server for webRTC.

Info Article:
http://www.linuxpromagazine.com/Issues/2009/106/FreeSWITCH


## Requirements

Linux Debin 10 Buster

## Install Apache and issue SSL-Certificats (Let's encrypt)
### Install 
These commands will install the apache webserver and certbot to get valid SSL-Certificates from Let's encrypt. More information: https://certbot.eff.org/
```
$ apt-get update && apt-get upgrade
$ apt-get install certbot
$ apt-get install apache2
```
You will also want to update /etc/hosts and /etc/hostname to reflect the domain name you will be using.

in /etc/hosts append your hostname to the end of the 127.0.1.1 line
```
127.0.1.1 <somedomain-name>
```
and in /etc/hostname replace the current name with your hostname.
```
<somedomain-name>
```

### Issue certificate
Next, we will create our certificate. Execute the following command and fill out any necessary fields.
```
certbot certonly --webroot -w /var/www/html/ -d <somehostname>
```

### Configure SSL for HTTP
We need to enable the SSL module for Apache2. On Debian, you can issue this command.
```
sudo a2enmod ssl
```
Next we need to create a virtual host in /etc/apache2/sites-available/. Create a file named your new domain name in /conf
```
nano /etc/apache2/sites-available/pbx.somedomain.com.conf
```
Then include the following configuration. Note, you will need to point the SSL certificates to the correct directory depending on your domain. You will also need to change the ServerName parameter to whatever your domain name is.
```
<VirtualHost *:443>
        ServerName somedomain-name
        DocumentRoot /var/www/html
        SSLEngine on
        SSLProtocol all -SSLv3 -SSLv3
        SSLCompression off
        SSLHonorCipherOrder On
        SSLCipherSuite EECDH+AESGCM:EECDH+AES:EDH+AES

        SSLCertificateFile /etc/letsencrypt/live/somedomain.com/cert.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/somedomain.com/privkey.pem
        SSLCertificateChainFile /etc/letsencrypt/live/somedomain.com/chain.pem

        ServerAdmin some-email@address.com

        ErrorLog ${APACHE_LOG_DIR}/error-devel.log
        CustomLog ${APACHE_LOG_DIR}/access-devel.log combined
        DirectoryIndex index.html
</VirtualHost>
```
Enable the new virtual host with ``a2ensite pbx.somedomain.com.conf``. This will create a symlink from sites-available to sites-enabled. Then Restart Apache2 to enable these changes with ``systemctl restart apache2``.

## Install FreeSWITCH
https://freeswitch.org/confluence/display/FREESWITCH/Debian+10+Buster
```
# apt-get update && apt-get install -yq gnupg2 wget lsb-release
# wget -O - https://files.freeswitch.org/repo/deb/debian-release/fsstretch-archive-keyring.asc | apt-key add -
 
# echo "deb http://files.freeswitch.org/repo/deb/debian-release/ `lsb_release -sc` main" > /etc/apt/sources.list.d/freeswitch.list
# echo "deb-src http://files.freeswitch.org/repo/deb/debian-release/ `lsb_release -sc` main" >> /etc/apt/sources.list.d/freeswitch.list
 
# apt-get update
  
#  apt-get build-dep freeswitch
  
# cd /usr/src/
# git clone https://github.com/signalwire/freeswitch.git -bv1.10 freeswitch
#  cd freeswitch
  
# ./bootstrap.sh -j
# ./configure
#  make
# make install
```
Install sounds:
```
$ make all cd-sounds-install cd-moh-install
```
Create SimLinks that Binaries are excessible everywhere:
```
$ ln -s /usr/local/freeswitch/bin/fs_cli /usr/bin/fs_cli
```

## Basic Configuration
https://freeswitch.org/confluence/display/FREESWITCH/Debian+Post-Install+Tasks
### Set Owner and Permissions

```
$ cd /usr/local
$ groupadd freeswitch
$ adduser --disabled-password  --quiet --system --home /usr/local/freeswitch --gecos "FreeSWITCH Voice Platform" --ingroup freeswitch freeswitch
$ chown -R freeswitch:freeswitch /usr/local/freeswitch/
$ chmod -R ug=rwX,o= /usr/local/freeswitch/
$ chmod -R u=rwx,g=rx /usr/local/freeswitch/bin/
```

### Start FreeSWITCH on boot
Copy the following content to ‘/lib/systemd/system/freeswitch.service’

```
[Unit]
Description=freeswitch
After=syslog.target network.target local-fs.target

[Service]
; service
Type=forking
PIDFile=/usr/local/freeswitch/run/freeswitch.pid
PermissionsStartOnly=true
ExecStart=/usr/local/freeswitch/bin/freeswitch -u freeswitch -g freeswitch -ncwait -nonat -rp
TimeoutSec=45s
Restart=on-failure
; exec
WorkingDirectory=/usr/local/freeswitch/bin
User=root
Group=daemon
LimitCORE=infinity
LimitNOFILE=100000
LimitNPROC=60000
;LimitSTACK=240
LimitRTPRIO=infinity
LimitRTTIME=7000000
IOSchedulingClass=realtime
IOSchedulingPriority=2
CPUSchedulingPolicy=rr
CPUSchedulingPriority=89
UMask=0007

[Install]
WantedBy=multi-user.target
```
Then execute the following command in the bash:
```
$ chmod 750 /lib/systemd/system/freeswitch.service
$ ln -s /lib/systemd/system/freeswitch.service /etc/systemd/system/freeswitch.service
$ systemctl daemon-reload
$ systemctl enable freeswitch.service
```

### Install WSS certificates
Freeswitch has to be started at least once before this step to make sure the certs folder is generated. Use the following command to append the certs to the wss.pem file.
```
/usr/local/freeswitch/bin/freeswitch
```
After starting the server, make sure to shut it down again by typing ``...`` and pressing Enter.
```
cat /etc/letsencrypt/live/somedomain.com/fullchain.pem /etc/letsencrypt/live/somedomain.com/privkey.pem > /usr/local/freeswitch/certs/wss.pem
```
Uncomment the following line in /usr/local/freeswitch/conf/directory/default.xml, or whatever directory you would like to be able to use mod_verto for WebRTC.
```
<param name="jsonrpc-allowed-event-channels" value="demo,conference,presence"/>
```

### Setup Configuration Files
The congiguration files are ready to go in this repository. Delete (or rename) the existing conf-Folder in /usr/local/freeswitch,  clone this repository and name it conf. You should restart Freeswitch now.
```
$ rm -r /usr/local/freesswitch/conf
$ git clone [this repository] conf
$ systemctl restart freeswitch
```

### Open Ports 
All of these Ports need to be opened in order for Freeswitch to run as intended.
| Protocol | Port | Used for |
|---|---|---|
| TCP | 80 | HTTP |
| TCP | 5060 -  5061 | Internal SIP TCP |
| TCP | 5080 – 5081 | External SIP TCP |
| TCP | 443 | HTTPS |
| TCP | 8081 -  8082 | WSS |
| TCP | 22 | SSH |
| UDP | 8021 | Event Sockets |
| UDP | 5060 – 5061 | Internal SIP UDP |
| UDP | 5080 – 5081 | External SIP UDP |
| UDP | 16384 – 32768 | FS MediaPorts |

### Some extro work to get ready for AWS
https://freeswitch.org/confluence/display/FREESWITCH/Amazon+EC2
Make sure the requiered ports are open!!!

make the following changes to the mentioned files.
``conf/vars.xml``
```
<X-PRE-PROCESS cmd="exec-set" data="bind_server_ip=curl -s http://instance-data/latest/meta-data/public-ipv4"/>
<X-PRE-PROCESS cmd="exec-set" data="external_rtp_ip=curl -s http://instance-data/latest/meta-data/public-ipv4"/>
<X-PRE-PROCESS cmd="exec-set" data="external_sip_ip=curl -s http://instance-data/latest/meta-data/public-ipv4"/>
```
The alternative to the above commands is to hard code the external IP addresses. However, this will require you to customize the vars.xml file for each instance (i.e. each external IP address) to which it is deployed. It will also have to be changed if you map/unmap an Elastic IP to the instance.
```
<X-PRE-PROCESS cmd="set" data="bind_server_ip=[AWS EIP]"/>
<X-PRE-PROCESS cmd="set" data="external_rtp_ip=[AWS EIP]"/>
<X-PRE-PROCESS cmd="set" data="external_sip_ip=[AWS EIP]"/>
 ```
``conf/autoload_configs/verto.conf.xml``
```
<param name="ext-rtp-ip" data="[AWS EIP]">
```

## Get ready for production use
### Set loglevel
If used as productive server you need to lower the loglevel otherwise the logfiles will overflow at some point. Open the file console.conf.xml and edit the setting part.
```
$ nano /usr/local/freeswitch/conf/autoload_configs/console.conf.xml
```
The setting part should now look like this:
```
<settings>
  <param name="colorize" value="true"/>
  <param name="loglevel" value="info"/>
</settings>
```

### Finalize - Start Server
In the end all you have to do is start the Freeswitch Server. 
```
/usr/local/freeswitch/bin/freeswitch
```

# Author
Lennart Paul - Metaverse School GmbH
