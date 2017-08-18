# 01.几个平台架构实现方式不同

* 015平台nginx\(lua+redis实现部分策略控制\) +vcache
* 061平台nginx） +vcache
* gsa.tbcache+haproxy +nds+rcs

关注定时脚本的一些操作，日志时间上的转换。

# 02.ats某些参数和工具

* 只缓存服务器给出明确指令的返回结果
* 不缓存任何带cookie的返回结果
* 不缓存任何&gt;400的返回结果
* 不缓存任何Body是空的返回结果
* 不缓存Range返回的返回结果

```
 # cache responses to cookies has 5 options:
   #   0 - do not cache any responses to cookies
   #   1 - cache for any content-type
   #   2 - cache only for image types
   #   3 - cache for all but text content-types
   #   4 - cache for all but text content-types except OS response
   #       without "Set-Cookie" or with "Cache-Control: public"
   # See also cache-responses-to-cookies in cache.config.
CONFIG proxy.config.http.cache.cache_responses_to_cookies INT 1


   # required headers: three options:
   #   0 - No required headers to make document cachable
   #   1 - "Last-Modified:", "Expires:", or "Cache-Control: max-age" required
   #   2 - explicit lifetime required, "Expires:" or "Cache-Control: max-age"
CONFIG proxy.config.http.cache.required_headers INT 0


   # Enabling this setting allows the proxy to cache empty documents. This currently
   # requires that the response has a Content-Length: header, with a value of "0".
CONFIG proxy.config.http.cache.allow_empty_doc INT 1



   #############################
   # negative response caching #
   #############################
CONFIG proxy.config.http.negative_caching_enabled INT 0
CONFIG proxy.config.http.negative_caching_lifetime INT 30


CONFIG proxy.config.http.cache.range.lookup INT 1


   #################
   # cache control #
   #################
CONFIG proxy.config.http.cache.http INT 1
   # Enabling this setting allows the proxy to cache empty documents. This currently
   # requires that the response has a Content-Length: header, with a value of "0".
CONFIG proxy.config.http.cache.allow_empty_doc INT 1
CONFIG proxy.config.http.cache.ignore_client_no_cache INT 1
CONFIG proxy.config.http.cache.ims_on_client_no_cache INT 1
CONFIG proxy.config.http.cache.ignore_server_no_cache INT 0
CONFIG proxy.config.http.cache.ignore_client_cc_max_age INT 1
CONFIG proxy.config.http.normalize_ae_gzip INT 1


   # The maximum number of alternates that are allowed for any given URL.
   # It is not possible to strictly enforce this if the variable
   #   'proxy.config.cache.vary_on_user_agent' is set to 1.
   # The default value for 'proxy.config.cache.vary_on_user_agent' is 0.
   # (0 disables the maximum number of alts check)
CONFIG proxy.config.cache.limits.http.max_alts INT 10

CONFIG proxy.config.http.cache.ignore_accept_mismatch INT 2
CONFIG proxy.config.http.cache.ignore_accept_language_mismatch INT 2
CONFIG proxy.config.http.cache.ignore_accept_charset_mismatch INT 2
```

# 03.插件的逻辑


## 3.1.事件时序图
![](/assets/01.事件时序图.png)
## 3.2.重写重存
![](/assets/02.重写重存.png)
## 3.3. transform插件
![](/assets/03.transform插件.png)
## 3.4. 302插件
![](/assets/04.302插件.png)
## 3.5. cache range
![](/assets/05.cacherange.png)







# 04.测试

见wiki

