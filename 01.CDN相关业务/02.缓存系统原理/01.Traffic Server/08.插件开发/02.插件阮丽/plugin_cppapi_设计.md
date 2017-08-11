# cpp api design
## plugin type 
从功能上分可以分为：

 1. Global plugin    全局插件
 2. TransactionPlugin 会话插件
 
 其中

## class Plugin  
![](cpp_api_image/UMLClassDiagram-Plugin.png) 

可供选择的钩子如下下面代码，每一个事件在plugin 基类中都能找到对应的虚函数。如果注册了对应的事件，只需要实现对应的虚函数即可。
* HOOK_READ_REQUEST_HEADERS_PRE_REMAP  在remap 之前就被触发。
* HOOK_READ_REQUEST_HEADERS_POST_REMAP   在remap 之后被触发。
* HOOK_SEND_REQUEST_HEADERS  在发送给源站请求之前被触发。
* HOOK_READ_RESPONSE_HEADERS  在源站响应头信息到达之后触发
* HOOK_SEND_RESPONSE_HEADERS  在发送给客户端响应之前触发
* HOOK_OS_DNS  在DNS查询之后触发
* HOOK_READ_REQUEST_HEADERS  在头信息被读取出来后触发
* HOOK_READ_CACHE_HEADERS  在缓存的头信息被读取出来之后触发  
* HOOK_CACHE_LOOKUP_COMPLETE 在缓存读取完之后触发
* HOOK_SELECT_ALT 多副本选择的时候触发

具体捕获到对应的事件后，可以通过Transaction &transaction 对象进行设置 和更改，具体的内容如下：
![](cpp_api_image/UMLClassDiagram-Transaction.png) 
该类 可做如下更改


Global插件编写流程：

1. TSPluginInit 函数为全局插件的入口，在函数中进行：创建Global插件类型  new GlobalHookPlugin()；
2. 可在GlobalHookPlugin构造函数中，注册对应的事件：registerHook(HOOK_READ_REQUEST_HEADERS_PRE_REMAP)；
3. 找到对应的事件的虚函数，进行对应的实现；

```cpp
//构造函数，GlobalPlugin 基类 初始化。
GlobalPlugin::GlobalPlugin(	 ignore_internal_transactions)
{
  utils::internal::initTransactionManagement();
  state_ = new GlobalPluginState(this, ignore_internal_transactions);
  TSMutex mutex = NULL;
  // 初始化，创建continuation ，绑定在state_->cont_ 中，在registerHook 使用
  state_->cont_ = TSContCreate(handleGlobalPluginEvents, mutex);
  TSContDataSet(state_->cont_, static_cast<void *>(state_));
}

//客户调用registerHook 注册相关事件，continuation 已经在构造函数中创建了，然后使用TSHttpHookAdd挂载该事件。
void
GlobalPlugin::registerHook(Plugin::HookType hook_type)
{
  TSHttpHookID hook_id = utils::internal::convertInternalHookToTsHook(hook_type);
  TSHttpHookAdd(hook_id, state_->cont_);
  LOG_DEBUG("Registered global plugin %p for hook %s", this, HOOK_TYPE_STRINGS[hook_type].c_str());
}

//continuation 回调函数，当有事件发生的时候会触发基类的这个函数。
// 然后通过 utils::internal::invokePluginForEvent(state->global_plugin_, txn, event); 通知GlobalPlugin里面的相关函数。
static int
handleGlobalPluginEvents(TSCont cont, TSEvent event, void *edata)
{
  TSHttpTxn txn = static_cast<TSHttpTxn>(edata);
  GlobalPluginState *state = static_cast<GlobalPluginState *>(TSContDataGet(cont));
  if (state->ignore_internal_transactions_ && (TSHttpTxnIsInternal(txn) == TS_SUCCESS)) {
    LOG_DEBUG("Ignoring event %d on internal transaction %p for global plugin %p", event, txn, state->global_plugin_);
    TSHttpTxnReenable(txn, TS_EVENT_HTTP_CONTINUE);
  } else {
    LOG_DEBUG("Invoking global plugin %p for event %d on transaction %p", state->global_plugin_, event, txn);
    utils::internal::invokePluginForEvent(state->global_plugin_, txn, event);
  }
  return 0;
}

//如果通知这些虚函数呢？？？？静态函数
void
utils::internal::invokePluginForEvent(TransactionPlugin *plugin, TSHttpTxn ats_txn_handle, TSEvent event)
{
//锁----！！！
  ScopedSharedMutexLock scopedLock(plugin->getMutex());
  ::invokePluginForEvent(static_cast<Plugin *>(plugin), ats_txn_handle, event);
}

void inline invokePluginForEvent(Plugin *plugin, TSHttpTxn ats_txn_handle, TSEvent event)
{
//事件翻译后，转换成cpp 的数据结构 反馈给外面
  Transaction &transaction = utils::internal::getTransaction(ats_txn_handle);
  switch (event) {
  case TS_EVENT_HTTP_PRE_REMAP:
    plugin->handleReadRequestHeadersPreRemap(transaction);
    break;
  case TS_EVENT_HTTP_POST_REMAP:
    plugin->handleReadRequestHeadersPostRemap(transaction);
    break;
  case TS_EVENT_HTTP_SEND_REQUEST_HDR:
    plugin->handleSendRequestHeaders(transaction);
    break;
  case TS_EVENT_HTTP_READ_RESPONSE_HDR:
    plugin->handleReadResponseHeaders(transaction);
    break;
  case TS_EVENT_HTTP_SEND_RESPONSE_HDR:
    plugin->handleSendResponseHeaders(transaction);
    break;
  case TS_EVENT_HTTP_OS_DNS:
    plugin->handleOsDns(transaction);
    break;
  case TS_EVENT_HTTP_READ_REQUEST_HDR:
    plugin->handleReadRequestHeaders(transaction);
    break;
  case TS_EVENT_HTTP_READ_CACHE_HDR:
    plugin->handleReadCacheHeaders(transaction);
    break;
  case TS_EVENT_HTTP_CACHE_LOOKUP_COMPLETE:
    plugin->handleReadCacheLookupComplete(transaction);
    break;
  case TS_EVENT_HTTP_SELECT_ALT:
    plugin->handleSelectAlt(transaction);
    break;

  default:
    assert(false); /* we should never get here */
    break;
  }
}

(gdb) bt
#0  ServerResponsePlugin::handleSendResponseHeaders (this=0x114d8a0, transaction=...) at ServerResponse.cc:77
#1  0x00007fffefbde44c in (anonymous namespace)::handleGlobalPluginEvents (cont=<value optimized out>, 
    event=TS_EVENT_HTTP_SEND_RESPONSE_HDR, edata=0x7fffedc4d080) at GlobalPlugin.cc:58
#2  0x0000000000599aa5 in HttpSM::state_api_callout (this=0x7fffedc4d080, event=<value optimized out>, data=<value optimized out>)
    at HttpSM.cc:1382
#3  0x000000000059edc0 in HttpSM::state_api_callback (this=0x7fffedc4d080, event=60000, data=0x0) at HttpSM.cc:1273
#4  0x00000000004cef28 in TSHttpTxnReenable (txnp=0x7fffedc4d080, event=TS_EVENT_HTTP_CONTINUE) at InkAPI.cc:5671
#5  0x00007fffefbdf7e5 in (anonymous namespace)::handleTransactionEvents (cont=<value optimized out>, event=<value optimized out>, 
    edata=0x7fffedc4d080) at utils_internal.cc:92
#6  0x0000000000599aa5 in HttpSM::state_api_callout (this=0x7fffedc4d080, event=<value optimized out>, data=<value optimized out>)
    at HttpSM.cc:1382
#7  0x000000000059f94f in HttpSM::set_next_state (this=0x7fffedc4d080) at HttpSM.cc:7164
#8  0x000000000059880f in HttpSM::handle_api_return (this=0x7fffedc4d080) at HttpSM.cc:1526
#9  0x0000000000599c93 in HttpSM::state_api_callout (this=0x7fffedc4d080, event=<value optimized out>, data=0x0) at HttpSM.cc:1464
#10 0x000000000059f410 in HttpSM::set_next_state (this=0x7fffedc4d080) at HttpSM.cc:6955
#11 0x000000000059880f in HttpSM::handle_api_return (this=0x7fffedc4d080) at HttpSM.cc:1526
#12 0x0000000000599c93 in HttpSM::state_api_callout (this=0x7fffedc4d080, event=<value optimized out>, data=0x0) at HttpSM.cc:1464
#13 0x000000000059edc0 in HttpSM::state_api_callback (this=0x7fffedc4d080, event=60000, data=0x0) at HttpSM.cc:1273
#14 0x00000000004cef28 in TSHttpTxnReenable (txnp=0x7fffedc4d080, event=TS_EVENT_HTTP_CONTINUE) at InkAPI.cc:5671
#15 0x00007fffefbdf7e5 in (anonymous namespace)::handleTransactionEvents (cont=<value optimized out>, event=<value optimized out>, 
    edata=0x7fffedc4d080) at utils_internal.cc:92
```

Remap 插件编写流程：

1. TSRemapNewInstance 为remap 插件的入口，在函数中进行：创建remap插件类型 new MyRemapPlugin(instance_handle);
2. 为MyRemapPlugin 类实现doRemap方法