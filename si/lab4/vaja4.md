# 4. Vaja: Zagon sistemskega nalagalnika preko omrežja

## Navodila

0. Uporabili bomo omrežje in navidezne računalnike iz prejšnjih vaje. 
1. Namestite ali prilagodite DHCP strežnik za zagon sistemskega nalagalnika preko omrežja?
2. Namestite ali prilagodite TFTP strežnik za zagon sistemskega nalagalnika preko omrežja?
3. Pripravite sistemski nalagalnik s pripadajočimi datotekami v mapi, ki jo ponuja TFTP strežnik.
4. Poženite klienta v načinu za omrežni zagon in naj pridobi sistemski nalagalnik preko omrežja.

## Dodatne informacije

Zagon Linux operacijskega sistema ima naslednje korake:

1. Ob pritisku na gumb za zagon računalnika procesor začne z izvajanjem kode na v naprej določenem naslovu, na primer na 0xFFFF0 pri prvih Intel procesorjih.
2. Izvajati začne ukaze shranjene v [Basic Input Output System (BIOS)](https://en.wikipedia.org/wiki/BIOS) sistemu, ki se nahaja na ločenem pomnilniku na matični plošči, ki zazna in upravlja s strojno opremo ter preda izvajanje sistemskemu nalagalniku. Izvede se [Power-On Self-Test (POST)](https://en.wikipedia.org/wiki/Power-on_self-test) proces, ki zazna ter preveri strojno opremo, na primer procesor, pomnilnik, grafična kartica, trdi diski ter ostale vhodno/izhodno naprave.
3. Nato se izvedejo še BIOS razširitve (BIOS Extensions), ki omogočajo izvedbo ukazov shranjenih v BIOS pomnilnikih razširitvenih kartic za njihov zagon, na primer mrežne kartice, diskovni krmilniki, grafični pospeševalniki in ostale naprave.
4. BIOS prebere prvih 512B na izbranem nosilcu in jih zažene, omenjenemu programu nudi tudi možnost za dostop do nadaljnjih podatkov na napravi. Teh prvih 512B na nosilcu imenujemo [Master Boot Record (MBR)](https://en.wikipedia.org/wiki/Master_boot_record), ki v prvih 446B vsebuje sistemski nalagalnik, nato v naslednjih 64B tabelo razdelkov in v zadnjih 2B še podpis. MBR se lahko nahaja na trdem disku, USB prenosnem pomnilniku, CD ali DVD nosilcu.
6. [Sistemski nalagalnik (Bootloader)](https://en.wikipedia.org/wiki/Bootloader) v MBR poskrbi za zagon operacijskega sistema. Ker sistemski nalagalnik za sodobni operacijski sistem potrebuje več prostora kot 446B, ga razdelimo v dva dela. 1. stopnja sistemskega nalagalnika se nahaja v MBR in poskrbi za zagon 2. stopnje sistemskega nalagalnika, ki se nahaja v eni od razdelkov na podatkovnem nosilcu.
7. Sistemski nalagalnik na 2. stopnji poskrbi za zagon operacijskega sistema, tako da požene [jedro (kernel)](https://en.wikipedia.org/wiki/Linux_kernel) z dodatnimi parametri ter [začetni navidezni disk (initial RAM disk - initrd or initramfs)](https://en.wikipedia.org/wiki/Initial_ramdisk) z začasnim datotečnim izhodiščnim datotečnim sistemom.
8. Nato se zažene [prvi program (init)](https://en.wikipedia.org/wiki/Init) na operacijskem sistemu po poskrbi za zagon in dobi procesorsko oznako 1.
9. Sedaj sledi zagon uporabniških programov, grafičnega okolja in drugih programov.

Pogosti sistemski nalagalniki:

- [GNU GRand Unified Bootloader (GRUB)](https://en.wikipedia.org/wiki/GNU_GRUB)
- [SYSLINUX](https://wiki.syslinux.org/wiki/index.php?title=SYSLINUX)
- [ISOLINUX](https://wiki.syslinux.org/wiki/index.php?title=ISOLINUX)
- [PXELINUX](https://wiki.syslinux.org/wiki/index.php?title=PXELINUX)
- [LILO](https://en.wikipedia.org/wiki/LILO_(bootloader))

Ukaz [`mount`](https://linux.die.net/man/8/mount) nam omogoča priključitev datotečnega sistema na nosilcu, da lahko do njega dostopamo v našem operacijskem sistemu.

## Podrobna navodila

### 1. Naloga

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

Omrežni kartici nastavimo v datoteki `/etc/network/interfaces` in sicer tako, da `enp0s3` predstavlja omrežno kartico v omrežju `NAT`, ki pridobi omrežni naslov avtomatsko preko DHCP in `enp0s8` predstavlja omrežno kartico v omrežju `Notranje omrežje`, ki ima statični naslov, ker bo preko nje deloval DHCP strežnik.

    nano /etc/network/interfaces

    auto enp0s3
    iface enp0s3 inet dhcp

    auto enp0s8
    iface enp0s8 inet static
      address 10.0.0.1
      netmask 255.255.255.0

Da se nastavitve upoštevajo, ponovno zaženemo delovanja omrežnih kartic na navideznemu računalniku.

    systemctl restart networking.service

Sedaj namestimo DHCP strežnik, na primer `isc-dhcp-server`.

    apt update
    apt install isc-dhcp-server

V datoteki `/etc/default/isc-dhcp-server` nastavimo omrežno kartico na kateri naj deluje `isc-dhcp-server` DHCP strežnik.

    nano /etc/default/isc-dhcp-server

    INTERFACESv4="enp0s8"

V datoteki `/etc/dhcp/dhcpd.conf` pa nastavimo katero omrežje bo upravljal DHCP strežnik, torej katere IP omrežne naslove bo dodeljeval omrežnim napravam, IP naslov glavnega prehoda omrežja (angl. gateway), IP naslov [DNS](https://en.wikipedia.org/wiki/Domain_Name_System) strežnika (na primer javni Cloudflare DNS strežnik z IP naslovom `1.1.1.1`), nastavimo ime sistemskega nalagalnika, ki ga bomo ponudili preko omrežja v okolju [Preboot Execution Environment (PXE)](https://en.wikipedia.org/wiki/Preboot_Execution_Environment) ter IP strežnika, ki ga ponuja.

    nano /etc/dhcp/dhcpd.conf
	
    subnet 10.0.0.0 netmask 255.255.255.0 {
	  range 10.0.0.100 10.0.0.200;
	  option routers 10.0.0.1;
	  option domain-name-servers 1.1.1.1;
      filename "pxelinux.0";
      next-server 10.0.0.1;
	}

Nato omogočimo usmerjanje v datoteki `/etc/sysctl.conf`.

    nano /etc/sysctl.conf

    net.ipv4.ip_forward=1

Da se sprememba parametrov jedra Linux-a upošteva, uporabimo ukaz `sysctl`.

    sysctl -p

Nato pa še nastavimo preslikovanje IP omrežni naslovov. Če paketa `iptables` še nimamo nameščenega, ga namestimo z upravljalcem paketov operacijskega sistema.

    apt install iptables

    iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

Poskrbimo da se pravila, ki jih vnesemo v `iptables`, ohranijo, tako da namestimo `iptables-persistent` in jih shranimo.

    apt install iptables-persistent

Da se nastavitve upoštevajo, ponovno zaženemo `isc-dhcp-server` DHCP strežnik.

    systemctl restart isc-dhcp-server.service

### 2. Naloga

Namestimo TFTP strežnik, na primer `tftpd`.

    apt install tftpd

Nameščeni `tftpd` strežnik že deluje s privzetimi nastavitvami, ki so podane v nastavitveni datoteki `/etc/inetd.conf`.

    nano /etc/inetd.conf

    tftp dgram udp wait nobody /usr/sbin/tcpd /usr/sbin/in.tftpd /srv/tftp

Ustvarimo mapo `/srv/tftp`, katero bo TFTP ponudil preko omrežja.

    mkdir /srv/tftp

V primeru, da z upravljalcem paketov ne najdete paketa `tftpd` potem namestite paket `tftpd-hpa` in v nastavitveni datoteki `/etc/default/tftpd-hpa` omogočite beleženje z zastavico `-v` in dostopnost novo dodanih datotek v mapi `/srv/tftp` z zastavico `-c`.

    apt install tftpd-hpa

    nano /etc/default/tftpd-hpa

    # /etc/default/tftpd-hpa

    TFTP_USERNAME="tftp"
    TFTP_DIRECTORY="/srv/tftp"
    TFTP_ADDRESS=":69"
    TFTP_OPTIONS="-v -c --secure"

### 3. Naloga

Sedaj potrebuje sistemski nalagalnik, ki omogoča zagon preko omrežja. Na primer, uporabimo `pxelinux`, tako da ga najprej pridobimo z upravljavcem paketov in nato prestavimo v mapo `/srv/tftp` da postane dostopen preko TFTP protokola v omrežju.

    apt install pxelinux
    cd /srv/tftp
    cp /usr/lib/PXELINUX/pxelinux.0 .

### 4. Naloga

Drugi navidezni računalnik nastavimo za zagon preko omrežja tako da kliknemo na gumb `Nastavitve ...` v meniju zgoraj in pod zavihkom `Sistem\Matična plošča` in oznako `Vrstni red zagona:` izberemo `Omrežje` ter vse ostale možnosti odkljukamo.

![Nastavite omrežnega zagona za navidezni računalnik v VirtualBox.](slike/vaja4-vbox1.png)

Drugi navidezni računalnik sedaj poženemo in ta požene PXE okolje za zagon preko omrežja. Navidezni računalnik uspešno pridobi IP naslov s strani našega DHCP strežnika, dobi IP naslov TFTP strežnika ter ime sistemskega nalagalnika, ki ga mora prenesti. Nato uspešno prenese sistemski nalagalnik `pxelinux.0` in ga požene. Izvajanje sistemskega nalagalnika se vstavi pri manjkajoči datoteki `ldlinux.c32`.

![Zagon navideznega računalnika v PXE okolje za omrežni zagon.](slike/vaja4-vbox2.png)

Komunikaciji med strežnikom in klientom lahko sledimo tudi z beležkami v `/var/log/syslog`, ki se samodejno posodabljajo. Najprej vidimo pridobivanje IP naslova preko DHCP strežnika, nato preko protokola TFTP prenos sistemskega nalagalnika `pxelinux.0` in nato iskanje datoteke `ldlinux.c32` na različnih mestih v datotečnem sistemu. Ker datoteke `ldlinux.c32` ni, se postopek omrežnega zagona sistema neuspešno zaključi.  

    tail -f /var/log/syslog

    Nov  3 15:27:06 debian dhcpd[679]: DHCPDISCOVER from 08:00:27:4f:53:22 via enp0s8
    Nov  3 15:27:07 debian dhcpd[679]: DHCPOFFER on 10.0.0.100 to 08:00:27:4f:53:22 via enp0s8
    Nov  3 15:27:09 debian dhcpd[679]: reuse_lease: lease age 82 (secs) under 25% threshold, reply with unaltered, existing lease for 10.0.0.100
    Nov  3 15:27:09 debian dhcpd[679]: DHCPREQUEST for 10.0.0.100 (10.0.0.1) from 08:00:27:4f:53:22 via enp0s8
    Nov  3 15:27:09 debian dhcpd[679]: DHCPACK on 10.0.0.100 to 08:00:27:4f:53:22 via enp0s8
    Nov  3 15:27:09 debian in.tftpd[3304]: connect from 10.0.0.100 (10.0.0.100)
    Nov  3 15:27:09 debian tftpd[3305]: tftpd: trying to get file: pxelinux.0
    Nov  3 15:27:09 debian tftpd[3305]: tftpd: serving file from /srv/tftp
    Nov  3 15:27:09 debian in.tftpd[3306]: connect from 10.0.0.100 (10.0.0.100)
    Nov  3 15:27:09 debian tftpd[3307]: tftpd: trying to get file: ldlinux.c32
    Nov  3 15:27:09 debian tftpd[3307]: tftpd: serving file from /srv/tftp
    Nov  3 15:27:09 debian in.tftpd[3308]: connect from 10.0.0.100 (10.0.0.100)
    Nov  3 15:27:09 debian tftpd[3309]: tftpd: trying to get file: /boot/isolinux/ldlinux.c32
    Nov  3 15:27:09 debian in.tftpd[3310]: connect from 10.0.0.100 (10.0.0.100)
    Nov  3 15:27:09 debian tftpd[3311]: tftpd: trying to get file: /isolinux/ldlinux.c32
    Nov  3 15:27:09 debian in.tftpd[3312]: connect from 10.0.0.100 (10.0.0.100)
    Nov  3 15:27:09 debian tftpd[3313]: tftpd: trying to get file: /boot/syslinux/ldlinux.c32
    Nov  3 15:27:09 debian in.tftpd[3314]: connect from 10.0.0.100 (10.0.0.100)
    Nov  3 15:27:09 debian tftpd[3315]: tftpd: trying to get file: /syslinux/ldlinux.c32
    Nov  3 15:27:09 debian in.tftpd[3316]: connect from 10.0.0.100 (10.0.0.100)
    Nov  3 15:27:09 debian tftpd[3317]: tftpd: trying to get file: /ldlinux.c32

V novejših različicah operacijskega sistema Linux se pa beležke beležijo v beležkah programa `systemd`, do katerih dostopamo preko ukaza `journalctl`. Zastavica `-u` nam nastavi spremljanje beležk le določenega programa, ter zastavica `-f` nam omogoča samodejno prikazovanje novih zapisov.

    journalctl -u tftpd-hpa.service -f

    Oct 31 19:18:18 debian systemd[1]: Starting tftpd-hpa.service - LSB: HPA's tftp server...
    Oct 31 19:18:18 debian tftpd-hpa[786]: Starting HPA's tftpd: in.tftpd.
    Oct 31 19:18:18 debian systemd[1]: Started tftpd-hpa.service - LSB: HPA's tftp server.
    Oct 31 19:34:02 debian in.tftpd[2776]: RRQ from 10.0.1.100 filename pxelinux.0
    Oct 31 19:34:02 debian in.tftpd[2777]: RRQ from 10.0.1.100 filename ldlinux.c32
    Oct 31 19:34:02 debian in.tftpd[2777]: sending NAK (1, File not found) to 10.0.1.100
    Oct 31 19:34:02 debian in.tftpd[2778]: RRQ from 10.0.1.100 filename /boot/isolinux/ldlinux.c32
    Oct 31 19:34:02 debian in.tftpd[2778]: sending NAK (1, File not found) to 10.0.1.100
    Oct 31 19:34:02 debian in.tftpd[2779]: RRQ from 10.0.1.100 filename /isolinux/ldlinux.c32
    Oct 31 19:34:02 debian in.tftpd[2779]: sending NAK (1, File not found) to 10.0.1.100
    Oct 31 19:34:02 debian in.tftpd[2780]: RRQ from 10.0.1.100 filename /boot/syslinux/ldlinux.c32
    Oct 31 19:34:02 debian in.tftpd[2780]: sending NAK (1, File not found) to 10.0.1.100
    Oct 31 19:34:02 debian in.tftpd[2781]: RRQ from 10.0.1.100 filename /syslinux/ldlinux.c32
    Oct 31 19:34:02 debian in.tftpd[2781]: sending NAK (1, File not found) to 10.0.1.100
    Oct 31 19:34:02 debian in.tftpd[2782]: RRQ from 10.0.1.100 filename /ldlinux.c32
    Oct 31 19:34:02 debian in.tftpd[2782]: sending NAK (1, File not found) to 10.0.1.100

Manjkajoča datoteka je del modulov, ki so potrebni za zagon sistemskega nalagalnika, ki pa potrebuje tudi nastavitveno datoteko. Do datotek najlažje pridemo, tako da vzamemo namestitveno sliko Linux distribucije, ki uporablja `isolinux` sistemski nalagalnik in jih prekopiramo v mapo `/srv/tftp`. Kasneje bomo uporabili še ostale datoteke, da uspešno zaženemo operacijski sistem do konca. Na primer, prenesemo namestitveno sliko distribucije [Mint](https://linuxmint.com/download.php) in ga vstavimo v prvi navidezni računalnik, tako da jo izberemo v meniju `Naprave\Optični pogoni\Izberi datoteko diska ...`. Slika je sedaj vstavljena in do nje dostopamo preko `/dev/sr0`, za dostop do datotek, jo moramo pa še priključiti z ukazom `mount` v poljubno prazno mapo. 

    mount /dev/sr0 /mnt

Manjkajoče datoteke prenesemo iz mape `isolinux` na priključeni namestitveni sliki.

    cp /mnt/isolinux/ldlinux.c32 .
    cp /mnt/isolinux/libutil.c32 .
    cp /mnt/isolinux/vesamenu.c32 .
    cp /mnt/isolinux/libcom32.c32 .
    cp /mnt/isolinux/splash.png .
	cp /mnt/isolinux/live.cfg .

Manjka nam še nastavitvena datoteka sistemskega nalagalnika, za katero lahko uporabimo datoteki `isolinux.cfg` in `live.cfg` z namestitvene slike. Sistemski nalagalnik `pxelinux` lahko hkrati ponudi tudi več nastavitvenih datotek, ki so shranjene v mapi `pxelinux.cfg` in vanjo prenesemo `isolinux.cfg` ter jo preimenujemo v `default`.

    mkdir pxelinux.cfg
    cp /mnt/isolinux/isolinux.cfg ./pxelinux.cfg/default

Sedaj ponovno zaženemo drugi navidezni računalnik, ki uspešno zažene sistemski nalagalnik s podanimi nastavitvami.

![Zagon sistemskega nalagalnika na drugem navideznem računalniku.](slike/vaja4-vbox3.png)
