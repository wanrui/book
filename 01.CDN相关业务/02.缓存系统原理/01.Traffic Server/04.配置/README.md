# Configuration Files
## cache.config

根据目标地址、客户端、URL 等信息配置Traffic server  服务器对缓存对象的存储方式。

## congestion.config
定义
Defines network conditions under which clients will receive retry messages instead of Traffic Server contacting origin servers.
## hosting.config
Allows Traffic Server administrators to assign cache volumes to specific origin servers or domains.
## ip_allow.config
Controls access to the Traffic Server cache based on source IP addresses and networks including limiting individual HTTP methods.
## log_hosts.config
Defines origin servers for which separate logs should be maintained.
## logging.config
Defines custom log file formats, filters, and processing options.
## metrics.config
Defines custom dynamic metrics using Lua scripting.
## parent.config
Configures parent proxies in hierarchical caching layouts.
## plugin.config
Control runtime loadable plugins available to Traffic Server, as well as their configurations.
records.config
Contains many configuration variables affecting Traffic Server operation, both the local node as well as a cluster in which the node may be a member.
## remap.config
Defines mapping rules used by Traffic Server to properly route all incoming requests.
## splitdns.config
Configures DNS servers to use under specific conditios.
## ssl_multicert.config
Configures Traffic Server to use different server certificates for SSL termination when listening on multiple addresses or when clients employ SNI.
## storage.config
Configures all storage devices and paths to be used for the Traffic Server cache.
## vaddrs.config
Deprecated file formerly used for cluster configuration.
## volume.config
Defines cache space usage by individual protocols.