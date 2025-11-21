# 3. Vaja: Prenos podatkov preko omrežja
## Navodila

0. Uporabite omrežje in navidezne računalnike iz 2. vaje.
1. Namestite program Wireshark in zajemite ter preglejte pakete iz naslednjih nalog.
2. Namestite Trivial File Transport Protocol (TFTP) strežnik in preko njega prenesite datoteko na drug navidezni računalnik.
3. Namestite Hypertext Transfer Protocol (HTTP) strežnik in preko njega prenesite datoteko na drug navidezni računalnik.
4. Primerjajte hitrost prenosa datoteke med protokoloma TFTP in HTTP.

## Dodatne informacije

Ukaz [`usermod`](https://linux.die.net/man/8/usermod) nam omogoča spreminjanje lastnosti računa uporabnika.

Ukaz [`dpkg`](https://www.man7.org/linux/man-pages/man1/dpkg.1.html) nam omogoča upravljanje s paketi operacijskega sistema Debian.

Ukaz [`systemctl`](https://www.man7.org/linux/man-pages/man1/systemctl.1.html) nam omogoča upravljanje systemd sistema in storitev operacijskega sistema.

Ukaz [`wget`](https://www.man7.org/linux/man-pages/man1/wget.1.html) predstavlja orodje za prenos datotek preko omrežja s protokoli HTTP, HTTPS in FTP.

Ukaz [`netstat`](https://linux.die.net/man/8/netstat) predstavlja orodje za prikaz omrežnih lastnosti računalnika in storitev.

Ukaz [`cp`](https://www.man7.org/linux/man-pages/man1/cp.1.html) nam omogoča kopiranje datotek in map od izvora k ponoru.
 
## Podrobna navodila
### 1. Namestitev Wireshark
Sedaj namestimo program za zajemanje paketkov na omrežju [Wireshark](https://www.wireshark.org/) preko upravljalca paketkov našega operacijskega sistema.

	apt install wireshark

Med namestitvijo se nam pojavi naslednja izbira:

	Dumpcap can be installed in a way that allows members of the "wireshark" system group to capture packets. This is recommended over the alternative of running Wireshark/Tshark directly as root, because less of the code will run with elevated privileges.

	For more detailed information please see /usr/share/doc/wireshark-common/README.Debian.gz once the package is installed.

	Enabling this feature may be a security risk, so it is disabled by default. If in doubt, it is suggested to leave it disabled. 

	Should non-superusers be able to capture packets?                                     

	<Yes>                                           <No>

Kjer izberemo možnost `Da`, in potem dodamo našega uporabnika v skupino `wireshark`, da mu omogočim dostop do zajemanja prometa na omrežnih karticah z ukazom [`usermod`](https://linux.die.net/man/8/usermod).

	usermod -a -G wireshark YOUR_USERNAME

V primeru, da ste izbrali možnost `Ne`, lahko ponovno prikličete nastavitev z ukazom [`dpkg-reconfigure`](https://www.man7.org/linux/man-pages/man1/dpkg.1.html).

	sudo dpkg-reconfigure wireshark-common

Sedaj ponovno poženemo navidezni računalnik in poženemo Wireshark, da spremlja promet na omrežni kartici `enp0s8`.

### 2. Namestitev TFTP strežnika

Na strežniku namestimo željeno implementacijo `TFTP` strežnika, na primer `tftpd-hpa` preko upravljalca paketkov našega operacijskega sistema.

	apt install tftpd-hpa

Nastavitve `TFTP` strežnika preverimo v nastavitveni datoteki `/etc/default/tftpd-hpa`, kjer nastavimo mapo, ki jo strežnik ponuja prek omrežja.

	nano /etc/default/tftpd-hpa

	# /etc/default/tftpd-hpa

	TFTP_USERNAME="tftp"
	TFTP_DIRECTORY="/srv/tftp"
	TFTP_ADDRESS=":69"
	TFTP_OPTIONS="--secure"

Delovanje strežnika preverimo z ukazom [`systemctl`](https://www.man7.org/linux/man-pages/man1/systemctl.1.html):

	systemctl status tftpd-hpa.service 

V mapi `/srv/tftp` ustvarimo poljubno datoteko, na primer:

	nano /srv/tftp/test.txt

	This is a test file.

Na klientu sedaj namestimo `TFTP` klienta preko upravljalca paketkov našega operacijskega sistema, da preverimo delovanje `TFTP` protokola s prenosom ustvarjene datoteke.

	apt install tftp-hpa

	tftp 
	(to) 10.0.1.1
	tftp> get test.txt
	tftp> quit

	cat test.txt

### 3. Namestitev HTTP strežnika

Na strežniku namestimo željeno implementacijo `HTTP` strežnika, na primer `lighttpd` preko upravljalca paketkov našega operacijskega sistema.

	apt install lighttpd

Nastavitve `HTTP` strežnika preverimo v nastavitveni datoteki `/etc/lighttpd/lighttpd.conf`, kjer nastavimo mapo, ki jo strežnik ponuja prek omrežja.

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

V mapi `/var/www/html` ustvarimo poljubno datoteko, na primer:

	nano /var/www/html/test.txt

	This is a test file.

Na klientu sedaj z ukazom [`wget`](https://www.man7.org/linux/man-pages/man1/wget.1.html) preverimo delovanje `HTTP` protokola s prenosom ustvarjene datoteke.

	wget http://10.0.1.1/test.txt

	--2024-10-18 15:53:56--  http://10.0.1.1/test.txt
	Connecting to 10.0.1.1:80... connected.
	HTTP request sent, awaiting response... 200 OK
	Length: 12 [text/plain]
	Saving to: ‘test.txt’

	test.txt            100%[===================>]      12  --.-KB/s    in 0s      

	2024-10-18 15:53:56 (2.30 MB/s) - ‘test.txt’ saved [12/12]

	cat test.txt

### 4. Testiranje hitrosti prenosa datoteke s protokoloma HTTP in TFTP

Katere storitve trenutno tečejo na našem strežniku in na katerih vratih poslušajo lahko preverimo z orodjem [`netstat`](https://linux.die.net/man/8/netstat).

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

Sedaj prenesemo poljubno veliko datoteko s spleta na naš strežnik, na primer namestitveno sliko operacijskega sistema [Mint](https://linuxmint.com/download.php) ter jo kopiramo v mapi `/srv/tftp` in `/var/www/html`.

	cp /home/aleks/Downloads/linuxmint-22-cinnamon-64bit.iso /srv/tftp/Mint.iso
	cp /home/aleks/Downloads/linuxmint-22-cinnamon-64bit.iso /var/www/html/Mint.iso

Na strežniku sedaj z ukazom `wget` prenesemo omenjeno datoteko preko protokola `HTTP`.

	wget http://10.0.1.1/Mint.iso

	--2024-10-18 16:19:32--  http://10.0.1.1/Mint.iso
	Connecting to 10.0.1.1:80... connected.
	HTTP request sent, awaiting response... 200 OK
	Length: 2907832320 (2.7G) [application/x-iso9660-image]
	Saving to: ‘Mint.iso’

	Mint.iso            100%[===================>]   2.71G   201MB/s    in 14s     

	2024-10-18 16:19:47 (194 MB/s) - ‘Mint.iso’ saved [2907832320/2907832320]

Sedaj pa prenesemo enako datoteko še z ukazom `tftp` preko protokola `TFTP`.

	tftp

	(to) 10.0.1.1
	tftp> verbose     
	Verbose mode on.
	tftp> get Mint.iso
	getting from 10.0.1.1:Mint.iso to Mint.iso [netascii]
	Received 2930509229 bytes in 853.7 seconds [27462515 bit/s]
	tftp> quit
