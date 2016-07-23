Simple library for sending file descriptors to other processes over
sockets.

# Introduction

The Unix socket API provides facilities for sending file descriptors
over domain sockets. This allows for some interesting possibilities, but
like much of the sockets API, the interface to this functionality is...
arcane.

This library provides a wrapper around this feature that makes it
simpler to work with. It doesn't provide the full range of options; you
can only do stream-oriented connections, for example.

## Examples

```c
/* client.c */
#include <stdio.h>
#include <string.h>
#include "fdmsg.h"

int main(void) {
    /* create an FD buffer big enough for one fd, and put our
     * stdout in it. The FDMSG_* macros are necessary due to
     * an implementation detail; see the comments in the header
     * file for a full explanation: */
    char fdbuf[FDMSG_BUFSZ(1)];
    int *fds = FDMSG_BUFDATA(&fdbuf[0]);
    char *data = "Have fun with my stdout!";
    int sock;
    fds[0] = 1;

    /* create the socket and connect: */
    sock = fdmsg_socket();
    if(sock < 0) {
        perror("fdmsg_socket()");
        return 1;
    }
    if(fdmsg_connect(sock, "/tmp/my-socket") < 0) {
        perror("fdmsg_connect()");
        return 1;
    }

    /* send some bytes and the file descriptor(s): */
    fdmsg_send(sock, data, strlen(data), fdbuf, 1);


    /* We don't even have to wait for the server; it's got a direct
     * connection to our stdout now. */
    return 0;
}
```

```c
/* server.c */
#include <stdio.h>
#include <unistd.h>
#include "fdmsg.h"

int main(void) {
    char fdbuf[FDMSG_BUFSZ(1)];
    char data[4096];
    int sock;
    int connfd;
    ssize_t data_count;

    /* create the socket and bind: */
    sock = fdmsg_socket();
    if(sock < 0) {
        perror("fdmsg_socket()");
        return 1;
    }
    if(fdmsg_bind(sock, "/tmp/my-socket") < 0) {
        perror("fdmsg_bind()");
        return 1;
    }

    /* This is standard socket API stuff; listen and accept: */
    if(listen(sock, SOMAXCONN) < 0) {
        perror("listen()");
        return 1;
    }
    connfd = accept(sock, NULL, NULL);
    if(connfd < 0) {
        perror("accept()");
        return 1;
    }

    /* recieve some bytes and file descriptors(s): */
    data_count = fdmsg_recv(connfd, data, sizeof data, fdbuf, 1);

    /* send the data down the recieved file descriptor: */
    write(FDMSG_BUFDATA(fdbuf)[0], data, data_count);

    return 0;
}
```

# License

ISC (simple permissive); see COPYING.
