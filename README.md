# TLS 1.3 + TCP Fast Open

1. [TCP Fast Open] added support to send data in the first packet of the TLS handshake.
2. [TLS 1.3] added support for 0-RTT,
allowing to send application data in the first packet of the TLS handshake.

which makes it possible to establish a encrypted connection and
send the application over TCP in one packet.

[TCP Fast Open]: https://tools.ietf.org/html/rfc7413
[TLS 1.3]: https://tools.ietf.org/html/rfc8446

## TL;DR;

### A simple use case

1. establish connection
2. send PING
3. receive PONG
4. close connection

### Results

Over a network with 10ms latency:

| Scenario                               | PING | PONG |
| -------------------------------------- | ----:| ----:|
| TLS 1.3 full handshake                 | 49ms | 59ms |
| TLS 1.3 0-RTT                          | 20ms | 52ms |
| TLS 1.3 full handshake + TCP Fast Open | 29ms | 39ms |
| TLS 1.3 0-RTT + TCP Fast Open          |  0ms | 32ms |

Over a network with 1ms latency:

| Scenario                               | PING | PONG |
| -------------------------------------- | ----:| ----:|
| TLS 1.3 full handshake                 | 13ms | 14ms |
| TLS 1.3 0-RTT                          |  2ms |  7ms |
| TLS 1.3 full handshake + TCP Fast Open | 11ms | 12ms |
| TLS 1.3 0-RTT + TCP Fast Open          |  0ms |  5ms |

* PING: duration until PING message is sent to the server
* PONG: duration until PONG message is received from the server

## TLS Connection

- TCP handshake
- TLS handshake
- Application data exchange
- TLS connection shutdown
- TCP connection shutdown

### TLS 1.3 - Full Handshake

A full TLS 1.3 handshake looks like:

    @startuml
    == TCP handshake ==
    c -> s: TCP SYN
    c <- s: TCP SYN+ACK
    c -> s: TCP ACK
    == TLS handshake ==
    c -> s: Client Hello
    c <- s: Server Hello, Change Cipher Spec, ...
    c -> s: Change Cipher Spec, Finished
    c -> s: PING
    c <- s: New Session Ticket
    c <- s: New Session Ticket
    c <- s: PONG
    == TLS shutdown ==
    c <- s: Alert: Close Notify
    c -> s: Alert: Close Notify
    == TCP shutdown ==
    c -> s: TCP FIN
    c <- s: TCP FIN
    @enduml

The communication changes direction 8 times, each change of direction
adds networks latency (here, 10ms):

    ... 0.000 ...
    c -> s: TCP SYN
    ... 0.010 ...
    c <- s: TCP SYN+ACK
    ... 0.020 ...
    c -> s: TCP ACK
    c -> s: Client Hello
    ... 0.038 ...
    c <- s: Server Hello, Change Cipher Spec, ...
    ... 0.049 ...
    c -> s: Change Cipher Spec, Finished
    c -> s: PING
    ... 0.059 ...
    c <- s: New Session Ticket
    c <- s: New Session Ticket
    c <- s: PONG
    c <- s: Alert: Close Notify
    ... 0.069 ...
    c -> s: Alert: Close Notify
    c -> s: TCP FIN
    ... 0.079 ...
    c <- s: TCP FIN

- a full `Server Hello` takes about 8ms longer than the normal network latency.

### TLS 1.3 - Session Resumption

TLS 1.3 supports a abbreveated handshake if

- client connected to the server previously and
- the previous connection was shutdown properly

With session resumption, the `Server Hello` is about 7-8ms faster.

...

### TLS 1.3 - 0-RTT data

TLS 1.3 supports the sending Application Data in the Client hello packet

- if a SSL session can be resumed
- server announced it supports early data

The packet flow:

    ... 0.000 ...
    c -> s: TCP SYN
    ... 0.010 ...
    c <- s: TCP SYN+ACK
    ... 0.020 ...
    c -> s: TCP ACK
    c -> s: Client Hello, Change Cipher Spec, PING
    ... 0.031 ...
    c <- s: Server Hello, Change Cipher Spec, Finished
    ... 0.042 ...
    c -> s: End of Early Data, Finished
    ... 0.052 ...
    c <- s: New Session Ticket
    c <- s: PONG
    c <- s: Alert: Close Notify
    ... 0.062 ...
    c -> s: Alert: Close Notify
    c -> s: TCP FIN
    ... 0.072 ...
    c <- s: TCP FIN

## TCP Fast Open

If

- TCP Fast Open is supported by the OS and
- enabled on both client and server and
- client established a connected to the server before

it can send data in the first packet of the TCP handshake:

    ... 0.000 ...
    c -> s: SYN + Client Data
    ... 0.010 ...
    c <- s: SYN+ACK
    ... 0.020 ...
    c -> s: ACK

which allows the server to send a response earlier.

### TLS 1.3 Full Handshake + TCP Fast Open

TCP Fast Open and TLS 1.3 can be combined

    ... 0.000 ...
    c -> s: SYN + Client-Hello
    ... 0.010 ...
    c <- s: SYN+ACK
    ... 0.017 ...
    c <- s: Server Hello, Change Cipher Spec, ...
    ... 0.019 ...
    c -> s: ACK
    ... 0.029 ...
    c -> s: Change Cipher Spec, Finished
    c -> s: PING
    ... 0.039 ...
    c <- s: New Session Ticket
    c <- s: New Session Ticket
    c <- s: PONG
    c <- s: Alert: Close Notify
    ... 0.049 ...
    c -> s: Alert: Close Notify
    c -> s: TCP FIN
    ... 0.059 ...
    c <- s: TCP FIN

### TLS 1.3 Session Resumption + TCP Fast Open

...

### TLS 1.3 0-RTT + TCP Fast Open

TCP Fast Open and TLS 1.3 can be combined

    ... 0.000 ...
    c -> s: SYN + Client-Hello, Change Cipher Spec, PING
    ... 0.010 ...
    c <- s: SYN+ACK
    ... 0.011 ...
    c <- s: Server Hello, Change Cipher Spec, Finished
    ... 0.019 ...
    c -> s: ACK
    ... 0.022 ...
    c -> s: End of Early Data, Finished
    ... 0.032 ...
    c <- s: New Session Ticket
    c <- s: PONG
    c <- s: Alert: Close Notify
    ... 0.042 ...
    c -> s: Alert: Close Notify
    c -> s: TCP FIN
    ... 0.052 ...
    c <- s: TCP FIN

# API usage

OpenSSL 1.1.1 added the necessary APIs to use TLS 1.3 session resumption
and 0-RTT.

## TLS Session Resumption

The client MUST cache the session tickets the server sends to allow reuse:

```c++
std::unique_ptr<SSL_SESSION, void (*)(SSL_SESSION *)> last_session(
    nullptr, &SSL_SESSION_free);

static int new_session_cb(SSL *s, SSL_SESSION *sess) {
  // store session in cache.
  last_session.reset(sess);

  return 1;
}

  //...
  // enable the session cache to allow session resumption
  SSL_CTX_set_session_cache_mode(
    ssl_ctx, SSL_SESS_CACHE_CLIENT | SSL_SESS_CACHE_NO_INTERNAL);
  SSL_CTX_sess_set_new_cb(ssl_ctx, new_session_cb);
  // ...
```

The `new_session_cb` will be called whenever the server sent a new
session (e.h. `New Session Ticket`). It can be used on the next SSL
connection to the same server.

```c++
  if (last_session) {
    SSL_set_session(ssl.get(), last_session.release());
  }
```

## TLS 1.3 0-RTT

If the current SSL connection has a resumed session and the server
announced it supports `early data`, the client can send data as part
of the handshake by using `SSL_write_early_data()` before `SSL_connect()`.

```c++
  if (SSL_get0_session(ssl)) != nullptr &&
      SSL_SESSION_get_max_early_data(SSL_get0_session(ssl)) > 0) {
    size_t written;
    auto ssl_res = SSL_write_early_data(ssl, transfer_buf.data(),
                                        transfer_buf.size(), &written);
    if (ssl_res != 1) {
      // handle error
    } else {
      // success
    }
  }

  // SSL_connect() ...
```

The TLS connection must properly signal a shutdown to make the session
resumable:

```c++
  // ...

  SSL_shutdown(ssl);
```

## TCP Fast Open

TCP Fast Open is supported on:

- Linux 3.x
- Windows 10
- FreeBSD 12
- MacOSX 10.11

### kernel side support

#### Linux

Linux requires enabling the server side support for TCP Fast Open
via `sysctl`.

See https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt

> `tcp_fastopen` - INTEGER
>   `0x01` - client (enabled by default)
>   `0x02` - server (disabled by default)

```sh
# current value
$ cat /proc/sys/net/ipv4/tcp_fastopen
1
# enable client and server
$ echo "3" | sudo tee /proc/sys/net/ipv4/tcp_fastopen
```

#### FreeBSD

On FreeBSD:

```sh
$ sysctl net.inet.tcp.fastopen.server_enable
0
$ sysctl net.inet.tcp.fastopen.client_enable
1
```

#### MacOS X

MacOS X has client and server support for TCP Fast Open enabled by default.

```sh
$ sysctl net.inet.tcp.fastopen
3
```

### server side support

If the kernel support is enabled, the server application can active
support for TCP Fast Open via a `setsockopt()`:

```c++
int on = 1;
setsockopt(server_sock, IPPROTO_TCP, TCP_FASTOPEN, &on, sizeof on);
```

*Note*: On MacOSX the socket must be in listening mode already
for the `setsockopt()` to succeed.

### client side support

Different API styles exist to enable sending application data
at connect time

- delay connect until first write
- use sendto()
- use new API

The `delay connect until first write` style allows easy integration
with existing socket abstractions like the one in OpenSSL.

#### delay connect until first write

Linux and FreeBSD allow to delay the connect until the first write
and use the existing socket APIs by enable the socket option with
`setsockopt()`.

On Linux:

```c++
int on = 1;
setsockopt(client_sock, IPPROTO_TCP, TCP_FASTOPEN_CONNECT,
  &on, sizeof on);
```

On MacOSX and FreeBSD:

```c++
int on = 1;
setsockopt(client_sock, IPPROTO_TCP, TCP_FASTOPEN,
  &on, sizeof on);
```

The `connect()` afterwards will succeed and the `send()` will return
the errors like `EINPROGRESS` that would otherwise happen with `connect()`.

```c++
// no op
connect(client_sock, addr, addr_len);

// SYN + data.
send(client_sock, data, data_len);
```

#### use sendto() to connect with data

On Linux `sendto()` can be used to established a connection and send data
in the first packet by setting the `MSG_FASTOPEN` flag:

```c++
sendto(sock, data, datalen, MSG_FASTOPEN, addr, addrlen);
```

It replaces the `connect()` + `send()`.

#### use a new API to connect with data

- Windows has `ConnectEx()`
- MacOSX has `connectx()`

# Examples

## Build Requirements

- cmake 3.4 or later
- C++14 capable compiler
- Openssl 1.1.1
- Operating Systems
  - Linux 4.11 or later
  - FreeBSD 12 or later
  - MacOSX 10.11 or later

## building

    $ cmake
    $ make

## running

1. Start server in one terminal

       $ ./src/tls13_ping_pong_server 127.0.0.1 3308

2. run the client in another terminal

       $ ./src/tls13_ping_pong_client 127.0.0.1 3308

## Tracing packets

Start server as before, but add tcpdump:

    $ sudo tcpdump -w tls13.pcap -i lo 'port 3308'
    $ ./src/tls13_ping_pong_client 127.0.0.1 3308
    $ wireshark tls13.pcap

### Let wireshark decrypt the TLS packets automatically

Wireshark 3.x.

    $ sudo tcpdump -w tls13.pcap -i lo 'port 3308'
    $ SSLKEYLOGFILE=keys.txt ./src/tls13_ping_pong_client 127.0.0.1 3308
    $ editcap --inject-secrets tls,keys.txt tls13.pcap tls13-with-keys-dsb.pcapng
    $ wireshark tls13-with-keys-dsb.pcapng


### Adding latency

To simulate real-life network delays all packets from and to port 3308 where delayed by 10ms
by using the [netem] network emulator of [tc].

    $ sudo tc qdisc add dev lo root handle 1: prio
    $ sudo tc qdisc add dev lo parent  1:3 handle 30: netem delay 10ms
    $ sudo tc filter add dev lo parent 1:0 protocol ip u32 match ip dport 3308 0xffff flowid 1:3
    $ sudo tc filter add dev lo parent 1:0 protocol ip u32 match ip sport 3308 0xffff flowid 1:3

[netem]: http://man7.org/linux/man-pages/man8/tc-netem.8.html
[tc]: http://man7.org/linux/man-pages/man8/tc.8.html
