使用Squid代理服务器，减少带宽使用并更快地交付最频繁请求的网页。本指南将向您介绍缓存系统的基本原理，并帮助您从Squid获得最大的收益。
- 发现哪种配置选项最适合您的网络
- 通过命令行选项更好地控制Squid，帮助您调试Squid
- 设计一个访问控制列表（ACL）来决定哪些用户被授予访问不同端口的权限。
- 了解日志文件和日志格式，以及如何根据需要定制它们
- 了解Squid的缓存管理器Web界面，这样您就可以实时监控流量，防止出现任何问题。
- 实现要在大型网络中使用的缓存层次结构
- 在加速器模式下使用Squid快速提升速度非常慢的网站的性能
- 编写自己的URL重写器来自定义Squid的行为
- 了解如何排除Squid故障
# 关于代理服务器:
Squid代理服务器使您能够缓存Web内容，并在后续请求时快速返回。系统管理员经常会遇到延迟和使用的带宽过大的问题，但是Squid通过在本地处理请求来解决这些问题。
通过在加速器模式下部署Squid，请求的处理速度比普通Web服务器快，使您的网站比其他人的执行速度更快！Squid代理服务器3.1初学者指南将帮助您安装和配置Squid，以便优化它以提高网络性能。
Squid代理服务器减少了您必须投入的工作量，从而节省了您的时间来最大限度地利用网络。无论您是只运行一个站点，还是负责整个网络，Squid都是一个非常宝贵的工具，可以极大地提高性能。
缓存和性能优化通常需要开发人员做大量的工作，但Squid会为您做所有这些工作。这本书将向你展示如何通过为你的网络定制来最大限度地利用鱿鱼。
您将了解可用的不同配置选项以及透明和加速模式，这些模式使您能够专注于网络的特定区域。将代理服务器应用于大型网络可能需要很多工作，因为您必须决定在何处设置限制以及哪些人应该具有访问权限，
但是本书中的简单示例将指导您逐步完成这些工作，以便您拥有一个代理服务器，该服务器通过你完成这本书的时间。
# 特性：
- 通过自定义Squid的访问控制列表和帮助程序，充分利用网络连接
- 设置和配置Squid以使您的网站更快、更高效地工作
- 无需事先了解Squid或代理服务器
- 初学者指南系列的一部分：许多实用的、易于理解的示例，并附有屏幕截图

# 关于作者
Kulbir Saini是一家位于印度海得拉巴的企业家。他在管理系统和网络基础设施方面有丰富的经验。除了作为一名自由开发者，他还为许多初创企业提供服务。通过他的博客，他一直是各种开源项目文档的积极贡献者，最著名的是Fedora项目和Squid。除了电脑，他的生活实际上围绕着电脑，他喜欢和朋友们去偏远的地方旅行。有关详细信息，请访问http://saini.co.in/。

# 目录
## 1 Squid入门
## 2 配置Squid
## 3 运行Squid
## 4 开始使用Squid强大的控制列表和访问规则
## 5 了解日志文件和日志格式
## 6 管理Squid和监控流量
## 7 通过身份验证保护您的Squid代理服务器
## 8 构建Squid缓存的层次结构
## 9 反代理模式下的Squid
## 10 Squid在拦截模式
## 11 编写URL重定向器和重写器
## 12 Squid故障排除

# 第一章 Squid入门
在本章中，我们将了解代理服务器和Web缓存的一般工作方式。根据我们在前言中了解到的系统要求，我们将继续为自己的操作系统下载正确的squid包。我们将学习如何编译和构建附加特性的Squid。我们还将学习从源代码手动编译squid的优势，而不是使用预编译的二进制包。

在最后一节中，我们将学习如何编译Squid源码包并进行安装Squid。安装是开始使用鱿鱼的关键部分。有时，我们需要使用自定义标志编译Squid，这取决于环境要求。
So let's get started with the real stuff.(让我们正式开始)
## Proxy server
代理服务器是位于请求Web文档的客户机和为文档提供服务的目标服务器（另一个计算机系统）之间的计算机系统。代理服务器以其最简单的形式简化了客户机和目标服务器之间的通信，而无需修改请求或回复。当我们从目标服务器发起资源请求时，代理服务器会劫持我们的连接，并将自己表示为目标服务器的客户机，代表我们请求资源。如果收到回复，代理服务器会将其返回给我们，让我们感觉我们已经与目标服务器进行了通信。

在高级表单中，代理服务器可以根据各种规则筛选请求，并且仅在可以根据可用规则验证请求时才允许通信。规则通常基于客户端或目标服务器的IP地址、协议、Web文档的内容类型、Web内容类型等。

![Proxy Server](Image/Proxy Server.png)

如上图所示，客户端不能直接向Web服务器发出请求。为了方便客户端和Web服务器之间的通信，我们使用代理服务器将它们连接起来，代理服务器充当客户端和Web服务器的通信媒介。

有时，代理服务器可以修改请求或答复，甚至可以在本地存储来自目标服务器的答复，以便在以后的阶段满足来自相同或其他客户机的相同请求。在本地存储回复以备日后使用，称为缓存。缓存是代理服务器常用的一种技术，用于节省带宽、增强Web服务器的功能以及改善最终用户的浏览体验。

- 减少带宽使用
- 通过缓存Web文档来实现减少页面加载时间来增强用户的浏览体验
- 增强网络访问策略
- 监控用户流量或上报单个用户或组的Internet使用情况
- 通过不将用户的计算机直接暴露于Internet来增强用户隐私
- 在不同的 Web服务器之间分配负载以减少单个服务器上的负载
- 增强性能不佳的Web服务器
- 使用集成的病毒/恶意软件检测系统筛选请求或答复
- 跨多个Internet连接的负载均衡网络流量
- 在局域网内中继流量
简单地说，代理服务器是客户机和目标服务器之间的代理，它具有一系列规则，根据这些规则验证每个请求或回复，然后相应地允许或拒绝访问。
## Reverse proxy (反向代理)
反向代理是一种在代理服务器本地存储来自Web服务器的答复、资源的技术。以便后续对同一资源的请求可以从代理服务器上的本地副本得到满足，有时甚至不需要实际联系Web服务器。代理服务器或Web缓存在提供缓存副本之前必须检查Web文档的本地存储副本是否仍然有效。

Web缓存主要用于：(Web caching is mostly used:)
- To reduce bandwidth usage. A large number of static web documents like CSS and JavaScript files, images, videos, and so on can be cached as they don't change frequently and constitutes the major part of a response from a web server.(减少带宽使用。大量静态Web文档（如css和javascript文件、图像、视频等）可以缓存，因为它们不经常更改，并且构成Web服务器响应的主要部分。)
- By ISPs to reduce average page load time to enhance browsing experience for their customers on Dial-Up or broadband.(通过ISP减少平均页面加载时间，增强用户在拨号或宽带上的浏览体验。)
- To take a load off a very busy web server by serving static pages/documents from a proxy server's cache.（通过从代理服务器的缓存中提供静态页面/文档来减轻非常繁忙的Web服务器的负载。）
## Getting Squid(获取Squid)
Squid可以从Squid的官方网站、世界各地的各种Squid镜像以及几乎所有流行操作系统的软件库中以多种形式（压缩源文件、版本控制系统的源代码、诸如RPM、DEB等二进制软件包）提供。许多Linux/Unix发行版附带有Squid。

可以从Squid官网下载各种版本Squid。为了充分使用Squid，最好从版本控制系统中查看最新的源码，以便我们获取最新的个功能和bug修复。但是注意，最新的源码可能不稳定，甚至不能正常工作。尽管VCS中的代码有助于学习或测试Squid的新功能，但强烈建议您不要将VCS中的代码用于产品的部署。

如果我们想安全起见，我们应该从旧版本下载最新的稳定版本或稳定版本。稳定的版本通常在发布之前进行测试，并且应该是开箱即用的。稳定版本可以直接用于生产部署。
## Time for action – identifying the right version (首先，确定正确的版本)
http://www.squid-cache.org/Versions/. 这个网站中维护这Squid的可用版本。对于生产环境，我们应该只使用稳定版本部分列出的版本。如果我们想在我们的环境中测试新的Squid特性，或者我们打算向Squid社区提供关于新版本的反馈，那么我们应该使用一个beta版本。

![](/Image/Stable Versions.png)

正如我们在前面的屏幕截图中看到的，该网站包含稳定版本的第一个生产发布日期和最新发布日期。如果我们单击任何一个版本，我们将被指向一个页面，其中包含该特定版本中所有版本的列表。让我们看一下3.1版的页面：

![](/Image/SquidVersion3_1.png)

对于每个版本，以及发布日期，都有用于下载源归档压缩文件的链接。

不同版本的Squid可能有不同的特征。例如，许多特性在Squid的2.7版本可能提供，但在Squid3.x中可能就不提供。某些功能可能已被弃用，或随着时间的推移而变得多余，它们通常会被删除。另一方面，Squid3.x增加新的特性同时通过修改已有的特性。

因此，我们应该始终以最新版本为目标，但根据环境的不同，我们可能会选择稳定版本或测试版。此外，如果我们需要在最新版本中不可用的特定功能，我们可以从其他分支中的可用版本中进行选择。
### What just happened?
我们在Squid的官方网站上简要浏览了包含Squid不同版本和版本的页面。我们还了解了应该下载哪些版本和版本，并将其用于不同的使用类型。
### Methods of obtaining Squid（获取Squid的方法）
确定了应该用于编译和安装的squid版本之后，让我们看看如何获得squid 3.1.10版。
### Using source archives（使用源码）
压缩源文件是获得鱿鱼最流行的方法。要下载源文件，请访问Squid下载页面，http://www.squid-cache.org/download/。本网页提供了从官方网站或全球可用镜像下载Squid不同版本和版本的链接。我们可以使用http或ftp来获取Squid源文件。
## Time for action – downloading Squid(下载Squid)
现在我们将从Squid的官方网站下载Squid 3.1.10：
1. 访问 http://www.squid-cache.org/Versions/.
2. 点击Version 3.1，截图如下：
   
   ![](/Image/Version 3_1.png)

3. 我们将进入一个显示3.1版中各种版本的页面。下载列中显示文本tar.gz的链接是指向Squid 3.1.10版压缩源归档的链接，如下图所示：
   
   ![](/Image/tar.gz.png)

4. 要使用Web浏览器下载Squid3.1.10，只需单击链接。
5. 或者，我们可以使用wget从命令行下载源归档文件，如下所示：
```Shell
wget http://www.squid-cache.org/Versions/v3/3.1/squid-3.1.10.tar.gz
```
### What just happened?
我们成功地从Squid的官方网站检索到了Squid版本3.1.10。检索其他稳定版本或beta版本的过程非常相似。
### 从Bazaar VCS获取最新的源代码
高级的用户对于使用Bazaar版本控制系统来获取最新的Squid源码非常有兴趣。如果对版本控制系统不熟悉，可以跳过这部分。Bazaar是一个流行的版本控制系统，用于跟踪项目历史并促进协作。从3.x版开始，Squid源代码已迁移到Bazaar。因此，为了从仓库中签出源代码，我们应该确保系统上安装了bazaar。要了解更多关于Bazaar或Bazaar安装和配置手册的信息，请访问Bazaar的官方网站http://bazaar.canonical.com/。

一旦设置好Bazaar，我们就应该从https://code.launchpad.net/squid/上镜像的Squid代码存储库下载Squid源码。从这里我们可以浏览Squid的所有版本和分支。
让我们熟悉页面布局：

![](/Image/SquidBazaar.png)

前面的截图所示， Name中Series：trunk表示开发分支，其中包含任在开发中的代码。Series：Mature表示分支机构是稳定的，可以用在产品中。

## Time for action – 使用 Bazaar 获取源码
现在我们已经熟悉了各种分支、版本和发布。让我们继续用bazaar检查源代码。要从任何分支下载代码，命令的语法如下：
```Shell
bzr branch lp:squid[/branch[/version]]
```
[branch]和[version]是可选参数。因此，如果我们想要得到分支3.1，那么命令如下：
```Shell
bzr branch lp:squid/3.1
````
前面的命令将从登陆页面下载源代码，可能需要相当长的时间，这取决于网络网速。如果想下载Squid 3.1.10版本，命令如下：
```Shell
bzr branch lp:squid/3.1/3.1.10
```
3.1是分支名称， 3.1.10是指定的squid版本。
### What just happened?
We learned to fetch the source code for any Squid branch or release using Bazaar from Squid's source code hosted on Launchpad.
### Have a go hero – fetching the source code(继续，获取源代码)
- 解决方案：
```Shell
bzr branch lp:squid/3.0/3.0.stable25
```
- 说明：如果我们浏览仓库上的特定版本，命令中要显示的指定版本号。
### 使用二进制包
Squid的二进制包是预先编译号的，可以直接安装。几乎所有的Linux/Unix操作系统的软件仓库中都提供二进制的安装包。根据操作系统的不同，选择稳定的，或者Beta版本的软件安装。
## 安装Squid(Installing Squid)
使用上一节中获取的源码编译安装Squid，或者也可以直接使用系统提供的包管理器安装已经编译好的二进制Squid。下面好好看看如何安装Squid。
### 从源码安装(Installing Squid from source code)
从源码安装需要三步骤：
1. 选择特性和操作系统特定的设置
2. 编译源码生产可执行文件
3. 将生成的可执行文件和其他必需的文件放在它们指定的位置，以便Squid正常工作。
我们可以使用自动化编译部署工具来执行上面的步骤。
### 编译Squid
编译SQUID是编译包含C/C++源代码和生成可执行文件的多个文件的过程。编译Squid非常简单，只需几个步骤即可完成。为了编译SQUID，我们需要一个兼容ANSI C/C++编译器。如果我们已经有一个GNU C/C++编译器（GNU编译器集合（GCC）和G++，它几乎在每一个Linux /UNIX操作系统上都是可用的），那么我们就准备开始实际编译了。
#### 为什么要选择编译？
与从二进制软件包安装squid相比，编译squid是一项有点痛苦的任务。但是，我们建议从源代码处编译Squid，而不是使用预编译的二进制文件。让我们来介绍一下从源代码处编译Squid的几个优点：
- 编译时，我们可以启用额外的功能特性，这些特性可能在预编译的二进制包中没有启用。
- 在编译时，我们还可以禁用特定环境不需要的额外功能。例如，我们可能不需要身份验证助手或ICMP支持。
- configure 探测系统的几个功能，并相应的启用和禁用它们，虽然预编译的二进制软件包将具有为编译源的系统检测到的功能。
- 使用 configure，我们可以指定Squid的安装位置。我们甚至可以在没有根用户或超级用户特权的情况下安装squid，这在预编译的二进制软件包中是不可能的。
虽然从源代码处编译安装squid比从二进制包安装squid有很多优点，但是二进制包有其自身的优点。例如，当我们处于损坏控制模式或危机情况下，我们需要让代理服务器快速启动和运行，使用二进制软件包进行安装将提供更快的安装。
### 解压下载的源码存档文件
如果获取的是源码存档压缩文件，则需要先解压提取出源码，然后进行配置编译。如果是使用Bazaar从仓库中获取Squid，则不需要执行此步骤。
```Shell
tar -xvzf squid-3.1.10.tar.gz
```
tar是一个常用的命令，用于提取各种类型的压缩文件。另一方面，它还可以用于将多个文件压缩到单个存档中。前面的命令将把归档文件提取到一个名为squid-3.1.10的目录中。
### 配置系统检查(Configure or system check)
配置系统检查是编译过程的第一步，在命令行运行./configure 命令实现。这个程序探测系统，确保安装了所需的软件包或库。它还检查系统功能，并收集有关系统体系结构和默认设置的信息，如可用的文件描述符等。在收集完所有信息后，生成makefiles，为下一步中使用它来编译Squid源代码。

运行不带任何参数的配置的configure将使用默认值。如果想更改默认的Squid设置，或者想禁用一些默认启用的可选功能，或者想在文件系统的另一个位置安装Squid，我们需要传递选项进行配置。使用以下命令查看可用选项和简短描述。

使用 ./configure --help 查看可用的配置选项。
```Shell
./configure --help | less
```
这条命令将显示configure的选项以及这些选项的简要说明的页面。使用上下箭头浏览信息。现在让我们一起讨论一些常用的配置选项：

#### --prefix
--prefix选项时最常用的选项。如果我们正在测试一个新版本，或者我们想要测试多个Squid版本，我们的系统上会安装多个Squid版本。为了识别不同的版本并防止版本之间的干扰或混淆，最好将它们安装在不同的目录中。例如，对于安装3.1.10版的squid，我们可以使用目录/opt/squid/3.1.10/，相应的configure命令将运行如下：
```Shell
./configure --prefix=/opt/squid/3.1.10/
```
同样，对于安装squid版本3.1，我们可以使用目录/opt/squid/3.1/。
>注意：
>从现在开始，变量${prefix}将表示我们安装Squid的位置，它就是执行./configure是的--prefix选项所表示的目录，如上面的命令所示。

Squid对不同类型的文件（如可执行文件和文档文件）的位置提供了更多的控制。它们的位置可以通过选项控制，例如--bindir、-sbindir等等。有关这些选项的详细信息，请查看配置帮助页。

现在，让我们检查可选功能和包。要启用任何可选功能，我们传递一个格式为--enable-FEATURE_NAME的选项，要禁用某个功能，选项格式为--disable-FEATURE_NAME 或--enable-FEATURE_NAME=no。例如，icmp是一个功能名称。
```Shell
./configure --enable-FEATURE # FEATURE will be enabled
./configure --disable-FEATURE # FEATURE will be disabled
./configure --enable-FEATURE=no # FEATURE will be disabled
```
Similarly, to compile Squid with an available package, we pass an option in the format --with-PACKAGE_NAME and to compile Squid without a package, we pass the option --without-PACKAGE_NAME. openssl is an example package name.
同样，为了用一个可用的包编译Squid，我们传递了一个格式为WITH-PACKAGE U NAME的选项，并且要编译没有包的Squid，我们传递了一个格式为WITH-PACKAGE U NAME的选项。openssl是一个包名称示例。

#### --enable-gnuregex
正则表达式用于构造Squid中的访问控制列表。如果我们运行的是基于Linux/Unix的现代操作系统，则无需担心此选项。但是，如果我们的系统没有对正则表达式的内置支持，我们应该使用--enable gnuregex启用对正则表达式的支持。
#### --disable-inline
Squid中使用了很对内联代码，这对于产品是很有用的。但是内联代码花费更长的时间编译，但是当我们只需要编译一次源代码就可以为产品使用设置Squid时，内联代码是有用的。当我们反复地编译Squid时，这个选项将在开发期间使用。
#### --disable-optimizations(禁止优化)
默认情况下，Squid是用编译器优化编译，这样可以获取更好的性能。但是，在调试问题或测试不同版本时应该使用此选项，因为这样可以减少编译时间。如果我们使用这个选项，--disable inline选项将被自动使用。
#### --enable-storeio(启用磁盘IO)
启用磁盘缓存时，Squid的性能严重依赖于磁盘I/O性能。Squid从缓存读取/写入文件的速度越快，满足Request所需的时间就越短，这反过(While in turn)来会导致更小的延迟。根据流量类型和使用情况，不同的存储技术可能会导致优化的性能各异。我们可以使用此选项构建支持各种存储I/O模型的Squid。请检查Squid源代码中的src/fs/目录以查找支持的存储I/O模型。
```Shell
./configure --enable-storeio=ufs,aufs,coss,diskd,null
```
#### --enable-removal-policies（启用删除策略）
在使用磁盘缓存时，我们指示Squid使用指定的磁盘空间来缓存Web文档。在一段时间内，空间被占用，而Squid仍需要更多的空间来缓存新文档。所以，Squid必须决定应该从缓存中删除或清除哪些旧文档，以便为存储新文档腾出空间。清除文件可以使缓存得到最大的使用价值的策略有许多种。策略基于堆和列表数据结构。默认情况下，列表数据结构处于启用状态。请检查squid源代码中的src/repl/目录以了解可用的删除策略。
```Shell
./configure --enable-removal-policies=heap,lru
```
#### --enable-icmp
此选项可以帮助确认对等的缓存和服务器之间大概的距离，从而估计出大概的延迟时间。只有网络中有对等的缓存时这才有用。
#### --enable-delay-pools(启用延迟池)
Squid使用延迟池来限制或控制一个客户机或一组客户机可以使用的带宽。延迟池就像是泄漏的桶，它将数据（Web流量）泄漏到客户机，并以受控的速率重新填充。当我们需要控制一组用户使用的带宽时，这些功能就派上用场了。
#### --enable-esi
此选项允许Squid使用边缘端包含（有关详细信息，请参阅http://www.esi.org）。如果启用此选项，则Squid将完全忽略来自客户端的缓存控制头。此选项仅用于在加速器模式下使用Squid时。
#### --enable-useragent-log(启用用户代理日志)
提供从客户端请求包中记录用户代理头的功能。
#### --enable-referer-log(启用引用日志)
如果启用此选项，Squid将能够从HTTP请求中写入引用头字段。
#### --disable-wccp(禁止Cisco Web缓存通信协议)
此选项禁用对Cisco Web缓存通信协议（WCCP）的支持。WCCP支持缓存之间的通信，这反过来有助于实现通信的本地化。默认情况下，启用WCCP支持。
#### --disable-wccpv2（禁止Cisco Web缓存通信协议V2版本）
与前面的选项类似，这将禁用对Cisco的WCCP版本2的支持。wccpv2是wccp的改进版本，内置支持负载平衡、扩展、容错和服务保证机制。默认情况下，启用WCCPv2支持。
#### --disable-snmp
在Squid Versions 3.X中，SNMP（简单网络管理协议）默认是启用的。SNMP在监测服务器和网络设备时非常受系统管理员欢迎。
#### --enable-cachemgr-hostname
缓存管理器(CGI)，利用网络接口管理Squid缓存和统计视图缓存的CGI，使用此选项设置访问缓存管理器的主机名，默认情况下，我们可以使用本地主机或者Squid服务器的IP地址访问缓存管理器Web接口。
```Shell
./configure --enable-cachemgr-hostname=squidproxy.example.com
```
#### --enable-arp-acl
Squid支持基于MAC(或者以太网Ethernet)地址构建的访问控制列表。此功能在默认情况下被禁用。如果我们想要基于以以太网地址控制客户端访问，我们应该启用这个特性。在学习Squid的同时启用这是一个好主意。
>注意：
>此选项将替换为 --enable-eui,默认启用eui。

#### --disable-htcp
Squid可以使用超文本缓存协议（HTCP）向相邻缓存发送和接收缓存摘要。此选项禁用HTCP支持。
#### --enable-ssl
Squid可以终止SSL连接。当Squid配置为反向代理模式时，Squid可以终止客户端启动的SSL连接，并在后端代表Web服务器处理它。这实质上意味着后端Web服务器将不必执行任何SSL工作，这意味着显著的节省计算资源。在这种情况下，Squid和后端Web服务器之间的通信将是纯HTTP，但客户端仍然将其视为与Web服务器的安全SSL连接。只有当Squid配置为在加速器或反向代理模式下工作时，这才有用。
#### --enable-cache-digests(启用缓存摘要)
缓存摘要是Squid与相邻的Squid以压缩文件的方式共享缓存的Web文档信息。
#### --enable-default-err-language(启用默认错误语言)
每当Squid遇到需要传递给客户端的错误(例如，找不到页面、拒绝访问或网络不可访问错误)时，Squid就会使用默认页面来显示这些错误。错误页以本地语言提供。此选项可用于指定所有错误页的默认语言。错误页的默认语言是英语。
```Shell
./configure --enable-default-err-language=Spanish
```
#### --enable-err-languages
默认情况下构建的Squid支持所有可用语言。如果我们只想用我们熟悉的语言构建Squid，那么我们可以使用这个选项。请检查Squid源代码中的errors/目录以了解可用的语言。
```Shell
/configure--enable err languages='English French German'
```
#### --disable-http-violations（禁用HTTP冲突）
Squid有配置选项，通过使用它们，我们可以通过替换HTTP请求或响应中的头字段来强制Squid违反HTTP协议标准。修补HTTP头违反了标准的HTTP规范。我们可以使用此选项禁用对各种HTTP冲突的支持。
#### --enable-ipfw-transparent（启用IPFW透明）
ipfirewall（ipfw）是由freebsd工作人员和志愿者维护的freebsd系统的防火墙应用程序。此选项在使用IPFW的系统上设置透明代理服务器时很有用。如果我们的系统没有ipfw，我们应该避免使用这个选项，因为squid将无法编译。默认的行为是自动检测，这可以很好地完成工作。
#### --enable-ipf-transparent(启用IP过滤器)
许多类Unix操作系统有IPFilter，IP过滤器（IPF）也是一个有状态的防火墙。它由netbsd、solaris等提供。如果我们的系统有IPF，那么我们应该启用这个选项，以便能够在透明模式下配置Squid。在系统上没有IPF的情况下启用此选项将导致编译错误。
#### --enable-pf-transparent（包过滤器）
Packet Filter(包过滤器)是另一个最初为OpenBSD开发的有状态防火墙应用程序。此选项在安装了PF的系统上实现透明代理服务器很有用。如果没有安装，则禁用此选项。
#### --enable-linux-netfliter
Netfilter是Linux内核2.4.x和2.6.x系列中的包过滤框架。此选项对于在基于Linux的操作系统上启用透明代理支持是非常有用的支持。
#### --enable-follow-x-forwarded-for
当代理转发HTTP请求时，代理在HTTP报文头中写入代理自己的基本信息和请求需要转发的客户机的基本信息。此选项使得Squid能够试着通过一个或多个代理服务器查找到转发请求的原始客户端的IP地址。
#### --disable-ident-lookups
#### --disable-internal-dns
#### --enable-default-hostsfile
#### --enable-auth
#### --enable-auth-basic
#### --enable-auth-ntlm
#### --enable-auth-negotiate
#### --enable-auth-digest
#### --enable-ntlm-fail-open
#### --enable-external-acl-helpers
#### --disable-translation
#### --disable-auto-locale
#### --disable-unlinkd
#### --with-default-user
#### --with-logdir
默认情况下Squid将把日志和错误报告写入到被设定好的文件${prefix}/var/logs/中。这个位置不同于所有其他进程和守护进程用来写日志的位置。为了快速访问squid日志，我们可能希望将它们放在默认的系统日志目录中，这在大多数基于Linux的操作系统中是/var/log/。请参阅以下语法示例以实现此目的：
```Shell
./configure --with-logdir=/var/log/squid/
```
#### --with-pidfile
Squid PID 文件的默认存储位置是${prefix}/var/run/squid.pid, 这不是存储PID文件的标准系统位置。在大多数基于Linux的操作系统中，PID文件都存储在/var/run/中。因此，我们希望使用下面选项来更改默认的pidfile位置：
```Shell
./configure --with-pidfile=/var/run/squid.pid
```
#### --with-aufs-threads
#### --without-pthreads
#### --with-openssl
#### --with-large-files
#### --with-filedescriptors



--disable-ident-lookups
This prevents Squid from performing ident lookups or identifying a username for every connection. Disabling this may prevent our system from a possible Denial of Service attack by a malicious client requesting a large number of connections.

--disable-internal-dns
Squid has its own implementation of DNS protocol and is capable of building DNS queries. If we want to use Squid's internal DNS, then we should not disable it. Otherwise, we can disable support for Squid's internal DNS feature by using this option and can use external DNS servers.

--enable-default-hostsfile
Using this option, we can select the default location of the hosts file. On most operating systems, it's located in the /etc/hosts directory.

Copy
./configure --enable-default-hostsfile=/some/other/location/hosts
--enable-auth
Squid supports various authentication mechanisms. This option enables support for authentication schemes. This configure option (and related enable auth options) are undergoing change.

Old Syntax
Previously, this option was used to enable authentication support and a list of authentication schemes was also passed. The authentication schemes from the list were then built during compilation.

Copy
./configure --enable-auth=basic,digest,ntlm
New Syntax
Now, this option is used only to enable global support for authentication and a list of authentication schemes is not passed along. The authentication scheme is enabled with the option --enable-auth-AUTHENTICATION_SCHEME where AUTHENTICATION_SCHEME is the name of the authentication scheme. By default, all the authentication schemes are enabled and the corresponding authentication helpers are built during compilation. Authentication helpers are external programs that can authenticate clients using various authentication mechanisms, against different user databases.

Copy
./configure --enable-auth
--enable-auth-basic
This option enables support for a Basic Authentication scheme and builds the list of helpers specified. If the list of helpers is not provided, this will enable all the possible helpers. A list of available helpers for this scheme can be found in the helpers/basic_auth/ directory in the Squid source code. To disable this authentication scheme, we can use --disable-auth-basic.

Copy
./configure --enable-auth-basic=PAM,NCSA,LDAP
If we want to enable this option but don't want to build any helpers, we should use "none" in place of a list of helpers.

Copy
./configure --enable-auth-basic=none
Previously, this option was known as --enable-basic-auth-helpers. The list of helpers is passed in a similar way.

Copy
./configure --enable-basic-auth-helpers=PAM,NCSA,LDAP
Note
The old and new option syntax for all other authentication schemes are similar.

--enable-auth-ntlm
Squid support for the NTLM authentication scheme is enabled with this option. The available helpers for this scheme reside in the helpers/ntlm_auth/ directory in the Squid source code. To disable NTLM authentication scheme support, use the --disable-auth-ntlm option.

Copy
./configure --enable-auth-ntlm=smb_lm,no_check
--enable-auth-negotiate
This option enables the Negotiate Authentication scheme. Details and syntax are similar to the above authentication scheme option.

Copy
./configure --enable-auth-negotiate=kerberos
--enable-auth-digest
This option enables support for Digest Authentication scheme. Other details are similar to the above option.

--enable-ntlm-fail-open
If this option is enabled and a helper fails while authenticating a user, it can still allow Squid to authenticate the user. This option should be used with care as it may lead to security loopholes.

--enable-external-acl-helpers
Squid supports external ACLs using helpers. If we are willing to use external ACLs, we should consider using this option. We can also use this option while learning. A list of external ACL helpers should be passed to build specific helpers. The default behavior is to build all the available helpers. A list of available external ACL helpers can be found in the helpers/external_acl/ directory in the Squid source code.

Copy
./configure --enable-external-acl-helpers=unix_group,ldap_group
--disable-translation
By default, Squid tries to present error and manual pages in a local language. If we don't want this to happen, then we may use this option.

--disable-auto-locale
Based on a client's request headers, Squid tries to automatically provide localized error pages. We can use this option to disable the automatic localization. The error_directory tag in the Squid configuration file must be configured if we use this option.

--disable-unlinkd
unlinkd is an external process which is used to make unlink system calls. This option disables unlinkd support in Squid. Disabling unlinkd is not a good idea as the unlink system call can block a process for a considerable amount of time, which can cause a delay in responses.

--with-default-user
We normally don't want to run Squid as the root user to omit any security risks. By default, Squid runs as the user nobody. However, if we have installed Squid from a pre-compiled binary, Squid may run as a 'squid' or 'proxy' user depending on the operating system we are using. Using this option, we can set the default user for running Squid. See the following example of how to use this option:

Copy
./configure --with-default-user=squid
--with-aufs-threads
Using this option, we can specify the number of threads to use when the aufs storage system is used for managing the cache directories. If this option is not used, Squid automatically calculates the number of threads that should be used:

Copy
./configure --with-aufs-threads=12
--without-pthreads
Older versions of Squid were built without POSIX threads support. Now, Squid is built with pthreads support by default, therefore, if we don't want to enable pthreads support, we'll have to explicitly disable it.

--with-openssl
If we want to build Squid with OpenSSL support, we can use this option to specify the OpenSSL installation path, if it's not installed in the default location:

Copy
./configure --with-openssl=/opt/openssl/ 
--with-large-files
Under heavy traffic, Squid's log files (especially the access log) grow quickly and in the long run the file size may become quite large. We can use this option to enable support for large log files.

Note
For better performance, it is good practice to rotate log files frequently instead of going with large files.

--with-filedescriptors
Operating systems use file descriptors (basically integers) to track the open files and sockets. By default, there is a limit on the number of file descriptors a user can use (normally 1024). Once Squid has accepted connections which have consumed all the available file descriptors to the Squid user, it can't accept more connections unless some of the file descriptors are released.

Under heavy load, Squid frequently runs out of file descriptors. We can use the following option to overcome the file descriptor shortage problem:

Copy
./configure --with-filedescriptors=8192
Note
We also need to increase the system-wide limit on the number of file descriptors available to a user.
### Have a go hero – 文件描述符
找出用户可用文件描述符的最大数量。另外，写下将最大可用文件描述符限制设置为8192的命令。
解决方案：要检查可用的文件描述符，请使用以下命令：
```Shell
ulimit -n
```
要将文件描述符限制设置为8192，我们需要下面的命令附加到/etc/security/limits.conf：
```Shell
username hard nofile 8192
username soft nofile 8192
```
只能是root用户或者超级用户执行前面的操作。

## Time for action – 执行配置脚本
现在，我们已经简要的简绍了几个选项，我们可以为构建Squid的环境设计一些选项。现在，我们准备使用以下选项运行configure命令：
```Shell
./configure --prefix=/opt/squid/ --with-logdir=/var/log/squid/ --with-pidfile=/var/run/squid.pid --enable-storeio=ufs,aufs --enable-removal-policies=lru,heap --enable-icmp --enable-useragent-log --enable-referer-log --enable-cache-digests --with-large-files
```
前面的命令将运行一段时间，检测系统环境的各种功能，并根据可用的库和模块做出决策。configure将调试输出写入同一目录中的config.log文件。在config.log中记录运行configure 命令时检查可能发生的任何错误。

如果一切正常，configure将在多个目录中生成 makefiles，下一步编译源代码时需要这些makefile文件。
### 刚刚讲述的内容
使用前面代码中提到的选项运行配置脚本，将生成编译启用模块的Squid源代码和源代码所需makefile脚本。它还将生成config.log和config.status文件。运行配置程序期间生成的所有消息都会记录到config.log文件中。config.status文件是一个可执行文件，可以运行它来重新创建makefiles。
### Have a go hero（试一试吧，好汉！） – 调试 configure 错误
在Squid的源码目录, 运行configure脚本, 就像下面这样:
```Shell
./configure --enable-storeio='aufs,disk'
```
现在试着检查出了什么问题并修复错误。

## Time for action(时机已到) – 编译源码
在了解我们的系统环境和构建需求后，我们需要进行实际的编辑，编译源码非常简单，只需要一个命令：
```Shell
make
```
我们不需要是root用户或超级用户来执行这个命令。根据系统硬件的不同，执行此命令可能需要的时间也不同。运行make将在终端产生大量输出信息，同时也会产生许多编译告警，在但是在大多数情况下可以忽略这些告警。

如果make出错，我们应该检查一下Squid bugzilla(开源的缺陷跟踪系统)是否有相同的问题。我们可以用我们的错误报告更新现有的错误，或者如果没有类似的错误，创建一个新的错误报告。有关故障排除和完成错误报告的详细信息，请参阅第12章“Squid故障排除”。

如果make结束，没有出错，那么就可以进入安装阶段。我们还可以再次运行make来验证是否成功编译了所有内容。再次运行make会生成许多类似于以下内容的行：
```Shell
Making all in compat
make[1]: Entering directory '/home/user/squid-source/compat'
make[1]: Nothing to be done for 'all'.
make[1]: Leaving directory '/home/user/squid-source/compat'
```
### 刚刚讲述的内容
我们刚刚运行了make命令，该命令将编译squid和相关模块的源代码，以生成可执行文件（如果完成时没有错误）。生成的可执行文件现在可以安装了

## Time for action（时机已到） – 部署安装 Squid
上一节中成功编译的源码将根据启用或禁用的功能和包生成所需的程序。但是，应该将它们移到指定的位置，以便使用。让我们执行安装的最后步骤。


根据 ${prefix},我们可能需要root用户或超级用户特权来安装Squid。 如果需要根用户或超级用户权限，我们应该首先使用以下命令切换到根用户或超级用户:
```Shell
su
```
现在，我们只需要运行make命令，其中install作为参数：

```Shell
make install
```
将程序移动到指定的位置，具体取决于运行配置程序时使用--prefix选项的路径。
### 刚刚讲述的内容
我们刚刚学习了如何安装Squid，即将生成的程序和其他基本文件放在指定的位置。
## Time for action – 浏览Squid文件目录
让我们看看安装期间生成的文件和目录。查看生成的目录和文件的最简单方法是使用tree命令。如果tree命令不可用，我们也可以使用ls命令列出文件。
```Shell
tree ${prefix} | less
```
${prefix}是./configure脚本 --prefix选项设置的目录，现在我们简单概述一下Squid部署期间生成的重要文件。一下的所有目录和文件都位于 --prefix指定的目录下：
### bin
没有root权限或超级用户权限的用户可以执行此目录包含的程序。
### bin/squidclient
squidclient 是一个具有高级功能的HTTP客户机，它允许修补HTTP请求以测试Squid服务器。运行squidclient显示可以使用的选项。
```Shell
${prefix}/bin/squidclient
```
### etc
etc是存放所有Squid相关的配置文件的地方
>注意:
>最好在使用configure配置脚本是设置 --sysconfdir=/etc/squid/选项，这样就可以测试不同安装的Squid时共享etc这个配置目录。
### etc/squid.conf
这是Squid配置文件的默认位置。安装期间生成的squid.conf是使用squid所需的最低配置。如果需要更改Squid配置，我们总是对此文件进行更改。
### etc/squid.conf.default
Squid生成这个默认配置文件，这样我们就可以将其复制并重命名为squid.conf，然后重新启动。
### etc/squid.conf.documented
这是squid.conf的完整文档版本，包含数千行注释。当我们安装了Squid版本时，我们应该总是参考这个文件来获取可用的配置标签。
### libexec
此目录包含在Squid编译期间生成的帮助程序。
### libexec/cachemgr.cgi
这个CGI程序提供了一个Web界面来管理称为缓存管理器的Squid缓存。
### sbin
此目录包含只能由具有root用户或超级用户权限执行的程序。
### sbin/squid
这是实际的squid程序，通常作为守护进程运行。
### share
这是squid使用的错误页面模板、文档和其他文件的位置。
### share/errors
此目录包含本地化的错误页模板。模板是HTML页面，我们可以通过修改这些HTML模板来自定义Squid显示的错误消息。
### share/icons
这个目录包含许多用于ftp或gopher目录列表的小图像。
### share/man
这是在编译期间为Squid、Squidclient和助手构建手册页的地方。手册页是可以使用命令man（在所有Linux/Unix发行版上都可用）查看的手册或帮助页。要查看位于/opt/squid/share/man/man8/squid.8的手册页，可以使用以下man命令：
```Shell
man /opt/squid/share/man/man8/squid.8
```
有关手册页的详细信息，请访问http://en.wikipedia.org/wiki/man_page。
### var
Squid运行时经常需要更改的文件。
### var/cache
这是将缓存的Web文档存储在硬盘上的默认目录。
### var/logs
这是Squid使用的所有日志文件（如cache.log、access.log等）的默认主页。
### What just happened?
我们刚刚查看了安装过程中生成的各种文件和目录，并简要概述了每个目录包含的内容。
### 从二进制软件包安装Squid
Squid二进制软件包在大多数操作系统的软件库中都可用，我们可以使用各自操作系统提供的软件包管理器来安装它们。接下来，我们将了解如何在一些操作系统上使用包管理器来安装Squid。
>注意：
>所有操作系统的软件存储库中可能没有最新版本或测试版。在这种情况下，我们应该从Squid网站获取最新或测试版，如本章前面所述。

#### Fedora, CentOS or Red Hat
yum是基于RPM的操作系统上一个流行的包管理器。Squid RPM在Fedora、Centos和Red Hat存储库中提供。要安装Squid，只需使用以下命令：
```Shell
yum install squid
```
#### Debian or Ubuntu
我们可以使用apt-get在Debian或Ubuntu上安装Squid：
```Shell
apt-get install squid
```
#### FreeBSD
Squid在FreeBSD端口集合中可用。以下命令可用于在FreeBSD上安装Squid：
```Shell
pkg_add -r squid31
```
有关freebsd中包管理的更多信息，请访问http://www.freebsd.org/doc/handbook/packages-using.html。
#### OpenBSD or NetBSD
在OpenBSD或NetBSD上安装Squid与在FreeBSD上安装Squid类似，可以使用以下命令执行：
```Shell
pkd_add squid31
```
要进一步了解OpenBSD和NetBSD中的包管理系统，请分别参阅http://www.openbsd.org/ports.html get和http://www.netbsd.org/docs/pkgsrc/using.html 安装二进制包。
#### Dragonfly BSD
要在蜻蜓BSD上安装Squid，可以使用以下命令：
```Shell
pkg_radd squid31
```
有关在Dragonfly BSD上安装二进制软件包的详细信息，请访问http://www.dragonfly bsd.org/docs/newhandbook/pkgsrc/。
#### Gentoo
我们可以使用emerge在Gentoo Linux上安装squid，如下所示：
```Shell
emerge =squid-3.1*
```
#### Arch Linux
要在arch linux上安装squid，我们可以使用包管理器 pacman，如下所示：
```Shell
pacman -s squid
```
有关pacman的更多信息，请访问https://wiki.archlinux.org/index.php/pacman。
#### 课后练习
1. 以下哪些Web文档不能由代理服务器缓存？
    - A HTML页
    - B jpeg图像
    - C 基于客户端IP地址生成输出的PHP脚本
    - D 一个javascript文件
2. 一下那种情况下，我们应该考虑 --enable diskio 选项
    - A 已启用RAM（主内存）中的缓存
    - B 已启用硬盘缓存
    - C 缓存被禁用
    - D 以上都不是
3. 删除策略选择何时影响Squid的整体性能？
    - A 如果缓存被禁用
    - B 如果在硬盘和RAM上启用缓存
    - C 删除策略选择与缓存无关
    - D 删除策略不会影响Squid的总体性能
## 总结
本章我们了解了代理服务器和Web缓存的作用和工作模式，尤其是在节省带宽和改善最终用户体验方面。然后我们继续研究Squid，它是一个强大的缓存代理服务器。以下是我们在本章学到的重要内容：
- 为生产或者开发而是用，获取Squid的各种方法
- 各种配置选项的含义
- 编译Squid源代码
- 从源和二进制包安装Squid
- 从源代码编译Squid的利弊
我们还讨论了安装期间Squid生成的目录结构和文件。既然我们知道了如何安装Squid，那么就可以学习如何根据给定网络环境的要求配置Squid了。我们将在下一章通过几个例子来了解这一点。
