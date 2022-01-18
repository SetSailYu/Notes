#  开发笔记


# 一、 .net6 core 微服务搭建笔记

## 1、添加Ocelot
#### ***Program.cs***
    // 添加配置文件
    builder.Host.ConfigureAppConfiguration((hostingContext, config) =>
    {
        config.AddJsonFile("ocelot.json",
                            optional: true,
                            reloadOnChange: true);
    });

    // 添加Ocelot
    builder.Services.AddOcelot()
                    .AddConsul();

    // 添加Ocelot
    app.UseOcelot();
#### ***ocelot.json***
    //********************************************单地址全匹配*****************************************************
    {
      "Routes": [
        {
          "DownstreamPathTemplate": "/{url}", //服务地址-------url变量
          "DownstreamScheme": "http",
          "DownstreamHostAndPorts": [
            {
              "Host": "localhost",
              "Port": 5726 //服务端口
            }
          ],
          "UpstreamPathTemplate": "/T5726/{url}", //网关地址----url变量     //冲突的还可以加重Priority
          "UpstreamHttpMethod": [ "Get", "Post" ]
        }
      ]
    }
    //********************************************单地址多实例负载均衡+Cousul*****************************************************
    {
        "Routes": [
            {
            "DownstreamPathTemplate": "/api/{url}", //服务地址-------url变量
            "DownstreamScheme": "http",
            "UpstreamPathTemplate": "/T/{url}", //网关地址----url变量     //冲突的还可以加重Priority
            "UpstreamHttpMethod": [ "Get", "Post" ],
            "UseServiceDiscovery": true,
            "ServiceName": "ConsulService", //consul服务名称
            "LoadBalancerOptions": {
                "Type": "RoundRobin" //轮询    LeastConnection-最少链接数的服务器    NoLoadBalance不负载均衡
            }
            }
        ],
        "GlobalConfiguration": {
            //"BaseUrl": "http://localhost:13763" //网关对外地址
            "ServiceDiscoveryProvider": {
            "Host": "192.168.3.230",
            "Port": 8500,
            "Type": "Consul" //由Consul提供服务发现，每次请求去consul
            }
            //"ServiceDiscoveryProvider1": {
            //  "Host": "192.168.3.230",
            //  "Port": 8500,
            //  "Type": "Consul", //由Consul提供服务发现，每次请求去consul
            //  "PollingInterval": 1000, //轮询consul，频率毫秒----down掉是不知道的
            //  //"Token": "footoken"    //需要ACl的话
            //}

        }
    }
    //********************************************完整版配置实例*****************************************************
    {
      "Routes": [
        {
          //服务名称，开启服务发现时需要配置
          "ServiceName": "",
          //下游服务路由模板
          "DownstreamPathTemplate": "/{url}",
          //下游服务http schema
          "DownstreamScheme": "http",
          //下游服务的地址，如果使用LoadBalancer的话这里可以填多项
          "DownstreamHostAndPorts": [
            {
              "Host": "192.168.1.205",
              "Port": 12001
            },
            {
              "Host": "192.168.1.205",
              "Port": 11001
            }
          ],
          "UpstreamPathTemplate": "/{url}",
          //上游请求http方法，可使用数组
          "UpstreamHttpMethod": [
            "GET",
            "POST",
            "PUT",
            "DELETE",
            "OPTIONS"
          ],
          /**
           * 负载均衡的算法：
           * LeastConnection        – 跟踪哪些服务正在处理请求，并将新请求发送到具有最少现有请求的服务。算法状态没有分布在Ocelot集群中。
           * RoundRobin             – 遍历可用服务并发送请求。算法状态没有分布在Ocelot集群中。
           * NoLoadBalancer          – 从配置或服务发现中获取第一个可用服务
           * CookieStickySessions   - 使用cookie将所有请求粘贴到特定服务器
           */
          "LoadBalancerOptions": {
            "Type": "RoundRobin"
            //以下配置再设置了 CookieStickySessions 后需要开启
            //用于粘性会话的cookie的密钥
            //"Key": "ASP.NET_SessionId",
            //会话被阻塞的毫秒数
            //"Expiry": 1800000
          },
          //缓存
          "FileCacheOptions": {
            "TtlSeconds": 15
          },
          //限流
          "RateLimitOptions": {
            //包含客户端白名单的数组。这意味着该阵列中的客户端将不受速率限制的影响
            "ClientWhitelist": [],
            //是否启用端点速率限制
            "EnableRateLimiting": true,
            //指定限制所适用的期间，例如1s，5m，1h，1d等。如果在该期间内发出的请求超出限制所允许的数量，则需要等待PeriodTimespan过去，然后再发出其他请求
            "Period": "1s",
            //指定可以在一定秒数后重试
            "PeriodTimespan": 1,
            //指定客户端在定义的时间内可以发出的最大请求数
            "Limit": 1
          },
          //熔断
          "QoSOptions": {
            //允许多少个异常请求
            "ExceptionsAllowedBeforeBreaking": 3,
            //熔断的时间，单位为毫秒
            "DurationOfBreak": 1000,
            //如果下游请求的处理时间超过多少则自如将请求设置为超时
            "TimeoutValue": 5000
          },
          "HttpHandlerOptions": {
            //是否开启路由追踪
            "UseTracing": true
          }
        }
      ],
      "GlobalConfiguration": {
        //外部暴露的Url
        "BaseUrl": "http://localhost:13763",
        //限流扩展配置
        "RateLimitOptions": {
          //指定是否禁用X-Rate-Limit和Retry-After标头
          "DisableRateLimitHeaders": false,
          //当请求过载被截断时返回的消息
          "QuotaExceededMessage": "Oh,Oops!",
          //当请求过载被截断时返回的http status
          "HttpStatusCode": 4421,
          //用来识别客户端的请求头，默认是 ClientId
          "ClientIdHeader": "ClientId"
        }
      }
    }
## 2、Docker安装Consul
>   转载自：[https://www.cnblogs.com/summerday152/p/14013439.html](https://www.cnblogs.com/summerday152/p/14013439.html)
### 2.1 拉取Consul镜像
    $ docker pull consul # 默认拉取latest
    $ docker pull consul:1.6.1 # 拉取指定版本
### 2.2 安装并运行
    docker run -d -p 8500:8500 --restart=always --name=consul consul:latest agent -server -bootstrap -ui -node=1 -client='0.0.0.0'
> * agent: 表示启动 Agent 进程。
> * server：表示启动 Consul Server 模式
> * client：表示启动 Consul Cilent 模式。
> * bootstrap：表示这个节点是 Server-Leader ，每个数据中心只能运行一台服务器。技术角度上讲 Leader 是通过 Raft 算法选举的，但是集群第一次启动时需要一个引导 Leader，在引导群集后，建议不要使用此标志。
> * ui：表示启动 Web UI 管理器，默认开放端口 8500，所以上面使用 Docker 命令把 8500 端口对外开放。
> * node：节点的名称，集群中必须是唯一的，默认是该节点的主机名。
> * client：consul服务侦听地址，这个地址提供HTTP、DNS、RPC等服务，默认是127.0.0.1所以不对外提供服务，如果你要对外提供服务改成0.0.0.0
> * join：表示加入到某一个集群中去。 如：-json=192.168.0.11。
### 2.3 关闭防火墙或开放8500端口
#### *【查看防火墙是否开启】*
    $ systemctl status firewalld
#### *【开启或关闭防火墙】*
    $ systemctl start firewalld
    $ systemctl stop firewalld
#### *【查看所有开启的端口】*
    $ firewall-cmd --list-ports
#### *【开启80端口】*
    $ firewall-cmd --zone=public --add-port=2181/tcp --permanent
#### *【重启防火墙，使其生效】*
    $ firewall-cmd --reload

### *如果是阿里云服务器，需要设置安全组：*
>  来到实例管理页面，点击更多，点击网络和安全组，点击安全组配置。
> 
>  ![图片1](https://img2020.cnblogs.com/blog/1771072/202011/1771072-20201120190901910-220933991.png)
> 
>  点击配置规则。
> 
>  ![图片2](https://img2020.cnblogs.com/blog/1771072/202011/1771072-20201120190907504-621983589.png)
> 
>  点击添加安全组规则，端口范围改为8500。
> 
>  ![图片3](https://img2020.cnblogs.com/blog/1771072/202011/1771072-20201120190912827-800573654.png)

### 2.4 测试访问
> 
>    ![图片4](https://cdn.jsdelivr.net/gh/SetSailYu/MyGallery/img/202201182103687.png)
>  1.  services：放置服务
>  2.  nodes：放置consul节点
>  3.  key/value：放置一些配置信息
>  4.  dc1：配置数据中心
> 
>  * 官方文档： [https://www.consul.io/docs](https://www.consul.io/docs)

