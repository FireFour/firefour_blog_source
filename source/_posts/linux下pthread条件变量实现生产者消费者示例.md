---
title: linux下pthread条件变量实现生产者消费者示例
date: 2018-05-17 15:45:50
tags: [linux, C/C++, pthread]
categories: coding
---
Linux下pthread线程同步主要有两种方法：信号量(semaphore)和条件变量(condition_variable)，在生产者消费者的实例中，通常用到的是信号量来进行同步。本文采用条件变量的通知机制，实现类似信号量的功能，完成生产者消费者示例，并在最后贴出代码。另外pthread的信号量有二值信号量和计数信号量两种，第一种信号量只有0和1两个值，用法类似条件变量，第二种就是传统的介于0-限制值的一个信号量，可以用来统计可用资源数目。本文就是用了`notempty`和`notfull`两个条件变量来表示资源空和资源已达上限两种状态。

### 程序步骤
首先，用户需要在main函数里创建consumer和producer两个线程，之后调用join方法等待两个线程退出。同时要用到两个条件变量`notempty`和`notfull`。在生产者线程中，生产一个资源之前先判断`notfull`，确保资源数没到上限，之后再生产一个资源，即往消息队列中push一个Packet，然后向消费者发送一个`notempty`信号表示已经生产了一个资源，可用资源数不为空。在消费者线程中，消费一个资源之前先判断`notempty`，确保当前消息队列中有资源，之后再消费一个资源，即从消息队列中pop一个Packet，然后向生产者发送一个`notfull`信号表示已经消费了一个资源，资源数没有到上限。生产者和消费者的处理函数正好是两个相反的对称的过程，可以用下图来表示：

![consumer_producer](http://p8tf9aeyd.bkt.clouddn.com/consumer_producer.png)

### 实现代码
```c++
#include <stdio.h>
#include <iostream>
#include <queue>
#include <pthread.h>
#include <string.h>

using namespace std;

const int BUFFER_SIZE = 1500;

typedef struct Packet_s{
	char buffer[BUFFER_SIZE];
	int len;
}Packet;

queue<Packet> packetQueue;
const int LIMIT_SIZE = 1500;
// 缓冲区相关数据结构
pthread_mutex_t lock; /* 互斥体lock 用于对缓冲区的互斥操作 */
// int readpos, writepos; /* 读写指针*/
pthread_cond_t notempty; /* 缓冲区非空的条件变量 */
pthread_cond_t notfull; /* 缓冲区未满的条件变量 */	


// 生产者线程处理函数
void *produce(void *data){
	for(int i=0; i<100000; i++){
		Packet tmp;
		memcpy(tmp.buffer, &i, sizeof(i));
		tmp.len = sizeof(i);
		
		// 假定队列的大小限制为1500，到达这个值认为队列满了，等待
		// 消费者取出后，再向队列里push
		pthread_mutex_lock(&lock);
		while(packetQueue.size() == LIMIT_SIZE){
			pthread_cond_wait(&notfull, &lock);
		}

		packetQueue.push(tmp);
		pthread_mutex_unlock(&lock); // 给互斥体变量解除锁
		
		pthread_cond_signal(&notempty);
		// 每一次push后设置notempty条件变量
		cout << "Producer[" << i << "]" << endl;
	}
	return NULL;
}

// 消费者线程处理函数
void *consume(void *data){
	while(1){
		pthread_mutex_lock(&lock);
		while(packetQueue.size() == 0){
			pthread_cond_wait(&notempty, &lock);
		}
		Packet packet = packetQueue.front();
		packetQueue.pop();
		pthread_mutex_unlock(&lock);	
		
		pthread_cond_signal(&notfull);
		int *data = (int *)packet.buffer;
		cout << "Consumer[" << *data << "]" << endl;
		if(*data == 99999)
			break;
	}	
	return NULL;	
}

int main(){
	//初始化条件变量和互斥锁
	pthread_mutex_init(&lock, NULL);
	pthread_cond_init(&notempty, NULL);
	pthread_cond_init(&notfull, NULL);

	pthread_t producer, consumer;

	/* 创建生产者和消费者线程*/
	pthread_create(&producer, NULL, produce, NULL);
	pthread_create(&consumer, NULL, consume, NULL);

	/* 等待线程退出 */
	pthread_join(producer, NULL);
	cout << "producer exit!" << endl;
	pthread_join(consumer, NULL);
	cout << "consumer exit!" << endl; 

	pthread_mutex_destroy(&lock);
	pthread_cond_destroy(&notempty);
	pthread_cond_destroy(&notfull);
	return 0;
}
```
### 关于条件变量
- 基本用法

    1. 声明条件变量数据结构`pthread_cond_t`
    2. 初始化条件变量`pthread_cond_init(&cond_var, NULL);`
    3. 发送信号`pthread_cond_signal(&cond_var);`和等待信号`pthread_cond_wait(&notempty, &lock);`

- 使用要点：
    - 条件变量只有一种正确使用的方式，几乎不可能用错。对于 wait 端：
        1. 必须与 mutex 一起使用，该布尔表达式的读写需受此 mutex 保护。
        2. 在 mutex 已上锁的时候才能调用 wait()。
        3. 把判断布尔条件和 wait() 放到 while 循环中。

    - 对于 signal/broadcast 端：
        1. 不一定要在 mutex 已上锁的情况下调用 signal （理论上）。
        2. 在 signal 之前一般要修改布尔表达式。
        3. 修改布尔表达式通常要用 mutex 保护（至少用作 full memory barrier）。
        4. 注意区分 signal 与 broadcast：“broadcast 通常用于表明状态变化，signal 通常用于表示资源可用。（broadcast should generally be used to indicate state change rather than resource availability。）”

- 关于mutex:

    `pthread_cond_wait()`中的wait的参数有一个mutex，这里的mutex和用于同步消息队列的mutex是不同的，可以简单理解为每一个共享资源都要对应一个mutex，消息队列是共享资源，因此线程对其读写要用mutex保护，保证每一个时刻只有一个线程可以对资源进行操作。而`pthread_cond_wait()`参数里的mutex是用来保护while循环条件中的共享资源的。因为通常情况下，每一个wait函数上面都会紧随着一个while循环作为判断，这是为了避免**异常唤醒**，即一个线程对某一条件变量signal可能会唤醒多个的线程，或者不相关的线程，因此要用一个条件作为约束，在我们的程序中就是用消息队列中资源的个数`packetQueue.size()`来防止异常唤醒。

### 参考资料
> [用条件变量实现事件等待器的正确与错误做法 ——陈硕的Blog](http://www.cppblog.com/Solstice/archive/2015/10/30/203094.html)
