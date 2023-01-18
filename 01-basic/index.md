# Sentinel基本概念


&emsp;Sentinel是阿里开源的一款高性能的限流框架。这里将对Sentinel的使用和实现进行介绍。

&emsp;这里先介绍下Sentinel中涉及到的基本概念，包括使用上或者实现上。主要是笔者在阅读文档和源码时经常会接触到的对象。

#### Resource

&emsp;资源是整个Sentinel最基本的一个概念。可以是一段代码，一个http请求，一个微服务，总而言之，他是Sentinel需要保证的实体。大部分情况下，我们可以使用方法签名，URL或者是服务名称来作为资源的名称。它在Sentinel中的体现是：ResourceWrapper，他有两个子类：

 1. StringResourceWrapper 使用string来标识一个资源

 2. MethodResouceWrapper 使用一个函数签名来标识一个资源

#### Node

&emsp;节点是用来存储统计数据的基本数据单元，Node本身只是一个接口，它有多个实现：

1. StatisticNode 唯一的直接实现类，实现了流量统计的基本方法，在StatisticSlot中使用

2. ClusterNode 继承自StatisticNode，对于某一个资源的全局统计

3. DefaultNode 继承自StatisticNode，对于某一个资源在相应上下文中的实现，保存了一个指向ClusterNode的引用。另外还保存了子节点列表，当在同一个context下多次调用SphU.entry不同资源时会创建子节点

4. EntranceNode 继承自DefaultNode，代表一个调用的根节点，一个Context会对应到一个EntranceNode

#### Context

&emsp;上下文是用来保存当前调用的元数据，存储在ThreadLocal中，它包含了几个信息：

1. EntranceNode 整个调用树的根节点，即入口

2. Entry 当前的调用点

3. Node 关联到当前调用点的统计信息

4. Origin 通常用来标识调用方，这在我们需要按照调用方来区分流控策略的时候会非常有用

&emsp;每当我们调用SphU.entry()或者 SphO.entry()获取访问资源许可的时候都需要当前线程处在某个context中，如果我们没有显式调用ContextUtil.enter()，默认会使用Default context。如果我们在一个上下文中多次调用SphU.entry()来获取多个资源，一个调用树就会被创建出来

#### NullContext

&emsp;超过系统能够创建的最大会话数则返回NullContext，后续不对该会话做过滤校验，直接放过。

&emsp;Entry

&emsp;每次SphU.entry()调用都会返回一个Entry，Entry保持了所有关于当前资源调用的信息：

1. createTime 这个资源调用的创建时间

2. currentNode SphU.entry请求进入的资源在当前上下文中的统计数据Node

3. originNode SphU.entry请求进入的资源对于特定origin调用方的统计数据node

&emsp;Entry的实现类为CtEntry，它其中除了上述信息之外，还保存了额外的信息：

1. parent 调用树链条中上一个entry

2. child 调用树链条中的下一个entry

3. chain 当前调用资源所使用的限流工作责任链，包括各个Slot

4. context 当前调用点所属的上下文

#### EntryType

&emsp;EntryType 说的是这次请求的流量类型，共有两种类型：IN 和 OUT 。

&emsp;IN：是指进入我们系统的入口流量，比如 http 请求或者是其他的 rpc 之类的请求。

&emsp;OUT：是指我们系统调用其他第三方服务的出口流量。

&emsp;入口、出口流量只有在配置了系统规则时才有效。

&emsp;设置Type 为 IN 是为了统计整个系统的流量水平，防止系统被打垮，用以自我保护的一种方式。

&emsp;设置Type 为 OUT 一方面是为了保护第三方系统，比如我们系统依赖了一个生成订单号的接口，而这个接口是核心服务，如果我们的服务是非核心应用的话需要对他进行限流保护；另一方面也可以保护自己的系统，假设我们的服务是核心应用，而依赖的第三方应用老是超时，那这时可以通过设置依赖的服务的 rt 来进行降级，这样就不至于让第三方服务把我们的系统拖垮。

#### Slot

&emsp;Entry 创建的时候，同时也会创建一系列功能插槽（slot chain），这些插槽有不同的职责，例如:

1. NodeSelectorSlot 负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级；

2. ClusterBuilderSlot 则用于存储资源的统计信息以及调用者信息，例如该资源的 RT, QPS,thread count 等等，这些信息将用作为多维度限流，降级的依据；

3. LogSlot 用于打印日志

4. StatisticSlot 则用于记录、统计不同纬度的 runtime 指标监控信息；

5. SystemSlot 则通过系统的状态，例如 load1 等，来控制总的入口流量；

6. AuthoritySlot 则根据配置的黑白名单和调用来源信息，来做黑白名单控制；

7. FlowSlot 则用于根据预设的限流规则以及前面 slot 统计的状态，来进行流量控制；

8. DegradeSlot 则通过统计信息以及预设的规则，来做熔断降级；

&emsp; Slot只绑定在CtEntry上

#### ProcessorSlotChain

&emsp;功能槽处理链，entry进入一个槽可以添加自己的动作，之后后fire动作会entry下一个槽，exit同理

&emsp;**注意，实现上相同资源共享一个ProcessorSlotChain ，可以跨Context**

#### LimitApp

&emsp;流控规则中的 limitApp 字段用于根据调用来源进行流量控制。该字段的值有以下三种选项，分别对应不同的场景：

1. default：表示不区分调用者，来自任何调用者的请求都将进行限流统计。如果这个资源名的调用总和超过了这条规则定义的阈值，则触发限流。

2. {some_origin_name}：表示针对特定的调用者，只有来自这个调用者的请求才会进行流量控制。例如 NodeA 配置了一条针对调用者caller1的规则，那么当且仅当来自 caller1 对 NodeA 的请求才会触发流量控制。

3. other：表示针对除 {some_origin_name} 以外的其余调用方的流量进行流量控制。例如，资源NodeA配置了一条针对调用者 caller1 的限流规则，同时又配置了一条调用者为 other 的规则，那么任意来自非 caller1 对 NodeA 的调用，都不能超过 other 这条规则定义的阈值。

&emsp;同一个资源名可以配置多条规则，规则的生效顺序为：{some_origin_name} > other > default

&emsp;介绍完了上面的基本概念，下面给出Sentinel的基本用法：

```

List<FlowRule> rules = new ArrayList<FlowRule>();
FlowRule rule1 = new FlowRule();
rule1.setResource(KEY);
// set limit qps to 20
rule1.setCount(20);
rule1.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule1.setLimitApp("default");
rules.add(rule1);
​
Entry entry = null;
​
try {
        entry = SphU.entry(KEY);
        // token acquired, means pass,do biz logic
} catch (BlockException e1) {
        //block,handle block logic
} catch (Exception e2) {
        // biz exception,handle biz exception logic
} finally {
        if (entry != null) {
                entry.exit();
        }
}
​
```

&emsp;如上，为sentinel的基本用法:

&emsp;先设定好规则，在进入需要受保护的资源前，尝试获取token，若成功获取token，则可以执行相关逻辑，否则抛出异常进行处理，最后释放所获得的token 。

