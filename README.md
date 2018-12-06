# LEO1 Assignment 2

## Overview

Two LXC containers are created on a Raspberry Pi, where the first container contains a web service that fetches random numbers from the second container.

## Setup

### LXC network bridge

The guide at [1] was used to create the network bridge for the LXC containers. This guide was also used to setup static IPs for the two containers, but this did not seem to always work, which is why the [startup instructions](#startup_instructions) does not rely on this.

### Containers

Two unprivileged containers are created following the guide at [2] for the unprivileged configuration and the provided slides for the assignment for the creation of the containers with `lxc-create -n [[name]] -t download -- -d alpine -r 3.4 -a armhf` with the values C1 and C2 for `[[name]]`.

The first container has the packages `lighttpd, php5, php5-cgi, php5-curl and php5-fpm` installed. The `lighttpd` package enables web server functionality, whereas the remaining packages enable PHP functionality for the web server. The packages are installed using the command `apk add [[package name(s) here]]` after running `apk update` . As per the provided slides, the `/etc/lighttpd/lighttpd.conf` configuration file is modified to include the `mod_fastcgi.conf` file, and then the Lighttpd web server is added to OpenRC using `rc-update add lighttpd default`.

The `index.php` file available from the Lighttpd web server is added at the path `/var/www/localhost/htdocs/` with the contents provided in the slides for the assignment. This script sends a request to the second container on port `8080` for the random numbers.

One issue with FastCGI for the Lighttpd web server was that it could not find the path `/usr/bin/php-cgi5`. This was seen in the `/var/log/lighttpd/error.log` log file with the messages:
```
2018-12-05 20:46:25: (mod_fastcgi.c.1102) the fastcgi-backend /usr/bin/php-cgi5 failed to start:                                                                                          
2018-12-05 20:46:25: (mod_fastcgi.c.1106) child exited with status 2 /usr/bin/php-cgi5                                                                                                    
2018-12-05 20:46:25: (mod_fastcgi.c.1109) If you're trying to run your app as a FastCGI backend, make sure you're using the FastCGI-enabled version.\nIf this is PHP on Gentoo, add 'fastc
2018-12-05 20:46:25: (mod_fastcgi.c.1395) [ERROR]: spawning fcgi failed.                                                                                                                  
2018-12-05 20:46:25: (server.c.1030) Configuration of plugins failed. Going down.  
```
This issue was fixed by renaming the path `/usr/bin/php-cgi` to `/usr/bin/php-cgi5` using `mv /usr/bin/php-cgi /usr/bin/php-cgi5`.

The second container has the package socat installed similarly to the first container's packages. The script to run with socat is the same as the one provided in the assignment slides, but with the small difference that `/dev/random` is replaced with `/dev/urandom`, since the first did not seem to provide any or enough random numbers on demand.
Also, `#!/bin/bash`  has been replaced with `#!/bin/sh` as this is what Alpine provides by default.
In order to allow the random number generator script to be executable, the executable permission is given using `chmod u+x /bin/rng.sh`.

## Startup instructions

The instructions after boot to start the containers and run their respective services are as follows:
1. `lxc-start -n C1 && lxc-start -n C2`
    1. -- WAIT until `lxc-ls --fancy` gives IPV4 for containers --
2. `sudo iptables -t nat -A PREROUTING -i usb0 -p tcp --dport 80 -j DNAT --to-destination 10.0.3.[[last digits here]]:80`
3. `lxc-attach -n C1`
    1. `openrc`
    2. `exit`
4. `lxc-attach -n C2`
    1. `socat -v -v tcp-listen:8080,fork,reuseaddr exec:/bin/rng.sh`

Instruction 1 starts both LXC containers. Since the IP-addresses are not assigned immediately, the containers are listed until the IP-address is available. This is necessary for the port forwarding in instruction 2, where the IP-address is needed to add a new rule to expose the Lighttpd web server on port `80` to the outside world through the Raspberry Pi's port `80`. Since the Raspberry Pi is connected as a USB gadget, the `usb0` interface is used for the connection.

Instruction 3 allows access to the first container, where the OpenRC system is run to start the preconfigured Lighttpd web server. Instruction 4 allows access to the second container, where socat is run (as per the provided slides), directing network traffic to run the `rng.sh` script on port `8080`.

`http://raspberrypi.local` can now be used to access the first container's web server providing random numbers.

## Sources
[1] https://angristan.xyz/setup-network-bridge-lxc-net/
[2] https://help.ubuntu.com/lts/serverguide/lxc.html#lxc-unpriv 
