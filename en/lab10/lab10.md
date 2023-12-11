# 10. Lab: Set up a virtual private network using certificates

## Instructions
 
0. Use the network and virtual machines from the previous exercises.
1. Add a third virtual machine to the network by cloning the client.
2. Set up and test OpenVPN operation between OpenVPN server, OpenVPN client and OpenVPN certificate authority via TUN mode.

## Additional information

[Asymmetric cryptography](https://en.wikipedia.org/wiki/Cryptography#Public-key_cryptography) is an approach where we use a public key for encryption and private keys for decryption.

[Digital certificate](https://en.wikipedia.org/wiki/Public_key_certificate) is a digital document that contains a private and public key and provides confirmation of the owner of the certificate or its public key.

A [Certificate Authority](https://en.wikipedia.org/wiki/Certificate_authority) is an entity that maintains, signs and issues digital certificates.

[Easy-RSA 3](https://easy-rsa.readthedocs.io/en/latest/) is a tool for managing public key infrastructure (X.509 PKI - Public Key Infrastructure).

[Public Key Infrastructure (PKI)](https://en.wikipedia.org/wiki/Public_key_infrastructure) a set of tools for managing certificates, public and private key pairs, and asymmetric cryptography.

[Diffie-Hellman key exchange (DH)](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange) is a mathematical method for securely exchanging cryptographic keys over a public channel.

## Detailed instructions

### 1. Task

Cloning is carried out by right-clicking on the virtual computer in the left column of the main VirtualBox window to open an additional menu and clicking on the `Clone...` option. A window for selecting cloning settings opens. You can specify `Name`, `Computer folder`, `Manage MAC addresses` and `Additional options`. For example, we choose `Debian Clone` as the name, we choose `/home/USER/VirtualBox VMs` as the folder where the virtual machine will be stored, we choose to create new MAC addresses for all network cards regardless of the settings by choosing `Enable MAC addresses of all network cards` (Include all network adapter MAC addresses) and leave the additional options unchecked. Press the `Next` button.

![Selecting clone settings.](images/lab10-clone1.png)

In the next step, choose `Full clone` for the cloning method and press the `Clone` button to start cloning.

![Choosing the cloning method.](images/lab10-clone2.png)

### 2. Task

The first virtual machine will act as an OpenVPN server, the second virtual machine will act as a certificate authority, and the third virtual machine will act as an OpenVPN client.

On the server, we first install the OpenVPN package `openvpn`, the and the OpenSSL server `openssh-server` for secure communication via the package manager of our operating system.

    apt install openvpn openssh-server

We move to the home folder `/home/USERNAME` and create a new folder in which we will create all the necessary files for using OpenVPN.

    cd /home/aleks
    mkdir vpn_cert
    cd /vpn_cert

Now let's create a folder with a special structure that will serve us to manage certificates and move into it. We start the initialization of the files we need. Let's create a private key (`pki/private/server.key`) and a certificate signing request (`pki/reqs/server.req`) containing the public key. We also create parameters for establishing a connection with the TLS protocol, which includes data for the Diffie-Hellman key exchange process (`pki/dh.pem`).

    make-cadir server
    cd server
    ./easyrsa init-pki
    ./easyrsa gen-req server
    ./easyrsa gen-dh

The signature request is securely transferred from the server to another virtual computer with a certificate authority using the `scp` command.

    scp pki/reqs/server.req aleks@CERTIFICATE_AUTHORITY_IP:

On the client, we also first install the OpenVPN package `openvpn` and the OpenSSL server `openssh-server` for secure communication via the package manager of our operating system.

    apt install openvpn openssh-server

We move to the home folder `/home/USERNAME` and create a new folder in which we will create all the necessary files for using OpenVPN.

    cd /home/aleks
    mkdir vpn_cert
    cd /vpn_cert

Now let's create a folder with a special structure that will serve us to manage certificates and move into it. We start the initialization of the files we need. Let's create a private key (`pki/private/client.key`) and a certificate signing request (`pki/reqs/client.req`) containing the public key.

    make-cadir client
    cd client
    ./easyrsa init-pki
    ./easyrsa gen-req client

The signature request is securely transferred from the client to another virtual computer with a certificate agency using the `scp` command.

    scp pki/reqs/client.req aleks@CERTIFICATE_AUTHORITY_IP:

On the computer with the certificate agency, we also first install the OpenVPN package `openvpn` and the OpenSSL server `openssh-server` for secure communication via the package manager of our operating system.

    apt install openvpn openssh-server

We move to the home folder `/home/USERNAME` and create a new folder in which we will create all the necessary files for using OpenVPN.

    cd /home/aleks
    mkdir vpn_cert
    cd /vpn_cert

Now let's create a folder with a special structure that will serve us to manage certificates and move into it. We start the initialization of the files we need. Let's build everything else necessary for the operation of the certificate agency, including the private (`pki/private/ca.key`) and public (`pki/ca.crt`) keys.

    make-cadir ca
    cd /ca
    ./easyrsa init-pki
    ./easyrsa build-ca

The signature request from the server and the client are located in the user's home folder, so we move them to the requests folder.

    mv /home/aleks/server.req pki/reqs/
    mv /home/aleks/client.req pki/reqs/

The certificate authority signs the signature request from the server in server mode (`server`) with its private key, creating a server certificate (`server.crt`) and securely sending it back to the server.

    ./easyrsa sign-req server server
    scp pki/issued/server.crt aleks@SERVER_IP:

With its private key, the certificate agency also signs the signature request from the client in client mode (`client`) and thus creates a client certificate (`client.crt`) and securely sends it back to the client.

    ./easyrsa sign-req client client
    scp pki/issued/client.crt aleks@CLIENT_IP:

To validate signed certificates by the certificate agency, the server and client also need the certificate agency's public key, which is also securely transmitted by the certificate agency.

    scp pki/ca.crt aleks@SERVER_IP:
    scp pki/ca.crt aleks@CLIENT_IP:

The certification agency now prepares a private and public key or certificate for accessing the VPN network as a client and signs it herself.

    ./easyrsa build-client-full caclient

Now we continue on the server where we create a folder for signed certificates and move the signed certificate into it from the home folder. We also move the certificate authority's public key into our keys folder.

    mkdir pki/issued
    mv /home/aleks/server.crt pki/issued/
    mv /home/aleks/ca.crt pki/

We move to the parent folder and create a configuration file for the OpenVPN server, which will work via the TCP protocol and create a tunnel on the 3rd layer of the network, i.e. `tun` mode. In the configuration file, set the protocol through which the VPN will work `proto tcp`, the layer on which the tunnel will be executed `dev tun`, the server will assign VPN IP addresses from the local network `server 10.8.0.0 255.255.255.0` and act as a subnet ` topology subnet` and enable visibility between clients in the `client-to-client` VPN network. We also add the public key of the certificate agency `ca.crt`, the private `server.key` and the public `server.crt` key of the server, as well as parameters for key exchange using the Diffie-Hellman process `dh.pem`.

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

Now let's start the OpenVPN server and test the operation.

    openvpn server.conf

We continue on the client where we create a folder for signed certificates and move the signed certificate into it from the home folder. We also move the certificate authority's public key into our keys folder.

    mkdir pki/issued
    mv /home/aleks/client.crt pki/issued/
    mv /home/aleks/ca.crt pki/

We move to the parent folder and create a configuration file for the OpenVPN client, which will work via the TCP protocol and create a tunnel on the 3rd layer of the network, i.e. `tun` mode. In the configuration file, set the protocol through which the VPN will work `proto tcp`, the layer on which the tunnel will be executed `dev tun`, set it to work as a client `client` and set the IP address of the OpenVPN server `remote SERVER_IP`. We also add the public key of the certificate agency `ca.crt`, the private `client.key` and the public `client.crt` key of the client.

    cd ..
    nano client.conf

    proto tcp
    client
    dev tun
    remote SERVER_IP

    ca client/pki/ca.crt
    cert client/pki/issued/client.crt
    key client/pki/private/client.key

Now let's run the OpenVPN client and test the operation.

    openvpn client.conf

We continue with the certification agency, which already has all the certificates in the right folders. We move to the parent folder and create a configuration file for the certificate agency's OpenVPN client, which will work via the TCP protocol and create a tunnel on the 3rd layer of the network, i.e. `tun` mode. In the configuration file, set the protocol through which the VPN will work `proto tcp`, the layer on which the tunnel will be executed `dev tun`, set it to work as a client `client` and set the IP address of the OpenVPN server `remote SERVER_IP`. We also add the public key of the certificate agency `ca.crt`, the private `caclient.key` and the public `caclient.crt` key of the client.

    cd ..
    nano caclient.conf

    proto tcp
    client
    dev tun
    remote SERVER_IP

    ca ca/pki/ca.crt
    cert ca/pki/issued/caclient.crt
    key ca/pki/private/caclient.key
    
Now let's run the OpenVPN client and test the operation.

    openvpn caclient.conf
