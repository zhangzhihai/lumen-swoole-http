# 目前PHP存在的问题
作为一种脚本语言，PHP的开发效率的确是高的；在使用了新的解析器Zend Engine3之后，PHP7的执行效率几乎翻了一倍。可是在实际中，一个LNMP架构的Web应用，性能表现往往不尽人意。也许可以在以下两个方面找到一些因由：

## NginX与PHP的连接
在LNMP架构里，NginX的主要工作是代理。客户端发过来的请求，除了静态资源，NginX都会转给PHP-FPM处理。尽管PHP-FPM提供进程池机制，不过NginX与客户端维持一层连接，与PHP-FPM也维持一层连接，自然少不了时间上和资源上的消耗。一个PHP-FPM可以接收多个请求连接，不过在同一时间内PHP-FPM进程最多处理一个请求。在高并发的情况下，只能通过增加PHP-FPM进程数来应对，而进程数目过大又不利于CPU调度。对于每个请求，PHP-FPM都会从入口文件开始加载和编译所需脚本，虽然这一步可以利用Opcache等缓存工具来加速，但是不能减少大量的重复性操作，如应用框架的初始化之类。

如果PHP不依靠NginX，自身能够提供HTTP服务，并且执行统一的一次应用初始化后，再去处理各种客户端请求，那么应用整体执行效率肯定是有所提升的。

## MySQL与PHP的连接
一般PHP与MySQL建立的通信连接是短连接，用完即断，再用重建。PHP也可以与MySQL建立长连接通信，一个PHP-FPM进程与MySQL建立长连接后，该进程在处理后续各个客户端请求时都可以一直复用这个连接（如果请求的是同一个MySQL数据库的话）；长连接不会被回收，直到所在进程被销毁。表面上，使用长连接，效率会更高。实际上，PHP-FPM在动态模式下，空闲进程会被回收，进程所建立的长连接也就没有了；而在静态模式下，进程会长驻内存，高并发时大量的PHP-FPM进程连接MySQL，MySQL本身是一个线程维系一个连接，线程太多会影响MySQL的调度效率；再者，MySQL会主动断开长时间无用的连接，PHP-FPM这边继续维持这个连接只会得到`server has gone away`错误。一般情况下，PHP与MySQL通信大多都是采用短连接，那么无可避免地在高并发的时候需要频繁地、重复地创建大量的连接。

如果PHP与MySQL之间能有一个长驻内存的、数量可控的连接池供给两者通信使用，就可以减少重复创建连接带来的消耗和稳定MySQL的线程数，进而提高两者之间的通信效率。