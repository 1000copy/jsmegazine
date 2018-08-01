## node+Nginx联手

作者：reco

node.js 做服务器？

久经考验的nginx 前置顶住压力，后面多个node服务器完成业务支撑，这样的做法是放心的，是走正道的。

这里要做一个实验：

    1. 一个nginx作为前台的服务器
    2. 全部请求，经过负载均衡，尽可能均衡的分步到后面的2台node服务器

# 准备node 

首先启动两台node，分别监听3000，3001端口。为了区分，helloworld会带一个端口返回，通知客户端，以便区别是谁在提供服务。

###node.js server

    $cat 1.js 
    require('http').createServer(function (request, response) {  
      response.end('hello world\n'+process.argv[2]);  
    }).listen(process.argv[2]);  
    $node 1.js 3000
    $node 1.js 3001

### 验证node 服务器启动

    λ curl localhost:3000
    hello world
    3000
    λ curl localhost:3001
    hello world
    3001

# 准备nginx 服务器的配置。

要点是通过upstream 指令把两个node服务器打成一个服务器池。然后通过location指令，要求全部根目录请求转发到这个池内。池会对它之内的服务器做负载均衡。

### nginx conf

    worker_processes  1;
    events {
        worker_connections  1024;
    }
    http {
         upstream node_server_pool {
           server localhost:3001 max_fails=1;
           server localhost:3000 max_fails=1;
        }
        server{
          listen       80;
          server_name localhost;
          location /
           {
            proxy_pass http://node_server_pool;
           }
        }
    }
        
# 模拟客户端访问

我用curl多次访问nginx服务器，可以通过返回的字符串知道服务器。你看，真的可以负载均衡:一会儿helloworld 3000，一会儿helloworld 3001。
        
          
    λ curl localhost
    hello world
    3000
    
    λ curl localhost
    hello world
    3001
    
    λ curl localhost
    hello world
    3001
    
    λ curl localhost
    hello world
    3000
    
你看，ngnix是公平的。哪怕就一个客户的多次访问都会换着服务器来。

# 参考

Node.js + Nginx - What now? - Stack Overflow - http://stackoverflow.com/questions/5009324/node-js-nginx-what-now
为高负载网络优化 Nginx 和 Node.js - 技术翻译 - 开源中国社区 - http://www.oschina.net/translate/optimising-nginx-node-js-and-networking-for-heavy-workloads
让node.js充分利用多核服务器的性能，运用nginx做反向代理和负载均衡 - snoopyxdy的日志 - 网易博客 - http://snoopyxdy.blog.163.com/blog/static/60117440201172954648952/
