# 课程设计

## ex

粗体文本：**粗体文本**  __粗体文本__
斜体文本：*斜体文本*    _斜体文本_
删除文本：~~删除文本~~
下划线：<u>下划线文本</u>

| 客户端 | 服务端 |
| ------ | ------ |
|        |        |



--------

上标和下标：2^10^，H~2~O
HTML 标签：2<sup>10</sup>，H<sub>2</sub>O



![9765809989d44f6fab21d869bfb5e2e4](C:\Users\86158\Desktop\课设\9765809989d44f6fab21d869bfb5e2e4.png)





## 代码

**客户端**

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>

#define PORT 8000      // 服务端监听的端口号

int main(int argc, char *argv[]) {
    if (argc != 3) {
        printf("Usage: %s <server IP> <filename>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    char *ip = argv[1];                    // 服务端IP地址
    char *filename = argv[2];              // 要发送的文件名

    char buffer[1024];                     // 缓冲区
    int sockfd;                            // 套接字文件描述符
    struct sockaddr_in serv_addr;          // 服务端地址结构体
    ssize_t read_len, sent_len, total_len = 0;  // 读取、发送、总共发送的数据长度
    FILE *fp;                              // 文件指针

    /* 打开要传输的文件 */
    if ((fp = fopen(filename, "rb")) == NULL) {
        perror("fopen");
        exit(EXIT_FAILURE);
    }

    /* 创建套接字 */
    if ((sockfd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    /* 初始化服务端地址结构体 */
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = inet_addr(ip);
    serv_addr.sin_port = htons(PORT);

    /* 建立连接 */
    if (connect(sockfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) == -1) {
        perror("connect");
        exit(EXIT_FAILURE);
    }

    printf("Connected to server %s:%d\n", ip, PORT);

    /* 读取并发送数据 */
    while ((read_len = fread(buffer, 1, sizeof(buffer), fp)) > 0) {
        sent_len = send(sockfd, buffer, read_len, 0);
        if (sent_len != read_len) {
            perror("send");
            exit(EXIT_FAILURE);
        }
        total_len += sent_len;
    }

    /* 关闭套接字和文件 */
    close(sockfd);
    fclose(fp);

    printf("Sent %zd bytes of file '%s' to server\n", total_len, filename);

    return 0;
}

```

**服务端**

```C
client.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <errno.h>

#define PORT 8000      // 服务端监听的端口号

int main(int argc, char *argv[]) {
    if (argc != 3) {
        printf("Usage: %s <file directory> <filename>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    char *dir = argv[1];                   // 接收到的文件目录
    char *filename = argv[2];              // 接收到的文件名

    char buffer[1024];                     // 缓冲区
    int listen_fd, conn_fd;                // 监听和连接文件描述符
    struct sockaddr_in serv_addr, client_addr; // 服务端和客户端地址结构体
    socklen_t cli_len;                     // 客户端地址字节长度
    ssize_t recv_len, written_len, total_len = 0;   // 接收、写入、总共接收的数据长度

    /* 创建监听套接字 */
    if ((listen_fd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    /* 初始化服务端地址结构体 */
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_port = htons(PORT);

    /* 绑定套接字和地址 */
    if (bind(listen_fd, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) == -1) {
        perror("bind");
        exit(EXIT_FAILURE);
    }

    /* 监听连接请求 */
    if (listen(listen_fd, 10) == -1) {
        perror("listen");
        exit(EXIT_FAILURE);
    }

    printf("Listening on port %d...\n", PORT);

    /* 接受连接请求 */
    cli_len = sizeof(client_addr);
    if ((conn_fd = accept(listen_fd, (struct sockaddr *)&client_addr, &cli_len)) == -1) {
        perror("accept");
        exit(EXIT_FAILURE);
    }

    printf("Accepted connection from %s:%d\n",
            inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));

    /* 打开文件并准备写入 */
    char filepath[1024];
    sprintf(filepath, "%s/%s", dir, filename);  // 拼接文件路径
    FILE *fp = fopen(filepath, "wb");
    if (fp == NULL) {
        perror("fopen");
        exit(EXIT_FAILURE);
    }

    /* 接收数据并写入文件 */
    while ((recv_len = recv(conn_fd, buffer, sizeof(buffer), 0)) > 0) {
        written_len = fwrite(buffer, 1, recv_len, fp);
        if (written_len != recv_len) {
            perror("fwrite");
            exit(EXIT_FAILURE);
        }
        total_len += written_len;
    }
    if (recv_len < 0 && errno != EAGAIN && errno != EWOULDBLOCK) {
        perror("recv");
        exit(EXIT_FAILURE);
    }

    /* 关闭套接字和文件 */
    close(conn_fd);
    fclose(fp);

    printf("Received %zd bytes and saved to file '%s'\n", total_len, filepath);

    return 0;
}

```

