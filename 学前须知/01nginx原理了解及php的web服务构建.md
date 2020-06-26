# nginx原理了解及php的web服务构建
## 什么是nginx?
>Nginx是一款自由的、开源的、高性能的HTTP服务器和反向代理服务器；同时也是一个IMAP、POP3、SMTP代理服务器；Nginx可以作为一个HTTP服务器进行网站的发布处理，另外nginx可以作为反向代理进行负载均衡的实现。(俄国人写的)
### 正向代理与反向代理
##### 正向代理
>比如我们想要访问facebook，但是因为国内的网络环境我们是访问不了的，我们就会去使用一些翻墙工具，帮助我们访问facebook，那么翻墙工具背后实际上就是一个可以访问国外网站的代理服务器，我们将请求发送给代理服务器，代理服务器去访问国外的网站，然后将访问到的数据传递给我们

>上述这样的代理模式称为正向代理，正向代理最大的特点是客户端非常明确要访问的服务器地址；服务器只清楚请求来自哪个代理服务器，而不清楚来自哪个具体的客户端；正向代理模式屏蔽或者隐藏了真实客户端信息

##### 反向代理
>明白了什么是正向代理，我们继续看关于反向代理的处理方式，举例如某宝网站，每天同时连接到网站的访问人数已经爆表，单个服务器远远不能满足人民日益增长的购买欲望了，此时就出现了一个大家耳熟能详的名词：分布式部署；也就是通过部署多台服务器来解决访问人数限制的问题；某宝网站中大部分功能也是直接使用nginx进行反向代理实现的

>多个客户端给服务器发送的请求，Nginx服务器接收到之后，按照一定的规则分发给了后端的业务处理服务器进行处理了。此时~请求的来源也就是客户端是明确的，但是请求具体由哪台服务器处理的并不明确了，Nginx扮演的就是一个反向代理角色

![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1587039851496&di=f2d6ea47a61ca5d2f532188cea0e98a3&imgtype=0&src=http%3A%2F%2F201903.oss-cn-hangzhou.aliyuncs.com%2Fjs%2F900301-65d94c0d393cdd5c5a5eeaa7f68034c3.png)
---
## 为什么选择nginx?
- 更快
>这表现在两个方面：一方面，在正常情况下，单次请求会得到更快的响应；另一方面，
在高峰期（如有数以万计的并发请求），Nginx可以比其他Web服务器更快地响应请求。
实际上

- 高扩展性
>Nginx的模块都是嵌入到二进制文件中执行的，无论官方发布的模块还是第三方模块都
是如此。这使得第三方模块一样具备极其优秀的性能，充分利用Nginx的高并发特性，因
此，许多高流量的网站都倾向于开发符合自己业务特性的定制模块

- 高可靠性
>高可靠性是我们选择Nginx的最基本条件，因为Nginx的可靠性是大家有目共睹的，很多家高流量网站都在核心服务器上大规模使用Nginx。Nginx的高可靠性来自于其核心框架代码的优秀设计、模块设计的简单性；另外，官方提供的常用模块都非常稳定，每个worker进程相对独立，master进程在1个worker进程出错时可以快速“拉起”新的worker子进程提供服务

- 低内存消耗
>一般情况下，10000个非活跃的HTTP Keep-Alive连接在Nginx中仅消耗2.5MB的内存，这是Nginx支持高并发连接的基础。

- 单机支持10万以上的并发连接
>这是一个非常重要的特性！随着互联网的迅猛发展和互联网用户数量的成倍增长，各大公司、网站都需要应付海量并发请求，一个能够在峰值期顶住10万以上并发请求的Server，
无疑会得到大家的青睐。理论上，Nginx支持的并发连接上限取决于内存，当然，能够及时地处理更多的并发请求，是与业务特点紧密相关的

- 热部署
>master管理进程与worker工作进程的分离设计，使得Nginx能够提供热部署功能，即可以在7×24小时不间断服务的前提下，升级Nginx的可执行文件。当然，它也支持不停止服务就更新配置项、更换日志文件等功能。
---
## web服务器的请求处理机制
````
<?php

<?php
/**
 * Create By: Will Yin
 * Date: 2020/4/16
 * Time: 17:49
 **/
class server{
    public $port;
    public $ip;
    protected $server;

    public function __construct($ip='0.0.0.0',$port)
    {
        $this->ip = $ip;
        $this->port = $port;
        $this->createScoket();
    }

    //创建套接字
    protected function createScoket(){
        $this->server = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);
        //复用还处于 TIME_WAIT(的工作进程)
        socket_set_option($this->server, SOL_SOCKET, SO_REUSEADDR, 1);
        //绑定套接字
        socket_bind($this->server,$this->ip,$this->port);
        //开始监听
        socket_listen($this->server);
    }

    //监听函数
    public function listen($callback)
    {
        if(!is_callable($callback)){
            throw new Exception('不是回调函数');
        }
        //利用死循环使其能够接收多个请求
        while(true){
            //等待客户端接入，返回的是客户端的连接,这里是阻塞的,只有请求过来才会继续执行下面代码
            $client = socket_accept($this->server);
            //读取客户端内容
            $buf = socket_read($client,1024);
            if(empty($this->checkRule("/GET\s(.*?)\sHTTP\/1.1/i",$buf))){
                socket_close($client);
                return;
            }
            //响应,call_user_func — 把第一个参数作为回调函数调用
            $response= call_user_func($callback,$buf); //回调$callback函数
            $this->response($response,$client);
            usleep(1000); //微妙为单位，1000000 微妙等于1秒
            socket_close($client);
        }
        socket_close($this->server);
    }
    //协议过滤
    protected function checkRule($reg,$buf){
        if(preg_match($reg,$buf,$match)){
            return $match;
        } else {
            return false;
        }
    }
    //客户端请求响应,返回数据给客户端
    protected function response($content,$client){
        //一个HTTP请求报文由四个部分组成：请求行、请求头部、空行、请求数据
        //HTTP响应报文也由三部分组成：响应行、响应头、响应体
        //响应行一般由协议版本、状态码及其描述组成 它们用空格分隔 比如 HTTP/1.1 200 OK
        //比如 GET /data/info.html HTTP/1.1
        $string = "HTTP/1.1 200 OK\r\n";
        $string .= "Content-Type: text/html;charset=utf-8\r\n";
        //长度后面必须添加 两组 \r\n\r\n
        $string .="Content-Length: ".strlen($content)."\r\n\r\n";
        socket_write($client,$string.$content);
    }
}


----------------------------------------------------------------------------------------------------------
<?php 

<?php
/**
 * Create By: Will Yin
 * Date: 2020/4/16
 * Time: 17:49
 **/
include 'web.php';
//http协议服务
$server = new Server('0.0.0.0', 9090);
////正式开始提供服务，启动服务
$server->listen(function ($buf) {
    //var_dump($buf);//打印连接用户的信息
    return 'hello this is will';
});

````
