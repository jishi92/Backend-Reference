# nginx和php的交互
https://www.jianshu.com/p/eab11cd1bb28
nginx不支持对外部程序的直接调用或解析，所有的外部程序（包括PHP）都必须通过fastCGI接口来调用。fastCGI接口在linux下是socket。

Nginx是个轻量级的HTTP server，必须借助第三方的FastCGI处理器才可以对PHP进行解析，因此其实这样看来nginx是非常灵活的，它可以和任何第三方提供解析的处理器实现连接从而实现对PHP的解析(在nginx.conf中很容易设置)。

FastCGI接口方式在脚本解析服务器上启动一个或者多个守护进程对动态脚本进行解析，这些进程就是FastCGI进程管理器，或者称为FastCGI引擎。 spawn-fcgi与PHP-FPM就是支持PHP的两个FastCGI进程管理器。
![](https://upload-images.jianshu.io/upload_images/3407216-389d482b4a76f4a2.png?imageMogr2/auto-orient/strip|imageView2/2/w/600/format/webp)  

CGI是通用网关协议，FastCGI则是一种常住进程的CGI模式程序。我们所熟知的PHP-FPM的全称是PHP FastCGI Process Manager，即PHP-FPM会通过用户配置来管理一批FastCGI进程，例如在PHP-FPM管理下的某个FastCGI进程挂了，PHP-FPM会根据用户配置来看是否要重启补全，PHP-FPM更像是管理器，而真正衔接Nginx与PHP的则是FastCGI进程。


	①CGI：是 Web Server 与 Web Application 之间数据交换的一种协议。
	②FastCGI：同 CGI，是一种通信协议，但比 CGI 在效率上做了一些优化。
	③PHP-CGI：是 PHP （Web Application）对 Web Server 提供的 CGI 协议的接口程序。
	④PHP-FPM：是 PHP（Web Application）对 Web Server 提供的 FastCGI 协议的接口程序，额外还提供了相对智能一些任务管理。

FastCGI像是一个常驻(long-live)型的CGI，它可以一直执行着，只要激活后，不会每次都要花费时间去fork一次，所以提高了CGI的性能。它还支持分布式的运算,即 FastCGI 程序可以在网站服务器以外的主机上执行，并且接受来自其它网站服务器来的请求。

PHP-CGI就是PHP实现的自带的FastCGI管理器。但有两个缺点：  
1，php-cgi变更php.ini配置后，需重启php-cgi才能让新的php-ini生效，不可以平滑重启。  
2，直接杀死php-cgi进程，php就不能运行了。  
而php-fpm的出现解决了这两个问题，PHP-FPM也是用于调度管理PHP解析器php-cgi的管理程序。 

Nginx通过location指令，将所有以php为后缀的文件都交给127.0.0.1:9000来处理，而这里的IP地址和端口就是FastCGI进程监听的IP地址和端口。

Nginx和PHP-FPM的进程间通信有两种方式

	一种是TCP
	一种是UNIX Domain Socket.

其中TCP是IP加端口，可以跨服务器。而UNIX Domain Socket不经过网络，只能用于Nginx跟PHP-FPM都在同一服务器的场景。

步骤：

	step1：用户将http请求发送给nginx服务器(用户和nginx服务器进行三次握手进行TCP连接)
	step2：nginx会根据用户访问的URI和后缀对请求进行判断
	step3：通过第二步可以看出，用户请求的是动态内容，nginx会将请求交给fastcgi客户端，通过fastcgi_pass将用户的请求发送给php-fpm
	如果用户访问的是静态资源呢，那就简单了，nginx直接将用户请求的静态资源返回给用户。
	step4：wrapper收到php-fpm转过来的请求后，wrapper会生成一个新的线程调用php动态程序解析服务器
	step5：php会将查询到的结果返回给nginx
	step6：nginx构造一个响应报文将结果返回给用户

即：Nginx -> FastCGI -> php-fpm -> FastCGI Wrapper -> php解析器


## PHP垃圾回收机制
https://blog.csdn.net/u011957758/article/details/76864400  
在php5.3的GC中，针对的垃圾做了如下说明：

	1：如果一个zval的refcount增加，那么此zval还在使用，肯定不是垃圾，不会进入缓冲区
    2：如果一个zval的refcount减少到0， 那么zval会被立即释放掉，不属于GC要处理的垃圾对象，不会进入缓冲区。
    3：如果一个zval的refcount减少之后大于0，那么此zval还不能被释放，此zval可能成为一个垃圾，将其放入缓冲区。PHP5.3中的GC针对的就是这种zval进行的处理。
PHP5.3的垃圾回收算法有以下几点特性：

1、并不是每次refcount减少时都进入回收周期，只有根缓冲区满额后在开始垃圾回收。

2、可以解决循环引用问题。

3、可以总将内存泄露保持在一个阈值以下。
## PHP5和PHP7区别

	1、性能提升：PHP7比PHP5.0性能提升了两倍。

1、变量存储字节减小，减少内存占用，提升变量操作速度  
2、改善数组结构，数组元素和hash映射表被分配在同一块内存里，降低了内存占用、提升了 cpu 缓存命中率  
3、改进了函数的调用机制，通过优化参数传递的环节，减少了一些指令，提高执行效率  
	
	2、以前的许多致命错误，现在改成抛出异常。
定义了一个新的类Throwable，包含了Exception 和 Error.  
PHP5.x 中所有的错误都是 fatal 或 recoverable 级别的错误，在 PHP7 中都能抛出一个 Error实例。但某些错误会抛出一个更具体的 Error 子类：TypeError、ParseError 以及 AssertionError。
包括语法错误，分母为0，位移操作负数，类型错误。  
	
	3、PHP 7.0比PHP5.0移除了一些老的不在支持的SAPI（服务器端应用编程端口）和扩展。
	
	4、PHP 7.0比PHP5.0新增了空接合操作符。
	
	5、PHP 7.0比PHP5.0新增加了结合比较运算符。
	
	6、PHP 7.0比PHP5.0新增加了函数的返回类型声明。
	
	7、PHP 7.0比PHP5.0新增加了标量类型声明。
	
	8、PHP 7.0比PHP5.0新增加匿名类。
	
	9、错误处理和64位支持
	
	10、声明返回类型


## 语句include和require的区别是什么？

require是无条件包含，也就是如果一个流程里加入require，无论条件成立与否都会先执行require，当文件不存在或者无法打开的时候，会提示错误，并且会终止程序执行

include有返回值，require没有(可能因为如此require的速度比include快)，include如果被包含的文件不存在的化，那么会提示一个错误，但是程序会继续执行下去

## php的fpm进程管理器的三种模式，优缺点是什么

1，static: 这种方式比较简单，在启动时master按照pm.max_children配置fork出相应数量的worker进程，即worker进程数是固定不变的

2，dynamic: 动态进程管理，首先在fpm启动时按照pm.start_servers初始化一定数量的worker，运行期间如果master发现空闲worker数低于pm.min_spare_servers配置数(表示请求比较多，worker处理不过来了)则会fork worker进程，但总的worker数不能超过pm.max_children，如果master发现空闲worker数超过了pm.max_spare_servers(表示闲着的worker太多了)则会杀掉一些worker，避免占用过多资源，master通过这4个值来控制worker数

3，ondemand: 这种方式一般很少用，在启动时不分配worker进程，等到有请求了后再通知master进程fork worker进程，总的worker数不超过pm.max_children，处理完成后worker进程不会立即退出，当空闲时间超过pm.process_idle_timeout后再退出

## 你的PHP是怎么做安全处理的，比如SQL注入、xss、csrf


## php的static和self的区别  

使用 self:: 或者 __CLASS__ 对当前类的静态引用，取决于定义当前方法所在的类。self就是写在哪个类里面, 实际调用的就是这个类.   

使用 static:: 不再被解析为定义当前方法所在的类，而是在实际运行时计算的。也可以称之为“静态绑定”，因为它可以用于（但不限于）静态方法的调用。static也称为后期静态绑定。  
static代表使用的这个类, 就是你在父类里写的static,然后被子类覆盖，使用的就是子类的方法或属性。  
后期静态绑定的解析会一直到取得一个完全解析了的静态调用为止。另一方面，如果静态调用使用 parent:: 或者 self:: 将转发调用信息。

## php的trait
php从以前到现在一直都是单继承的语言，无法同时从两个基类中继承属性和方法，为了解决这个问题，php出了Trait这个特性

用法：通过在类中使用use 关键字，声明要组合的Trait名称，具体的Trait的声明使用Trait关键词，Trait不能实例化

一个类可以组合多个Trait，通过逗号相隔，如下
	
	use trait1,trait2

当不同的trait中，却有着同名的方法或属性，会产生冲突，可以使用insteadof或 as进行解决，insteadof 是进行替代，而as是给它取别名。  
Trait也可以互相组合，还可以使用抽象方法，静态属性，静态方法等。

## isset和empty的区别

## array_merge函数与array+array的区别
+的效率更高
区别如下：

当下标为数值且key重复时：  
array_merge()不会覆盖掉原来的值，会把key重新排序追加在后面。  
array＋array合并数组则会保留前面key对应的value，而舍弃后面重复key的value。  

当下标为字符且key重复时：  
array_merge()此时会覆盖掉前面相同键名的值，而 把其他key重新排序追加在后面  
array＋array仍然把会保留前面key对应的value，而舍弃后面重复key的value。

## FPM和swoole
### PHP-FPM

Master 主进程 / Worker 多进程模式。

启动 Master，通过 FastCGI 协议监听来自 Nginx 传输的请求。

每个 Worker 进程只对应一个连接，用于执行完整的 PHP 代码。

PHP 代码执行完毕，占用的内存会全部销毁，下一次请求需要重新再进行初始化等各种繁琐的操作。

只用于 HTTP Server。

### Swoole

Master 主进程（由多个 Reactor 线程组成）/ Worker 多进程（或多线程）模式

启动 Master，初始化 PHP 代码，由 Reactor 监听 Socket 句柄的事件变化。

Reactor 主线程负责子多线程的均衡问题，Manager 进程管理 Worker 多进程，包括 TaskWorker 的进程。

每个 Worker 接受来自 Reactor 的请求，只需要执行回调函数部分的 PHP 代码。

只在 Master 启动时执行一遍 PHP 初始化代码，Master 进入监听状态，并不会结束进程。

不仅可以用于 HTTP Server，还可以建立 TCP 连接、WebSocket 连接。

#### Swoole 加速的原理

由 Reactor（epoll 的 IO 复用方式）负责监听 Socket 句柄的事件变化，解决高并发问题。
通过内存常驻的方式节省 PHP 代码初始化的时间，在使用笨重的框架时，用 swoole 加速效果是非常明显的。

#### swoole是什么？
php与外部通信需要借助系统的socket。通常使用的 nginx就是封装了的socket，可以实现并发处理。客户端发送请求到nginx/apache，再转发到fastcgi端口交给php处理swoole把系统的socket集成到php底层，php可以直接通过swoole与客户端交互。也就是说swoole是个封装了底层socket的网络库。

Swoole 虽然是标准的 PHP 扩展，实际上与普通的扩展不同。普通的扩展只是提供一个库函数。而 swoole 扩展在运行后会接管 PHP 的控制权，进入事件循环。当 IO 事件发生后，swoole 会自动回调指定的 PHP 函数。

强大的 TCP/UDP Server 框架，支持多线程，EventLoop，事件驱动，异步，Worker 进程组，Task 异步任务，毫秒定时器，SSL/TLS 隧道加密。

## include和require的区别  
include 就是包含，如果被包含的文件不存在的话， 那么则会提示一个错误，但是程序会继续执行下去。

require 意思是需要，如果被包含文件不存在或者 无法打开的时候，则会提示错误，并且会终止程序的执行。

这两种结构除了在如何处理失败之外完全一样。

once 的意思是一次，那么 include_once 和 require_once 表示只包含一次，避免重复包含。  
Require的效率比require_once的效率更高，因为require_once在包含文件时要进行判断文件是否已经被包含。

## 写几个魔术方法并说明作用？

__call()当调用不存在的方法时会自动调用的方法

__autoload()在实例化一个尚未被定义的类是会自动调用次方法来加载类文件

__set()当给未定义的变量赋值时会自动调用的方法

__get()当获取未定义变量的值时会自动调用的方法

__construct()构造方法，实例化类时自动调用的方法

__destroy()销毁对象时自动调用的方法

__unset()当对一个未定义变量调用unset()时自动调用的方法

__isset()当对一个未定义变量调用isset()方法时自动调用的方法

__clone()克隆一个对象

__tostring()当输出一个对象时自动调用的方法

## Final关键字  
PHP 5 新增了一个 final 关键字。如果父类中的方法被声明为 final，则子类无法覆盖该方法。如果一个类被声明为 final，则不能被继承。

## 对象接口interface
使用接口（interface），可以指定某个类必须实现哪些方法，但不需要定义这些方法的具体内容。

接口是通过 interface 关键字来定义的，就像定义一个标准的类一样，但其中定义所有的方法都是空的。

接口中定义的所有方法都必须是公有，这是接口的特性。

## 匿名类
PHP 7 开始支持匿名类。 匿名类很有用，可以创建一次性的简单对象  

	// PHP 7 之前的代码
	class Logger
	{
	    public function log($msg)
	    {
	        echo $msg;
	    }
	}
	
	$util->setLogger(new Logger());
	
	// 使用了 PHP 7+ 后的代码
	$util->setLogger(new class {
	    public function log($msg)
	    {
	        echo $msg;
	    }
	});

## 类的自动加载  
spl_autoload_register() 函数可以注册任意数量的自动加载器，当使用尚未被定义的类（class）和接口（interface）时自动去加载。通过注册自动加载器，脚本引擎在 PHP 出错失败前有了最后一个机会加载所需的类。

尽管 __autoload() 函数也能自动加载类和接口，但更建议使用 spl_autoload_register() 函数。 spl_autoload_register() 提供了一种更加灵活的方式来实现类的自动加载（同一个应用中，可以支持任意数量的加载器，比如第三方库中的）。因此，不再建议使用 __autoload() 函数，在以后的版本中它可能被弃用。