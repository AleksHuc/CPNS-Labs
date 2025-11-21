# 3. Lab: Operation of simple programs over the network

## Instructions

0. Use the network and virtual machines from 2. lab.
1. Install a Trivial File Transfer Protocol (TFTP) server and find out how it works (`tftpd`)?
2. Find out what `inetd` is and how it works?
3. Implement a simple Hypertext Transfer Protocol (HTTP) server running through `inetd`.

## Additional information

The command [`ls`](https://linux.die.net/man/1/ls) allows us to print the contents of the folders in the file system.

The [`mkdir`](https://linux.die.net/man/1/mkdir) command is used to create new folders in the file system.

The [`touch`](https://linux.die.net/man/1/touch) command allows us to create files and change file timestamps.

The [`echo`](https://linux.die.net/man/1/echo) command is used to output lines of text to standard output.

The command [`awk`](https://linux.die.net/man/1/awk) allows us to find and process text patterns from strings.

The command [`wc`](https://linux.die.net/man/1/wc) allows us to count newlines, words and bytes in individual text files.

The [`chmod`](https://linux.die.net/man/1/chmod) command is used to change how users, groups, and others are allowed to access files.

The command [`curl`](https://linux.die.net/man/1/curl) allows us to transfer data from the server to the local computer via a bunch of supported protocols.

## Detailed instructions

### 1. Task

We install the TFTP server via the package manager of the installed Linux distribution on the first virtual computer with a DHCP server. [TFTP](https://en.wikipedia.org/wiki/Trivial_File_Transfer_Protocol) is a simple protocol for transferring files over a network to remote computers, the specification of which can be found in [RFC 1350](https://www.rfc-editor. org/rfc/rfc1350). For example, let's install the `tftpd` implementation of the TFTP protocol (there are also `tftpd-hpa`, `atftpd` ...).

    apt install tftpd

The installed `tftpd` server is already running with the default settings specified in the configuration file `/etc/inetd.conf`. For example, with the following line:

    tftp dgram udp wait nobody /usr/sbin/tcpd /usr/sbin/in.tftpd /srv/tftp

We can see that `tftpd` needs `inetd` to run. At the moment, the last parameter is important for us, which points to the `/srv/tftp` folder, which we check to see if it exists, and we see that it doesn't.

    ls /srv/tftp
    
    ls: cannot access '/srv/tftp': No such file or directory

In case you cannot find the `tftpd` package with the package manager, then install the `tftpd-hpa` package and follow the instruction from here.

The mentioned folder serves to store files that the TFTP server will be able to transfer over the network, so we create it and save any file in it.

    mkdir /srv/tftp
    touch test.txt

Now let's test the operation of the TFTP server by installing the TFTP client `tftp` on another virtual machine and downloading the test file.

    apt install tftp

    tftp
    connect IP_DHCP_SERVER
    get FILE_NAME
    quit

### 2. Task

When installing `tftpd`, we found that the package [`openbsd-inetd`](https://man.openbsd.org/inetd) was also installed, which is one of the many implementations of `inetd` (there are also `xinetd` , `rinetd`, `rlinetd`, `inetutils-inetd` ...). We also found that `tftpd` works through `inetd` and we also configure it through the `/etc/inetd.conf` configuration file.

So, `inetd` is a program that runs in the background of the operating system (service, daemon), which listens on the configured network ports and enables connections to them. When a network device establishes a connection to a configured port, `inetd` runs the program specified for that port and forwards all received data to the program's standard input. When the running program finishes executing, `inetd` passes the data from the program's standard output through the network port back to the original network device that established the connection. `inetd` allows us to run programs over the network, without the program itself taking care of it.

The individual programs that `inetd` runs, are set in the `/etc/inetd.conf` file with the following parameters:

| Service name or port | Socket type | Protocol | wait/nowait | User   | Program path   | Program arguments            |
|----------------------|-------------|----------|-------------|--------|----------------|------------------------------|
| tftp                 | dgram       | udp      | wait        | nobody | /usr/sbin/tcpd | /usr/sbin/in.tftpd /srv/tftp |

### 3. Task

Let's write a simple HTTP server that simply returns the requested HyperText Markup Language ([HTML](https://en.wikipedia.org/wiki/HTML)) file on `GET` request.

    nano server.sh

    #!/bin/bash

    read request

    file=$( echo $request | awk '$1 == "GET" { print $2 }' )

    echo "HTTP/1.1 200 OK"
    echo "Content-Length: $(wc -c < $1$file)"
    echo ""
    cat $1$file

Let's check the execution rights of our HTTP server and enable them.

    ls -ls

    4 -rw-r--r-- 1 aleks aleks  185 Oct 24 11:55 server.sh

    chmod u+x server.sh

Let's create an HTML file that our HTTP server will return.

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

Let's test the operation of the HTTP server locally.

    ./server.sh /home/aleks

    GET /home/aleks/index.html HTTP/1.1

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


If we don't already have an `inetd` package installed, we install it with a package manager.

    apt install openbsd-inetd

Now add our HTTP server to the configuration file `/etc/inetd.conf` and then restart the `inetd` service. For the name of the service, select `http` or the port `80` can also be used. We choose `stream` for the socket type because we will be using Transmission Control Protocol ([TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)), which means we choose `tcp` for the protocol. We set the server so that it is always available and does not wait for the program it ran to finish using the `nowait` setting. Then we select a user that has the permissions to run our simple HTTP server, for example `aleks`. The next parameter is the absolute path to our program, which is `/home/aleks/server.sh` and the arguments that are given when starting the program. The first argument is, by convention, the very name of the `server.sh` program. The second argument points to the folder where our `index.html` is located, which is `/home/aleks`.

    nano /etc/inetd.conf

    http stream tcp	nowait aleks /home/aleks/server.sh server.sh /home/aleks

    systemctl restart inetd.service

We test the operation over the network with the program for sending HTTP requests from the command console `curl` on the second virtual machine.

    apt install curl

    curl 10.0.1.1/index.html --verbose