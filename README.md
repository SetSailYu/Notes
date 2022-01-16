#  开发笔记


# 一、 .net6 core 微服务搭建笔记

## 1、添加Oeclot
#### ***Program.cs***
    // 添加配置文件
    builder.Host.ConfigureAppConfiguration((hostingContext, config) =>
    {
        config.AddJsonFile("oeclot.json",
                            optional: true,
                            reloadOnChange: true);
    });

    // 添加Oeclot
    builder.Services.AddOcelot()
                    .AddConsul();

    // 添加Ocelot
    app.UseOcelot();
#### ***oeclot.json***
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






