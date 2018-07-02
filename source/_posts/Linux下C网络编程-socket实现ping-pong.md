---
title: Linux下c/c++网络编程:socket实现ping-pong
date: 2018-05-10 09:31:58
tags: [C/C++, socket, udp]
categories: coding
---
在Linux下实现socket网络编程的示例网上有很多，在工作中遇到的场景是要测试跨机器的网络时延。之前采用了一个开源工具[`sfnettest`](http://www.openonload.org/download/sfnettest/)来做回环测试，但是`sfnt`封装了较多层，一些小的定制化改动不太方便，于是打算模仿`sfnt-pingpong`自己在Linux下写一个`ping-pong`程序来测试回环的时延，顺便也练练手。

### 原理
程序将client和server端写在了一起，通过命令行参数来区分。运行时先运行server，再运行client。程序开始工作后，client端会先向server端发ping，具体为调用`sendto()`函数(这里用的是无连接的UDP协议)向server端发送一个512个字节长度的UDP报文，之后调用`recvfrom()`阻塞等待server端的pong返回。Server端先调用`recvfrom()`阻塞等待客户端发来的ping，收到后再把收到的报文调用`sendto()`原路返回给客户端，这样就完成了一次`ping-pong`，一次`ping-pong`的时间除以2就是跨机器的时延。

### 获取纳秒级时间
Linux下获取时间的方法又三种：`gettimeofday()`, `clock_gettime()`, `rdtsc(rdtscp)`
1. `gettimeofday()`

    这种方式使用unix时间戳来获取微秒级别的精度，缺陷就在于当时间精度要求到纳秒级别时，测量精度不够。
2. `clock_gettime()`

    这种方式和上面的类似，也是用unix时间戳来获取，区别在于能够获取到纳秒级别的时间。
3. `rdtscp(rdtsc)`

    这种方法是在C语言里内嵌汇编，直接读取cpu的`timestamp counter`时间戳寄存器中的值，因此在三种方法中精度也是最高。但这种方法也有自己的缺陷，首先必须要绑核，保证程序在同一个cpu上运行，否则不同cpu的`timestamp counter`初始值不同没法度量。其次这种测试方法还受到cpu频率的影响，要通过额外的工作来避免cpu跳频。同时该方法只能用来度量两个时间节点的`period`，不能度量每个时刻的实际时间。

我们的`ping-pong`程序采用第三种方法度量时间。

### 注意
`socket`的UDP编程中，接收端需要调用`bind()`来绑定socket，发送端不需要绑定直接发送即可，发送端重复绑定会报错。

### 代码

```c++
#include <stdio.h>
#include <sys/socket.h>
#include <netinet/in.h>  
#include <netdb.h>  
#include <fcntl.h>  
#include <unistd.h>  
#include <stdlib.h>
#include <sys/stat.h>  
#include <sys/types.h>  
#include <arpa/inet.h>
#include <time.h>
#include <iostream>
#include <string.h>
#include <stdint.h>
#include <csignal>
#include <iostream>
#include <assert.h>
#include <math.h>

using namespace std;

/////////////////////////////////////////////////////////
const float CPU_freq = 2.194844;
// const float CPU_freq = 2.808002;

typedef struct timestamp {
    uint32_t coreId;
    uint64_t value;
} timestamp_t;

inline void read_timestamp_counter(timestamp_t * t) {
    uint64_t highTick, lowTick;
    asm volatile ("rdtscp" : "=d"(highTick), "=a"(lowTick), "=c"(t->coreId));
    t->value = highTick << 32 | lowTick;
}

inline uint64_t
diff_timestamps(const timestamp_t * before,
                const timestamp_t * after) {
    assert(before->coreId == after->coreId);
    return after->value - before->value;
}

inline uint64_t
cycle_since_timestamp(const timestamp_t * previous) {
    timestamp_t now;
    read_timestamp_counter(&now);
    return diff_timestamps(previous, &now);
}

uint64_t get_time(){
	struct timespec tv;
    clock_gettime(CLOCK_REALTIME,&tv);
    
	uint64_t clock = (tv.tv_sec % 1000) * 1e9 + tv.tv_nsec;
	
	return clock;
}
/////////////////////////////////////////////////////////

typedef unsigned int SOCKET;
#define INVALID_SOCKET  -1  
#define SOCKET_ERROR    -1  

#define BUFFER_SZ 1200
#define SEND_BUF_LEN 512

int server_port = 33456;
int client_port = 33457;

const char* ServerIP = "127.0.0.1";

int Send_num = 200000;

const int RTT_LEN = 1000000;
uint64_t *rtt = new uint64_t[RTT_LEN];

int mode = -1;

// void sig_handler(int sig) {
//     if ( sig == SIGINT){
//         // sender
//         if(mode == 0){
            
//         }
//         // receiver
//         else{
            
//         }
//         delete []rtt;
//         close(m_sock);
//     }
//     exit(0);
// }


int main(int argc, char *argv[]){
    if(argc != 2){
        std::cout << "Usage: APP NAME, MODE(0:Servver, 1:Client)" << std::endl;
    }
    mode = atoi(argv[1]);

    /*Server: pong*/
    /*Client: ping*/

    //注册中断信号
    // signal( SIGINT, sig_handler );

    ///Server
    if(mode == 0){
        SOCKET m_sock;
        char buffer[BUFFER_SZ];
        //Create a socket
        if((m_sock = socket(AF_INET , SOCK_DGRAM , IPPROTO_UDP )) == INVALID_SOCKET)
        {
            return 0;
        }
        sockaddr_in my_addr;
        sockaddr_in remote_addr; //客户端网络地址结构体  
        memset(&my_addr,0,sizeof(sockaddr_in));
        //Prepare the sockaddr_in structure
        my_addr.sin_family = AF_INET;
        my_addr.sin_addr.s_addr = htonl(INADDR_ANY);
        my_addr.sin_port = htons( server_port );

        //Bind
        if( bind(m_sock ,(sockaddr *)&my_addr , sizeof(sockaddr)) == SOCKET_ERROR)
        {
            printf("Socket bind failed\n");
            close(m_sock);
            return 0;
        }

        int sin_size=sizeof(struct sockaddr_in);  

        cout << "waiting for packets..." << endl;
        for(int i=0; i < Send_num; i++)
        {
            // do_pong
            int nRecEcho = recvfrom(m_sock, (char*)buffer, BUFFER_SZ , 0, (struct sockaddr *)&remote_addr, (socklen_t *)&sin_size);  
            if (nRecEcho < 0)  
            {  
                printf("[SERVER]recv error/n");  
                break;  
            }
            else if (nRecEcho != 512){
                printf("invalid packet received!\n");
            }
            
            int len = sendto(m_sock, buffer, nRecEcho, 0, (struct sockaddr *)&remote_addr, sizeof(struct sockaddr));
            if(len == -1){
                std::cout << "[SERVER]send failed!" << std::endl;
            }
        }
        close(m_sock);
        cout << "complete!" << endl;
    }
    ///Client
    else{
        SOCKET m_sock;
        char buffer[SEND_BUF_LEN];

        if((m_sock = socket(AF_INET , SOCK_DGRAM , IPPROTO_UDP )) == INVALID_SOCKET)
        {
            return 0;
        }
        sockaddr_in remote_addr;
        memset(&remote_addr,0,sizeof(sockaddr_in));
        //Prepare the sockaddr_in structure
        remote_addr.sin_family = AF_INET;
        remote_addr.sin_addr.s_addr = inet_addr(ServerIP);//服务器IP地址  
        remote_addr.sin_port = htons( server_port );


        sockaddr_in my_addr;
        memset(&my_addr,0,sizeof(sockaddr_in));
        //Prepare the sockaddr_in structure
        my_addr.sin_family = AF_INET;
        my_addr.sin_addr.s_addr = htonl(INADDR_ANY);
        my_addr.sin_port = htons( client_port );

        //Bind
        if( bind(m_sock ,(sockaddr *)&my_addr , sizeof(sockaddr)) == SOCKET_ERROR)
        {
            printf("Socket bind failed\n");
            close(m_sock);
            return 0;
        }

        memset(buffer, 36, SEND_BUF_LEN);

        int sin_size=sizeof(struct sockaddr_in);  

        // do_ping
        for(int i=0; i < Send_num; i++){
            timestamp_t start;
            read_timestamp_counter(&start);

            buffer[0] = 0xff;
            buffer[1] = 0xee;
            buffer[2] = 0x00;
            buffer[3] = 0x02;//little endien
            memcpy(buffer + 110, &i, sizeof(i));
            memcpy(buffer + 50, &(start), sizeof(start));
            // int len = sendto(m_sock, buffer, sizeof(i), 0, NULL, 0);
            // std::cout << "sending: " << i << std::endl;
            int len = sendto(m_sock, buffer, SEND_BUF_LEN, 0, (struct sockaddr *)&remote_addr, sizeof(struct sockaddr));
            if(len == -1){
                std::cout << "socket send failed!" << std::endl;
            }

            int nRecEcho = recvfrom(m_sock, (char*)buffer, BUFFER_SZ , 0, (struct sockaddr *)&remote_addr, (socklen_t *)&sin_size);  
            if (nRecEcho < 0)  
            {  
                printf("[CLIENT]recv error/n");  
                break;  
            }
            else if (nRecEcho != 512){
                printf("[CLIENT]invalid packet received!\n");
            }

            uint64_t latency = cycle_since_timestamp(&start);
            rtt[i] = latency / CPU_freq / 2;
        }
        close(m_sock);

        uint64_t min = 99999, max = 0, sum = 0;
        for(int i=0; i < Send_num; i++){
            // fout << rtt[i] << endl;
            if (rtt[i] < min){
                min = rtt[i];
            }
            if (rtt[i] > max){
                max = rtt[i];
            }
            sum += rtt[i];
        }
        double avg = (double)sum / Send_num;
        double std = 0.0;
        double tmp = 0.0;    
        for(int i=0; i < Send_num; i++){
            tmp += (rtt[i] - avg) * (rtt[i] - avg);
        }
        std = sqrt(tmp/Send_num);

        cout << "num: " << Send_num << endl;
        cout << "avg: " << sum / Send_num << endl;
        cout << "min: " << min << endl;
        cout << "max: " << max << endl;
        cout << "std: " << std << endl;

        cout << "total receive valid packets: " << Send_num << endl;
    }

    return 0;
}
```