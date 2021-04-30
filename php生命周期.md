# php生命周期

## 启动阶段

当在CLI模式下，c语言中的main函数将会执行。如果是在fpm模式下，需要等到fpm启动worker进程时。我们称这个过程为MINIT。

## 请求初始化阶段

在cli模式运行的php程序只会有一次请求，但是在pfm模式下运行的会处理数次请求，这取决于你的pfm配置，到底是采用处理一定量请求后退出还是无限处理。总之每当一个新的请求到来，php会执行**request startup**，我们称为RINIT。

## 请求终止阶段

当处理完一个请求时，典型的场景比如一次http请求结束，php将来到**request shutdown**，这个过程被称为RSHUTDOWN。

## 终止阶段

当处理完一定量的请求，php将会终止，该阶段为**module shutdown**,我们称为MSHUTDOWN。

![../../_images/php_classic_lifetime.png](https://www.phpinternalsbook.com/_images/php_classic_lifetime.png)

# 模型

## cli模型

cli模式下的模型异常简单，进程执行结束，不需要复杂的操作。

## web模型

在web模式下，情况会复杂很多，由于请求会同时到达，所以需要考虑并行的模型。

* 多进程模型
* 多线程模型

使用多进程模型，每个进程都会有自己的MINIT过程，每个进程使用独立的数据空间。该模型非常常见，PHP-CLI,PHP-FPM，PHP-CGI均使用该模型。

使用多线程模型，进程只有一个MINIT，同时要求扩展支持ZTSmode。(似乎是没有这样的实例)

以下是进程模型的示意图:

![../../_images/php_lifetime_process.png](https://www.phpinternalsbook.com/_images/php_lifetime_process.png)

以下是线程模型的示意图:

![../../_images/php_lifetime_thread.png](https://www.phpinternalsbook.com/_images/php_lifetime_thread.png)

# 扩展hook

## PHP_MINIT_FUNCTION

分配持久化资源或者是以后每个请求都要使用的信息。在该阶段没有多进程或者是多线程，你可以随意使用全局变量。同样你也不能在该阶段申请request阶段的资源。

全局INI配置正是在该阶段注册

仅可读的zend_string可在该阶段注册

## PHP_MSHUTDOWN_FUNCTION

你基本上是在执行MINIT阶段相反的操作，比如释放全局变量，释放INI配置。同样你也不能在该阶段使用request阶段的资源

## PHP_RINIT_FUNCTION

在该阶段，你可以使用ZendMemoryManager进行动态资源的申请。通过调用emalloc()。ZendMemoryManager(以下简称为**ZMM**)将会跟踪内存使用当请求结束时将会尝试释放请求阶段的内存。该阶段会涉及到数据竞争问题所以谨慎使用全局变量。

## PHP_RSHUTDOWN_FUNCTION

基本上在执行与RINIT阶段相反的操作。

举几个RSHUTDOWN执行的场景,让你有个直观的感受:

* 用户shutdown方法执行(register_shutdown_function())
* 对象析构函数被调用
* php输出buffer被刷新
* 达到max_execution_time

## PHP_PRSHUTDOWN_FUNCTION

这个hook非常少见，调用在RSHUTDOWN后,同样很少见的有GINIT、GSHUTDOWN(我不知道在哪里注册这个hook)

## PHP_MINFO_FUNCTION

当使用phpinfo()时会触发该函数获取扩展相关信息，该hook期待得到有关扩展的描述。

该hook在cli模式下使用php --ri 也会调用到。

hook流程图:

![](https://www.phpinternalsbook.com/_images/php_extensions_lifecycle.png)