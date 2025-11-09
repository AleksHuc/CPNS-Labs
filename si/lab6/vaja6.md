# 6. Vaja: Upravljanje omrežja

## Navodila

0. Uporabite omrežje in navidezne računalnike iz prejšnjih vaj.
1. Postavit Simple Network Management Protocol (SNMP) strežnik.
2. Pridobite podatke s SNMP strežnika preko SNMPv1 protokola.
3. Prejete podatke naredimo berljive.
4. Pridobite podatke s SNMP strežnika preko SNMPv3 protokola.
5. Namestite spletno aplikacijo za zbiranje SNMP podatkov.

## Dodatne informacije

[Simple Network Management Protocol (SNMP)](https://en.wikipedia.org/wiki/Simple_Network_Management_Protocol) je standardni protokol z izmenjevanje podatkov preko omrežja o stanju računalnikov in omrežja.

SNMP uporablja za prenos podatkov format [Type Length Value (TLV)](https://en.wikipedia.org/wiki/Type-length-value) oziroma bolj specifično [ASN.1 BER kodiranje](https://en.wikipedia.org/wiki/X.690#BER_encoding).

Za upravljanje s SNMP podatki uporabljamo podatkovno zbirko [Management Information Base (MIB)](https://en.wikipedia.org/wiki/Management_information_base).

Pogosto uporabljane spletne aplikacije za zbiranje SNMP podatkov:

- [ICINGA](https://icinga.com/)
- [Nagios](https://www.nagios.org/)
- [Zabbix](https://www.zabbix.com/)
- [Cacti](https://www.cacti.net)

## Podrobna navodila

### 1. Naloga

Namestimo SNMP strežnik na prvi navidezni računalnik, na primer `snmpd`, ter SNMP odjemalca za preizkus delovanje, na primer `snmp`.

    apt install snmpd snmp

### 2. Naloga

Do SNMP podatkov dostopamo z ukazom `snmpwalk`, kadar želimo izpis vseh dostopnih podatkov in z ukazom `snmpget`, kadar želimo dostopati do posameznega podatka. Na primer, uporabimo `snmpwalk` in navedemo da uporabljamo verzijo protokola `SNMPv1`, dostopamo do skupnosti `public` in dostopamo do SNMP strežnika na IP naslovu `localhost`.

    snmpwalk -v 1 -c public localhost

Skupnosti nam pri verziji protokola SNMPv1 omogočajo določanje podskupin podatkov do katerih lahko dostopamo preko njih in jih urejamo v nastavitveni datoteki `/etc/snmp/snmpd.conf`. Ustvarimo novo skupnost, ki nam omogoča dostop do vseh podatkov, ki jih hrani SNMP strežnik.

    nano /etc/snmp/snmpd.conf

    rocommunity community

Da se nastavitve upoštevajo, ponovno zaženemo `snmpd` SNMP strežnik in preskusimo delovanje z ukazom `snmpwalk`.

    systemctl restart snmpd.service

    snmpwalk -v 1 -c community localhost

V nastavitveni datoteki `/etc/snmp/snmpd.conf` tudi omejimo dostop do SNMP strežnika na podlagi IP naslovov odjemalcev. V našem primeru bomo omogočilo dostop z vseh IP naslovov. Da se nastavitve upoštevajo, ponovno zaženemo `snmpd` SNMP strežnik.

    nano /etc/snmp/snmpd.conf

    agentaddress 0.0.0.0,[::1]

	systemctl restart snmpd.service

Preskusimo delovanje z ukazom `snmpwalk` z drugega navideznega računalnika.

    apt install snmp

    snmpwalk -v 1 -c community 10.0.0.1

### 3. Naloga

Podatki se izpišejo v formatu TLV in vidimo, da ima vsak podatek svoj hierarhično urejen identifikator Object Identifier (OID). Izpis podatkov lahko naredimo bolj berljiv, če uporabimo podatkovno zbirko MIB, ki preslika OID-je v opisne identifikatorje. Za namestitev zbirke MIB potrebujemo omogočiti dodatne ne odprtokodne repozitorije `non-free` in `contrib` v upravljalcu paketov na obeh navideznih računalnikih, da lahko potem namestimo zbirke na obeh.

    nano /etc/apt/sources.list

    deb http://deb.debian.org/debian/ trixie main non-free-firmware non-free contrib
	deb-src http://deb.debian.org/debian/ trixie main non-free-firmware non-free contrib

    apt update
    apt install snmp-mibs-downloader

Preskusimo delovanje zbirke MIB z ukazom `snmpwalk`.

    snmpwalk -m all -v 1 -c community localhost

### 4. Naloga

Glavna pomanjkljivost protokola verzije `SNMPv1` je, da ne podpira overjanja (angl. authentication) uporabnikov in ne šifrira (angl. encryption) poslanih podatkov. Verzija `SNMPv3` odpravi ti dve pomanjkljivosti in vsebuje tudi kopico drugih izboljšav. Za dostop do podatkov sedaj potrebujemo ustvariti uporabnika in zato uporabite že v naprej pripravljen program. Ustvarimo dva uporabnika ter jima na primer omogočimo samo branje podatkov, način overjanja in geslo za overjanje ter način šifriranja in ključ za šifriranje.

    apt install libsnmp-dev

    systemctl stop snmpd.service

    net-snmp-create-v3-user -ro -a SHA -A kpovkaboom -x AES -X kpovkaboom testuser

	adding the following line to /var/lib/snmp/snmpd.conf:
	   createUser testuser SHA "kpovkaboom" AES "kpovkaboom"
	adding the following line to /etc/snmp/snmpd.conf:
	   rouser testuser

	net-snmp-create-v3-user -ro -a SHA -A kpovkaboom -x AES -X kpovkaboom kpovuser

	adding the following line to /var/lib/snmp/snmpd.conf:
	   createUser kpovuser SHA "kpovkaboom" AES "kpovkaboom"
	adding the following line to /etc/snmp/snmpd.conf:
	   rouser kpovuser

Preverimo v nastavitvenih datotekah `/var/lib/snmp/snmpd.conf` in `/usr/share/snmp/snmpd.conf`, če se je uporabnik pravilno ustvaril in nato ponovno zaženemo SNMP strežnik ter preskusimo delovanje z ukazom `snmpwalk`.

    nano /var/lib/snmp/snmpd.conf

    createUser testuser SHA "kpovkaboom" AES "kpovkaboom"
	createUser kpovuser SHA "kpovkaboom" AES "kpovkaboom"

    nano /etc/snmp/snmpd.conf

    rouser testuser
	rouser kpovuser

    systemctl restart snmpd.service

    nano /var/lib/snmp/snmpd.conf

    usmUser 1 3 0x80001f8880eae8572fe92f376700000000 "kpovuser" "kpovuser" NULL .1.3.6.1.6.3.10.1.1.3 0x0ba65e9f4aae705647dafac724a659e5ed59e5ed594b9f .1.3.6.1.6.3.10.1.2.4 0x0ba65e9f4aae705647dafac724a659e5 ""
	usmUser 1 3 0x80001f8880eae8572fe92f376700000000 "testuser" "testuser" NULL .1.3.6.1.6.3.10.1.1.3 0x0ba65e9f4aae705647dafac724a659e5ed59e5ed594b9f .1.3.6.1.6.3.10.1.2.4 0x0ba65e9f4aae705647dafac724a659e5 ""

    snmpwalk -v 3 -m all -a SHA -A kpovkaboom -x AES -X kpovkaboom -l authPriv -u testuser 10.0.0.1

Če pogledamo tajnopisa gesel obeh uporabnikov, vidimo, da sta enaka. Tajnopisa gesel sta ustvarjena brez uporabe soli in zato sta podvržena [napadu z mavričnimi tabelami](https://en.wikipedia.org/wiki/Rainbow_table).

### 5. Naloga

Namestimo spletno aplikacijo za pridobivanje podatkov preko SNMP protokola, na primer `cacti`.

    apt install cacti

Med namestitvijo izberemo da bo spletna aplikacija `cacti` uporabljali spletni strežnik `apache2` in pritisnemo na gumb `OK`.

![Izbira spletnega strežnika za spletno aplikacijo `cacti`.](slike/vaja6-cacti1.png)

Za potrebe hranjenja podatkov uporabimo podatkovno bazo za katero izberemo, da se avtomatsko nastavi na privzete nastavitve s pritiskom na gumb `Yes`.

![Izbira avtomatske nastavitve podatkovne baze na privzete nastavitve.](slike/vaja6-cacti2.png)

Nato še vnesemo geslo za administratorski račun ter pritisnemo na gumb `OK` in ga potem s ponovnim vnosom in pritiskom na gumb `OK` še potrdimo.

![Izbira administratorskega gesla.](slike/vaja6-cacti3.png)

![Potrditev administratorskega gesla.](slike/vaja6-cacti4.png)

Do spletne aplikacije dostopamo preko brskalnika.

    http://10.0.0.1/cacti

V spletno aplikacijo se vpišemo z uporabniškim imenom `admin` in geslom, ki smo ga izbrali med namestitvijo.

Za prejemanje SNMP podatkov moramo dodati naš strežnik, tako da pod menijem na levi izberemo `Create\New Device` in izpolnimo obrazec za dodajanje nove naprave. Na primer, za opis `Description` izberemo `SNMP Server`, za IP naslov `Hostname` izberemo IP naslov našega SNMP strežnika `10.0.0.1`, za verzijo protokola SNMP `SNMP Version` izberemo verzijo `Version 3`, za nivo varnosti `SNMP Security Level`izberemo `authPriv`, za uporabnika `SNMP Username (v3)` izberemo uporabnika, ki smo go prej ustvarili `testuser`, Za protokol za overjanje `SNMP Auth Protocol` izberemo `SHA` ter vnesemo geslo za oviranje pod `SNMP Password (v3)`, za protokol za šifriranje `SNMP Privacy Protocol (v3)` izberemo `AES-128` ter vnesem ključ za šifriranje pod `SNMP Privacy Passphrase` in vse ostale nastavitve pustimo na privzetih vrednostih. Obrazec potrdimo s pritiskom na gumb ustvari `Create` spodaj desno.

![Nastavitev SNMP strežnika za dostop do podatkov.](slike/vaja6-cacti5.png)

Do podatkov sedaj dostopamo preko zavihkov zgoraj levo, na primer do grafov `Graphs`.

![Prikaz SNMP podatkov z grafi.](slike/vaja6-cacti6.png)

Če imamo težave z zagonom strežnika `apache2`, preverimo ali je port `80` že zaseden z ukazom `netstat` in ustavimo proces, ki ga uporablja.

    apt install net-tools

    netstat -putln

    Active Internet connections (only servers)
	Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
	tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      835/cupsd           
	tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      891/lighttpd        
	tcp6       0      0 ::1:631                 :::*                    LISTEN      835/cupsd           
	tcp6       0      0 :::80                   :::*                    LISTEN      891/lighttpd        
	udp        0      0 0.0.0.0:5353            0.0.0.0:*                           576/avahi-daemon: r 
	udp        0      0 0.0.0.0:67              0.0.0.0:*                           873/dhcpd           
	udp        0      0 0.0.0.0:68              0.0.0.0:*                           566/dhclient        
	udp        0      0 0.0.0.0:69              0.0.0.0:*                           869/in.tftpd        
	udp        0      0 0.0.0.0:161             0.0.0.0:*                           839/snmpd           
	udp        0      0 0.0.0.0:60251           0.0.0.0:*                           576/avahi-daemon: r 
	udp6       0      0 :::5353                 :::*                                576/avahi-daemon: r 
	udp6       0      0 :::69                   :::*                                869/in.tftpd        
	udp6       0      0 ::1:161                 :::*                                839/snmpd           
	udp6       0      0 :::54171                :::*                                576/avahi-daemon: r 
