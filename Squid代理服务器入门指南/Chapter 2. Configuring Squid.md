# Chapter 2. Configuring Squid（第二章 配置Squid）
我们已经了解了如何编译Squid源代码和从源代码和二进制包安装Squid。在本章中，我们将学习如何根据给定网络的要求配置Squid。我们将了解Squid configure 文件中的一般语法，然后我们将继续探索优化Squid的不同选项。我们将简要介绍一些选项，但有专门的章节介绍这些选项，同时详细探讨其他选项。

在本章中的主要内容：
- 1 快速上手Squid
- 2 配置文件的语法
- 3 HTTP端口，最重要的配置命令
- 4 访问控制列表(ACL)
- 5 控制访问Squid的各种组件
- 6 缓存对等或者相邻的代理缓存
- 7 在主内存和硬盘中缓存Web文档
- 8 调整Squid以加强节约带宽和减少网络延迟
- 9 修改带有请求和响应的HTTP头
- 10 使用DNS服务器来配置Squid
- 11 与日志记录相关的一些命令
- 12 其他重要或常用的配置命令
# 一 、Quick start(快速上手)
在详细研究配置文件之前，让我们从最低配置开始。找到配置文件/opt/squid/etc/squid.conf，我们先进行添加必要的修改，以快速设置最小代理服务器。
```Shell
cache_dir ufs /opt/squid/var/cache/ 500 16 256
acl my_machine src 192.0.2.21 # Replace with your IP address
http_access allow my_machine
```
在配置文件的顶部添加(确保我们相应的把IP地址修改了)。现在，我们需要创建缓存目录。我们可以通过使用一下命令创建：
```Shell
$ /opt/squid/sbin/squid -z
```
我们现在可以运行代理服务器了，可以通过运行以下命令来完成这一点：
```Shell
$ /opt/squid/sbin/squid
```
Squid 将开始监听机器上所有网络接口上的端口3128（默认）。现在，我们可以将浏览器配置为使用我们主机IP作为机器的IP地址，端口3128的quid作为HTTP代理服务器。

配置浏览器后，尝试浏览http://www.example.com/。我们已经将Squid配置为HTTP代理服务器，尝试浏览http://www.example.com:897/ 并观察您收到的消息。显示的消息是Squid发送给您的拒绝访问消息。

现在，让我们继续详细了解配置文件。
# 二、 配置文件的语法
Squid的配置文件通常位于/etc/squid/squid.conf、/usr/local/squid/etc/squid.conf或$prefix/etc/squid.conf，其中$prefix是传递给--prefix选项的值，该选项在编译Squid之前传递给configure命令。

在较新的Squid版本，Squid.conf的文档化版本，称为squid.conf.documented，可以在squid.conf旁边找到。在本章中，我们将介绍配置文件中可用的一些导入指令。有关配置文件中使用的所有指令的详细说明，请查看http://www.squid-cache.org/doc/config/。

Squid文档化的配置文件中的语法类似于Linux/Unix上的许多其他程序。通常，在配置文件中使用的指令前面有几行注释是相当有用的。这使得配置指令变得很容易理解，甚至对于不熟悉使用配置文件的人来说也可以读懂。通常，我们只需要读注释理解特定的指令使用合适的选项。
Syntax of the configuration file
Squid's configuration file can normally be found at /etc/squid/squid.conf, /usr/local/squid/etc/squid.conf, or ${prefix}/etc/squid.conf where ${prefix} is the value passed to the --prefix option, which is passed to the configure command before compiling Squid.

In the newer versions of Squid, a documented version of squid.conf, known as squid.conf.documented, can be found along side squid.conf. In this chapter, we'll cover some of the import directives available in the configuration file. For a detailed description of all the directives used in the configuration file, please check http://www.squid-cache.org/Doc/config/.

The syntax for Squid's documented configuration file is similar to many other programs for Linux/Unix. Generally, there are a few lines of comments containing useful related documentation before every directive used in the configuration file. This makes it easier to understand and configure directives, even for people who are not familiar with configuring applications using configuration files. Normally, we just need to read the comments and use the appropriate options available for a particular directive.

The lines beginning with the character # are treated as comments and are completely ignored by Squid while parsing the configuration file. Additionally, any blank lines are also ignored.

Copy
# Test comment. This and the above blank line will be ignored by Squid.
Let's see a snippet from the documented configuration file (squid.conf.documented)

Copy
#  TAG: cache_effective_user
# If you start Squid as root, it will change its effective/real
# UID/GID to the user specified below.  The default is to change
# to UID of nobody.
# see also; cache_effective_group
#Default:
# cache_effective_user nobody
In the previous snippet, the first line mentions the name of the directive, that is in this case, cache_effective_user. The lines following the tag line provide brief information about the usage of a directive. The last line shows the default value for the directive, if none is specified.

Types of directives
Now, let's have a brief look at the different types of directives and the values that can be specified.

Single valued directives
These are directives which take only one value. These directives should not be used multiple times in the configuration file because the last occurrence of the directive will override all the previous declarations. For example, logfile_rotate should be specified only once.

Copy
logfile_rotate 10
# Few lines containing other configuration directives
logfile_rotate 5
In this case, five logfile rotations will be made when we trigger Squid to rotate logfiles.

Boolean-valued or toggle directives
These are also single valued directives, but these directives are generally used to toggle features on or off.

Copy
query_icmp on
log_icp_queries off
url_rewrite_bypass off
We use these directives when we need to change the default behavior.

Multi-valued directives
Directives of this type generally take one or more than one value. We can either specify all the values on a single line after the directive or we can write them on multiple lines with a directive repeated every time. All the values for a directive are aggregated from different lines:

Copy
hostname_aliases proxy.exmaple.com squid.example.com
Optionally, we can pass them on separate lines as follows:

Copy
dns_nameservers proxy.example.com
dns_nameservers squid.example.com
Both the previous code snippets will instruct Squid to use proxy.example.com and squid.example.com as aliases for the hostname of our proxy server.

Directives with time as a value
There are a few directives which take values with time as the unit. Squid understands the words seconds, minutes, hours, and so on, and these can be suffixed to numerical values to specify actual values. For example:

Copy
request_timeout 3 hours
persistent_request_timeout 2 minutes
Directives with file or memory size as values
The values passed to these directives are generally suffixed with file or memory size units like bytes, KB, MB, or GB. For example:

Copy
reply_body_max_size 10 MB
cache_mem 512 MB
maximum_object_in_memory 8192 KB
As we are familiar with the configuration file syntax now, let's open the squid.conf file and learn about the frequently used directives.

Have a go hero – categorize the directives
Open the documented Squid configuration file and find out at least three directives of each type that we discussed before. Don't use the directives already used in the examples.

# 三、
