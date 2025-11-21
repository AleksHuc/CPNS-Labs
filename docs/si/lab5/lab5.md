# 5. Vaja: Zagon operacijskega sistema Linux preko omrežja

## Navodila

0. Uporabili bomo omrežje in navidezne računalnike iz prejšnjih vaje. 
1. Pripravite jedro in začetni navidezni disk.
2. Popravite nastavitve sistemskega nalagalnika, da zaženejo jedro in začetni navidezni disk.
3. Pripravite Network File System (NFS) strežnik za zagotavljanje omrežnega diska.
4. Popravite nastavitve sistemskega nalagalnika, da uporablja omrežni disk za zagon operacijskega sistema.

## Dodatne informacije

[Navodila](https://www.kernel.org/doc/html/latest/admin-guide/nfs/nfsroot.html) za priklop korenskega datotečnega sistema preko NFS (nfsroot).

## Podrobna navodila

### 1. Naloga

Jedro in začetni navidezni disk prav tako pridobimo z namestitvene slike izbranega operacijskega sistema Linux ter jih prenesemo v mapo `/srv/tftp`.

    cd /srv/tftp
    mkdir casper
    cd casper
    cp /mnt/casper/vmlinuz .
    cp /mnt/casper/initrd.lz .
    cd ..

### 2. Naloga

Preverimo vsebino nastavitvene datoteke sistemskega nalagalnika.

    nano pxelinux.cfg/default

	default vesamenu.c32
	include live.cfg

Vidimo, da datoteka `default` ne vsebuje neposrednih nastavitev vendar jih uvozi iz datoteke `live.cfg`. Zato jo odpremo in pogledamo definicijo prve vrstice menija za zagon operacijskega sistema z imenom `Start Linux Mint`.

	nano live.cfg

    label live
        menu label Start Linux Mint
        menu default
        kernel /casper/vmlinuz
        append boot=casper initrd=/casper/initrd.lz username=mint hostname=mint quiet splash --

Vidimo, da so poti do jedra in začetnega navideznega diska absolutne in jih moramo spremeniti v relativne (brez / spredaj). Prav tako odstranimo `quiet splash --` zastavice, da omogočimo izpis beležk med zagonom.

    label live
        menu label Start Linux Mint
        menu default
        kernel casper/vmlinuz
        append boot=casper initrd=casper/initrd.lz username=mint hostname=mint

### 3. Naloga

Namestimo [NFS](https://en.wikipedia.org/wiki/Network_File_System) strežnik, ki nam omogoča učinkovit dostop do datoteke preko omrežja. Na primer, namestimo `nfs-kernel-server`.

    apt install nfs-kernel-server

V nastavitveni datoteki NFS strežnika `/etc/exports` nastavimo mapo, ki jo bomo ponudili preko omrežja, na primer `/media/cdrom` in sicer vsem, ki dostopajo do strežnika ter v načinu samo za branje `*(ro)`.

    nano /etc/exports 

    /media/cdrom *(ro)

Da se nastavitve upoštevajo, ponovno zaženemo `nfs-kernel-server` NFS strežnik.

    systemctl restart nfs-kernel-server.service

Sedaj poskrbimo, da so vse datoteke namestitvene slike na voljo v tej mapi. Naprej odklopimo mapo `/mnt`, kjer imamo trenutno priklopljeno namestitveno sliko.

    umount /mnt

Nato pa priklopimo namestitveno sliko v mapo `/media/cdrom`.

    mount /dev/sr0 /media/cdrom

Delovanje NFS strežnika preskusimo lokalno tako, da sedaj poskusimo priklop omrežnega diska v mapo `/mnt`.

    mount -t nfs localhost:/media/cdrom /mnt

    ls /mnt

    umount /mnt

### 4. Naloga

Sedaj moramo še v nastavitveni datoteki sistemskega nalagalnika `live.cfg` omogočiti dostop do datotečnega sistem operacijskega sistema preko protokola NFS. Tako da nastavimo protokol za omrežni zagon na NFS `netboot=nfs`, izhodišče datotečnega sistema na mapo, ki jo ponuja NFS strežnik `nfsroot=10.0.0.1:/media/cdrom` ter dodamo še avtomatsko pridobivanje IP naslova ob zagonu operacijskega sistema `ip=dhcp`.

    nano live.cfg

	label live
        menu label Start Linux Mint
        menu default
        kernel casper/vmlinuz
        append boot=casper initrd=casper/initrd.lz username=mint hostname=mint netboot=nfs nfsroot=10.0.0.1:/media/cdrom ip=dhcp

