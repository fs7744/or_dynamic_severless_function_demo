## OpenResty 是什么？

OpenResty 实际是将代理功能的 nginx 核心 + 各类nginx功能模块组合打包在一起，从而构成了一个全功能 的web 应用服务器。

第一代的 OpenResty 最早是雅虎中国的一个公司项目，起步于 2007 年 10 月。当时兴起了 OpenAPI 的热潮，用于满足各种 Web Service 的需求，在公司领导的支持下，基于 Perl 和 Haskell 实现了开源的 OpenResty。最初的定位是服务于公司外的开发者，像其他的 OpenAPI 那样，但后来越来越多地是为雅虎中国的搜索产品提供内部服务。这是第一代的 OpenResty，当时的想法是，提供一套抽象的 web service，能够让用户利用这些 web service 构造出新的符合他们具体业务需求的 Web Service 出来，所以有些“meta web service”的意味，包括数据模型、查询、安全策略都可以通过这种 meta web service 来表达和配置。同时这种 web service 也有意保持 REST 风格。

而现在大家熟知的 OpenResty实际是第二代，一般称之为ngx_openresty。章亦春在加入淘宝数据部门的量子团队之后，决定对 OpenResty 进行重新设计和彻底重写，和他的同事王晓哲一起设计了第二代的 OpenResty。在王晓哲的提议下，选择基于 nginx 和 lua 进行开发, 主要因为lua的小巧，灵活脚本扩展，以及C/C++便捷互通能力。

ngx_openresty 目前有两大应用目标：

1. 通用目的的 web 应用服务器。在这个目标下，现有的 web 应用技术都可以算是和 OpenResty 或多或少有些类似，比如 Nodejs, PHP 等等。ngx_openresty 的性能（包括内存使用和 CPU 效率）算是最大的卖点之一。
2. Nginx 的脚本扩展编程，用于构建灵活的 Web 应用网关和 Web 应用防火墙。有些类似的是 NetScaler。其优势在于 Lua 编程带来的巨大灵活性。

## OpenResty 中提供的 lua 扩展点
OpenResty 处理流程参考下图：
![执行阶段概念](https://moonbingbing.gitbooks.io/openresty-best-practices/content/images/openresty_phases.png)

* init_by_lua*: 自定义初始化，比如全局公用变量等
* init_worker_by_lua*: 初始化worker进程独有的变量等
* set_by_lua*: 流程分支处理判断变量初始化
* rewrite_by_lua*: 转发、重定向、缓存等功能(例如特定请求代理到外网)
* access_by_lua*: IP 准入、接口权限等情况集中处理(例如配合  iptable 完成简单防火墙)
* content_by_lua*: 内容生成
* header_filter_by_lua*: 响应头部过滤处理(例如添加头部信息)
* body_filter_by_lua*: 响应体过滤处理(例如完成应答内容统一成大写)
* log_by_lua*: 会话完成后本地异步完成日志记录(日志可以记录在本地，还可以同步到其他机器)

## OpenResty 如何实现动态插件？

lua 代码如何加载

demo

severless function