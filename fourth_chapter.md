# Fourth chapter

## Sockets and Socket programming
### 1. Socket API

In linux, sockets are the pipes or the software wires that are used for the exchange of the data using the TCP/IP protocol stack.

The server program opens a sockets, waits for someone to connect to it. The socket can be created to communicate over the TCP or the UDP protocol and the underlying networking layer can be IPv4 or IPv6.

In the Linux systems programming, the TCP protocol is denoted by a macro called **SOCK_STREAM** and the UDP protocol is denoted by a macro called **SOCK_DGRAM**. Either of the above macros are passed as the second argument to the **socket** API.

The protocol IPv4 is denoted by **AF_INET** and the protocol IPv6 is denoted by **AF_INET6**. Either of these macros are passed as the first argument to the **socket** API.

The socket API usually takes the following form.

    socket (Address family, transport protocol, IP protocol);

for a TCP socket:

    socket(AF_INET, SOCK_STREAM, 0);

for a UDP socket:

    socket(AF_INET, SOCK_DGRAM, 0);

The return value of the **socket** API is the actual socket connection. The below code snippet will give an example of the usage:

    int sock;
    
    // create a TCP socket
    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        perror("failed to open a socket");
        return -1;
    }
    
    printf("the socket address is %d\n", sock);


The returned socket address is then used as the communication channel.

To create a server, we must use a **bind** system call to tell others that we are running at some port and ip. Like naming the connection and allowing others to talk with us by referring to the name.


    bind(Socket Address, Server Address Structure, length of the Server address structure);
    

    ret = bind(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr));
    

It is advised to perform the `setsockopt` call with `SO_REUSEADDR` option after a call to the `socket`. This is described nicely [here](http://www.unixguide.net/network/socketfaq/4.5.shtml).

In brief, if you stopped the server for some time and started back again quickly, the `bind` may fail. This is because the OS still contain the context associated to your server (ip and port) and does not allow others to connect with the same information. The context gets cleared with the `setsockopt` option before the bind.

The setsockopt option would look like the below.

    int setsockopt(int sock_fd, int level, int optname, const void *optval, socklen_t optlen);

The basic and most common usage of the `setsockopt` is like the below:

    int reuse_addr = 1;   // turn on the reuse address operation in the OS
    
    ret = setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &reuse_addr, sizeof(reuse_addr));

A server must register to the OS that it is ready to perform accepting the connections by using the `listen` system call. It takes the below form:

    int listen(int sock_fd, int backlog);
    
The sock_fd is the socket address returned by the `socket` system call. The backlog defines the number of pending connections on to the socket. If the backlog connections cross the limit of `backlog`, the client will get a connection refused error. The usual call to the listen for an in-system and embedded server would be as follows:

    ret = listen(sock, 10);     // server will only perform replies to a max of 10 clients

The `accept` system call is followed after the call to the `listen` system call.

The `accept` system call takes the below form:

    int accept(int sock_fd, struct sockaddr *addr, socklen_t *addrlen);
    
The sock_fd is the socket address returned by the `socket` system call. the addr argument is filled by the OS and gives us the address of the neighbor. the addrlen is also filled by the OS and gives us the type of the data structure that the second argument contain. Such as if the length is of size  `struct sockaddr_in` then the address is of the IPv4 type and if its of size `struct sockaddr_in6` then the address is of the IPv6 type.

The `accept` function most commonly can be written as:

    struct sockaddr_in remote;
    socklen_t remote_len;
    
    ret = accept(sock, (struct sockaddr *)&remote_addr, &remote_len);

In case of a client, we do not have to call the `bind`, `listen` and `accept` system calls but call the `connect` system call.

The `connect` system call takes the following form:

    int connect(int sock_fd, const struct sockaddr *addr, socklen_t addrlen);
    
The `connect` system call allows the client to connect to a peer defined in the `addr` argument and the peer's length in `addrlen` argument.

The `connect` system call most commonly takes the following form:

    char server_addr[] = "127.0.0.1"
    int server_port = 45454;
    
    struct sockaddr_in server = {
        .family                 = AF_INET,
        .sin_addr.s_addr        = inet_addr(server_addr),
        .sin_port               = htons(server_port),
    };
    
    ret = connect(sock_fd, (struct sockaddr *)&server, sizeof(server));

    
The below sample programs describe about the basic server and client programs. The server programs creates the TCP IPv4 socket, sets up the socket option `SO_REUSEADDR`, binds, adds a backlog of 10 connections and waits on the accept system call. The server ip and port are given as the command line arguments.


The accept returns a socket that got connected to this server. The below program simply prints the socket address onto the screen and stops the program.


```
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define SAMPLE_SERVER_CONN 10

int main(int argc, char **argv)
{
    int ret;
    int sock;
    int conn;
    int set_reuse = 1;
    struct sockaddr_in remote;
    socklen_t remote_len;

    if (argc != 3) {
        fprintf(stderr, "%s [ip] [port]\n", argv[0]);
        return -1;
    }

    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        perror("failed to socket\n");
        return -1;
    }

    ret = setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &set_reuse, sizeof(set_reuse));
    if (ret < 0) {
        perror("failed to setsockopt\n");
        close(sock);
        return -1;
    }

    remote.sin_family = AF_INET;
    remote.sin_addr.s_addr = inet_addr(argv[1]);
    remote.sin_port = htons(atoi(argv[2]));

    ret = bind(sock, (struct sockaddr *)&remote, sizeof(remote));
    if (ret < 0) {
        perror("failed to bind\n");
        close(sock);
        return -1;
    }

    ret = listen(sock, SAMPLE_SERVER_CONN);
    if (ret < 0) {
        perror("failed to listen\n");
        close(sock);
        return -1;
    }

    remote_len = sizeof(struct sockaddr_in);

    conn = accept(sock, (struct sockaddr *)&remote, &remote_len);
    if (conn < 0) {
        perror("failed to accept\n");
        close(sock);
        return -1;
    }

    printf("new client %d\n", conn);

    close(conn);

    return 0;
}
```

**Example: Sample server program
**

The client program is described below. It creates a TCP IPv4 socket, connect to the server, on a successful connection, it prints the connection result and stops the program.

```
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define SAMPLE_SERVER_CONN 10

int main(int argc, char **argv)
{
    int ret;
    int sock;
    struct sockaddr_in remote;
    socklen_t remote_len;

    if (argc != 3) {
        fprintf(stderr, "%s [ip] [port]\n", argv[0]);
        return -1;
    }

    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        perror("failed to socket\n");
        return -1;
    }

    remote.sin_family = AF_INET;
    remote.sin_addr.s_addr = inet_addr(argv[1]);
    remote.sin_port = htons(atoi(argv[2]));

    remote_len = sizeof(struct sockaddr_in);

    ret = connect(sock, (struct sockaddr *)&remote, remote_len);
    if (ret < 0) {
        perror("failed to accept\n");
        close(sock);
        return -1;
    }

    printf("connect success %d\n", ret);

    close(sock);

    return 0;
}
```

**Example: Sample client program
**


### 2. Sending and Receiving over the Sockets

We have seen the server and client connect to each other over sockets. Now that connection is established, the rest of the steps are the data-transfer. The data-transfers are performed using the simple system calls, `recv`, `send`, `recvfrom` and `sendto`.

The `recv` system call receives the data over the TCP socket, i.e. the socket is created with **SOCK_STREAM** option. The `recvfrom` system call receives the data over the UDP socket, i.e. the socket is created with **SOCK_DGRAM** option. 

The `send` system call sends the data over the TCP socket  and the `sendto` system call sends the data over the UDP socket.

The `recv` system call takes the following form:

    ssize_t recv(int sock_fd, void *buf, size_t len, int flags);
    
The `recv` function receives data from the sock_fd into the buf of length len. The options of `recv` are specified in the flags argument. Usually the flags are specified as 0. However, for a non blocking mode of socket operation **MSG_DONTWAIT** option is used.

The example `recv`:

    recv_len = recv(sock, buf, sizeof(buf), 0);
    
The `recv_len` will return the length of the bytes received. `recv_len` is 0 or less than 0, meaning that the other end has closed the connection and we must close the connection. Otherwise, the `recv` function call will be restarted by the OS repeatedly causing heavy CPU loads.

The `recvfrom` system call is much similar to the `recv` and takes the following form:

    ssize_t recvfrom(int sock_fd, void *buf, size_t len, int flags, struct sockaddr *addr, socklen_t *addrlen);
    
The `recvfrom` is basically `recv` + `accept` for the UDP. The address of the sender and the length are notified in the addr and addrlen arguments. The rest of the arguments are same.

The example `recvfrom`:

    struct sockaddr_in remote;
    socklen_t remote_len = sizeof(remote);
    
    recv_len = recvfrom(sock, buf, sizeof(buf), 0, (struct sockaddr *)&remote, &remote_len);
    
The `recv_len` will return the length of bytes received. The address of the sender is filled in to the remote and the length in the remote_len.

The `send` system call takes the following form:

    ssize_t send(int sock_fd, const void *buf, size_t len, int flags);
    
The `send` will return the length of bytes sent over the connection. The buffer buf of length len is sent over the connection. The flags are similar to that of `recv` and most commonly used flag is the `MSG_DONTWAIT`.

The example `send`:

    sent_bytes = send(sock, buf, buf_len, 0);
    
The `sent_bytes` less than 0 means an error.

The `sendto` system call takes the following form:

    ssize_t sendto(int sock_fd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t dest_len);
    
The `sendto` is same as `send` with address.

The client program now performs a send system call to send "Hello" message to the server. The server program then receives over a recv system call to receive the message and prints it on the console.
