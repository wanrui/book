# plugin 简介

[TOC]

## HTTP事务流程：
  必须要理解的是trafficserver是个程序，主要的业务是无数的transaction，每个transaction都是一个用户的http连接处理，不仅包含了用户的tcp连接，还包含了traffic server与后端的通信和本地操作等。一个transaction只是用户一个tcp连接上执行的一次事务，还有一个session概念，是一个client和server之间tcp概念上的连接。一个session可以包括很多个transaction。对用户来说，一个request和response是一个transaction

## 原理：

  标准的面向过程做插件的过程。一个HTTP有个处理流程，包括request头部处理（你可以改url），dns查询（你可以决定去哪个后台获取数据）、从后台或缓存拉取数据、返回内容等。只要是http请求，这个流程就是固定的。因此插件系统就是在这些流程上注册回调函数。这里的回调函数还不是直接调用，还会传递一个event事件参数，用于表示在当前的钩子上发生的事情，使plugin可以更好的处理。
  除了被调用，trafficserver还要提供调用方法。这里提供的调用方式可不是一般意义上的函数调用，而是类似远程过程调用。插件通过将自己要被执行的代码（action）发送给server（就连发送都是要指明ip地址和端口的），然后通过查询server返回的接口来获得action执行的状态。这里的action就是traffic server里面的协程概念，整个过程类似golang的go func(){}()关键字操作。
  除了这种远程调用，很多函数插件也是可以直接调用的。
## 协程：
  Trafficserver的超高并发自然需要协程的概念（ng也是）。Traffic server自己实现的协程叫做continuation，结构体用TSCont表示。

  一个TSCont代表一个异步执行的代码块，这个代码块拥有自己的执行状态，并且可以被

  协程是用户空间管理的线程，也就是说调度算法是在用户空间的程序中实现的。可以保存程序执行的状态，可以在某时刻拉出来执行。多个协程在一个操作系统的线程上执行，或者是M个协程在N个线程上执行。如此带来的好处是可以任性的阻塞，不必担心资源的浪费问题。所以协程本质上也是一种应对阻塞调用的方式。其他的重要思想还有异步。貌似操作系统更倾向于异步，而不是倾向于协程。
  Trafficserver底层大量基于异步，但向上提供的并发却大量基于协程的概念。
## 插件类型：
	1. 普通插件
	2. 内容变换
	内容变换是修改request的内容或者response的内容。由于内容是变长的，所以traffic server定义了vconnection（结构体TSVConn）和vio。Vconnection代表从一个buffer到另一个buffer的连接，经过这个连接的数据可以根据连接指定的变化方法变化。这也就是内容变换的本质。本质上TSVConn是一个continuation，所以也具备协continuation具备的数据通知能力。 而VIO是VConnection两端。一个input一个output，由于可以多个vconnection串行，所以一个vconnection的output vio就可以另一个vconnection的input。Vconnection的本质是变换，VIO的本质是内存buffer。
	2.其他协议插件
	这个就比较底层了。一般的插件都是服务于http协议的，你也可以直接跳过http协议支持别的协议，或者是支持http之上的其他协议。课件traffic server对其网络基础结构的信心。

插件编写：
		 1.  向主程序注册插件：TSPluginRegister。可以不注册，主要是为了兼容性
		 2.  向某个全局钩子位置添加钩子回调：TSHttpHookAdd
			-  注册的钩子可以是全局的也可能是trasaction、session相关的。如果是transaction相关的，通过TSHttpTxn txnp = (TSHttpTxn)edata;获得transaction的指针。使用TSHttpTxnHookAdd函数添加transactionhook。
			-   如果是session相关的，使用TSHttpSsnHookAdd进行注册。Plugin中获得session的方法变为TSHttpSsn ssion = (TSHttpSsn)edata;
 
插件允许发起网络连接，使用TSNetConnect()发起只连接traffic server的http连接，TSHttpConnect()发起向任意地址的http连接。

## ATS plugin 的应用 
![](cpp_api_image/Possible_Traffic_Server_Plugins.jpg) 


## Plugin 异步事件模型
Asynchronous Event Model  

1. 利用可用的多个cpu和多个I / O设备的并发性。
2. 管理并发许多同步客户端连接。例如,一个服务器可以为每个连接创建一个线程,允许操作系统(OS)来控制线程之间切换。

traffic server 内部应该有如下实现：
```cpp
for (;;) {
   event = get_next_event();
   handle_event (event);
}
```

## HTTP HOOK 种类

- Global HTTP hooks
	用TSHttpHookAdd 函数在TSPluginInit函中进行挂载钩子 
- Transaction hooks
	为指定 HTTP transaction挂载钩子，不是在TSPluginInit函数中使用。
	https://docs.trafficserver.apache.org/en/latest/developer-guide/plugins/continuations/writing-handler-functions.en.html
- Transformation hooks	
	使用TSHttpTxnHookAdd 挂载transform 事件
- Session hooks
	TSHttpSsnHookAdd() 
- HTTP select alternate hook

## HTTP HOOK 回调
TSEventFunc 回调函数定义如下：
```cpp
static int function_name (TSCont contp, TSEvent event, void *edata)
```
其中edata 类型的数据根据不同的事件触发类型都不一样，一般HTTP事务 中的回调类型为TSHttpTxn ，所有事件 对应的edata 类型如下：

|   Event  |  Event Sender   |   Data Type  |
| --- | --- | --- |
|	TS_EVENT_HTTP_READ_REQUEST_HDR	|	TS_HTTP_READ_REQUEST_HDR_HOOK	|	TSHttpTxn	|
|	TS_EVENT_HTTP_PRE_REMAP	|	TS_HTTP_PRE_REMAP_HOOK	|	TSHttpTxn	|
|	TS_EVENT_HTTP_OS_DNS	|	TS_HTTP_OS_DNS_HOOK	|	TSHttpTxn	|
|	TS_EVENT_HTTP_SEND_REQUEST_HDR	|	TS_HTTP_SEND_REQUEST_HDR_HOOK	|	TSHttpTxn	|
|	TS_EVENT_HTTP_READ_CACHE_HDR	|	TS_HTTP_READ_CACHE_HDR_HOOK	|	TSHttpTxn	|
|	TS_EVENT_HTTP_READ_RESPONSE_HDR	|	TS_HTTP_READ_RESPONSE_HDR_HOOK	|	TSHttpTxn	|
|	TS_EVENT_HTTP_SEND_RESPONSE_HDR	|	TS_HTTP_SEND_RESPONSE_HDR_HOOK	|	TSHttpTxn	|
|	TS_EVENT_HTTP_SELECT_ALT	|	TS_HTTP_SELECT_ALT_HOOK	|	TSHttpTxn	|
|	TS_EVENT_HTTP_TXN_START	|	TS_HTTP_TXN_START_HOOK	|	TSHttpTxn	|
|	TS_EVENT_HTTP_TXN_CLOSE	|	TS_HTTP_TXN_CLOSE_HOOK	|	TSHttpTxn	|
|	TS_EVENT_HTTP_SSN_START	|	TS_HTTP_SSN_START_HOOK	|	TSHttpSsn	|
|	TS_EVENT_HTTP_SSN_CLOSE	|	TS_HTTP_SSN_CLOSE_HOOK	|	TSHttpSsn	|
|	TS_EVENT_NONE	|		|		|
|	TS_EVENT_CACHE_LOOKUP_COMPLETE	|	TS_HTTP_CACHE_LOOKUP_COMPLETE_HOOK	|	TSHttpTxn	|
|	TS_EVENT_IMMEDIATE	|	TSVConnClose() TSVIOReenable()TSContSchedule()	|		|
|	TS_EVENT_IMMEDIATE	|	TS_HTTP_REQUEST_TRANSFORM_HOOK	|		|
|	TS_EVENT_IMMEDIATE	|	TS_HTTP_RESPONSE_TRANSFORM_HOOK	|		|
|	TS_EVENT_CACHE_OPEN_READ	|	TSCacheRead()	|	Cache VC	|
|	TS_EVENT_CACHE_OPEN_READ_FAILED	|	TSCacheRead()	|	TS_CACHE_ERROR code	|
|	TS_EVENT_CACHE_OPEN_WRITE	|	TSCacheWrite()	|	Cache VC	|
|	TS_EVENT_CACHE_OPEN_WRITE_FAILED	|	TSCacheWrite()	|	TS_CACHE_ERROR code	|
|	TS_EVENT_CACHE_REMOVE	|	TSCacheRemove()	|		|
|	TS_EVENT_CACHE_REMOVE_FAILED	|	TSCacheRemove()	|	TS_CACHE_ERROR code	|
|	TS_EVENT_NET_ACCEPT	|	TSNetAccept() TSHttpTxnServerIntercept()TSHttpTxnIntercept()	|	TSNetVConnection	|
|	TS_EVENT_NET_ACCEPT_FAILED	|	TSNetAccept() TSHttpTxnServerIntercept()TSHttpTxnIntercept()	|		|
|	TS_EVENT_HOST_LOOKUP	|	TSHostLookup()	|	TSHostLookupResult	|
|	TS_EVENT_TIMEOUT	|	TSContSchedule()	|		|
|	TS_EVENT_ERROR	|		|		|
|	TS_EVENT_VCONN_READ_READY	|	TSVConnRead()	|	TSVIO	|
|	TS_EVENT_VCONN_WRITE_READY	|	TSVConnWrite()	|	TSVIO	|
|	TS_EVENT_VCONN_READ_COMPLETE	|	TSVConnRead()	|	TSVIO	|
|	TS_EVENT_VCONN_WRITE_COMPLETE	|	TSVConnWrite()	|	TSVIO	|
|	TS_EVENT_VCONN_EOS	|	TSVConnRead()	|	TSVIO	|
|	TS_EVENT_NET_CONNECT	|	TSNetConnect()	|	TSVConn	|
|	TS_EVENT_NET_CONNECT_FAILED	|	TSNetConnect()	|	TSVConn	|
|	TS_EVENT_HTTP_CONTINUE	|		|		|
|	TS_EVENT_HTTP_ERROR	|		|		|
|	TS_EVENT_MGMT_UPDATE	|	TSMgmtUpdateRegister()	|		|

处理方式举例如下：

```cpp
static int
blacklist_plugin (TSCont contp, TSEvent event, void *edata)
{
   TSHttpTxn txnp = (TSHttpTxn) edata;
   switch (event) {
      case TS_EVENT_HTTP_OS_DNS:
         handle_dns (txnp, contp);
         return 0;
      case TS_EVENT_HTTP_SEND_RESPONSE_HDR:
         handle_response (txnp);
         return 0;
      default:
         break;
   }
   return 0;
}
```

![](cpp_api_image/Asynchronous_Event_Model.jpg) 


## Plugin 可用API 
按照API的功能分类，可分为如下5项：

1. HTTP header manipulation functions
	获得信息和操作HTTP头、url、& MIME标头。
2. HTTP transaction functions
	获得和修改HTTP事务信息(例如:客户端IP相关事务;得到服务器IP,得到父代理信息)
3. IO functions
	操纵vconnections(虚拟连接,用于网络和磁盘I / O)
3. Network connection functions
	打开的连接到远程服务器。
4. Statistics functions
	为你的插件活动定义和计算统计数据。
5. Traffic Server management functions
	获得Traffic Server 配置和统计变量的值。







