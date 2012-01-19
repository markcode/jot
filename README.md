Jot Server
==========

Node.js pixel server - Setup with Scribe and Nginx; "jot"

Version: 0.1

Author: Mark W. B. Ashcroft

Author URI: http://analumic.com

Last Mod: 16 JAN 12


License (See LICENSE file for full license)
-------------------------------------------
Copyright (c) 2012, Mark W. B. Ashcroft

Copyright (c) 2012, Analumic

Licensed under the GNU GPLv2;

This program is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation version 2. See LICENSE file for details.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with this program; if not, write to the Free Software Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA 02111-1307 USA


Introduction
------------
This setup combines Facebook Scribe, Node.js, Maxmind GeoIP, Nginx on EC2 to run as a web analytics pixel server


Notes
-----

ami-cf33fea6 (micro): Performance testing return around - per internal request.

if want to ab (apache benchmark) on nginx
  apt-get install apache2-utils
	
(so far, havent finished jot.js, scribe, error, geo etc)
ami-cf33fea6 (micro): ab -t 10 http://127.0.0.1/pixel.gif returned 1311.14 rps.
ami-cf33fea6 (micro): ab -t 30 -c 10 http://ec2-174-129-65-40.compute-1.amazonaws.com/pixel.gif returned 916.97 rps.
ami-cf33fea6 (micro): ab -t 30 -c 40 http://127.0.0.1/pixel.gif returned 502.22 rps.

The micro will refuse ab tests more often with "Connection reset by peer" messages.

AMIs
----
Ubuntu 10.10 "Maverick"

ami-cf33fea6 (Ubuntu 10.10 64Bit EBS 8GB from: http://alestic.com/)

Install
-------

    sudo su		#to be root user, else use sudo

    apt-get -y update && apt-get upgrade

###Install Dependencies	
	
    apt-get -y install libboost-dev libevent-dev python-dev automake pkg-config libtool flex bison
    apt-get -y install ant
    apt-get -y install openjdk-6-jdk
    apt-get -y install bjam
    apt-get -y install libboost-all-dev
    apt-get -y install git git-core
    apt-get -y install curl build-essential openssl libssl-dev libpcre3-dev zlib1g

###Install Nginx

	nginx=stable
	add-apt-repository ppa:nginx/$nginx
	apt-get install nginx	

commands

	service nginx start
	service nginx stop
	service nginx restart
	
###Can reboot instance if want more memory (sudo su)

	reboot

###Install thrift

    cd opt
    git clone git://git.apache.org/thrift.git
    cd thrift
    git reset 9f3296bca00927ec5bac7ccdecdf2fbd68be9744 --hard
    ./bootstrap.sh
    ./configure
    make		#takes ages!
    make install

###Hum may not need py's

    cd lib/py
    python setup.py install

###Install fb303

    cd ..
    cd ..

    cd contrib/fb303
    ./bootstrap.sh
    ./configure
    make
    make install

###Install scribe

    cd ..
    cd ..
    cd ..

    git clone http://github.com/facebook/scribe.git
    cd scribe

    ./bootstrap.sh
    make
    make install
	
###Hum may not need py's but did anyway

    cd lib/py
    python setup.py install

###Finally fix a bug by doing this

    echo "/usr/local/lib" >> /etc/ld.so.conf
    /sbin/ldconfig

###Can test scribe

    scribed -c /opt/scribe/examples/example1.conf
    ctrl+c

###Scribe - create default config file for scribed to use

	mkdir /usr/local/scribe/
	vi /usr/local/scribe/scribe.conf
	INS /jot/config/scribe.conf 	#or whatever

###Scribe - now setup scribed so will be able to control start/stop/restart such as on reboot

	mkdir /var/log/jot
	#scribe - edit jot/config/scribed file path to scribe.conf (or other conf file).
	vi /etc/init.d/scribed
	#INS jot/config/scribed (file contents), need to set /usr/local/scribe/scribe.conf
	chmod ugo+x /etc/init.d/scribed
	update-rc.d scribed defaults
	// then add crontab 
	crontab -e
	INS 
	@reboot /etc/init.d/scribed start	

###Install node from source (apt-get install nodejs worked but acted funny! so install from source)

    cd opt
    git clone https://github.com/joyent/node.git && cd node
    ./configure
    make		#takes ages * ages - lunch!
    make install
    node -v
	
had problems geting node-waf to work with node so reinstalled node from source

	wget http://nodejs.org/dist/v0.6.7/node-v0.6.7.tar.gz
	tar -zxvf node-v0.6.7.tar.gz
	cd node-v0.6.7
    ./configure
    make		#takes ages * ages - lunch!
    make install
    node -v	

###Install npm for node modules (within node script directory "jot")

to install locally within node script directory /var/jot or globally using -g

	cd /opt/
    curl http://npmjs.org/install.sh | sh

	npm install thrift -g
    npm install scribe -g

###Install jot.js program

    cd /var/
    mkdir jot
    cd jot
    vi jot.js
    INS...	

###Now fix node errors

Anoyingly (at the time of writing this) when run jot.js both node modules (scribe and thrift) generate 
Error: The "sys" module is now called "util".

I go through the module files and replace require('sys') with require('util'). 
I'm new to node so there's probably a simple error capture function I should use in my jot.js script.

###Nginx config notes

nginx security, change server name sent to browser to not show nginx version

	vi /etc/nginx/sites-available/default
	server {
		server_tokens off;

nginx config, probably don't need the access log if using this as dedicated scribe server

	vi /etc/nginx/sites-available/default
	server {
		#access_log  /var/log/nginx/localhost.access.log;
		
Proxying node through nginx

can specify which web directory to proxy: location /website_directory {

	vi /etc/nginx/sites-available/default			
		upstream node_cluster {
			ip_hash;   
			server 127.0.0.1:8000;
			#server 127.0.0.1:8001;
			#server 127.0.0.1:8002;
		}

		server {
			...
			location / {
				#root   /var/www;
                #index  index.html index.htm;

				proxy_set_header X-Real-IP $remote_addr;
				proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
				proxy_set_header Host $http_host;

				proxy_pass http://node_cluster/;
				proxy_redirect off;
				
				keepalive_requests 0;
			}

###Monitor jot.js with Monit

Need to keep an eye on jot.js to make sure it keeps running. 
Nice article here: http://howtonode.org/a9fcf4f29cc999af32c93c7e43c9688c164ccd12/deploying-node-upstart-monit

###Install GeoIP databases (from Maxmind)

Conform with Maxmind license; "This product includes GeoLite data created by MaxMind, available from
http://maxmind.com/"

	cd tmp
	wget http://geolite.maxmind.com/download/geoip/database/GeoLiteCountry/GeoIP.dat.gz
	gunzip GeoIP.dat.gz
	mkdir /usr/local/share/GeoIP/
	mv GeoIP.dat /usr/local/share/GeoIP/
	wget http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz
	gunzip GeoLiteCity.dat.gz
	mv GeoLiteCity.dat /usr/local/share/GeoIP/
	wget http://geolite.maxmind.com/download/geoip/database/GeoIPv6.dat.gz
	gunzip GeoIPv6.dat.gz
	mv GeoIPv6.dat /usr/local/share/GeoIP/
	wget http://geolite.maxmind.com/download/geoip/database/GeoLiteCityv6-beta/GeoLiteCityv6.dat.gz
	gunzip GeoLiteCityv6.dat.gz
	mv GeoLiteCityv6.dat /usr/local/share/GeoIP/

###Install GeoIP C library

	cd opt
	wget -c  http://geolite.maxmind.com/download/geoip/api/c/GeoIP.tar.gz
	tar -xzf GeoIP.tar.gz
	rm GeoIP.tar.gz
	cd GeoIP-1.4.8	#or whatever the version is
	./configure
	make
	make check
	make install

###Geoip for node.js

had major problems getting geoip for node working through npm so install manually like

	cd /opt
	git clone https://github.com/kuno/GeoIP
	cd GeoIP
	node-waf configure build #shit, did'nt work on my EC2
	
tryed

	/opt/node/tools/waf-light configure build #configured but did'nt build
	/opt/waf/waf configure
	/usr/bin/node-waf/waf-light configure build
	/usr/bin/node-waf/waf configure
	node-waf configure && node-waf build
	
	
	
now can run by refering to this build like

	var City = geoip.City;
	var city = new City('/usr/local/share/GeoIP/GeoLiteCity.dat', true);
	var sync_data = city.lookupSync('8.8.8.8');
	console.log(sync_data);	

###Output Scribe logs

Now all is working can output Scribe's logs to somewhere for processing.
This default setup ouputs text log files to /var/log/jot/

Many alternate choices; Scribe has built in support for HDFS (Hadoop) or
use your favourate database, like: MongoDB, Kyoto Cabinet.

I prefer to output to log files (default setup), then add a connector either
in jot.js or piping

	tail -f /var/log/jot/foo_current
	
to Sphinx Search for realtime search/analytics.

###Other

The setup described here is just general; you'll need to apply all nessisary security/redundency to your EC2 and nginx setup! 

###Problems with node geoip

people say that wpn error is due to old python version >2.7 which is found on Ubumtu 10.10, so will try and upgrade

	cd opt
	curl -kL http://xrl.us/pythonbrewinstall | bash
	
tryed (but still no luck)

	apt-get install nodejs-dev
	apt-get install libexpat-dev

	

still working on setup, come back soon...
-----------------------------------------
