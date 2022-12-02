---
title: TCP Keep Alives on Linux
tags: [tcp, linux]
style: fill
color: primary
description: A look at real-world application and understanding of TCP Keep Alives.
---

# Preamble

There are many individuals who are conceptually aware of TCP Keep Alives and perhaps even their use-case. However in my experience there are few who know when and how they apply. In this blog post I hope to provide to walk through a scenario from problem statement to the investigation phase to identifying the root cause.


## Architecture

Our scenario starts with a simplified architecture of a Client, Stateful Network Device and a HTTP Server. In the real world there is often significantly more architecture involving the transit network. However for this case routing or how packets forward and return is not applicable.

```
        +---------+        +---------+        +---------+
        |  Our    |  --->  | Stateful|  --->  |  TCP    |
        |  Client |  <---  | Device  |  <---  |  Server |
        +---------+        +---------+        +---------+
```

## Application

The client needs to request some work to be done. This request is a long lived TCP connection and the application is specifically designed to not close the socket. This use-case is similar to any application which is using a long-polling method.

The code for the example application is below. However you can replace this application with any application with a similar use-case and the same theory applies.

```py
#Standard Imports
import socket

serversocket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
serversocket.bind(('0.0.0.0', 8089))
serversocket.listen()

try:
    while True:
        connection, address = serversocket.accept()
        buf = connection.recv(128)
        print(buf)
except KeyboardInterrupt:
    serversocket.close()
```

The SO_KEEPALIVE option is disabled by default on Linux because the TCP keepalive mechanism can consume additional resources on both the client and server, such as network bandwidth and CPU time. In many cases, a TCP connection will be terminated by the application before the keepalive mechanism is needed, so enabling the option by default would add unnecessary overhead. Additionally, some applications may not be designed to handle the keepalive mechanism, and enabling the option could cause unexpected behavior. For these reasons, the SO_KEEPALIVE option is disabled by default, and it is up to the user or application to enable it if needed.


<!---
linux checks via "ss -to" command, showing active TCB and other details

not all processes are actively using keep alive for their socket

RFC1122 / RFC9293: in order for each operating systems' TCP implementation to be considered compliant with the TCP Standard the operating system must allow the application to turn keep-alives on or off for each socket and they must default to off. Linux offers this socket option "SO_KEEPALIVE"

idle timeouts

https://www.man7.org/linux/man-pages/man7/socket.7.html

https://www.ietf.org/rfc/rfc9293.html#name-tcp-keep-alives

import socket

clientsocket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
clientsocket.connect(('18.218.61.202', 8089))
data = 'hello'
clientsocket.send(data.encode())

--->