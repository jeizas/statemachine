### 状态机使用

#### 使用场合

- 电商场景（订单、物流、售后）、社交（IM消息投递）、分布式集群管理（分布式计算平台任务编排）等场景都有大规模的使用

####  一、[spring statemachine](http://projects.spring.io/spring-statemachine/#quick-start)

1. 优点

- 配置状态以及转移需要的事件，采用层次化状态机结构简化复杂状态配置，比原有实现方式更加解耦合，逻辑更加清晰，方便维护
- 可定义状态转移需要执行的action
- 基于ZooKeeper实现的分布式状态机 
- 内置redis存储
- 有事件监听器 

2. 缺点

- 单例statemachine状态机实例较重，状态转移依赖statemachine，不支持并发
- 工厂模式会中statemachine会存储context

3. 实现方式

- 单例

  只有一个statemachine实例，重置状态机(设置当前状态)并处理statemachine.send(event)，实例不能够被多线程共享。 

  StateMachineListener 跟踪状态转移

  StateChangeInterceptor 改变状态转移链的变化

  ~~~
  stateMachine.stop();
  stateMachine.getStateMachineAccessor()
      .doWithAllRegions(access -> access.resetStateMachine(
          new DefaultStateMachineContext<States, Events> (sourceState, null, null, extendedState)));
  stateMachine.start ();
  boolean sendEvent = stateMachine.sendEvent(event);
  ~~~

  

- 工厂

  获取stateMachine

  ~~~
  public StateMachine<S, E> create(String machineId) {
          StateMachine<S, E> stateMachine = stateMachineFactory.getStateMachine(machineId);
          stateMachine.start();
          return stateMachine;
      }
  ~~~

#### 二、[squirrel-foundation](http://hekailiang.github.io/squirrel/)

squirrel的事件处理模型与spring-statemachine比较类似，squirrel的事件执行器的作用点粒度更细，通过预处理，将一个状态迁移分解成exit trasition entry 这三个action event，再递交给执行器分别执行(这个设计挺不错)

**优点**

- 代码写的不错，设计域很清晰，测试case以及项目文档都比较详细；
- 功能该有的都有，支持exit、transition、entry动作，状态转换过程被细化为tranistionBegin->exit->transition->entry->transitionComplete->transitionEnd，并且提供了自定义扩展机制，能够方便的实现状态持久化以及状态trace等功能；
- StateMachine实例创建开销小，设计上就不支持单例复用，因此状态机的本身的生命流管理也更清晰，避免了类似spring statemachine复用statemachine导致的deadlock之类的问题；
- 代码量适中，扩展和维护相对而言比较容易；

**缺点**

- 注解方式定义状态转换，不支持自定义状态枚举、事件枚举；
- interceptor的实现粒度比较粗，如果需要对特定状态的某些切入点进行逻辑处理需要在interceptor内部进行逻辑判断，例如在transitionEnd后某些状态下需要执行一些特定action，需要transitionEnd回调中分别处理；

#### 三、[stateless4j](https://github.com/oxo42/stateless4j)

**优点**

- 足够轻量，创建StateMachine实例开销小
- 支持基本的事件迁移、exit/entry action、guard、dynamic permit(相同的事件不同的condition可到达不同的目标状态)
- 核心代码千行左右，基于现有代码二次开发的难度也比较低

**缺点**

- 支持的动作只包含了entry exit action，不支持transition action
- 在状态迁移的模型中缺少全局的observer(缺少interceptor扩展点)，例如要做state的持久化就很恶心([扩展stateMutator在设置目标状态的同时完成持久化](https://stackoverflow.com/questions/29990009/how-can-we-persist-states-and-transitions-in-stateless4j-based-state-machine)的方案将先于entry进行persist实际上并不是一个好的解决方案)
- 状态迁移的模型过于简单，这也导致了本身支持的action和提供的扩展点有限

**结论**

- stateless4j足够轻量，同步模型，在app中使用比较合适，但在服务端解决复杂业务场景上stateless4j确实略显单薄

#### 四、DEMO演示

​	详情在idea



#### 参考链接

* [状态机引擎选型](https://segmentfault.com/a/1190000009906317)
* [状态机选型简记](http://childe.net.cn/2018/04/28/%E7%8A%B6%E6%80%81%E6%9C%BA%E9%80%89%E5%9E%8B%E7%AE%80%E8%AE%B0/)
* [基于注解的实现](https://www.codetd.com/article/1010726)
* [Linux C 编程技巧--利用有限状态机模型编程](https://blog.csdn.net/zqixiao_09/article/details/50239337)
* [复杂业务状态的处理：从状态模式到 FSM](https://juejin.im/entry/5a26b17df265da4319562263)

