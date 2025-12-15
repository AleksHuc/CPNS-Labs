# 11. Vaja: Upravljanje omrežja s požarno pregrado

## Navodila

0. Uporabite omrežje in navidezne računalnike iz prejšnjih vaj. 
1. Omogočite dostop drugega navideznega računalnika do spleta preko prvega navideznega računalnika.
2. Na prvem navideznem računalniku preprečite drugemu navideznemu računalniku dostop do IP naslova `8.8.8.8`.
3. Na prvem navideznem računalniku preprečite dostop drugega navideznega računalnika do prvega navideznega računalnika preko varne lupine (SSH).
4. Na prvem navideznem računalniku preprečite prenos paketkov preko protokola HTTP po tem, ko smo že prenesli 1 MB podatkov.

## Dodatne informacije

[Netfilter](https://en.wikipedia.org/wiki/Netfilter) je ogrodje, ki je del Linux jedra in nam omogoča upravljanje z omrežjem, na primer: filtriranje paketkov, preslikovanje IP naslovov in vrat za namen usmerjanja ter preprečevanja dostopa do delov omrežja.

[iptables](https://en.wikipedia.org/wiki/Iptables) predstavljajo zastarelo orodje za filtriranje paketov na osnovi Netfilter sistema kavljev (angle. hook).

[nftables](https://en.wikipedia.org/wiki/Nftables) novo orodje, ki nadomesti `iptables` ter omogoča dodatne funkcionalnosti, kot je na primer preiskovanje paketov.

## Podrobna navodila

### 1. Naloga

Če smo do sedaj uporabljali `iptables`, potem izbrišemo vsa pravila.

    iptables -t nat -L
    iptables -t nat -F
	iptables-save > /etc/iptables/rules.v4

Trenutne nastavitve `nftables` lahko preverimo z ukazom `nft`. [Navodila](https://www.netfilter.org/projects/nftables/manpage.html) za uporabo `nft`.

    nft list ruleset

Orodje `nftables` uporablja tabele (`tables`) in verige (`chains`) za hierarhično hranjenje pravil (`rules`). Vse tabele, verige in pravila imajo svoje identifikatorje (`handles`), ki jih uporabljamo za brisanje. Na primer:

    nft -a list table mytable
    nft delete rule mytable mychain handle 10

Na prvem navideznem računalniku omogočimo delovanje prehoda, tako da ustvarimo novo tabelo `table`, novo verigo `chain` in novo pravilo `rule` za preslikovanje IP naslovov paketov.

    nft add table mytable
    nft 'add chain mytable postroutingchain { type nat hook postrouting priority -100; }'
    nft add rule mytable postroutingchain masquerade

Prav tako moramo omogočiti usmerjanje na prvem navideznem računalnik v datoteki `/etc/sysctl.d/sysctl.conf`.

    nano /etc/sysctl.d/sysctl.conf

    net.ipv4.ip_forward=1

Da se sprememba parametrov jedra Linux-a upošteva, uporabimo ukaz `sysctl`.

    sysctl -p /etc/sysctl.d/sysctl.conf

### 2. Naloga

Na prvem navideznem računalniku dodamo brez stansko pravilo za filtriranje vseh paketov s ponornim IP naslovom `8.8.8.8`, ki izvirajo iz drugega navideznega računalnika.

    nft 'add chain mytable forwardchain { type filter hook forward priority -100; }'
    nft add rule mytable forwardchain ip daddr . ip saddr { 8.8.8.8 . SECOND_VM_IP } drop

### 3. Naloga

Na prvem navideznem računalniku dodamo brez stansko pravilo za filtriranje vseh paketov s protokolom [`SSH`](https://en.wikipedia.org/wiki/Secure_Shell), ki izvirajo iz drugega navideznega računalnika.

    nft 'add chain mytable inputchain { type filter hook input priority -100; }'
    nft add rule mytable inputchain tcp dport . ip saddr { 22 . SECOND_VM_IP } drop

### 4. Naloga

Na prvem navideznem računalniku dodamo stanjsko pravilo za filtriranje vseh paketov s protokolom [`HTTP`](https://en.wikipedia.org/wiki/HTTP), ko presežemo 1MB prejetih paketov. Če nimamo še nameščenege `HTTP` strežnika ga namestimo, ustvarimo poljubno datoteko z velikostjo manjšo od 1 MB, na primer z ukazom [`dd`](https://man7.org/linux/man-pages/man1/dd.1.html), in jo prenesemo v mapo `\var\www\html`.

	apt update
	apt install lighttpd

	dd if=/dev/zero of=output.dat  bs=400K  count=1
	1+0 records in
	1+0 records out
	409600 bytes (410 kB, 400 KiB) copied, 0.00104389 s, 392 MB/s

	mv output.dat /var/www/html/

Sedaj dodamo pravila za hranjenje prenesene količine podatkov in vklopimo izpuščanje paketkov, ko presežemo izbrano mejo. To lahko dosežemo z uporabo števca, ki šteje število prenešenih paketov in bajtov ter kvote v bajtih (`bytes`, `kbytes`, `mbytes`), ki predstavlja mejo za sprožitev pravila (`over` in `until`).	
	
	nft add table filter
	nft add counter filter http-traffic
	nft add quota filter http-quota over 1 mbytes
	nft add chain filter output { type filter hook output priority 0 \; }
	nft add rule filter output tcp sport http counter name http-traffic
	nft add rule filter output tcp sport http quota name http-quota drop

Preverimo stanje trenutnih pravil.

	nft list table filter
	
	table ip filter {
		counter http-traffic {
			packets 0 bytes 0
		}

		quota http-quota {
			over 1 mbytes
		}

		chain output {
			type filter hook output priority filter; policy accept;
			tcp sport 80 counter name "http-traffic"
			tcp sport 80 quota name "http-quota" drop
		}
	}

Na drugem računalniku poženemo zahtevo za prenos datoteke in ponovno preverimo stanje pravil na prvem računalniku.

	wget 10.0.0.1/output.dat

	--2024-12-13 16:37:00--  http://10.0.0.1/output.dat
	Connecting to 10.0.0.1:80... connected.
	HTTP request sent, awaiting response... 200 OK
	Length: 409600 (400K)
	Saving to: ‘output.dat’

	output.dat          100%[===================>] 400.00K  --.-KB/s    in 0.001s  

	2024-12-13 16:37:00 (306 MB/s) - ‘output.dat’ saved [409600/409600]

	nft list table filter

	table ip filter {
		counter http-traffic {
			packets 26 bytes 411225
		}

		quota http-quota {
			over 1 mbytes used 411225 bytes
		}

		chain output {
			type filter hook output priority filter; policy accept;
			tcp sport 80 counter name "http-traffic"
			tcp sport 80 quota name "http-quota" drop
		}
	}

Ponovno na drugem računalniku poženemo zahtevo za prenos datoteke in ponovno preverimo stanje pravil na prvem računalniku.

	wget 10.0.0.1/output.dat
	--2024-12-13 16:38:24--  http://10.0.0.1/output.dat
	Connecting to 10.0.0.1:80... connected.
	HTTP request sent, awaiting response... 200 OK
	Length: 409600 (400K)
	Saving to: ‘output.dat.1’

	output.dat.1        100%[===================>] 400.00K  --.-KB/s    in 0.002s  

	2024-12-13 16:38:24 (220 MB/s) - ‘output.dat.1’ saved [409600/409600]

	nft list table filter

	table ip filter {
		counter http-traffic {
			packets 54 bytes 822554
		}
	
		quota http-quota {
			over 1 mbytes used 822554 bytes
		}

		chain output {
			type filter hook output priority filter; policy accept;
			tcp sport 80 counter name "http-traffic"
			tcp sport 80 quota name "http-quota" drop
		}
	}

Še enkrat na drugem računalniku poženemo zahtevo za prenos datoteke in ponovno preverimo stanje pravil na prvem računalniku.

	wget 10.0.0.1/output.dat

	--2024-12-13 16:39:28--  http://10.0.0.1/output.dat
	Connecting to 10.0.0.1:80... connected.
	HTTP request sent, awaiting response... 200 OK
	Length: 409600 (400K)
	Saving to: ‘output.dat.2’

	output.dat.2         50%[=========>          ] 200.54K  --.-KB/s    eta 6m 24s

	nft list table filter

	table ip filter {
		counter http-traffic {
			packets 85 bytes 1202102
		}

		quota http-quota {
			over 1 mbytes used 1 mbytes
		}

		chain output {
			type filter hook output priority filter; policy accept;
			tcp sport 80 counter name "http-traffic"
			tcp sport 80 quota name "http-quota" drop
		}
	}

Števec in kvoto lahko poenostavimo z naslednjima ukazoma in sedaj prenos preko HTTP ponovno deluje.
	
	nft reset counters
	nft reset quotas

	
	
	


