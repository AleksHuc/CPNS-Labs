# 7. Lab: Network time

## Instructions
 
1. Implement the client for Time Protocol defined in RFC 868 and capture the exchanged packets with Wireshark.
2. Implement the client for Network Time Protocol defined in RFC 5905 and capture the exchanged packets with Wireshark.
3. Run time synchronization of your virtual machine and capture the exchanged packets with Wireshark.

## Additional information

[RFC (Released For Comments) 868](https://datatracker.ietf.org/doc/html/rfc868) defines the [Time Protocol (TP)](https://en.wikipedia.org/wiki/Time_Protocol), that works over [UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol) and [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) protocols. [RFC 5905](https://datatracker.ietf.org/doc/html/rfc5905) defines the [Network Time Protocol (NTP)](https://en.wikipedia.org/wiki/Network_Time_Protocol) that superseeded the Time Protocol and is today the main standard used for time synchornization for devices connected to the computer networks. 

TP and NTP servers:

- ntp1.arnes.si
- ntp2.arnes.si

## Detailed instructions

### 1. Task
Implement a simple program in any language that sends a query to the TP server and can determine the time from the response.

An example of a program in the Python programming language using TCP:

    import socket
    import struct
    import datetime

    port = 37
    host = "ntp1.arnes.si"

    sock = socket.socket()
    sock.connect((host, port))

    # Read 4 bytes from a socket.
    data = sock.recv(4)

    # When we have numbers larger then 1 byte. Internet orders
    # bytes in such a way that the first byte is most important
    # -> BigEndian. Intel processors order bytes in such a way
    # that the last byte is most important -> LittleEndian.
    num = struct.unpack("!I", data)[0]

    print(data, num)
    
    # Starting time in unix is from 1.1.1970 in RFC is 1.1.1900.
    # Difference in seconds is 2208988800.
    
    networktime = datetime.datetime.fromtimestamp(num-2208988800)

    print(networktime)

An example of a program in the Java programming language Using UDP:

    import java.io.DataInputStream;
    import java.io.IOException;
    import java.net.Socket;
    import java.time.LocalDateTime;
    import java.time.ZoneOffset;

    public class NTP {

        public static void main(String args[])
        {
            try {
                Socket socket = new Socket("ntp1.arnes.si", 37);

                DataInputStream dis = new DataInputStream(socket.getInputStream());
                int i = dis.readInt();  // Unsigned integer!
                
                System.out.println(i);
                System.out.println(Integer.toBinaryString(i));
                System.out.println(Long.toBinaryString(i));
                
                // When cast into long the most important bit is
                // propagated into the first bytes of the long.
                // We multiply it with 0xffffffffL, which is 4
                // bytes of 1, so that the propagated 1 from the
                // long top 4 bytes are set back to 0.
                
                long time = (long)i & 0xffffffffL;
                System.out.println(Long.toBinaryString(0xffffffffL));
                System.out.println("Time:" + time);

                // NTP time starts from 1.1.1900
                // Epoch time starts from 1.1.1970

                long cas_epoch = time - 2208988800L;
                
                System.out.println("cas_epoch:" + cas_epoch);
                System.out.println("LocalTime:" + LocalDateTime.
                
                ofEpochSecond(cas_epoch, 0, ZoneOffset.of("+1")));

            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

### 2. Task

Implement a simple program in any language that sends a query to the NTP server and can determine the time from the response.

An example of a program in the Python programming language:

	import socket
	import struct, time


	host = "pool.ntp.org"
	port = 123
	address = (host, port)

	# Leap Indicator = 0 (00), Version Number = 3 (011), Mode = 3 - client (011)
	msg = '\x1b' + 47 * '\0'

	# Reference time (in seconds since 1900-01-01 00:00:00)
	TIME1970 = 2208988800  # 1970-01-01 00:00:00

	# Connect to the server and receive the response.
	client = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
	client.sendto(msg.encode('utf-8'), address)
	msg, address = client.recvfrom(48)

	# Read 12 32-bit unsigned Integer values in Big-Endian mode. The 11-th value presents the time in seconds.
	t = struct.unpack("!12I", msg)[10]
	t -= TIME1970
	print(time.ctime(t))

### 3. Task

In the Debian operating system, we manage time through a tool called [`timedatectl`](https://www.man7.org/linux/man-pages/man1/timedatectl.1.html).

	timedatectl status

	               Local time: Mon 2024-11-25 13:59:12 CET
	           Universal time: Mon 2024-11-25 12:59:12 UTC
	                 RTC time: Mon 2024-11-25 12:59:12
	                Time zone: Europe/Ljubljana (CET, +0100)
	System clock synchronized: no
	              NTP service: active
	          RTC in local TZ: no

To trigger another NTP query, restart the `timesyncd` service.

	systemctl restart systemd-timesyncd 