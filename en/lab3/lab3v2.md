# 3. Lab: Data transfer over the network
## Instructions

0. Use the network and virtual machines from 2. Lab.
1. Install Wireshark and capture and inspect the packets from the following tasks.
2. Install a Trivial File Transport Protocol (TFTP) server and use it to transfer the file to another virtual machine.
3. Install a Hypertext Transfer Protocol (HTTP) server and use it to transfer the file to another virtual machine.
4. Compare the file transfer speed between TFTP and HTTP protocols.

## Additional information

The [`usermod`](https://linux.die.net/man/8/usermod) command allows us to change the user account properties.

The [`dpkg`](https://www.man7.org/linux/man-pages/man1/dpkg.1.html) command allows us to manage packages of the Debian operating system.

The [`systemctl`](https://www.man7.org/linux/man-pages/man1/systemctl.1.html) command allows us to manage the systemd system and operating system services.

The command [`wget`](https://www.man7.org/linux/man-pages/man1/wget.1.html) is a tool for downloading files over the network using the HTTP, HTTPS and FTP protocols.

The [`netstat`](https://linux.die.net/man/8/netstat) command is a tool to display a computer's network properties and services.

The [`cp`](https://www.man7.org/linux/man-pages/man1/cp.1.html) command copies files and directories from source to destination.
 
## Detailed instructions
### 1. Instalation of Wireshark
Now let's install the network packet capture program [Wireshark](https://www.wireshark.org/) through our operating system's packet manager.

	apt install wireshark

During the installation, the following choice appears:

	Dumpcap can be installed in a way that allows members of the "wireshark" system group to capture packets. This is recommended over the alternative of running Wireshark/Tshark directly as root, because less of the code will run with elevated privileges.

	For more detailed information please see /usr/share/doc/wireshark-common/README.Debian.gz once the package is installed.

	Enabling this feature may be a security risk, so it is disabled by default. If in doubt, it is suggested to leave it disabled. 

	Should non-superusers be able to capture packets?                                     

	<Yes>                                           <No>

Where we select the `Yes` option, and then add our user to the `wireshark` group to give him access to capture NIC traffic with the [`usermod`](https://linux.die.net/man/8/usermod).

	usermod -a -G wireshark YOUR_USERNAME

In case you selected `No`, you can recall the configuration with the [`dpkg-reconfigure`](https://www.man7.org/linux/man-pages/man1/dpkg.1.html) command.

	sudo dpkg-reconfigure wireshark-common

Now we restart the virtual machine and run Wireshark to monitor the traffic on the `enp0s8` NIC.

### 2. Instalation of TFTP server

On the server, we install the desired `TFTP` server implementation, for example `tftpd-hpa` via the package manager of our operating system.

	apt install tftpd-hpa

We check the `TFTP` server settings in the configuration file `/etc/default/tftpd-hpa`, where we set the folder that the server offers over the network.

	nano /etc/default/tftpd-hpa

	# /etc/default/tftpd-hpa

	TFTP_USERNAME="tftp"
	TFTP_DIRECTORY="/srv/tftp"
	TFTP_ADDRESS=":69"
	TFTP_OPTIONS="--secure"

We check the operation of the server with the command [`systemctl`](https://www.man7.org/linux/man-pages/man1/systemctl.1.html):

	systemctl status tftpd-hpa.service 

Create an arbitrary file in the `/srv/tftp` folder, for example:

	nano /srv/tftp/test.txt

	This is a test file.

On the client, we now install the `TFTP` client through the package manager of our operating system, in order to check the operation of the `TFTP` protocol by downloading the created file.

	apt install tftp-hpa

	tftp 
	(to) 10.0.1.1
	tftp> get test.txt
	tftp> quit

	cat test.txt

### 3. Instalation of HTTP server

We install the desired HTTP server implementation on the server, for example `lighttpd' via the package manager of our operating system.

	apt install lighttpd

We check the `HTTP` settings of the server in the configuration file `/etc/lighttpd/lighttpd.conf`, where we set the folder that the server offers over the network.

	nano /etc/lighttpd/lighttpd.conf

	server.modules = (
        "mod_indexfile",
        "mod_access",
        "mod_alias",
        "mod_redirect",
	)

	server.document-root        = "/var/www/html"
	server.upload-dirs          = ( "/var/cache/lighttpd/uploads" )
	server.errorlog             = "/var/log/lighttpd/error.log"
	server.pid-file             = "/run/lighttpd.pid"
	server.username             = "www-data"
	server.groupname            = "www-data"
	server.port                 = 80

	# features
	#https://redmine.lighttpd.net/projects/lighttpd/wiki/Server_feature-flagsDetails
	server.feature-flags       += ("server.h2proto" => "enable")
	server.feature-flags       += ("server.h2c"     => "enable")
	server.feature-flags       += ("server.graceful-shutdown-timeout" => 5)
	#server.feature-flags       += ("server.graceful-restart-bg" => "enable")

	# strict parsing and normalization of URL for consistency and security
	# https://redmine.lighttpd.net/projects/lighttpd/wiki/Server_http-parseoptsDetails
	# (might need to explicitly set "url-path-2f-decode" = "disable"
	#  if a specific application is encoding URLs inside url-path)
	server.http-parseopts = (
  		"header-strict"           => "enable",# default
  		"host-strict"             => "enable",# default
  		"host-normalize"          => "enable",# default
  		"url-normalize-unreserved"=> "enable",# recommended highly
  		"url-normalize-required"  => "enable",# recommended
  		"url-ctrls-reject"        => "enable",# recommended
  		"url-path-2f-decode"      => "enable",# recommended highly (unless breaks app)
 	   #"url-path-2f-reject"      => "enable",
  		"url-path-dotseg-remove"  => "enable",# recommended highly (unless breaks app)
 	   #"url-path-dotseg-reject"  => "enable",
 	   #"url-query-20-plus"       => "enable",# consistency in query string
	)

	index-file.names            = ( "index.php", "index.html" )
	url.access-deny             = ( "~", ".inc" )
	static-file.exclude-extensions = ( ".php", ".pl", ".fcgi" )

	# default listening port for IPv6 falls back to the IPv4 port
	include_shell "/usr/share/lighttpd/use-ipv6.pl " + server.port
	include_shell "/usr/share/lighttpd/create-mime.conf.pl"
	include "/etc/lighttpd/conf-enabled/*.conf"

	#server.compat-module-load   = "disable"
	server.modules += (
    	    "mod_dirlisting",
    	    "mod_staticfile",
	)

Create an arbitrary file in the `/var/www/html` folder, for example:

	nano /var/www/html/test.txt

	This is a test file.

On the client, we now use the command [`wget`](https://www.man7.org/linux/man-pages/man1/wget.1.html) to check the operation of the `HTTP` protocol by downloading the created file.

	wget http://10.0.1.1/test.txt

	--2024-10-18 15:53:56--  http://10.0.1.1/test.txt
	Connecting to 10.0.1.1:80... connected.
	HTTP request sent, awaiting response... 200 OK
	Length: 12 [text/plain]
	Saving to: ‘test.txt’

	test.txt            100%[===================>]      12  --.-KB/s    in 0s      

	2024-10-18 15:53:56 (2.30 MB/s) - ‘test.txt’ saved [12/12]

	cat test.txt

### 4. HTTP and TFTP file transfer speed testing

We can check which services are currently running on our server and which ports they are listening on with the [`netstat`](https://linux.die.net/man/8/netstat) tool.

	apt install net-tools

	netstat -ntulp

	Active Internet connections (only servers)
	Proto Recv-Q Send-Q Local Address           Foreign A	ddress         State       PID/Program name    
	tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      678/cupsd           
	tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      3144/lighttpd       
	tcp6       0      0 ::1:631                 :::*                    LISTEN      678/cupsd           
	tcp6       0      0 :::80                   :::*                    LISTEN      3144/lighttpd       
	udp        0      0 0.0.0.0:35725           0.0.0.0:*                           629/avahi-daemon: r 
	udp        0      0 0.0.0.0:5353            0.0.0.0:*                           629/avahi-daemon: r 
	udp        0      0 0.0.0.0:67              0.0.0.0:*                           707/dhcpd           
	udp        0      0 0.0.0.0:68              0.0.0.0:*                           563/dhclient        
	udp        0      0 0.0.0.0:69              0.0.0.0:*                           2741/in.tftpd       
	udp6       0      0 :::5353                 :::*                                629/avahi-daemon: r 
	udp6       0      0 :::42225                :::*                                629/avahi-daemon: r 
	udp6       0      0 :::69                   :::*                                2741/in.tftpd       

Now we download any large file from the web to our server, for example the installation image of the operating system [Mint](https://linuxmint.com/download.php) and copy it to `/srv/tftp` and `/var/www /html`.

	cp /home/aleks/Downloads/linuxmint-22-cinnamon-64bit.iso /srv/tftp/Mint.iso
	cp /home/aleks/Downloads/linuxmint-22-cinnamon-64bit.iso /var/www/html/Mint.iso

On the server, we now use the `wget` command to download the mentioned file via the `HTTP` protocol.

	wget http://10.0.1.1/Mint.iso

	--2024-10-18 16:19:32--  http://10.0.1.1/Mint.iso
	Connecting to 10.0.1.1:80... connected.
	HTTP request sent, awaiting response... 200 OK
	Length: 2907832320 (2.7G) [application/x-iso9660-image]
	Saving to: ‘Mint.iso’

	Mint.iso            100%[===================>]   2.71G   201MB/s    in 14s     

	2024-10-18 16:19:47 (194 MB/s) - ‘Mint.iso’ saved [2907832320/2907832320]

Now let's transfer the same file with the `tftp` command via the `TFTP` protocol.

	tftp

	(to) 10.0.1.1
	tftp> verbose     
	Verbose mode on.
	tftp> get Mint.iso
	getting from 10.0.1.1:Mint.iso to Mint.iso [netascii]
	Received 2930509229 bytes in 853.7 seconds [27462515 bit/s]
	tftp> quit
