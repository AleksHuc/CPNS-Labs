# 11. Lab: Network management with the Firewall

## Instructions

0. Use the network and virtual machines from the previous exercises.
1. Enable the second virtual machine to access the Internet through the first virtual machine.
2. On the first virtual machine, prevent the second virtual machine from accessing the IP address `8.8.8.8.
3. On the first virtual machine, prevent the second virtual machine from accessing the first virtual machine via secure shell (SSH).
4. On the first virtual machine, prevent the transfer of packets via the HTTP protocol after we have already transferred 1 MB of data.

## More information

[Netfilter](https://en.wikipedia.org/wiki/Netfilter) is a framework that is part of the Linux kernel and allows us to manage the network, for example: filter packets, map IP addresses and ports for the purpose of routing and preventing access to parts of the network.

[iptables](https://en.wikipedia.org/wiki/Iptables) represents an outdated packet filtering tool based on the Netfilter hook system.

[nftables](https://en.wikipedia.org/wiki/Nftables) a new tool that replaces `iptables` and provides additional functionality, such as packet inspection.

## Detailed instructions

### 1. Task

If we used `iptables` until now, then we delete all the rules.

    iptables -t nat -L
    iptables -t nat -F

The current `nftables` rules can be checked with the `nft` command. [Manual](https://www.netfilter.org/projects/nftables/manpage.html) for using `nft`.

    nft list ruleset

The `nftables` tool uses tables (`tables`) and chains (`chains`) to store rules hierarchically (`rules`). All tables, chains and rules have their own identifiers (`handles') which we use for deletion. For example:

    nft -a list table mytable
    nft delete rule mytable mychain handle 10

On the first virtual machine, enable the gateway by creating a new `table`, a new `chain` and a new `rule` to map packet IP addresses.

    nft add table mytable
    nft 'add chain mytable postroutingchain { type nat hook postrouting priority -100; }'
    nft add rule mytable postroutingchain masquerade

We also need to enable routing on the first virtual machine in the `/etc/sysctl.conf` file.

    nano /etc/sysctl.conf

    net.ipv4.ip_forward=1

To take into account the changes in Linux kernel parameters, use the `sysctl` command.

    sysctl -p

### 2. Task

On the first VM, we add a stateless rule to filter all packets with the sink IP address `8.8.8.8` originating from the second virtual machine.

    nft 'add chain mytable forwardchain { type filter hook forward priority -100; }'
    nft add rule mytable forwardchain ip daddr . ip saddr { 8.8.8.8 . SECOND_VM_IP } drop

### 3. Task

On the first virtual machine, add a stateless rule to filter all [`SSH`](https://en.wikipedia.org/wiki/Secure_Shell) packets originating from the second virtual machine.

    nft 'add chain mytable inputchain { type filter hook input priority -100; }'
    nft add rule mytable inputchain tcp dport . ip saddr { 22 . SECOND_VM_IP } drop

### 4. Task

On the first virtual machine, add a stateful rule to filter all packets with the [`HTTP`](https://en.wikipedia.org/wiki/HTTP) protocol when the received packet size exceed 1MB. If you don't have an `HTTP` server installed yet, install it, create any file with a size less than 1 MB, for example with the [`dd`](https://man7.org/linux/man-pages/man1/dd.1.html) command, and move it to the `\var\www\html` folder.

	apt update
	apt install lighttpd

	dd if=/dev/zero of=output.dat  bs=400K  count=1
	1+0 records in
	1+0 records out
	409600 bytes (410 kB, 400 KiB) copied, 0.00104389 s, 392 MB/s

	mv output.dat /var/www/html/

Now we add rules for storing the transferred amount of data and enable packet dropping when the selected limit is exceeded.

	nft add table filter
	nft add counter filter http-traffic
	nft add quota filter http-quota over 1 mbytes
	nft add chain filter output { type filter hook output priority 0 \; }
	nft add rule filter output tcp sport http counter name http-traffic
	nft add rule filter output tcp sport http quota name http-quota drop

Let's check the status of the current rules.

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

We run a file transfer request on the second computer and check the status of the rules on the first computer again.

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

We run the file transfer request again on the second computer and check the status of the rules on the first computer again.

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

We run the file transfer request again on the second computer and check the status of the rules on the first computer again.

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

We can simplify the counter and quota with the following commands and now the HTTP transfer is working again.
	
	nft reset counters
	nft reset quotas
