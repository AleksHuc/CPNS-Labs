# 2. Lab: Network setup with automatic network address assignment

## Instructions

1. Set up two virtual machines. The first has two network adapters, where the first is connected to the `NAT` network and the second to the `Internal network`. The other has one network adapter connected to the `Internal network`.
2. Place a Dynamic Host Configuration Protocol (DHCP) server on the first virtual machine that has two network adapters. DHCP should run on the `Internal network` and allow other virtual machines to automatically obtain network addresses.
3. Make sure that the virtual computer that is connected only to the `Internal network` can access the Internet through the virtual computer on which the DHCP server is running.

## Additional information

The [`sysctl`](https://linux.die.net/man/8/sysctl) command allows us to manage Linux kernel parameters at runtime.

Network setup instructions for configuration file [`/etc/network/interfaces`](https://manpages.debian.org/stretch/ifupdown/interfaces.5.en.html).

Firewall [`iptables`](https://linux.die.net/man/8/iptables) or [`nf_tables`](https://manpages.debian.org/buster/nftables/nft.8.en.html) allows the management and filtering of network packets entering or traversing a single network adapter.

The command [`journalctl`](https://www.man7.org/linux/man-pages/man1/journalctl.1.html) allows us to read `systemd` logs.

The [`systemctl`](https://www.man7.org/linux/man-pages/man1/systemctl.1.html) command allows us to manage `systemd` system and manage programs running in the background.

## Detailed instructions

### 1. Task

Let's create two virtual computers as we did in the previous labs, where the first has two network adapters, where the first is connected to the `NAT` network and the second to the `Internal network`.

![Setting the first adapter on the first virtual machine to the `NAT` network.](images/lab2-vbox1.png)

![Setting the second network adapter on the first virtual machine to the `Internal network`.](images/lab2-vbox2.png)

The other one has one network adapter, which is connected to the `Internal network`. We must be careful to choose the same name for both virtual computers network adapters with the `Internal network`, for example, `intnet`.

![Setting the first adapter on the second virtual machine to a `Internal network`.](images/lab2-vbox3.png)

### 2. Task

We start the first virtual computer and check the status of both configured network adapters with the `ip` command.

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

We get the following printout, from which we see that both network adapters were unable to obtain IP network addresses, which is the result of missing configuration and the operation of the `network-manager` program, which manages networks. To continue setting up the network, we will turn off the `network-manager` and prepare the necessary file settings to achieve the desired operation.

    su -
    systemctl stop NetworkManager.service
    systemctl disable NetworkManager.service

We set the network adapters in the file `/etc/network/interfaces` in such a way that `enp0s3` represents the network adapter in the `NAT` network, which obtains the network address automatically via DHCP, and `enp0s8` represents the network adapter in the `Internal network`, which has a static address because the DHCP server will work through it. Open the settings file with any text editor and add the following settings.

    nano /etc/network/interfaces

    auto enp0s3
    iface enp0s3 inet dhcp

    auto enp0s8
    iface enp0s8 inet static
      address 10.0.1.1
      netmask 255.255.255.0

For the settings to be taken into account, we restart the operation of the network adapters on the virtual computer.

    systemctl restart networking.service

We have now prepared everything necessary to install and configure the [DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) server, which will assign IP network addresses automatically to all computers in the `Internal network`. The specification of the protocol can be found in [RFC2131](https://www.rfc-editor.org/rfc/rfc2131). With the package manager we can search by the names of packages that can be installed with the command `apt search NAME`. For example, let's install the DHCP server implementation `isc-dhcp-server`.

    apt update
    apt install isc-dhcp-server

After the installation, the DHCP server gives us an error that it could not start. We arrive at the listed error by reviewing the latest logs and finding out that the settings of the network adapter on which it will work and the settings of the subnet that it will manage are missing.

    journalctl -xe -n 100

In the `/etc/default/isc-dhcp-server` file, set the network adapter on which the `isc-dhcp-server` DHCP server should run.

    nano /etc/default/isc-dhcp-server

    INTERFACESv4="enp0s8"

In the file `/etc/dhcp/dhcpd.conf`, we set which network will be operated by the DHCP server, i.e. which IP network addresses will be assigned network devices, the IP address of the main gateway, the [DNS](https://en.wikipedia.org/wiki/Domain_Name_System) server (for example, the public Cloudflare DNS server with the IP address `1.1.1.1`) and other network properties.

    nano /etc/dhcp/dhcpd.conf
    
    subnet 10.0.1.0 netmask 255.255.255.0 {
      range 10.0.1.100 10.0.1.200;
      option routers 10.0.1.1;
      option domain-name-servers 1.1.1.1;
    }

For the settings to take effect, restart the `isc-dhcp-server' DHCP server.

    systemctl restart isc-dhcp-server.service

To check the operation of our DHCP server, we now start the second virtual computer and check its IP network address. If DHCP is working correctly, the second virtual machine should get an address from the 10.0.1.100 to 10.0.1.200 IP address block automatically, if the `network-manager` is running on it. If both computers are accessible via the network, check with the `ping' command, as we did in the previous exercises.

    ip a

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host valid_lft forever preferred_lft forever
    2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:40:07:ad brd ff:ff:ff:ff:ff:ff
        inet 10.0.1.100/24 brd 10.0.1.255 scope global dynamic noprefixroute enp0s3 valid_lft 514sec preferred_lft 514sec
        inet6 fe80::a00:27ff:fe40:7ad/64 scope link noprefixroute valid_lft forever preferred_lft forever

On the second virtual computer, you can also manually add DNS servers in the `/etc/dhcp/dhclient.conf` file.

    nano /etc/dhcp/dhclient.conf

    prepend domain-name-servers 1.1.1.1;

On the second virtual machine, we can restart networking to renew DHCP for a new IP address to also obtain new DNS settings.

    systemctl restart isc-dhcp-server.service

### 3. Task

We achieved that the virtual computer is running a DHCP server that automatically assigns IP network addresses in the `Internal network`, which we verified with another virtual computer. When testing the operation of the network, we found out with the `ping' command that the second virtual computer can access the first and vice versa, but it does not have access to the Internet. We will now address this shortcoming by turning on network routing on the first virtual machine.

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


First, let's enable routing on the first virtual machine in the `/etc/sysctl.d/sysctl.conf` file.

    nano /etc/sysctl.d/sysctl.conf

    net.ipv4.ip_forward=1

To take into account the changes in Linux kernel parameters, use the `sysctl` command.

    sysctl -p /etc/sysctl.d/sysctl.conf

With the packet manager, we install the `iptables` package, which represents a firewall and allows mapping IP addresses to network addresses by our first virtual machine.

    apt install iptables

    iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

We can also check the entry of the rule in `iptables`.

    sudo iptables -t nat -L -v

Now let's check again the availability of local and public IP network addresses.

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

The rules we enter in `iptables` are not maintained when the system restarts. By using the `iptables-persistent` package, we can store them and enable them to be automatically taken into account at restart.

    apt install iptables-persistent
    iptables-save > /etc/iptables/rules.v4