# 4. Vaja: Zagon sistemskega nalagalnika preko omrežja

## Navodila

0. Uporabili bomo omrežje in navidezne računalnike iz prejšnjih vaje. 
1. Namestite ali prilagodite DHCP strežnik za zagon sistemskega nalagalnika preko omrežja?
2. Namestite ali prilagodite TFTP strežnik za zagon sistemskega nalagalnika preko omrežja?
3. Pripravite sistemski nalagalnik s pripadajočimi datotekami v mapi, ki jo ponuja TFTP strežnik.
4. Poženite klienta v načinu za omrežni zagon in naj pridobi sistemski nalagalnik preko omrežja.

## Dodatne informacije

Zagon Linux operacijskega sistema ima naslednje korake:

### Zagon z BIOS

**1. Fizični zagon računalnika:** Po pritisku na gumb [CPU](https://en.wikipedia.org/wiki/Central_processing_unit) začne na vnaprej določenem naslovu (reset vektor; pri procesorjih [x86](https://en.wikipedia.org/wiki/X86) tipično `0xFFFFFFF0`, tako pri 32 in 64-bitnih različicah).

**2. Zagon strojne opreme:** [Basic Input Output System (BIOS)](https://en.wikipedia.org/wiki/BIOS), ki se nahaja na ločenem pomnilniku ([Read Only Memory - ROM](https://en.wikipedia.org/wiki/Read-only_memory)) na matični plošči, izvede [Power-On Self-Test (POST)](https://en.wikipedia.org/wiki/Power-on_self-test) ki, zazna inicializira ter preiskusi strojno opremo, na primer procesor, pomnilnik, grafično kartico, trdi diski ter ostale vhodno/izhodno naprave in jih zažene. Po potrebi se izvedejo še [BIOS razširitve (BIOS Extensions)](https://en.wikipedia.org/wiki/BIOS#Extensions_(option_ROMs)), ki omogočajo izvedbo ukazov shranjenih v BIOS pomnilnikih razširitvenih kartic za njihov zagon, na primer mrežne kartice, diskovni krmilniki, grafični pospeševalniki in ostale naprave.

**3. Izbira zagonske naprave:** Glede na vrstni red zagona BIOS naloži [Master Boot Record (MBR)](https://en.wikipedia.org/wiki/Master_boot_record) (prvih 512 B) iz izbranega diska v `0x7C00` in skoči tja.

Struktura MBR:

- 446 B zagonska koda,
- 64 B (4×16 B) tabela razdelkov,
- 2 B podpis `0x55AA`.

**4. Sistemski nalagalnik:** Zagonska koda ali [Sistemski nalagalnik (Bootloader)](https://en.wikipedia.org/wiki/Bootloader) v MBR v enem ali več korakih poskrbi za zagon operacijskega sistema. 1. stopnja v MBR je zelo majhna; naloži stopnjo 1.5 (v `post-MBR` prostoru ali v `BIOS Boot` razdelku pri [GUID Partition Table (GPT)](https://en.wikipedia.org/wiki/GUID_Partition_Table)) in nato 2. stopnjo kot datoteko v direktorju `/boot`. 

**5. Nalaganje jedra:** 2. stopnja sistemskega nalagalnika naloži Linuxovo [jedro (kernel - vmlinuz)](https://en.wikipedia.org/wiki/Linux_kernel) in [začetni navidezni disk (initial RAM disk - initramfs (prej initrd))](https://en.wikipedia.org/wiki/Initial_ramdisk) ter posreduje parametre jedra.

### Zagon z UEFI

**1. Fizični zagon računalnika in 2. Zagon strojne opreme:** potekata zelo podobno, le da uporabljata [Unified Extensible Firmware Interface (UEFI)](https://en.wikipedia.org/wiki/UEFI) namesto BIOS-a (UEFI faze SEC/PEI/DXE, UEFI Extensions, UEFI gonilniki).

**3. Izbira zagonske naprave:** UEFI uporabi [NVRAM](https://wikileaks.org/ciav7p1/cms/page_26968097.html) vnose (`BootOrder`, `Boot####`) in naloži EFI aplikacijo (`*.efi`) iz ESP – EFI System Partition (FAT32, običajno 100–512 MiB).

**4. Sistemski nalagalnik:** Nalagalnik je EFI aplikacija na ESP (npr. GRUB EFI, systemd-boot, rEFInd, Limine) ali neposreden zagon jedra (EFI-stub/direct boot). Možen Secure Boot (npr. prek shim).

**5. Nalaganje jedra:** Bootloader ali UEFI (direct) naloži jedro in initramfs, posreduje cmdline in UEFI memory map/hand-off. Ob Secure Boot je veriga podpisov lahko razširjena do jedra (in pogosto initramfs).

### Kaj stori Linux po nalaganju jedra

**1. Initramfs:** Jedro razširi initramfs (cpio arhiv v RAM-u), naloži gonilnike, nastavi [`udev`](https://en.wikipedia.org/wiki/Udev), poišče korenski datotečni sistem.

**2. Preklop na root:** Priklopi korenski datotečni sistem (npr. na `/`) in izvede `switch_root/pivot_root`. Priklop datotečnega sistema na operacijski sistem se izvede z ukazom [`mount`](https://linux.die.net/man/8/mount).

**3. PID 1 – init:** Zažene [`/sbin/init`](https://en.wikipedia.org/wiki/Init) (najpogosteje [`systemd`](https://en.wikipedia.org/wiki/Systemd)), ki potem zažene storitve, nastavi tip izvajanja [`runlevel`](https://en.wikipedia.org/wiki/Runlevel), prijavni zaslon, grafično okolje ipd.

### Omrežnem zagonu in zagonu z medijev

**Optični mediji:** Zagon poteka po standardu [El Torito](https://en.wikipedia.org/wiki/ISO_9660#El_Torito); MBR se ne uporablja.

**Mrežni zagon:** [Okolje za zagon preko omrežja (Preboot Execution Environment - PXE)](https://en.wikipedia.org/wiki/Preboot_Execution_Environment) prenese zagonsko sliko prek TFTP/HTTP (pogosto PXELINUX ali iPXE).

### Pogosti sistemski nalagalniki
- [GNU GRand Unified Bootloader (GRUB)](https://en.wikipedia.org/wiki/GNU_GRUB) – zmogljiv odprtokodni nalagalnik iz projekta GNU, ki podpira več operacijskih sistemov in konfiguracij.
- [systemd-boot](https://en.wikipedia.org/wiki/Systemd-boot) – preprost UEFI zagonski upravitelj (prej znan kot gummiboot), vključen v `systemd`, ki omogoča izbiro med nameščenimi sistemi in urejanje parametrov jedra.
- [rEFInd](https://en.wikipedia.org/wiki/REFInd) - grafični zagonski upravitelj za UEFI sisteme, namenjen enostavnemu izbiranju med več operacijskimi sistemi; nastal je kot razširitev opuščenega projekta rEFIt.
- [Limine](https://wiki.archlinux.org/title/Limine) - sodoben, večplatformni zagonski nalagalnik z podporo za BIOS in UEFI, razvit kot referenčna implementacija Limine zagonskega protokola; omogoča zagon Linuxa in veriženje (chainload) drugih nalagalnikov.
- [SYSLINUX](https://wiki.syslinux.org/wiki/index.php?title=SYSLINUX) - zbirka lahkih zagonskih nalagalnikov, prvotno namenjenih BIOS/MBR sistemom. Vključuje več specializiranih različic, npr. [ISOLINUX](https://wiki.syslinux.org/wiki/index.php?title=ISOLINUX) za zagonske CD/DVD medije (ISO 9660), [PXELINUX](https://wiki.syslinux.org/wiki/index.php?title=PXELINUX) za omrežni zagon prek PXE protokola, in [EXTLINUX/EFILINUX](https://wiki.syslinux.org/wiki/index.php?title=EXTLINUX) za zagon z različnih datotečnih sistemov (tudi UEFI podpora v EFILINUX modulu).
- [LILO](https://en.wikipedia.org/wiki/LILO_(bootloader)) - starejši Linux zagonski nalagalnik za BIOS sisteme; zgodovinsko je bil privzet v mnogih distribucijah pred prehodom na GRUB (razvoj LILO je bil zaključen leta 2015).

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

Da se nastavitve upoštevajo, ponovno zaženemo `isc-dhcp-server` DHCP strežnik.

    systemctl restart isc-dhcp-server.service

Nato omogočimo usmerjanje tako, da ustvarimo datoteko `/etc/sysctl.d/sysctl.conf` in v njej omogočimo usmerjanje paketkov.

    nano /etc/sysctl.d/sysctl.conf

    net.ipv4.ip_forward=1

Da se sprememba parametrov jedra Linux-a upošteva, uporabimo ukaz `sysctl`.

    sysctl -p /etc/sysctl.d/sysctl.conf

Nato pa še nastavimo preslikovanje IP omrežni naslovov. Če paketa `iptables` še nimamo nameščenega, ga namestimo z upravljavcem paketov operacijskega sistema.

    apt install iptables

    iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

	iptables -t nat -L -v

	Chain PREROUTING (policy ACCEPT 2 packets, 1152 bytes)
 	 pkts bytes target     prot opt in     out     source               destination         

	Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 	 pkts bytes target     prot opt in     out     source               destination         

	Chain OUTPUT (policy ACCEPT 20 packets, 2816 bytes)
 	 pkts bytes target     prot opt in     out     source               destination         

	Chain POSTROUTING (policy ACCEPT 6 packets, 1122 bytes)
 	 pkts bytes target     prot opt in     out     source               destination         
   	   14  1694 MASQUERADE  all  --  any    enp0s3  anywhere             anywhere   

Poskrbimo da se pravila, ki jih vnesemo v `iptables`, ohranijo, tako da namestimo `iptables-persistent` in jih shranimo.

    apt install iptables-persistent

	iptables-save > /etc/iptables/rules.v4

### 2. Naloga

Namestimo TFTP strežnik, na primer `tftpd-hpa` in v nastavitveni datoteki `/etc/default/tftpd-hpa` omogočimo beleženje z zastavico `-v` in dostopnost novo dodanih datotek v mapi `/srv/tftp` z zastavico `-c`.

    apt install tftpd-hpa

    nano /etc/default/tftpd-hpa

    # /etc/default/tftpd-hpa

    TFTP_USERNAME="tftp"
    TFTP_DIRECTORY="/srv/tftp"
    TFTP_ADDRESS=":69"
    TFTP_OPTIONS="-v -c --secure"

	systemctl restart tftpd-hpa.service

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

Komunikaciji med strežnikom in klientom lahko sledimo tudi z beležkami v `systemd`, do katerih dostopamo preko ukaza `journalctl`. Zastavica `-u` nam nastavi spremljanje beležk le določenega programa, ter zastavica `-f` nam omogoča samodejno prikazovanje novih zapisov. Najprej vidimo pridobivanje IP naslova preko DHCP strežnika, nato preko protokola TFTP prenos sistemskega nalagalnika `pxelinux.0` in nato iskanje datoteke `ldlinux.c32` na različnih mestih v datotečnem sistemu. Ker datoteke `ldlinux.c32` ni, se postopek omrežnega zagona sistema neuspešno zaključi.  

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
