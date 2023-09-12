# UDP Tutorial

In this tutorial, we will discuss how to get a basic UDP client and server up and running using `C/C++`. The User Datagram Protocol (UDP) is a transport layer networking protocol. Unlike TCP it does not offer any reliability guarantees, nor does it respond to congestion on the network. It is simply a way of addressing network packets to a specific process which is often running on another machine. That packet is addressed by a *port* number ranging from 1 to 65535, though some of the port numbers are reserved for specific applications (e.g., `ssh`, `http`, `https`, etc.). Any single port can only have one process listening on it, but a single process can listen on many ports. 

UDP is a transport layer protocol. This means that is one layer above the Internet Protocol (IP). IP is used for addressing physical and virtual machines. When a process wants to send a packet to another machine, it creates an IP packet containing an IP address. Inside the body of that packet, there will be an UDP packet containing source and destination port numbers among other things.

## Sockets
You may have heard the expression that "in Linux, everything is a file." That remains true for network programming. We use a special type of file called a socket. When we write data to a socket, Linux will take care of the details of sending that data out on the network (i.e., constructing the packets and shuttling the bits to the network card, etc.).  Conversely, when we read from a socket, Linux will tell us a little bit of info about where the packet came from. But first we need to tell the OS a little bit of information about what kinds of messages the socket should expect.

### Configure listener socket

We will need a few `addrinfo` structs. We will be passing these to various functions which will fill them in with details that we can later use or retrieve.
```c
  struct addrinfo hints, *res, *p;
```

We will use the `getaddrinfo()` function to setup all the structs we will need later. The things we need to pass the function are: the hostname/IP, the port, some details about packets, and a place where the function can store the information we care about. In the following, the hostname/IP is set to `NULL` meaning that function does not need to set the destination IP because this is a listener connection.
The `PORT` is just the port number, written as a string. The pointer to `hints` is where we tell the OS some details about the packets. In the first four lines below, we are telling the OS that both versions of IP (IPv4 and IPv6) are fine. The next line, we use `SOCK_DGRAM` to indicate that we want to use UDP---if we wanted TCP, we would used `SOCK_STREAM`. The `AI_PASSIVE` tells the OS to fill in IP address automatically. This function returns a non-zero status upon failure. The `res` pointer holds the results---all the structs that the function may have returned (it may return multiple if the machine has multiple IPs and/or if both IPv4 and IPv6 are being used).
```c
  memset(&hints, 0, sizeof hints); // make sure the struct is empty
  hints.ai_family = AF_UNSPEC;     // don't care IPv4 or IPv6
  hints.ai_socktype = SOCK_DGRAM; // UDP datagram sockets
  hints.ai_flags = AI_PASSIVE;     // fill in my IP for me

  if ((status = getaddrinfo(NULL, PORT, &hints, &res)) != 0) {
    fprintf(stderr, "getaddrinfo error: %s\n", gai_strerror(status));
    exit(1);
  }
```

Assuming we're fine with using any of the returned structs, as long as they work, we can just iterate through the list stored in `res` and try continue our setup with each set of connection information until one succeeds. In particular, the setup we need to do is 
1. Determine if the struct is for IPv4 or IPv6, and cast it to the appropriate struct.
2. Convert the IP address into human readable form
3. Create a socket with the provided info in the struct
4. Bind the socket to the IP address
```c
  for (p = res; p != NULL; p = p->ai_next) {
    void *ina;

    /** step 1 **/
    if (p->ai_family == AF_INET) { // IPv4
      struct sockaddr_in * sa4 = (struct sockaddr_in *)p->ai_addr;
      ina = &(sa4->sin_addr);
    } else { // IPv6
      struct sockaddr_in6 * sa6 = (struct sockaddr_in6 *)p->ai_addr;
      ina = &(sa6->sin6_addr);
    }

    /** step 2 **/
    inet_ntop(p->ai_family, ina, ipstr, sizeof ipstr);

    printf("Listening on interface: %s\n", ipstr);
    info(logger, "Listening on interface: %s\n", ipstr);

    /** step 3 **/
    if ((listener_fd = socket(p->ai_family, p->ai_socktype, p->ai_protocol)) == -1) {
      perror("server: socket");
      continue;
    }

    /** step 4 **/
    if (bind(listener_fd, p->ai_addr, p->ai_addrlen) == -1) {
      perror("server: bind");
      continue;
    }

    break;
  }
```
In the code above `listener_fd` is the file descriptor of the socket that we will ultimately want to read from.

Notice that unlike with TCP, we do not need to call `listen()` because we are not waiting on a connection (UDP is connectionless).

## Reading from the socket
We have already created the socket, so we will use this to get the system to fill in some information about received packets. We will also declare two `char[]` for storing
```c
  struct sockaddr_storage peer_addr;
  char host[MAXBUF];
  char service[20];
```

Assign the same values to `hints`.
```c
  len = sizeof(peer_addr);

  memset(&hints, 0, sizeof hints); // make sure the struct is empty
  hints.ai_family = AF_UNSPEC;     // don't care IPv4 or IPv6
  hints.ai_socktype = SOCK_DGRAM; // UDP datagram sockets
  hints.ai_flags = AI_PASSIVE;     // fill in my IP for me
```

Clear the memory comprising the structs.
```c
memset(&peer_addr, 0, sizeof peer_addr);
memset(buffer, 0, sizeof buffer);
```

Then use `recvfrom()` to read from socket `listener_fd` into `buffer`.
```c
bytes_read = recvfrom(listener_fd, 
			          (char *)buffer, // buffer where results are stored
					  MAXBUF, // size of "buffer"
                      NULL,
                      (struct sockaddr *)&peer_addr, // put sender details here
                      &len); // size of peer_addr struct
```

The function `recvfrom()` returns `-1` if no bytes were read, i.e., no message was received. If a message was received, we can extract info about the message and the sender. The function `getnameinfo()` will return the hostname of the sender.
```c
if (bytes_read != -1) {
    if (peer_addr.ss_family == AF_INET) { // IPv4 */
        size = sizeof(struct sockaddr_in);
    } else {
        size = sizeof(struct sockaddr_in6);
    }

    memset(host, 0, sizeof host);
    memset(service, 0, sizeof service);
    debug(&logger, "Received message: %s\n", buffer);
    
    if ((status = getnameinfo((struct sockaddr *)&peer_addr, size, host, sizeof host, service, sizeof service, 0)) != 0) {
        fprintf(stderr, "getnameinfo error: %s\n", gai_strerror(status));
        exit(1);
    }
    debug(&logger, "Received message from %d\n", host);
    for (i = 0; i < host_count; i++) {
        if (host == hosts[i]) pings_received[i] = 1;
    }

    if (check_pings(pings_received, host_count) == 0) {
        fprintf(stderr, "READY\n");
        ready = 1;
    }
}

```

## Sending UDP messages
The setup for sending is much the same as with receiving. First, fill the structs using `getaddrinfo()`. Note that this time you'll want to provide the destination hostname instead of `NULL` as the first argument.

```c
if ((status = getaddrinfo(hostname, PORT, &hints, &res)) != 0) {
    fprintf(stderr, "getaddrinfo error: %s\n", gai_strerror(status));
    exit(1);
}
```

Next, configure the socket based on the structs from the previous step. You can again iterate through the results, and again you can stop as soon as you find one that works.
```c
// Configure socket
for (p = res; p != NULL; p = p->ai_next) {
	if ((sender_fd = socket(p->ai_family, p->ai_socktype, p->ai_protocol)) == -1) {
	    perror("peer: socket");
	    continue;
	}
    break;
}
```

To send the actual message, use the `sendto()` function, which accepts the socket file descriptor (`sender_fd`) from the previous step, the message (`ping` in the example below), and some flags and structs.
```c
// send ping
if ((status = sendto(sender_fd, ping, ping_len, 0, p->ai_addr, p->ai_addrlen)) == -1) {
    fprintf(stderr, "Error pinging host: %s\n", hosts[i]);
    perror("peer: sending");
}
```

Note that this is **not a broadcast**, this only sends to the one `host` from `getaddrinfo()`. If you want to send to multiple hosts, you will need to perform this setup for each host.
## Parallelism
By default, the `recvfrom()` call is a blocking call. This means that if you try to send and receive messages in the same thread, you risk a deadlock. To avoid this, I recommend one of the following two approaches.
1. If you provide `recvfrom()` with the `MSG_DONTWAIT` flag instead of `NULL`, it won't block. So whether there is a message or not, you can move on to the sending part of your code. Since, the function is not blocking, it's smart to put a short sleep, so that the CPU isn't blasting at 100% for no reason.
2. You can create multiple threads: one that sends messages on a loop, and others that each receive messages from one peer (written to one socket).
