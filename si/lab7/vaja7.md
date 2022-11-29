# 7. Vaja: Omrežni čas

## Navodila
 
1. Implementirajte RFC 868.

## Dodatne informacije

[RFC (Released For Comments) 868](https://datatracker.ietf.org/doc/html/rfc868) definira protokol za omrežni čas [Network Time Protocol (NTP)](https://en.wikipedia.org/wiki/Network_Time_Protocol), ki deluje preko [UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol) in [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) protokolov.

NTP strežniki:
- ntp1.arnes.si
- ntp2.arnes.si

## Podrobna navodila

### 1. Naloga
Implementirajte preprost program v poljubnem jeziku, ki pošlje poizvedbo na NTP strežnik in zna razbrati čas iz odgovora.

Primer programa v programskem jeziku Python:

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

Primer programa v programskem jeziku Java:

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
