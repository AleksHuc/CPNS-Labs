# 10. Vaja: Postavitev navideznega zasebnega omrežja z uporabo certifikatov

## Navodila
 
0. Uporabite omrežje in navidezne računalnike iz prejšnjih vaj. 
1. Dodajte še tretji navidezni računalnik v omrežje, tako da klonirate klienta.
2. Nastavite in preizkusite delovanje OpenVPN med OpenVPN strežnikom, OpenVPN klientom in OpenVPN certifikatne agencije preko načina TUN. 

## Dodatne informacije

[Asimetrična kriptografija](https://en.wikipedia.org/wiki/Cryptography#Public-key_cryptography) je pristop, pri katerem za enkripcijo uporabljamo javni ključ in za dekripcijo uporabljamo privatni ključe.

[Digitalni certifikat](https://en.wikipedia.org/wiki/Public_key_certificate) je digitalni dokument, ki vsebuje privatni in javni ključ in zagotavlja potrjevanje lastnika certifikata oziroma njegovega javnega ključa.

[Certifikatna agencija](https://en.wikipedia.org/wiki/Certificate_authority) je entiteta, ki hrani, podpisuje in izdaja digitalne certifikate.

[Easy-RSA 3](https://easy-rsa.readthedocs.io/en/latest/) je orodje za upravljanje s strukturo javnih ključev (X.509 PKI - Public Key Infrastructure).

[Public Key Infrastructure (PKI)](https://en.wikipedia.org/wiki/Public_key_infrastructure) skupek orodij za upravljanje s certifikati, pari javnih in privatnih ključev in asimetrično kriptografijo.

[Diffie-Hellman key exchange (DH)](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange) je matematična metoda za varno izmenjavo kriptografskih ključev preko javnega kanala.

## Podrobna navodila

### 1. Naloga

Kloniranje izvedemo tako, da z desnim klikom na navidezni računalnik v levem stolpcu v glavnem oknu VirtualBox-a opremo dodatni meni in kliknemo na možnost `Kloniraj ...`. Odpre se nam okno za izbiro nastavitev kloniranja. Določimo lahko `Ime`, `Mapo računanika`, `Upravljanje z MAC naslovi` in `Dodatne možnosti`. Na primer, za ime izberemo `Debian Clone`, za mapo v katerem bo shranjen navidezni računalnik izberemo `/home/USER/VirtualBox VMs`, izberemo, da ustvarimo nove MAC naslove za vse omrežne kartice ne glede na nastavitve z izbiro `Vključimo MAC naslove vseh omrežnih kartic` (angl. Include all network adapter MAC addresses) in pustimo dodatne možnosti odkljukane. Pritisnemo gumb `Naprej`.

![Izbira nastavitev kloniranja.](slike/vaja10-clone1.png)

V naslednjem koraku izbiramo za način kloniranja `Poln klon` in pritisnemo gumb `Kloniraj`, da pričnemo s kloniranjem.

![Izbira načina kloniranja.](slike/vaja10-clone2.png)

### 2. Naloga

Prvi navidezni računalnik bo opravljal vlogo OpenVPN strežnika, drugi navidezni računalnik bo opravljal vlogo certifikatne agencija in tretji navidezni računalnik bo opravljal vlogo OpenVPN klienta.

Na strežniku najprej namestimo OpenVPN paket `openvpn` ter OpenSSL strežnik `openssh-server` za varno komunikacijo preko upravljalca paketkov našega operacijskega sistema.

    apt install openvpn openssh-server

Se premaknemo v domačo mapo `/home/USERNAME` in ustvarimo novo mapo v kateri bomo ustvarili vse potrebne datoteke za uporabo OpenVPN.

    cd /home/aleks
    mkdir vpn_cert
    cd /vpn_cert

Sedaj ustvarimo mapo s posebno strukturo, ki nam bo služila za upravljanje s certifikati in se premaknemo vanjo. Poženemo inicializacijo datotek, ki jih potrebujemo. Ustvarimo zasebni ključ (`pki/private/server.key`) in zahtevek za podpis certifikata (`pki/reqs/server.req`), ki vsebuje javni ključ. Ustvarimo tudi parametre za vzpostavitev povezave s protokolom TLS, ki vključuje podatke za postopek Diffie-Hellman izmenjavo ključev (`pki/dh.pem`).

    make-cadir server
    cd server
    ./easyrsa init-pki
    ./easyrsa gen-req server
    ./easyrsa gen-dh

Zahtevek za podpis varno prenesemo s strežnika na drugi navidezni računalnik z certifikatno agencijo z ukazom `scp`.

    scp pki/reqs/server.req aleks@CERTIFICATE_AUTHORITY_IP:

Na klientu prav tako najprej namestimo OpenVPN paket `openvpn` ter OpenSSL strežnik `openssh-server` za varno komunikacijo preko upravljalca paketkov našega operacijskega sistema.

    apt install openvpn openssh-server

Se premaknemo v domačo mapo `/home/USERNAME` in ustvarimo novo mapo v kateri bomo ustvarili vse potrebne datoteke za uporabo OpenVPN.

    cd /home/aleks
    mkdir vpn_cert
    cd /vpn_cert

Sedaj ustvarimo mapo s posebno strukturo, ki nam bo služila za upravljanje s certifikati in se premaknemo vanjo. Poženemo inicializacijo datotek, ki jih potrebujemo. Ustvarimo zasebni ključ (`pki/private/client.key`) in zahtevek za podpis certifikata (`pki/reqs/client.req`), ki vsebuje javni ključ.

    make-cadir client
    cd client
    ./easyrsa init-pki
    ./easyrsa gen-req client

Zahtevek za podpis varno prenesemo s klienta na drugi navidezni računalnik z certifikatno agencijo z ukazom `scp`.

    scp pki/reqs/client.req aleks@CERTIFICATE_AUTHORITY_IP:

Na računalniku s certifikatno agencijo tudi najprej namestimo OpenVPN paket `openvpn`, program za upravljanje s certifikati `easy-rsa` ter OpenSSL strežnik `openssh-server` za varno komunikacijo preko upravljalca paketkov našega operacijskega sistema.

    apt install openvpn easy-rsa openssh-server

Se premaknemo v domačo mapo `/home/USERNAME` in ustvarimo novo mapo v kateri bomo ustvarili vse potrebne datoteke za uporabo OpenVPN.

    cd /home/aleks
    mkdir vpn_cert
    cd /vpn_cert

Sedaj ustvarimo mapo s posebno strukturo, ki nam bo služila za upravljanje s certifikati in se premaknemo vanjo. Poženemo inicializacijo datotek, ki jih potrebujemo. Zgradimo še vse potrebno za delovanje certifikatne agencije, vključno z zasebnim (`pki/private/ca.key `) in javnim (`pki/ca.crt`) ključem.

    make-cadir ca
    cd /ca
    ./easyrsa init-pki
    ./easyrsa build-ca

Zahtevka za podpis s strani strežnika in klienta se nahajata v domači mapi uporabnika, zato ju premaknemo v mapo za zahtevke.

    mv /home/aleks/server.req pki/reqs/
    mv /home/aleks/client.req pki/reqs/

Certifikatna agencija s svojim privatnim ključem podpiše zahtevek za podpis s strežnika v načinu za strežnike (`server`) in tako ustvari certifikat strežnika (`server.crt`) ter ga varno pošlje strežniku nazaj.

    ./easyrsa sign-req server server
    scp pki/issued/server.crt aleks@SERVER_IP:

Certifikatna agencija s svojim privatnim ključem podpiše še zahtevek za podpis s klienta v načinu za kliente (`client`) in tako ustvari certifikat klienta (`client.crt`) ter ga varno pošlje klientu nazaj.

    ./easyrsa sign-req client client
    scp pki/issued/client.crt aleks@CLIENT_IP:

Za potrditev podpisanih certifikatov s strani certifikatne agencije potrebujeta strežnik in klient še javni ključ certifikatne agencije, ki ga prav tako varno posreduje certifikatna agencija. 

    scp pki/ca.crt aleks@SERVER_IP:
    scp pki/ca.crt aleks@CLIENT_IP:

Certifikatna agencija si sedaj pripravi še privatni in javni ključ oz. certifikat za dostop v omrežje VPN kot klient in si ga tudi sama podpiše.

    ./easyrsa build-client-full caclient

Sedaj nadaljujemo na strežniku kjer ustvarimo mapo za podpisane certifikate in vanj iz domače mape premaknemo podpisan certifikat. Prav tako premaknemo javni ključ certifikatne agencije v našo mapo s ključi.

    mkdir pki/issued
    mv /home/aleks/server.crt pki/issued/
    mv /home/aleks/ca.crt pki/

Premaknemo se v nadrejeno mapo ter ustvarimo nastavitveno datoteko za OpenVPN strežnik, ki bo deloval preko protokola TCP in ustvaril tunel na 3. plasti omrežja, torej način `tun`. V nastavitveni datoteki nastavimo protokol preko katerega bo deloval VPN `proto tcp`, plast na kateri se bo izvedel tunel `dev tun`, VPN IP naslove bo strežnik dodeljeval iz lokalnega omrežja `server 10.8.0.0 255.255.255.0` in deloval kot podomrežje `topology subnet` ter omogočimo vidljivost med klienti v VPN omrežju `client-to-client`. Dodamo še javni ključ certifikatne agencije `ca.crt`, zasebni `server.key` in javni `server.crt` ključ strežnika ter parametre za izmenjavo ključa s postopkom Diffie-Hellman `dh.pem` .

    cd ..
    nano server.conf

    proto tcp
    server 10.8.0.0 255.255.255.0
    dev tun
    topology subnet
    client-to-client

    ca server/pki/ca.crt
    cert server/pki/issued/server.crt
    key server/pki/private/server.key
    dh server/pki/dh.pem

Sedaj še poženemo strežnik OpenVPN in preizkusimo delovanje.

    openvpn server.conf

Nadaljujemo na klientu kjer ustvarimo mapo za podpisane certifikate in vanj iz domače mape premaknemo podpisan certifikat. Prav tako premaknemo javni ključ certifikatne agencije v našo mapo s ključi.

    mkdir pki/issued
    mv /home/aleks/client.crt pki/issued/
    mv /home/aleks/ca.crt pki/

Premaknemo se v nadrejeno mapo ter ustvarimo nastavitveno datoteko za OpenVPN klienta, ki bo deloval preko protokola TCP in ustvaril tunel na 3. plasti omrežja, torej način `tun`. V nastavitveni datoteki nastavimo protokol preko katerega bo deloval VPN `proto tcp`, plast na kateri se bo izvedel tunel `dev tun`, nastavimo da deluje kot klient `client` in nastavimo IP naslov OpenVPN strežnika `remote SERVER_IP`. Dodamo še javni ključ certifikatne agencije `ca.crt`, zasebni `client.key` in javni `client.crt` ključ klienta.

    cd ..
    nano client.conf

    proto tcp
    client
    dev tun
    remote SERVER_IP

    ca client/pki/ca.crt
    cert client/pki/issued/client.crt
    key client/pki/private/client.key

Sedaj še poženemo klienta OpenVPN in preizkusimo delovanje.

    openvpn client.conf

Nadaljujemo še na certifikatni agenciji, ki ima vse certifikate že v pravih mapah. Premaknemo se v nadrejeno mapo ter ustvarimo nastavitveno datoteko za OpenVPN klienta certifikatne agencije, ki bo deloval preko protokola TCP in ustvaril tunel na 3. plasti omrežja, torej način `tun`. V nastavitveni datoteki nastavimo protokol preko katerega bo deloval VPN `proto tcp`, plast na kateri se bo izvedel tunel `dev tun`, nastavimo da deluje kot klient `client` in nastavimo IP naslov OpenVPN strežnika `remote SERVER_IP`. Dodamo še javni ključ certifikatne agencije `ca.crt`, zasebni `caclient.key` in javni `caclient.crt` ključ klienta.

    cd ..
    nano caclient.conf

    proto tcp
    client
    dev tun
    remote SERVER_IP

    ca ca/pki/ca.crt
    cert ca/pki/issued/caclient.crt
    key ca/pki/private/caclient.key

Sedaj še poženemo klienta OpenVPN in preizkusimo delovanje.

    openvpn caclient.conf
