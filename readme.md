

本项目是从 laravel 的 illuminate/container 基础上修改而来的.


## 介绍


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


