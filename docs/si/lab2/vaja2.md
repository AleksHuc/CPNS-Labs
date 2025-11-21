# 2. Vaja: Vzpostavitev omrežja z avtomatskim dodeljevanjem omrežnih naslovov 

## Navodila

1. Postavite dva navidezna računalnika. Prvi ima dve omrežni kartici, kjer je prva povezana na omrežje `NAT` in druga na `Notranje omrežje`. Drugi ima eno omrežno kartico, ki je povezana na`Notranje omrežje`.
2. Postavite [Dynamic Host Configuration Protocol (DHCP)](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) strežnik na prvem navideznem računalniku, ki ima dve omrežno kartici. DHCP naj deluje na omrežju `Notranje omrežje` in omogoča drugim navideznim računalnikov avtomatsko pridobivanje omrežnih naslovov.
3. Poskrbite, da navidezni računalnik, ki je povezano samo v `Notranje omrežje` lahko dostopa do spleta preko navideznega računalnika na katerem teče DHCP strežnik.

## Dodatne informacije

Ukaz [`sysctl`](https://linux.die.net/man/8/sysctl) nam omogoča upravljanje s parametri jedra Linux operacijskega sistema med izvajanjem.

Navodila za nastavitev omrežja v nastavitveni datoteki [`/etc/network/interfaces`](https://manpages.debian.org/stretch/ifupdown/interfaces.5.en.html).

Požarni zid [`nftables`](https://manpages.debian.org/buster/nftables/nft.8.en.html) omogoča upravljanje in filtriranje omrežnih paketov, ki vstopajo ali prehajajo posamezni omrežni vmesnik.

Ukaz [`journalctl`](https://www.man7.org/linux/man-pages/man1/journalctl.1.html) nam omogoča branje `systemd` beležk oz logov.

Ukaz [`systemctl`](https://www.man7.org/linux/man-pages/man1/systemctl.1.html) nam omogoča upravljanje s stanjem `systemd` sistema in upravljanje s programi, ki se izvajajo v ozadju.

## Podrobna navodila

### 1. Naloga

Ustvarimo dva navidezna računalnika kot smo to naredili pri prejšnjih vajah, kjer ima prvi dve omrežni kartici, kjer je prva povezana na omrežje `NAT` in druga na `Notranje omrežje`.

![Nastavitev prve omrežne kartice na prvem navideznem računalniku na omrežje `NAT`.](slike/vaja2-vbox1.png)

![Nastavitev druge omrežne kartice na prvem navideznem računalniku na `Notranje omrežje`.](slike/vaja2-vbox2.png)

Drugi ima pa eno omrežno kartico, ki je povezana na `Notranje omrežje`. Paziti moramo, da pri obeh navideznih računalnikih na omrežnih karticah z omrežjem `Notranje omrežje` izberemo enako ime, na primer `intnet`.

![Nastavitev prve omrežne kartice na drugem navideznem računalniku na `Internal network`.](slike/vaja2-vbox3.png)

### 2. Naloga

Zaženemo prvi navidezni računalnik ter preverimo stanje na obeh nastavljenih omrežnih karticah z ukazom `ip`.

    ip a

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host valid_lft forever preferred_lft forever
    2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:26:d5:82 brd ff:ff:ff:ff:ff:ff
    3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:86:d1:5b brd ff:ff:ff:ff:ff:ff
        inet6 fe80::a00:27ff:fe86:d15b/64 scope link noprefixroute valid_lft forever preferred_lft forever

Dobimo naslednji izpis, iz katerega razberemo da obe mrežni kartici nista pridobili IP omrežnih naslovov, kar je posledica manjkajočih nastavitev in delovanja programa `network-manager`, ki upravlja z omrežji. Za nadaljnjo nastavljanja omrežja bomo izklopili `network-manager` in pripravili potrebne nastavitve datoteke, da dosežemo željeno delovanje.

    su -
    systemctl stop NetworkManager.service
    systemctl disable NetworkManager.service

Omrežni kartici nastavimo v datoteki `/etc/network/interfaces` in sicer tako, da `enp0s3` predstavlja omrežno kartico v omrežju `NAT`, ki pridobi omrežni naslov avtomatsko preko DHCP in `enp0s8` predstavlja omrežno kartico v omrežju `Notranje omrežje`, ki ima statični naslov, ker bo preko nje deloval DHCP strežnik. Nastavitveno datoteko odpremo s poljubni urejevalnikom besedil in dodamo naslednje nastavitve.

    nano /etc/network/interfaces

    auto enp0s3
    iface enp0s3 inet dhcp

    auto enp0s8
    iface enp0s8 inet static
      address 10.0.1.1
      netmask 255.255.255.0

Da se nastavitve upoštevajo, ponovno zaženemo delovanja omrežnih kartic na navideznemu računalniku.

    systemctl restart networking.service

Sedaj smo pripravili vse potrebno, da namestimo in nastavimo [DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) strežnik, ki bo dodeljeval IP omrežne naslove avtomatsko vsem računalnikom v omrežju `Notranje omrežje`. Specifikacijo protokola lahko najdemo v [RFC2131](https://www.rfc-editor.org/rfc/rfc2131). Z upravljavcem paketov lahko iščemo po imenih paketov, ki jih lahko namestimo z ukazom `apt search NAME`. Namestimo na primer, implementacijo DHCP strežnika `isc-dhcp-server`.

    apt update
    apt install isc-dhcp-server

Po koncu namestitve nam DHCP strežni sporoči napako, da se ni mogel zagnati. Do izpisan napake pridemo, tako da pregledamo najnovejše beležke in ugotovimo, da manjkajo nastavitve omrežne kartice, na kateri bo deloval ter nastavitve podomrežja, ki ga bo upravljal.

    journalctl -xe -n 100

V datoteki `/etc/default/isc-dhcp-server` nastavimo omrežno kartico na kateri naj deluje `isc-dhcp-server` DHCP strežnik.

    nano /etc/default/isc-dhcp-server

    INTERFACESv4="enp0s8"

V datoteki `/etc/dhcp/dhcpd.conf` pa nastavimo katero omrežje bo upravljal DHCP strežnik, torej katere IP omrežne naslove bo dodeljeval omrežnim napravam, IP naslov glavnega prehoda omrežja (angl. gateway), IP naslov [DNS](https://en.wikipedia.org/wiki/Domain_Name_System) strežnika (na primer javni Cloudflare DNS strežnik z IP naslovom `1.1.1.1`) ter tudi ostale lastnosti omrežja.

    nano /etc/dhcp/dhcpd.conf
	
    subnet 10.0.1.0 netmask 255.255.255.0 {
	  range 10.0.1.100 10.0.1.200;
	  option routers 10.0.1.1;
	  option domain-name-servers 1.1.1.1;
	}

Da se nastavitve upoštevajo, ponovno zaženemo `isc-dhcp-server` DHCP strežnik.

    systemctl restart isc-dhcp-server.service

Da preverimo delovanje našega DHCP strežnika, sedaj poženemo drugi navidezni računalnik in preverimo njegov IP omrežni naslov. Če DHCP pravilno deluje, bi moral drugi navidezni računalnik dobiti naslov iz bloka IP naslovov od 10.0.1.100 do 10.0.1.200 avtomatično, če na njem teče `network-manager`. Če sta oba računalnika dostopna preko omrežje preverimo z ukazom `ping`, kot smo to storili na prejšnjih vaja.

    ip a

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host valid_lft forever preferred_lft forever
    2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:40:07:ad brd ff:ff:ff:ff:ff:ff
        inet 10.0.1.100/24 brd 10.0.1.255 scope global dynamic noprefixroute enp0s3 valid_lft 514sec preferred_lft 514sec
        inet6 fe80::a00:27ff:fe40:7ad/64 scope link noprefixroute valid_lft forever preferred_lft forever

Na drugemu navideznemu računalniku lahko DNS strežnike dodamo tudi ročno v datoteki `/etc/dhcp/dhclient.conf`.

    nano /etc/dhcp/dhclient.conf

    prepend domain-name-servers 1.1.1.1;

Na drugem navideznem računalniku, lahko ponovno zaženemo omrežje in tako zaprosimo DHCP za nov IP naslov in tako pridobimo nove nastavitve za DNS.

    systemctl restart isc-dhcp-server.service

### 3. Naloga

Dosegli smo, da navidezni računalnik poganja DHCP strežnik, ki avtomatsko dodeljuje IP omrežne naslove v omrežju `Notranje omrežje`, kar smo preverili z drugim navideznim računalnikom. Ob preskušanju delovanja omrežja smo z ukazom `ping` ugotovili, da drugi navidezni računalnik lahko dostopa do prvega in obratno, vendar pa nima dostopa do spleta. Sedaj bomo odpravili to pomanjkljivost, tako da bomo vklopili omrežno usmerjanje na prvem navideznem računalniku.

    ping 10.0.1.1 -c 4

    PING 10.0.1.1 (10.0.1.1) 56(84) bytes of data.
    64 bytes from 10.0.1.1: icmp_seq=1 ttl=64 time=1.81 ms
    64 bytes from 10.0.1.1: icmp_seq=2 ttl=64 time=2.89 ms
    64 bytes from 10.0.1.1: icmp_seq=3 ttl=64 time=0.679 ms
    64 bytes from 10.0.1.1: icmp_seq=4 ttl=64 time=1.56 ms

    --- 10.0.1.1 ping statistics ---
    4 packets transmitted, 4 received, 0% packet loss, time 3006ms
    rtt min/avg/max/mdev = 0.679/1.734/2.890/0.788 ms

    ping 8.8.8.8 -c 4

    PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.

    --- 8.8.8.8 ping statistics ---
    4 packets transmitted, 0 received, 100% packet loss, time 3062ms


Najprej omogočimo usmerjanje na prvem navideznem računalnik v datoteki `/etc/sysctl.d/sysctl.conf`.

    nano /etc/sysctl.d/sysctl.conf

    net.ipv4.ip_forward=1

Da se sprememba parametrov jedra Linux-a upošteva, uporabimo ukaz `sysctl`.

    sysctl -p /etc/sysctl.d/sysctl.conf

Z upravljalcem paketov namestimo paket `iptables`, ki predstavlja požarni zid in omogoča preslikovanje IP omrežni naslovov, ki ga bo izvedel naš prvi navidezni računalnik.

    apt install iptables

    iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

Vnos pravila v `iptables` lahko tudi preverimo.

    sudo iptables -t nat -L -v

Sedaj ponovno preverimo dostopnost lokalnih in javnih IP omrežnih naslovov.

    ping 10.0.1.1 -c 4

    PING 10.0.1.1 (10.0.1.1) 56(84) bytes of data.
    64 bytes from 10.0.1.1: icmp_seq=1 ttl=64 time=169 ms
    64 bytes from 10.0.1.1: icmp_seq=2 ttl=64 time=1.64 ms
    64 bytes from 10.0.1.1: icmp_seq=3 ttl=64 time=1.69 ms
    64 bytes from 10.0.1.1: icmp_seq=4 ttl=64 time=1.97 ms

    --- 10.0.1.1 ping statistics ---
    4 packets transmitted, 4 received, 0% packet loss, time 3021ms
    rtt min/avg/max/mdev = 1.638/43.630/169.224/72.511 ms

    ping 8.8.8.8 -c 4

    PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
    64 bytes from 8.8.8.8: icmp_seq=1 ttl=61 time=17.4 ms
    64 bytes from 8.8.8.8: icmp_seq=2 ttl=61 time=15.0 ms
    64 bytes from 8.8.8.8: icmp_seq=3 ttl=61 time=15.9 ms
    64 bytes from 8.8.8.8: icmp_seq=4 ttl=61 time=13.9 ms

    --- 8.8.8.8 ping statistics ---
    4 packets transmitted, 4 received, 0% packet loss, time 3004ms
    rtt min/avg/max/mdev = 13.862/15.541/17.444/1.313 ms

Pravila, ki jih vnesemo v `iptables` se ne ohranijo ob ponovnem zagonu sistema. Z uporabo paketa `iptables-persistent`, jih lahko shranimo ter omogočimo, da se avtomatsko upoštevajo ob ponovnem zagonu.

    apt install iptables-persistent
    iptables-save > /etc/iptables/rules.v4