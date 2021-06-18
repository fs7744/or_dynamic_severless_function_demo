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

由于 lua 的动态语言特点，我们可以比较方便做到动态插件机制。

首先我们来了解这一切的基石：lua 模块加载机制。

### lua 模块加载机制

#### 一个模块是什么样？

例如： xxxmodule.lua 文件内容
``` lua
local module = {} -- 注意不要使用全局变量，会造成变量污染，导致无法卸载模块
 
-- 定义一个函数
function module.func1()
    io.write("这是一个公有函数！\n")
end

return module
```

#### 如何加载模块？

Lua提供了一个名为require的函数用来加载模块。要加载一个模块，只需要简单地调用就可以了。例如：

``` lua
local a = require("xxxmodule")
a.func1() -- "这是一个公有函数！\n"
```

#### 到底是怎么工作的呢？

1. `require` 函数会在模块path列表搜索模块，openresty可以指定如下两种：
    * lua 库： `lua_package_path  "./?.lua;/usr/local/openresty/luajit/share/luajit-2.1.0-beta3/?.lua;";`
    * c 库： `lua_package_cpath "./?.so;/usr/local/lib/lua/5.1/?.so;";`

2. 找到模块文件之后，就会解析执行整个文件的内容（类似函数 `loadstring`），由于最后是return 模块变量，我们就可以使用这个变量的函数等等一切了

3. 如果开启了 `lua_code_cache on`，  `require` 函数会将第二步拿到的变量存在 `package.loaded` 这个table 中，达到缓存效果

#### 那么如何卸载呢？

非常简单，只需一句：

``` lua
package.loaded['xxxmodule'] = nil
```

### severless function simple demo

所以我们可以基于这样的动态机制，实现 severless function 或者动态插件机制

``` lua
http {
    default_type  application/json;
    lua_code_cache on;

    lua_package_path  "$prefix/deps/share/lua/5.1/?.lua;$prefix/deps/share/lua/5.1/?/init.lua;$prefix/src/?.lua;$prefix/src/?/init.lua;;./?.lua;/usr/local/openresty/luajit/share/luajit-2.1.0-beta3/?.lua;/usr/local/share/lua/5.1/?.lua;/usr/local/share/lua/5.1/?/init.lua;/usr/local/openresty/luajit/share/lua/5.1/?.lua;/usr/local/openresty/luajit/share/lua/5.1/?/init.lua;";
    lua_package_cpath "$prefix/deps/lib64/lua/5.1/?.so;$prefix/deps/lib/lua/5.1/?.so;;./?.so;/usr/local/lib/lua/5.1/?.so;/usr/local/openresty/luajit/lib/lua/5.1/?.so;/usr/local/lib/lua/5.1/loadall.so;";


    # 简单模拟模块
    init_by_lua_block {
        MockPackages = {} 
    }
    server {
        listen       8222;
        server_name  localhost;

        location /add {
            
            # 比如替换为 request body 去做模块创建，这里为了简单就用写死的代码来模拟
            # 内容为通过 loadstring 转换 lua code 字符串为函数
            # 并将函数结果 当前时间存在全局变量中
            access_by_lua_block {
                local lua_src = [[
                ngx.update_time()
                return tostring(ngx.now())
                ]]
                local f, e = loadstring(lua_src, "module xxxmodule")
                MockPackages['xxxmodule'] = f()
                ngx.say('add function success')
            }
        }
        
        location /run {

            # 这里获取缓存结果并输出出来
            access_by_lua_block {
                if MockPackages['xxxmodule'] then
                    ngx.say(MockPackages['xxxmodule'])
                else 
                    ngx.say('no function')
                end
            }
        }
    }
}
```

启动并测试 `mkdir -p logs && /usr/bin/openresty -p ./ -c nginx.conf -g 'daemon off;'`

1. call http://127.0.0.1:8222/run
    return no function

2. call http://127.0.0.1:8222/add
    return add function success

3. call http://127.0.0.1:8222/run
    return 1624022896.703

4. call http://127.0.0.1:8222/add
    return add function success

5. call http://127.0.0.1:8222/run
    return 1624022918.674

可以看到值已经被改变了
### 这种severless function demo的问题

* 管理以及定位问题

    实际环境会有很多机器实例，对应的severless function 在哪几台机器哪几个nginx中的哪些worker 进程上加载，加载多久， 需要完整规划方案

* 资源隔离

    更大的问题是：
    * 由于多个函数会同在一个worker 进程，无论性能和资源都会收到相互影响
    * 别人可以在其中轻松加入恶意代码