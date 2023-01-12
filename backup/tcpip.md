# TCP/IP - "Reliable" component on Unreliable network

The digital infrastructure is so successful that it is even considered extremely stable. But for us ([distributed system] developers), it takes a lot effort. Human being's reliable digital dream is based on unreliable network.

In this article, I will show you part of the "solution" to unreliable network. Article structure is as following:

* What is TCP/IP?
* What does TCP/IP guarantee for distributed systems?
* How does TCP/IP work?
* Why is network still unreliable?

### What is TCP/IP?

![data flow](https://upload.wikimedia.org/wikipedia/commons/thumb/c/c4/IP_stack_connections.svg/525px-IP_stack_connections.svg.png)

TCP/IP (**T**ransmission **C**ontrol **P**rotocol/**I**nternet **P**rotocol) is a suite of communication protocols used to interconnect network devices on the internet.

In *Transport* layer, TCP provides **point-to-point connection-oriented reliable streaming** abstraction for application programs.

* **point-to-point**: a host computer is regarded as an end system; the application process with a port in a host computer is regarded as a point => a TCP connection defines: the communication channel between two applications with port in different end system.
* **connection-oriented**: one-to-one communication; before connection established, two end system need to do 3-Way handshake to establish connection to ensure communal arguments when transferring data and the abilities to send or receive.
* **reliable streaming**: a message is assembled into smaller packets/segments before they are then transmitted over the internet and reassembled in the right order at the destination address.

In *Network layer*, IP provides **end-to-end** abstraction, and is responsible for **addressing** and **route**. (only consider IPv4)

* **end-to-end**: a host computer to another host computer.
* **addressing** & **route**: IPv4 is divided into *network* number and *host* number; according to IP address, a device (end system, switch, router) must first find a subnetwork then identifying the destination end system => IP tells it the direction to destination.

### What does TCP/IP guarantee for distributed systems?

TCP/IP is responsible for providing data transmission services for networked end systems.

Processes in *Application* layer just need to invoke network related system calls to communicate to each other. The communication methods encapsulated as Socket. They are all in between *Application* layer and *Transport* layer, so the protocol in *Transport* layer is very important to programmers.

As mentioned before, TCP is the most popular protocol in *Transport* layer. It provides **point-to-point connection-oriented reliable streaming** to simplify the stuff. Applications do not need to care about cumbersome details (such as *unordered slices*, *duplicate slices*) in network transmission. All an application needs to do is connect to another destination end system and tell OS provided Socket to send data to destination. The read and write operations are the same with normal file operations.

TCP/IP guarantees **reliable streaming** for distributed systems (applications). That is: what application reads is absolutely "right" response (not consider *Byzantine fault*). It might lose some data behind because of network latency, but must be right response from right end system (another host computer in an established TCP connection). TCP also tries to reduce the impact of network instability (like congestion control), but it is a drop in a bucket.

### How does TCP/IP work?

In this section, I will show you how TCP/IP works in Client-Server scheme.

##### Application Layer

In *Application* layer, there is a **point-to-point** connection between client process and server process.

Socket is used for network programming, and is used as normal file descriptor. For example, there are two simple programs:

* [echo_server.c](https://gist.github.com/huang-feiyu/e56d160fc84c6fdca7700715064d1de3)

  1. `socket`: create an end point of TCP connection

  2. `bind`: bind a name{IPv4, port} to thej socket

  3. `listen`: listen at the socket
  4. `accept`: wait for accepting a connection on the socket
  5. `recv` & `send`: read from client_fd & write to client_fd
  6. `close`: close local client file descriptor

* [client.c](https://gist.github.com/huang-feiyu/c3b75e858d9f4095d69635bef9c9f71e)

  1. `socket`: create an end point of TCP connection
  2. `connect`: initiate a connection to server{IPv4, port}
  3. `recv` & `send`: read from server_fd & write to server_fd
  4. `close`: close local client file descriptor


![Client Server](https://media.geeksforgeeks.org/wp-content/uploads/20220330131350/StatediagramforserverandclientmodelofSocketdrawio2-448x660.png)

Using socket for network programming seems very easy and handy. Normal socket is provided by **OS kernel** -- OS kernel provides **net syscall** for programmers. To fully understand how it works, we need to dive into the water.

##### Transport Layer

> Based on [Sponge](https://github.com/huang-feiyu/sponge) TCP socket library.

As mentioned before, TCP is a protocol in *Transport* layer, which provides **point-to-point connection-oriented reliable streaming**.

A TCP socket contains a TCP connection (TCP state machine), and is responsible for **read IPv4 datagram from Internet** and **write IPv4 datagram to Internet**. The architecture of TCP socket is as following figure:

* A TCP socket involves two threads:<br/>**Owner main**: one for the owner (application) so that it can call `connect`, `listen`, `accept`, `read`, `write`;<br/>**TCP main**: one for the TCP connection to read and parse datagrams from the wire (OS network stack), state transition...
* **Owner** & **TCP Socket** ["socket" -> thread_data]: before `read` & `write`, owner must establish a connection between local process and foreign process<br/>Owner calls `read`: Read incoming (from foreign application) bytes *reassembled* by the TCP Connection; TCP Connection reads inbound byte stream from *receiver* and writes to to local application<br/>Owner calls `write`: Write outcoming (to foreign application) bytes to others; TCP Connection reads outbound bytes from local application and writes to *sender*
* **TCP Socket** & **Network stack** [others -> datagram_adapter]:<br/>Data coming from internet: TCP Connection reads datagram from internet and retrieves segment from it, then writes to *receiver* and updates *sender*<br/>Data sending to internet: Network stack reads segment from *sender* (already emplaced in queue) and performs IP layer encapsulation to datagram

![Sponge Architecture](https://user-images.githubusercontent.com/70138429/212103527-1f85c8b7-4c2e-4512-902c-0df568f72641.png)

Behind reads and writes, there is an event loop to help handling everything -- that is **TCP main**. TCP main uses *event loop* to handle with asynchronous I/O. In our case, TCP Connection needs to poll two file descriptors (thread_data and datagram_adapter) so that Socket can handle incoming bytes and outcoming bytes from both *local* and *network*. The poll scheme is as below:

```python
# Rule(fd, direction, callback, interest, cancle)
# - fd:        file descriptor
# - direction: to read to the fd or write to the fd (POLLIN or POLLOUT)
# - callback:  handle with the data coming in or coming out
# - interest:  whether this rule is "alive" (such as TCPConnection.active())
# - cancle:    when rule is "dead", call cancle to deal with the left things
for each rule:
    if rule.interest():
        poll_fds.add(rule.fd, rule.direction)

if poll_fds.empty():
    return

poll(poll_fds.fds, poll_fds.size, timeout)
for each result:
    if result == dead:
        result.rule.cancle()
    if poll_ready:
        result.rule.callback()
```

Diving into TCP connection, it is mainly about how the TCP finite state machine works. I will show you later.

![TCP Socket](https://camo.githubusercontent.com/db33fda8acf71ca715d9a60ba8a4af61fabacf85cbb983b88bb06d667cc53920/68747470733a2f2f6c7a782d6669677572652d6265642e6f62732e6475616c737461636b2e636e2d6e6f7274682d342e6d79687561776569636c6f75642e636f6d2f4669677572656265642f3230323230313232323235363234332e706e67)

In summary, a TCP Socket maintains TCP finite state machine in background and provides public API for applications. When reads or writes signals to "socket" or network, TCP Connection must do something to respond to the events.

---

More specifically, TCP also try to reduce the impact of network problems on the application, such as *Reliable transmission*, *Error detection*, *Flow control*, *Congestion control*.

![TCP header](https://camo.githubusercontent.com/798366de357c7b7e64129dafcec03f6428082bd00dbacd69573396d29cafeb72/68747470733a2f2f66696265726269742e636f6d2e74772f77702d636f6e74656e742f75706c6f6164732f323031332f31322f5443502d7365676d656e742e706e67)

* *Reliable transmission*

Before the famous *3-Way handshake* and *4-Way goodbye*, we must know how TCP guarantees in-order byte stream. It is mostly about *receiver* in TCP Connection. TCP receiver is responsible for taking over data from internet and reassembles it according to `seqno` and `isn`.

In order to transmit data reliably, the end points must establish connection. That is what *3-Way handshake* does.

* handshake #1: client sends SYN to server
* handshake #2: server receives SYN from client; server sends {SYN, ACK} to client
* handshake #3: client receives {SYN, ACK} from server and establishes connection; client sends {ACK, [data]} to server
* server receives {ACK, [data]} and establishes connection

After establishing connection, we need also handle with shutdown. That is what *4-Way goodbye* does.

* goodbye #1: client is ready to close local write and sends FIN to server
* goodbye #2: server receives FIN from client; server sends ACK to client
* client receives ACK from server, if there is an ACK to FIN, really close local write
* goodbye #3: server is ready to close local write and sends FIN to client
* goodbye #4: client receives FIN from sever; client send ACK to server
* server receives ACK from client, if there is an ACK to FIN, really close local write; client waits for a while to ensure server receives ACK

(For more details, read [sponge - lab4_doc](https://github.com/huang-feiyu/sponge/blob/main/writeups/lab4.md))

There is also timeout retransmission, but it is easy in our case -- just re-transmit when timer is out. 

![TCP state transition](https://user-images.githubusercontent.com/70138429/210497471-3a873eb8-f394-4642-ad8f-e7b0dccbc08b.png)

* *Error detection*

In TCP, error detection is very simple.

```
for (size_t i = 0; i < data.size(); i++) {
    uint16_t val = uint8_t(data[i]);
    if (not _parity) {
        val <<= 8;
    }
    _sum += val;
    _parity = !_parity;
}
```

1. Fill up TCP header besides `checksum`
2. Use the code above to compute 16-bit (2 bytes) `checksum` of header and then payload
3. Update `checksum` of the header

* *Flow control*

To ensure that receiver will not be overwhelmed with data, sender needs to control the speed it sends data.

A receiver has the limit of buffer size (reassembled data + un-assembled data + idle). The window size is capacity minus the size of reassemble data.

The sender always updates local maintained foreign window size according to segment's `win`. When sending data, it will only send no more than `win` data (update local `win` by the length of bytes sent). When `win` is empty/zero, *sender* needs also to response to ACK message via empty segment.

In our case, flow control is very easy -- sender sends no more than what affordable bytes of receiver; under that condition, sender always sends as much as possible.

* *Congestion control*

To prevent any node from being overloaded with excessive packets, TCP needs a mechanism that limits the flow of packets at each node of the network.

In general, there are 3 phases in real world: slow start, congestion avoidance and congestion detection. But in our case, it is also easy -- there is only congestion detection.

When an ACK of segment is timeout, sender thinks there is a congestion so it reset timer and double timeout limit (of course, sender retransmit the youngest segment). When an ACK is received,  reset timeout limit.

---

In summary, TCP works as state machine and uses a lot of mechanisms to ensure realiability. 

### Why is network still unreliable?

TCP/IP does its best to make the network "reliable", but it is a drop in the bucket. Reliability is not just right data (although TCP cannot provide absolutely right data, we need to check data in *Application* layer, too), but also low latency. But in real world, **delay is unbounded**.

The main reason why network is so unreliable is that market chooses ***asynchronous packet networks*** -- there is no reserved network resources for a connection. So almost everything of network needs to queue:

* Switch queues packet to the same destination, if overload, it will randomly lose packets
* TCP flow control also has an impact on delay
* Queue in local/foreign machine for time slice

If we try to make everything roughly the best, it costs a lot. Therefore, the complexity of network is remained to be solved by distributed system developers. It provides reliable byte stream after all.
