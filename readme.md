
本项目是从 [laravel](https://laravel.com) 的 [illuminate/container](https://github.com/illuminate/container) 基础上修改而来.

旨在实现双容器策略. 从而在 swoole, roadrunner, reactPHP 等项目中, 一个php 进程处理多个请求时, 为每个请求提供一个进程级别的容器, 和一个请求级别的容器. 后者可以从前者中获取实例.


## 使用方法

### 引入package

    composer require commune/container

您将得到:

-   ContainerTrait : 请求级 ioc 容器的trait
-   RecursiveContainer : 递归 ioc 容器的trait
-   IlluminateAdapter : illuminate container 的适配器.

### 使用 IoC 容器的trait

定义一个 IoC 容器

    /**
     * 标准的 IoC 容器.
     *
     * Class Container
     * @package Commune\Container
     */
    class Container implements ContainerContract
    {
        use ContainerTrait;

    }

使用此容器在进程启动时注册所有工厂方法. 这些方法会保存到静态属性里.

然后在请求开始时, 直接 new 实例, 会共享工厂方法, 而生成的单例又隔离开.

    class Foo {
        public $count = 0;
    }

    $container = new Container();
    $container->singleton(Foo::class, function(){
        return new Foo;
    });

    $server->on('request', function($req, $res) {
        $requestContainer = new Container();

        $foo = $requestContainer->get(Foo::class);

        // 请求之间的单例是相互隔离的.
        assert(0, $foo->count);
        $foo->count ++;
    });


### 递归容器


```RecursiveContainer``` 是递归容器的trait. 所谓递归容器, 指当前容器持有一个父级容器. 如果当前容器无法获取实例, 则会到父容器中查找.


因此, 在swoole 中, 可以定义一个进程级 (worker) 容器, 用于管理各种全局的单例, 尤其是和IO有关的. 例如 mysql, redis, 日志等等.

    class WorkerContainer  implements ContainerContract
    {
        use ContainerTrait;
    }

然后定义一个请求级容器, 用于处理与请求相关, 偏逻辑的信息.

    class RequestContainer  implements ContainerContract
    {
        use RecursiveContainer;
    }

最后将父容器传给子容器, 则子容器既可以拿到全局的单例, 又能保证请求之间隔离.

    // 定义进程级容器. 也可以用adpater去承载一个已有容器, 需要实现 ContainerContract 接口
    $workerContainer = new WorkerContainer();

    // 定义请求级容器.
    $requestContainer = new RequestContainer($workerContainer);

    // 分开注册
    $workerContainer->singleton(Foo::class, function() {
        return new Foo::class;
    });

    $requestContainer->singleton(Bar::class, function() {
        return new Bar::class;
    });

    // 请求级容器可以拿到进程级容器的实例.
    $foo = $requestContainer->get(Foo::class);


这样就可以实现双容器策略. 至于 laravel 的 register, boot 等机制则需要自己手动去实现. Trait 只提供container 相关的功能.


## 常见内存泄露原因

请求级容器每一个请求到来时会重新实例化. 所以要避免容器无法释放导致内存泄露.

一些经验是, 注册工厂方法时用 use ($container) 的方式从外部获取container, 或是创建实例时使用匿名类, 都可能导致闭包无法释放资源, 从而造成内存泄露.


推荐用一个静态变量注册已创建的请求级容器, 方便查看请求级容器是否正确释放:


    class RequestContainer extends RecursiveContainer
    {
        public static $running = [];

        protected $traceId;

        public function __construct(ContainerContract $workerContainer, string $traceId)
        {
            $this->traceId = $traceId;
            self::$running[$traceId] = true;

            parent::__constrcut($workerContainer);
        }


        public function __destruct()
        {
            unset(self::$running[$this->traceId]);
        }
    }



## 双容器策略介绍


容器(container)技术(可以理解为全局的工厂方法), 已经是现代项目的标配. 基于容器, 可以进一步实现控制反转, 依赖注入. [Laravel](https://github.com/laravel/laravel) 的巨大成功就是构建在它非常强大的IoC容器 [illuminate/container](https://github.com/illuminate/container) 基础上的. 而 PSR-11 定义了标准的 [container](https://github.com/php-fig/container) , 让更多的 PHP 项目依赖容器实现依赖解耦, 面向接口编程.

另一方面, PHP 天生一个进程响应一次请求的模型, 已经不能完全适应开发的需要. 于是 [Swoole](https://www.swoole.com/), [reactPHP](https://reactphp.org/), [roadrunner](https://github.com/spiral/roadrunner) 也越来越流行. 它们共同的特点是一个 php worker 进程在生命周期内要响应多个请求, 甚至同一时间同时运行多个请求 (协程).

在这些引擎上使用传统只考虑单请求的容器技术, 就容易发生单例相互污染, 内存泄露等问题 (姑且称之为PHP容器的"请求隔离问题" ). 于是出现了各种策略以解决之.

多轮对话机器人框架 [CommuneChatbot](https://github.com/thirdgerb/chatbot-studio) 使用 swoole 做通信引擎, 同时非常广泛地使用了容器和依赖注入. 在本项目中使用了 "双容器策略" 来解决 "请求隔离问题", 解决方案的代码在 [https://github.com/thirdgerb/container]()

所谓"双容器策略", 总结如下:

-   同时运行 "进程级容器" 与 "请求级容器"
-   "进程级容器" :
    -   传统的IoC 容器, 例如 Illuminate/container
-   "请求级容器" :
    -   所有工厂方法注册到容器的静态属性上
    -   在 worker 进程初始化阶段 注册服务
    -   每个请求到来后, 实例化一个请求容器.
    -   请求中生成的单例, 挂载到容器的动态属性上.
    -   持有"进程级容器", 当绑定不存在时, 到"进程级容器" 上查找之.
    -   请求结束时进行必要清理, 防止内存泄露


