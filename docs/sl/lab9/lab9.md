# 9. Vaja: Postavitev navideznega zasebnega omrežja z uporabo deljene skrivnosti

## Navodila

0. Uporabite omrežje in virtualne stroje iz prejšnjih vaj.
1. Nastavite in preizkusite delovanje tunela OpenVPN med strežnikom in odjemalcem v načinu TUN z deljeno skrivnostjo prek UDP.
2. Nastavite in preizkusite delovanje tunela OpenVPN med strežnikom in odjemalcem tudi prek načina TAP z deljeno skrivnostjo prek TCP.
3. Nastavite in preizkusite delovanje tunela OpenVPN med strežnikom in odjemalcem v načinu TUN s prstnimi odtisi potrdil prek UDP.

## Dodatne informacije

[Simetrična kriptografija](https://en.wikipedia.org/wiki/Cryptography#Symmetric-key_cryptography) je pristop, pri katerem za enkripcijo in dekripcijo uporabljamo skupno skrivnost.

[Navidezno zasebno omrežje (Virtual Private Network - VPN)](https://en.wikipedia.org/wiki/Virtual_private_network) razširi lokalno omrežje preko javnega omrežja in omogoči uporabnikom pošiljanje in prejemanje podatkov čez javno omrežje, kot da so povezani direktno preko lokalnega omrežja. VPN omogoča funkcionalnosti lokalnega omrežja preko javnega omrežja, zagotavlja varno povezavo ter omogoča lažje upravljanje z lokalnim omrežjem.

[OpenSSL](https://www.openssl.org/) je orodje za kriptografijo in varno komunikacijo.

[OpenVPN](https://en.wikipedia.org/wiki/OpenVPN) je odprto kodna implementacija VPN, ki omogoča varno povezavo med dvema točkama v omrežju in se funkcionalno deluje kot razširitev lokalnega omrežja.

TUN - tunel [IP protokola](https://en.wikipedia.org/wiki/Internet_Protocol) na 3. omrežni plasti.

TAP - tunel [Ethernet protokola](https://en.wikipedia.org/wiki/Ethernet) na 2. omrežni plasti.

## Podrobna navodila

### 1. Naloga

Namestimo OpenVPN paket `openvpn` preko upravljalca paketkov našega operacijskega sistema na obeh navideznih računalnikih.

    apt install openvpn

Se premaknemo v domačo mapo `/home/USERNAME` in ustvarimo novo mapo v kateri bomo ustvarili vse potrebne datoteke za uporabo OpenVPN.

    cd /home/aleks
    mkdir vpn_simple
    cd /vpn_simple

Preden začnemo z nastavljanjem strežnika in klienta si pogledamo primera nastavitvenih datotek, ki se nahajata v mapi `/usr/share/doc/openvpn/examples/sample-config-files`.

    nano /usr/share/doc/openvpn/examples/sample-config-files/server.conf
    nano /usr/share/doc/openvpn/examples/sample-config-files/client.conf

Za prenos podatkov VPN ustvari šifriran tunel, ki v našem primeru pričakuje deljeno skrivnost, ki je simetrični ključ. Najprej ga ustvarimo na prvem navideznem računalniku ter namestimo `ssh` strežnik za varni prenos ključa na drug navidezni računalnik in omogočimo pravice za dostop ključa našemu uporabniku.

    openvpn --genkey secret key.key

    apt install openssh-server

    chown aleks:aleks key.key

Na drugem navideznem računalniku uporabimo ukaz `scp`, da varno prenesemo ključ s prvega navideznega računalnika.

    scp aleks@10.0.0.1:/home/aleks/vpn_simple/key.key /home/aleks/vpn_simple/key.key

Na prvem navideznem računalniku ustvarimo nastavitveno datoteko za OpenVPN strežnik, ki bo deloval preko protokola UDP in ustvaril tunel na 3. plasti omrežja, torej način `tun`. V nastavitveni datoteki nastavimo protokol preko katerega bo deloval VPN `proto udp4`, plast na kateri se bo izvedel tunel `dev tun`, VPN IP naslov strežnika in klienta `ifconfig 10.35.1.1 10.35.1.2` ter ključ za šifriranje `secret key.key`  in način šifriranja `cipher AES-256-CBC`.

    nano server_tun.conf

    proto udp4
	dev tun
	ifconfig 10.35.1.1 10.35.1.2
	secret key.key
	cipher AES-256-CBC

Na drugem navideznem računalniku ustvarimo nastavitveno datoteko za OpenVPN klient, ki bo deloval preko protokola UDP in tunela na 3. plasti omrežja, torej v načinu `tun`. V nastavitveni datoteki nastavimo zunanji IP naslov OpenVPN strežnika `remote 10.0.0.1`, protokol preko katerega bo dostopal do VPN `proto udp4`, plast na kateri se bo izvedel tunel `dev tun`, VPN IP naslov klienta in strežnika `ifconfig 10.35.1.2 10.35.1.1` ter ključ za šifriranje `secret key.key` in način šifriranja `cipher AES-256-CBC`.

    nano client_tun.conf

	remote 10.0.0.1
	proto udp4
	dev tun
	ifconfig 10.35.1.2 10.35.1.1
	secret key.key
	cipher AES-256-CBC
    
Najprej poženemo OpenVPN strežnik an prvem navideznem računalniku.

    openvpn server_tun.conf

Nato pa poženemo OpenVPN klienta na drugem navideznem računalniku in preverimo povezljivost z ukazom `ping`.

    openvpn client_tun.conf

    ping 10.35.1.1

### 2. Naloga

Ustvarimo OpenVPN strežnik, ki bo deloval preko protokola TCP in ustvaril tunel na 2. plasti omrežja, torej način `tap`. V nastavitveni datoteki nastavimo protokol preko katerega bo deloval VPN `proto tcp4-server`, plast na kateri se bo izvedel tunel `dev tap` ter ključ za šifriranje `secret key.key` in način šifriranja `cipher AES-256-CBC`.

    nano server_tap.conf

    proto tcp4-server
    dev tap
    secret key.key
    cipher AES-256-CBC

Na drugem navideznem računalniku ustvarimo nastavitveno datoteko za OpenVPN klient, ki bo deloval preko protokola TCP in tunela na 2. plasti omrežja, torej v načinu `tap`. V nastavitveni datoteki nastavimo zunanji IP naslov OpenVPN strežnika `remote 10.0.0.1`, protokol preko katerega bo dostopal do VPN `proto tcp4-client`, plast na kateri se bo izvedel tunel `dev tap` ter ključ za šifriranje `secret key.key` in način šifriranja `cipher AES-256-CBC`.

    nano client_tap.conf

    remote 10.0.0.1
    proto tcp4-client
    dev tap
    secret key.key
    cipher AES-256-CBC

Najprej poženemo OpenVPN strežnik an prvem navideznem računalniku. Ker je tunel na 2. plasti omrežja, moramo še ročno dodati IP naslov našemu strežniku z ukazom `ip`.

    openvpn server_tap.conf

    ip addr add 10.35.1.1/24 dev tap0

    ip link set dev tap0 up

Nato pa poženemo OpenVPN klienta na drugem navideznem računalniku in ročno dodamo IP naslov našemu klientu z ukazom `ip` ter preverimo povezljivost z ukazom `ping`.

    openvpn client_tap.conf

    ip addr add 10.35.1.2/24 dev tap0

    ip link set dev tap0 up

    ping 10.35.1.1

In nato preizkusimo delovanje še s prvega navideznega računalnika.

    ping 10.35.1.2

### 3. Naloga

Zdaj pa ustvarimo OpenVPN z uporabo sodobnega nadomestka za deljeni šifrirni ključ z uporabo samopodpisanih prstnih odtisov potrdil. Na strežniku najprej ustvarimo strežniško potrdilo s parom zasebnega in javnega ključa ter preberemo njegov prstni odtis.

	openssl req -x509 -newkey ec:<(openssl ecparam -name secp384r1) -keyout server.key -out server.crt -nodes -sha256 -days 3650 -subj "/CN=openvpn-server"

	openssl x509 -fingerprint -sha256 -noout -in server.crt

	sha256 Fingerprint=72:4A:87:DA:AF:E4:A9:5A:88:B0:C3:38:4E:5B:37:AD:E3:CC:A4:A6:73:54:09:F0:E7:D4:04:AA:D0:59:5B:72

Zdaj naredimo enako na odjemalcu.

	openssl req -x509 -newkey ec:<(openssl ecparam -name secp384r1) -keyout client.key -out client.crt -nodes -sha256 -days 3650 -subj "/CN=openvpn-client"

	openssl x509 -fingerprint -sha256 -noout -in client.crt

	sha256 Fingerprint=2B:15:CF:CB:A4:6A:6F:F7:D4:B6:F9:6A:6D:8E:98:73:B6:8C:B2:BD:42:AA:13:BC:82:3F:F5:A5:E6:73:55:E2

Na strežniku ustvarimo novo konfiguracijsko datoteko, ki uporablja prstni odtis odjemalca, in jo zaženemo.

	nano server_tun_tls.conf

	proto udp4
	dev tun
	ifconfig 10.35.1.1 10.35.1.2
	tls-server
	dh none
	cert server.crt
	key  server.key
	data-ciphers AES-256-GCM
	peer-fingerprint 2B:15:CF:CB:A4:6A:6F:F7:D4:B6:F9:6A:6D:8E:98:73:B6:8C:B2:BD:42:AA:13:BC:82:3F:F5:A5:E6:73:55:E2

	openvpn server_tun_tls.conf

Na odjemalcu ustvarimo novo konfiguracijsko datoteko, ki uporablja prstni odtis strežnika, in jo zaženemo.

	nano client_tun_tls.conf

	remote 10.0.1.1
	proto udp4
	dev tun
	ifconfig 10.35.1.2 10.35.1.1
	tls-client
	cert client.crt
	key  client.key
	data-ciphers AES-256-GCM
	peer-fingerprint 72:4A:87:DA:AF:E4:A9:5A:88:B0:C3:38:4E:5B:37:AD:E3:CC:A4:A6:73:54:09:F0:E7:D4:04:AA:D0:59:5B:72

	openvpn client_tun_tls.conf
	
In nato preizkusimo tudi povezavo s strani odjemalca.

    ping 10.35.1.1

