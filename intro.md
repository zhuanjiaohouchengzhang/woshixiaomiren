## RPC产品相关

Apus是小米工程团队为小米和生态链提供的服务化的RPC框架。该框架是基于Facebook Swift开发的分布式服务框架，实现高可扩展性的RPC调用同时提供了服务发现及服务治理能力，支持自定义注册中心、分组策略、负载路由策略，以及监控指标统计等功能。

Apus对标业界主流RPC框架，如Dubbo，Motan，Finagle和gRPC等。
业界 RPC 框架大致分为两类，一种是偏重服务治理，另一种侧重跨语言调用。
服务治理型的RPC框架典型的是Dubbo，Motan等，提供了强大的服务管理能力，缺点是需要使用特定语言编程，跨语言调用支持难度大；
跨语言调用的RPC框架典型的有Thrift，gRPC等，提供了跨语言调用支持，但服务管理能力较弱，没有服务发现和注册的模块。

Apus底层使用Thrift，支持不同语言通过Thrift协议调用，目前服务注册和治理支持已Java，多语言的SDK已在开发过程中。在此之上我们实现了服务治理中心，支持分组策略和负载路由策略，使之在支持跨语言调用的同时仍然具备强大的服务管理能力。小米的RPC框架已经在内部使用多年，经过小米众多业务千锤百炼，积累了大量实践，并且在大量实践的基础上，我们能提供更及时，更精准的技术支持。

除此之外，Apus还有极高的扩展性，比如Apus内置的监控模块支持了Open-Falcon，在金山云可以很方便的看到调用方法、次数和异常次数等监控指标：  
client监控项  
```
thriftcli-${ifaceClassName}-${methodName}：client 调用 methodname 方法的时间，次数
thriftcli-${ifaceClassName}-${methodName}_fail：namethodName 调用失败次数
thriftcli-${ifaceClassName}-exception-${exceptionName}：exceptionName 异常的调用次数
```
server监控项
```
method/${methodName}：methodName 调用次数，时间
method/ALL_THRIFT_METHODS：所有 method 调用次数，时间
method/${methodName}_fail：methodName 调用失败次数
exception/${exceptionName}: exception 次数 
method/ALL_THRIFT_METHODS_FAIL: 所有 method 调用失败次数
```
如果你想实现其他监控系统的集成，比如AWS的CloudWatch也非常简单，只需实现以下接口即可：  
Metrics
```
public void count(Counter counter)
public void countDuration(Method method, long duration): 统计方法执行时间，次数
public void countException(Method method, Throwable throwable)：统计异常次数
```

Counter
```
private String name：Counter name，表示统计的一个指标 Key
private long longValue：调用次数
private String tags：标签
private long time：执行时间
....
```
简单即是美，Apus使用起来非常简单。仅需三步就可以让一个服务端应用跑起来：  
1.接口定义：
```java
public class EchoServer {
    @ThriftService("EchoServer")
    public interface Iface {
        @ThriftMethod("echo")
        String echo(String input);
    }

    // 异步接口，如果不需要可以不定义
    @ThriftService("EchoServer")
    public interface AsyncIface {
        @ThriftMethod("echo")
        ListenableFuture<String> echo(String input);    
    }
}
```
2.接口实现:
```java
public class EchoServerImpl implements EchoServer.Iface {
    @Override
    public String echo(String input) {
        return input;
    }
}
```
3.服务启动：
```java
public class Server {
    public static void main(String[] args) {
        ServerConfig<EchoServerImpl> config = new ServerConfig.Builder<>(EchoServerImpl.class)
                .registryFactory(new ServiceCenterRegistryFactory(EchoServerImpl.class))
                .port(12346).build();
        Bootstrap<EchoServerImpl> bootstrap = new Bootstrap<>(config);
        bootstrap.start();
    }
}
```
如上，我们完成了一个简单的服务端应用，接下来我们实现一个客户端对其进行调用  
```java
public class Client {
    public static void main(String[] args) throws InterruptedException {
        // client 调用同步接口
        String request = "Hello Xiaomi!";
        ClientConfig<EchoServer.Iface> syncClientConfig =
                new ClientConfig.ClientConfigBuilder<>(EchoServer.Iface.class)
                        .registryFactory(new ZookeeperRegistryFactory(EchoServer.Iface.class))
                        .build();
        EchoServer.Iface newSyncClient = new SwiftClientManager<>(syncClientConfig).createClient();
        String response = newSyncClient.echo(request);
        System.out.println(response);

        // client 调用异步接口
        String asyncRequest = "Hello Xiaomi!(async)";
        ClientConfig<EchoServer.AsyncIface> asyncIfaceClientConfig =
                new ClientConfig.ClientConfigBuilder<>(EchoServer.AsyncIface.class)
                        .registryFactory(new ZookeeperRegistryFactory(EchoServer.Iface.class))
                        .build();
        EchoServer.AsyncIface newAsyncClinet = new SwiftClientManager<>(asyncIfaceClientConfig).createClient();
        ListenableFuture<String> future = newAsyncClinet.echo(asyncRequest);
        Futures.addCallback(future, new FutureCallback<String>() {
            @Override
            public void onSuccess(String result) {
                System.out.println(result);
            }

            @Override
            public void onFailure(Throwable t) {
                t.printStackTrace();
            }
        });
        Thread.sleep(1000);
    }
}

```
如上所示，我们完成了构建一个完整的简单服务的所有工作，详细的用户文档参见[使用指南](http://docs.api.xiaomi.com/apus/)