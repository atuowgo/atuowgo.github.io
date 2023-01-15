# Dubbo Provider启动都做了什么


> 这节介绍Provider的启动过程，既服务发布。

先从Demo看起：

```
public class ApplicationApi {

    public static void main(String[] args) throws Exception {
        ServiceConfig<DemoServiceImpl> service = new ServiceConfig<>();
        service.setApplication(new ApplicationConfig("dubbo-demo-api-provider"));
        service.setRegistry(new RegistryConfig("zookeeper://127.0.0.1:2181"));
        service.setInterface(DemoService.class);
        service.setRef(new DemoServiceImpl());
        service.export();
        System.in.read();
    }
}
```

上面Demo主要做了几件事

1. 新建一个ServiceConfig对象，既服务配置对象
2. 为服务配置设置ApplicationConfig，既服务对应的项目配置
3. 为服务配置设置RegistryConfig，既注册中心配置
4. 指定服务关联的接口DemoService
5. 指定服务关联的实现DemoServiceImpl
6. 调用ServiceConfig的export方法暴露服务

这里重点关注export方法。

在这之前先说下服务暴露的主要流程：一个应用(```ApplicationConfig```)包含很多服务，一个服务(```ServiceConfig```)需要服务提供者(```ProviderConfig```)通过某种协议(```ProtocolConfig```)将自身的信息注册到注册中心(```RegistriesConfig```)，这就是服务暴露主要做的事情。

export方法主要围绕上面几个组件展开。

#### 1.设置默认配置

export会先检查当前环境是否准备就绪，主要是检查各组件配置，如ProviderConfig、ProtocolConfig、ApplicationConfig、ModuleConfig、RegistriesConfig、MonitorConfig等。对于某些为空的配置项则会使用默认值，如：

1. 创建ProviderConfig，设置prefix，如果有传则使用原值，没有则新建一个ProviderConfig，并设置prefix为```dubbo.provider```
2. 创建ProtocolConfig，设置prefix，如果有则使用原值，如果没有设置则新建一个ProtocolConfig，使用默认协议dubbo,并设置prefix为```dubbo.protocol.```

以上各配置项都继承自AbstractConfig

![](abstractConfig.png)

各配置的实例对象会托管到ConfigMananger，同时被ServiceConfig引用。各实例初始化后的主要值如下：

![](export.png)

PS:当id为空时，会设置默认值为```default```。

#### 2.暴露服务

ServiceConfig准备完配置后就开始进行服务的暴露，主要内容为```doExportUrls```方法。

![](doExportUrls.png)

该方法分为两个过程：

1. 加载所有的注册中心配置RegistriesConfig
2. 遍历所有的协议配置ProtocolConfig，使用各种协议将服务本身注册到所有的注册中心

#### 2.1 加载注册中心配置

加载ServiceConfig指定的所有注册中心配置对象，并以URL的形式返回。需要提及的一点时，Dubbo中为了增加统一的拦截处理，会把URL的协议改为```registry```，等拦截处理完后再修改回来。

![](changeRegistryProtocol.png)

如开头例子的URL会从

```
zookeeper://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-demo-api-provider&dubbo=2.0.2&pid=1204&timestamp=1606640163895
```

修改为：

```
registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-demo-api-provider&dubbo=2.0.2&pid=1204&registry=zookeeper&timestamp=1606640163895
```

PS：Dubbo使用URL用于在各个扩展点之间传递数据，一个标准的 URL 格式至多可以包含如下的几个部分

```
protocol://username:password@host:port/path?key=value&key=value
```

#### 2.2 注册服务

ServiceConfig会遍历所有的协议配置ProtocolConfig，对于每种协议，会将服务自身注册到所有配置的注册中心，过程如下：

![](doExportUrlsFor1Protocol.png)

前面都在准备URL的各种参数，以便于构造ServiceConfig对应的URL对象，开头例子Service对应的URL为：

```
dubbo://10.35.25.111:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-api-provider&bind.ip=10.35.25.111&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello&pid=18164&release=&side=provider&timestamp=1606657116399
```

构造完URL后便正式进入服务暴露的过程，包括本地暴露和远程暴露。这两种方式都是可选的，可以通过scope参数来控制，可选值包括

1. none：不对外进行暴露
2. remote：只暴露远程服务
3. local：只暴露本地服务
4. 空：同时暴露本地服务和远程服务（默认值）

#### 2.2.1 暴露本地服务

指暴露在同一个JVM里面，不用通过调用注册中心如ZK来进行远程通信。例如：在同一个服务，自己调用自己的接口，就没必要进行网络IP连接来通信。如下为本地暴露的内容

![](exportLocal.png)

本地服务暴露给注册中心的URL为:

```
injvm://127.0.0.1/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-api-provider&bind.ip=10.35.25.111&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello&pid=23480&release=&side=provider&timestamp=1606665693215
```

本地暴露时主要做几件事：

1. 拷贝原URL，将协议转为```injvm```
2. 设置URL的host为```127.0.0.1```，端口为```0```
3. **将本地URL转为Exporter**

#### 2.2.2 暴露远程服务

指暴露给远程客户端的IP和端口号，以便通过网络来实现通信。在原URL基础上，会尝试再增加一些可选的参数，如监控:monitor，invoker代理:proxy，最后Service对外开放的URL为：

```
dubbo://10.35.25.111:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-api-provider&bind.ip=10.35.25.111&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello&pid=23480&release=&side=provider&timestamp=1606665693215
```

然后会将该Service的URL添加到Registry的URL中，同时使用Registry的URL转为Exporter。

![](exportRemote.png)

从2.7.0版本开始，增加了服务元数据存储，将原来注册到注册中心的URL进行减负，抽取一些参数存储到元数据存储中，减少注册中心的负担。

![](metadata.png)

#### 3. 代理逻辑

Dubbo可以选择多种通信协议，也可以选择不同的注册中心，为了统一处理，抽象出了一个Proxy层和Protocol层。不管是本地暴露，还是远程暴露，都涉及到```ProxyFactory```和```Protocol```接口，二者都是动态来适配具体的代理插桩逻辑和协议实现逻辑,这两个都用到了上一节提到的自适应扩展机制SPI。

#### 3.1 ProxyFactory

ProxyFactory的定义如下：

```
@SPI("javassist")
public interface ProxyFactory {
    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;

    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException;

    @Adaptive({PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;
}
```

ServiceConfig里调用方式为：

```
private static final ProxyFactory PROXY_FACTORY = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
```


ProxyFactory通过指定参数```proxy```的值来动态选择实现，默认使用```javassist```，如下为dubbo-rpc-api模块提供的实现：

![](ProxyFactory.png)

既ServiceConfig里ProxyFactory的具体实现类为```org.apache.dubbo.rpc.proxy.javassist.JavassistProxyFactory```。

JavassistProxyFactory使用Javassist来实现动态代理，主要实现方法如下：

![](JavassistProxyFactory.png)

1.将目标接口包装为```org.apache.dubbo.common.bytecode.Wrapper```的子类，Wrapper使用Javassist以硬编码的方式代理了目标接口的动作以实现动态代理的功能，减少了反射的调用。对于开头例子的接口，Wrapper处理后的主要内容如下:

```
public class Wrapper0 extends Wrapper implements DC {

	...

	 public Object invokeMethod(Object var1, String var2, Class[] var3, Object[] var4) throws InvocationTargetException {
        DemoService var5;
        try {
            var5 = (DemoService)var1;
        } catch (Throwable var8) {
            throw new IllegalArgumentException(var8);
        }

        try {
            if ("sayHello".equals(var2) && var3.length == 1) {
                return var5.sayHello((String)var4[0]);
            }
        } catch (Throwable var9) {
            throw new InvocationTargetException(var9);









        }

        throw new NoSuchMethodException("Not found method \"" + var2 + "\" in class org.apache.dubbo.demo.DemoService.");
    }

    public Wrapper0() {
    }

}
```

2.构造AbstractProxyInvoker对象并返回，AbstractProxyInvoker实现了```org.apache.dubbo.common.Node.Invoker```接口。

```
public interface Invoker<T> extends Node {

    /**
     * get service interface.
     *
     * @return service interface.
     */
    Class<T> getInterface();

    /**
     * invoke.
     *
     * @param invocation    调用
     * @return result       结果
     * @throws RpcException
     */
    Result invoke(Invocation invocation) throws RpcException;

}
```

***Invoker表示Service对应接口的可执行对象***，通过传入Invocation指明调用哪个方法以及方法参数。具体到JavassistProxyFactory，invoke方法的实现则是使用Wrapper动态代理技术。

![](Invoker.png)

#### 3.2 Protocol

Protocol的定义如下：

```
@SPI("dubbo")
public interface Protocol {
    int getDefaultPort();

    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    void destroy();
}
```


ServiceConfig里调用方式为：

```
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

Protocol比较特殊，上节提到过Protocol接口方法上没有指定key，而是直接使用的URL中```protocol```属性字段的值。因而本地暴露使用的```InjvmProtocol```实例，远程暴露使用的```RegistryProtocol```实例。

Protocol接口的作用在于，将Invoker以特定的协议暴露出去，并返回对应的Expoter。Exporter表示一个对外暴露的可执行对象，可以通过对应的key获得内部的可执行对象Invoker。

开头例子进行本地暴露和远程暴露的过程如下：

![](ProtocolExport.png)

本地暴露只是简单的返回一个InjvmExprter（InjvmExporter会缓存在InjvmProtocol内部），仅供JVM内部使用。

远程暴露会先经过RegistryProtocol，在这里会拿到原始的协议即让DubboProtocol处理一遍，DubboProtocol内部会***启动一个本地服务***并将Invoker包装为DubboExporter并返回（DubboExporter会缓存在DubboProtocol内部）；***然后会去访问注册中心，将该服务注册上去，同时订阅该服务的变化***；最后返回包装过后的DestryableExpoter。

经过该步骤后Provider端已经启动本地服务以便接收外部请求，同时也将服务的路由信息注册到了ZK,以便外部发现该服务，可以正式对外提供服务。如下为ZK发布的服务路由信息：

![](zk.png)

#### 4. Provider的服务模型

#### 4.1 本地服务启动

上面提到，DubboProtocol会在本地启动一个本地服务，Dubbo内部将服务抽象为```org.apache.dubbo.remoting.exchange.ExchangeServer```

```
public interface ExchangeServer extends Server {

    /**
     * get channels.
     *
     * @return channels
     */
    Collection<ExchangeChannel> getExchangeChannels();

    /**
     * get channel.
     *
     * @param remoteAddress
     * @return channel
     */
    ExchangeChannel getExchangeChannel(InetSocketAddress remoteAddress);

}
```

ExchangeServer位于```Exchange```层，即信息交换层，是对```Request```和```Response```的抽象。主要用于整合各通信方式，提供统一的通信模型：请求-响应模型。

创建ExchangeServer核心动作如下：

![](ExchangeServer.png)

Exchanges使用了SPI来获取Exchanger的实现，并调用bind方法，

```
@SPI(HeaderExchanger.NAME)
public interface Exchanger {

    /**
     * bind.
     *
     * @param url
     * @param handler
     * @return message server
     */
    @Adaptive({Constants.EXCHANGER_KEY})
    ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException;

    /**
     * connect.
     *
     * @param url
     * @param handler
     * @return message channel
     */
    @Adaptive({Constants.EXCHANGER_KEY})
    ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException;

}
```

bind方法的意思为：将服务对应的URL绑定到ExchangeHandler处理器上。Exchanger的默认实现为```org.apache.dubbo.remoting.exchange.support.header.HeaderExchanger```基于消息头的信息交换。

```
public class HeaderExchanger implements Exchanger {

    public static final String NAME = "header";

    @Override
    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
    }

    @Override
    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

}
```

在这里调用了下一层：协议转换层```Transporter```。

```
@SPI("netty")

public interface Transporter {

    /**
     * Bind a server.
     *
     * @param url     server url
     * @param handler
     * @return server
     * @throws RemotingException
     * @see org.apache.dubbo.remoting.Transporters#bind(URL, ChannelHandler...)
     */
    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
    Server bind(URL url, ChannelHandler handler) throws RemotingException;

    /**
     * Connect to a server.
     *
     * @param url     server url
     * @param handler
     * @return client
     * @throws RemotingException
     * @see org.apache.dubbo.remoting.Transporters#connect(URL, ChannelHandler...)
     */
    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;

}
```

Transporter用于提供统一的协议转换模型，将Exchange层“有序”的Requqest-Response模型中的消息，通过```Channel```(通道)“无序”的进行发送/接受。

Transporter默认的实现为```NettyTransporter```，既底层使用Netty进行通信。

对于服务端而言，就是包装Netty的API，启动一个Netty后台服务，并传入一个请求处理器，在请求到来时进行处理并返回响应。Netty在启动BootStrapServer时，需要指定编解码处理器，还要指定消息处理器，消息处理器就是Transporter层的ChannelHander/Exchange层的ExchangeHandler，编解码器则位Transporter层的下层：codec层。

![](code.png)

编解码器也是通过SPI来定位具体实现类的，接口为```org.apache.dubbo.remoting.Codec2```，key为```codec```。具体到DubboProtocol，使用的是Dubbo内部自己的协议，实现类为```org.apache.dubbo.rpc.protocol.dubbo.DubboCountCodec```。DubboCountCodec使用```org.apache.dubbo.rpc.protocol.dubbo.DubboCodec```来解码消息，将解码的多条消息整合到MultiMessage中。

DubboCodec处理Netty接收的消息，对消息进行编解码，然后包装为```org.apache.dubbo.remoting.exchange.Request```或者```org.apache.dubbo.remoting.exchange.Response```对象，即处理为Request-Response模型，再交由消息处理器处理。

在Dubbo协议中，会为每次Request-Response交互分配一个唯一id，称为消息id。消息id通过编码，最终会作为请求内容的一部分发送出去，同时响应内容中也会带有消息id，通过解码，会把id解析出来。通过消息id,“无序”的tcp交互，在Exchange层就变得“有序”起来。

#### 4.2 处理消息

NettyServer在解码消息后，会出现相应的事件，如```connect```,```read````,```write```等，会包装为ChannelHandler的事件，再交由Transporter层的ChannelHandler处理，最后再进行包装交由Exchange层的ExchangeHandler处理。

具体到Dubbo协议，处理器是在DubboProtocol中传入的，该处理器为ExchangeHandlerAdapter的子类，最终的事件处理动作会在这里处理。在传到codec层时该Handler会经过多层包装：

![](Handler.png)

拿到Invocation对象后，便可以获得对应的serviceKey，从而通过DubboProtocol内部缓存的Exporter中拿到对应的Exporter，拿到Exporter便能拿到Invoker，从而调用对应的具体实现类的方法，触发真正的调用。

概括Provider主要做的内容就是：Provider将接口实现通过Wrapper类包装起来，作为Invoker，以便可以通过Invocation来动态执行实现类的方法；将接口名作为ServiceKey，以及Provider本身的ip和端口注册到注册中心，作为Exporter；Exporter跟Invoker一一映射，外界通过ServiceKey可以获取JVM缓存的Exporter，从而获取到Invoker，进而通过Wrapper对象调用目标类的方法。

整个核心过程经过扩展后为：

![](ProviderDetail.png)
