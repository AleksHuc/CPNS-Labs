# 3. Vaja: Delovanje preprostih programov preko omrežja

## Navodila

0. Uporabite omrežje in navidezne računalnike iz 2. vaje. 
1. Namestite Trivial File Transfer Protocol (TFTP) strežnik in ugotovite kako deluje (`tftpd`)?
2. Ugotovite kaj je `inetd` in kako deluje?
3. Implementirajte preprost Hypertext Transfer Protocol (HTTP) strežnik, ki deluje preko `inetd`.

## Dodatne informacije

Ukaz [`ls`](https://linux.die.net/man/1/ls) nam omogoča izpis vsebine map datotečnega sistema.

Ukaz [`mkdir`](https://linux.die.net/man/1/mkdir) se uporablja za ustvarjanje novih map datotečnega sistema.

Ukaz [`touch`](https://linux.die.net/man/1/touch) nam omogoča ustvarjanje datotek in spreminjanje časovnih značk datotek.

Ukaz [`echo`](https://linux.die.net/man/1/echo) se uporablja za izpis vrstic teksta na standardni izhod.

Ukaz [`awk`](https://linux.die.net/man/1/awk) nam omogoča iskanje in obdelavo tekstovnih vzorcev iz nizov.

Ukaz [`wc`](https://linux.die.net/man/1/wc) nam omogoča štetje novih vrstic, besed in bajte v posameznih tekstovnih datotekah.

Ukaz [`chmod`](https://linux.die.net/man/1/chmod) se uporablja za spreminjanje načina dovoljenega dostopa do datotek uporabnikom, skupinam ter vsem ostalim.

Ukaz [`curl`](https://linux.die.net/man/1/curl) nam omogoča prenos podatkov s strežnika na lokalni računalnik preko kopice podprtih protokolov.

## Podrobna navodila

### 1. Naloga

Namestimo TFTP strežnik preko upravljavca paketov nameščene Linux distribucije na prvi navidezni računalnik z DHCP strežnikom. [TFTP](https://en.wikipedia.org/wiki/Trivial_File_Transfer_Protocol) je preprost protokol za prenos datotek preko omrežja na oddaljene računalnike, katerega specifikacijo lahko najdemo v [RFC 1350](https://www.rfc-editor.org/rfc/rfc1350). Na primer, namestimo `tftpd` implementacijo protokola TFTP (obstajajo še `tftpd-hpa`, `atftpd` ...).

    apt install tftpd

Nameščeni `tftpd` strežnik že deluje s privzetimi nastavitvami, ki so podane v nastavitveni datoteki `/etc/inetd.conf`. Na primer, z naslednjo vrstico:

    tftp dgram udp wait nobody /usr/sbin/tcpd /usr/sbin/in.tftpd /srv/tftp

Vidimo, da `tftpd` potrebuje `inetd`, da deluje. Trenutno je za nas pomemben zadnji parameter, ki kaže na mapo `/srv/tftp`, za katero preverimo ali obstaja, ter vidimo, da ne.

    ls /srv/tftp
    
    ls: cannot access '/srv/tftp': No such file or directory

V primeru, da z upravljalcem paketov ne najdete paketa `tftpd` potem namestite paket `tftpd-hpa` in sledite navodilom od tukaj naprej.

Omenjena mapa služi za shranjevanje datotek, ki jih bo strežnik TFTP lahko prenašal preko omrežja, zato jo ustvarimo in vanj shranimo poljubno datoteko.

    mkdir /srv/tftp
    touch test.txt

Sedaj preskusimo delovanje TFTP strežnika, tako da na drugem navideznem stroju namestimo TFTP klienta in prenesemo testno datoteko.

    apt install tftp

    tftp
    connect IP_DHCP_SERVER
    get FILE_NAME
    quit

### 2. Naloga

Pri namestitvi `tftpd` smo ugotovili, da se je namestil tudi paket [`openbsd-inetd`](https://man.openbsd.org/inetd), ki je eden izmed mnogih implementacij `inetd` (obstajajo še `xinetd`, `rinetd`, `rlinetd`, `inetutils-inetd` ...).  Prav tako smo ugotovili, da `tftpd` deluje preko `inetd` in ga tudi nastavljamo preko `/etc/inetd.conf` nastavitvene datoteke.

Torej `inetd` je program, ki se izvaja v ozadju operacijskega sistema (angl. service, deamon), ki posluša na nastavljenih omrežnih vratih in omogoča povezovanje na njih. Ko omrežna naprava vzpostavi povezavo na nastavljena vrata, `inetd` zažene program, ki je določen za ta vrata ter posreduje vse prejete podatke na standardni vhod programa. Ko pognani program zaključi izvajanje, `inetd` posreduje podatke s standardnega izhoda programa preko omrežnih vrat nazaj izvirni omrežni napravi, ki je vzpostavila povezavo. `inetd` nam omogoča delovanje programov preko omrežja, brez da zato poskrbi program sam.

Posamezne programe, ki jih `inetd` poganja so nastavljeni v datoteki `/etc/inetd.conf` določimo z naslednjimi parametri:

| Ime storitve ali vrata | Tip vtičnika | Protokol | Čakaj na končanje | Uporabnik | Program        | Argumenti                    |
|------------------------|--------------|----------|-------------------|-----------|----------------|------------------------------|
| tftp                   | dgram        | udp      | wait              | nobody    | /usr/sbin/tcpd | /usr/sbin/in.tftpd /srv/tftp |

### 3. Naloga

Napišemo preprost HTTP strežnik, ki na zahtevo `GET` preprosto vrne zahtevano HyperText Markup Language ([HTML](https://en.wikipedia.org/wiki/HTML)) datoteko.

    nano server.sh

    #!/bin/bash

    read request

    file=$( echo $request | awk '$1 == "GET" { print $2 }' )

    echo "HTTP/1.1 200 OK"
    echo "Content-Length: $(wc -c < $1$file)"
    echo ""
    cat $1$file

Preverimo pravice za izvajanje našega HTTP strežnika in jih omogočimo.

    ls -ls

    4 -rw-r--r-- 1 aleks aleks  185 Oct 24 11:55 server.sh

    chmod u+x server.sh

Ustvarimo še HTML datoteko, ki jo bo naš HTTP strežnik vračal.

    nano index.html

    <!DOCTYPE html>
    <html>
        <head>
            <title>KPOV</title>
        </head>
        <body>
            <h1>3. Labs</h1>
            <p>It works!</p>
        </body>
    </html>

Preskusimo delovanje HTTP strežnika lokalno.

    ./server.sh /home/aleks

    GET /index.html HTTP/1.1

    HTTP/1.1 200 OK
    Content-Length: 142

    <!DOCTYPE html>
    <html>
        <head>
            <title>KPOV</title>
        </head>
        <body>
            <h1>3. Labs</h1>
            <p>It works!</p>
        </body>
    </html>

Če slučajno še nimamo nameščenega `inetd` programa, ga namestimo z upravljalcem paketov.

    apt install openbsd-inetd

Sedaj dodamo naš HTTP strežnik v nastavitveno datoteko `/etc/inetd.conf` ter nato ponovno zaženemo storitev `inetd`. Za ime storitve izberemo `http` oz. lahko tudi vrata `80`. Za tip vtičnika izberemo `stream`, ker bomo uporabljali Transmission Control Protocol ([TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)), kar pomeni, da za protokol izberemo `tcp`. Strežnik nastavimo, da je vedno na voljo in ne čaka na zaključek programa, ki ga poganja, z nastavitvijo `nowait`. Nato izberemo uporabnika, ki ima pravico za izvajanje našega enostavnega HTTP strežnika, na primer `aleks`. Sledi absolutna pot do našega programa, ki je `/home/aleks/server.sh` in argumenti, ki se podajo pri zagonu programa. Prvi argument je po konvenciji kar ime programa `server.sh`. Drugi argument pa kaže na mapo v kateri se nahaja naš `index.html`, ki je `/home/student`.

    nano /etc/inetd.conf

    http stream tcp	nowait aleks /home/aleks/server.sh server.sh /home/aleks

    systemctl restart inetd.service

Preko omrežja preskusimo delovanje s programom za pošiljanje HTTP zahtevkov iz ukazne konzole `curl` na drugem navideznem računalniku.

    apt install curl

    curl 10.0.1.1/index.html --verbose