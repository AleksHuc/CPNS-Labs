# 9. Lab: Set up of a virtual private network using a shared secret

## Instructions
 
0. Use the network and virtual machines from the previous labs.
1. Set up and test OpenVPN operation between server and client in TUN mode with a shared secret over UDP.
2. Set up and test the operation of OpenVPN between the server and the client also via TAP mode with a shared secret over TCP

## Additional information

[Symmetric cryptography](https://en.wikipedia.org/wiki/Cryptography#Symmetric-key_cryptography) is an approach where a shared secret is used for encryption and decryption.

[Virtual Private Network (VPN)](https://en.wikipedia.org/wiki/Virtual_private_network) extends the local network over the public network and allows users to send and receive data over the public network as if they were connected directly through local network. VPN enables the functionality of the local network over the public network, ensures a secure connection and enables easier management of the local network.

[OpenSSL](https://www.openssl.org/) is a toolkit for general-purpose cryptography and secure communication.

[OpenVPN](https://en.wikipedia.org/wiki/OpenVPN) is an open source VPN implementation that allows a secure connection between two points on a network and functions functionally as an extension of a local network.

TUN - [IP Protocol](https://en.wikipedia.org/wiki/Internet_Protocol) tunnel at network layer 3.

TAP - [Ethernet protocol](https://en.wikipedia.org/wiki/Ethernet) tunnel at network layer 2.

## Detailed instructions

### 1. Task

Let's install OpenVPN package `openvpn` through our operating system's package manager on both virtual machines.

    apt install openvpn

We move to the home folder `/home/USERNAME` and create a new folder in which we will create all the necessary files for using OpenVPN.

    cd /home/aleks
    mkdir vpn_simple
    cd /vpn_simple

Before we start setting up the server and client, let's take a look at the example configuration files located in the folder `/usr/share/doc/openvpn/examples/sample-config-files`.

    nano /usr/share/doc/openvpn/examples/sample-config-files/server.conf
    nano /usr/share/doc/openvpn/examples/sample-config-files/client.conf

For data transfer, the VPN creates an encrypted tunnel, which in our case expects a shared secret, which is a symmetric key. First, we create it on the first virtual computer and install the `ssh` server to securely transfer the key to another virtual computer and enable access permissions to the key to our user.

    openvpn --genkey secret key.key

    apt install openssh-server

    chown aleks:aleks key.key

On the second virtual machine, we use the `scp` command to securely transfer the key from the first virtual machine.

    scp aleks@10.0.0.1:/home/aleks/vpn_simple/key.key /home/aleks/vpn_simple/key.key

On the first virtual computer, we create a configuration file for the OpenVPN server, which will work via the UDP protocol and create a tunnel on the 3rd layer of the network, i.e. `tun` mode. In the configuration file, set the protocol through which the VPN will work `proto udp`, the layer on which the tunnel will be executed `dev tun`, the VPN IP address of the server and the client `ifconfig 10.35.1.1 10.35.1.2` and the encryption key `secret key.key`.

    nano server_tun.conf

    proto udp
    dev tun
    ifconfig 10.35.1.1 10.35.1.2
    secret key.key
    providers legacy default

On another virtual computer, we create a configuration file for the OpenVPN client, which will work via the UDP protocol and a tunnel on the 3rd layer of the network, i.e. in `tun` mode. In the configuration file, set the outside IP address of the OpenVPN server `remote 10.0.0.1`, the protocol through which it will access the VPN `proto udp`, the layer on which the tunnel will be executed `dev tun`, the VPN IP address of the client and server `ifconfig 10.35.1.2 10.35.1.1` and the encryption key `secret key.key`.

    nano client_tun.conf

    remote 10.0.0.1
    proto udp
    dev tun
    ifconfig 10.35.1.2 10.35.1.1
    secret key.key
    providers legacy default

First, we start the OpenVPN server on the first virtual computer.

    openvpn server_tun.conf

Then we run the OpenVPN client on another virtual computer and check connectivity with the `ping` command.

    openvpn client_tun.conf

    ping 10.35.1.1

### 2. Task

Let's create an OpenVPN server that will work via the TCP protocol and create a tunnel on the 2nd layer of the network, i.e. `tap` mode. In the settings file, we set the protocol through which the VPN `proto tcp-server` will work, the layer on which the `dev tap` tunnel will be executed and the encryption key `secret key.key`.

    nano server_tap.conf

    proto tcp-server
    dev tap
    secret key.key
    providers legacy default

On another virtual computer, create a configuration file for the OpenVPN client, which will work via the TCP protocol and a tunnel on the 2nd layer of the network, i.e. in `tap` mode. In the configuration file, set the outside IP address of the OpenVPN server `remote 10.0.0.1`, the protocol through which it will access the VPN `proto tcp-client`, the layer on which the tunnel will be executed `dev tap` and the encryption key `secret key.key' `.

    nano client_tap.conf

    remote 10.0.0.1
    proto tcp-client
    dev tap
    secret key.key
    providers legacy default

First, we start the OpenVPN server on the first virtual computer. Since the tunnel is on the 2nd layer of the network, we still need to manually add the IP address to our server using the `ip` command.

    openvpn server_tap.conf

    ip addr add 10.35.1.1/24 dev tap0

    ip link set dev tap0 up

Then we run the OpenVPN client on another virtual computer and manually add the IP address to our client with the `ip` command and check connectivity with the `ping` command.

    openvpn client_tap.conf

    ip addr add 10.35.1.2/24 dev tap0

    ip link set dev tap0 up

    ping 10.35.1.1

And then we also test the operation from the first virtual computer.

    ping 10.35.1.2
