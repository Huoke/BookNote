# 第12章 squid故障排除
在前面的章节中，我们学习了以不同模式安装和配置Squid代理服务器。然后我们紧接着进一步学习使用强大的URL重定向工具定制Squid。在部署产或者改变品的模式之前，当配置squid或者测试所有功能时，我们会非常小心，因为这可能会产生问题进而影响到我们的客户。这些问题可能是由于配置错误、Squid的bug、操作系统限制，甚至是由于网络问题造成的。在本章中，我们将学习常见的已知问题，以及如何以有效的方式解决这些问题。

在本章中，我们将学习如下内容：
1. 一些常见问题
2. 调试问题
3. 获取联机帮助并上报错误
# 一、一些常见问题
大多数出错是由于配置错误或者配置混淆造成的，这些问题通常被人们误以为是squid错误或者操作系统错误。如果你意识到squid用户遇到的这些常见问题有标准的解决方案，你就可以很快的解决这些问题。下面，让我们一起看看一些常见的问题：
## 1.1 无法写日志(Cannot write to log files)
通常， 当我们启动squid时，我们可能会收到类似如下的告警：
```Shell
WARNING: Cannot write log file: /opt/squid/var/logs/cache.log
/opt/squid/var/logs/cache.log: Permission denied
  messages will be sent to 'stderr'.
```
当运行squid的用户对包含日志文件的目录或日志文件本身没有写权限时,通常会发生这种情况。这种情况很大程度上是可以避免的，如果我们在操作系统中使用二进制包安装就可以避免这个问题，原因是在安装二进制包时，这些权限会被提前设置好。
## 1.2 Time for action – 修改日志文件的所有权(changing the ownership of log files)
通过更改日志目录和文件的所有权，可以很快地解决此问题。Squid要么由用户nobody运行，要么使用Squid配置文件中的cache-effective-user指令运行。
```Shell
chown –R nobody:nobody /opt/squid/var/logs/
```
>注意：
>根据你安装的squid的日志目录替换用户名、组名。

### What just happened？
我们了解到Squid应该拥有包含日志文件的目录的所有权，以便能够正确地记录消息。我们还学习了如何使用chown命令更改所有权。
## 1.3 无法确定主机名(Could not determine hostname)
此问题出现通常会伴随着另一个问题，如下所示：
```Shell
FATAL: Could not determine fully qualified hostname.  Please set 'visible_hostname'
Squid Cache (Version 3.1.10): Terminated abnormally.
```
当squid无法绑定到确定的IP地址所限定的主机名时，会发生这样的问题，注意了，对于squid 3.2之后的版本，这个错误将从致命转为告警，squid仍将使用localhost名称运行。此问题可以使用配置文件中visible_hostname命令设置合适的主机名来解决，如下所示：
```Shell
visible_hostname proxy.example.com
```
前面提供的主机名现在应该有DNS记录将其解析为代理服务器的IP地址。在一组代理集群中，该主机名对于每个代理服务器都应该是唯一的，以解决IP转发的问题。
## 1.4 无法创建交换目录(Cannot create swap directories)
当我们尝试使用squid命令创建新的交换目录时，可能会出现如下错误：
```Shell
[root@saini ~]# /opt/squid/sbin/squid -z
2010/11/10 00:42:34| Creating Swap Directories
FATAL: Failed to make swap directory /opt/squid/var/cache: (13) Permission denied
[root@saini ~]#
```
从前面的错误消息中可以清楚地看到，Squid没有足够的权限来创建交换目录。
## 1.5 操作时间–修改缓存目录权限（Time for action – fixing cache directory permissions）

```Shell
mkdir /opt/squid/var/cache
chown nobody:nobody /opt/squid/var/cache
```
上面的命令是将创建缓存目录，并将所有权转移给squid用户。如果我们现在尝试创建交换目录，命令将成功并输出如下内容：
```Shell
[root@saini etc]# /opt/squid/sbin/squid -z
2010/11/10 00:44:16| Creating Swap Directories
2010/11/10 00:44:16| /opt/squid/var/cache exists
2010/11/10 00:44:16| Making directories in /opt/squid/var/cache/00
2010/11/10 00:44:16| Making directories in /opt/squid/var/cache/01
```
### Waht just happend?
我们学习了如何创建具有适当所有权的缓存目录，这样，squid可以创建交换目录，而不会出现任何问题。
```Shell
2010/11/10 00:33:56| /opt/squid/var/cache: (2) No such file or directory
FATAL:  Failed to verify one of the swap directories, Check cache.log
        for details.  Run 'squid -z' to create swap directories
        if needed, or if running Squid for the first time.
Squid Cache (Version 3.1.10): Terminated abnormally.
```
这个错误通常发生在一下情况:
- 第一次运行squid时没有创建交换目录
- 使用cache_dir指令更新（添加/修改）现有交换目录后运行squid。
## 1.6 创建交换目录
通过下面的命令可以修复此错误
```Shell
squid -z
```
每次我们添加新的交换目录或修改配置文件中的cache_dir时，都应该运行这个命令。如果我们在运行前一个命令之后运行squid，一切都会好起来的。
### What just happened?
我们了解到，每当修改squid缓存目录时，都应该先运行squid -z，这样squid就可以正确地创建交换目录。
## 1.7 地址已经使用（Address already in use）
另一个常见的错误是地址已在使用中、无法绑定套接字或无法打开HTTP端口，如下所示：
```Shell
2010/11/10 01:04:20| commBind: Cannot bind socket FD 16 to [::]:8080: (98) Address already in use
FATAL: Cannot open HTTP Port
Squid Cache (Version 3.1.10): Terminated abnormally.
```
当我们启动Squid时，它试图将自己绑定到一个或多个网络接口，该端口是在squid 的配置文件中http_port命令指定的端口。前面提到的错误发生在另一个程序已经在监听Squid试图绑定到的端口时。

为了解决这个问题，我们首先要找出哪个程序正在监听有问题的端口。找到监听端口的程序的过程取决于我们使用的操作系统。以下方法用于常用的操作系统：
- 对于基于Linux的操作系统，我们可以使用以下命令：
  ```Shell
  lsof –i :8080
  ```
  不要忘记用squid占用的的端口号替换 8080。
- 对于Openbsd和Netbsd，我们可以使用fstat命令，如下所示：
  ```Shell
  fstat | grep 8080
  ```
这将给我们一个涉及端口8080的连接列表。
- 对于Freebsd和Dragonflybsd
用于确定在FreeBSD和DragonFlyBSD端口上侦听的程序的程序是sockstat，可以如下使用：
```Shell
sockstat -4l | grep 8080
```
前面的命令将向我们显示在端口8080上侦听的程序。

一旦确定是谁在侦听8080端口，我们就可以通过如下办法解决：
- 如果程序很重要，我们可能需要使用http_port指令更改squid http端口，然后重新启动squid。
- 关闭占用端口8080端口的程序，然后启动Squid。但是，这可能会影响使用其他程序提供的服务的客户机。
如果我们在配置文件中使用IP和/或通配符配置同一端口两次，也可能发生此问题。所以，最好将配置文件也加倍
### What just happened
我们了解了lsof、fstat和sockstat命令的用法，以找出在系统上特定端口上侦听的程序。我们还学习了当另一个程序在同一个端口上监听时，使Squid工作的可能方法。
## 1.8 带下划线的URL导致无效的URL(URLs with underscore results in an invalid URL)
默认配置squid不会发生此错误，但当我们强制squid根据标准检查URL时，可能会发生此错误。在公共DNS系统中，不允许使用下划线。只有当本地解析程序被配置为允许时，它才对本地解析主机有效。关于这个问题有两个重要的指示。让我们看看它们
#### 执行主机检查（Enforce hostname checks）
强制squid根据标准检查每个主机名， 指令是check_hostname。squid默认表现不只是将主机名限制为标准情况，而且当check_hostname设置为on时，squid将强制检查非法的主机名的URL请求，这将导致无效的URL消息。为了解决这个问题，我们将check_hostname设置为off，这样squid就不会强制执行检查。
#### 允许下划线（Allow underscore）
另一个设置域名中是否允许带下划线的指令是allow_underscore。squid默认是允许域名带下划线的。如果不想在域名中带下划线，可以将这个选项设置为off，要解决 URLs with underscore results in an invalid URL， 需要将此选项默认设置为on。
>注意:
>只有当check_name指令设置为on时，才能使用allow_underscore。

## 1.9 squid会慢慢变慢(Squid becomes slow over time)
当我们视图从系统中获取大量信息时，会出现这个问题。在大多数情况下，发生这种情况是因为我们将cache-mem 设置的值太高，并且没有足够的内存供其他进程正常运行，使得整个系统内存不足。
如前几章所述，cache_mem 是用于在主内存中缓存Web文档的内存量，而squid占用的总内存大于cache_mem，当cache_mem设置过高时，会影响squid的运行。

我们可以通过以下三步来解决这个问题:
- 1 除了操作系统和其他基本进程所消耗的内存之外,我们应该分析系统上可用的总内存。然后，我们应该相应地设置缓存内存，以便有足够的空闲内存供Squid和其他进程在不进行任何交换的情况下执行。
- 2 其次，我们可以尝试使用以下memory_pools指令关闭内存池：
  ```Shell
  memory_pools off
  ```
- 3 我们知道squid在主内存上保存了所有在磁盘上缓存文件的索引。因此，磁盘缓存越大，内存中索引占用就越大，如果前面两种方法都不起作用，那么我们可以尝试减少缓存目录的大小。
## 1.10 请求或者回复太大（The request or reply is too large）
有时，客户机会定时的收到"The request or reply is too large"这样的错误信息，这样的错误时因为当回复/请求的头或者正文超过了允许的最大值时，会发生此错误。

和这个错误相关的squid 配置文件指令是request_header_max_size、request_body_max_size、reply_header_max_size和reply_body_max_size ，合理的设置这些值就可以解决此问题。
## 1.11 拒绝访问代理服务器(Access denied on the proxy server)
有时会遇到棘手的问题，比如，所有的客户通过我们的代理服务器访问运行我们代理服务器上的网站时，可能被拒绝。这是因为设置squid时，我们的ACL允许所有网络，但就是忘记了squid服务器的IP地址。因此，我们可以通过扩展squid提供的ACL来解决这个问题，包括给我们的代理服务器分配其他的IP地址。
注意，在修改完ACL后要重新启动squid守护进程。
## 1.12 到达同级代理服务器时连接被拒绝（Connection refused when reaching a sibling proxy server）
当使用cache_peer 指令添加同级代理服务器时，如果我们恰好给同级代理服务器输入错误的HTTP端口，和正确的ICP端口，那么ICP通信正常使得我们的缓存以为配置是正确的。但是由于错误的HTTP端口将导致拒绝连接。即使我们的配置正确，并且我们的兄弟姐妹更改了他们的HTTP端口，也可能发生这种情况。再次检查HTTP端口将修复此解决方案。
# 二、调试问题
大多数情况下，我们会遇到众所周知的问题，这些问题是配置错误或操作系统限制的结果。因此，通过调整配置文件可以很容易地解决这些问题。但是，有时我们可能会遇到无法直接解决的问题，或者我们甚至无法通过查看日志文件来识别这些问题。

默认情况下，Squid只将基本信息记录到cache.log。为了检查或调试问题，我们需要增加日志的冗长性，以便Squid可以告诉我们更多它正在采取的操作，这可能帮助我们找到问题的根源。我们可以使用squid配置文件中的debug_options指令，方便地从squid提取有关其操作的信息。
让我们看一下debug选项格式：
```Shell
debug_options rotate=N section,verbosity [section,verbosity]...
```
- rotate  参数指定cache.log的文件数，n默认是1。rotate 选项有助于防止在冗长级别较高时由于日志消息过多而浪费磁盘空间。
- section 参数是一个整数，用于标识squid的组件。当section为all时表示squid的所有组件。
- verbosity 参数表示每个部分的级别，也是一个整数
让我们看看不同冗长级别的含义:

| verbosity level | Description |
| ---- | ---- |
| 0 | 只记录关键或致命的消息 |
| 1 | 记录警告和重要问题 |
| 2 | 记录次要问题、恢复和定期的高级操作 |
| 3-5 |  几乎所有有用的东西都包含在冗长的第5级中 |
| 6-9 | 在冗长的5级以上，这是非常冗长的。详细描述了单个事件、信号等 |


一下是默认配置:
```Shell
debug_options rotate=1 ALL,1
```
squid 的配置中verbosity 设置为1，这意味着Squid将尝试记录尽可能少的信息量。
section 的值可以查看源码文件来确定，在大多数源文件中，我们可以找到注释行，如下面的示例所示，它来自access_log.cc：
```Shell
/*
...
  * DEBUG: section 46    Access Log
...
*/
```
上面的注释告诉我们，访问日志section号是46。在squid源代码的doc/debug-section s.txt中可以找到节号和相应的squid组件的列表。下表列出了Squid 3.1.10版的一些重要章节号：
| Section number | squid组件 |
| ---- | ---- |
| 0 |  Announcement Server, Client Database, Debug Routines, DNS Resolver Daemon, UFS Store Dump Tool |
| 1 | 主循环， 启动程序 |
| 2 | Unlink Daemon | 
| 3 | Configuration File Parsing, Configuration Settings |
| 4 | Error Generation |
| 6 | Disk I/O Routines |
| 9 | File Transfer Protocol (FTP) |
| 11 | Hypertext Transfer Protocol (HTTP) |
| 12 | Internet Cache Protocol (ICP) |
| 14 | IP Cache, IP Storage, and Handling |
| 15 | Neighbor Routines |
| 16 | Cache Manager Objects |
| 17 | Request Forwarding |
| 18 | Cache Manager Statistics |
| 20 | Storage Manager, Storage Manager Heap-based replacement, Storage Manager Logging Functions, Storage Manager MD5 Cache Keys, Storage Manager Swapfile Metadata, Storage Manager Swapfile Unpacker, Storage Manager Swapin Functions, Storage Manager Swapout Functions, Store Rebuild Routines, Swap Dir base object |
| 23 |  URL Parsing, URL Scheme parsing |
| 28 | Access Control |
| 29 | Authenticator, Negotiate Authenticator, NTLM Authenticator |
| 31 | Hypertext Caching Protocol |
| 32 | Asynchronous Disk I/O |
| 34 |  Dnsserver interface |
| 35 | FQDN Cache |
| 44 | Peer Selection Algorithm |
| 46 | Access Log |
| 50 | Log file handling |
| 51 | Filedescriptor Functions |
| 55 | HTTP Header |
| 56 | HTTP Message Body |
| 57 | HTTP Status-line |
| 58 | HTTP Reply (Response) |
| 61 | Redirector |
| 64 | HTTP Range Header |
| 65 | HTTP Cache Control Header |
| 66 | HTTP Header Tools |
| 67 | String |
| 68 | HTTP Content-Range Header |
| 70 | Cache Digest |
| 71 | Store Digest Manager |
| 72 | Peer Digest Routines |
| 73 | HTTP Request |
| 74 | HTTP Message |
| 76 | Internal Squid Object handling |
| 78 | DNS lookups, DNS lookups; interacts with lib/rfc1035.c |
| 79 | Disk IO Routines, Squid-side DISKD I/O functions, Squid-side Disk I/O functions, Storage Manager COSS Interface, Storage Manager UFS Interface |
| 84 | Helper process maintenance |
| 89 | NAT / IP Interception |
| 90 | HTTP Cache Control Header, Storage Manager Client-Side Interface |
| 92 | Storage File System |
## 2.1 调试HTTP请求
```Shell
debug_options ALL,1 11,5
```
在修改完squid的配置后重新启动squid服务，现在，我们尝试使用代理服务器浏览www.example.com，将在cache.log文件中注意到类似以下的输出。请注意，我们已经从以下日志消息中删除了时间戳，以便更清晰地查看：
```Shell
httpStart: "GET http://www.example.com/"
http.cc(86) HttpStateData: HttpStateData 0x8fc4318 created
httpSendRequest: FD 14, request 0x8f73678, this 0x8fc4318.
The AsyncCall HttpStateData::httpTimeout constructed, this=0x8daead0 [call315] 
The AsyncCall HttpStateData::readReply constructed, this=0x8daf0c8 [call316] 
The AsyncCall HttpStateData::SendComplete constructed, this=0x8daf120 [call317] 
httpBuildRequestHeader: Host: example.com
httpBuildRequestHeader: User-Agent: Mozilla/5.0 (X11; U; Linux i686; en-US; rv:1.9.2.3) Gecko/20100403 Fedora/3.6.3-4.fc13 Firefox/3.6.3 GTB7.1
httpBuildRequestHeader: Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
httpBuildRequestHeader: Accept-Language: en-us,en;q=0.5
httpBuildRequestHeader: Accept-Encoding: gzip,deflate
httpBuildRequestHeader: Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
httpBuildRequestHeader: Keep-Alive: 115 
httpBuildRequestHeader: Proxy-Connection: keep-alive
httpBuildRequestHeader: If-Modified-Since: Fri, 30 Jul 2010 15:30:18 GMT
httpBuildRequestHeader: If-None-Match: "573c1-254-48c9c87349680"
comm.cc(166) will call HttpStateData::SendComplete(FD 14, data=0x8fc4318) [call317]
entering HttpStateData::SendComplete(FD 14, data=0x8fc4318)
AsyncCall.cc(32) make: make call HttpStateData::SendComplete [call317]
...
```
### What just happened?
我们学习了使用debug_options指令为cache.log文件中的HTTP请求生成更多的调试输出。同样，我们也可以调试Squid的其他组件。
现在，让我们学习一种使用debug_options指令和cache.log调试访问控件的方法。
## 2.2 访问控制调试
通常，使用不同的ACL类型构建ACL非常容易，而且会按照预期正常工作。但是随着我们的配置越来越大，ACL会变得混乱，很难指出问题的真正原因，例如：拒绝访问的消息或允许拒绝访问对象。要在这种情况下调试我们的ACL，我们可以利用debug_options指令，以便通过squid看到ACL的逐步处理。我们将学习调试我们示例的配置。

请考虑配置文件中的以下访问控制行：
```Shell
acl example dstdomain .example.com
acl png urlpath_regex -i \.png$

http_access deny png example
http_access allow localhost
http_access allow localnet
http_access deny all
```
如果我们查阅squid组件的 section 表，则访问控制的节号为28。因此，我们将在配置文件中添加以下行：
```Shell
debug_options ALL,1 28,3
```
以上配置将访问控制部分的详细级别设置为3，其他组件的详细级别设置为1。一旦我们添加了上一行配置，就需要重启squid服务。

现在，打开浏览器并尝试访问URL http://www.example.com/default.png。我们会得到一个拒绝访问的页面。现在，如果我们查看cache.log文件，我们可以找到以下类似的行：
```Shell
...
1.	ACLChecklist::preCheck: 0x8fa3220 checking 'http_access deny png example'
2.	ACLList::matches: checking png
3.	ACL::checklistMatches: checking 'png'
4.	aclRegexData::match: checking '/default.png'
5.	aclRegexData::match: looking for '\.png$'
6.	aclRegexData::match: match '\.png$' found in '/default.png'
7.	ACL::ChecklistMatches: result for 'png' is 1
8.	ACLList::matches: checking example
9.	ACL::checklistMatches: checking 'example'
10.	aclMatchDomainList: checking 'www.example.com'
11.	aclMatchDomainList: 'www.example.com' found
12.	ACL::ChecklistMatches: result for 'example' is 1
13.	aclmatchAclList: 0x8fa3220 returning true (AND list satisfied)
...
```
>注意：
>请注意，为了便于查看，时间戳已从日志消息中删除，并且这些消息的编号仅用于解释目的

现在，让我们尝试理解前面列出的不同日志消息的含义。
在第一行中，Squid说它将根据访问规则 http_access deny png example 处理当前请求。
在第二行中，squid获取规则png中使用的第一个acl以进行进一步处理。
在第四行中，squid说acl png正在对照/default.png进行检查，这是我们请求的URL中的URL路径。
在第六行中，Squid记录它在当前请求的URL路径中找到了与aclppng匹配的内容。
第七行声明acl png的结果，即1，这意味着acl已成功匹配。由于无法确定访问规则的确切结果，Squid将进一步处理该规则。
第八行和第九行表示现在将处理 example ACL。
第十行表示Squid将把ACL example与www.example.com进行匹配，www.example.com是我们请求中的目标域。
第十一行说找到了匹配。
第十三行表示返回了true，并且满足了AND列表（AND png和example ACLs上的操作）。
当前访问规则http_access deny png示例已匹配，将拒绝访问此URL。

所以，正如我们看到的，我们可以配置squid来记录消息，然后继续调试我们的ACL。

### What just happened?
我们刚刚学习了如何通过配置squid来调试访问控制，以便在处理单个访问规则时记录更多信息。
## 2.3 调试HTTP响应
尝试使用debug_options指令调试来自不同服务器的HTTP响应。
## 2.4 获取联机帮助并报告错误
如果我们真的陷入了一个squid错误，无法自行解决，那么我们应该考虑将该错误或问题发布到鱿鱼用户的邮件列表中。
有关与Squid相关的不同邮件列表的信息，请访问http://www.squid-cache.org/support/mailing-lists.html。
我们还应该考虑使用订阅Squid公告列表，从中我们可以获得关键的安全性并定期发布公告。
我们甚至可以参与 Squid开发，并通过订阅Squid开发人员的邮件列表了解Squid开发人员的工作。

关于Squid的另一个很好的在线信息来源是Squid wiki本身，可以在http://wiki.squid cache.org/上找到。
Squidwiki包含许多常见问题解答和各种操作系统的配置示例。
Squidwiki是一项社区工作，如果发现了过时的示例或配置，那么我们可以在网站bug下将它们报告给Bugzilla。或者，我们可以获得一个帐户，帮助改进wiki上的文章。

最后，如果我们真的碰到了Squid本身的一个bug，那么我们可以在http://bugs.squid cache.org/上提交一个详细的bug报告。
在提交新的bug之前，我们必须检查是否存在类似的bug。如果存在类似的bug，我们应该将bug报告附加到现有的bug中。
我们需要先创建一个bugzilla帐户，然后才能提交bug。此外，在提交bug报告时，我们应该提到以下信息，以便开发人员在试图找到bug源时能够掌握足够的信息：
- 我们安装的鱿鱼的版本和发行号。
- 操作系统名称和版本。
- 当错误发生时，我们正在做的操作是什么？
- 有没有复制这个错误的方法？如果是，请说明所有步骤。
- 任何跟踪备份或核心转储。请查看http://wiki.squid-cache.org/squidfaq/bugreporting以获取跟踪返回或核心转储。
- 可能有助于开发人员的任何其他相关系统特定信息。

在提交bug之后，我们应该定期检查bug页面的更新，并提供开发人员要求的任何附加信息。
# 三、小测试
- 当你遇到squid的问题时，你的第一步是什么？
    - 向Squid开发人员发送有关它的个人消息
    - 报告squid Bugzilla的一个bug
    - 检查cache.log文件是否有警告或错误
    - 重新启动Squid服务器
- 以下配置行有什么问题？
```Shell
debug_options 3,5 9,4 28,4 ALL,1
```
  - 这是错误的语法
  - 我们不能使用多节、冗长级别对
  - all不是整数，将导致配置错误
  - 虽然语法正确，但此配置行有语义错误。详细级别或所有部分将设置为1，因为最后提到了“全部”、“1”，并且将覆盖以前的详细级别。
- 一般来说，我们应该在生产模式中保持较低的冗长程度。为什么？
    - 当冗长级别设置为高时，Squid不会写入访问日志。
    - squid将淹没cache.log文件，在冗长级别较高时会导致不必要的磁盘空间消耗。
    - 在生产模式下部署时，Squid不支持高冗长级别。
    - 详细日志可以保护客户端的私有信息。
# 四、总结
我们了解了Squid用户面临的一些常见问题，以及如何通过修改Squid配置文件中的各种指令来快速解决这些问题。我们还通过cache.log了解了如何调试Squid的各个组件。

具体来说，我们包括：
- 一些常见问题及其解决办法
- 通过cache.log文件调试Squid的特定组件
- 使用在线资源获取帮助以解决Squid问题
- 向Squid开发人员报告错误
我们学习了各种跟踪Squid问题的方法，以及调试和解决问题的步骤。如果我们遇到问题，我们可以通过Squid 用户邮件列表与其他Squid用户联系。
