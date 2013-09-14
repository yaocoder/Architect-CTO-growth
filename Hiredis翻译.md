#HIREDIS#
****
Hiredis是redis数据库一个轻量的C语言客户端库。

之所以轻量是由于它只是简单的提供了对redis操作语句支持的接口，并没有实现具体的操作语句的功能。但正是由于这种设计使我们只要熟悉了通用的redis操作语句就可以很容易的使用该库和redis数据库进行交互。

除了支持发送命令和接收应答/应答数据，它提供了对应答数据的解析操作。而且这个基于I/O层的数据流解析操作设计考虑到了复用性，可以对应答数据进行通用的解析操作。

Hirides仅仅支持二进制安全的redis协议，所以你只能针对版本号大于等于1.2.0的redis服务端使用。

库包含多种API，包括同步命令操作API、异步命令操作API和对应答数据进行解析的API。

##升级
版本0.9.0是hiredis很多特性一次大的更新。但是对现有代码进行升级应该不会造成大的问题。升级时，要记住的关键一点是大于等于0.9.0的版本是使用redisContext*来保持连接状态，之前的版本只是使用了无状态的文件描述符。

##同步API
有几个API需要介绍
>redisContext *redisConnect(const char *ip, int port);  
>void *redisCommand(redisContext *c, const char *format, ...);  
>void freeReplyObject(void *reply); 

###连接redis数据库
函数 redisConnect 被用来创建一个 redisContext。这个 context 是hiredis持有的连接状态。redisConnect 结构体有一个整型的 err 变量来标识连接错误码，如果连接错误则为非零值。变量 errstr 标识连接结果的文字描述。更多这方面的信息会在以下**Errors**章节说明。当你使用 redisConnect 来创建连接时应该检查err变量来判断是否连接正常建立。
>redisContext *c = redisConnect("127.0.0.1", 6379);  
>if (c != NULL && c->err) {  
>		printf("Error: %s\n", c->errstr);		
>		// handle error		
>	}

###发送命令到redis
有多种方法可以发送命令到redis。  

首先介绍的是redisCommand。此函数类似于printf的使用方式，如
>reply = redisCommand(context, "SET foo bar");

类似于printf的s%格式化方式，如
>reply = redisCommand(context, "SET foo %s", value);

当你需要发送二进制安全的命令可以采用%b的格式化方式，同时需要一个字符串指针和size_t类型的字符串长度参数,如下
>reply = redisCommand(context, "SET foo %b", value, (size_t) valuelen);

在API内部，Hiredis根据不同的参数分割命令转化为操作redis数据库的标准命令，你可以格式化多个参数来构造redis的命令，如下

`reply = redisCommand(context, "SET key:%s %s", myid, value);`

###处理redis应答
当命令被成功执行后redisCommand会有相应的返回值。如果有错误发生，返回值为NULL并且redisReply结构体中的err变量将会被设置成相应的值（请参照**Errors**章节）。**一旦有错误发生context不能被重用并且你需要建立一个新的连接**。

redisCommand执行后返回值类型为redisReply。通过redisReply结构体中的type变量可以确定命令执行的情况。

*  REDIS\_REPLY\_STATUS：
	* 返回执行结果为状态的命令。比如set命令的返回值的类型是REDIS\_REPLY\_STATUS，然后只有当返回信息是"OK"时，才表示该命令执行成功。可以通过reply->str得到文字信息，通过reply->len得到信息长度。

*  REDIS\_REPLY\_ERROR：
	* 返回错误。错误信息可以通过reply->str得到文字信息，通过reply->len得到信息长度。


*  REDIS\_REPLY\_INTEGER：
	* 返回整型标识。可以通过reply->integer变量得到类型为long long的值。

*  REDIS\_REPLY\_NIL:
	* 返回nil对象，说明不存在要访问的数据。

*  REDIS\_REPLY\_STRING:
	* 返回字符串标识。可以通过reply->str得到具体值，通过reply->len得到信息长度。

*  REDIS\_REPLY\_ARRAY:
	* 返回数据集标识。数据集中元素的数目可以通过reply->elements获得，每个元素是个redisReply对象，元素值可以通过reply->element[..index..].*形式获得，用在获取多个数据结果的操作。

执行完命令调用后应该通过freeReplyObject()释放redisReply，**对于嵌套对象（比如数组）要注意，并不需要嵌套进行释放，这样是有害的会造成内存破坏。**

**Important:**hiredis当前版本 (0.10.0)当使用异步API时会自己释放replies对象。这意味着你使用异步API时并不需要主动调用freeReplyObject 。relpy对象当回调返回时将会被自动释放。但是这种行为也许会在将来的版本中改变，所以升级时请密切关注升级日志。

###清理连接资源###
断开连接并且释放context使用以下函数
>void redisFree(redisContext *c);

此函数立马关闭socket并且释放创建context时分配的资源。

###发送多个命令参数###
和redisCommand函数相似，redisCommandArgv函数可以用于传输多个命令参数。函数原型为
>void *redisCommandArgv(redisContext *c, int argc, const char **argv, const size_t *argvlen);

，类似于 lpush， del key1 key2..., zadd key score1 member1 score2 member2...这类命令， 其中 argc是传递参数的个数， argv主要用于传递的string的value, 而argvlen 是每个string的size。

此函数返回值与redisCommand相似。参考[https://gist.github.com/dspezia/1455082](https://gist.github.com/dspezia/1455082)

###管线（Pipelining）###
为了搞清楚Hiredis在阻塞连接下的管线操作，需要理解其内部执行过程。

当任何类似于redisCommand的函数被调用，Hiredis首先将命令格式化为redis支持的命令协议。被格式化后的命令被放入context的输出缓冲区，这个缓冲区是动态的，所以它可以容纳任意数量的命令。在命令进入输出缓冲区后，redisGetReply 函数被调用。这个函数有以下两种执行方式：
  
1. 输入缓冲区非空：  

	* 从输入缓冲区中尝试解析单独的reply对象并且返回reply
	* 如果没有reply能被解析，执行步骤2
	

2. 输入缓冲区为空：

	*  将整个输出缓冲区写入socket
	*  从socket中读取数据直到有一个reply能被解析


Hiredis为了有效利用socket还提供了redisGetReply的接口。对于管线命令，需要完成的唯一事情就是填充输出缓冲区。有两个函数被用于执行此操作，这两个函数基本与redisCommand函数功能类似，但是他们不返回reply
>void redisAppendCommand(redisContext *c, const char *format, ...);  
>void redisAppendCommandArgv(redisContext *c, int argc, const char **argv, const size_t *argvlen);

当这两个函数被一次或多次调用时，通过redisGetReply依次返回replies。redisGetReply的返回值要么是REDIS_OK或是REDIS_ERR，REDIS_ERR意味着获得reply发生了错误，想要得到具体的错误原因可以通过err变量来获取。

以下通过一个简单例子说明管线的使用：
>redisReply *reply;  
>redisAppendCommand(context,"SET foo bar");  
>redisAppendCommand(context,"GET foo");  
>redisGetReply(context,&reply); // reply for SET  
>freeReplyObject(reply);  
>redisGetReply(context,&reply); // reply for GET  
>freeReplyObject(reply);

redisGetReply这个API也可以被用来实现阻塞的订阅模式

>reply = redisCommand(context,"SUBSCRIBE foo");  
>freeReplyObject(reply);  
>while(redisGetReply(context,&reply) == REDIS_OK)   
>{  
>    // consume message  
>    freeReplyObject(reply);  
>}


###Errors###
如果某些函数（如redisConnect， redisCommand（调用不成功，函数返回值为NULL或者REDIS_ERR，此时context结构体中的err成员为非零值，可能为以下几种常量

* REDIS\_ERR\_IO:当创建连接时（试着写socket或者读socket）发生的I/O错误。如果你在代码中包含了errno.h头文件，你便能得到准确的错误码。

* REDIS\_ERR\_EOF:redis服务端已经关闭了此连接。

* REDIS\_ERR\_PROTOCOL:服务端解析协议时发生了错误。

* REDIS\_ERR\_OTHER:其他错误。目前仅仅表示无法解析目标主机名的错误。

在错误情况下，可以通过context结构体中的errstr成员得到错误的确切描述。

##异步API##

Hiredis自带的异步API很容易和一些基于事件的库结合使用。比如和libev、ibevent的结合使用。

###连接###
函数redisAsyncConnect被用来和redis建立非阻塞连接。它返回redisAsyncContext的结构体，结构体的err成员用来检查在创建连接的过程中是否发生了错误。因为创建的是非阻塞的连接，内核并不能立马返回一个连接指定主机的结果。
>redisAsyncContext *c = redisAsyncConnect("127.0.0.1", 6379);  
>if (c->err)   
>{  
>printf("Error: %s\n", c->errstr);  
>// handle error  
>}

异步Context可设置一个响应断开连接事件的回调函数，当连接断开时会相应执行。回调函数的原型为
>void(const redisAsyncContext *c, int status);

在断开连接的情况下，当连接是由用户自己断开的status参数为REDIS_OK，如果出现了其他错误status参数为REDIS_ERR，当错误时通过err成员可以得到准确的错误码。

当回调执行完毕后context对象会自己释放资源。此事件的回调函数给你创建一个新连接提供了便利。

一个context对象仅能设置一次断开连接的回调，如果再进行下一次设置将会返回REDIS_ERR。设置断开连接回调函数的原型为：
>int redisAsyncSetDisconnectCallback(redisAsyncContext *ac, redisDisconnectCallback *fn);

###发送操作命令并且响应回调事件###
在异步的情况下，redis的操作指令将会被自动加入事件循环队列。由于发送命令执行的过程是异步的，当命令执行完毕后将会调用相应的回调函数。回调函数的原型为
>void(redisAsyncContext *c, void *reply, void *privdata);

privdata参数是由调用者自己定义的数据类型。

以下是进行异步命令操作的函数：
>int redisAsyncCommand(redisAsyncContext *ac, redisCallbackFn *fn, void *privdata,const char *format, ...); 
> 
>int redisAsyncCommandArgv(
  redisAsyncContext *ac, redisCallbackFn *fn, void *privdata,
  int argc, const char **argv, const size_t *argvlen);

命令的使用方式和上面所讲的同步接口类似。执行成功返回REDIS\_OK，否则返回REDIS\_ERR。比如连接已经关闭的情况下再使用redisAsyncCommand向此连接执行命令就会返回REDIS\_ERR。

回调执行完毕后如果reply不为空，回调执行完毕后将自动对reply的资源进行回收。

当context发生错误时回调得到的reply将为空。

###断开连接###
一个异步的连接可以通过下面这个函数终止
>void redisAsyncDisconnect(redisAsyncContext *ac);

当这个函数被调用时，连接并不会被立即关闭，而是等待所有这个连接的异步命令操作执行完毕，并且回调事件已经执行完毕后才关闭此连接，这时在响应关闭连接事件的回调函数中得到的状态为REDIS_OK，此连接的资源也将会被自动回收。

###将其挂接到事件库X###
在context对象被创建后进行很少的几步操作就可以进行挂接。参看目录adapters/下是如何挂接到libev 和 libevent下的。

##应答解析API##
Hiredis自带的答复解析API ，可以很容易与更高层次的语言进行绑定。TODO：



 



