---
weight: 1
bookFlatSection: true
title: "6 Client-Server Background"
---

# 6 Client-Server Background

クライアント-サーバの世界なのです。ネットワーク上のあらゆることが、クライアント・プロセスとサーバ・プロセスとの対話、またはその逆を扱っています。たとえば、`telnet` を考えてみよう。ポート 23 のリモートホストに `telnet` で接続すると（クライアント）、そのホスト上のプログラム（telnetd と呼ばれるサーバ）が起動します。このプログラムは、送られてきた `telnet` 接続を処理し、ログインプロンプトを表示するなどの設定を行います。

<figure>
  <img
  src="/bgnet/images/cs.svg"
  alt="[Client-Server Interaction Diagram]">
  <figcaption>クライアント-サーバの相互作用</figcaption>
</figure>

クライアントとサーバ間の情報のやりとりは、上の図のようにまとめられます。

クライアントとサーバのペアは、`SOCK_STREAM`、`SOCK_DGRAM`、その他（同じことを話している限り）何でも話すことができることに注意してください。クライアントとサーバのペアの良い例としては、`telnet`/`telnetd`、`ftp`/`ftpd`、`Firefox`/`Apache` などがあります。`ftp` を使うときはいつも、リモートプログラム `ftpd` があなたにサービスを提供します。

多くの場合、1つのマシンには1つのサーバしかなく、そのサーバは `fork()` を使用して複数のクライアントを処理します。基本的なルーチンは、サーバが接続を待ち、それを `accept()` し、それを処理するために子プロセスを `fork()` する、というものです。これが、次の節で紹介するサンプルサーバが行っていることです。


## 6.1 A Simple Stream Server

このサーバがすることは、ストリーム接続で `Hello, world!` という文字列を送り出すだけです。このサーバをテストするために必要なことは、あるウィンドウでこのサーバを実行し、別のウィンドウからこのサーバに telnet でアクセスすることだけです。

```
$ telnet remotehostname 3490
```

ここで、`remotehostname` は実行するマシンの名前です。

[サーバコード](https://beej.us/guide/bgnet/examples/server.c)

```{.c .numberLines}
/*
** server.c -- a stream socket server demo
*/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h>
#include <arpa/inet.h>
#include <sys/wait.h>
#include <signal.h>

#define PORT "3490"  // the port users will be connecting to

#define BACKLOG 10   // how many pending connections queue will hold

void sigchld_handler(int s)
{
    // waitpid() might overwrite errno, so we save and restore it:
    int saved_errno = errno;

    while(waitpid(-1, NULL, WNOHANG) > 0);

    errno = saved_errno;
}


// get sockaddr, IPv4 or IPv6:
void *get_in_addr(struct sockaddr *sa)
{
    if (sa->sa_family == AF_INET) {
        return &(((struct sockaddr_in*)sa)->sin_addr);
    }

    return &(((struct sockaddr_in6*)sa)->sin6_addr);
}

int main(void)
{
    int sockfd, new_fd;  // listen on sock_fd, new connection on new_fd
    struct addrinfo hints, *servinfo, *p;
    struct sockaddr_storage their_addr; // connector's address information
    socklen_t sin_size;
    struct sigaction sa;
    int yes=1;
    char s[INET6_ADDRSTRLEN];
    int rv;

    memset(&hints, 0, sizeof hints);
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags = AI_PASSIVE; // use my IP

    if ((rv = getaddrinfo(NULL, PORT, &hints, &servinfo)) != 0) {
        fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(rv));
        return 1;
    }

    // loop through all the results and bind to the first we can
    for(p = servinfo; p != NULL; p = p->ai_next) {
        if ((sockfd = socket(p->ai_family, p->ai_socktype,
                p->ai_protocol)) == -1) {
            perror("server: socket");
            continue;
        }

        if (setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &yes,
                sizeof(int)) == -1) {
            perror("setsockopt");
            exit(1);
        }

        if (bind(sockfd, p->ai_addr, p->ai_addrlen) == -1) {
            close(sockfd);
            perror("server: bind");
            continue;
        }

        break;
    }

    freeaddrinfo(servinfo); // all done with this structure

    if (p == NULL)  {
        fprintf(stderr, "server: failed to bind\n");
        exit(1);
    }

    if (listen(sockfd, BACKLOG) == -1) {
        perror("listen");
        exit(1);
    }

    sa.sa_handler = sigchld_handler; // reap all dead processes
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;
    if (sigaction(SIGCHLD, &sa, NULL) == -1) {
        perror("sigaction");
        exit(1);
    }

    printf("server: waiting for connections...\n");

    while(1) {  // main accept() loop
        sin_size = sizeof their_addr;
        new_fd = accept(sockfd, (struct sockaddr *)&their_addr, &sin_size);
        if (new_fd == -1) {
            perror("accept");
            continue;
        }

        inet_ntop(their_addr.ss_family,
            get_in_addr((struct sockaddr *)&their_addr),
            s, sizeof s);
        printf("server: got connection from %s\n", s);

        if (!fork()) { // this is the child process
            close(sockfd); // child doesn't need the listener
            if (send(new_fd, "Hello, world!", 13, 0) == -1)
                perror("send");
            close(new_fd);
            exit(0);
        }
        close(new_fd);  // parent doesn't need this
    }

    return 0;
}
```

一応、構文的にわかりやすいように、1つの大きな `main()` 関数にまとめてあります。もし、その方が良いと思われるなら、自由に小さな関数に分割してください。

（また、この `sigaction()` 全体は、あなたにとって新しいものかもしれません---それは大丈夫です。このコードは、`fork()` された子プロセスが終了するときに現れるゾンビプロセスを刈り取る役割を担っているのです。ゾンビをたくさん作ってそれを刈り取らないと、システム管理者が怒りますよ。）

このサーバからデータを取得するには、次の節に記載されているクライアントを使用します。


## 6.2 A Simple Stream Client

こいつはサーバよりもっと簡単です。このクライアントがすることはコマンドラインで指定したホスト、ポート 3490 に接続するだけです。サーバが送信する文字列を取得します。

[クライアントソース](https://beej.us/guide/bgnet/examples/client.c)。

```{.c .numberLines}
/*
** client.c -- a stream socket client demo
*/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <netdb.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <sys/socket.h>

#include <arpa/inet.h>

#define PORT "3490" // the port client will be connecting to

#define MAXDATASIZE 100 // max number of bytes we can get at once

// get sockaddr, IPv4 or IPv6:
void *get_in_addr(struct sockaddr *sa)
{
    if (sa->sa_family == AF_INET) {
        return &(((struct sockaddr_in*)sa)->sin_addr);
    }

    return &(((struct sockaddr_in6*)sa)->sin6_addr);
}

int main(int argc, char *argv[])
{
    int sockfd, numbytes;
    char buf[MAXDATASIZE];
    struct addrinfo hints, *servinfo, *p;
    int rv;
    char s[INET6_ADDRSTRLEN];

    if (argc != 2) {
        fprintf(stderr,"usage: client hostname\n");
        exit(1);
    }

    memset(&hints, 0, sizeof hints);
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;

    if ((rv = getaddrinfo(argv[1], PORT, &hints, &servinfo)) != 0) {
        fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(rv));
        return 1;
    }

    // loop through all the results and connect to the first we can
    for(p = servinfo; p != NULL; p = p->ai_next) {
        if ((sockfd = socket(p->ai_family, p->ai_socktype,
                p->ai_protocol)) == -1) {
            perror("client: socket");
            continue;
        }

        if (connect(sockfd, p->ai_addr, p->ai_addrlen) == -1) {
            close(sockfd);
            perror("client: connect");
            continue;
        }

        break;
    }

    if (p == NULL) {
        fprintf(stderr, "client: failed to connect\n");
        return 2;
    }

    inet_ntop(p->ai_family, get_in_addr((struct sockaddr *)p->ai_addr),
            s, sizeof s);
    printf("client: connecting to %s\n", s);

    freeaddrinfo(servinfo); // all done with this structure

    if ((numbytes = recv(sockfd, buf, MAXDATASIZE-1, 0)) == -1) {
        perror("recv");
        exit(1);
    }

    buf[numbytes] = '\0';

    printf("client: received '%s'\n",buf);

    close(sockfd);

    return 0;
}
```

クライアントを実行する前にサーバを実行しない場合、`connect()` は "Connection refused" を返すことに注意してください。非常に便利です。


## 6.3 Datagram Sockets {#datagram}

UDP データグラムソケットの基本は、上記の [5.8 sendto() and recvfrom()](docs/system-calls-or-bust/#sendtorecv) ですでに説明しましたので、ここでは `talker.c` と `listener.c` という2つのサンプルプログラムのみを紹介します。

`listener` は、ポート 4950 で入ってくるパケットを待つマシンに座っています。`talker` は、指定されたマシンのそのポートに、ユーザがコマンドラインに入力したものを含むパケットを送信します。

データグラムソケットはコネクションレス型であり、パケットを無慈悲に発射するだけなので、クライアントとサーバには IPv6 を使用するように指示することにしています。こうすることで、サーバが IPv6 でリッスンしていて、クライアントが IPv4 で送信するような状況を避けることができます。（接続された TCP ストリームソケットの世界では、まだ不一致があるかもしれませんが、一方のアドレスファミリーの `connect()` でエラーが発生すると、他方のアドレスファミリーの再試行が行われます。）

[listener ソースコード](https://beej.us/guide/bgnet/examples/listener.c)

```{.c .numberLines}
/*
** listener.c -- a datagram sockets "server" demo
*/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>

#define MYPORT "4950"	// the port users will be connecting to

#define MAXBUFLEN 100

// get sockaddr, IPv4 or IPv6:
void *get_in_addr(struct sockaddr *sa)
{
	if (sa->sa_family == AF_INET) {
		return &(((struct sockaddr_in*)sa)->sin_addr);
	}

	return &(((struct sockaddr_in6*)sa)->sin6_addr);
}

int main(void)
{
	int sockfd;
	struct addrinfo hints, *servinfo, *p;
	int rv;
	int numbytes;
	struct sockaddr_storage their_addr;
	char buf[MAXBUFLEN];
	socklen_t addr_len;
	char s[INET6_ADDRSTRLEN];

	memset(&hints, 0, sizeof hints);
	hints.ai_family = AF_INET6; // set to AF_INET to use IPv4
	hints.ai_socktype = SOCK_DGRAM;
	hints.ai_flags = AI_PASSIVE; // use my IP

	if ((rv = getaddrinfo(NULL, MYPORT, &hints, &servinfo)) != 0) {
		fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(rv));
		return 1;
	}

	// loop through all the results and bind to the first we can
	for(p = servinfo; p != NULL; p = p->ai_next) {
		if ((sockfd = socket(p->ai_family, p->ai_socktype,
				p->ai_protocol)) == -1) {
			perror("listener: socket");
			continue;
		}

		if (bind(sockfd, p->ai_addr, p->ai_addrlen) == -1) {
			close(sockfd);
			perror("listener: bind");
			continue;
		}

		break;
	}

	if (p == NULL) {
		fprintf(stderr, "listener: failed to bind socket\n");
		return 2;
	}

	freeaddrinfo(servinfo);

	printf("listener: waiting to recvfrom...\n");

	addr_len = sizeof their_addr;
	if ((numbytes = recvfrom(sockfd, buf, MAXBUFLEN-1 , 0,
		(struct sockaddr *)&their_addr, &addr_len)) == -1) {
		perror("recvfrom");
		exit(1);
	}

	printf("listener: got packet from %s\n",
		inet_ntop(their_addr.ss_family,
			get_in_addr((struct sockaddr *)&their_addr),
			s, sizeof s));
	printf("listener: packet is %d bytes long\n", numbytes);
	buf[numbytes] = '\0';
	printf("listener: packet contains \"%s\"\n", buf);

	close(sockfd);

	return 0;
}
```

`getaddrinfo()` の呼び出しで、最終的に `SOCK_DGRAM` を使用していることに注意してください。また、`listen()` や `accept()` は必要ないことに注意してください。これは非接続型データグラムソケットを使用する利点の1つです！

[talker.c ソースコード](https://beej.us/guide/bgnet/examples/talker.c)

```{.c .numberLines}
/*
** talker.c -- a datagram "client" demo
*/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>

#define SERVERPORT "4950"	// the port users will be connecting to

int main(int argc, char *argv[])
{
	int sockfd;
	struct addrinfo hints, *servinfo, *p;
	int rv;
	int numbytes;

	if (argc != 3) {
		fprintf(stderr,"usage: talker hostname message\n");
		exit(1);
	}

	memset(&hints, 0, sizeof hints);
	hints.ai_family = AF_INET6; // set to AF_INET to use IPv4
	hints.ai_socktype = SOCK_DGRAM;

	if ((rv = getaddrinfo(argv[1], SERVERPORT, &hints, &servinfo)) != 0) {
		fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(rv));
		return 1;
	}

	// loop through all the results and make a socket
	for(p = servinfo; p != NULL; p = p->ai_next) {
		if ((sockfd = socket(p->ai_family, p->ai_socktype,
				p->ai_protocol)) == -1) {
			perror("talker: socket");
			continue;
		}

		break;
	}

	if (p == NULL) {
		fprintf(stderr, "talker: failed to create socket\n");
		return 2;
	}

	if ((numbytes = sendto(sockfd, argv[2], strlen(argv[2]), 0,
			 p->ai_addr, p->ai_addrlen)) == -1) {
		perror("talker: sendto");
		exit(1);
	}

	freeaddrinfo(servinfo);

	printf("talker: sent %d bytes to %s\n", numbytes, argv[1]);
	close(sockfd);

	return 0;
}
```

と、これだけです！`listener` をあるマシンで実行し、次に `takler` を別のマシンで実行します。それらのコミュニケーションをご覧ください！核家族で楽しめるG級興奮体験です！

今回はサーバを動かす必要もありません。`talker` はただ楽しくパケットをエーテルに発射し、相手側に `recvfrom()` の準備が出来ていなければ消えてしまうのです。UDP データグラムソケットを使用して送信されたデータは、到着が保証されていないことを思い出してください！

過去に何度も述べた、もうひとつの小さなディテールを除いては、コネクテッド・データグラム・ソケットです。このドキュメントのデータグラムセクションにいるので、ここでこれについて話す必要があります。例えば、`talker` が `connect()` を呼び出して `listener` のアドレスを指定したとします。それ以降、`talker` は `connect()` で指定されたアドレスにのみ送信と受信ができます。このため、`sendto()` と `recvfrom()` を使う必要はなく、単に `send()` と `recv()` を使えばいいのです。
