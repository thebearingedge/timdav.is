---
title: UNIX Domain Sockets vs. TCP Sockets
date: 2019-12-01 10:43:20
tags:
---

The term inter-process communication (IPC) is usually used to describe operating system mechanisms that allow processes to share information with one another **on the same computer**. But in reality, it's a very general term that doesn't require processes to be running on the same machine.
<!-- excerpt -->

```
mysqli_real_connect(): (HY000/2002): No such file or directory
```

I cannot remember how long ago I first came across this PHP error message. It was probably around 7 or 8 years ago when I was hacking together my first web site on the LAMP stack. It feels like a really long time ago; back before I knew anything about programming, Linux, the World Wide Web, or the Internet. I couldn't tell you what the error means, but I got past it and my site went live and some years have passed since I switched to PostgreSQL for most projects.

Nowadays, when I see this error, it's because a student forgot to start their MySQL database server. I suggest they start the `mysql` service in their Ubuntu dev environment and then everything works as expected. But how did I _know_ that this was the issue?

**How the hell does this error suggest that MySQL is not running?!**

At some point I just memorized the possible reasons that this errors occurs. So I figured I'd write a little about it.

## Computer Processes in Web Applications

[Computer processes](https://en.wikipedia.org/wiki/Process_(computing)) are the programs running on a computer. You can see which programs are running on a machine by opening Activity Monitor (macOS), Task Manager (Windows), or executing the `top` command (Linux). Some of these programs need to work together as a system for some larger purpose.

When I teach "full stack" web development, I spend a lot of time talking about a few types of programs and how they work together as the tiers of a Web application:

- Web Client programs like Google Chrome, HTTPie, or Postman that work with...
- Web Server programs like Apache, Nginx, or custom Node.js servers that work with...
- Database Server programs like MySQL, PostgreSQL, or MongoDB

If you are having connectivity issues, then your first step is to verify that the client, server, and data tiers are all running.

## TCP Sockets

When I was self-studying, I slowly pieced together how clients and servers shoot [HTTP messages](https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages) at each other over [TCP connections](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Protocol_operation). I don't often interact with sockets directly; these are hidden behind client libraries or web frameworks, where HTTP request and response messages are the center of attention.

Here's my simplified summary of an HTTP exchange between a client and a server over TCP:

1. Assuming the client knows the network address and port of the server, the client can attempt to establish a TCP connection to it.
1. If the network is up and the server is listening, then the connection is established via a [fancy handshake](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_establishment).
1. The client sends an HTTP Request message over the connection.
1. The server processes the request and then sends an HTTP Response message over the connection.
1. The client and server then agree to close the TCP connection with [another fancy handshake](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_termination).

While a TCP connection is open, the client and server programs each receive a special data structure from their respective host operating systems. This data structure is a "socket" and it represents a network communication endpoint. Each program writes to or reads from its socket whenever it wants to exchange data with the program at the other end of the connection.
<br/>
{% image fancybox center group:travel victorian-network-sockets.png "I like imagining people using tin cans on a string to simulate a telephone line." %}

After I got my head around the ideas of [hosts](https://en.wikipedia.org/wiki/Host_(network)), [IP addresses](https://en.wikipedia.org/wiki/IP_address), and [ports](https://en.wikipedia.org/wiki/Port_(computer_networking)), my mental model of how clients, servers, and databases communicate with one another firmed up.

Sure, I got tripped up by the occasional error:

- `EADDRINUSE` - my server program is trying to bind to an occupied port
- `ECONNREFUSED` - my client program is trying to connect to the wrong address or port

But these all sort of "fit" within my mental model of network communications.

## Inter-process Communication

The term [inter-process communication (IPC)](https://en.wikipedia.org/wiki/Inter-process_communication) is usually used to describe operating system mechanisms that allow processes to share information with one another **on the same computer in the same operating system**. But in reality, it's a more general term that doesn't require processes to be running on the same computer. In fact, network sockets, like those summarized above, are among the various IPC facilities that exist.

## UNIX Domain Sockets

This is a topic I've never really looked into until now. I'm often trying to come up with quick and simple ways to make underlying technologies (like TCP) more concrete to my students. So I've been playing around with the [`net` module](https://nodejs.org/docs/latest-v10.x/api/net.html#net_net) while writing some Node.js.

Then I saw this:

> The `net` module supports IPC with named pipes on Windows, and **UNIX domain sockets** on other operating systems.
[&mdash; nodejs.org/docs](https://nodejs.org/docs/latest-v10.x/api/net.html#net_ipc_support)

There are a number of ways to start a `net.Server`, including listening at a UNIX socket path:

```
server.listen(path[, backlog][, callback])
```

So after a quick bit of reading I came to understand that a [UNIX domain socket](https://en.wikipedia.org/wiki/Unix_domain_socket) can be used to set up bi-directional communication between processes, much like TCP connections, but without the overhead (or security implications) of using the operating system's network stack. These sockets are represented as files _in the filesystem hierarchy_ as opposed to just file descriptors (see [Everything is a file](https://en.wikipedia.org/wiki/Everything_is_a_file)). This means that you can actually see the file once it's been created by the socket server program.

> If the UNIX domain socket (that is visible as a file system path) is created and used in conjunction with one of Node.js' API abstractions such as net.createServer(), it will be unlinked as part of server.close().
[&mdash; nodejs.org/docs](https://nodejs.org/docs/latest-v10.x/api/net.html#net_identifying_paths_for_ipc_connections)

Naturally, I had to fool around with it, and I ended up with a little [GitHub repository](https://github.com/thebearingedge/socket-test) for a barebones "chat" server that doesn't work over the network. Super useful, right?
<br/>
{% image fancybox center group:travel socket-test.gif "Fooling around with a socket server." %}

Still not satisfied with how much time I had wasted, I decided to start an HTTP server with Node.js and have it listen on a socket path so I could try to manually type it HTTP requests via the command line.

For science.

Back to this PHP gem:

```
mysqli_real_connect(): (HY000/2002): No such file or directory
```

I've always seen a `host` or `port` being passed to `mysqli_real_connect()` so the above error doesn't really line up with my understanding. What file?

Digging into the documentation `mysqli_real_connect()` gives a hint near the bottom.

> **Note:** Specifying the **socket** parameter will not explicitly determine the type of connection to be used when connecting to the MySQL server. How the connection is made to the MySQL database is determined by the **host** parameter.
[&mdash; PHP.net](https://www.php.net/manual/en/mysqli.real-connect.php)</cite>

But I'm usually using `'localhost'` as my `host` parameter when connecting to MySQL. The MySQL docs clear this up.

> On Unix, MySQL programs treat the host name `localhost` specially, in a way that is likely different from what you expect compared to other network-based programs: the client connects using a Unix socket file. The `--socket` option or the `MYSQL_UNIX_PORT` environment variable may be used to specify the socket name.
[&mdash; mysql.com](https://dev.mysql.com/doc/refman/8.0/en/connecting.html)

I'd seen this part of the MySQL configuration a long time ago. Even looking at it now, my eyes go straight to the `port` because I think of database servers as network services. "Oh, yes! That's MySQL's default port!"

```
# /etc/mysql/mysql.conf.d/mysqld.cnf
[mysqld]
user            = mysql
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
port            = 3306
basedir         = /usr
datadir         = /var/lib/mysql
tmpdir          = /tmp
lc-messages-dir = /usr/share/mysql
skip-external-locking
```

**The part I was fuzzy on was the `socket` setting**. MySQL _also_ accepts connections via a special type of file known as a [Unix domain socket](https://en.wikipedia.org/wiki/Unix_domain_socket).

So as an experiment I ran `ls -lah /var/run/mysqld` in my Ubuntu Docker container.

```
total 8.0K
drwxr-xr-x 1 mysql root 4.0K Dec  2 03:01 .
drwxr-xr-x 1 root  root 4.0K Dec  1 16:59 ..
```

Nothing. Then `sudo service mysql start` and look again.

```
total 16K
drwxr-xr-x 1 mysql root  4.0K Dec  2 02:33 .
drwxr-xr-x 1 root  root  4.0K Dec  1 16:59 ..
-rw-r----- 1 mysql mysql    5 Dec  2 02:33 mysqld.pid
srwxrwxrwx 1 mysql mysql    0 Dec  2 02:33 mysqld.sock
-rw------- 1 mysql mysql    5 Dec  2 02:33 mysqld.sock.lock
```

Holy shit!

There's an _empty file_ there, named `mysql.sock` and its type is `s` (for socket!).

This explains the error message `No such file or directory`. When passing `'localhost'` as the first parameter to `mysqli_real_connect()`, it's assumed that MySQL has been started and `/var/run/mysqld/mysqld.sock` exists. When MySQL is stopped, it deletes the socket file.
